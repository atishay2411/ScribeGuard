# ScribeGuard — New Team Handoff Guide

**Project:** ScribeGuard — Agentic AI Clinical Documentation Platform  
**Prepared for:** Incoming development team  
**Last Updated:** May 2026

---

## Table of Contents

1. [What This Project Is](#1-what-this-project-is)
2. [Where We Left Off](#2-where-we-left-off)
3. [Prerequisites](#3-prerequisites)
4. [Environment Setup — Step by Step](#4-environment-setup--step-by-step)
5. [Running the Application](#5-running-the-application)
6. [Project Structure](#6-project-structure)
7. [Key Concepts to Understand First](#7-key-concepts-to-understand-first)
8. [The Agent Pipeline](#8-the-agent-pipeline)
9. [Database & Migrations](#9-database--migrations)
10. [Configuration Reference](#10-configuration-reference)
11. [API Quick Reference](#11-api-quick-reference)
12. [How to Add a New Agent](#12-how-to-add-a-new-agent)
13. [Known Issues & Gotchas](#13-known-issues--gotchas)
14. [Future Roadmap (Layers 2 & 3)](#14-future-roadmap-layers-2--3)
15. [Contacts & Resources](#15-contacts--resources)

---

## 1. What This Project Is

ScribeGuard is a **multi-agent AI clinical documentation platform** built on top of OpenMRS. Its core job is:

1. A physician records a patient encounter through their browser microphone.
2. Seven specialized AI agents run in sequence: the audio is transcribed (Whisper), a structured SOAP note is generated (GPT-4), medications and clinical entities are extracted, and the physician reviews and approves the result.
3. Only after explicit physician approval does the system write back to the patient's OpenMRS record via FHIR R4 (creating an Encounter, clinical-note Observation, Conditions, AllergyIntolerance, MedicationRequests, and vital-sign Observations).

The key design philosophy is **physician-in-the-loop**: the system cannot write anything to OpenMRS without an explicit approval action. Every agent run, edit, and approval is logged in an immutable audit trail.

---

## 2. Where We Left Off

### Completed (Layer 1 — MVP)

- Full audio capture pipeline (browser MediaRecorder → FastAPI → disk storage)
- Whisper transcription with quality scoring
- GPT-4 SOAP note generation with version history
- Medication, allergy, condition, vital sign, and follow-up extraction
- Physician review workflow: open review → edit sections → explicit approve/revert
- OpenMRS FHIR R4 submission (Encounter, Observation, AllergyIntolerance, Condition, MedicationRequest)
- Simulation mode (`OPENMRS_SIMULATE=true`) for testing without a live OpenMRS server
- Full audit trail with queryable timeline
- React frontend: Dashboard (recording), Encounter Workspace (review), History, Patient directory

### Not Yet Started (Future Layers)

- **Layer 2:** Prescription Reconciliation Agent — compare extracted medications against existing OpenMRS MedicationRequests to detect duplicates/conflicts
- **Layer 3:** Dosage Anomaly Detector — validate prescriptions against clinical databases (OpenFDA, DrugBank)
- E-prescribing integration
- Mobile app (was descoped; web-only approach is current)
- Patient reminder system triggered by audit events
- HIPAA-grade audit log export

---

## 3. Prerequisites

Install these before anything else:

| Tool | Version | Purpose | Install |
|------|---------|---------|---------|
| Python | 3.11+ | Backend runtime | [python.org](https://python.org) |
| Node.js | 18+ | Frontend runtime | [nodejs.org](https://nodejs.org) |
| Docker Desktop | Latest | PostgreSQL container | [docker.com](https://docker.com) |
| Git | Any | Version control | Usually pre-installed |
| OpenAI API Key | — | Whisper + GPT-4 | [platform.openai.com](https://platform.openai.com) |
| OpenMRS (optional) | 2.x + FHIR2 module | EHR integration | See note below |

**OpenMRS note:** For development and testing, you do NOT need a live OpenMRS instance. Set `OPENMRS_SIMULATE=true` in your `.env` file and the OpenMRS Integration Agent will simulate all FHIR writes with stable fake UUIDs.

If you want a real OpenMRS instance, run the OpenMRS Reference Application Docker image:
```bash
docker run -d -p 8080:8080 openmrs/openmrs-reference-application-distro:latest
```
Then wait ~5 minutes for it to fully start. Default credentials: `Admin` / `Admin123`.

---

## 4. Environment Setup — Step by Step

### Step 1 — Clone the repository

```bash
git clone <repo-url>
cd ScribeGuard
```

### Step 2 — Start the database

```bash
docker compose up -d db
```

This starts PostgreSQL 16 on port 5432 with:
- **User:** `scribeguard`
- **Password:** `scribeguard`
- **Database:** `scribeguard`

Verify it's running:
```bash
docker ps | grep scribeguard_db
```

### Step 3 — Set up the backend

```bash
cd backend

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate       # macOS/Linux
# .\venv\Scripts\activate      # Windows

# Install all Python dependencies
pip install -r requirements.txt
```

### Step 4 — Configure environment variables

Create the file `backend/.env` with the following content (fill in your own values):

```env
# ── Database ──────────────────────────────────────────────────────────
DATABASE_URL=postgresql://scribeguard:scribeguard@localhost:5432/scribeguard

# ── AI Provider ───────────────────────────────────────────────────────
# Use "openai" for cloud (requires OPENAI_API_KEY)
# Use "local" for free on-device inference (requires Ollama installed)
SERVICE_PROVIDER=openai
OPENAI_API_KEY=sk-your-key-here

# ── OpenMRS Integration ───────────────────────────────────────────────
FHIR_SERVER=http://localhost:8080/openmrs/ws/fhir2/R4
OPENMRS_USER=Admin
OPENMRS_PASSWORD=Admin123

# Set to true to skip real OpenMRS calls (great for development)
OPENMRS_SIMULATE=true

# ── Agent Runtime ─────────────────────────────────────────────────────
AGENT_MAX_RETRIES=2
AGENT_RETRY_BASE_DELAY_SECONDS=1.5
AGENT_AUDIT_ENABLED=true

# ── Storage ───────────────────────────────────────────────────────────
AUDIO_STORAGE_DIR=./audio_store

# ── CORS (must include your frontend URL) ─────────────────────────────
CORS_ORIGINS=http://localhost:5173,http://localhost:5174
```

### Step 5 — Run database migrations

```bash
# From the backend/ directory, with venv activated
alembic upgrade head
```

If you hit migration issues, you can alternatively use the convenience script:
```bash
python create_tables.py
```

Verify tables were created:
```bash
python -c "from app.db.database import engine; from sqlalchemy import inspect; print(inspect(engine).get_table_names())"
```

You should see 13+ tables including `encounters`, `transcripts`, `soap_notes`, etc.

### Step 6 — Set up the frontend

```bash
cd ../frontend   # from project root: cd frontend

npm install
```

That's it. Vite handles everything else at dev time.

---

## 5. Running the Application

You need two terminal windows running simultaneously.

### Terminal 1 — Backend API

```bash
cd backend
source venv/bin/activate
uvicorn app.main:app --reload --port 8000
```

The API is now running at: http://localhost:8000  
Swagger UI (interactive API docs): http://localhost:8000/docs

### Terminal 2 — Frontend

```bash
cd frontend
npm run dev
```

The UI is now running at: http://localhost:5173

### Verifying Everything Works

1. Open http://localhost:5173 — you should see the ScribeGuard dashboard
2. Open http://localhost:8000/health — should return `{"status": "ok"}`
3. Open http://localhost:8000/docs — Swagger UI with all endpoints

### Docker (Alternative — runs both backend + DB)

If you prefer to run everything via Docker:

```bash
# Make sure backend/.env is configured first
docker compose up --build
```

This starts PostgreSQL + the FastAPI backend. You still need to run the frontend separately with `npm run dev`.

---

## 6. Project Structure

```
ScribeGuard/
├── backend/
│   ├── app/
│   │   ├── agents/                  # All AI agents
│   │   │   ├── base.py              # Agent abstract class + AgentResult
│   │   │   ├── context.py           # AgentContext (shared state per run)
│   │   │   ├── intake.py            # EncounterIntakeAgent
│   │   │   ├── transcription.py     # TranscriptionAgent (Whisper)
│   │   │   ├── note_generation.py   # ClinicalNoteGenerationAgent (GPT-4 SOAP)
│   │   │   ├── medication_extraction.py  # MedicationExtractionAgent
│   │   │   ├── physician_review.py  # PhysicianReviewAgent (approval gate)
│   │   │   ├── audit.py             # AuditTraceabilityAgent
│   │   │   ├── prompts/             # Version-pinned AI prompts (DO NOT edit casually)
│   │   │   └── openmrs/             # OpenMRS sub-agents
│   │   │       ├── auth.py
│   │   │       ├── patient_context.py
│   │   │       ├── encounter_mapper.py
│   │   │       ├── note_writer.py
│   │   │       ├── verifier.py
│   │   │       └── integration.py   # Composite orchestrator for OpenMRS
│   │   ├── orchestrator/
│   │   │   ├── orchestrator.py      # THE central coordinator — read this first
│   │   │   └── registry.py          # Agent registry
│   │   ├── models/                  # SQLAlchemy ORM models (one per table)
│   │   ├── repositories/            # Data access layer (never use ORM directly in agents)
│   │   ├── routers/                 # FastAPI HTTP routes
│   │   ├── clients/                 # AI provider abstraction (OpenAI / local)
│   │   ├── openmrs/                 # FHIR R4 HTTP client
│   │   ├── schemas/                 # Pydantic v2 API schemas
│   │   ├── db/database.py           # SQLAlchemy engine + session factory
│   │   ├── config.py                # All configuration (read from .env)
│   │   └── main.py                  # FastAPI app bootstrap + router registration
│   ├── alembic/                     # Database migrations
│   ├── requirements.txt
│   ├── Dockerfile
│   └── create_tables.py             # Convenience: runs create_all (use alembic in prod)
├── frontend/
│   └── src/
│       ├── pages/                   # Full-page React components
│       │   ├── Dashboard.tsx        # Recording interface
│       │   ├── EncounterWorkspace.tsx  # Main review interface
│       │   └── History.tsx          # Past encounters
│       ├── components/              # Reusable UI components
│       │   ├── AgentTimeline.tsx    # Live pipeline visualization
│       │   ├── SoapEditor.tsx       # Editable SOAP note form
│       │   ├── MedicationPanel.tsx
│       │   └── EntityPanels.tsx     # Allergies, Conditions, Vitals, Follow-ups
│       ├── api/                     # Typed HTTP client (fetch wrapper)
│       │   ├── types.ts             # TypeScript interfaces mirroring backend schemas
│       │   ├── client.ts            # Base HTTP client
│       │   └── encounters.ts        # All encounter-related API calls
│       └── hooks/
│           └── useLiveCaption.ts    # Audio capture hook
├── docker-compose.yml
├── ARCHITECTURE.md                  # Deep-dive engineering reference (read this!)
├── README.md
├── Documentation/                   # This folder — project docs for new team
└── test_transcripts/                # Sample audio files for testing
```

---

## 7. Key Concepts to Understand First

Before diving into the code, read these files in this order:

1. **`README.md`** — 5-minute overview
2. **`ARCHITECTURE.md`** — The engineering bible. Explains every design decision.
3. **`backend/app/orchestrator/orchestrator.py`** — The central coordinator. Everything flows through here.
4. **`backend/app/agents/base.py`** — The `Agent` abstract class every agent implements.
5. **`backend/app/agents/context.py`** — `AgentContext`: the shared state bundle passed into every agent.

### The Most Important Rules

- **Only the orchestrator mutates `encounter.processing_stage`.** Individual agents may set transient flags but cross-stage transitions always go through `orchestrator.py`.
- **The `PhysicianReviewAgent` is the approval gate.** OpenMRS submission is blocked until `encounter.status == 'approved'`. This is enforced server-side; the frontend cannot bypass it.
- **Agents never talk to the database directly.** They use `ctx.transcripts`, `ctx.soap_notes`, etc. — lazy repository accessors on `AgentContext`.
- **Prompts are version-pinned.** Do not edit files in `agents/prompts/` without incrementing the version constant in the corresponding agent. This ensures reproducibility.

---

## 8. The Agent Pipeline

The complete workflow is:

```
Browser Mic
    │
    ▼
POST /encounters/{id}/intake?auto_run=true
    │
    ▼
[1] EncounterIntakeAgent     → validates audio, saves to disk, sets encounter metadata
    │
    ▼
[2] TranscriptionAgent       → Whisper call → raw text → quality scoring → saves Transcript
    │
    ▼
[3] ClinicalNoteGenerationAgent → GPT-4 SOAP prompt → JSON → saves versioned SoapNote
    │
    ▼
[4] MedicationExtractionAgent → parses Plan section → saves structured Medications
    │
    ▼ (auto-pipeline stops here; physician review is a manual gate)
    │
PHYSICIAN REVIEWS IN UI
    │
    ├── PhysicianReviewAgent.open_review()
    ├── PhysicianReviewAgent.edit(section, text)   ← zero or more edits
    └── PhysicianReviewAgent.approve()             ← required before submission
    │
    ▼
POST /encounters/{id}/submit
    │
    ▼
[6] OpenMRSIntegrationAgent  → authenticate → resolve patient → map to FHIR → write → verify
    │
    ▼
[7] AuditTraceabilityAgent   → aggregates event stream → queryable timeline
```

**Encounter `processing_stage` values (in order):**
```
created → audio_received → transcribing → transcribed →
generating_soap → soap_drafted → extracting_meds →
ready_for_review → in_review → approved →
submitting → submitted
(failed at any step)
```

---

## 9. Database & Migrations

### Schema Overview

| Table | Purpose |
|-------|---------|
| `encounters` | Root record for each patient visit |
| `transcripts` | Whisper output + quality signals |
| `soap_notes` | Versioned SOAP drafts (only one `is_current=true` per encounter at a time) |
| `medications` | Structured drug entities |
| `allergies` | Allergy records |
| `conditions` | Diagnosis records with ICD-10/SNOMED |
| `vital_signs` | Vital observations |
| `follow_ups` | Follow-up instructions |
| `physician_edits` | Per-section edit log |
| `physician_approvals` | Approval records |
| `submission_records` | OpenMRS write attempts + returned UUIDs |
| `agent_runs` | One row per agent invocation |
| `audit_events` | Immutable event stream (never delete from this) |
| `patient_contexts` | Cached OpenMRS chart snapshots |

### Migration Commands

```bash
# Apply all pending migrations (do this after pulling new code)
alembic upgrade head

# Check current migration version
alembic current

# Create a new migration after changing a model
alembic revision --autogenerate -m "describe_your_change"

# Rollback one migration
alembic downgrade -1
```

### Resetting the Database (Development Only)

```bash
# WARNING: destroys all data
python reset_db.py
alembic upgrade head
```

---

## 10. Configuration Reference

All configuration lives in `backend/app/config.py` and is read from `backend/.env`.

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `DATABASE_URL` | — | Yes | PostgreSQL connection string |
| `SERVICE_PROVIDER` | `openai` | No | `openai` or `local` |
| `OPENAI_API_KEY` | `""` | If using OpenAI | Your OpenAI API key |
| `WHISPER_MODEL` | `whisper-1` | No | Whisper model name |
| `SOAP_MODEL` | `gpt-4o-mini` | No | GPT model for SOAP generation |
| `MEDICATION_MODEL` | `gpt-4o-mini` | No | GPT model for medication extraction |
| `FHIR_SERVER` | `http://localhost:8080/...` | No | OpenMRS FHIR endpoint |
| `OPENMRS_USER` | `Admin` | No | OpenMRS username |
| `OPENMRS_PASSWORD` | `Admin123` | No | OpenMRS password |
| `OPENMRS_SIMULATE` | `false` | No | Set `true` to skip real OpenMRS calls |
| `AGENT_MAX_RETRIES` | `2` | No | Max retries per agent |
| `AGENT_RETRY_BASE_DELAY_SECONDS` | `1.5` | No | Linear backoff base |
| `AGENT_AUDIT_ENABLED` | `true` | No | Set `false` to disable audit logging |
| `AUDIO_STORAGE_DIR` | `./audio_store` | No | Where audio files are stored on disk |
| `CORS_ORIGINS` | `http://localhost:5173,...` | No | Comma-separated allowed origins |
| `LOCAL_WHISPER_MODEL` | `base` | No | If `SERVICE_PROVIDER=local`: faster-whisper model size |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | No | If `SERVICE_PROVIDER=local`: Ollama URL |
| `OLLAMA_LLM_MODEL` | `llama3.2` | No | If `SERVICE_PROVIDER=local`: Ollama model |

### Using Local AI (Free, No API Key)

Set `SERVICE_PROVIDER=local` in `.env`, then:

1. Install [Ollama](https://ollama.ai): `brew install ollama` (macOS)
2. Pull the model: `ollama pull llama3.2`
3. Start Ollama: `ollama serve`

The `faster-whisper` library runs on CPU by default — no GPU required, but transcription will be slower than OpenAI's hosted Whisper.

---

## 11. API Quick Reference

Full interactive docs at: **http://localhost:8000/docs**

### Core Workflow Endpoints

```
POST   /encounters                              Create a new encounter
POST   /encounters/{id}/intake?auto_run=true    Upload audio + run full pipeline
GET    /encounters/{id}/pipeline                Check pipeline status (poll this)
GET    /encounters/{id}/soap                    Get current SOAP note
PATCH  /encounters/{id}/soap                    Update SOAP note
POST   /encounters/{id}/review/open             Open physician review
PATCH  /encounters/{id}/review/edit             Edit a SOAP section
POST   /encounters/{id}/review/approve          Approve (required before submit)
POST   /encounters/{id}/submit                  Submit to OpenMRS
GET    /encounters/{id}/audit/timeline          Full audit timeline
GET    /agents                                  List all agents + their status
GET    /health                                  Health check
```

### Useful Dev Endpoints

```
GET    /encounters                              List all encounters
GET    /encounters/stats                        Dashboard stats
GET    /encounters/{id}/medications             List extracted medications
GET    /encounters/{id}/submissions             Submission history
GET    /agents/{agent_name}/runs                Run history for a specific agent
```

---

## 12. How to Add a New Agent

Adding a new agent (e.g., for Layer 2 Prescription Reconciliation) takes four steps:

### Step 1 — Create the agent file

```python
# backend/app/agents/reconciliation.py
from app.agents.base import Agent, AgentResult
from app.agents.context import AgentContext

class ReconciliationAgent(Agent):
    name = "ReconciliationAgent"
    version = "1.0.0"
    description = "Compares extracted medications against existing OpenMRS prescriptions"

    async def run(self, ctx: AgentContext) -> AgentResult[dict]:
        # ctx.encounter — the current encounter
        # ctx.medications — repository for medication data
        # ctx.db — SQLAlchemy session
        ...
        return AgentResult(success=True, data={...})
```

### Step 2 — Register the agent

In `backend/app/orchestrator/registry.py`, add to `build_default_registry()`:

```python
from app.agents.reconciliation import ReconciliationAgent

registry.register(ReconciliationAgent())
```

### Step 3 — (Optional) Add a router

```python
# backend/app/routers/reconciliation.py
@router.post("/encounters/{encounter_id}/reconcile")
async def run_reconciliation(encounter_id: str, ...):
    result = await orchestrator.run_agent("ReconciliationAgent", encounter, actor="system")
    ...
```

Register it in `main.py`:
```python
from app.routers import reconciliation
app.include_router(reconciliation.router)
```

### Step 4 — (Optional) Add a new database table

1. Create `backend/app/models/reconciliation.py` with your SQLAlchemy model
2. Import it in `backend/app/models/__init__.py` so `Base.metadata` picks it up
3. Run: `alembic revision --autogenerate -m "add_reconciliation_table"`
4. Run: `alembic upgrade head`

That's it — retry logic, audit emission, and state tracking are handled automatically by the orchestrator.

---

## 13. Known Issues & Gotchas

### Audio Upload Size
Large audio files (>25 MB) may timeout with the default Uvicorn settings. If testing long recordings, either increase `--timeout-keep-alive` or pre-chunk audio client-side.

### OpenMRS UUID Configuration
The `OPENMRS_DEFAULT_PRACTITIONER_UUID` and `OPENMRS_DEFAULT_LOCATION_UUID` in `config.py` are set to OpenMRS Reference Application defaults. If your OpenMRS instance uses different UUIDs, update these values in `.env`.

### FHIR2 Module Required
The FHIR R4 integration requires the `fhir2` module to be installed and enabled in your OpenMRS instance. It is included by default in the Reference Application but may need to be enabled separately in custom installations.

### Alembic Migration Order
The 9 existing migrations must be run in order (`alembic upgrade head` does this automatically). Do not try to run individual migration files manually — always use `alembic`.

### CORS in Production
The `CORS_ORIGINS` setting in `.env` must include whatever domain your frontend is deployed on. The default `http://localhost:5173` is for local development only.

### Simulation Mode is Per-Agent
`OPENMRS_SIMULATE=true` only affects the OpenMRS Integration Agent. The rest of the pipeline (Whisper, GPT-4) still makes real API calls when `SERVICE_PROVIDER=openai`. Use `SERVICE_PROVIDER=local` too if you want a fully offline development setup.

### TypeScript Strict Mode
The frontend uses TypeScript strict mode (`"strict": true` in `tsconfig.json`). All new frontend code must be fully typed — `any` types will cause build failures.

---

## 14. Future Roadmap (Layers 2 & 3)

The project proposal defined three layers. Layer 1 is complete. The extension points for Layers 2 and 3 are already stubbed in `ARCHITECTURE.md` Section 9.

### Layer 2 — Prescription Reconciliation Agent
- **Where to start:** `backend/app/openmrs/medication.py` already has helpers for reading existing `MedicationRequest` resources from OpenMRS
- **What to build:** A `ReconciliationAgent` that runs after `MedicationExtractionAgent`, compares extracted medications against OpenMRS chart medications, and flags duplicates, conflicts, and outdated prescriptions
- **New table needed:** `reconciliation_findings` (or similar)

### Layer 3 — Dosage Anomaly Detector
- **Depends on:** Layer 2 (needs the reconciled medication list)
- **External APIs to integrate:** OpenFDA Drug API, DrugBank (requires license)
- **What to build:** A `DosageAnomalyAgent` that validates each medication's dose against the patient's clinical parameters (weight, age, renal function from vital signs)

### Other Future Work
- **E-prescribing:** Add an e-Rx submission sub-agent under `agents/openmrs/`, mirroring `note_writer.py`
- **Patient reminders:** Subscribe a new agent to the `audit_events` stream filtering `review.approved` events
- **HIPAA-grade audit shipping:** Mirror `audit_events` table to immutable storage (AWS S3 + Glacier, etc.)
- **Mobile app:** The original proposal included a React Native app; this was descoped to web-only but the API is mobile-ready

---

## 15. Contacts & Resources

### Key Documentation Files (in this repo)
- `ARCHITECTURE.md` — Engineering reference, the most important document
- `README.md` — Quick-start overview
- `Documentation/PROJECT_DEFINITION.md` — Full project definition (this folder)
- `Documentation/NEW_TEAM_HANDOFF.md` — This file

### External Documentation
- [FastAPI Docs](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 Docs](https://docs.sqlalchemy.org/en/20/)
- [Alembic Docs](https://alembic.sqlalchemy.org/en/latest/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [OpenMRS FHIR2 Module](https://wiki.openmrs.org/display/projects/OpenMRS+FHIR2+Module)
- [FHIR R4 Specification](https://hl7.org/fhir/R4/)
- [Pydantic v2 Docs](https://docs.pydantic.dev/latest/)
- [React Router v7 Docs](https://reactrouter.com/en/main)

### Interactive API Explorer
When the backend is running: **http://localhost:8000/docs**

All 40+ endpoints are documented with request/response schemas, and you can execute requests directly from the browser.
