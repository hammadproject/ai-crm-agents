# AI-Powered CRM — Multi-Agent Backend

A FastAPI backend for a CRM system where six autonomous AI agents handle lead qualification, email intelligence, sales pipeline analysis, customer success monitoring, meeting scheduling, and analytics — coordinated by a central orchestrator.

This is a **backend/API project** (Python + FastAPI + PostgreSQL). There is no bundled frontend — pair it with your own dashboard, or drive it directly through the auto-generated API docs at `/docs`.

---

## What's Actually Here

| Layer | Status |
|---|---|
| FastAPI backend, routers, models, DB schema | ✅ Implemented |
| 6 agent classes + orchestrator wiring | ✅ Implemented (agent logic scaffolded, see below) |
| LLM connection (OpenAI/Anthropic) |  Placeholder — `orchestrator.py` currently uses a mock LLM; wire in a real client before relying on agent output |
| Redis / Celery / message queue |  Referenced in code (`BaseAgent` supports a `redis_client`) but not required to run the API — optional, for scaling agent pub/sub |


If you're picking this up to build on, the honest starting point is: real API + DB + agent skeletons, LLM calls need to be connected.

---

## Architecture

### 6 Agents (`agents/`)

1. **Lead Qualification Agent** — scores incoming leads, enriches contact data, routes high-value prospects
2. **Email Intelligence Agent** — drafts responses, sentiment analysis, categorization
3. **Sales Pipeline Agent** — tracks deal progress, predicts close probability, flags stalled deals
4. **Customer Success Agent** — health scoring, churn risk detection, retention triggers
5. **Meeting Scheduler Agent** — calendar-aware scheduling, meeting prep, follow-up tasks
6. **Analytics Agent** — dashboards, forecasts, performance insights

All six inherit from `agents/base_agent.py`, which provides:
- `think()` — LLM reasoning call
- `use_tool()` — invoke a tool from the agent's toolkit
- `publish_event()` / `subscribe_event()` — inter-agent pub/sub (via Redis if configured)
- `collaborate()` — request another agent to perform a task and await its response
- `log_activity()` — audit trail logging

### Orchestrator (`workflows/orchestrator.py`)

Central coordinator that instantiates all six agents and exposes workflow methods (`process_new_lead`, `process_email`, `analyze_deal`, `monitor_customer`, `schedule_meeting`, `generate_dashboard`) called by the API layer.

### API (`api/`, `main.py`)

REST endpoints grouped by resource (`/api/leads`, `/api/deals`, `/api/customers`, `/api/emails`, `/api/meetings`, `/api/analytics`), plus direct agent-trigger endpoints (`/api/agents/qualify-lead`, etc.) and two webhooks (`/webhooks/email-received`, `/webhooks/form-submission`) for external systems to push events in.

### Database (`database/`)

PostgreSQL schema (`schema.sql`) covering companies, contacts, deals, customers, emails, and meetings, with SQLAlchemy models in `models.py`.

---

## Tech Stack

| Component | Technology |
|---|---|
| API Framework | FastAPI |
| Database | PostgreSQL (SQLAlchemy ORM, Alembic migrations) |
| Agent Orchestration | LangChain-style agent pattern (custom `BaseAgent`) |
| LLM | OpenAI / Anthropic (bring your own API key) |
| Caching / Pub-Sub | Redis (optional) |
| Async Task Queue | Celery (optional, for background agent work at scale) |

---

## Getting Started

### Option A — automated setup (Linux/macOS, requires PostgreSQL installed)

```bash
chmod +x setup.sh
./setup.sh
```

This creates a virtualenv, installs dependencies, creates the `ai_crm` PostgreSQL database, runs table creation, and writes a starter `.env`.

### Option B — manual setup

```bash
# 1. Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure environment
cp .env.example .env
# Edit .env: DATABASE_URL, OPENAI_API_KEY / ANTHROPIC_API_KEY, etc.

# 4. Create the database tables
python -c "from database.models import Base; from database.connection import engine; Base.metadata.create_all(bind=engine)"

# 5. Run the API
python run.py
# or: uvicorn main:app --reload
```

The API is then available at:
- **Server:** http://localhost:8000
- **Interactive docs:** http://localhost:8000/docs
- **Health check:** http://localhost:8000/health

### Connecting a real LLM

`workflows/orchestrator.py` currently initializes a `MockLLM` in `_init_llm()`. Replace it with your provider of choice, e.g.:

```python
from langchain.chat_models import ChatOpenAI
# or: from langchain.chat_models import ChatAnthropic

def _init_llm(self):
    return ChatOpenAI(temperature=0.7, api_key=settings.OPENAI_API_KEY)
```

Until this is wired up, agent `.think()` calls will return a mock response rather than real AI output.

---

## Key API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Service + agent status |
| `POST` | `/api/agents/qualify-lead` | Trigger Lead Qualification Agent on a lead payload |
| `POST` | `/api/agents/analyze-email` | Trigger Email Intelligence Agent |
| `POST` | `/api/agents/analyze-deal/{deal_id}` | Trigger Sales Pipeline Agent on a specific deal |
| `POST` | `/api/agents/monitor-customer/{customer_id}` | Trigger Customer Success Agent |
| `POST` | `/api/agents/schedule-meeting` | Trigger Meeting Scheduler Agent |
| `POST` | `/api/agents/generate-dashboard` | Run Analytics Agent synchronously |
| `POST` | `/webhooks/email-received` | External systems push inbound emails here |
| `POST` | `/webhooks/form-submission` | External systems push new leads here |

Full request/response schemas are available at `/docs` once the server is running.

---

## Environment Variables

See `.env.example` for the full list. The essentials to get running:

```
DATABASE_URL=postgresql://crm_user:crm_password@localhost:5432/ai_crm
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
REDIS_URL=redis://localhost:6379/0   # optional unless using pub/sub or Celery
SECRET_KEY=
```

Per-agent feature flags (`LEAD_QUALIFICATION_ENABLED`, etc.) are also available if you want to selectively disable agents.

---

## What's Not Included (Roadmap)

- Frontend dashboard — this repo is API-only
- Real LLM wiring (currently mocked — see above)
- Celery worker entrypoint (`workflows/worker.py` referenced in docs/scripts but not present — needed only if you want async task distribution at scale)
- Authentication middleware (JWT dependency is in `requirements.txt`, not yet applied to routes)
- Gmail/Calendar OAuth flow implementation (env vars are present; the integration code isn't)

---

## License

MIT.
