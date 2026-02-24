# Source Tree Analysis: Sellh (صله)

> **Updated — 2026-02-24 | n8n-Powered Architecture**

---

## Current Project Root (As Found)

```
Sellh صله/
├── docs/                            ← BMAD-generated knowledge base
├── _bmad/                           ← BMAD AI workflow framework
├── _bmad-output/                    ← Planning and implementation artifacts
├── Senior Project Week#6.pdf        ← Weekly progress report
└── مستند متطلبات الأعمال.docx      ← Business requirements document
```

---

## Recommended Source Structure (n8n Architecture) | الهيكل المقترح

```
Sellh صله/
│
├── app/                             ← Flask Application
│   ├── __init__.py                  ← App factory, Flask init, Blueprint registration
│   ├── config.py                    ← Configuration (loads .env)
│   │
│   ├── models/                      ← SQLAlchemy models
│   │   ├── user.py                  ← USERS table
│   │   ├── factory.py               ← FACTORY table
│   │   ├── local_product.py         ← LOCAL_PRODUCT table
│   │   ├── procurement_item.py      ← PROCUREMENT_ITEM table
│   │   ├── ai_match_log.py          ← AI_MATCH_LOG table
│   │   └── n8n_execution_log.py     ← N8N_EXECUTION_LOG table
│   │
│   ├── routes/                      ← Flask Blueprint route handlers
│   │   ├── auth.py                  ← POST /auth/login, POST /auth/logout
│   │   ├── factories.py             ← GET/POST/PUT /factories
│   │   ├── products.py              ← GET/POST /products
│   │   ├── procurement.py           ← POST /procurement/match ⭐
│   │   └── analytics.py             ← GET /analytics/local-content-score
│   │
│   ├── services/                    ← Business logic
│   │   ├── n8n_client.py            ← Calls n8n webhooks ⭐ (replaces ai_matching.py)
│   │   ├── factory_service.py       ← Factory CRUD
│   │   ├── product_service.py       ← Product CRUD
│   │   └── analytics_service.py     ← Reads pre-computed scores from DB
│   │
│   └── middleware/
│       └── auth.py                  ← @require_admin, @require_buyer decorators
│
├── n8n/                             ← n8n Workflow Definitions ⭐ NEW
│   ├── workflows/
│   │   ├── ai-match-workflow.json   ← Webhook → OpenAI Embed → Match → Log ⭐
│   │   ├── compliance-monitor.json  ← Daily schedule → Score calc → Alert email ⭐
│   │   └── factory-notification.json← Webhook → Send email notification ⭐
│   └── README.md                    ← How to import these workflows into n8n
│
├── dashboard/                       ← Streamlit Dashboard (Phase 1 Frontend)
│   ├── main.py                      ← Entry point ⭐
│   └── pages/
│       ├── match.py                 ← Procurement matching page
│       └── analytics.py             ← Compliance score page (Green/Yellow/Red)
│
├── database/                        ← SQL database files
│   ├── schema.sql                   ← Full schema (all tables including N8N_EXECUTION_LOG) ⭐
│   ├── migrations/                  ← Future schema changes
│   └── seed_data.sql                ← Sample factories/products for dev
│
├── tests/
│   ├── test_api.py                  ← Flask endpoint tests
│   ├── test_n8n_client.py           ← Mock n8n webhook call tests ⭐
│   └── conftest.py                  ← Pytest fixtures
│
├── main.py                          ← Flask app entry point ⭐
├── docker-compose.yml               ← n8n + Flask + PostgreSQL ⭐
├── Dockerfile                       ← Flask container
├── requirements.txt                 ← Python dependencies (no torch/transformers!)
├── .env.example                     ← Environment variables template
└── README.md
```

---

## Key Changes from Original Structure | التغييرات الرئيسية

| Before (Sentence-Transformers) | After (n8n) |
|-------------------------------|-------------|
| `ai/embedder.py` | ❌ Removed |
| `ai/matcher.py` | ❌ Removed |
| `app/services/ai_matching.py` | ✅ Replaced by `app/services/n8n_client.py` |
| *(no workflow folder)* | ✅ Added `n8n/workflows/` |
| *(no docker-compose)* | ✅ Added `docker-compose.yml` |
| `requirements.txt` with `torch`, `sentence-transformers` | ✅ Simplified — only `requests` needed |

---

## Critical Files | الملفات الحرجة

| File | Purpose | Priority |
|------|---------|----------|
| `main.py` | Flask app entry point | ⭐ Critical |
| `app/services/n8n_client.py` | Calls n8n webhooks for AI tasks | ⭐ Critical |
| `n8n/workflows/ai-match-workflow.json` | The AI matching pipeline in n8n | ⭐ Critical |
| `n8n/workflows/compliance-monitor.json` | Daily compliance automation | ⭐ Critical |
| `database/schema.sql` | Full database schema | ⭐ Critical |
| `docker-compose.yml` | Full stack deployment | 🔸 High |
| `dashboard/main.py` | Streamlit entry point | 🔸 High |
| `.env` | n8n URL, secret, OpenAI key config | 🔸 High |

---

## n8n Client Service Example | مثال على خدمة n8n

```python
# app/services/n8n_client.py
import requests
import os

N8N_BASE_URL = os.getenv("N8N_BASE_URL", "http://localhost:5678")
N8N_WEBHOOK_SECRET = os.getenv("N8N_WEBHOOK_SECRET", "")

def call_match_product(procurement_id: int, description: str) -> dict:
    """Call n8n AI matching webhook and return matches."""
    response = requests.post(
        f"{N8N_BASE_URL}/webhook/match-product",
        json={"procurement_id": procurement_id, "description": description},
        headers={"X-Webhook-Secret": N8N_WEBHOOK_SECRET},
        timeout=30
    )
    response.raise_for_status()
    return response.json()

def notify_factory_registered(factory_id: int, factory_name: str) -> None:
    """Trigger n8n notification workflow after factory registration."""
    requests.post(
        f"{N8N_BASE_URL}/webhook/factory-registered",
        json={"factory_id": factory_id, "factory_name": factory_name},
        headers={"X-Webhook-Secret": N8N_WEBHOOK_SECRET},
        timeout=10
    )
```

---

## Integration Flow | سير التكامل

```
dashboard/main.py
    └── HTTP → Flask API (localhost:5000)
                    └── app/services/n8n_client.py
                            └── POST → n8n (localhost:5678/webhook/match-product)
                                        └── OpenAI API (cloud)
                                        └── SQL Database (localhost)
                                        └── Returns matches to Flask
                    └── Flask returns matches to Dashboard
```
