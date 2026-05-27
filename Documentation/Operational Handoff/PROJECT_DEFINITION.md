# ScribeGuard — Project Definition Document

> Based on the CS595 2026 Project Info Template

---

## 1. Project Definition

**Project Title:** ScribeGuard — Agentic AI Clinical Documentation Platform

**Project ID:** CS595-2026-ScribeGuard

**Author(s):** Atishay Jain, Shifaan Nadaf, and team

**LOF Pillar:** ☑ Patient Engagement

**Last Updated:** May 2026

---

## 2. Introduction

### Problem Description

Physicians spend an estimated 37–49% of their working time on documentation, with primary care doctors entering data for more than two hours per every hour of direct patient contact. This documentation burden is a leading cause of physician burnout, reduced time with patients, and delayed care delivery.

In OpenMRS-based health systems — which are common across low- and middle-income countries — the problem is compounded because clinical staff must manually transcribe handwritten or dictated notes into structured EHR fields after each encounter. This creates:

- **Accuracy risk:** Manual re-entry introduces transcription errors.
- **Timeliness risk:** Notes may be entered hours or days after the encounter, relying on memory.
- **Compliance risk:** Incomplete documentation makes audit and billing more difficult.
- **Burnout:** Administrative burden displaces clinical work.

ScribeGuard addresses this by automating the transcription and structuring of clinical encounters while keeping the physician in control. The system captures audio from the physician-patient encounter, routes it through a chain of AI agents that produce a structured SOAP note and FHIR-aligned clinical entities, and then requires the physician to explicitly review and approve the output before anything is written back to the patient's OpenMRS record.

### Purpose

ScribeGuard is an ambient AI clinical documentation assistant that reduces physician documentation burden by automatically transcribing encounters and generating structured SOAP notes, while guaranteeing that no AI output reaches the EHR without explicit physician approval.

**Goals and Objectives:**
1. Reduce time spent on post-encounter documentation from 2+ hours per day to under 30 minutes.
2. Improve documentation completeness and consistency through structured SOAP output.
3. Integrate natively with OpenMRS via FHIR R4 so extracted data lands in the correct structured fields (not just as free-text notes).
4. Provide a full audit trail of every AI action and physician decision for compliance and quality improvement.
5. Design the system so future layers (prescription reconciliation, dosage anomaly detection) can be added without re-architecting the core pipeline.

### Scope

**Included in Scope (Layer 1 — Completed):**
- Browser-based audio capture via the Web MediaRecorder API
- Whisper-powered transcription with quality scoring
- GPT-4 SOAP note generation with version history
- Structured extraction of medications, allergies, conditions, vital signs, and follow-ups
- Physician review workflow: open → edit → approve (explicit gate before any EHR write)
- OpenMRS FHIR R4 submission: Encounter, clinical-note Observation, AllergyIntolerance, Condition, MedicationRequest, vital-sign Observations
- Full audit trail (per-agent, per-section edit, approval records) with queryable timeline
- Simulation mode for development/testing without a live OpenMRS server
- React web dashboard and encounter workspace

**Excluded from Scope (Deferred to Layers 2 & 3):**
- Prescription reconciliation (comparing new prescriptions against existing OpenMRS MedicationRequests)
- Dosage anomaly detection (validating prescriptions against clinical drug databases)
- React Native mobile application (descoped from original proposal; web-only approach adopted)
- E-prescribing integration
- Real-time captioning / streaming transcription
- HIPAA-grade encrypted audit log export
- Multi-tenant / multi-clinic deployment

### Audience

**Primary Users:**

| User Type | Characteristics | Primary Need |
|-----------|----------------|--------------|
| Attending Physicians | Clinical experts, not necessarily tech-savvy, time-constrained, frustrated with documentation | Fast, accurate SOAP note generation; clear approval workflow; confidence that AI output is correct before it enters the chart |
| Residents / Medical Students | Training environment, learning documentation standards | Structured SOAP templates with AI guidance; educational feedback via audit trail |
| Clinical Informatics Staff / OpenMRS Admins | Technical users managing EHR configuration | FHIR R4 compatibility; simulation mode; audit data export |
| QA / Compliance Officers | Non-clinical, auditing workflows | Immutable audit trail showing every AI and human action |

**User Characteristics:**
- Physicians are high-expertise in their domain but have low tolerance for workflow interruptions
- Physicians may be suspicious of AI accuracy — the explicit approval gate is essential for trust
- Many users operate in low-bandwidth environments (OpenMRS is widely deployed in under-resourced settings)
- The interface must be usable with minimal training

