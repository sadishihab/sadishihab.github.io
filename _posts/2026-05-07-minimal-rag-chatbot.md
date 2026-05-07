---
title: "Building a Multilingual RAG Chatbot for Facebook Messenger with OpenAI, FAISS & FastAPI"
date: 2026-05-07
layout: post
permalink: /minimal-rag-chatbot/
categories: [AI, RAG, Python, FastAPI]
tags: [RAG, OpenAI, FAISS, FastAPI, LLM, Chatbot, Bangla, Multilingual, Facebook Messenger, Python]
description: "Learn how I built a production-ready, multilingual (Bangla / Banglish / English) Retrieval-Augmented Generation chatbot for Facebook Messenger — from knowledge base curation to embeddings, vector search, prompt engineering, and safe deployment."
author: "Md. Shihabuddin Sadi"
---

*Software Engineer · DevOps & Cloud Native Engineer · AI / RAG Application Developer*  
*May 07, 2026*  

<br>

### **Summary**

This project showcases how I built a **production-ready multilingual chatbot** for an interior design company in Dhaka, Bangladesh, using **Retrieval-Augmented Generation (RAG)** on **Facebook Messenger**.  
The bot accepts customer queries in **Bangla, Banglish, or English**, retrieves the most relevant answer from a curated knowledge base of **224 Q&A entries across 14 intents**, and always replies in **formal Bangla** using **OpenAI's GPT-4o-mini**.  
It demonstrates real-world AI engineering practices such as **embedding pipelines, FAISS vector search, similarity-threshold tuning, prompt engineering, webhook integration**, and a **4-stage safe deployment workflow** — all built **without framework abstractions** (no LangChain, no LlamaIndex) so the entire pipeline stays transparent and debuggable end-to-end.  
The goal was to **ship a real customer-facing AI product** while deeply understanding every component of the RAG stack.  
<br>
🔗 **GitHub Repository:** [sadishihab/minimal-rag-chatbot](https://github.com/sadishihab/minimal-rag-chatbot)

<br>

### Table of Contents

- [Summary](#summary)
- [Key Technologies Used](#key-technologies-used)
- [1. Project Overview](#1-project-overview)
- [Architecture Diagram](#architecture-diagram)
- [2. Folder Structure](#2-folder-structure)
- [3. Setup Instructions](#3-setup-instructions)
- [4. The RAG Pipeline Explained](#4-the-rag-pipeline-explained)
- [5. Common Errors & Fixes](#5-common-errors--fixes)
- [6. Learning Outcomes](#6-learning-outcomes)
- [References](#references)
- [License](#license)
<br>

### **Key Technologies Used**

`Python 3.13` · `OpenAI API` · `text-embedding-3-small` · `gpt-4o-mini` · `FAISS` · `FastAPI` · `Uvicorn` · `Facebook Messenger Platform` · `Pytest` · `NumPy` · `python-dotenv`  
<br>

### **1. Project Overview**

The project demonstrates a **two-phase RAG architecture** deployed as a Facebook Messenger bot:

- **Knowledge Base** – 224 curated Q&A entries across 14 intents in Bangla, Banglish, and English  
- **Embedder** – OpenAI `text-embedding-3-small` (1536-dim vectors)  
- **Vector Store** – FAISS `IndexFlatIP` with L2-normalized vectors for exact cosine similarity  
- **Retriever** – Top-k=3 search with a similarity threshold of 0.3 for graceful fallback  
- **Prompt Builder** – Cross-lingual system prompt enforcing formal Bangla output  
- **LLM** – OpenAI `gpt-4o-mini` (temperature 0.3, max 500 tokens)  
- **Webhook Server** – FastAPI + Uvicorn integrated with Facebook Graph API  
- **Human-in-the-Loop** – Graceful fallback when confidence is low ("share your number, our manager will call")  
<br>

#### **Architecture Diagram**
```pgsql
        ┌──────────────────────────── INGESTION (offline) ────────────────────────────┐
        │                                                                              │
        │  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐       │
        │  │  Knowledge Base  │───►│  OpenAI Embedder │───►│     Indexer      │───┐   │
        │  │  JSON · 224 Q&A  │    │ embedding-3-small│    │ L2-normalize+IP  │   │   │
        │  └──────────────────┘    └──────────────────┘    └──────────────────┘   │   │
        │                                                                          │   │
        │                                                              ┌───────────▼─┐ │
        │                                                              │  Disk Store │ │
        │                                                              │ .index +meta│ │
        │                                                              └───────┬─────┘ │
        └──────────────────────────────────────────────────────────────────────┼───────┘
                                                                               │ load
        ┌──────────────────────────── QUERY (online) ───────────────────────────┼───────┐
        │                                                                       ▼       │
        │  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐         │
        │  │ Facebook Messenger│──►│  FastAPI Webhook │──►│   Embed Query    │         │
        │  │   User message    │   │ Verify · parse   │    │ 1536-dim vector  │         │
        │  └──────────────────┘    └──────────────────┘    └────────┬─────────┘         │
        │           ▲                                                │                   │
        │           │                                                ▼                   │
        │           │                            ┌──────────────────────────────────┐    │
        │           │                            │   Retriever (top-k=3, thr 0.3)   │    │
        │           │                            │      ◄── FAISS Index             │    │
        │           │                            └────────────────┬─────────────────┘    │
        │           │                                             │                      │
        │           │                                  ┌──────────┴──────────┐           │
        │           │                                  │                     │           │
        │           │                            (match)                  (no match)     │
        │           │                                  │                     │           │
        │           │                                  ▼                     ▼           │
        │           │                       ┌──────────────────┐   ┌──────────────────┐  │
        │           │                       │ Prompt Builder   │   │  Fallback reply  │  │
        │           │                       │ Sys+Ctx+Query    │   │ "Share number,   │  │
        │           │                       └────────┬─────────┘   │  manager calls"  │  │
        │           │                                ▼              └─────────┬────────┘  │
        │           │                       ┌──────────────────┐              │           │
        │           │                       │   GPT-4o-mini    │              │           │
        │           │                       │ Formal Bangla out│              │           │
        │           │                       └────────┬─────────┘              │           │
        │           │                                │                        │           │
        │           └────────────────────────────────┴────────────────────────┘           │
        │                                  reply via Graph API                             │
        └───────────────────────────────────────────────────────────────────────────────────┘

```
<br>

### **2. Folder Structure**
```bash
├── data/                       # Knowledge base (JSON)
│   └── knowledge_base.json     # 224 Q&A entries · 14 intents
├── ingestion/                  # Offline pipeline
│   ├── loader.py               # Loads + validates KB
│   ├── embedder.py             # OpenAI embeddings
│   └── indexer.py              # FAISS index builder
├── retrieval/                  # Online pipeline
│   ├── retriever.py            # Top-k semantic search
│   └── prompt_builder.py       # Cross-lingual prompt assembly
├── api/                        # Web layer
│   ├── main.py                 # FastAPI app
│   └── messenger.py            # Graph API client + webhook
├── vector_store/               # Persisted FAISS artifacts
│   ├── faiss.index
│   └── metadata.json
├── tests/                      # Pytest suite
│   └── test_loader.py          # 12 passing tests
├── config.py                   # Centralized config + thresholds
├── .env.example                # Required environment variables
└── requirements.txt

```
<br>

### **3. Setup Instructions**

**Clone and install**
```bash
git clone https://github.com/sadishihab/minimal-rag-chatbot.git
cd minimal-rag-chatbot
python -m venv .venv
source .venv/bin/activate         # On Windows: .venv\Scripts\activate
pip install -r requirements.txt

```
<br>

**Configure environment**
```bash
cp .env.example .env
# Then edit .env with your keys:
# OPENAI_API_KEY=sk-...
# FB_PAGE_ACCESS_TOKEN=...
# FB_VERIFY_TOKEN=...

```
<br>

**Build the FAISS index (one-time)**
```bash
python -m ingestion.loader      # Validate KB
python -m ingestion.embedder    # Generate embeddings
python -m ingestion.indexer     # Build + persist FAISS index

```
<br>

**Run the webhook server**
```bash
uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload

```
<br>

**Test in the terminal first**
```bash
python -m retrieval.retriever   # Interactive REPL for retrieval testing

```
<br>

### **4. The RAG Pipeline Explained**

To keep the pipeline transparent and learnable, every stage was built directly on the OpenAI Python SDK and FAISS — no framework wrappers.

**Embedding strategy**

I embed the **question** field of each Q&A entry, not the answer. Customers send questions, so questions live in the searchable vector space; answers stay in metadata and are retrieved by index.

```python
# ingestion/embedder.py (simplified)
response = client.embeddings.create(
    model="text-embedding-3-small",
    input=[entry["question"] for entry in kb]
)
vectors = np.array([d.embedding for d in response.data], dtype="float32")
faiss.normalize_L2(vectors)     # Cosine similarity via inner product

```
<br>

**FAISS index — exact cosine similarity**

For 224 vectors, an exact `IndexFlatIP` is faster than any approximate index and trivially correct. L2-normalizing the vectors turns inner product into cosine similarity.

```python
# ingestion/indexer.py (simplified)
index = faiss.IndexFlatIP(1536)
index.add(vectors)
faiss.write_index(index, "vector_store/faiss.index")

```
<br>

**Retrieval with similarity threshold**

The retriever returns the top-k=3 hits. If the best match falls below similarity 0.3, the bot triggers a graceful human-in-the-loop fallback instead of hallucinating an answer.

```python
# retrieval/retriever.py (simplified)
def retrieve(query: str, top_k: int = 3, threshold: float = 0.3):
    qv = embed(query)
    faiss.normalize_L2(qv)
    scores, idxs = index.search(qv, top_k)
    if scores[0][0] < threshold:
        return None                  # Trigger fallback
    return [metadata[i] for i in idxs[0]]

```
<br>

**Cross-lingual prompt engineering**

Input arrives in Bangla, Banglish, or English. The system prompt **always** forces formal Bangla output, and the retrieved entries are injected as context.

```python
# retrieval/prompt_builder.py (simplified)
SYSTEM = (
    "You are a customer-service assistant for an interior design company. "
    "You MUST always respond in formal Bangla, regardless of the input language. "
    "Use only the context provided. If unsure, say you'll have a manager follow up."
)

```
<br>

**Safe 4-stage deployment workflow**

To avoid breaking the live customer experience, every change rolls out through four gates:

1. **Terminal REPL** – test retrieval logic offline
2. **Local web** – test the webhook with `ngrok` and a test message
3. **Test Facebook page** – test on a private FB page with the same Graph API
4. **Live Facebook page** – promote only after the previous gates pass

<br>

### **5. Common Errors & Fixes**

**❌ Webhook verification fails on Facebook**

Facebook expects an exact match on the `hub.verify_token` query param.
```bash
# Make sure FB_VERIFY_TOKEN in .env matches what you typed in the FB app dashboard.
echo $FB_VERIFY_TOKEN

```

**❌ Bot replies in English instead of Bangla**

The system prompt's language constraint must be **non-negotiable** and the model must be instructed not to translate user input verbatim. Lowering temperature (0.3) helps too.

**❌ FAISS returns irrelevant top match**

Three common causes:
- The KB doesn't actually contain a matching intent → add the entry, rebuild the index.
- The threshold is too low → raise from 0.3 to 0.4 and re-test.
- You embedded the answer instead of the question → re-embed using the question field.

**❌ `openai.RateLimitError` during indexing**

Batch your embedding calls, or add retry-with-backoff. For 224 entries one batched call is usually enough.

**❌ Pods of Bangla characters render as boxes on Messenger**

Make sure responses are sent as UTF-8 JSON, not Latin-1, and that your hosting platform doesn't strip non-ASCII characters in logs/middleware.

<br>

### **6. Learning Outcomes**

- Designed a **production RAG pipeline** end-to-end without framework abstractions
- Made deliberate decisions about **what to embed** (questions, not answers) and why
- Implemented **cosine similarity via L2-normalized inner product** — the standard FAISS trick
- Tuned **similarity thresholds** to balance helpful answers vs safe fallbacks
- Engineered **cross-lingual prompts** that accept three input languages and enforce one output language
- Curated a **multilingual knowledge base** with parallel entries per intent for robust retrieval
- Integrated with the **Facebook Messenger Platform** (Graph API, webhook verification, page tokens)
- Adopted a **4-stage staged rollout** discipline for safe production changes
- Designed **human-in-the-loop fallback** so the bot is an assistant, not a replacement
- Wrote **pytest tests** for the data layer (schema, ID uniqueness, language enums, intent coverage)

<br>

### **References**
- [OpenAI API Documentation](https://platform.openai.com/docs)
- [FAISS GitHub](https://github.com/facebookresearch/faiss)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Facebook Messenger Platform](https://developers.facebook.com/docs/messenger-platform)
<br>

### **License**
This project is open-source and available under the MIT License
