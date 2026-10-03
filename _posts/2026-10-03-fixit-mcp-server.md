---
title: "Building FixIt: An MCP Server for Alexa+ That Answers From the Real Manual, Not a Guess"
date: 2026-10-03
layout: post
permalink: /fixit-mcp/
categories: [AI, MCP, AWS, Python]
tags: [MCP, Model Context Protocol, Alexa+, Amazon Bedrock, AgentCore, AI Agents, RAG, Document AI, Python, Evaluation, Open Source]
description: "How I built FixIt, an open-source MCP server for Alexa+ that diagnoses appliance problems from the real manufacturer manual. No model runs inside any tool, every answer cites its page, and the weak numbers are published next to the strong ones."
author: "Md. Shihabuddin Sadi"
---

*Software Engineer · AI Agents, MCP & RAG · Voice Agents · DevOps & Cloud Native*  
*October 03, 2026*  

<br>

### **Summary**

Your dryer beeps and shows **tE1**. The paper manual went out with the packaging. A web search gives you five forum threads about a different model, and an AI chatbot gives you a confident answer that may or may not match the machine in your laundry room.

**FixIt** answers from the **real manufacturer manual for the appliance you own**, and tells you which page it came from. Ask about an error code, or describe a symptom in plain words ("my washer has too many suds"), and you get the manual's causes and repair steps. If the manual doesn't say, FixIt says that too.

It is a self-hosted **MCP server** (spec 2025-11-25, Streamable HTTP) built for **Alexa+**, deployed on **Amazon Bedrock AgentCore**, and built for the *Build, Ship, Shape: Amazon Developer Hackathon*. The design rule that shaped everything: **no model runs inside any tool.** All the AI work happens offline. Every live answer is a fast, deterministic, testable lookup.

