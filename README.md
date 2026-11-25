# 🚀 Enterprise IT Support Agentic RAG Copilot

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.110%2B-009688?logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-Agentic_RAG-orange)
![Pinecone](https://img.shields.io/badge/Vector_DB-Pinecone-black?logo=pinecone&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)

An end-to-end **Forward Deployed Engineer (FDE)** solution that transforms a prototype Agentic RAG pipeline into a production-ready internal enterprise product. Built using **LangGraph, FastAPI, Pinecone, Groq, Tavily, and Vanilla JS/HTML/CSS**, this system autonomously routes, grades, and retrieves internal IT documentation while seamlessly falling back to real-time web search when internal knowledge is insufficient.

---

## 📋 Table of Contents
1. [Business Context & Problem Statement](#-1-business-context--problem-statement)
2. [Forward Deployed Engineering (FDE) Approach](#-2-forward-deployed-engineering-fde-approach)
3. [System Architecture](#-3-system-architecture)
4. [Agentic RAG Decision Workflow](#-4-agentic-rag-decision-workflow)
5. [Technology Stack](#-5-technology-stack)
6. [Project Structure](#-6-project-structure)
7. [Installation & Local Setup](#-7-installation--local-setup)
8. [Docker Deployment](#-8-docker-deployment)
9. [Evaluation & Routing Scenarios](#-9-evaluation--routing-scenarios)
10. [Knowledge Base Ingestion Pipeline](#-10-knowledge-base-ingestion-pipeline)
11. [API Reference](#-11-api-reference)

---

## 🎯 1. Business Context & Problem Statement

### Client Profile
**NovaRetail** — A simulated 3,000-employee retail enterprise managing distributed corporate and store-level IT operations.

### The Operational Bottleneck
NovaRetail's IT department maintains extensive internal documentation (VPN setup guides, password policies, MFA enforcement rules, software allowlists, and service-desk runbooks). Despite this, Tier-1 support remains overwhelmed by repetitive tickets due to:
* **Information Silos:** Employees struggle to locate relevant policies across fragmented documents.
* **Keyword Search Limitations:** Legacy search returns noisy, unranked results.
* **Hallucination Risks:** Standard LLM wrappers fabricate answers when internal documentation is missing or outdated.
* **Static Knowledge:** Internal KBs cannot answer real-time vendor outage queries (e.g., live Microsoft 365 or Slack disruptions).

### Core Engineering Objectives
1. **Private-First Retrieval:** Prioritize verified internal company knowledge before touching the public internet.
2. **Self-Correction & Grading:** Evaluate retrieved context relevance before generating an answer.
3. **Controlled Web Fallback:** Trigger external search (Tavily) *only* when private evidence is weak, flagging external answers to users.
4. **Query Rewriting & Retry Guards:** Automatically reformulate ambiguous user queries while enforcing loop limits to control latency and token costs.
5. **Auditability & Transparency:** Expose step-by-step execution traces (`Router → Retrieval → Grading`) and log queries to SQLite for compliance.
6. **Self-Service Knowledge Ingestion:** Provide an authenticated admin interface to upload new policy documents (`.pdf`, `.docx`, `.md`, `.txt`) without code changes.

---

## 🛠️ 2. Forward Deployed Engineering (FDE) Approach

Unlike isolated Jupyter Notebook experiments, Forward Deployed Engineering requires bridging the gap between raw AI capabilities and end-user operational workflows:

```text
[Customer Pain Point] ──▶ [Discovery & Requirements] ──▶ [System Architecture]
                                                                 │
[Observe & Iterate]   ◀── [Docker & Deployment]      ◀── [Agentic Graph & API]
         ▲                                                       │
         └─────────────── [Security, Audit & UI]     ◀───────────┘
```

This repository implements every layer of that lifecycle—from vector ingestion and state-machine orchestration to role-protected REST APIs, audit logging, and an interactive employee UI.

---

## 🏗️ 3. System Architecture

```text
                   ┌─────────────────────┐
                   │   Employee / Admin  │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ HTML/CSS/JavaScript │
                   │   Interactive UI    │
                   └──────────┬──────────┘
                              │ REST API (POST /api/chat, /api/ingest)
                              ▼
                   ┌─────────────────────┐
                   │   FastAPI Backend   │◄───► [SQLite Audit Logs]
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │     LangGraph       │
                   │ Agentic State Graph │
                   └──────────┬──────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
       ┌───────────────┐              ┌──────────────┐
       │  Private KB   │              │  Tavily API  │
       │  (Pinecone)   │              │ (Web Search) │
       └───────┬───────┘              └──────┬───────┘
               │                             │
               └──────────────┬──────────────┘
                              ▼
                     ┌────────────────┐
                     │    Groq LLM    │
                     │ Grounded Reply │
                     └────────────────┘
```

---

## 🧠 4. Agentic RAG Decision Workflow

Traditional RAG follows a rigid linear path (`Retrieve → Generate`), which fails silently on bad retrievals. This system implements a cyclic, self-evaluating state machine via **LangGraph**:

```text
User Query
   │
   ▼
[1] Intent Router
   ├── Conversational / Greeting ──────────► Direct LLM Response
   │
   └── Technical / Policy Query
                │
                ▼
[2] Retrieve from Private Pinecone KB
                │
                ▼
[3] Grade Private Evidence (Relevance Check)
       ┌────────┴────────┐
       │                 │
     GOOD               WEAK
       │                 │
       ▼                 ▼
Generate Grounded   [4] Tavily Web Search
  KB Response            │
                         ▼
                  [5] Grade Web Evidence
                    ┌────┴─────┐
                    │          │
                  GOOD        WEAK
                    │          │
                    ▼          ▼
              Generate Web  [6] Rewrite Query
               Response        │
               (w/ Warning)    ▼
                         Retry Private KB
                               │
                        Max Retries Reached? ──► Graceful Fallback Response
```

---

## 💻 5. Technology Stack

| Layer | Technology | Role in Architecture |
| :--- | :--- | :--- |
| **Orchestration** | **LangGraph** | Stateful cyclic graph managing routing, grading, and retry loops |
| **Inference Engine** | **Groq API** | Ultra-low-latency LLM inference for routing, grading, and generation |
| **Embeddings** | **HuggingFace / OpenAI** | Dense vector representations (`all-MiniLM-L6-v2` / `text-embedding-3-small`) |
| **Vector Database** | **Pinecone** | Managed serverless vector store with namespace isolation |
| **Web Fallback** | **Tavily Search API** | AI-optimized real-time web search for external vendor updates |
| **Backend API** | **FastAPI** | High-performance async REST endpoints and OpenAPI documentation |
| **Frontend** | **HTML5 / CSS3 / JS** | Lightweight, dependency-free employee chat and admin upload portal |
| **Audit & Compliance** | **SQLite** | Persistent logging of user queries, decision traces, and latency metrics |
| **Containerization** | **Docker** | Production-ready container packaging for consistent deployment |

---

## 📂 6. Project Structure

```text
FDE_Agentic_RAG_IT_Copilot/
├── app/
│   ├── api/
│   │   └── routes.py              # REST endpoints (/chat, /ingest, /health)
│   ├── core/
│   │   ├── config.py              # Pydantic settings & environment validation
│   │   └── logging.py             # Structured application logging
│   ├── rag/
│   │   ├── state.py               # LangGraph TypedDict state & structured outputs
│   │   ├── vectorstore.py         # Pinecone client & embedding initialization
│   │   └── workflow.py            # Core Agentic RAG nodes & conditional edges
│   ├── services/
│   │   ├── audit.py               # SQLite audit trail persistence
│   │   └── ingestion.py           # Multi-format document parser & text splitter
│   └── main.py                    # FastAPI app entrypoint & static file mounting
├── data/
│   └── sample_kb/                 # Seed enterprise IT documentation
│       ├── company_it_handbook.md
│       └── service_desk_runbook.md
├── static/
│   ├── css/style.css              # UI styling & decision trace badges
│   └── js/app.js                  # Frontend state, API calls & file upload logic
├── templates/
│   └── index.html                 # Main application interface
├── tests/
│   └── test_ingestion.py          # Unit tests for document chunking & parsing
├── .env.example                   # Template for required environment variables
├── Dockerfile                     # Multi-stage/slim container definition
├── ingest_sample_kb.py            # CLI script to seed initial Pinecone index
├── requirements.txt               # Pinned Python dependencies
├── run.py                         # Uvicorn server runner
└── README.md
```

---

## ⚙️ 7. Installation & Local Setup

### Prerequisites
* Python **3.10+**
* Active API keys for **Groq**, **Pinecone**, and **Tavily** (plus **OpenAI** if using OpenAI embeddings/fallbacks).

### Step 1: Clone & Create Virtual Environment
```bash
git clone https://github.com/yourusername/FDE_Agentic_RAG_IT_Copilot.git
cd FDE_Agentic_RAG_IT_Copilot

python -m venv venv
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate
```

### Step 2: Install Dependencies
```bash
pip install -r requirements.txt
```

### Step 3: Configure Environment Variables
Copy `.env.example` to `.env` and populate your credentials:
```bash
cp .env.example .env
```

```env
# LLM & Embedding Providers
GROQ_API_KEY=gsk_your_groq_api_key_here
OPENAI_API_KEY=sk-your_openai_api_key_here

# External Search
TAVILY_API_KEY=tvly-your_tavily_api_key_here

# Vector Database (Pinecone)
PINECONE_API_KEY=pcsk_your_pinecone_api_key_here
PINECONE_INDEX_NAME=fde-it-support-rag
PINECONE_NAMESPACE=company-it-kb

# Model Configuration
GROQ_MODEL=llama-3.3-70b-versatile
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=all-MiniLM-L6-v2

# Security & App Settings
ADMIN_API_KEY=replace-with-strong-random-secret
APP_ENV=development
```

#### Environment Variable Reference

| Variable | Description | Default | Required |
| :--- | :--- | :--- | :---: |
| `GROQ_API_KEY` | API key for primary Groq LLM inference | — | ✅ |
| `TAVILY_API_KEY` | API key for external web search fallback | — | ✅ |
| `PINECONE_API_KEY` | API key for Pinecone vector database | — | ✅ |
| `OPENAI_API_KEY` | API key for OpenAI embeddings/fallback (if configured) | — | ⚠️ |
| `PINECONE_INDEX_NAME` | Target Pinecone index name | `fde-it-support-rag` | ❌ |
| `PINECONE_NAMESPACE` | Namespace isolating company documents | `company-it-kb` | ❌ |
| `GROQ_MODEL` | Groq model ID for routing, grading, and generation | `llama-3.3-70b-versatile` | ❌ |
| `EMBEDDING_MODEL` | HuggingFace or OpenAI embedding model name | `all-MiniLM-L6-v2` | ❌ |
| `ADMIN_API_KEY` | Header secret (`X-Admin-Key`) protecting `/api/ingest` | `change-me` | ✅ |

> **Security Best Practice:** Never commit `.env` to version control. Always rotate `ADMIN_API_KEY` before deploying to shared or production environments.

### Step 4: Seed the Knowledge Base
Ingest the sample NovaRetail IT handbook and service-desk runbook into Pinecone:
```bash
python ingest_sample_kb.py
```

### Step 5: Launch the Application
```bash
python run.py
```
* **Web Application:** `http://127.0.0.1:8000`
* **Interactive Swagger API Docs:** `http://127.0.0.1:8000/docs`

---

## 🐳 8. Docker Deployment

To build and run the application in an isolated container:

```bash
# Build the Docker image
docker build -t fde-it-copilot .

# Run the container using your local .env file
docker run -d \
  --name it-copilot \
  -p 8000:8000 \
  --env-file .env \
  fde-it-copilot
```

---

## 🧪 9. Evaluation & Routing Scenarios

The copilot dynamically adapts its execution path based on query intent and evidence quality:

| Scenario | Sample User Prompt | Execution Path | System Behavior |
| :--- | :--- | :--- | :--- |
| **1. Internal Policy Hit** | *"How do I connect to the company VPN from home?"* | `Router (KB) → Retrieve → Grade (GOOD) → Generate` | Answers strictly from internal KB; zero external web calls. |
| **2. Security Compliance** | *"Can IT support ask me for my MFA code?"* | `Router (KB) → Retrieve → Grade (GOOD) → Generate` | Enforces internal zero-trust policy found only in private docs. |
| **3. Real-Time Outage** | *"What is the latest Microsoft Teams outage guidance?"* | `Router (KB) → Grade (WEAK) → Tavily → Grade (GOOD) → Generate` | Detects stale/missing KB info, fetches live status, and adds an external source disclaimer. |
| **4. Ambiguous Query** | *"My work chat app is crashing after the update."* | `KB (WEAK) → Web (WEAK) → Rewrite Query → Retry KB` | Reformulates vague phrasing into targeted technical search terms with retry limits. |
| **5. Chitchat / Greeting** | *"Hello, good morning!"* | `Router (DIRECT) → Generate` | Bypasses vector DB and web search entirely to save latency and API costs. |

---

## 📥 10. Knowledge Base Ingestion Pipeline

Authorized administrators can upload new policies directly via the UI sidebar (**Add Company Document**) using the configured `ADMIN_API_KEY`:

```text
File Upload (.pdf, .docx, .md, .txt)
       │
       ▼
[MIME & Extension Validation]
       │
       ▼
[Document Loader & Recursive Text Chunking]
       │
       ▼
[Dense Vector Embedding Generation]
       │
       ▼
[Upsert to Pinecone Namespace] ──► Immediately Queryable by Copilot
```

---

## 🔌 11. API Reference

### `GET /api/health`
Verifies service availability and configuration status.

### `POST /api/chat`
Executes the Agentic RAG workflow and returns the grounded response alongside its full decision trace.

**Request:**
```json
{
  "question": "How do I reset my company password?"
}
```

**Response:**
```json
{
  "answer": "To reset your NovaRetail password, navigate to the internal SSO portal...",
  "source_used": "private_kb",
  "trace": [
    "Router → KB",
    "Private KB retrieval → 4 chunks",
    "KB evidence grade → GOOD",
    "Answer generation → PRIVATE KB"
  ],
  "citations": ["company_it_handbook.md"]
}
```

### `POST /api/ingest`
Protected endpoint (`Header: X-Admin-Key: <ADMIN_API_KEY>`) that accepts `multipart/form-data` file uploads, chunks the content, and indexes vectors into Pinecone.