### Overview

ScribeGuard is a web application with a React frontend and a FastAPI backend. The backend implements a multi-agent AI architecture where each step in the clinical documentation pipeline is handled by a specialized autonomous agent. A central orchestrator coordinates the agents, handles retries, logs every invocation, and maintains a durable audit trail in PostgreSQL.

The workflow:
1. Physician starts a new encounter in the dashboard and records audio via the browser microphone.
2. The audio is uploaded to the backend, which runs the full auto-pipeline: intake validation → Whisper transcription → GPT-4 SOAP generation → medication/entity extraction.
3. The physician reviews the AI-generated SOAP note in the Encounter Workspace, edits any sections as needed, and explicitly clicks "Approve."
4. After approval, the physician submits to OpenMRS. The OpenMRS Integration Agent authenticates, resolves the patient, maps the approved SOAP data to FHIR R4 resources, writes them, and verifies the write by reading back the created resources.
5. The full encounter — audio, transcript, SOAP versions, physician edits, approval, submission records, and all agent run logs — is stored in PostgreSQL and accessible via the audit timeline.

### Glossary / Terminology

| Term | Definition |
|------|-----------|
| SOAP Note | Structured clinical note format: Subjective, Objective, Assessment, Plan |
| FHIR R4 | Fast Healthcare Interoperability Resources, Release 4 — the standard API/data format for EHR integration |
| OpenMRS | Open Medical Record System — an open-source EHR platform widely used in global health |
| Whisper | OpenAI's speech-to-text model used for transcription |
| Agent | An autonomous software component with a single, well-defined responsibility in the pipeline |
| Orchestrator | The component that coordinates agents, handles retries, and maintains pipeline state |
| AgentContext | The shared state bundle (database session, encounter, repositories) passed into every agent |
| Processing Stage | Fine-grained pipeline state (e.g., `transcribing`, `soap_drafted`, `in_review`, `submitted`) |
| Encounter Status | User-visible lifecycle state (`pending`, `approved`, `pushed`, `failed`) |
| SOAP Version | Each time the SOAP note is regenerated, a new version is created; `is_current=true` identifies the active one |
| Simulation Mode | `OPENMRS_SIMULATE=true` — the OpenMRS agent returns fake UUIDs without making real FHIR calls |
| CIEL | Clinical Interface Engine for Languages — a medical concept coding system used in OpenMRS |
| ICD-10 | International Classification of Diseases, 10th revision — used for condition coding |
| SNOMED | Systematized Nomenclature of Medicine — used alongside ICD-10 for condition coding |
| Alembic | SQLAlchemy's database migration tool |
| Pydantic | Python data validation library (v2 used) for API schemas and configuration |

### Skills Needed

**Backend Development:**
- Python 3.11+ (async/await, Pydantic v2)
- FastAPI framework
- SQLAlchemy 2.0 ORM + Alembic migrations
- PostgreSQL

**AI/ML:**
- OpenAI API usage (Whisper, GPT-4 family)
- Prompt engineering (structured JSON output, clinical domain)
- Optional: faster-whisper, Ollama for local inference

**Healthcare / EHR:**
- FHIR R4 resource types (Encounter, Observation, Condition, AllergyIntolerance, MedicationRequest)
- OpenMRS data model and FHIR2 module
- Basic clinical terminology (SOAP, ICD-10, SNOMED, CIEL)

**Frontend:**
- React 19 + TypeScript
- React Router v7
- Browser MediaRecorder API

**DevOps:**
- Docker / Docker Compose
- Basic Linux/shell skills for environment setup

**LOF Partner Involvement:** OpenMRS community and any clinical partner providing physician access for user testing.

---

## 3. System Overview

### Architecture

ScribeGuard uses a **multi-agent pipeline architecture** with four horizontal layers:

