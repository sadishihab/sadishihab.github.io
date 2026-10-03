---
title: "Building a Voice Agent That Refuses to Guess: Claim Intake on the AssemblyAI Voice Agent API"
date: 2026-10-03
layout: post
permalink: /claim-intake-agent/
categories: [AI, Voice AI, Python, AssemblyAI]
tags: [Voice Agents, AssemblyAI, Speech Recognition, FastAPI, WebSockets, AudioWorklet, Validation, Docker, NGINX, LLM]
description: "How I built a voice agent for insurance claim intake where nothing reaches the record unless server-side code approves it, and how biasing the speech recogniser toward real policy numbers quietly made my own validator useless."
author: "Md. Shihabuddin Sadi"
---

*Software Engineer · DevOps & Cloud Native Engineer · AI / RAG & Voice Agent Developer*  
*October 03, 2026*  

<br>

### **Summary**

Speech recognition gets things wrong. Everyone building voice agents knows that. What worries me is what happens next. A typical agent hears "KD4-1188", writes it down and moves on. If it heard wrong, nothing in the system notices. The claim sits on the wrong policy until somebody gets denied months later and nobody can explain why.

I built a voice agent for insurance claim intake around one rule: **the agent proposes, code decides.** The agent listens to the caller and calls a `record_field` tool with what it thinks it heard. Plain Python checks that value against the real policy data and returns one of three verdicts (accepted, unconfirmed or rejected) together with the exact sentence the agent has to say next. Nothing reaches the claim record without passing that check.

The most useful thing I learned came from breaking it. To improve accuracy I biased the speech recogniser toward the list of valid policy numbers. Accuracy went up, and the recogniser started rewriting misheard numbers into real ones that belonged to other customers. Those passed validation perfectly. This post covers how the system works, how that happened, and what I changed.