This post covers how it works, what broke, what I measured (including the numbers that don't flatter it), and what I'd tell anyone building tools for voice assistants.  
<br>
🎬 **Demo video (2 min):** [youtu.be/EbuSbJQJrXQ](https://youtu.be/EbuSbJQJrXQ)  
🔗 **GitHub Repository (MIT):** [sadishihab/fixit-mcp](https://github.com/sadishihab/fixit-mcp)  
📦 **Spin-off library:** [pdf-encoding-repair](https://github.com/sadishihab/pdf-encoding-repair) on [PyPI](https://pypi.org/project/pdf-encoding-repair/)

<br>

### Table of Contents

- [Summary](#summary)
- [Key Technologies Used](#key-technologies-used)
- [1. The Problem: Confident Answers About the Wrong Machine](#1-the-problem-confident-answers-about-the-wrong-machine)
- [2. The Design Rule: No Model Inside Any Tool](#2-the-design-rule-no-model-inside-any-tool)
- [Architecture Diagram](#architecture-diagram)
- [3. Six Tools, and What Each One Refuses to Say](#3-six-tools-and-what-each-one-refuses-to-say)
- [4. The PDFs That Lied Quietly](#4-the-pdfs-that-lied-quietly)
- [5. Extraction You Can Audit](#5-extraction-you-can-audit)
- [6. An Empty Field Is Not a Fact](#6-an-empty-field-is-not-a-fact)
- [7. Symptoms Without a Model](#7-symptoms-without-a-model)
- [8. Deploying on Bedrock AgentCore](#8-deploying-on-bedrock-agentcore)
- [9. Measuring It Honestly](#9-measuring-it-honestly)
- [10. The $12 Lesson About Credits](#10-the-12-lesson-about-credits)
- [11. Honest About the Client](#11-honest-about-the-client)
- [12. Built to Be Contributed To](#12-built-to-be-contributed-to)
- [13. How I Worked](#13-how-i-worked)
- [14. Learning Outcomes](#14-learning-outcomes)
- [References](#references)
- [License](#license)
<br>

### **Key Technologies Used**

`Python 3.12` · `uv` · `MCP Python SDK (spec 2025-11-25)` · `Streamable HTTP` · `MCP Apps` · `Amazon Bedrock AgentCore Runtime` · `AgentCore Memory` · `Amazon Bedrock (Claude, Nova Pro)` · `Amazon Polly` · `CloudWatch` · `SNS` · `ECR` · `Docker (ARM64)` · `FastAPI` · `PyMuPDF` · `pydantic` · `GitHub Actions`  
<br>

### **1. The Problem: Confident Answers About the Wrong Machine**

Appliance error codes are a perfect trap for a language model. They are short, cryptic, and **model-specific**: the same two characters can mean different things on two dryers from the same brand. A model that has read thousands of forum posts will happily produce a fluent, plausible answer, and fluency is exactly the problem. The person reading it has no way to tell whether it came from their manual or from someone else's.

Voice makes it worse. On a screen you might notice a hedge or check a link. Through a speaker, a confident sentence just sounds true.

So the requirements were simple to state and strict to meet:

- Answer **from the manual for the appliance this household actually owns**
- **Cite the page**
- When the manual is silent, **say so**, and never fill the gap with something that sounds right
- Be **fast enough for voice**, and **testable** enough to trust  
<br>

### **2. The Design Rule: No Model Inside Any Tool**

The obvious design is RAG at request time: retrieve some manual chunks, hand them to a model, let it write an answer. I deliberately didn't do that inside the tools.

Instead, the work is split in two:

- **Offline:** manuals are parsed, repaired, and extracted into structured records (error codes, causes, ordered repair steps, safety warnings, symptom tables) using Bedrock models under a strict schema, then audited against the source text. This happens once per manual.
- **Online:** the MCP tools look those records up in memory. No model call, no retrieval ranking, no sampling.

This buys four things at once:

1. **Speed.** A lookup is milliseconds. The tool never becomes the slow part of a voice turn.
2. **Determinism.** The same question returns the same record every time, so behaviour can be pinned in tests.
3. **Auditability.** Every field traces back to a page, and the extraction was checked against the source before it ever reached a user.
4. **Cost.** Extraction is paid once, not on every question.

The assistant (Alexa+, or any MCP client) still does what models are good at: understanding the question and phrasing the answer. It just isn't allowed to *be* the source. **The model presents, the tool decides.**  
<br>

#### **Architecture Diagram**
```pgsql
  OFFLINE (once per manual)                                      — the only place models are used
  ┌──────────────────────────────────────────────────────────────────────────────────────┐
  │  Manufacturer PDF ─► PyMuPDF parse ─► encoding repair ─► Bedrock extraction ─► audit  │
  │                      (section-aware)  (pdf-encoding-     (strict pydantic     (row     │
  │                                        repair)            schema)             counts,  │
  │                                                                               verbatim │
  │                                                                               join)    │
  │                                         └──────────────► error codes + symptom data   │
  └───────────────────────────────────────────────────────────────┬──────────────────────┘
                                                                  │ baked into the image
  ONLINE (every question)                                         ▼  — no model calls, ever
  ┌──────────────────────┐   MCP · Streamable HTTP   ┌──────────────────────────────────────┐
  │  Alexa+              │   IAM SigV4               │  Amazon Bedrock AgentCore Runtime     │
  │  (demo: simulated    │ ────────────────────────► │  FixIt MCP server (stateless)         │
  │   Alexa+ client,     │                           │  6 tools · in-memory lookups          │
  │   clearly labelled)  │ ◄──────────────────────── │  2 MCP Apps cards                     │
  └──────────────────────┘   cited answer / card     └───────────────┬──────────────────────┘
                                                                     │
                                         ┌───────────────────────────┼─────────────────────┐
                                         ▼                           ▼                     ▼
                                 AgentCore Memory            CloudWatch dashboard     SNS alerts
                                 (household appliances)      + error/latency alarms
```
<br>

### **3. Six Tools, and What Each One Refuses to Say**

FixIt exposes six MCP tools. What makes them useful isn't only what they return; it's what they are *not allowed* to return.

| Tool | What it does | What it refuses to do |
|---|---|---|
| `diagnose_error` | Error code → meaning, likely causes, ordered repair steps, safety warnings, page citation | Guess at a code that isn't in the manual |
| `diagnose_symptom` | Plain-language problem → the manual's troubleshooting rows | Invent a cause; it returns *not found* instead |
| `check_warranty` | Deterministic date check against the recorded warranty | Claim anything is *covered* |
| `list_my_appliances` | What this household owns | — |
| `add_appliance` | Add by model number, link to its manual | — |
| `remove_appliance` | Remove one | — |

Here is what it looks like in practice, from the demo:

```text
User:   My dryer is showing tE1, what should I do?
FixIt:  tE1 — Temperature sensor failure
        Likely cause: temperature sensor failure
        Repair: 1. Turn off the dryer and call for service
        Source: manufacturer manual, p. 31

User:   My dryer shows ZZ99.
FixIt:  The code ZZ99 isn't in the manuals we have.
```

`diagnose_error` and `diagnose_symptom` also render **MCP Apps** cards, so a screen-capable client shows the causes, steps, and citation visually, with distinct states for *found*, *not found*, and *which appliance did you mean?*  
<br>

### **4. The PDFs That Lied Quietly**

The first real bug had nothing to do with AI.

Two of the seven manuals extracted as text that *looked* like text: right length, plausible spacing, plausible punctuation. It was garbage. Every character had been shifted by a **constant offset** because of a broken font encoding, and the offset was **different in each manual**.

Nothing errored. PyMuPDF happily returned strings. A RAG pipeline would have embedded them, indexed them, and retrieved them, and nobody would have noticed until a model started answering from nonsense.

The fix detects the pattern, works out the offset per document, repairs the text, and has **guards against false positives** so clean PDFs are left untouched. Because this is a silent failure that anyone ingesting PDFs can hit, I didn't leave it buried in FixIt. It became a standalone MIT-licensed library, **[pdf-encoding-repair](https://github.com/sadishihab/pdf-encoding-repair)**, published on PyPI with CI and a tagged release.

The general lesson: **the most dangerous ingestion bugs don't throw exceptions.** They produce output that is the right shape and the wrong content.  
<br>

### **5. Extraction You Can Audit**

Offline extraction is where the models earn their keep, and also where they can quietly drop or invent rows. A table with twelve symptoms that comes back with eleven is a silent loss. A "cause" that appears nowhere in the manual is a silent invention.

So extraction runs under two audits:

- **Row-count audit:** the number of extracted rows must match the number of rows in the source table.
- **Verbatim-join audit:** extracted text must join back to the source text, so a field the model paraphrased or made up fails loudly.

The symptom tables (114 rows across two manuals) were extracted with **Amazon Nova Pro** under these audits, for about **$0.10**. Where a manual's own text is cut off, the record is marked **"incomplete in the manual"** rather than completed by the model. You can see that tag on the suds card in the demo.

Today the data covers **7 real manufacturer manuals**, **46 error-code records** from five of them, and **114 symptom rows** from two.  
<br>

### **6. An Empty Field Is Not a Fact**

This is the finding I'm proudest of, because it came from watching real conversations rather than reading test output.

A user asked whether an error code was dangerous. The tool returned the record faithfully, and its safety-warning list was **empty**, because the manual lists no warning for that code. The assistant turned that into: *"It's safe."*

Nothing in the data said that. The model read an absence and reported it as a fact.

The fix was a rule, enforced in the prompt and pinned with tests so it can't quietly regress:

> **Say what the tool returned, and stop.** An empty warning list becomes *"the manual doesn't list a specific warning for this code"*, never *"it's safe."* An unknown code becomes *"not found"*, never a nearby guess. A warranty date becomes *"the recorded warranty ended on…"*, never *"you're covered."*

`check_warranty` is built the same way. It's a deterministic date comparison that reports only the **recorded** date. Coverage depends on terms, receipts, and conditions FixIt can't see, so it never claims coverage at all.

If you're building tools for assistants, this generalises: **design your tool outputs so that the honest sentence is the easy sentence to produce.**  
<br>

### **7. Symptoms Without a Model**

Error codes are easy to look up. Symptoms are hard: people say "it's full of bubbles", "too much foam", "soapy mess". Keeping to the no-model rule meant `diagnose_symptom` had to be a **keyword matcher**, which is the honest choice and also the limited one.

So I measured it against a bank of invented paraphrases:

| Measurement | Before tuning | After tuning |
|---|---|---|
| Tuning set recall | 25 / 47 | **43 / 47** |
| Wrong first match | 4 | **0** |
| "None of these" cases wrongly matched | 7 / 31 | **0 / 31** |
| **Held-out recall** | 4 / 10 | **4 / 10** |

The last row is the important one. On phrasings I hadn't tuned against, recall didn't move, and there is one known wrong answer ("rocking back and forth"). Tuning made the matcher better at the examples I'd written, not at language in general.

I publish that row on purpose. A matcher that returns *not found* on six out of ten novel phrasings is limited. A matcher that confidently matched the wrong manual row would be dangerous. FixIt is designed to fail in the first way.  
<br>

### **8. Deploying on Bedrock AgentCore**

The server runs on **Amazon Bedrock AgentCore Runtime** as an ARM64 container (platform V2) with **IAM SigV4** inbound auth. Getting there was the most friction-heavy part of the build, and I logged every snag as it happened.

**State disappeared between conversations.** Each Runtime session runs in its own microVM. My first version kept household appliances in local SQLite, which worked perfectly in Docker and forgot everything in production, because every new session started from a clean slate. Household state moved to **AgentCore Memory**, which survives across sessions.

**IAM permissions fanned out.** `CreateAgentRuntime` needs sub-permissions that the docs didn't list, so they surfaced one `AccessDenied` at a time.

**The docs pointed the wrong way.** The custom-container guide I followed first didn't fit an MCP server, and an AWS sample broke under `mcp` 2.x. I pinned `mcp>=1.30,<2` and moved on.

**Cold starts are real.** Warm calls are fine for voice. A **brand-new session takes about 5 seconds**, and it isn't documented how often Alexa+ would open one. I also saw an occasional front-door 504 (twice) and a harmless but confusing `Session termination failed: 404` log line.

**Every deploy follows the same routine:** build the image, run it locally and smoke-test it, push it to ECR with a **source-drift check** that refuses to continue unless the image matches the current git commit, deploy a new runtime version (same ARN, version + 1), smoke-test the live runtime, then confirm a real household's data in Memory was left untouched.

**Observability:** a CloudWatch dashboard, alarms on errors and tool latency, and SNS email alerts.  
<br>

### **9. Measuring It Honestly**

**Grounding.** I built an evaluation that asks realistic questions end to end and has a second model judge whether every claim in the answer is supported by the tool output. In one run of **63 cases**, **62 were fully grounded** (Claude Sonnet 4.5 answering, Claude Opus 4.6 judging). The case file has since grown to 72 cases, and I haven't re-run it on Claude, so 62/63 is the number I can stand behind.

**Judges need judging too.** I tried Amazon Nova Pro as both answerer and judge on six new symptom cases. Three of six passed, and the Nova judge **missed an invented "normal"**: the answer called a condition normal when the manual said no such thing, and the judge let it through. A cheap judge that misses fabrications isn't cheap. It's blind.

**Latency.** From Dhaka, with roughly 320 ms of network round trip, warm calls measured about **500–650 ms** client side. Cold sessions take about **5 seconds**.

None of these numbers are perfect, and that's the point of publishing them. A number you only share when it flatters you isn't a measurement.  
<br>

### **10. The $12 Lesson About Credits**

The hackathon came with a **$150 AWS credit**, and I planned to spend only that. Most of the stack was covered: AgentCore, Memory, ECR, CloudWatch, SNS, Polly, and Nova all billed against the credit.

**Claude models on Bedrock did not.** They are sold through **AWS Marketplace**, and Marketplace charges aren't covered by promotional credits. I found out from my card statement, about **$12** in September, and then confirmed it with the organizers.

The fix was operational rather than clever: an inline IAM **deny policy** on Anthropic models (Nova stays allowed), lifted only while recording the demo and put straight back, plus a $3 budget alert. It's now an entry in the friction log, because the next builder deserves to know before the statement arrives.  
<br>

### **11. Honest About the Client**

Alexa+ developer tools (account linking, the local inspector, add-on submission) **aren't available to hackathon participants**; the organizers confirmed this by email. So the demo uses a **simulated Alexa+ client**: a FastAPI backend and a web page that drive the *deployed* server through a Bedrock tool-use loop and render the MCP Apps cards, with optional Amazon Polly voice.

It is labelled as simulated on every screen, and the project claims no more than that. The MCP server is real and deployed. The client is a stand-in, and the next step is the OAuth account-linking path that would let real Alexa+ call it.  
<br>

### **12. Built to Be Contributed To**

I want FixIt to outlive the hackathon, so it's set up like a project other people can work on:

- **MIT license**, CI on every push, tagged releases, and a CHANGELOG
- A **no-AWS quickstart** (`make try-it`) so contributors can run the server without an AWS account
- **"Good first issue"** tickets, issue and PR templates, labels, and a security policy
- An **architecture guide** with a diagram that colours the parts that call models differently from the parts that never do
- A **guide for manufacturers** who want their manuals supported
- A **friction log with 60+ real entries** for the Amazon developer teams, each with the task, the steps, expected vs. actual, a severity, the workaround, and a suggestion  
<br>

### **13. How I Worked**

I built FixIt with AI assistance and the discipline I'd expect from a team:

1. **Planning chat:** design, review, and breaking every change into one small step
2. **Claude Code in the repo:** implementation, with a report after each step
3. **A plain terminal:** I ran every command myself and audited each report for what was *verified*, what was *unverified*, and what was *inferred*

A few rules did a lot of work. **Nothing is "working" until there is output that shows it.** **Before every push, a leak check** greps the outgoing history for account and resource identifiers, which must never land in a public repo. And **the deployed image must match the commit** it claims to be, or the deploy stops.  
<br>

### **14. Learning Outcomes**

- Designed an **MCP server for voice** where no model runs inside any tool, keeping answers fast, deterministic, and testable  
- Built an **offline extraction pipeline** with row-count and verbatim-join audits so models can't silently drop or invent rows  
- Found and fixed **silent PDF encoding corruption**, and shipped the fix as an open-source library on PyPI  
- Learned that **an empty field is not a fact**, and designed tool outputs so the honest sentence is the easy one  
- **Measured a keyword matcher honestly**, including the held-out number that didn't improve  
- Deployed on **Amazon Bedrock AgentCore** with **AgentCore Memory** for state across microVM sessions  
- Built **observability** with CloudWatch dashboards, alarms, and SNS alerts  
- Ran a **grounding evaluation** with an LLM judge, and learned to evaluate the judge too  
- Discovered how **Marketplace billing** interacts with promotional credits, and contained it with IAM  
- Kept a **60+ entry friction log** that turns my rough edges into the platform team's backlog  

<br>

### **Try It**

Watch the [2-minute demo](https://youtu.be/EbuSbJQJrXQ), then clone [the repo](https://github.com/sadishihab/fixit-mcp) and run `make try-it`. You don't need an AWS account.

If you're building AI agents, MCP servers, or voice experiences and need them to be right rather than just fluent, I'm happy to talk: [book a 30-minute call](https://calendly.com/sadi-shihab/30min) or find me on [LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi).

<br>

### **References**
- [FixIt on GitHub](https://github.com/sadishihab/fixit-mcp)
- [FixIt demo video](https://youtu.be/EbuSbJQJrXQ)
- [FixIt on Devpost](https://devpost.com/software/fixit-ai-home-appliance-repair-agent-for-alexa)
- [pdf-encoding-repair on GitHub](https://github.com/sadishihab/pdf-encoding-repair)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Amazon Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/)
- [Amazon Bedrock](https://aws.amazon.com/bedrock/)
- [PyMuPDF Documentation](https://pymupdf.readthedocs.io/)
<br>

### **License**
This project is open-source and available under the MIT License
