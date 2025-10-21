diff --git a/README.md b/README.md
index 8b137891791fe96927ad78e64b0aad7bded08bdc..db8970b09de50802506b7fac4c8534338746dc99 100644
--- a/README.md
+++ b/README.md
@@ -1 +1,306 @@
+# SDLC Copilot
 
+An AI-powered SDLC orchestration platform that takes a project from requirements to deployment guidance through a structured, review-driven workflow.
+
+It combines:
+- **Backend orchestration** with FastAPI + LangGraph.
+- **Multi-model LLM generation/revision** (Gemini, Groq/Qwen, Anthropic/Claude).
+- **Human-in-the-loop approvals** at key SDLC checkpoints.
+- **Frontend workflow UI** built with React + Vite + Tailwind.
+
+---
+
+## Table of Contents
+- [What this project does](#what-this-project-does)
+- [High-level architecture](#high-level-architecture)
+- [End-to-end workflow](#end-to-end-workflow)
+- [Backend deep dive](#backend-deep-dive)
+  - [Core responsibilities](#core-responsibilities)
+  - [State model](#state-model)
+  - [Graph orchestration model](#graph-orchestration-model)
+  - [API surface](#api-surface)
+  - [Persistence and sessioning](#persistence-and-sessioning)
+- [Frontend deep dive](#frontend-deep-dive)
+  - [Routing and phase flow](#routing-and-phase-flow)
+  - [Phase UX and review loop](#phase-ux-and-review-loop)
+- [Project structure](#project-structure)
+- [Local development setup](#local-development-setup)
+- [Configuration](#configuration)
+- [How a typical run works](#how-a-typical-run-works)
+- [Known implementation notes](#known-implementation-notes)
+- [Roadmap ideas](#roadmap-ideas)
+
+---
+
+## What this project does
+
+`SDLC Copilot` guides a project through these phases:
+1. Requirements intake
+2. User stories generation + review
+3. Functional design generation + review
+4. Technical design generation + review
+5. Frontend code generation + review
+6. Backend code generation + review
+7. Security review + remediation
+8. Test case generation + review
+9. QA testing + remediation loop
+10. Deployment steps generation
+
+The system supports **iterative revisions** at approval checkpoints and transitions to the next stage only after approval-like decisions.
+
+---
+
+## High-level architecture
+
+```mermaid
+flowchart LR
+    U[Product Owner / User] --> FE[React Frontend]
+    FE -->|HTTP JSON| API[FastAPI Backend]
+    API --> WF[LangGraph SDLC Workflow]
+    WF --> LLM1[Gemini]
+    WF --> LLM2[Groq Qwen]
+    WF --> LLM3[Anthropic Claude]
+    API --> R[(Redis Session Store)]
+
+    subgraph SDLC Graph
+      N1[User Stories]
+      N2[Functional Doc]
+      N3[Technical Doc]
+      N4[Frontend Code]
+      N5[Backend Code]
+      N6[Security Review]
+      N7[Test Cases]
+      N8[QA Testing]
+      N9[Deployment]
+      N1-->N2-->N3-->N4-->N5-->N6-->N7-->N8-->N9
+    end
+
+    WF --- SDLC Graph
+```
+
+---
+
+## End-to-end workflow
+
+The backend compiles a LangGraph state machine (`SDLCState`) where each phase has:
+- output artifact(s),
+- message history,
+- status field,
+- conditional transition function.
+
+Human review stages are configured with `interrupt_before`, so execution pauses before review nodes. The API then accepts owner feedback (`approved` or comments), updates graph state, and resumes execution.
+
+This yields a deterministic, resumable approval pipeline rather than a single-shot generation call.
+
+---
+
+## Backend deep dive
+
+### Core responsibilities
+- Serve phase-wise REST APIs for generation/review/retrieval.
+- Build and execute the SDLC graph.
+- Manage workflow session continuity using a `thread_id` and Redis cache.
+- Marshal graph state into pydantic response models.
+
+### State model
+`SDLCState` is the central contract with typed fields for:
+- requirements,
+- user stories,
+- functional/technical documents,
+- frontend/backend code,
+- security reviews,
+- test cases,
+- QA report,
+- deployment steps,
+- status fields and revision counters.
+
+### Graph orchestration model
+The graph is built in `SDLCGraphBuilder` and wires nodes in sequence with conditional edges:
+- review nodes branch to either:
+  - revision node (feedback path), or
+  - next phase (approved path).
+- QA has pass/fail branching:
+  - fail → backend fix loop,
+  - pass → deployment generation.
+
+### API surface
+The FastAPI app exposes:
+
+- Health/status:
+  - `GET /health`
+  - `GET /status`
+
+- User stories:
+  - `POST /stories/generate`
+  - `POST /stories/review/{session_id}`
+
+- Functional docs:
+  - `POST /documents/functional/generate/{session_id}`
+  - `POST /documents/functional/review/{session_id}`
+
+- Technical docs:
+  - `POST /documents/technical/generate/{session_id}`
+  - `POST /documents/technical/review/{session_id}`
+
+- Code:
+  - `POST /code/frontend/generate/{session_id}`
+  - `POST /code/frontend/review/{session_id}`
+  - `POST /code/backend/generate/{session_id}`
+  - `POST /code/backend/review/{session_id}`
+
+- Security:
+  - `GET /security/review/get/{session_id}`
+  - `POST /security/review/review/{session_id}`
+
+- Testing & QA:
+  - `GET /test/cases/get/{session_id}`
+  - `POST /test/cases/review/{session_id}`
+  - `GET /qa/testing/get/{session_id}`
+
+- Deployment:
+  - `GET /deployment/get/{session_id}`
+
+### Persistence and sessioning
+- **LangGraph MemorySaver** checkpoints workflow state while running.
+- **Redis** stores normalized session artifacts for API retrieval and validation.
+- `session_validator` enforces legal phase transitions and prevents invalid calls.
+
+---
+
+## Frontend deep dive
+
+### Routing and phase flow
+- `/` → project requirements intake form.
+- `/sdlc` → multi-phase workspace with:
+  - phase sidebar,
+  - phase content panel,
+  - approval/feedback bar.
+
+Phase progression is controlled by `completedPhases` + lock/unlock logic in the sidebar.
+
+### Phase UX and review loop
+- Initial submission calls `/stories/generate`.
+- Every reviewable phase has:
+  - **Approve & Continue** → sends `{ feedback: "approved" }` to mapped review endpoint.
+  - **Feedback input** → sends free-text feedback for revision.
+- Phase screens fetch generated artifacts from backend and render them (markdown/code/cards).
+
+---
+
+## Project structure
+
+```text
+PairProgramer/
+├─ backend/
+│  ├─ app.py                         # FastAPI entrypoint and endpoint handlers
+│  ├─ workflow_test.py               # Scripted workflow runner
+│  ├─ requirements.txt
+│  └─ src/sdlccopilot/
+│     ├─ graph/sdlc_graph.py         # LangGraph topology
+│     ├─ nodes/                      # SDLC phase node implementations
+│     ├─ helpers/                    # LLM helper methods per phase
+│     ├─ prompts/                    # Prompt templates
+│     ├─ states/                     # Pydantic state models
+│     ├─ llms/                       # Provider wrappers (Gemini/Groq/Anthropic)
+│     ├─ requests.py                 # Request DTOs
+│     └─ responses.py                # Response DTOs
+├─ frontend/
+│  ├─ src/pages/                     # Home + SDLC pages
+│  ├─ src/phases/                    # Phase-specific views
+│  ├─ src/components/                # Shared UI components
+│  ├─ src/types.ts
+│  └─ config.ts                      # Backend URL from env
+└─ notebooks/                        # Prompt/prototype notebooks
+```
+
+---
+
+## Local development setup
+
+### Prerequisites
+- Python 3.10+
+- Node.js 18+
+- Redis server
+
+### 1) Backend
+```bash
+cd backend
+python -m venv .venv
+source .venv/bin/activate
+pip install -r requirements.txt
+uvicorn app:app --reload --host 0.0.0.0 --port 8000
+```
+
+### 2) Frontend
+```bash
+cd frontend
+npm install
+npm run dev
+```
+
+### 3) Open app
+- Frontend: `http://localhost:5173`
+- Backend docs: `http://localhost:8000/docs`
+
+---
+
+## Configuration
+
+### Backend env vars
+Set in backend runtime environment (or `.env`):
+
+- `PROJECT_ENVIRONMENT` (`development` uses constants/mocks in nodes)
+- `REDIS_HOST`
+- `REDIS_PORT`
+- `REDIS_PASSWORD`
+- `GROQ_API_KEY`
+- `ANTHROPIC_API_KEY`
+- `GEMINI_API_KEY` (used by Gemini wrapper)
+- `GOOGLE_API_KEY` (also set in app bootstrap)
+- `LANGSMITH_API_KEY`
+- `LANGSMITH_PROJECT`
+- `LANGSMITH_TRACING`
+
+### Frontend env vars
+In `frontend/.env`:
+
+```env
+VITE_BACKEND_URL=http://localhost:8000
+```
+
+---
+
+## How a typical run works
+
+1. User submits requirements from Home page.
+2. Backend creates `session_id`, starts graph, halts at first review checkpoint.
+3. Frontend displays generated artifact.
+4. User approves or submits feedback.
+5. Backend updates graph state and resumes.
+6. Loop repeats until deployment steps are generated.
+
+---
+
+## Known implementation notes
+
+- Some files include prototype/test artifacts and notebooks; this is both a production API and experimentation workspace.
+- Review flow is strict: direct calls out of sequence are blocked by server-side session validation.
+- QA state modeling and frontend QA rendering should be kept aligned (backend returns structured QA object; frontend currently maps it similarly to test-case cards).
+- `PROJECT_ENVIRONMENT=development` enables deterministic fallback constants and simulated delay in node handlers.
+
+---
+
+## Roadmap ideas
+
+- Add auth + per-user session ownership.
+- Replace in-memory graph checkpointing with persistent checkpointer.
+- Add richer deployment output format (checklists + IaC stubs).
+- Add contract tests for each phase endpoint and state transition.
+- Add observability dashboards for token usage and phase latency.
+
+---
+
+If you'd like, I can also generate:
+- a **sequence diagram** for one full approval cycle,
+- a **state transition table** for each status field,
+- and a **contributor guide** (`CONTRIBUTING.md`) with local test commands.
