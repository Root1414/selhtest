# Architecture Document: Sellh (صله)

> **Updated — 2026-02-24 | n8n-Powered Architecture**

---

## 1. Executive Summary | الملخص التنفيذي

Sellh is a **Layered Web Application with n8n as the AI & Workflow Engine**. The backend is Python/Flask, which exposes a REST API and delegates all AI matching logic and automation workflows to **n8n** — a self-hosted workflow automation platform. n8n handles semantic product matching via its built-in AI nodes (OpenAI Embeddings + Vector Store), compliance monitoring automation, notifications, and audit logging.

> **لماذا n8n؟** يوفر n8n بيئة بصرية لبناء سير العمل (Workflows) دون الحاجة لكتابة كود Python معقد، مع دعم كامل لنماذج الذكاء الاصطناعي مثل OpenAI وتخزين المتجهات (Vector Store).

---

## 2. Architecture Overview | نظرة معمارية شاملة

```
┌─────────────────────────────────────────────────────────┐
│                   PRESENTATION LAYER                    │
│     Streamlit Dashboard  │  Flask HTML Templates        │
└──────────────────┬──────────────────────────────────────┘
                   │ HTTP
┌──────────────────▼──────────────────────────────────────┐
│               FLASK REST API (Port 5000)                │
│  - Auth (Admin / Buyer sessions)                        │
│  - Factory & Product CRUD                               │
│  - Calls n8n webhooks for AI tasks                     │
└──────────┬───────────────────────┬──────────────────────┘
           │ Webhook POST          │ SQL queries
           ▼                       ▼
┌──────────────────────┐  ┌────────────────────────────────┐
│   n8n (Port 5678)    │  │        SQL DATABASE            │
│  ┌────────────────┐  │  │  USERS / FACTORY               │
│  │ AI Match Flow  │  │  │  LOCAL_PRODUCT                 │
│  │ Webhook Trigger│  │  │  PROCUREMENT_ITEM              │
│  │ OpenAI Embed.  │  │  │  AI_MATCH_LOG                  │
│  │ Vector Search  │  │  │  N8N_EXECUTION_LOG             │
│  │ Score & Rank   │  │  └────────────────────────────────┘
│  │ HTTP Response  │  │
│  └────────────────┘  │
│  ┌────────────────┐  │
│  │ Compliance Flow│  │
│  │ Schedule(daily)│  │
│  │ Score Calc.    │  │
│  │ Alert Email    │  │
│  └────────────────┘  │
└──────────────────────┘
```

---

## 3. n8n Workflow Definitions | تعريفات سير العمل في n8n

### Workflow 1: AI Product Matching (مطابقة المنتجات بالذكاء الاصطناعي)

**Trigger:** Webhook `POST /webhook/match-product`  
**Steps:**

```
[Webhook Node]
  ↓ Receives: { procurement_description, procurement_id }
[OpenAI Embeddings Node]
  ↓ Generates vector for the input text (text-embedding-3-small)
[Postgres Node] ← Fetches all LOCAL_PRODUCT descriptions + IDs
[Code Node (JS)]
  ↓ Calculates cosine similarity between query vector and all product vectors
  ↓ Sorts by score, returns top N matches with factory info
[Postgres Node] ← Writes results to AI_MATCH_LOG
[Respond to Webhook Node]
  ↓ Returns: { matches: [...], total_matches: N }
```

### Workflow 2: Compliance Score Monitor (مراقبة درجة الامتثال)

**Trigger:** Schedule (every day at 08:00 AM Riyadh time)  
**Steps:**

```
[Schedule Trigger Node]
[Postgres Node] ← Queries total items vs matched items per organization
[Code Node (JS)] ← Calculates local content percentage
[IF Node]
  → If score < target:
      [Email Node] ← Sends alert to procurement manager
      [Postgres Node] ← Logs alert in N8N_EXECUTION_LOG
  → If score ≥ target:
      [Postgres Node] ← Logs OK status
```

### Workflow 3: New Factory Notification (إشعار تسجيل مصنع جديد)

**Trigger:** Webhook `POST /webhook/factory-registered`  
**Steps:**

```
[Webhook Node] ← Triggered by Flask when Admin adds factory
[Email Node] ← Notifies admin team of new factory registration
[Postgres Node] ← Updates factory status to "active"
```

---