```
┌─────────────────────────────────────────────────────────┐
│                React Web Frontend                       │
│  Dashboard | Encounter Workspace | History | Patients   │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP (fetch / REST)
                       ▼
┌─────────────────────────────────────────────────────────┐
│              FastAPI HTTP API (11 routers)               │
│   /encounters, /pipeline, /review, /audit, /agents ...  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Agent Orchestrator                          │
│   Retries | State transitions | Audit emission           │
│   ┌──────────────────────────────────────────────────┐  │
│   │ 7 Specialized Agents                             │  │
│   │  Intake → Transcription → NoteGen → MedExtract  │  │
│   │  → PhysicianReview → OpenMRSIntegration → Audit │  │
│   └──────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────┴─────────────┐
          ▼                          ▼
┌──────────────────┐      ┌──────────────────────────────┐
│   PostgreSQL     │      │   External Services           │
│   (13 tables)    │      │   OpenAI (Whisper + GPT-4)    │
│   encounters     │      │   OpenMRS FHIR R4 API         │
│   transcripts    │      │   (optional: Ollama + local   │
│   soap_notes     │      │    faster-whisper)            │
│   audit_events   │      └──────────────────────────────┘
│   agent_runs     │
│   ...            │
└──────────────────┘
```

**Key architectural decisions and rationale:**

- **Dedicated agents instead of a single pipeline function:** Each agent has its own failure mode (Whisper rate limits, GPT JSON parse errors, OpenMRS auth errors). Typed agents make those boundaries addressable individually with clean retry semantics.
- **Repository pattern:** Agents never touch SQLAlchemy sessions directly. They use `ctx.transcripts`, `ctx.soap_notes`, etc. This keeps agents testable in isolation.
- **Explicit orchestrator as single state source:** Without a central orchestrator, audit gaps appear when agents partially succeed. The orchestrator is the only code that advances `encounter.processing_stage` across stage boundaries.
- **Versioned SOAP notes:** Regenerating a draft after physician edits must not destroy their work. Previous versions are marked `superseded`; the new one becomes `is_current=true`.
- **OpenMRS split into sub-agents:** So the failure reason (auth vs. mapping vs. write vs. verify) is distinct in the audit trail.

### Technologies Used

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend framework | React | 19.x |
| Frontend language | TypeScript | 6.0 |
| Frontend build tool | Vite | 8.x |
| Frontend routing | React Router DOM | 7.x |
| Frontend icons | Lucide React | 1.x |
| Backend framework | FastAPI | 0.115 |
| Backend runtime | Python | 3.11+ |
| Backend server | Uvicorn | 0.30 |
| ORM | SQLAlchemy | 2.0 |
| Migrations | Alembic | 1.13 |
| Data validation | Pydantic v2 | 2.9 |
| Database | PostgreSQL | 16 |
| Database driver | psycopg2-binary | 2.9 |
| AI: transcription | OpenAI Whisper | via openai SDK ≥1.54 |
| AI: SOAP generation | OpenAI GPT-4o-mini | via openai SDK |
| AI: local alternative | faster-whisper + Ollama | ≥1.0.3 |
| FHIR HTTP client | httpx | ≥0.27 |
| Retry logic | tenacity | ≥9.0 |
| Structured logging | structlog | ≥24.4 |
| PDF/DOCX parsing | pypdf, python-docx | ≥4.0 / ≥1.1 |
| PDF generation | fpdf2 | ≥2.8 |
| Containerization | Docker / Docker Compose | Latest |

### Dependencies

**External Services:**
- **OpenAI API** — Whisper transcription + GPT-4 SOAP/medication generation. Requires a paid API key. Costs are proportional to audio duration and note generation volume.
- **OpenMRS FHIR2 API** — the EHR integration endpoint. OpenMRS must have the FHIR2 module installed and enabled. Default endpoint: `http://localhost:8080/openmrs/ws/fhir2/R4`.
- **Ollama (optional)** — local LLM server for offline/free inference. Requires separate installation.

**Internal Dependencies:**
- PostgreSQL must be running before the backend can start.
- The frontend assumes the backend is running at `http://localhost:8000` (or whatever the Vite proxy is configured to).
- Alembic migrations must be run before the first API request.

### Hardware and Software Requirements

**Development Machine:**
- OS: macOS, Linux, or Windows (WSL2 recommended for Windows)
- RAM: 8 GB minimum (16 GB recommended if running local Whisper)
- Disk: 5 GB free (more if storing long audio files)
- CPU: Any modern x86-64 processor
- GPU: Not required for `SERVICE_PROVIDER=openai`; helpful for `SERVICE_PROVIDER=local`

**Required Software:**
- Python 3.11+
- Node.js 18+
- Docker Desktop (for PostgreSQL)
- A modern browser (Chrome, Firefox, Safari) with microphone access for audio capture

**Recommended:**
- VS Code or PyCharm for development
- Postman or the built-in Swagger UI for API testing

