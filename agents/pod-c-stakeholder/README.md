# Pod C - Simulated Stakeholder Roleplay Agent

Pod C's agent for the Multi-Agent Training Evaluation Platform. A trainee uploads a project **proposal**. An AI **client persona** (for example a budget-focused CFO) grills them about it in a chat and gets tougher when their answers are weak. When the session ends, the system suggests a **score** with a quote from the chat behind every point, and a human **admin** reviews it before it counts.

> **Status: folder skeleton only.** The folders below exist so each developer can start on their own branch. The code arrives through pull requests (see [Status and next steps](#status-and-next-steps)).

## Contents
1. [What it does](#what-it-does)
2. [Where it fits in the platform](#where-it-fits-in-the-platform)
3. [Architecture at a glance](#architecture-at-a-glance)
4. [Tech stack](#tech-stack)
5. [Repository layout and owners](#repository-layout-and-owners)
6. [Getting started](#getting-started)
7. [Working agreements](#working-agreements)
8. [Design rules we must not break](#design-rules-we-must-not-break)
9. [Parked for later](#parked-for-later)
10. [Status and next steps](#status-and-next-steps)
11. [Glossary](#glossary)

## What it does
1. The trainee uploads a proposal (PDF, DOCX or TXT). Scanned pages are read with OCR. The system builds a short structured **brief** of the proposal's claims, budget, timeline and weak points.
2. The trainee starts a session with a **persona**. The persona opens with a challenge taken from the proposal, then reacts to each answer: vague answers make it tougher, specific and calm answers ease it off.
3. When the session ends (the trainee ends it, the turn limit is reached, or the persona's concerns are all resolved), a scoring job rates the trainee on a few dimensions. Every point in the score cites a real message from the chat.
4. An admin reviews the suggested score and its confidence, approves or overrides it, and a PDF/HTML report is generated.
5. Approved results are made available to Pod D, which diagnoses skill gaps across all pods.

Admins configure the personas (concerns, tone, patience, turn limits) and the scoring (dimensions, weights, thresholds). Every edit creates a new version, and each session records the exact versions it used, so any old score can be explained.

## Where it fits in the platform
```
 Trainee --> [A] Mock Interview   --+
         --> [B] Peer Feedback    --+--> signals + cited evidence --> [D] Skill-Gap Diagnostic
         --> [C] Stakeholder Roleplay (this repo folder) --+
            (an admin reviews each pod's results before they leave the pod)
```
Four pods build four agents and the TPM connects them on Days 8-9. We therefore:
- publish one shared output format, the **Signal Envelope**, so Pod D can build against it early;
- release only **admin-approved** results;
- keep our own database (nothing reads another pod's tables) and let others **pull** results through an API;
- provide a **fallback mode** that returns sample results if our live system is unavailable.

## Architecture at a glance
```
 API (FastAPI) --> LLM Gateway --> LLM provider
   |  |--> PostgreSQL    system of record: sessions, chat turns, scores, configs
   |  |--> Redis         locks, caches, job queue (safe to wipe; nothing important lives only here)
   |  '--> Blob storage  uploaded files and generated reports
 Worker (Celery, same image) <-- jobs via Redis: ingest | assess | report
 Nightly job: delete data 30 days after a session ends
```
- **Chat turns** run inside the API so replies can stream live. **Slow work** (OCR, scoring, PDF) runs as background jobs. Job messages carry IDs only, never proposal text or personal data.
- **One LLM call per chat turn** returns both an evaluation of the trainee's answer and the persona's reply. Plain code then enforces the rules (escalation steps, turn limit). About 11 LLM calls per 8-turn session.
- **The LLM only judges and speaks.** Weighted maths, quote checking, escalation limits and confidence are plain code.
- **No RAG.** A proposal fits in the prompt, and every claim in the brief carries the ids of the proposal chunks it came from.

## Tech stack
Python 3.12 | FastAPI | LangGraph (turn graph and scoring graph) | PostgreSQL + SQLAlchemy + Alembic | Redis + Celery | Tesseract OCR | pytest + ruff | **uv** for environments and dependencies.

## Repository layout and owners
Everything for Pod C lives inside this folder, so other pods' work never collides with ours.

| Folder | Purpose | Owner |
|---|---|---|
| `app/ingest/` | file parsing, OCR, proposal segments, the brief, upload endpoints | **Dev A** |
| `app/reports/` | HTML to PDF report with citations | **Dev A** |
| `app/agent/` | turn graph, persona prompts, escalation guard, turn limit, session logic | **Dev B** |
| `app/scoring/` | scoring graph, quote checker, signals, weighted blend, confidence, review logic, integration endpoint | **Dev C** |
| `app/schemas/` | shared data shapes every module codes against | **Dev D** |
| `app/db/` | table models | **Dev D** |
| `app/llm/` | the LLM interface, a scripted `FakeLLM` for tests, the real provider client later | **Dev D** |
| `app/api/` | API skeleton and shared dependencies (each owner adds their own routers) | **Dev D** |
| `app/workers/` | Celery app and jobs | **Dev D** (job bodies by the dev named on each task) |
| `alembic/versions/` | database migrations. **Only Dev D writes migrations.** | **Dev D** |
| `tests/` | automated tests, runnable without Postgres or Docker | everyone |
| `samples/` | made-up proposals, personas and scoring profiles. Never real data | everyone |
| `docs/` | API description, AI-use log (one file per developer), design notes | everyone |

The shared contracts every owner codes against are `app/schemas/` (data shapes), `app/llm/` (the LLM interface) and `app/db/` (tables). Until real API keys arrive, everything is tested with `FakeLLM`.

## Getting started
> These commands work once the scaffold pull request has been merged.

This project uses [**uv**](https://docs.astral.sh/uv/), a fast tool that replaces `pip`, `venv` and `pip-tools`. It creates the virtual environment, installs exactly the versions in `uv.lock`, and can install Python itself.

```
pip install uv                  # or use the installer from the uv documentation
uv sync                         # creates .venv and installs everything from uv.lock
uv run ruff check .             # lint
uv run pytest                   # tests (35 pass without any database)
uv run uvicorn app.main:app --reload     # API docs at http://localhost:8000/docs
```
Optional, for a real database: `docker compose up -d` then `uv run alembic upgrade head`.
To add a dependency: `uv add <package>` and commit both `pyproject.toml` and `uv.lock`.
**On the company network**, if uv reports an untrusted certificate, set `UV_SYSTEM_CERTS=true` (or add `--system-certs`) so it uses the Windows certificate store. Do not turn verification off.

Settings come from `PODC_*` environment variables; copy `.env.example` to `.env`. `PODC_LLM_PROVIDER=fake` (the default) needs no key.

## Working agreements
- **Nobody pushes to `main`.** Work on a branch named `feature/podc-<area>-<short-desc>`, created from the latest `main`. Changes reach `main` only through a reviewed pull request.
- **Small pull requests** (aim for under about 400 lines). Rebase on `main` before opening. `uv run ruff check .` and `uv run pytest` must pass.
- **Every PR description says** what and why, how it was tested, and what AI help was used.
- **Review:** one teammate reviews (peer review) plus the TPM. Fix every comment the same day.
- **Stay in your own folder.** Changes to `app/schemas/`, `app/db/` and `app/llm/` go in small, separate PRs that Dev D reviews. **Only Dev D writes migrations** (ask for the column you need) so we never get two migration heads.
- **No secrets in git.** `.env` is ignored. Use made-up data only.
- **AI tools (BRD section 5.4):** approved tools only, signed in with your work account; never paste real trainee-style data, keys or secrets; keep your AI-use log in `docs/ai-use-log/` current; be able to explain every line you ship. The TPM may check your AI-tool history on Days 3 and 5.

## Design rules we must not break
- **No ungrounded findings (BRD BR-12, an automatic fail).** Every score cites a real chat message and quotes it. Code verifies the quote is in that message and drops anything that is not. A database link from each piece of evidence to a real turn backs this up.
- **The persona may only challenge what the proposal says.** Its questions cite the proposal chunks they rely on, and code checks those exist.
- **Trainee data is sensitive (BR-13).** Prompts use a pseudonymous `trainee_ref`, never names. Chat data is deleted 30 days after a session ends. Only admin-approved results leave the pod.
- **Proposals are untrusted text.** Only the trainee's own messages are scored, so instructions hidden in a proposal cannot change a score.
- **Nothing fails automatically on turn count.** If concerns are still open at the limit, the assessment is flagged for the admin, who decides.
- **Jobs must be safe to run twice,** and carry IDs only.

## Parked for later
Login and roles, deployment (cloud, Kubernetes, CI/CD) and the user interface are deliberately out of scope for now. Until login exists, the API will use a dev-only fake user and role that refuses to run outside local and dev environments.

## Status and next steps
| Step | Who |
|---|---|
| Scaffold PR: `pyproject.toml` + `uv.lock`, shared schemas, the 12 table models with the first migration, `FakeLLM`, API skeleton, Celery job stubs, sample data and tests (all prepared, 35 passing) | Dev D |
| Proposal parsing, OCR, segments, brief | Dev A |
| Turn graph, persona prompts, escalation guard, session logic | Dev B |
| Quote checker, signals, blend, confidence, scoring graph | Dev C |
| Real LLM provider client, once API keys arrive | Dev D |
| Open questions for the TPM (cloud and budget, login system, scoring rubric, retention, shared skill list) | team lead |

## Glossary
| Term | Meaning |
|---|---|
| **Proposal** | The document the trainee uploads and is questioned about. |
| **Brief** | A structured summary of the proposal, with the chunks each point came from. |
| **Persona** | The AI client: role, company, concerns, tone, patience. |
| **Turn** | One trainee message plus the persona's reply. The persona's opening challenge is turn 0 and is not counted. |
| **Escalation level (0-4)** | How tough the persona currently is. |
| **Session** | One roleplay run by one trainee. |
| **Assessment** | The suggested score for a session. **Evidence** is one cited quote behind it. |
| **Version** | Each admin edit of a persona or scoring profile saves a new numbered version. Sessions record the versions they used. |
| **Cohort (`cohort_id`)** | One batch of trainees going through the program. Used to group results. |
| **`trainee_ref`** | A pseudonymous trainee ID so names never travel between pods. |
| **Signal Envelope** | The shared result format sent to Pod D. |
| **IdP** | Identity provider: the existing login system (parked). |
