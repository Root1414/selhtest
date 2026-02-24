# Development Guide: Sellh (صله)

> **Updated — 2026-02-24 | n8n-Powered Architecture**

---

## Prerequisites | المتطلبات المسبقة

| Tool | Version | Purpose |
|------|---------|---------|
| Python | 3.10+ | Flask backend |
| pip | Latest | Python packages |
| Node.js | 18+ | Required by n8n |
| n8n | Latest (npm or Docker) | AI workflows & automation |
| VS Code | Latest | IDE |
| SQLite / PostgreSQL | Built-in / 15+ | Database |
| Docker (optional) | Latest | Containerized deployment |

---

## Setup: Flask Backend | إعداد Flask

### 1. Create virtual environment
```bash
cd "Sellh صله"
python -m venv venv
venv\Scripts\activate        # Windows
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
```

**requirements.txt:**
```
flask>=2.3.0
requests>=2.31.0          # For calling n8n webhooks
sqlalchemy>=2.0.0
bcrypt>=4.0.0
python-dotenv>=1.0.0
streamlit>=1.28.0
pandas>=2.0.0
pytest>=7.4.0
```

> 📝 **Note:** No `sentence-transformers` or `torch` needed — AI is handled by n8n!

### 3. Configure environment
```bash
cp .env.example .env
```

**.env:**
```env
# Flask
FLASK_APP=main.py
FLASK_ENV=development
FLASK_DEBUG=1
SECRET_KEY=your-secret-key

# Database
DATABASE_URL=sqlite:///sellh.db

# n8n integration
N8N_BASE_URL=http://localhost:5678
N8N_WEBHOOK_SECRET=your-shared-secret-here

# App
LOCAL_CONTENT_TARGET=60.0
```

### 4. Initialize database
```bash
sqlite3 sellh.db < database/schema.sql
```

### 5. Run Flask
```bash
flask run --port=5000
```
📍 `http://localhost:5000`

---

## Setup: n8n | إعداد n8n

### Option A — Install via npm (Simple)
```bash
npm install -g n8n
n8n start
```
📍 n8n UI at `http://localhost:5678`

### Option B — Run via Docker (Recommended for production)
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

---

## Import n8n Workflows | استيراد سير العمل

Once n8n is running:

1. Open `http://localhost:5678`
2. Create an account (first time)
3. Go to **Workflows → Import from File**
4. Import each JSON file from `n8n/workflows/`:
   - `n8n/workflows/ai-match-workflow.json`
   - `n8n/workflows/compliance-monitor.json`
   - `n8n/workflows/factory-notification.json`

---

## Configure n8n Credentials | إعداد البيانات السرية في n8n

In n8n UI → **Credentials → Add Credential**:

### OpenAI Credential
- Type: OpenAI API
- API Key: `sk-...` (your OpenAI key)

### PostgreSQL Credential (if using PostgreSQL)
- Host: `localhost`
- Port: `5432`
- Database: `sellh`
- User / Password: your DB credentials

---

## Activate Workflows | تفعيل سير العمل

For each imported workflow:
1. Open the workflow
2. Click the **Active** toggle (top right)
3. The Webhook URL will be shown — copy it to your `.env` if needed

**Webhook URLs (defaults):**
```
POST http://localhost:5678/webhook/match-product
POST http://localhost:5678/webhook/factory-registered
```

---

## Run the Full Stack | تشغيل النظام كاملاً

Open 3 terminals:

```bash
# Terminal 1: n8n
n8n start

# Terminal 2: Flask API
venv\Scripts\activate
flask run --port=5000

# Terminal 3: Streamlit Dashboard
streamlit run dashboard/main.py
```

| Service | URL |
|---------|-----|
| n8n Editor | http://localhost:5678 |
| Flask API | http://localhost:5000 |
| Streamlit Dashboard | http://localhost:8501 |

---

## Testing | الاختبارات

### Test Flask API
```bash
pytest tests/ -v
```

### Test n8n Webhook Manually
```bash
curl -X POST http://localhost:5678/webhook/match-product \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: your-shared-secret-here" \
  -d '{"procurement_id": 1, "description": "Wooden office desk for executive use"}'
```

### Test the Full Matching Flow (End-to-End)
```bash
curl -X POST http://localhost:5000/api/procurement/match \
  -H "Content-Type: application/json" \
  -d '{"description": "Office desk wooden", "quantity": 5, "unit": "piece"}'
```

---

## Docker Compose (Full Stack) | نشر كامل بـ Docker

```yaml
# docker-compose.yml
version: "3.8"
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports: ["5678:5678"]
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=your_password
    volumes:
      - n8n_data:/home/node/.n8n

  flask:
    build: .
    restart: unless-stopped
    ports: ["5000:5000"]
    environment:
      - N8N_BASE_URL=http://n8n:5678
      - DATABASE_URL=postgresql://sellh:password@postgres/sellh
    depends_on: [n8n, postgres]

  postgres:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_DB: sellh
      POSTGRES_USER: sellh
      POSTGRES_PASSWORD: password
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./database/schema.sql:/docker-entrypoint-initdb.d/schema.sql

volumes:
  n8n_data:
  pg_data:
```

```bash
docker-compose up -d
```

---

## Troubleshooting | استكشاف الأخطاء

| Problem | Solution |
|---------|----------|
| n8n not starting | Check Node.js ≥ 18 is installed |
| Webhook not responding | Ensure workflow is **Active** in n8n UI |
| 502 N8N_UNAVAILABLE | n8n service is down — run `n8n start` |
| OpenAI error in n8n | Check API key in n8n Credentials |
| DB connection fails | Verify DATABASE_URL in `.env` |
| Port 5678 in use | Stop other n8n instance or change port |
