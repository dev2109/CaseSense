# CaseSense

A document search and review assistant for a human reviewer. Create a case, upload `.txt` and text-based `.pdf` files, ask questions, and see answers that are grounded in uploaded passages with citations.

**This is not an autonomous decision-making system.** The reviewer decides what to do with the answer.

**Status:** Stage 1 only (architecture and implementation plan). The application is not runnable yet. Backend, frontend, database, and AI features are not implemented.

Uploaded document text will later be sent to the configured AI provider for embeddings and answers. This project does **not** currently provide local-only or private processing.

---

## Stage 1 inspection

Current workspace before this project:

- Cursor started in the user home directory (not an existing app repo).
- There was **no** CaseSense codebase, `package.json`, or Python backend to preserve.
- `C:\Users\Admin\source\repos` existed and was the chosen project location.
- Tooling found on this machine so far: **Python 3.12** (`py -3.12`), **Node.js v20.13.1**, **npm 10.8.0**. Git and GitHub CLI were not on `PATH` at inspection time and are being installed so the repo can be initialized and pushed.

Proposed repository root (this folder):

```text
C:\Users\Admin\source\repos\CaseSense
```

---

## What we will build

A reviewer:

1. Creates a **case**.
2. Uploads `.txt` and text-based `.pdf` documents into that case.
3. Asks a question about those documents.
4. Gets an answer based **only** on uploaded content, plus the document and passage that support each part of the answer.
5. Sees a clear “I couldn’t find that in the uploaded documents” message when evidence is missing.
6. Can view previous questions and answers for the case.

Unsupported in v1: OCR, scanned PDFs, other file types, autonomous actions, and searching across cases.

---

## Architecture (target)

Two apps in one repository:

| Part | Technology | Role |
|------|------------|------|
| `frontend/` | React, TypeScript, Vite, Feature-Sliced Design | UI only. Talks to the FastAPI backend. **Never** calls the AI provider. |
| `backend/` | Python, FastAPI, SQLAlchemy | Cases, uploads, extraction, chunking, retrieval, answers, citations. |
| Database | Microsoft SQL Server | Cases, documents, chunks (including embeddings), Q&A history. Inspect with SSMS. |
| AI | Provider interface (embeddings + LLM) | Replaceable. Model names live in configuration, not scattered in code. |

```text
Browser (React)
    │  HTTP JSON / multipart (local origin only)
    ▼
FastAPI  /api/v1/...
    │
    ├── SQL Server (cases, documents, chunks, Q&A)
    ├── Local upload directory (server-controlled path only)
    └── AI provider adapters (embeddings + chat)
```

### Why this split

- The browser must never see API keys or internal prompts.
- SQL Server stores structured case data you can inspect in SSMS.
- A small provider interface lets you swap OpenAI, Azure OpenAI, or another vendor without rewriting the app.

### Feature-Sliced Design (frontend)

FSD is a way of grouping UI code by **business meaning**, not by “all components in one folder.” Layers (top = more specific):

| Layer | Meaning | Examples in CaseSense |
|-------|---------|------------------------|
| `app` | App startup: routing, providers, global styles | Router, API client provider |
| `pages` | Full screens | Case list page, case detail page |
| `widgets` | Large composed regions used on a page | Document panel, Q&A panel |
| `features` | User actions | Create case, upload document, ask question |
| `entities` | Business objects and API types | Case, document, answer, citation |
| `shared` | Reusable pieces with no product meaning | Button, API client, env config |

**Where new code belongs:** if it is a whole screen, put it in `pages`. If it is one user action (upload), put it in `features`. If it is a type or small display of a Case, put it in `entities`. If many features could reuse it, put it in `shared`. Do not split every tiny button into its own slice.

### Backend layout (planned)

```text
backend/
  app/
    api/v1/          # versioned HTTP routes
    core/            # settings, logging, CORS
    db/              # engine, session, migrations
    models/          # SQLAlchemy tables
    schemas/         # Pydantic request/response models
    services/        # cases, documents, RAG
    providers/       # LLM + embedding interfaces and adapters
    storage/         # upload directory handling
  tests/
  eval/              # synthetic documents and questions
```

### Data model (planned)

- **Case:** id, title, created_at
- **Document:** id, case_id, original filename, file type, processing status, error message, created_at
- **DocumentChunk:** id, document_id, chunk_order, text, page_number (nullable), embedding
- **QuestionAnswer:** id, case_id, question, answer, citations (JSON), created_at

Foreign keys and indexes will be enforced. Uploads will **not** accept a client-supplied filesystem path.

### Retrieval (planned, v1)

Simple RAG **without** a separate vector database:

1. Extract and normalize text.
2. Split into overlapping chunks; keep PDF page numbers when possible.
3. Store chunks and embeddings in SQL Server.
4. For a question, embed the question, score chunks **for that case only** (cosine similarity in application code or equivalent).
5. Send only the question + retrieved passages to the LLM.
6. Validate that cited chunk/document ids were actually retrieved.
7. If nothing is similar enough (configurable threshold), abstain.

**Performance limit:** scoring many chunks in the application (or via SQL without a vector index) is fine for a portfolio-sized case (tens to low hundreds of documents). It will not scale to millions of chunks. The retrieval interface will be replaceable so SQL Server vector search or a dedicated vector DB can be added later.

### AI safety (planned)

- Document text is **untrusted data**, not instructions (including “ignore prior instructions”).
- Answers come only from retrieved passages; no evidence → abstain.
- Structured model output (answer + citations) is validated on the backend.
- No API keys or prompts in the browser.
- Default logging will not include API keys, full document text, or full question text.

---

## Implementation stages

| Stage | Goal | Runnable? |
|-------|------|-----------|
| **1 (this commit)** | Inspect repo, architecture, plan | Docs only |
| **2** | Backend + frontend skeleton, `.env`, SQL Server connection, health check | Yes: empty apps + `/health` |
| **3** | Create and list cases | Yes |
| **4** | TXT/PDF upload, extraction, validation, document status | Yes |
| **5** | Chunking, embeddings, retrieval, answers with citations | Yes |
| **6** | Full UI workflow and loading/empty/error states | Yes |
| **7** | Evaluation set, security checks, complete README | Yes |
| **8** | Acceptance-criteria review and honest gap report | Review |

Each later stage will list files added or changed, how to run it, and setup you must do on Windows.

---

## What you must set up later (not required for Stage 1)

These are documented now so you can prepare. They are **not** wired up yet.

1. **SQL Server** (Developer/Express/LocalDB) and **SSMS**.
2. **ODBC Driver 18 for SQL Server** (typical Windows driver for SQLAlchemy + `pyodbc` or `aioodbc`).
3. An empty database (for example `CaseSense`).
4. An AI provider API key (kept in `.env`, never committed).
5. Backend Python venv and frontend `npm install` (Stage 2).

---

## How to run Stage 1

There is no server to start. Open this folder and review this README.

---

## GitHub

A public GitHub remote is intended after this Stage 1 documentation is committed. Secrets will never be committed. If push fails locally, Git and `gh` must be installed and you must be logged in (`gh auth login`).

---

## OpenAPI

FastAPI’s interactive docs (`/docs`) will be linked here after Stage 2 exists. They do not exist yet.
