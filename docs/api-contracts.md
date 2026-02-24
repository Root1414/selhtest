# API Contracts: Sellh (صله)

> **Updated — 2026-02-24 | n8n-Powered Architecture**

---

## Overview | نظرة عامة

**Flask REST API** handles auth, CRUD, and user-facing data.  
**n8n Webhooks** handle all AI matching and automation workflows.

| Component | Base URL | Purpose |
|-----------|----------|---------|
| Flask API | `http://localhost:5000/api` | Auth, factories, products, procurement |
| n8n Webhooks | `http://localhost:5678/webhook` | AI matching, notifications (internal only) |

**Auth:** Session-based (current) → JWT (future)  
**Format:** JSON

---

## Flask Endpoints — Authentication | المصادقة

### POST /api/auth/login
```json
// Request
{ "username": "string", "password": "string" }

// Response 200
{ "success": true, "user": { "id": 1, "username": "admin_user", "role": "Admin" } }

// Response 401
{ "success": false, "message": "Invalid credentials" }
```

### POST /api/auth/logout
```json
// Response 200
{ "success": true, "message": "Logged out" }
```

---

## Flask Endpoints — Factory Management | إدارة المصانع
> 🔒 Admin only

### GET /api/factories
```json
// Response 200
{
  "factories": [
    { "id": 1, "name": "شركة الأثاث السعودي", "location": "الرياض", "license_number": "SA-2023-0042" }
  ],
  "total": 42
}
```

### POST /api/factories
```json
// Request
{ "name": "string", "location": "string", "license_number": "string", "contact_info": "string" }

// Response 201
{ "success": true, "factory_id": 43 }
```
> ⚡ After saving to DB, Flask calls `POST /webhook/factory-registered` in n8n to trigger notification workflow.

### PUT /api/factories/{id}
```json
// Response 200
{ "success": true, "message": "Factory updated" }
```

---

## Flask Endpoints — Local Products | المنتجات المحلية
> 🔒 Admin only (write), Buyer (read)

### GET /api/products
**Query params:** `category_id`, `factory_id`, `q` (search)

```json
// Response 200
{
  "products": [
    {
      "id": 5, "name": "مكتب خشبي مكتبي",
      "description": "Office wooden desk, 140×70 cm",
      "factory": { "id": 1, "name": "شركة الأثاث السعودي" },
      "category": { "id": 2, "name": "Office Furniture" },
      "price": 850.00
    }
  ],
  "total": 157
}
```

### POST /api/products
```json
// Request
{ "factory_id": 1, "category_id": 2, "name": "string", "description": "string", "price": 850.00 }

// Response 201
{ "success": true, "product_id": 158 }
```

---

## Flask Endpoints — Procurement & AI Matching | المشتريات والمطابقة

### POST /api/procurement/match
> Accessible by **Buyer** role  
> Flask saves the item to DB, then **calls n8n webhook** and returns the AI results.

```json
// Request
{
  "description": "Wooden office desk for executive use",
  "quantity": 10,
  "unit": "piece",
  "origin_country": "Germany",
  "estimated_value": 15000.00
}

// Response 200 (after n8n processes and responds)
{
  "procurement_id": 88,
  "matches": [
    {
      "rank": 1,
      "similarity_score": 0.94,
      "product": { "id": 5, "name": "مكتب خشبي مكتبي", "price": 850.00 },
      "factory": { "id": 1, "name": "شركة الأثاث السعودي", "location": "الرياض" }
    },
    {
      "rank": 2,
      "similarity_score": 0.87,
      "product": { "id": 12, "name": "طاولة مكتبية فاخرة" },
      "factory": { "id": 3, "name": "مصنع الأجهزة الصناعية", "location": "جدة" }
    }
  ],
  "total_matches": 8
}
```

**Internal flow:**
```
Flask: Save PROCUREMENT_ITEM to DB
Flask: POST http://localhost:5678/webhook/match-product
         { "procurement_id": 88, "description": "Wooden office desk..." }
n8n:  OpenAI Embed → Search LOCAL_PRODUCT → Rank → Write AI_MATCH_LOG
n8n:  Respond with matches JSON
Flask: Return matches to Buyer
```

### GET /api/procurement/{id}/matches
```json
// Response 200 — same structure as POST above (reads from AI_MATCH_LOG)
```

---

## Flask Endpoints — Analytics | التحليلات

### GET /api/analytics/local-content-score
> Reads pre-computed score from DB (calculated daily by n8n scheduled workflow)

```json
// Response 200
{
  "local_content_score": 67.4,
  "target_threshold": 60.0,
  "status": "green",
  "items_matched": 234,
  "items_total": 347,
  "last_calculated": "2026-02-24T08:00:00Z",
  "trend": [
    { "month": "2026-01", "score": 61.2 },
    { "month": "2026-02", "score": 67.4 }
  ]
}
```

**Status values:** `"green"` ≥ target | `"yellow"` within 5% | `"red"` below target

---

## n8n Webhook Specs (Internal) | مواصفات n8n الداخلية

> These are called by Flask internally — **not exposed to end users**

### POST /webhook/match-product
**Auth:** Header `X-Webhook-Secret: {shared_secret}`

```json
// Input (from Flask)
{ "procurement_id": 88, "description": "Wooden office desk for executive use" }

// Output (to Flask)
{
  "matches": [
    { "rank": 1, "similarity_score": 0.94, "product_id": 5, "factory_id": 1 }
  ],
  "execution_id": "n8n-exec-12345"
}
```

### POST /webhook/factory-registered
**Auth:** Header `X-Webhook-Secret: {shared_secret}`

```json
// Input (from Flask)
{ "factory_id": 43, "factory_name": "شركة المعادن السعودية", "admin_user": "admin1" }
// n8n sends notification email — no meaningful response body needed
```

---

## Error Codes | رموز الأخطاء

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `INVALID_REQUEST` | Missing or invalid parameters |
| 401 | `UNAUTHORIZED` | Not authenticated |
| 403 | `FORBIDDEN` | Insufficient role |
| 404 | `NOT_FOUND` | Resource not found |
| 502 | `N8N_UNAVAILABLE` | n8n webhook did not respond |
| 500 | `INTERNAL_ERROR` | Server error |

```json
// Error Response Format
{ "success": false, "error_code": "N8N_UNAVAILABLE", "message": "AI service temporarily unavailable. Please retry." }
```
