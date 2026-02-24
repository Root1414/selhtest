# Project Overview: Sellh (صله)

> **Updated — 2026-02-24 | n8n-Powered Architecture**

---

## Executive Summary | الملخص التنفيذي

**Sellh (صله)** is an AI-powered Saudi local content compliance platform that helps organizations meet Vision 2030 localization targets. The system uses **n8n** as its AI and workflow automation engine to intelligently match imported procurement items with Saudi-made alternatives, enabling compliance monitoring, automated alerts, and smart supplier ranking.

---

## Problem Statement | بيان المشكلة

1. **Discovering local alternatives** — Procurement teams can't find Saudi-made substitutes for imported items.
2. **Semantic mismatch** — "Wooden Desk" may exist locally as "Office Table"; keyword search fails.
3. **Compliance risk** — Without real-time visibility, organizations fall below local content targets.
4. **Manual burden** — No automated monitoring or alerting for compliance thresholds.

---

## Solution with n8n | الحل باستخدام n8n

### Why n8n? | لماذا n8n؟

| Feature | Benefit |
|---------|---------|
| Visual workflow builder | Build and modify AI pipelines without writing Python ML code |
| Built-in AI nodes | OpenAI Embeddings, text processing, vector search — ready to use |
| Webhook triggers | Flask calls n8n synchronously when a buyer submits a procurement item |
| Scheduler | Daily automated compliance checks with email alerts |
| Self-hosted | Data stays on your own server (no cloud vendor lock-in) |

---

## System Modules | وحدات النظام

### Module 1: Factory & Product Registry (سجل المصانع)
- Admin-managed database of Saudi factories (name, location, license)
- Categorized local product catalogue
- Managed via Flask + SQL

### Module 2: AI Matching Engine → **n8n Workflow**
- **Old:** Python Sentence-Transformers running in Flask code
- **New:** n8n Workflow triggered via webhook
  - Receives procurement item description
  - OpenAI Embeddings → vectorize text
  - Cosine similarity vs all LOCAL_PRODUCT entries
  - Ranks by: match score + cost + proximity
  - Logs results to `AI_MATCH_LOG`

### Module 3: Compliance Monitoring → **n8n Scheduled Workflow**
- Daily n8n schedule calculates local content score
- Traffic-light alert (Green/Yellow/Red)
- Auto-email when score drops below target

---

## Tech Stack | التقنيات

| Layer | Current (Phase 1) | Future (Phase 2+) |
|-------|-------------------|-------------------|
| **Workflow/AI** | n8n (self-hosted) | n8n + Vector Store node |
| **AI Model** | OpenAI text-embedding-3-small | OpenAI Assistants / RAG |
| **Backend** | Python / Flask | Flask / Microservices |
| **Database** | SQL (SQLite/PostgreSQL) | PostgreSQL + n8n Vector Store |
| **Frontend** | Streamlit | React.js |
| **Notifications** | n8n Email node | n8n Email + SMS + Slack |

---

## Architecture Diagram | مخطط المعمارية

```
[Buyer/Admin] → [Streamlit / Flask API]
                         │
              ┌──────────▼──────────┐
              │   Flask REST API    │ ← Auth, CRUD, routes
              └──────────┬──────────┘
                         │ Webhook calls
              ┌──────────▼──────────┐
              │       n8n           │ ← All AI + automation
              │  ┌──────────────┐   │
              │  │ Match Workflow│  │ ← Triggered per request
              │  │ Compliance   │  │ ← Daily schedule
              │  │ Notifications│  │ ← Event-driven
              │  └──────────────┘  │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │    SQL Database     │
              └─────────────────────┘
```

---

## Development Phases | مراحل التطوير

| Phase | Status | Stack |
|-------|--------|-------|
| **Phase 1** (Academic) | 🟡 In Progress | Flask + n8n + SQLite + Streamlit |
| **Phase 2** (Post-grad) | 🔜 Planned | + React.js + PostgreSQL + Docker Compose |
| **Phase 3** (Scale) | 🔮 Future | + n8n Vector Store + RAG + Microservices |

---

## Links | التوثيق التفصيلي

- [Architecture](./architecture.md) — Full system design and n8n workflow definitions
- [Development Guide](./development-guide.md) — Setting up Flask + n8n locally
- [API Contracts](./api-contracts.md) — Flask endpoints and n8n webhook specs
- [Data Models](./data-models.md) — Database schema
- [Source Tree Analysis](./source-tree-analysis.md) — Project folder structure
