---
layout: default
title: Md. Shihabuddin Sadi — AI / RAG Application Developer | Production Chatbots & Agents
description: I build production RAG chatbots and AI agents that ship — multilingual support, grounded retrieval, no hallucinations. Ex-Samsung R&D · 15+ years of software engineering. Book a call.
image: /vector-forge-og-image-v2.png
---

<!-- Preconnect to shields.io for faster badge loading -->
<link rel="preconnect" href="https://img.shields.io" crossorigin>

<!-- Open Graph / social share -->
<meta property="og:title" content="Md. Shihabuddin Sadi — AI / RAG Application Developer">
<meta property="og:description" content="Production RAG chatbots and AI agents that ship. Multilingual, grounded, no hallucinations. Ex-Samsung R&D · 15+ years of software engineering.">
<meta property="og:type" content="website">
<meta property="og:url" content="https://sadishihab.github.io/">


## I build production RAG chatbots and AI agents that ship.

**Multilingual support. Grounded retrieval. No hallucinations.**
Built for real users, real traffic, real outcomes.

I run **Vector Forge** · Ex-Samsung R&D · 15+ years of software engineering · Based in Dhaka, Bangladesh · Available worldwide remote

<div style="display:flex; flex-wrap:wrap; gap:12px; margin: 20px 0 30px 0;">

<a href="https://calendly.com/sadi-shihab/30min" style="background:#0a66c2; color:white; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600;">
📅 Book a 30-min call
</a>

<a href="#featured-project" style="background:#f3f2ef; color:#0a66c2; padding:12px 24px; border-radius:6px; text-decoration:none; font-weight:600; border:1px solid #0a66c2;">
See my work
</a>

</div>

<br>

---

## What I Build

Production RAG chatbots over your docs, PDFs, Notion, or SQL — with citations, not hallucinations. Multilingual AI agents that handle Bangla, Banglish, English, and other low-resource or script-mixed languages, which makes them especially useful for South Asian, Middle East, and emerging-market audiences.

Beyond that, I build Messenger, WhatsApp, Telegram, and Slack bots wired to real business data, custom AI copilots embedded inside SaaS products, and the evaluation pipelines, observability, and guardrails that keep all of it from silently regressing in production.

I also handle the cloud infrastructure to keep it running reliably — Kubernetes, AWS, Terraform, CI/CD, Prometheus, Grafana. One contractor, one accountable line, no hand-off between the AI person and the DevOps person.

<br>

---

<a id="featured-project"></a>

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

## Why Teams Hire Me

I've shipped real software for 15+ years — not just AI demos.

I bring engineering rigor: evals, logging, retrieval tuning, and guardrails. The unglamorous work that decides whether your AI survives contact with real users.

And because I can build both the AI and the cloud infrastructure it runs on, there's no hand-off between the AI person and the DevOps person. One contractor, one accountable line.

<br>

---

## How We'd Work Together

1. **30-min discovery call** — tell me about your product, your data, and where AI fits
2. **Scoped proposal within 48 hours** — what I'd build, timeline, cost
3. **Build, ship, iterate** — typically 2–6 weeks for a production RAG pilot
4. **Optional ongoing support** — evals, observability, infra, and iteration

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

### AI / LLM / RAG

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

<a href="https://www.kernel.org/">
<img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" height="28">
</a>

</div>

<br>

### Monitoring & Observability

<div style="display:flex; flex-wrap:wrap; gap:5px;">

<a href="https://prometheus.io/">
<img src="https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white" height="28">
</a>

<a href="https://grafana.com/">
<img src="https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white" height="28">
</a>

</div>

<br>

---

## Open Source

**[anna-developer-docs](https://github.com/Anna-Partners/anna-developer-docs)** — *Documentation corrections, merged*

While building on a new AI application platform, I lost a day to behaviour that contradicted the documentation. Rather than work around it, I traced each discrepancy through the runtime source and wrote up seven findings with replacement text.

All seven were verified as accurate. Six were merged into the public developer documentation, including a capability string that no longer existed in the runtime, a required manifest field missing from the reference table, and a config schema documented with the wrong data type.

The seventh turned out to be a platform bug rather than a docs error — editing a resource through the web UI silently reset its visibility, causing publish failures that appeared to be user error. Confirmed and fixed in the following release.

> *"One of the best community write-ups we've received — seven precise findings, each verified against actual runtime behavior. We verified all seven items and every single one was accurate."* — platform engineering team

**Why it's here:** most of these were found by reading the runtime source rather than re-reading the docs. That habit — verifying behaviour instead of trusting documentation — is the same one that keeps production systems debuggable.

<br>

**[anna-app-template](https://github.com/sadishihab/anna-app-template)** — *reusable scaffold, published for other builders*

After shipping an app on the same platform, I extracted the parts worth reusing so the next person doesn't repeat the discovery. JSON-RPC transport with a forward queue for concurrent reverse-RPC calls, persistent storage and model sampling that degrade gracefully when unavailable, three-platform binary CI, and a publish runbook documenting the failure mode at each step.

Clone, run the rename script, get a running plugin — verified from a clean clone rather than assumed.

<br>

---

## Other Work

A selection of supporting projects across cloud infrastructure, DevOps automation, and software engineering.

**[Error Journal](https://github.com/sadishihab/error-journal)**
A diagnostic tool that fingerprints errors deterministically so the same underlying failure is recognised across different machines, timestamps, and pod names — then surfaces what fixed it last time. 109 curated diagnoses across seven languages plus Kubernetes, Docker, and shell. Shipped as single-file binaries for three platforms via a GitHub Actions matrix.
**Tech:** Python (stdlib only) · PyInstaller · GitHub Actions · JSON-RPC

**[Anna App Template](https://github.com/sadishihab/anna-app-template)**
A reusable scaffold extracted from a shipped application — working JSON-RPC transport, persistent storage, model sampling, three-platform binary builds, and a publish runbook. Verified end-to-end from a clean clone.
**Tech:** Python · PyInstaller · GitHub Actions · JSON-RPC

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

**[Python Automation](https://github.com/sadishihab/automation-with-python)**
Engineering utilities and workflow automation in Python.
**Tech:** Python · Bash

[See all repositories on GitHub](https://github.com/sadishihab?tab=repositories)

<br>

---

## Background

15+ years of software engineering across embedded systems, mobile, full-stack, cloud, and AI.

I started at Samsung R&D Bangladesh, where I worked on firmware for handsets shipped across Middle East, Africa, and Bangladesh — including the Bengali Calendar for the Bangladesh region and language support for Swahili, Yoruba, Igbo, Hausa, and Amharic on Samsung feature phones used by millions. That's where multilingual production software became muscle memory, which weirdly turned out to be great prep for the multilingual RAG work I do now.

After Samsung, I co-founded **Training Pool**, Bangladesh's first online training marketplace and SaaS platform. Took it from idea to live product with paying users. Before that, I ran a small dev studio building Android multiplayer games and Bangladesh client projects.

These days I run **Vector Forge**, shipping production RAG applications and AI agents for founders, agencies, and mid-market teams.

[See full work history on LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi/)

<br>

---

## Blog

[Read my latest posts](/blog/) · [RSS feed](https://sadishihab.github.io/feed.xml)

<br>

---

## Let's Talk

If your chatbot is hallucinating, your AI feature isn't making it past the demo stage, or you want to add a real RAG system to your product without it embarrassing you in front of customers — let's talk.

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
