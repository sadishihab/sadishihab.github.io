---
layout: default
title: Md. Shihabuddin Sadi — AI Agents, MCP Servers, RAG & Voice Agents
description: AI agents, MCP servers, RAG chatbots and voice agents that answer from the source, not a guess. Ex-Samsung R&D, 17+ years. Available for contract work.
image: /vector-forge-og-image-v2.png
---

<!-- Preconnect to shields.io for faster badge loading -->
<link rel="preconnect" href="https://img.shields.io" crossorigin>

<!-- Open Graph / social share -->
<meta property="og:title" content="Md. Shihabuddin Sadi — AI Agents, MCP, RAG & Voice Agents">
<meta property="og:description" content="Production AI agents, MCP servers, RAG chatbots and voice agents that answer from the source, not a guess. Ex-Samsung R&D · 17+ years.">
<meta property="og:image" content="https://sadishihab.github.io/vector-forge-og-image-v2.png">
<meta name="twitter:card" content="summary_large_image">
<meta property="og:type" content="website">
<meta property="og:url" content="https://sadishihab.github.io/">


## I help teams ship reliable AI systems for real customers.

**AI agents, MCP servers, RAG chatbots and voice agents that answer from the source, not a guess.**
Grounded in your real documents, with validation layers that refuse to write down what nobody said. Built for real users, real traffic, real outcomes.

I run **Vector Forge** · Ex-Samsung R&D · 17+ years of software engineering · Based in Dhaka, Bangladesh · Available worldwide remote

Available for direct engagements and for white-label contract work behind agencies and product studios.

<div style="display:flex; flex-wrap:wrap; gap:12px; margin: 20px 0 30px 0;">

<a href="https://calendly.com/sadi-shihab/30min" style="background:#0a66c2; color:white; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600;">
📅 Book a 30-min call
</a>

<a href="#featured-project" style="background:#f3f2ef; color:#0a66c2; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600; border:1px solid #0a66c2;">
See my work
</a>

<a href="https://youtu.be/EbuSbJQJrXQ" style="background:#f3f2ef; color:#0a66c2; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600; border:1px solid #0a66c2;">
🎬 Watch the FixIt demo
</a>

</div>

<br>

---

## What I Build

**MCP servers and agent tools** that plug your product, documents or data into AI assistants. The tools are deterministic, the answers cite their source, and where it matters there is no model in the live request path, so responses are fast, predictable and testable. Hosted on AWS (Amazon Bedrock AgentCore, or your own containers), with observability from day one.

Production RAG chatbots over your docs, PDFs, Notion, or SQL — with citations, not hallucinations. That includes the unglamorous ingestion work: messy PDFs, broken text encodings, and schema-checked extraction audited against the source. Multilingual AI agents that handle Bangla, Banglish, English, and other low-resource or script-mixed languages, which makes them especially useful for South Asian, Middle East, and emerging-market audiences.

Voice agents that take calls and fill in structured records — intake, verification, booking, triage — with a validation layer so a mis-heard account number or name never quietly lands in your database. If your agent is going to write to a system of record, that layer is not optional.

Beyond that, I build Messenger, WhatsApp, Telegram, and Slack bots wired to real business data, custom AI copilots embedded inside SaaS products, and the evaluation pipelines, observability, and guardrails that keep all of it from silently regressing in production.

I also handle the cloud infrastructure to keep it running reliably — AWS, Kubernetes, Terraform, CI/CD, CloudWatch, Prometheus, Grafana. One contractor, one accountable line, no hand-off between the AI person and the DevOps person.

<br>

---

<a id="featured-project"></a>

## Featured Project — FixIt: Appliance Repair Answers from the Real Manual

A self-hosted **MCP server built for Alexa+** that diagnoses home appliance problems from the **real manufacturer manual for the appliance you own**, and cites the page. Ask about an error code, or describe a symptom in plain words ("my washer has too many suds"), and FixIt returns the manual's causes and repair steps. If the manual doesn't say, FixIt says that too. It never invents a repair step or a safety claim.

Built for the Build, Ship, Shape: Amazon Developer Hackathon (Alexa+ track, plus the AWS Builder and Open Source mini challenges), and maintained as an open-source project.