**Production Deployment (not yet configured):**
- Any Linux VPS or container orchestration platform (Kubernetes, ECS)
- Persistent volume for PostgreSQL data and audio storage
- Reverse proxy (Nginx, Caddy) with TLS termination
- CORS configuration updated to production domain

### Data Description and Sources

**Data Generated by the System:**
- **Audio files:** Browser MediaRecorder output (WebM/OGG/MP4 depending on browser). Stored on the backend filesystem at `AUDIO_STORAGE_DIR`. Not stored in the database.
- **Transcripts:** Plain text output from Whisper with quality metadata (word count, duration, quality score, detected issues). Stored in the `transcripts` table.
- **SOAP Notes:** Structured JSON with four text sections (Subjective, Objective, Assessment, Plan). Versioned — all versions retained. Stored in `soap_notes`.
- **Clinical Entities:** Medications (with dose, route, frequency, indication), allergies (with severity), conditions (with ICD-10 and SNOMED codes), vital signs, follow-up instructions. Each stored in its own table.
- **Audit Events:** Immutable append-only log of every meaningful system action. Stored in `audit_events`.
- **Agent Run Logs:** Timing, status, input/output summaries, and error context for every agent invocation. Stored in `agent_runs`.
- **Submission Records:** OpenMRS write attempt history including returned FHIR resource UUIDs. Stored in `submission_records`.

**Data Sources:**
- Encounter audio: live capture from physician's browser microphone
- Patient demographics and chart context: fetched live from OpenMRS via FHIR R4 at encounter start
- Medication/allergy/condition data: extracted by AI from the transcript + SOAP note

**Data Accessibility:**
- All data is stored in a single PostgreSQL database; accessible to any team member with the `DATABASE_URL`
- Audio files are stored on the backend host filesystem; accessible via the server's file system or via the export endpoints
- No external data sources require authentication beyond OpenAI (API key) and OpenMRS (username/password)

**Sample/Test Data:**
- `test_transcripts/` directory contains sample audio files and transcript text for pipeline testing
- `seed.py` can populate the database with test data
- `OPENMRS_SIMULATE=true` enables end-to-end testing without a real OpenMRS instance

### Pre-Work / Setup

Before the project can be built and run, the following must be in place:

1. **OpenAI API account and key** — create at https://platform.openai.com. Costs are pay-as-you-go; a typical development session (10–20 test encounters) costs under $1.
2. **Docker Desktop** installed and running for PostgreSQL.
3. **Python virtual environment** created and `requirements.txt` installed.
4. **`backend/.env` file** created from the template in this handoff guide.
5. **Alembic migrations** run (`alembic upgrade head`).
6. **OpenMRS instance** (optional for development — `OPENMRS_SIMULATE=true` bypasses this).

---

## 4. Functional Requirements

### List of User Tasks

| # | Task | Actor |
|---|------|-------|
| 1 | Record Encounter | Physician |
| 2 | Review Transcript | Physician |
| 3 | Edit SOAP Note | Physician |
| 4 | Approve SOAP Note | Physician |
| 5 | Revert Approval | Physician |
| 6 | Submit to OpenMRS | Physician |
| 7 | View Audit Timeline | Physician / QA |
| 8 | View Encounter History | Physician / Admin |
| 9 | Search Encounters | Physician / Admin |
| 10 | View Patient Context | Physician |
| 11 | View Agent Run Status | Physician / Admin |
| 12 | Add Medication Manually | Physician |
| 13 | Edit Medication | Physician |
| 14 | Delete Medication | Physician |
| 15 | Export Encounter Data | Admin |

### User Stories

**As a physician, I want to:**
- Record a patient encounter using my laptop/tablet microphone so that I don't have to type notes during the visit.
- See a SOAP note generated from the recording within 2–3 minutes so I can quickly verify and approve it.
- Edit any section of the AI-generated SOAP note before approval so that I can correct inaccuracies.
- See low-confidence sections highlighted so I know where to focus my review.
- Explicitly approve the note before it is submitted to the EHR so I maintain medical-legal control over the record.
- View the patient's current medications, allergies, and conditions from OpenMRS so I have chart context during review.
- See a live status indicator showing where in the processing pipeline my recording is.
- View the complete history of all encounters so I can find past notes quickly.

