---
title: "Building Error Journal: An Error Diagnosis App That Remembers What Fixed It Last Time"
date: 2026-10-03
layout: post
permalink: /error-journal/
categories: [AI, Python, DevOps, Anna]
tags: [AI Agents, Python, Kubernetes, Docker, JSON-RPC, Fingerprinting, Debugging, Developer Tools, Anna, LLM]
description: "How I built Error Journal, a deterministic error diagnosis app for the Anna AI platform. Paste any error, get a verified fix, and if you hit the same problem again on a different machine, it tells you what fixed it last time."
author: "Md. Shihabuddin Sadi"
---

*Software Engineer · DevOps & Cloud Native Engineer · AI / RAG Application Developer*  
*October 03, 2026*  

<br>

### **Summary**

Every developer knows this loop. Something breaks, you dig through logs, you find the fix, you move on. A few weeks later the exact same thing breaks again, on a different machine, with a different pod name, and you start from zero, because your past self never wrote anything down.

**Error Journal** writes it down for you. It is an app on the [Anna](https://anna.partners) AI platform: you paste any error (a Python traceback, a Kubernetes crash loop, a failed Docker build) and get the root cause, ordered fix steps, and a command to verify the fix. The interesting part is what happens the second time. The app recognises the same underlying problem even when the log looks completely different, and says:

> This is the 3rd time you have hit this in payments-api, first seen 12 August. What fixed it last time: raised the memory limit to 512Mi.

Under the hood it is **deterministic error fingerprinting**, a **109-entry curated diagnosis base**, a **per-user incident journal** in persistent storage, and a **JSON-RPC plugin** written in pure Python standard library. Building it on a brand new platform also meant finding and reporting real platform bugs, one of which turned into a new platform feature built from my proposal.  
<br>
🔗 **Live app:** [anna.partners/store/@sadi/error-journal](https://anna.partners/store/@sadi/error-journal)  
🔗 **GitHub Repository:** [sadishihab/error-journal](https://github.com/sadishihab/error-journal)

<br>

### Table of Contents

- [Summary](#summary)
- [Key Technologies Used](#key-technologies-used)
- [1. The Problem: AI Answers Are Stateless](#1-the-problem-ai-answers-are-stateless)
- [Architecture Diagram](#architecture-diagram)
- [2. Folder Structure](#2-folder-structure)
- [3. Deterministic Fingerprinting](#3-deterministic-fingerprinting)
- [4. Three Answer Tiers: Never Fabricate](#4-three-answer-tiers-never-fabricate)
- [5. Real Values in Fix Steps, Safely](#5-real-values-in-fix-steps-safely)
- [6. A Plugin That Talks Back](#6-a-plugin-that-talks-back)
- [7. Bugs That Only Real Input Could Find](#7-bugs-that-only-real-input-could-find)
- [8. Building on a Brand New Platform](#8-building-on-a-brand-new-platform)
- [9. How I Worked: Three Surfaces](#9-how-i-worked-three-surfaces)
- [10. Learning Outcomes](#10-learning-outcomes)
- [References](#references)
- [License](#license)
<br>

### **Key Technologies Used**

`Python 3 (stdlib only)` · `JSON-RPC 2.0 over stdio` · `SHA-256` · `Regex normalisation` · `Anna Persistent Storage` · `LLM sampling` · `PyInstaller` · `GitHub Actions` · `Vanilla JS + CSS` · `Anna App CLI`  
<br>

### **1. The Problem: AI Answers Are Stateless**

Any capable AI model can read a traceback and suggest a fix. That part is not special anymore. What none of them do is remember **your** errors in a way you can rely on.

Conversational memory does not solve this. It is fuzzy, it paraphrases, and it cannot tell you with certainty that today's `CrashLoopBackOff` is the same failure you fixed three weeks ago. To answer that reliably, you need the same input to produce the same identifier every time, no matter how noisy the log is.

So the design rule from day one was: **the model presents, the tool decides.** Anything that must be correct (the fingerprint, the history, the verified fix) is computed by deterministic code. The model's job is to talk to the user.

The app has three layers:

- **Fingerprinter** – normalises the raw log, classifies it across a dozen ecosystems, and hashes the stable remainder  
- **Knowledge base** – 109 hand-written diagnoses, each with a root cause, ordered fix steps, and a verify command  
- **Journal** – a per-user incident log in Anna Persistent Storage: occurrence count, first seen, contexts, and what fixed it  

The user sees it through three tabs: **Diagnose**, **Logbook**, and **Recurring** (errors that keep coming back). A small "Did this fix it?" control on each result lets them record what worked, so the next answer gets better.  
<br>

#### **Architecture Diagram**
```pgsql
   ┌─────────────────────── Anna Agent / App window ───────────────────────┐
   │   user pastes an error  ──►  diagnose_error(log, context)              │
   └───────────────────────────────────┬────────────────────────────────────┘
                                       │ JSON-RPC 2.0 over stdin/stdout
   ┌───────────────────────────────────▼────────────────────────────────────┐
   │                     error_journal_plugin.py                            │
   │                                                                        │
   │   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐   │
   │   │  fingerprint.py  │──►│   knowledge.py   │──►│  fill_placeholders│  │
   │   │ strip · classify │   │ 109 curated fixes│   │ allow-listed real │  │
   │   │ scope · SHA-256  │   │                  │   │ values in commands│  │
   │   └────────┬─────────┘   └────────┬─────────┘   └──────────────────┘   │
   │            │              (no curated match)                           │
   │            │                      ▼                                    │
   │            │          ┌──────────────────────┐                         │
   │            │          │ cached or sampled    │──► labelled "generated" │
   │            │          │ model diagnosis      │──► or honest "unknown"  │
   │            │          └──────────────────────┘                         │
   │            ▼                                                           │
   │   ┌──────────────────────────────────────────┐                         │
   │   │ journal(): reverse RPC to host storage   │                         │
   │   │ count · first_seen · contexts · fixes    │                         │
   │   └──────────────────────────────────────────┘                         │
   └────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
        "This is the 3rd time you have hit this ... What fixed it last time: ..."
```
<br>

### **2. Folder Structure**
```bash
├── executas/error-journal/
│   ├── fingerprint.py            # Pure stdlib: normalise, classify, hash
│   ├── knowledge.py              # 109 curated diagnoses
│   ├── error_journal_plugin.py   # JSON-RPC server (the actual plugin process)
│   ├── test_*.py                 # Independent test suites
│   ├── stress_fingerprint.py     # Corpus of real-world messy logs
│   └── executa.json              # Identity, version, binary build targets
├── bundle/                       # App UI: Diagnose, Logbook, Recurring tabs
├── skills/error-journal/
│   └── SKILL.md                  # When and how the agent should call the tool
├── manifest.json                 # Permissions, capabilities, UI wiring
├── app.json                      # Marketplace listing
└── .github/workflows/
    └── build-executa.yml         # PyInstaller binaries for 3 platforms
```
<br>

### **3. Deterministic Fingerprinting**

Two logs of the same bug almost never look identical:

```
2026-08-16T10:22:31Z  pod/payments-api-5d8f9c7b6d-x2k9p  CrashLoopBackOff
2026-09-04T03:11:02Z  pod/payments-api-7c4a1b2e9f-qq81z  CrashLoopBackOff
```

Byte for byte, these share almost nothing. To a person, they are obviously the same problem. The fingerprinter's job is to make the code agree with the person.

It works in four passes:

1. **Strip terminal noise.** ANSI colour codes are removed before any matching happens.
2. **Classify.** Ordered detectors (Kubernetes, Python, Docker, Node, Go, Java, Rust, Ruby, PHP, git, databases, systemd, build tools, shell) each try to recognise the error. Most specific first, first match wins.
3. **Scrub volatile tokens.** Timestamps, UUIDs, container IDs, memory addresses, IPs, pod suffixes, absolute paths, line numbers, and sizes become stable placeholders like `<TS>`, `<HEXID>`, `<PATH>`.
4. **Hash.** The category and the scrubbed template are hashed with SHA-256.

```python
# fingerprint.py (simplified)
def fingerprint(raw: str) -> Fingerprint:
    clean = ORPHAN_ANSI_RE.sub("", ANSI_RE.sub("", raw))

    category, signal, identity, scope = "unknown", clean, {}, None
    for detector in DETECTORS:
        hit = detector(clean)
        if hit:
            category, signal, identity, scope = hit
            break

    # Scrub the signal, THEN append the scope. Scrubbing a scope destroys it.
    template = scrub(signal)[:400]
    if scope:
        template = f"{template} [{scope}]"

    digest = hashlib.sha256(
        f"v{FINGERPRINT_VERSION}|{category}|{template}".encode()
    ).hexdigest()
    return Fingerprint(fingerprint=f"sha256:{digest[:32]}", ...)
```

Two design decisions matter more than the regexes.

**The hash is scoped to the workload, not just the error type.** An early version collapsed every `CrashLoopBackOff` into one bucket. Technically correct, practically useless: "you have hit this 40 times" tells you nothing. Scoped to the failing service (`payments-api-5d8f9c7b6d-x2k9p` becomes `payments-api`), it turns into "payments-api has crash-looped 4 times", which you can act on. The same idea applies to Docker builds (by failing step), image pulls (by repository), and database errors (by host).

**The algorithm is versioned.** `FINGERPRINT_VERSION` is part of the hashed input. If the normalisation rules ever change in a way that would split or merge existing fingerprints, the version bump makes that explicit instead of silently corrupting every user's history. It is one of the few things in the repo with a written rule: never bump it without a deliberate decision.  
<br>

### **4. Three Answer Tiers: Never Fabricate**

A wrong fix during an outage costs more than an honest "I don't know". So every answer comes from exactly one of three tiers, and the tier is always visible:

| Tier | Source | How it is presented |
|---|---|---|
| **Curated** | One of 109 hand-written entries | Verified, authoritative |
| **Generated** | Cached or freshly sampled model diagnosis | Labelled as not verified, check before running anything destructive |
| **Unknown** | Nothing reliable available | Says so plainly |

Generated diagnoses are cached by fingerprint, so the second user with the same unusual error gets the same answer instantly instead of a fresh, possibly different guess.

Storage follows the same honesty principle. If persistent storage is unavailable, diagnosis still works and the app says once that history is unavailable. **The fix is the product. Memory is the bonus.** Nothing breaks because the journal is down.  
<br>

### **5. Real Values in Fix Steps, Safely**

Generic advice like `kubectl logs <pod> --previous` makes the user do the substitution themselves. Error Journal fills in the real values the detectors already extracted:

```bash
kubectl logs payments-api-5d8f9c7b6d-x2k9p --previous
```

The catch: those values come from **user-pasted text**, and they land in **commands people copy straight into a shell**. That is a textbook injection path. So substitution is gated by a strict allow-list:

```python
# error_journal_plugin.py (simplified)
SAFE_VALUE_RE = re.compile(r"^[A-Za-z0-9._:/@-]{1,100}$")

def _resolve_placeholder(name: str, identity: dict):
    if name == "password":
        return None          # never substituted, under any circumstance
    for key in PLACEHOLDER_IDENTITY_KEYS.get(name, ()):
        value = identity.get(key)
        if value is None:
            continue
        value = str(value)
        return value if SAFE_VALUE_RE.match(value) else None
    return None
```

If a value contains a space, a semicolon, a backtick, a `$`, a quote, or a newline, the placeholder stays as it is. The worst case is a slightly less convenient command, never a dangerous one. A dedicated test suite covers the injection cases.  
<br>

### **6. A Plugin That Talks Back**

On Anna, a tool plugin is a long-running process that speaks JSON-RPC 2.0 over stdin and stdout. Requests come in from the agent, results go back out. Simple, until the plugin needs something from the host, like reading the journal or asking the model for a generated diagnosis.

Those are **reverse RPCs**: the plugin sends a request to the host and waits for the answer on the same stdin it reads normal work from. That means two kinds of traffic share one channel, and while the plugin is waiting for a storage response, a brand new request from the agent can arrive first.

Answering it out of order, or dropping it, would both be bugs. The fix is a forward queue:

```python
# error_journal_plugin.py (simplified)
def reverse_rpc(method, params, invoke_id=None, timeout=None):
    rpc_id = f"rev-{next(_rpc_ids)}"
    _write({"jsonrpc": "2.0", "id": rpc_id, "method": method, "params": params})

    while True:
        msg = _next_message(timeout=remaining)
        if msg.get("method"):
            _forward_queue.append(msg)   # new work: not ours to answer now
            continue
        if msg.get("id") != rpc_id:
            continue                     # stale or unknown response
        return msg.get("result") or {}
```

The main loop drains the queue before reading new input, so every request is answered exactly once, in order. Timeouts raise a single `StorageUnavailable` exception that the rest of the code already knows how to degrade around.

The whole plugin is **Python standard library only**, packaged with PyInstaller into single binaries for Linux x86_64, macOS arm64, and Windows x86_64 by a GitHub Actions workflow, with a smoke test on each platform before packaging.  
<br>

### **7. Bugs That Only Real Input Could Find**

The scariest bugs in this project never crashed anything. The app just quietly said "I don't recognise this" and gave the user nothing. Invisible failures are the expensive kind. A stress corpus of real, messy logs found them:

**❌ ANSI colour codes broke detection**

A traceback copied from a CI log was unrecognisable. Worse, the reset code `\x1b[0m` matched the duration regex and became `<DURATION>`. Fix: strip escape sequences, including orphaned ones like `[0m`, before any matching.

**❌ Log prefixes killed matching entirely**

```
Aug 16 10:22:31 web-01 app[1234]: KeyError: 'user_id'
```

This went unrecognised because the detector was anchored to the start of the line. But this is how most people actually paste logs: syslog lines, pytest gutters, docker-compose service tags. Fix: a dedicated prefix rule that removes these before classification.

**❌ Cross-language collisions**

Java's `NullPointerException` and JavaScript's `TypeError` were being classified as Python, because the Python detector matched anything ending in `Error` or `Exception`. Fix: order detectors from most specific to least, and give each language a real signature instead of a suffix.

Today the stress corpus runs on every change and reports how many real-world logs fall through to "unknown". The current count is 0 of 46. The test suites print PASS or FAIL summaries instead of relying on exit codes, and the rule is to read the output, not trust a green tick.  
<br>

### **8. Building on a Brand New Platform**

Anna is a young platform, and building one of the first apps on its marketplace meant hitting edges nobody had hit yet. I treated those as part of the job: reproduce precisely, report clearly, and propose a fix where I could.

**A bug that only failed in production.** Storage worked perfectly in the local harness and failed in production with `-32021 storage_token missing`. Every declaration on my side checked out. I wrote up a full reproduction: the capabilities extracted from the shipped binary, the platform's stored manifest, the grants, and a side by side of the local and production paths. Anna's team found the root cause (the host only minted plugin storage tokens for one family of capability strings) and fixed it in `v1.1.0-beta.159`. They also changed the local harness to apply the same capability gate as production, so the "works locally, fails in prod" gap closed for every developer.

**Model paraphrasing of output that must be exact.** The app computes a headline like "This is the 3rd time you have hit this". If the model rewords or drops it, the whole point of the app disappears. Prompt instructions to "print this verbatim" helped, but not reliably. I proposed a platform channel for presentation-critical fields that bypasses the model entirely. Anna built it as a new primitive, **`_display` verbatim blocks**, shipped in `v1.1.0-beta.178`: a tool result can carry markdown blocks that the chat UI renders byte for byte.

**Documentation.** Six corrections to the official developer docs were merged, and Anna's team credited the reports in their public release notes. The app itself was approved for the marketplace on 20 September 2026, after two review rounds that were blocked by platform issues, not app bugs.

I also extracted a reusable **[anna-app-template](https://github.com/sadishihab/anna-app-template)** from the project, verified from a clean clone, so the next app does not have to rediscover all of this.  
<br>

### **9. How I Worked: Three Surfaces**

I built this with AI assistance, but with the same discipline I would expect from a team:

1. **Planning chat** – design, review, and breaking work into small steps
2. **Claude Code in the repo** – edits and commits, bound by hard rules in a `CLAUDE.md` file
3. **A plain shell** – independent verification of everything the assistant claims

The hard rules are boring on purpose: stage files by explicit path and never `git add -A`, never bump a version or `FINGERPRINT_VERSION` unasked, ask for a plan before code on anything user-facing, run every test suite before committing, and stop for approval on any user-facing copy. Every commit is confirmed with `git log` and `git status` from the separate shell, because "I committed this" and "this is committed" are not always the same thing.

That workflow caught real problems before users did. One example: a regenerated manifest had silently dropped the agent skill from the app bundle. A one-line `grep` from the verification shell found it, `git log -S` pinpointed the exact commit that removed it, and the fix restored the entry in the same shape that had worked before.  
<br>

### **10. Learning Outcomes**

- Designed **deterministic fingerprinting** that turns noisy, non-identical logs into stable identifiers  
- Learned why **scope matters more than category** for useful repeat detection  
- **Versioned an algorithm** whose output is persisted, so changes can never silently corrupt user history  
- Built a **three-tier answer model** that is honest about what is verified, generated, or unknown  
- Treated **user-pasted text as untrusted input** and gated command substitution with an allow-list  
- Implemented **bidirectional JSON-RPC over stdio** with a forward queue for interleaved traffic  
- Designed for **graceful degradation**: the core feature works with storage completely down  
- Shipped **cross-platform binaries** from a stdlib-only Python codebase via GitHub Actions  
- Used a **real-world stress corpus** to find invisible failures that unit tests never would  
- Wrote **precise platform bug reports** that led to two platform fixes and one new platform feature  
- Practised **AI-assisted engineering with guardrails**: plan first, verify independently, commit deliberately  

<br>

### **Try It**

Paste your next error into [Error Journal on the Anna Marketplace](https://anna.partners/store/@sadi/error-journal). Then paste it again next month and see what it remembers.

If you are building AI agents or tools and want this kind of reliability in your own product, I am happy to talk: [book a 30 minute call](https://calendly.com/sadi-shihab/30min) or find me on [LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi).

<br>

### **References**
- [Error Journal on GitHub](https://github.com/sadishihab/error-journal)
- [Anna App Template](https://github.com/sadishihab/anna-app-template)
- [Anna Platform](https://anna.partners)
- [Anna Developer Docs](https://github.com/Anna-Partners/anna-developer-docs)
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [PyInstaller Documentation](https://pyinstaller.org/)
<br>

### **License**
This project is open-source and available under the MIT License