**[🎬 Watch the demo](https://youtu.be/EbuSbJQJrXQ)** · **[📦 View source (MIT)](https://github.com/sadishihab/fixit-mcp)** · **[🏁 Devpost](https://devpost.com/software/fixit-ai-home-appliance-repair-agent-for-alexa)**

> The finding that shaped the whole build: in real conversations, the assistant described an appliance code as "safe" because the manual's safety-warning list for it was empty. An empty field is not a fact. Now an empty field produces "the manual doesn't list one", an unknown code produces "not found", and both rules are pinned by tests so they can't quietly regress.

**Key decisions:**

- **No model inside any tool.** All AI work happens offline: manuals are parsed, repaired and extracted into structured records under a strict schema, then audited against the source text. The six live tools are fast in-memory lookups: diagnose an error code, diagnose a symptom, check the recorded warranty, and list, add or remove the household's appliances. Two of them render MCP Apps visual cards.
- **Silent PDF corruption, found and fixed at the source.** Two of the seven manuals extracted as plausible-looking text that was actually garbage, each shifted by a constant character offset (a different offset per manual). Detection and repair, with guards against false positives, became a standalone open-source library: [pdf-encoding-repair](https://github.com/sadishihab/pdf-encoding-repair).
- **The warranty tool reports, it never promises.** It is a deterministic date check that states the recorded warranty date, and never claims something is covered.
- **Measured, including the weak spots.** In one grounding-eval run, 62 of 63 cases were fully grounded. The keyword symptom matcher was tuned from 25 to 43 of 47 test paraphrases with 0 wrong first matches, but held-out recall stayed at 4 of 10, and that is published rather than hidden. Warm calls take roughly 500–650 ms from Dhaka (about 320 ms of it network round trip). A brand-new session takes about 5 seconds.
- **Production deployment on AWS.** The server runs on Amazon Bedrock AgentCore Runtime with IAM SigV4 auth. Household state lives in AgentCore Memory, because every Runtime session is its own microVM and local storage forgot everything between conversations. A CloudWatch dashboard, error and latency alarms, and SNS alerts watch it.
- **Honest about the client.** Alexa+ developer tools aren't open to outside builders yet, so the demo uses a clearly labelled *simulated* Alexa+ client, and the project claims no more than that.

**Stack:** Python 3.12 · MCP Python SDK (spec 2025-11-25, Streamable HTTP, MCP Apps) · Amazon Bedrock AgentCore (Runtime, Memory) · Amazon Bedrock (Claude, Nova Pro) · Amazon Polly · CloudWatch · SNS · ECR · Docker · FastAPI · PyMuPDF · pydantic · GitHub Actions

**At a glance:** 7 real manufacturer manuals · 46 error-code records · 114 symptom rows · 6 MCP tools · 60+ entry friction log for the Amazon developer teams

**Built to be contributed to:** MIT license, CI, tagged releases, a quickstart that runs without an AWS account, "good first issue" tickets, and a guide for manufacturers who want their manuals supported.

[View on GitHub](https://github.com/sadishihab/fixit-mcp)

<br>

---

## Featured Project — Claim Intake Voice Agent

A voice agent that takes insurance claims by phone and **cannot write a value into the record unless server-side code approves it**. The agent listens and proposes. A validator decides, returning one of three verdicts — accepted, unconfirmed, or rejected — along with the exact sentence the agent then reads back, spelled phonetically.

**[🎧 Call it yourself](https://claims.sadishihab.com)** · **[📊 See what validation catches](https://claims.sadishihab.com/compare)**

> The finding that shaped the whole build: I fed the speech recogniser the list of valid policy numbers to improve accuracy. It improved — and it started rewriting mis-heard numbers into real ones. A transcript that began as *C411* came back as a genuine policy number belonging to a different customer, and it passed validation perfectly, because the match had been manufactured before the check ever ran.

**Key decisions:**

- **The deciding layer contains no AI.** Plain Python reads the policy data and returns a verdict. Clever systems make confident mistakes; boring ones don't.
- **An exact match is not proof.** Policy numbers require spoken confirmation whatever the match quality, because the transcript stopped being an independent observation the moment the recogniser knew the answers.
- **Three attempts to fix it upstream, all measured, all null.** Transcription mode, turn-detection patience, and tool-schema hints each left the same failure in place. The negative result is the argument for validating downstream.
- **Consent is bound to the question asked.** Agreeing a value was heard correctly is not agreeing to overwrite a value already recorded. Two consents, enforced in code.
- **Full evidence trail** — every attempt, including the rejections, linked to what the caller actually said and when. Crash-safe, and purged after 24 hours so caller data doesn't accumulate.
- **206 tests**, of which 62 cover the validation layer alone — all three verdicts, plus confusable policy pairs like `BX7-4402` and `BX7-4420`. Several pin design decisions rather than behaviour, so a later change that undoes one fails and explains why it existed.

**Stack:** Python 3.14 · AssemblyAI Voice Agent API (Universal-3.5 Pro) · FastAPI · raw WebSocket relay · AudioWorklet (PCM16 @ 24 kHz) · Server-sent events · Docker · NGINX · Let's Encrypt

**Recognised by the platform team:** three documentation corrections from this build were published by AssemblyAI, and the recogniser-bias finding was escalated to their research team.

[View on GitHub](https://github.com/sadishihab/claim-intake-agent)

<br>

---

## Featured Project — Minimal RAG Chatbot

A production multilingual RAG chatbot deployed on Facebook Messenger for **Minimal Limited**, an interior design company in Dhaka. Customers send questions in **Bangla, Banglish, or English** — the bot always replies in **formal Bangla**, grounded in a curated knowledge base, with graceful human takeover when confidence is low.

> The one decision that paid off most: embed the question, not the answer. Customers send questions, so questions belong in the searchable space. Fixed more "wrong answer" bugs than any prompt tweak.

**Key decisions:**

- Architected from scratch — **no LangChain, no LlamaIndex** — so every line of the pipeline is transparent and debuggable in production
- **Similarity-threshold fallback**: if the top match isn't strong enough, the bot says *"share your number, our manager will call"* instead of hallucinating
- **Cross-lingual prompt engineering**: input accepted in any of three languages, output strictly enforced as formal Bangla
- **4-stage safe deployment**: terminal, local web, test FB page, live page
- **12 passing pytest tests** covering schema, language enums, intent coverage, and answer-length rules

**Stack:** Python 3.13 · OpenAI (`text-embedding-3-small`, `gpt-4o-mini`) · FAISS (`IndexFlatIP`, L2-normalized) · FastAPI · Uvicorn · Facebook Graph API · Pytest

**At a glance:** 224 curated Q&A entries · 14 intents · top-k=3 retrieval · embedding-dim 1536

[Read the full case study](/minimal-rag-chatbot/) · [View on GitHub](https://github.com/sadishihab/minimal-rag-chatbot)

<br>

---

## Featured Project — Error Journal

Paste any error — a Python traceback, a Kubernetes pod crash, a Docker build failure — and get a real fix back. Hit the exact same problem again later, even on a different machine, and it recognizes it and tells you what fixed it last time. Live on the Anna App Store.

**[🚀 Try it live](https://anna.partners/store/@sadi/error-journal)** · **[📦 View source](https://github.com/sadishihab/error-journal)**

> The design call that shaped the whole thing: two logs of the same underlying error almost never look byte-identical — timestamps, pod names, and file paths all differ. Strip everything volatile, classify what remains, hash it, and the same problem gets recognized as the same problem no matter how differently it's phrased each time.

**Key decisions:**

- **109 curated diagnoses**, hand-written and verified, across Python, JavaScript/Node, Go, Java, Rust, Ruby, PHP, plus Kubernetes, Docker, shell, and networking. Outside that list, it says *"not in my playbook"* honestly rather than inventing a fix — a wrong fix during an outage is worse than no fix.
- **Runnable commands, not templates.** Real pod names, ports, and module names get substituted into fix steps, gated behind a strict allow-list so error text pasted by a user can never become a shell-injection vector in a command someone copies and runs.
- **Python, stdlib only** — no dependency chain to break, shipped as single-file binaries for Linux, macOS, and Windows via PyInstaller in a GitHub Actions matrix, with a smoke test on every platform before release.
- **Testing surfaced real bugs**, including ANSI color codes silently breaking detection when copied from CI logs, and log-line prefixes like syslog and pytest tags causing correct errors to go unrecognized.

**Stack:** Python (stdlib only) · PyInstaller · JSON-RPC · GitHub Actions

**Also produced:** a [reusable app template](https://github.com/sadishihab/anna-app-template) extracted from this build, so the next developer on the same platform doesn't have to rediscover the same setup issues.

[View on GitHub](https://github.com/sadishihab/error-journal)

<br>

---

## Why Teams Hire Me

I've shipped real software for 17+ years — not just AI demos.

I bring engineering rigor: evals, logging, retrieval tuning, and guardrails. The unglamorous work that decides whether your AI survives contact with real users.

I keep the model out of the places it doesn't belong. In FixIt, every live answer is a deterministic lookup into data extracted and audited offline; in the claim intake agent, the layer that decides what gets written contains no AI at all. Models are excellent at proposing. Code should decide.

I also measure before I ship, and I publish the weak numbers alongside the strong ones. Three separate attempts to tune my way out of a speech recognition problem were tested and thrown away because the numbers said they didn't work. FixIt's symptom matcher reports its held-out recall of 4 in 10 right next to its tuned results, because a number you only show when it's flattering isn't a measurement.

And I don't trust a metric until I know what it hides. On a recent project, FP16 quantization looked fine on mean error while 15% of gripper commands silently flipped sign — close became open. The average was healthy and the system was broken. Finding that class of failure is most of what reliability work actually is.

And because I can build both the AI and the cloud infrastructure it runs on, there's no hand-off between the AI person and the DevOps person. One contractor, one accountable line.

<br>

---

## For Agencies & Product Studios

If your team sells AI work and needs the engineering layer behind it, I work as a white-label contractor under your brand and under NDA. You keep the client relationship. I stay invisible unless you want me on the call.

Where I usually come in:

- **MCP servers and assistant integrations** — getting a client's product or knowledge into AI assistants with tools that are fast, testable, cited, and deployed with monitoring
- **RAG that has to hold up** — retrieval quality, grounding, citations, evaluation, and a defensible answer when the system doesn't know
- **Agent and voice backends** — tool calling, server-side validation, confirmation flows, and the boundary between what the model proposes and what gets written
- **The demo-to-production gap** — the build worked in the pitch and started failing with real users, and someone needs to find out why
- **Infrastructure** — Docker, AWS, Kubernetes, Terraform, CI/CD, monitoring, and observability, so the AI work doesn't stall waiting on deployment

Scoped projects or ongoing capacity, whichever suits the engagement.

<br>

---

## How We'd Work Together

1. **30-min discovery call** — tell me about your product, your data, and where AI fits
2. **Scoped proposal within 48 hours** — what I'd build, timeline, cost
3. **Build, ship, iterate** — typically 2–6 weeks for a production RAG, agent or voice pilot
4. **Optional ongoing support** — evals, observability, infra, and iteration

Agency engagements are usually faster: a short technical call on the work already sold, a scoped estimate, and a start date.

<div style="margin: 20px 0;">
<a href="https://calendly.com/sadi-shihab/30min" style="background:#0a66c2; color:white; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600;">
📅 Book a discovery call
</a>
</div>

<br>

---

## What People Say

> "Sadi is a highly skilled solutions architect, DevOps expert, and technical project manager with a deep understanding of software development, system architecture, and cloud infrastructure. His ability to streamline complex processes, optimize workflows, and enhance system efficiency made him a key asset to Samsung's R&D initiatives. I highly recommend Md. Shihabuddin Sadi to anyone seeking a dedicated, skilled, and forward-thinking technical leader."
>
> — **Md Elme Focruzaman Razi**, Senior Staff Engineer at Samsung R&D Institute Bangladesh ([LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi/details/recommendations/))

<br>

---

## Tech Stack

### AI Agents, MCP & AWS AI

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://modelcontextprotocol.io/">
<img src="https://img.shields.io/badge/Model%20Context%20Protocol-1A1A1A?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://modelcontextprotocol.io/docs/extensions/apps">
<img src="https://img.shields.io/badge/MCP%20Apps-3B3B98?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://aws.amazon.com/bedrock/agentcore/">
<img src="https://img.shields.io/badge/Bedrock%20AgentCore-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white" height="28">
</a>

<a href="https://aws.amazon.com/bedrock/">
<img src="https://img.shields.io/badge/Amazon%20Bedrock-01A88D?style=for-the-badge&logo=amazon-aws&logoColor=white" height="28">
</a>

<a href="https://aws.amazon.com/ai/generative-ai/nova/">
<img src="https://img.shields.io/badge/Amazon%20Nova-8C4FFF?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://aws.amazon.com/polly/">
<img src="https://img.shields.io/badge/Amazon%20Polly-527FFF?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://www.anthropic.com/claude-code">
<img src="https://img.shields.io/badge/Claude%20Code-D97757?style=for-the-badge&logoColor=white" height="28">
</a>

</div>

<br>

### AI / LLM / RAG & Document AI

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://openai.com/">
<img src="https://img.shields.io/badge/OpenAI-412991?style=for-the-badge&logo=openai&logoColor=white" height="28">
</a>

<a href="https://github.com/facebookresearch/faiss">
<img src="https://img.shields.io/badge/FAISS-0467DF?style=for-the-badge&logo=meta&logoColor=white" height="28">
</a>

<a href="https://fastapi.tiangolo.com/">
<img src="https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white" height="28">
</a>

<a href="https://www.uvicorn.org/">
<img src="https://img.shields.io/badge/Uvicorn-2C2C2C?style=for-the-badge&logo=uvicorn&logoColor=white" height="28">
</a>

<a href="https://docs.pydantic.dev/">
<img src="https://img.shields.io/badge/Pydantic-E92063?style=for-the-badge&logo=pydantic&logoColor=white" height="28">
</a>

<a href="https://pymupdf.readthedocs.io/">
<img src="https://img.shields.io/badge/PyMuPDF-1B5E20?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://numpy.org/">
<img src="https://img.shields.io/badge/NumPy-013243?style=for-the-badge&logo=numpy&logoColor=white" height="28">
</a>

<a href="https://docs.pytest.org/">
<img src="https://img.shields.io/badge/Pytest-0A9EDC?style=for-the-badge&logo=pytest&logoColor=white" height="28">
</a>

<a href="https://developers.facebook.com/docs/messenger-platform">
<img src="https://img.shields.io/badge/Messenger%20Platform-0084FF?style=for-the-badge&logo=messenger&logoColor=white" height="28">
</a>

</div>

<br>

### Voice & Real-Time

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://www.assemblyai.com/">
<img src="https://img.shields.io/badge/AssemblyAI-2545D3?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://www.assemblyai.com/docs/voice-agents/voice-agent-api">
<img src="https://img.shields.io/badge/Voice%20Agent%20API-1B1B3A?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API">
<img src="https://img.shields.io/badge/WebSockets-4A4A4A?style=for-the-badge&logo=socketdotio&logoColor=white" height="28">
</a>

<a href="https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API">
<img src="https://img.shields.io/badge/Web%20Audio%20API-BF360C?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events">
<img src="https://img.shields.io/badge/Server--Sent%20Events-006064?style=for-the-badge&logoColor=white" height="28">
</a>

</div>

<br>

### AI Agents & Verification

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://bob.ibm.com/">
<img src="https://img.shields.io/badge/IBM%20Bob%202.0-052FAD?style=for-the-badge&logo=ibm&logoColor=white" height="28">
</a>

<a href="https://en.wikipedia.org/wiki/Mutation_testing">
<img src="https://img.shields.io/badge/Mutation%20Testing-5C2D91?style=for-the-badge&logoColor=white" height="28">
</a>

</div>

<br>

### Model Optimization & Edge Inference

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://docs.openvino.ai/">
<img src="https://img.shields.io/badge/OpenVINO-0068B5?style=for-the-badge&logo=intel&logoColor=white" height="28">
</a>

<a href="https://github.com/openvinotoolkit/nncf">
<img src="https://img.shields.io/badge/NNCF%20INT8-1B4F72?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://pytorch.org/">
<img src="https://img.shields.io/badge/PyTorch-EE4C2C?style=for-the-badge&logo=pytorch&logoColor=white" height="28">
</a>

<a href="https://www.sbert.net/">
<img src="https://img.shields.io/badge/sentence--transformers-2C3E50?style=for-the-badge&logoColor=white" height="28">
</a>

<a href="https://mujoco.org/">
<img src="https://img.shields.io/badge/MuJoCo-4A235A?style=for-the-badge&logoColor=white" height="28">
</a>

</div>

<br>

### Programming

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://www.python.org/">
<img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" height="28">
</a>

<a href="https://isocpp.org/">
<img src="https://img.shields.io/badge/C++-00599C?style=for-the-badge&logo=cplusplus&logoColor=white" height="28">
</a>

<a href="https://en.wikipedia.org/wiki/C_(programming_language)">
<img src="https://img.shields.io/badge/C-A8B9CC?style=for-the-badge&logo=c&logoColor=black" height="28">
</a>

<a href="https://www.java.com/">
<img src="https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=java&logoColor=white" height="28">
</a>

<a href="https://www.gnu.org/software/bash/">
<img src="https://img.shields.io/badge/Bash-121011?style=for-the-badge&logo=gnu-bash&logoColor=white" height="28">
</a>

<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript">
<img src="https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black" height="28">
</a>

<a href="https://yaml.org/">
<img src="https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white" height="28">
</a>

<a href="https://www.mysql.com/">
<img src="https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white" height="28">
</a>

</div>

<br>

### DevOps & Cloud Infrastructure

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://www.docker.com/">
<img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" height="28">
</a>

<a href="https://kubernetes.io/">
<img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" height="28">
</a>

<a href="https://aws.amazon.com/">
<img src="https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white" height="28">
</a>

<a href="https://aws.amazon.com/ecr/">
<img src="https://img.shields.io/badge/Amazon%20ECR-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white" height="28">
</a>

<a href="https://www.terraform.io/">
<img src="https://img.shields.io/badge/Terraform-623CE4?style=for-the-badge&logo=terraform&logoColor=white" height="28">
</a>

<a href="https://www.jenkins.io/">
<img src="https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white" height="28">
</a>

<a href="https://www.ansible.com/">
<img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" height="28">
</a>

<a href="https://github.com/features/actions">
<img src="https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white" height="28">
</a>

<a href="https://nginx.org/">
<img src="https://img.shields.io/badge/NGINX-009639?style=for-the-badge&logo=nginx&logoColor=white" height="28">
</a>

<a href="https://letsencrypt.org/">
<img src="https://img.shields.io/badge/Let's%20Encrypt-003A70?style=for-the-badge&logo=letsencrypt&logoColor=white" height="28">
</a>

<a href="https://www.kernel.org/">
<img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" height="28">
</a>

</div>

<br>

### Monitoring & Observability

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://aws.amazon.com/cloudwatch/">
<img src="https://img.shields.io/badge/Amazon%20CloudWatch-FF4F8B?style=for-the-badge&logo=amazon-aws&logoColor=white" height="28">
</a>

<a href="https://prometheus.io/">
<img src="https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white" height="28">
</a>

<a href="https://grafana.com/">
<img src="https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white" height="28">
</a>

</div>

<br>

---

## Open Source & Platform Contributions

**[pdf-encoding-repair](https://github.com/sadishihab/pdf-encoding-repair)** — *new open-source library, published on [PyPI](https://pypi.org/project/pdf-encoding-repair/)*

Some PDFs extract as text that looks like text but isn't: a broken font encoding shifts every character by a constant offset, and search, RAG and extraction pipelines downstream quietly ingest the garbage. Nothing errors, so nobody notices.

I hit this on two of the seven manufacturer manuals behind FixIt, each with a different offset. Rather than patch it inside one project, I extracted the detection and repair into a standalone MIT-licensed library, with guards against false positives so clean PDFs are left untouched, CI, and a tagged release. Anyone ingesting PDFs can now reuse the fix instead of rediscovering it.

<br>

**[fixit-mcp](https://github.com/sadishihab/fixit-mcp)** — *open to contributors, with a friction log for the platform teams*

FixIt is MIT licensed and set up for outside contributors: "good first issue" tickets, issue and PR templates, a security policy, an architecture guide, and a guide for appliance manufacturers who want their manuals supported. A quickstart runs the whole thing without an AWS account.

Alongside it is a friction log of 60+ real entries for the Amazon developer teams, each with what was attempted, what happened, a severity, the workaround, and a suggestion. It covers IAM permissions that `CreateAgentRuntime` needs but the docs don't list, a container guide that doesn't fit MCP servers, an undocumented Alexa+ session model, and the discovery that Claude models on Bedrock are billed through AWS Marketplace, outside promotional credits.

<br>

**[anna-developer-docs](https://github.com/Anna-Partners/anna-developer-docs)** — *Documentation corrections, merged*

While building Error Journal on Anna's platform, I lost a day to behaviour that contradicted the documentation. Rather than work around it, I traced each discrepancy through the runtime source and wrote up seven findings with replacement text.

All seven were verified as accurate. Six were merged into the public developer documentation, including a capability string that no longer existed in the runtime, a required manifest field missing from the reference table, and a config schema documented with the wrong data type.

The seventh turned out to be a production bug rather than a docs error — a storage-token issue that took the platform team a proper investigation to root cause, traced to a resource silently resetting its visibility when edited through the web UI. Confirmed and fixed in the following release.

> *"One of the best community write-ups we've received — seven precise findings, each verified against actual runtime behavior. We verified all seven items and every single one was accurate."* — platform engineering team

**Why it's here:** most of these were found by reading the runtime source rather than re-reading the docs. That habit — verifying behaviour instead of trusting documentation — is the same one that keeps production systems debuggable.

<br>

**[claim-intake-agent](https://github.com/sadishihab/claim-intake-agent)** — *findings published by AssemblyAI*

The same habit, on a different platform. Building the voice agent surfaced three places where the published message-sequence and browser-integration docs disagreed with the machine-readable API schema — a field name for reply audio, a field name for agent transcripts, and the identifier on tool results. Each was verified against the live API rather than assumed, reported, and published as documentation corrections.

A fourth finding was a model behaviour rather than a docs error: biasing transcription toward a list of known values can rewrite a mis-heard value into one of them, which is dangerous anywhere the transcript is the evidence being validated. That one was escalated to their research team.

<br>

**[anna-app-template](https://github.com/sadishihab/anna-app-template)** — *reusable scaffold, published for other builders*

After shipping Error Journal on the same platform, I extracted the parts worth reusing so the next person doesn't repeat the discovery. JSON-RPC transport with a forward queue for concurrent reverse-RPC calls, persistent storage and model sampling that degrade gracefully when unavailable, three-platform binary CI, and a publish runbook documenting the failure mode at each step.

Clone, run the rename script, get a running plugin — verified from a clean clone rather than assumed.

<br>

---

## Other Work

A selection of supporting projects across AI research engineering, cloud infrastructure, DevOps automation, and software engineering.

**[Counterexample — PR Verification on IBM Bob 2.0](https://github.com/sadishihab/counterexample)** — *evidence over opinions, not tests that pass*
A pull-request verification tool for AI-generated code, where green CI is a weak signal because the same reasoning that wrote a bug often wrote the tests around it. It runs two independent checks and merges them into one Review Receipt: diff-scoped mutation testing (an AST mutator that injects bugs only into the lines a PR changed, then runs the PR's own tests against each mutant), and claim falsification, where IBM Bob extracts the concrete claims a PR makes from its diff and linked issue, spawns one subagent per claim, and has each write and run an adversarial test to break it. A GitHub Action posts the mutation layer's result on every pull request.

Three findings worth the click. **On a real test PR, mutation testing scored a reproducible 100% while the PR was still wrong** — the bug computes a discount from the wrong variable, which none of the four mutation operator families can express. Claim falsification caught it: 4 of 5 claims falsified, each with a failing test and real output, including a silent breaking change for existing callers that was never planted. **The tool had a bug of its own** — in Python's pathlib, joining a temp directory with an absolute path silently discards the temp directory, so concurrent mutant writes landed on the real repo instead of an isolated copy. It surfaced as unexplained working-tree corruption and is now guarded by a loud error. And **CI reported 0% for the wrong reason**: pytest was collecting unrelated files from GitHub's merge-commit checkout, so every mutant errored identically. The receipt now shows per-mutant error detail, and the PR comment states plainly that it covers mutation testing, not requirement-level verification.

On the Bob side: a custom mode with scoped tool permissions, two custom skills, and five parallel subagents in isolated contexts. The first full review cost 1.71 Bobcoins. [See it on a real pull request](https://github.com/sadishihab/counterexample-demo-checkout/pull/1).

**Tech:** Python 3.11 · IBM Bob 2.0 (custom modes, skills, subagents) · AST mutation testing · pytest · GitHub Actions

**[Bimanual VLA — Table Setting in Simulation](https://github.com/sadishihab/bimanual-vla)** — *measurement over assumption*
Two simulated SO-101 arms set a table in MuJoCo: a scripted expert picks four props from a randomized layout, hands a prop between arms when no single arm can reach both the prop and its slot, records the successes as a LeRobot v3.0 dataset, trains an ACT policy on it, and converts the checkpoint to OpenVINO IR for Intel inference hardware.

The engineering interest isn't the robotics — it's the discipline. Every design decision traces to a measurement, and the README opens with a table of what was measured and what explicitly was not.

Three findings worth the click. **FP16 quantization looked healthy on mean error while 14.94% of gripper commands flipped sign** — close became open, which drops whatever the arm is holding; INT8 flips 0.92% and is 3.46× smaller. **The original parity check passed and was wrong**, because it ran against synthetic noise where a badly wrong precision looks exact; on real frames the same model was off by three orders of magnitude more. And the underperforming policy was handled as a **controlled experiment rather than a result to bury**: three causes diagnosed, one isolated by removing an image-task confound from the training data, attention measurably redirected (non-plate target contact 0/30 → 7/30) while task competence stayed flat, exactly as the two untouched causes predict.

**Tech:** Python 3.11 · MuJoCo · LeRobot 0.4.4 (ACT, 51.6M params) · PyTorch · OpenVINO + NNCF · MiniLM-L6

**[Single-Node Kubernetes Cluster](https://github.com/sadishihab/Single-Node-Kubernetes-Cluster)**
Multi-service web app (React, Node.js, MongoDB) deployed on a single-node Kubernetes cluster using Minikube — Deployments, Services, Ingress, ConfigMaps, Secrets, PV/PVC.
**Tech:** Kubernetes · Docker · Minikube · NGINX Ingress

**[Kubernetes on AWS (EKS)](https://github.com/sadishihab/eks)**
End-to-end CI/CD on AWS EKS — Fargate, eksctl, Jenkins, DockerHub, and ECR integrations.
**Tech:** AWS · EKS · Fargate · Jenkins · Docker

**[Terraform IaC](https://github.com/sadishihab/terraform)**
Infrastructure as Code patterns for repeatable, auditable cloud deployments.
**Tech:** Terraform · AWS

**[Prometheus + Grafana Monitoring](https://github.com/sadishihab/prometheus)**
Monitoring and observability setup for cloud-native applications.
**Tech:** Prometheus · Grafana

**[Ansible Automation](https://github.com/sadishihab/ansible)**
Configuration management and infrastructure automation playbooks.
**Tech:** Ansible · Playbooks

[See all repositories on GitHub](https://github.com/sadishihab?tab=repositories)

<br>

---

## Background

17+ years of software engineering across embedded systems, mobile, full-stack, cloud, and AI.

I started at Samsung R&D Bangladesh, where I worked on firmware for handsets shipped across Middle East, Africa, and Bangladesh — including the Bengali Calendar for the Bangladesh region and language support for Swahili, Yoruba, Igbo, Hausa, and Amharic on Samsung feature phones used by millions. That's where multilingual production software became muscle memory, which weirdly turned out to be great prep for the multilingual RAG work I do now.

After Samsung, I co-founded **Training Pool**, Bangladesh's first online training marketplace and SaaS platform. Took it from idea to live product with paying users. Before that, I ran a small dev studio building Android multiplayer games and Bangladesh client projects.

These days I run **Vector Forge**, shipping production RAG applications, AI agents, MCP servers, and voice agents for founders, agencies, and mid-market teams.

[See full work history on LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi/)

<br>

---

## Blog

[Read my latest posts](/blog/) · [RSS feed](https://sadishihab.github.io/feed.xml)

<br>

---

## Let's Talk

If your chatbot is hallucinating, your voice agent is writing down things nobody said, your AI feature isn't making it past the demo stage, or you want to add a real RAG system to your product without it embarrassing you in front of customers — let's talk.

Want your product or your documentation inside AI assistants, through an MCP server that answers from the source and says so when it doesn't know? Same conversation.

Running an agency with AI work sold and no one to build the reliable version of it? That conversation is even shorter.

<div style="display:flex; flex-wrap:wrap; gap:12px; margin: 20px 0 30px 0;">

<a href="https://calendly.com/sadi-shihab/30min" style="background:#0a66c2; color:white; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600;">
📅 Book a 30-min call
</a>

<a href="mailto:sadi.shihab@gmail.com" style="background:#f3f2ef; color:#0a66c2; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600; border:1px solid #0a66c2;">
✉️ Email me
</a>

</div>

- **LinkedIn:** [linkedin.com/in/md-shihabuddin-sadi](https://www.linkedin.com/in/md-shihabuddin-sadi/)
- **GitHub:** [github.com/sadishihab](https://github.com/sadishihab)
- **Email:** sadi.shihab@gmail.com