**As a compliance officer, I want to:**
- See a complete, timestamped log of every AI action and physician decision so I can audit the documentation process.
- Know which version of the AI model and prompt generated each note so I can assess reproducibility.
- See a record of all physician edits (original text vs. edited text, by section) so I can assess how much the AI output was modified.

### Use Cases

**UC-01: Record and Process Encounter**
- Actor: Physician
- Precondition: Physician has navigated to Dashboard
- Flow: Select patient → Click "Start Recording" → Conduct encounter → Click "Stop" → System uploads audio and auto-runs full pipeline (Intake → Transcription → SOAP → Medication extraction)
- Postcondition: Encounter is in `ready_for_review` stage; SOAP note and entities are available for review

**UC-02: Review and Approve SOAP Note**
- Actor: Physician
- Precondition: Encounter is in `ready_for_review` stage
- Flow: Navigate to Encounter Workspace → Open Review → Review SOAP sections (edit if needed) → Click "Approve & Lock"
- Postcondition: Encounter status is `approved`; edit log is persisted; SOAP note status is `approved`

**UC-03: Submit to OpenMRS**
- Actor: Physician
- Precondition: Encounter status is `approved`
- Flow: Click "Submit to OpenMRS" → System runs OpenMRS Integration Agent (auth → patient lookup → FHIR mapping → write → verify) → Submission record created with resource UUIDs
- Postcondition: Encounter status is `pushed`; FHIR resources exist in OpenMRS with UUIDs stored in the database

**UC-04: Revert and Re-Approve**
- Actor: Physician
- Precondition: Encounter status is `approved` but not yet submitted
- Flow: Click "Revert Approval" → Edit additional sections → Re-approve
- Postcondition: New approval record created; previous approval marked superseded

**UC-05: View Audit Timeline**
- Actor: Physician / Compliance Officer
- Precondition: Any encounter exists
- Flow: Navigate to Encounter Workspace → Select "Audit" tab → View chronological timeline of all events
- Postcondition: Full clinical event history displayed with timestamps, actors, and details

### Prioritization

| Priority | Requirement | Status |
|----------|-------------|--------|
| P0 — Must Have | Audio capture and upload | Complete |
| P0 — Must Have | Whisper transcription | Complete |
| P0 — Must Have | GPT-4 SOAP note generation | Complete |
| P0 — Must Have | Physician review and approval gate | Complete |
| P0 — Must Have | OpenMRS FHIR R4 submission | Complete |
| P0 — Must Have | Immutable audit trail | Complete |
| P1 — Should Have | Medication extraction | Complete |
| P1 — Should Have | Allergy, condition, vital sign extraction | Complete |
| P1 — Should Have | Patient chart context from OpenMRS | Complete |
| P1 — Should Have | Agent run timeline visualization | Complete |
| P2 — Could Have | Prescription reconciliation (Layer 2) | Not started |
| P2 — Could Have | Dosage anomaly detection (Layer 3) | Not started |
| P3 — Future | Mobile app (React Native) | Descoped |
| P3 — Future | E-prescribing integration | Not started |
| P3 — Future | HIPAA audit log export | Not started |

### List of Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-01 | The system shall capture audio from the physician's browser microphone and upload it to the backend. | P0 |
| FR-02 | The system shall transcribe uploaded audio using Whisper and store the transcript with quality metadata. | P0 |
| FR-03 | The system shall generate a structured SOAP note from the transcript using a GPT-4 family model. | P0 |
| FR-04 | The system shall preserve all versions of SOAP notes; previous versions shall not be deleted when a new version is generated. | P0 |
| FR-05 | The system shall extract structured medication entities (name, dose, route, frequency, indication) from the SOAP Plan section. | P1 |
| FR-06 | The system shall extract allergies, conditions (with ICD-10/SNOMED), vital signs, and follow-up instructions from the SOAP note. | P1 |
| FR-07 | The system shall allow a physician to open a review phase, edit any SOAP section, and record the original and edited text. | P0 |
| FR-08 | The system shall require an explicit physician approval action before any EHR submission is permitted. This requirement shall be enforced server-side. | P0 |
| FR-09 | The system shall allow a physician to revert an approval if no submission has yet occurred. | P1 |
| FR-10 | The system shall submit the approved encounter to OpenMRS via FHIR R4, creating Encounter, Observation (clinical note + vitals), AllergyIntolerance, Condition, and MedicationRequest resources. | P0 |
| FR-11 | The system shall store the FHIR resource UUIDs returned by OpenMRS on the corresponding database rows. | P0 |
| FR-12 | The system shall log every agent invocation (status, timing, attempt count, error context) and every clinical/business event in an immutable audit trail. | P0 |
| FR-13 | The system shall expose the audit trail as a queryable, chronological timeline via the HTTP API. | P1 |
| FR-14 | The system shall fetch and display the patient's current chart context (demographics, active medications, allergies, conditions, recent encounters) from OpenMRS at encounter start. | P1 |
| FR-15 | The system shall surface the current pipeline stage to the physician in real time via the frontend. | P1 |
| FR-16 | The system shall support a simulation mode (`OPENMRS_SIMULATE=true`) that skips real OpenMRS calls and returns stable fake UUIDs for development/CI purposes. | P1 |
| FR-17 | The system shall support a local AI provider option (faster-whisper + Ollama) that requires no external API keys. | P2 |
| FR-18 | The system shall allow physicians to manually add, edit, and delete medication entries from the encounter record. | P1 |
| FR-19 | The system shall expose all encounter data (encounters, transcripts, SOAP notes, medications, audit events) via a documented REST API with Swagger UI. | P0 |
| FR-20 | The system shall retain a submission history showing every attempt to submit to OpenMRS, including failures and the reason for failure. | P1 |