## 4. Technology Stack | التقنيات المستخدمة

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Automation / AI Engine** | n8n (self-hosted) | All workflows, AI matching, scheduling |
| **AI Model** | OpenAI `text-embedding-3-small` | Text vectorization for semantic matching |
| **Vector Logic** | n8n Code Node (JS cosine similarity) | Similarity scoring |
| **Backend API** | Python / Flask | REST endpoints, auth, DB CRUD |
| **Database** | SQL (SQLite → PostgreSQL) | Relational data store |
| **Frontend (Phase 1)** | Streamlit | Analytics dashboard |
| **Frontend (Phase 2)** | React.js | Full platform |
| **Dev Environment** | VS Code | IDE |

---

## 5. n8n ↔ Flask Integration | التكامل بين n8n وFlask

| Action | Flask Does | n8n Does |
|--------|-----------|----------|
| Buyer submits procurement item | Saves to DB, calls `POST /webhook/match-product` | Embeds text → finds matches → logs → responds |
| Admin adds factory | Saves to DB, calls `POST /webhook/factory-registered` | Sends notification email |
| Daily compliance check | Nothing (passive) | Scheduled trigger → calculates score → alerts |

**Webhook communication pattern:**

```python
# Flask calls n8n webhook (example)
import requests

response = requests.post(
    "http://localhost:5678/webhook/match-product",
    json={
        "procurement_id": item_id,
        "procurement_description": description
    },
    timeout=30
)
matches = response.json()["matches"]
```

---

## 6. System Modules | وحدات النظام

### 6.1 Module 1: Factory & Product Registry (سجل المصانع)
_Unchanged from original design — managed via Flask + SQL only_

### 6.2 Module 2: AI Recommendation Engine → **Now an n8n Workflow**
- Old: Python Sentence-Transformers running inline in Flask
- **New:** n8n Workflow #1 (AI Match Flow) called via webhook
- Benefits: Visual editing of AI logic, swap AI models without code changes, built-in retry/error handling

### 6.3 Module 3: Analytics Dashboard
- Score calculation moved to n8n Workflow #2 (scheduled)
- Flask just reads the pre-computed score from DB
- Streamlit displays the result

---

## 7. Authentication & Security | الأمان

| Aspect | Approach |
|--------|----------|
| **User Auth** | Flask session-based (Admin / Buyer) |
| **n8n security** | n8n runs on internal network only (not public-facing) |
| **Webhook security** | Shared secret header `X-Webhook-Secret` validated in n8n |
| **API keys** | OpenAI key stored in n8n Credentials (encrypted) |

---

## 8. Deployment Architecture | معمارية النشر

### Development (Local)
```
localhost:5000  ← Flask API
localhost:8501  ← Streamlit Dashboard
localhost:5678  ← n8n UI + Webhook server
localhost:5432  ← PostgreSQL (or SQLite file)
```

### Production
```
Server / VPS
├── Flask API (Gunicorn + Nginx, port 443)
├── n8n (Docker container, internal port 5678)
├── PostgreSQL (Docker container, internal port 5432)
└── Streamlit → React.js (CDN)
```

**Recommended Docker Compose setup:**
```yaml
services:
  n8n:
    image: n8nio/n8n
    ports: ["5678:5678"]
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - DB_TYPE=postgresdb
    volumes: ["n8n_data:/home/node/.n8n"]

  flask:
    build: .
    ports: ["5000:5000"]
    environment:
      - N8N_WEBHOOK_URL=http://n8n:5678

  postgres:
    image: postgres:15
    volumes: ["pg_data:/var/lib/postgresql/data"]
```

---

## 9. Key Design Decisions | قرارات التصميم

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **AI Engine** | n8n (not Python libs) | Visual editing, no-code AI nodes, easy to modify |
| **Embeddings Model** | OpenAI text-embedding-3-small | High quality, multilingual (Arabic + English) |
| **Workflow trigger** | Webhooks (sync) + Schedule (async) | Sync for matching, async for monitoring |
| **Deployment** | Docker Compose (n8n + Flask + DB) | Easy to self-host, portable |
| **Flask role** | Thin API layer | Just auth + CRUD; delegates AI to n8n |

---

## 10. Future Roadmap | خارطة الطريق

| Phase | Change |
|-------|--------|
| **Phase 2** | React.js frontend, Node.js gateway, n8n remains as AI engine |
| **Phase 3** | Add n8n's built-in Vector Store node (replacing JS cosine similarity) |
| **Phase 3** | RAG workflow in n8n using OpenAI Assistants or Langchain nodes |
| **Phase 3** | n8n PDF parsing workflow to auto-extract factory product data from brochures |
