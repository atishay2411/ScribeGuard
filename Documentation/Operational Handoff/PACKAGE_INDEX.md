# ScribeGuard — New Team Package Index

This folder contains everything the new team needs to pick up and continue the project.

---

## Start Here

| Document | Purpose | Read Order |
|----------|---------|-----------|
| [NEW_TEAM_HANDOFF.md](NEW_TEAM_HANDOFF.md) | Step-by-step setup, install, run, and build instructions. Covers known issues, architecture walk-through, and how to extend the system. | **1st — Read this first** |
| [PROJECT_DEFINITION.md](PROJECT_DEFINITION.md) | Full project definition based on the CS595 template: problem statement, scope, functional requirements, sprint history, and open questions. | **2nd** |
| [../README.md](../README.md) | Quick-start overview and repository layout | **3rd** |
| [../ARCHITECTURE.md](../ARCHITECTURE.md) | Deep-dive engineering reference: agent contracts, state machine, extension points, design rationale | **4th (then keep as reference)** |

---

## What's in This Package

```
Documentation/
├── PACKAGE_INDEX.md              ← You are here
├── NEW_TEAM_HANDOFF.md           ← Setup + development guide for new team
├── PROJECT_DEFINITION.md         ← Filled-out CS595 project definition template
├── CS595 2026 - Project Info Template (1).docx   ← Original blank template from instructor
├── annotated-ProjectProposal (2).pdf             ← Original project proposal
└── annotated-ProjectProposalAddendum (1).pdf     ← Revised proposal (scope changes)

Root of repository:
├── README.md                     ← Project overview
├── ARCHITECTURE.md               ← Engineering reference
├── docker-compose.yml            ← PostgreSQL + API containers
├── backend/                      ← FastAPI backend (Python)
├── frontend/                     ← React frontend (TypeScript)
└── test_transcripts/             ← Sample audio files for testing
```

---

## TL;DR — Fastest Possible Setup

```bash
# 1. Clone and enter the repo
git clone <repo-url> && cd ScribeGuard

# 2. Start the database
docker compose up -d db

# 3. Set up backend
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 4. Create backend/.env (copy and fill in your OpenAI key)
#    See NEW_TEAM_HANDOFF.md Section 4 for the full template

# 5. Run migrations
alembic upgrade head

# 6. Start backend (Terminal 1)
uvicorn app.main:app --reload --port 8000

# 7. Start frontend (Terminal 2)
cd ../frontend && npm install && npm run dev
```

Open http://localhost:5173 — you're running.  
Open http://localhost:8000/docs — interactive API explorer.

---

## Where We Left Off

Layer 1 (ambient documentation → OpenMRS submission) is **complete**.  
Layers 2 and 3 (prescription reconciliation, dosage anomaly detection) are **not started**.

See [NEW_TEAM_HANDOFF.md → Section 14](NEW_TEAM_HANDOFF.md#14-future-roadmap-layers-2--3) for the roadmap.