Built solo in September 2026 for the [AssemblyAI Voice Agent Hackathon](https://lablab.ai/ai-hackathons/assemblyai-voice-agent-hackathon) on lablab.ai.  
<br>
🔗 **Live demo:** [claims.sadishihab.com](https://claims.sadishihab.com) (use policy `KD4-1188`, Denise Holloway)  
🔗 **What validation catches:** [claims.sadishihab.com/compare](https://claims.sadishihab.com/compare)  
🔗 **GitHub Repository:** [sadishihab/claim-intake-agent](https://github.com/sadishihab/claim-intake-agent)

<br>

### Table of Contents

- [Summary](#summary)
- [Key Technologies Used](#key-technologies-used)
- [1. The Problem: A Wrong Value Looks Like a Right One](#1-the-problem-a-wrong-value-looks-like-a-right-one)
- [Architecture Diagram](#architecture-diagram)
- [2. Folder Structure](#2-folder-structure)
- [3. Three Verdicts, Decided by Code](#3-three-verdicts-decided-by-code)
- [4. Field by Field: Where the Thresholds Came From](#4-field-by-field-where-the-thresholds-came-from)
- [5. The Day the Validator Became Useless](#5-the-day-the-validator-became-useless)
- [6. Three Attempts to Fix It Upstream, All Null](#6-three-attempts-to-fix-it-upstream-all-null)
- [7. Consent Is Bound to the Question](#7-consent-is-bound-to-the-question)
- [8. The Browser Was Transcribing Itself](#8-the-browser-was-transcribing-itself)
- [9. Reading the Wire Instead of the Docs](#9-reading-the-wire-instead-of-the-docs)
- [10. Evidence, Privacy and Deployment](#10-evidence-privacy-and-deployment)
- [11. How I Worked](#11-how-i-worked)
- [12. Learning Outcomes](#12-learning-outcomes)
- [References](#references)
- [License](#license)
<br>

### **Key Technologies Used**

`AssemblyAI Voice Agent API` · `Universal-3.5 Pro` · `Python 3.14` · `FastAPI` · `Uvicorn` · `WebSockets` · `AudioWorklet` · `Server-Sent Events` · `pytest` · `Docker` · `NGINX` · `Let's Encrypt` · `DigitalOcean`  
<br>

### **1. The Problem: A Wrong Value Looks Like a Right One**

Claim intake is a form filled in by speech recognition. The fields are ordinary: policy number, claimant name, date of loss, callback phone, type of loss and a free-text description. Some of those values are unforgiving, though. A policy number that is one character off still looks like a policy number, and if it happens to match a different customer's policy, the claim lands on their file.

The obvious fix is to make recognition better. I spent a fair part of the month trying, and section 6 explains why it did not work. The approach that does work is to assume the transcript can be wrong and build the system around that assumption.

So the design splits the job in two:

- **The agent** (AssemblyAI's Voice Agent API) runs the conversation. It hears the caller, decides when they have finished speaking, chooses what to say and says it.
- **The validator** (plain Python with no model inside it) owns the record. It decides what is allowed in, and it writes the sentence the agent must speak.

The agent never gets to decide that a value is correct. It can only propose one.  
<br>

#### **Architecture Diagram**
```pgsql
 ┌─ Caller's browser ─────────────────────────────────────────────────┐
 │  mic ──► AudioWorklet ──► PCM16 at 24 kHz, 50 ms chunks            │
 │  speaker ◄── one AudioContext at the device's own rate             │
 │  review panel ◄── server-sent events from the evidence log         │
 └─────────────────────────────────┬──────────────────────────────────┘
                                   │  wss://claims.sadishihab.com/ws/call
 ┌─────────────────────────────────▼──────────────────────────────────┐
 │  NGINX: TLS (Let's Encrypt) · Upgrade headers · 1 h timeouts       │
 └─────────────────────────────────┬──────────────────────────────────┘
                                   │  ws://127.0.0.1:8002  (loopback only)
 ┌─────────────────────────────────▼──────────────────────────────────┐
 │  FastAPI relay  (Docker, non-root)                                 │
 │                                                                    │
 │  caller audio ────────────────────────►  Voice Agent API           │
 │  reply.audio  ◄────────────────────────  STT · LLM · TTS           │
 │                                                                    │
 │  tool.call record_field(field, value) ◄── agent proposes           │
 │       │                                                            │
 │       ▼                                                            │
 │  ClaimRecord ──► validators.py ──► policies.json                   │
 │       │   accepted · unconfirmed · rejected + readback             │
 │       ▼                                                            │
 │  tool.result after reply.done ────────►  agent speaks it           │
 │       │                                                            │
 │       ▼                                                            │
 │  calls/<session>.jsonl   append-only, deleted after 24 h           │
 └────────────────────────────────────────────────────────────────────┘
```
<br>

### **2. Folder Structure**
```bash
├── protocol.py           # Session config, record_field tool, tool.result ordering
├── intake.py             # ClaimRecord: per-call state, guards, evidence log, retention
├── validators.py         # One pure function per field, each returns a Verdict
├── policies.json         # Fake policy data, including confusable twins
├── review.py             # FastAPI: browser relay, review panel, /compare, SSE
├── review.html           # Call button and live review panel
├── compare.html          # Naive vs validated, computed when the page loads
├── comparison.json       # Verbatim transcripts taken from real calls
├── pcm-processor.js      # AudioWorklet: microphone capture and resampling
├── agent.py · audio.py   # Terminal client used during development
├── seed_demo.py          # Demo data for rehearsing the recording
├── deck.py               # Builds the submission slide deck
├── test_*.py             # 206 tests
├── Dockerfile            # python:3.14-slim, runs as a non-root user
└── CLAUDE.md             # Hard-won rules for future coding sessions
```
<br>

### **3. Three Verdicts, Decided by Code**

Every value goes through one tool. The Voice Agent API expects a flat tool schema, without the nested `function` wrapper OpenAI uses:

```json
{
  "type": "function",
  "name": "record_field",
  "description": "Record one piece of claim information the caller has given you. Call this every time the caller provides a value. Do not write anything down without calling this first.",
  "parameters": {
    "type": "object",
    "properties": {
      "field": {
        "type": "string",
        "enum": ["policy_number", "claimant_name", "date_of_loss",
                 "callback_phone", "loss_type", "description"]
      },
      "value":     { "type": "string" },
      "confirmed": { "type": "boolean" }
    },
    "required": ["field", "value"]
  }
}
```

Each validator returns the same shape:

```python
# validators.py (simplified)
@dataclass(frozen=True)
class Verdict:
    status: str            # "accepted" | "unconfirmed" | "rejected"
    value: str | None      # normalised value, None if nothing may be written
    reason: str            # internal diagnostic, never spoken
    readback: str          # the exact sentence the agent must say next
    confirmable: bool = False
```

| Verdict | What gets written | What the agent does |
|---|---|---|
| **Accepted** | The normalised value | Moves to the next field without reading anything back |
| **Unconfirmed** | Nothing | Reads the value back, letters in the NATO alphabet, and waits |
| **Rejected** | Nothing | Explains what is wrong and asks again |

The readback is written by the validator and the agent speaks it word for word. Left to itself, the model reads a value back in its own way, something like "B X 7 4 4 0 2", which is close to useless over a bad line. The validator says "Bravo X-ray seven, four four zero two", and the confusable twin `BX7-4420` comes out as "four four two zero". Two real policies belonging to two different people, and two clearly different sentences.

The `reason` field is deliberately never spoken. My first spec told the agent to explain rejections using it, and a rejected name produces text like `not close to 'Marcus Halloway' (ratio 0.60)`. Read aloud, that tells someone who just failed to match a policy the real holder's name. The readback for that case asks "Are you calling about someone else's policy?" and gives nothing away.  
<br>

### **4. Field by Field: Where the Thresholds Came From**

Each field has its own pure function, and most of them needed a decision that I wanted evidence for.

**Policy number.** Two letters, a digit, a hyphen, four digits, with no fuzzy matching at all. A near miss is rejected and never offered as "did you mean". A "did you mean" path is exactly how `BX7-4402` turns into `BX7-4420`, and there is a test whose only job is to fail if someone adds one later. The normaliser accepts what callers actually say: "BX74402", "B X 7 4 4 0 2", and later "Bravo X-ray seven four four zero two".

**Claimant name.** Compared to the policy holder with a similarity ratio. Before picking a threshold I measured realistic mishearings against genuinely different people:

```
mishearings of the holder     0.857 to 0.966
different real people         0.00  to 0.60
threshold                     0.80  (an empty gap on both sides)
```

The hardest case is in the database on purpose. Denise Holloway is a real holder on another policy, and against Marcus Halloway she scores 0.60, so she is rejected. "Dennis Holloway" against Denise Holloway scores 0.93, lands in the unconfirmed band, and the agent asks the caller to spell the surname.

**Date of loss.** Strict `YYYY-MM-DD`, converted by the agent before it calls the tool. A date with no year is rejected and the readback asks for the year, because filling in the current year would silently decide whether the claim falls inside the policy period. While testing this I found that Python's `date.fromisoformat` (3.11 and later) also accepts ISO week dates, so `2026-W22-1` parsed cleanly as 25 May 2026, a date nobody said. The validator now checks a strict pattern before parsing. A date up to seven days outside the policy period is unconfirmed rather than rejected, since one day out is more likely a memory slip than fraud. Future dates are always rejected and can never be confirmed.

**Callback phone.** Ten digits are accepted. Seven digits, with no area code, come back unconfirmed.

**Loss type.** Free speech is mapped to collision, theft, fire, water, glass, vandalism or other. When more than one category matches, the verdict is unconfirmed and the agent offers the choice. "Someone hit my car on purpose" matches both collision and vandalism, and the difference between an accident and a deliberate act changes how a claim is handled.

**Description.** Free text, always accepted.  
<br>

### **5. The Day the Validator Became Useless**

Spelled-out alphanumerics were the weak point. On my laptop's old microphone, "KD4-1188" came through as `3841188`, `MACKKDK41138` and `D411`. The validator rejected every one of them, which was correct, but a caller who cannot get past the first field does not have a product.

AssemblyAI lets you pass `keyterms`, a list of words and phrases to bias recognition toward. I generated the list from `policies.json` and ran a synthetic test on four policy numbers, spoken naturally, spelled out and in the NATO alphabet. Accuracy went up. I also tested the obvious danger, near-miss numbers that are not on file, to check that the bias would not pull them onto real policies. None of them snapped, so I adopted it.

Then I placed a real call and deliberately read a wrong policy number. The event log keeps the partial transcripts, and this is what it shows:

```
partial   "Yes."
partial   "Yes, my policy number is"
partial   "Yes, my policy number is C411."
final     "Yes, my policy number is KD4-1188."
TOOL      policy_number = "KD4-1188"  ->  accepted  (exact match on file)
```

`KD4-1188` is real. It belongs to Denise Holloway. The recogniser was unsure, reached for the closest entry in the list I had given it, and produced a number the caller never said. My validator compared it with the database, found an exact match, and accepted it.

(Audio is not stored, so the part about reading a wrong number is my recollection. The revision from `C411` to `KD4-1188` is in the log.)

Nothing in the validator was broken. It did exactly what it was written to do. Its power rested on an assumption I had never written down: that the transcript is an **independent observation** of what the caller said. An exact match against the database only means something if the recogniser has no idea what is in the database. Once I handed it the answer key, "this matches a real policy" stopped being evidence. Every guard downstream stayed intact and stopped working.

My synthetic test missed it because clean text-to-speech gives the recogniser no reason to reach for a prior, and a human voice on a bad microphone gives it every reason. I had noted that the test audio was unrepresentative and adopted the change anyway. That was the real mistake.

Two changes followed:

1. **Keyterms were removed.** A test asserts that the session config never sends them, with a docstring explaining why, so nobody improves accuracy back into the bug.
2. **Policy numbers always need spoken confirmation**, whatever the match quality. An exact match now comes back unconfirmed, the agent reads the number back in NATO alphabet, and only the caller's yes promotes it. The `confirmed` flag is bound to the exact value that was read back, and setting it on a first call does nothing, so the model cannot skip the question to save a turn.

The rule now lives in the project's `CLAUDE.md`: never bias the recogniser toward the answer key when the recogniser's output is the evidence being validated.  
<br>

### **6. Three Attempts to Fix It Upstream, All Null**

With keyterms gone, the accuracy problem was back. I tried three other levers and measured each one before changing anything:

| Lever | What I measured | Result |
|---|---|---|
| `transcription_mode: max_accuracy` | Recovery of four policy numbers in three speaking styles | Baseline was already 12 of 12 on clean audio, and stayed perfect with noise added down to 5 dB SNR. Nothing to improve on that stimulus |
| Turn-taking patience | Characters spelled with 800 ms and 1200 ms pauses, counting how many turns one number split into | Splitting fell from 2.00 to 1.50 turns per number at 800 ms. Correct recovery did not change |
| Format hints in the tool schema | The same spelled stimulus, plus near-miss numbers | No change in splitting or recovery. Zero near-misses snapped onto real policies, so it was at least harmless |

What settled it was `KD4-1188`. It failed in every configuration, the same way each time:

```
no tools          'K.'  '4-1-1-8-8.'
tools, plain      'K.'  '4. 1.'  '8. 8.'
tools + hints     'K.'  '4-1.'   '1-8-8.'
```

Three different ways of splitting the turns, and the same missing D. In one run the recogniser produced a single, perfectly patient turn and still dropped it. That is an acoustic confusion between "kay" and "dee", and no setting in the config reaches it. None of the three went into the code. Shipping a change that measures at zero is how the keyterms problem happened, just from the other direction.

A caveat on these numbers: the clips were synthesised speech and the harness was a scratch script that is not in the repository, so treat them as indicative.

The negative result ended up being the strongest argument for the architecture. If the recogniser cannot be configured out of a failure, something has to catch the failure afterwards. What actually rescues the caller lives in the validator. After a second rejection the agent switches to asking for the letters as words, and its examples are generated from letters that appear in **no** policy on file, currently Zulu and Quebec. My first version said "Bravo for B, Kilo for K". Those are the first letters of two real policies, and a caller repeating the example back could be transcribed as input.  
<br>

### **7. Consent Is Bound to the Question**

Once confirmation existed, the interesting bugs moved into how the guards interact. Before submitting I ran an automated cloud code review over the whole project. It found two real bugs in the same seam, both introduced by earlier fixes of mine.

**Changing an accepted policy number skipped the readback.** `BX7-4402` was accepted for Marcus Halloway. The caller then gave a different number, the agent asked "I already have an answer for that one. Did you want to change it?", the caller said yes, and the record flipped to `KD4-1188`, Denise Holloway's policy. No NATO readback was ever spoken for the new number. The active policy changed with it, so the name and date fields were then checked against the wrong holder.

**A near-miss date overwrote an accepted one.** The caller had agreed that a date just outside the policy period was still their answer. They had never agreed to replace the date already on file.

Both had the same cause. The guard protecting accepted values ran before confirmation was applied, and the confirmation check could not tell which question the caller had said yes to. The fix records what kind of question each hold asked, and moves the guard to the end so it sees the status the value would really be written under:

```python
# intake.py (simplified)
def record(self, field, value, confirmed=False):
    verdict = VALIDATORS[field](value, policy=self.policy, today=self.today)
    verdict = self._promote(field, verdict, confirmed)   # only the exact value read back
    verdict = self._refuse_repeat(field, verdict)        # same unanswered value twice is a loop
    verdict = self._guard_recorded(field, verdict)       # runs last: a change needs its own yes
    self._append_attempt(field, value, verdict)
    if verdict.status == "accepted":
        self.fields[field] = verdict.value
    return verdict
```

Changing a recorded value now takes two separate yeses, one to the NATO readback of the new value and one to replacing what is on file. A hold that asks the caller to choose, like collision or vandalism, cannot be confirmed away at all, because a yes does not answer a choice.

The review also found `assert ... or True` in my test suite. It could never fail, and the condition it pretended to check was false. A test that asserts nothing is worse than no test, because it makes every other test less believable. Since then the important tests are mutation-checked: remove the behaviour on purpose and confirm that a specific test fails.  
<br>

### **8. The Browser Was Transcribing Itself**

The first browser version worked and echoed badly. The agent kept hearing its own voice and transcribing it as the caller. In one call the two of them said "Goodbye" to each other five times.

Before guessing, I searched the logs for the fingerprint of self-capture: a `transcript.user` that is a near-verbatim copy of what the agent had just said.

| Client | User turns | Verbatim echoes |
|---|---|---|
| Terminal client (headphones) | 41 | 0 |
| Browser, before the fix | 20 | 14 |
| Browser, after the fix | 9 | 0 |

The browser reported `echoCancellation: true`, and the microphone was the real input device rather than a loopback monitor. So echo cancellation was enabled and not doing anything. The cause was a line I had written straight from an integration guide:

```js
const ctx = new AudioContext({ sampleRate: 24000 });
```

The Voice Agent API wants 24 kHz audio, so forcing the context to 24 kHz looks sensible. In Firefox, a context at a non-default sample rate gets its own media graph ([Bugzilla 1387454](https://bugzilla.mozilla.org/show_bug.cgi?id=1387454)), and the echo canceller only uses output from the default graph as its reference ([Bugzilla 1849108](https://bugzilla.mozilla.org/show_bug.cgi?id=1849108)). My device runs at 48 kHz, so the agent's voice was playing in a graph the canceller could not see.

The fix is a single context at the device's own rate for both directions, with conversion at the edges. On the way in, an AudioWorklet resamples and batches:

```js
// pcm-processor.js (simplified)
class PcmProcessor extends AudioWorkletProcessor {
  constructor() {
    super();
    this.ratio = sampleRate / 24000;   // 2 on a 48 kHz device
    this.out = new Int16Array(1200);   // 50 ms at 24 kHz
    this.n = 0;
    this.pos = 0;
  }
  process(inputs) {
    const ch = inputs[0][0];
    if (!ch) return true;
    for (; this.pos < ch.length; this.pos += this.ratio) {
      const s = Math.max(-1, Math.min(1, ch[Math.floor(this.pos)]));
      this.out[this.n++] = s * 0x7fff;
      if (this.n === this.out.length) {
        this.port.postMessage(this.out.slice().buffer);
        this.n = 0;
      }
    }
    this.pos -= ch.length;
    return true;
  }
}
registerProcessor("pcm-processor", PcmProcessor);
```

On the way out, each reply chunk goes into a 24 kHz buffer that the context resamples on playback, scheduled on the audio clock rather than with timers. The batching matters as well. At 48 kHz the worklet runs every 2.7 ms, and without collecting into 50 ms chunks the upstream message rate would have doubled to around 375 per second. After the fix, full calls on laptop speakers with no headphones came through clean. A regression test fails if anyone puts the forced sample rate back, and its docstring links both Bugzilla entries.  
<br>

### **9. Reading the Wire Instead of the Docs**

I wrote the WebSocket protocol by hand instead of using an SDK, and logged every inbound event to a JSONL file from the first day. That log settled several arguments with the documentation. At the time, the message-sequence page disagreed with the machine-readable AsyncAPI schema and the live API in three places:

| Message | The page said | The schema and live API use |
|---|---|---|
| `reply.audio` | `audio` | `data` |
| `transcript.agent` | `transcript` | `text` |
| `tool.result` | `tool_call_id` | `call_id` |

I reported those along with the sample-rate issue in the browser guide, and AssemblyAI's product team published updates to both pages. The keyterms finding went to their research team.

Tool results need care beyond the field names:

```python
# protocol.py (simplified)
await ws.send(json.dumps({
    "type": "tool.result",
    "call_id": call_id,                 # not tool_call_id
    "result": json.dumps(verdict),      # a JSON string, not a nested object
}))
```

A result has to go out **after** the agent's current reply finishes (`reply.done`), and has to be dropped if that reply was interrupted by the caller. The simple rule misses one case: a tool can finish after `reply.done` has already fired, and a result that is only ever flushed on `reply.done` would then wait forever. So the queue also tries to flush from the `tool.call` handler when no reply is in flight. Scripted tests against a fake server cover the ordering, including the interrupted case.

A few smaller things that are easy to get wrong. The Voice Agent API uses `Authorization: Bearer <key>`, while AssemblyAI's other APIs take the raw key. Closing the socket without sending `session.end` keeps the session billable for a 30-second resume window. And Ctrl+C in the terminal client needed an explicit signal handler, because from Python 3.12 `asyncio.run` cancels the main task instead of raising `KeyboardInterrupt` inside it, which would have skipped `session.end` on every hangup.  
<br>

### **10. Evidence, Privacy and Deployment**

**Every attempt is evidence.** Each `record_field` call appends one line to `calls/<session>.jsonl` as it happens: field, raw value, normalised value, verdict, reason, readback, seconds since the call started, and the caller's actual words. JSON Lines means a crash costs one line instead of the whole file. I tested that by killing the process with `kill -9` in the middle of a call. A clean hangup also writes a summary JSON.

**The review panel** tails that log over server-sent events, so the claim builds up on screen while you talk. A reviewer can see that a policy number was rejected twice before it stuck, and what the caller said each time. Rejected and unconfirmed attempts stay visible after a field is accepted, because the history is the point.

**`/compare`** renders real transcripts from the logs twice, once as a system that trusts the transcript and once through the real validators. The validators run when the page loads, so the comparison cannot drift away from the code. Five scenes come from live calls and one from seeded demo data, and the page says which.

**Privacy.** The app is public, so the panel is scoped to the caller's own session. There is no list of other calls and no global stream, the interactive API docs are switched off, and concurrent calls are capped so nobody can run up the bill on my key. The logs hold names and phone numbers, so a sweep on startup and every hour deletes any call more than 24 hours past its last write. It goes by modification time, so a long call can never be deleted while it is still being written.

**Deployment** is a Docker container on a DigitalOcean droplet, running as a non-root user and published only on loopback. NGINX sits in front and handles TLS with Let's Encrypt. WebSockets need two headers spelled out, or the page loads fine and calls silently never start:

```nginx
location / {
    proxy_pass http://127.0.0.1:8002;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;   # the 60s default hangs up on a caller looking for their policy
    proxy_send_timeout 3600s;
}
```
<br>

### **11. How I Worked**

I built this solo with AI assistance, using the same three surfaces I use on other projects:

1. **A planning chat** for design, review, and breaking the work into small single-task steps
2. **Claude Code in the repo** for edits, tests and commits, bound by a `CLAUDE.md` file of hard-won rules such as "reply.audio carries audio in data" and "never bias the recogniser toward the answer key"
3. **A plain shell** for checking what the assistant claims, and for placing real calls

The habit that paid off most was not trusting synthetic data on its own. Every important finding in this post came from a real call: the echo, the keyterms snap, the loss-type question that got stuck in a loop, and the agent reading back an accepted phone number three times. Synthetic tests were good at proving that a fix stayed fixed. They were bad at finding the problems in the first place.

The same applied to writing about it. Three times I caught my own write-up claiming more than the logs supported. One example: the comparison page originally said every value the naive system recorded was wrong, when one of them was actually right. Each was corrected before submission.  
<br>

### **12. Learning Outcomes**

- Built a voice agent where **the model proposes and code decides**, so a misheard value cannot reach the record silently  
- Designed a **three-verdict validator** whose readbacks are generated by code and spoken verbatim  
- Found that **biasing a recogniser toward the answer key** destroys the independence a validator depends on, and removed it  
- Chose thresholds from **measured distributions** instead of round numbers  
- Ran **three recogniser-tuning experiments**, accepted the null results, and shipped none of them  
- Learned that **consent must be bound to the question asked**, and fixed two guard-ordering bugs where it was not  
- Traced a **browser echo bug** to Firefox's per-sample-rate media graphs and fixed it with one AudioContext and worklet resampling  
- Read the **wire protocol** directly, found documentation errors, and got them corrected upstream  
- Built a **crash-safe evidence trail** with 24-hour retention, and a public deployment that cannot show one caller another caller's data  
- Kept **206 tests**, several of which pin design decisions so that a future change undoing one fails with an explanation  

<br>

### **Try It**

Open [claims.sadishihab.com](https://claims.sadishihab.com), click **Start call** and allow the microphone. Use policy `KD4-1188` and the name Denise Holloway, and agree when the agent reads the number back. Then try to break it: give a wrong policy number, say a name one letter off, or give the 2nd of June 2025 as the date of loss, one day after that policy's cover ended.

If you are building a voice or AI agent that writes to a real system of record and want this kind of reliability in your own product, I am happy to talk: [book a 30 minute call](https://calendly.com/sadi-shihab/30min) or find me on [LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi).

<br>

### **References**
- [Claim Intake Agent on GitHub](https://github.com/sadishihab/claim-intake-agent)
- [AssemblyAI Voice Agent API documentation](https://www.assemblyai.com/docs/voice-agents/voice-agent-api)
- [AssemblyAI Voice Agent Hackathon on lablab.ai](https://lablab.ai/ai-hackathons/assemblyai-voice-agent-hackathon)
- [Mozilla Bugzilla 1387454: one media graph per sample rate](https://bugzilla.mozilla.org/show_bug.cgi?id=1387454)
- [Mozilla Bugzilla 1849108: echo cancellation reference from non-default sinks](https://bugzilla.mozilla.org/show_bug.cgi?id=1849108)
- [MDN: AudioWorklet](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorklet)
- [MDN: Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
<br>

### **License**
This project is open-source and available under the MIT License.
