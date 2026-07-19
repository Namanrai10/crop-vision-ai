# CropVision AI

Satellite-based crop disease detection and advisory platform for smallholder farmers,
agronomists, and agricultural cooperatives.

**Live demo credentials**
- Farmer: `farmer@cropvision.ai` / `farmer123`
- Agronomist: `agronomist@cropvision.ai` / `agro123`
- Admin: `admin@cropvision.ai` / `admin123`

---

## Architecture (30-second read)

```
              ┌─────────────────────┐
   n8n  ────► │  /api/n8n/ingest    │  (webhook)
              └──────────┬──────────┘
                         ▼
              ┌─────────────────────┐    ┌──────────────────┐
              │  Async job queue    │───►│  vision.py       │  Claude Sonnet 4.5
              │  (BullMQ-shaped     │    │  (image ➝ JSON)  │  via emergent
              │   in-process)       │    └────────┬─────────┘  integrations
              └─────────────────────┘             │
                                                  ▼
                                        ┌───────────────────┐
                                        │  langgraph_agent  │  stateful graph:
                                        │  (multi-agent     │  diagnose →
                                        │   reasoning)      │  severity →
                                        └────────┬──────────┘  advisory →
                                                 │             confidence gate →
                                                 ▼             escalation
                                        ┌───────────────────┐
                                        │ MongoDB (detects, │
                                        │ alerts, audit)    │
                                        └────────┬──────────┘
                                                 ▼
              ┌─────────────────────┐    ┌───────────────────┐
              │ React + TS + Tailwind│◄───│  FastAPI /api/*   │
              │ (i18n, framer, chart)│    │  JWT+RBAC+audit   │
              └─────────────────────┘    └───────────────────┘
```

### Responsibility boundary — n8n vs LangGraph
| | n8n | LangGraph |
| --- | --- | --- |
| Job | orchestration + integrations | reasoning |
| Owns | Webhook trigger, retries, WhatsApp/SMS/email delivery, escalation notification, dead-letter | Disease refinement, severity scoring, bilingual advisory generation, confidence-gated retry loop, escalation decision |
| Files | `/app/n8n_workflow.json` | `/app/backend/langgraph_agent.py` |

The FastAPI backend implements the **same orchestration natively** (via `job_queue.py`) so
the full pipeline runs end-to-end in this environment even without a running n8n instance.

---

## Backend

- FastAPI + Motor + MongoDB (2dsphere geo index on `fields.location`)
- JWT access + refresh token rotation, RBAC (`farmer` / `agronomist` / `admin`)
- Rate limiting on auth endpoints
- Structured audit log for every state-changing action
- In-process async job queue with retry + dead-letter (drop-in replacement point for BullMQ)
- Observability: `/api/health`, `/api/metrics`, `/api/admin/pipeline`

Files:
- `server.py` — API layer + queue wiring
- `models.py` — Pydantic models
- `auth.py` — JWT + password hashing + RBAC
- `vision.py` — Claude Sonnet 4.5 vision inference (image ➝ JSON) + offline fallback
- `langgraph_agent.py` — stateful multi-agent reasoning graph
- `job_queue.py` — worker pool + retry/dead-letter
- `seed.py` — sample data

## Frontend

- React 19 + React Router 7 + Tailwind + framer-motion + recharts + sonner
- Bilingual (English / Hindi) via `react-i18next` — no hardcoded strings
- Auth flows fully wired to backend (login, refresh rotation, role-based nav)
- Skeleton/empty/error states everywhere
- `data-testid` on every interactive + informational element

Pages:
- `/` — Cinematic landing page (scroll-driven, spectral visuals)
- `/login`, `/signup` — Auth
- `/dashboard` — Live operational dashboard (fleet metrics, trend chart, field cards, alert feed)
- `/fields` — Field inventory
- `/fields/:id` — Detail: 14-day trend, detection history, LangGraph reasoning trace
- `/alerts` — Full alert feed
- `/queue` — Agronomist escalation queue (RBAC-gated)
- `/admin` — Pipeline health + audit trail (RBAC-gated)

---

## Run locally

```bash
# Backend
cd backend
pip install -r requirements.txt
python seed.py                     # seed users + fields + detections
sudo supervisorctl restart backend # or: uvicorn server:app --reload --port 8001

# Frontend
cd frontend
yarn
yarn start
```

Environment variables (backend/.env):
- `MONGO_URL`, `DB_NAME`
- `EMERGENT_LLM_KEY` (Claude Sonnet 4.5 via emergentintegrations)
- `JWT_SECRET`, `JWT_ALGORITHM`, `ACCESS_TOKEN_MINUTES`, `REFRESH_TOKEN_DAYS`

Frontend (frontend/.env): `REACT_APP_BACKEND_URL`.

## n8n workflow

Import `/app/n8n_workflow.json`. Required env vars in n8n:
```
CROPVISION_API           # e.g. https://your-host
CROPVISION_TOKEN         # bearer token for polling
CROPVISION_INGEST_KEY    # matches backend
TWILIO_SID, TWILIO_WHATSAPP_FROM
ALERT_FROM_EMAIL, AGRONOMIST_EMAIL, SLACK_ALERTS_WEBHOOK
```

## Placeholder / stub flags

| Item | Status |
| --- | --- |
| Claude Sonnet 4.5 vision inference | **Fully wired** (real API) with graceful offline fallback |
| LangGraph reasoning | **Fully wired** (real LLM calls, deterministic fallback prompts) |
| SMS / WhatsApp delivery | **Mocked** — alerts persisted to `alerts` collection with `channel:"dashboard"` and shown in real time. Wire Twilio via `n8n_workflow.json`. |
| n8n orchestration | **Exportable JSON provided** — same pipeline runs natively in FastAPI so app is end-to-end functional without n8n |
| Real satellite tile pipeline | **Sample-image based** — inference endpoint accepts any base64 crop image (JPEG/PNG/WEBP) |

All other flows (auth, RBAC, job queue, retry, audit, i18n, RBAC-gated routes, dashboards, historical trends) are fully implemented, not stubbed.