---

## 5. Non-Functional Requirements

| ID | Requirement | Notes |
|----|-------------|-------|
| NFR-01 | The full auto-pipeline (intake → transcription → SOAP → medications) shall complete within 3 minutes for a 10-minute audio recording under normal load. | Dependent on OpenAI API latency |
| NFR-02 | The system shall not allow any AI output to reach OpenMRS without physician approval. | Enforced server-side in PhysicianReviewAgent |
| NFR-03 | The audit trail shall be append-only; no audit events shall be deleteable via the API. | Enforced at the repository layer |
| NFR-04 | All configuration shall be injectable via environment variables with no hard-coded secrets in source code. | config.py + .env pattern |
| NFR-05 | The backend API shall serve CORS-restricted responses; allowed origins shall be configurable. | Via CORS_ORIGINS env var |
| NFR-06 | The system shall support retry with configurable backoff for transient agent failures. | AGENT_MAX_RETRIES, AGENT_RETRY_BASE_DELAY_SECONDS |
| NFR-07 | Database schema changes shall be managed via Alembic migrations (no manual DDL). | 9 migrations completed |

---

## 6. Sprint History

### Sprint 1 — Core Pipeline
- Set up FastAPI backend skeleton with PostgreSQL
- Implemented agent base class and orchestrator
- Implemented EncounterIntakeAgent, TranscriptionAgent, ClinicalNoteGenerationAgent
- Built React dashboard with MediaRecorder audio capture
- Established database schema (initial migration)

### Sprint 2 — OpenMRS Integration + Full Entity Extraction
- Implemented MedicationExtractionAgent and clinical entity extraction (allergies, conditions, vitals)
- Built PhysicianReviewAgent with edit/approve/revert state machine
- Implemented OpenMRS Integration Agent (auth, patient context, FHIR mapping, write, verify)
- Built Encounter Workspace UI with SOAP editor, medication panel, entity panels
- Added patient context sidebar (live OpenMRS chart fetch)
- Completed audit trail and agent timeline visualization
- Added simulation mode for OpenMRS
- Completed Alembic migration suite (9 migrations)
- Bug fixes: OpenMRS submission step spinner, PDF attachment population

---

## 7. Open Questions / Blanks

The following items from the project definition template were not fully resolved during the semester:

- **Production deployment architecture:** Not yet designed. The system runs locally; a production hosting plan (cloud provider, container orchestration, TLS, secrets management) has not been specified.
- **HIPAA compliance audit:** The audit trail design was inspired by HIPAA requirements, but a formal compliance review was not conducted.
- **OpenFDA / DrugBank integration:** Required for Layer 3 but not researched in detail. DrugBank requires a license agreement; OpenFDA is free.
- **Performance benchmarking:** The 3-minute pipeline target (NFR-01) was not formally measured under load.
- **User acceptance testing:** No formal usability testing with real physicians was conducted. The UI is based on design assumptions about physician workflows.
- **Authentication / Authorization:** The current system has a stub login page but no real authentication. In production, this would need proper auth (OAuth2, OpenID Connect with OpenMRS, or similar).
- **Data retention policy:** How long audio files, transcripts, and notes should be retained was not defined.
