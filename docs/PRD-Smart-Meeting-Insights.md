# Product Requirements Document (PRD)

## Smart Meeting Insights System

| Document | Details |
|----------|---------|
| **Version** | 1.0 (MVP) |
| **Status** | Draft |
| **Last updated** | 2026-03-30 |
| **Scope** | Small working MVP |

---

## 1. Executive summary

The Smart Meeting Insights System lets users upload plain-text meeting transcripts, automatically derives **summary**, **action items**, and **decisions** using AI, persists structured results, and surfaces them in a **web dashboard**. The MVP targets a single team or pilot: low operational overhead, clear APIs, and room to scale.

---

## 2. Problem statement

### 2.1 Context

Meeting notes and transcripts are often long, inconsistent, and hard to reuse. Stakeholders waste time re-reading raw text to find what was agreed, who owns follow-ups, and what decisions were made.

### 2.2 Problem

- **Discovery**: Important outcomes are buried in unstructured text.
- **Accountability**: Action items are not consistently captured or traceable to a meeting.
- **Knowledge loss**: Decisions are not stored in a queryable, structured form.

### 2.3 Outcome (MVP success)

Users can upload a transcript file, wait for processing, and reliably see an AI-generated summary, a list of action items (with assignees when inferable), and explicit decisions—**all stored and visible in one dashboard**—without manual copy-paste into multiple tools.

---

## 3. Goals and non-goals (MVP)

### 3.1 Goals

- Accept **text file** uploads (`.txt`; UTF-8).
- Run **AI extraction** for summary, action items, and decisions.
- **Persist** structured output linked to the source meeting/transcript.
- **Dashboard** to list meetings and inspect details.

### 3.2 Non-goals (MVP)

- Real-time meeting capture, audio/video ingestion, or live transcription.
- Calendar integration, SSO enterprise rollout, or fine-grained RBAC beyond a simple auth stub (optional).
- Multi-tenant billing, SLA guarantees, or compliance certifications.
- Full-text search across all transcripts at scale (can be a follow-on).

---

## 4. User personas

| Persona | Role | Needs | MVP behavior |
|---------|------|--------|--------------|
| **Alex – Team lead** | Runs recurring syncs | Quick scan of decisions and open actions | Opens dashboard, opens latest meeting, copies action list |
| **Jordan – IC contributor** | Attends meetings | Know what they must do and by when | Sees action items; optional assignee/due hints from AI |
| **Sam – Ops / PM** | Tracks commitments | Auditable record per meeting | Views stored structured fields; re-opens past meetings |
| **DevOps (internal)** | Ships MVP | Simple deploy, observable failures | .NET API + React + Python service; health checks |

---

## 5. User journeys (step by step)

### 5.1 Upload and process

1. User signs in (MVP: optional single shared API key or basic auth).
2. User selects **Upload transcript** and chooses a `.txt` file.
3. System validates file type and size; creates a **Meeting** record in `Pending` state.
4. Backend enqueues or triggers **AI extraction** (sync for tiny files acceptable in MVP).
5. On success, status becomes **Completed**; structured fields are saved.
6. On failure, status **Failed** with an error message; user may retry upload.

### 5.2 Review in dashboard

1. User opens **Dashboard** → list of meetings (title, date, status).
2. User selects a meeting → detail view: summary, action items, decisions, link to original filename/metadata.
3. User refreshes or navigates back; data loads from API (not only client cache).

---

## 6. Functional requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | Accept upload of a single plain-text transcript file per request (`.txt`, UTF-8). | P0 |
| FR-2 | Enforce maximum file size (MVP default: **5 MB**) and reject unsupported types with clear errors. | P0 |
| FR-3 | Create a meeting/transcript entity with metadata: original filename, upload timestamp, uploader id (if auth exists). | P0 |
| FR-4 | Invoke AI pipeline to produce: **summary** (short paragraph or bullet list), **action items** (text + optional assignee + optional due date), **decisions** (text). | P0 |
| FR-5 | Persist structured output in a database, idempotent per meeting version (MVP: one version per upload). | P0 |
| FR-6 | Expose REST APIs for: create upload, get meeting by id, list meetings (paginated). | P0 |
| FR-7 | Dashboard: list meetings and show detail view with summary, actions, decisions. | P0 |
| FR-8 | Show processing status: Pending, Processing, Completed, Failed. | P0 |
| FR-9 | (Optional P1) Export structured result as JSON from UI or API. | P1 |

### 6.1 AI output schema (logical)

- **Summary**: string (markdown or plain text; MVP: plain text).
- **Action items**: array of `{ text, assignee?: string, dueDate?: string (ISO date) }`.
- **Decisions**: array of `{ text }`.

Validation: empty arrays allowed; summary may be short if transcript is thin; surface model uncertainty in UI only if API returns a confidence flag (optional P1).

---

## 7. Non-functional requirements

| ID | Category | Requirement | MVP note |
|----|----------|-------------|----------|
| NFR-1 | **Performance** | P95 **&lt; 2 seconds** for **read** APIs (`GET` meeting, `GET` list) under nominal load. | Excludes AI processing time; cache-friendly payloads. |
| NFR-2 | **Performance** | AI extraction latency: not capped at 2s; target **&lt; 60s** P95 for typical 10-page transcript in MVP; async job acceptable. | If sync, document timeout clearly in API. |
| NFR-3 | **Scalability** | Architecture supports **horizontal scaling** of stateless API instances and **worker** processes for AI jobs. | Single instance OK for pilot; no shared in-memory-only state for job status. |
| NFR-4 | **Availability** | MVP: best-effort; define **health endpoints** for API and Python service. | |
| NFR-5 | **Security** | Transcripts may contain sensitive data: TLS in transit; secrets in env/config; no logging of full transcript bodies by default. | |
| NFR-6 | **Observability** | Structured logs with `meetingId`, correlation id; basic metrics (request count, errors, job duration). | |

---

## 8. API design

Base URL: `https://api.example.com/v1` (MVP: configurable host).

### 8.1 Authentication (MVP)

- Header: `Authorization: Bearer <token>` or API key `X-Api-Key` (choose one for MVP).  Let us use - Header: `Authorization: Bearer <token>`

### 8.2 Endpoints

#### `POST /meetings/upload`

**Purpose**: Upload transcript and start processing.

- **Content-Type**: `multipart/form-data`
- **Fields**: `file` (required), optional `title` (string)

**Response** `202 Accepted` (async) or `201 Created` (sync MVP): - let use `202 Accepted` (async)

```json
{
  "meetingId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "Processing",
  "title": "Sprint planning",
  "originalFileName": "planning-2026-03-30.txt"
}
```

**Errors**: `400` validation, `413` payload too large, `415` unsupported media type.

---

#### `GET /meetings`

**Purpose**: Paginated list for dashboard.

**Query**: `page` (default 1), `pageSize` (default 20, max 100), optional `status`.

**Response** `200 OK`:

```json
{
  "items": [
    {
      "meetingId": "550e8400-e29b-41d4-a716-446655440000",
      "title": "Sprint planning",
      "status": "Completed",
      "createdAtUtc": "2026-03-30T14:00:00Z",
      "updatedAtUtc": "2026-03-30T14:00:05Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1
}
```

**NFR**: P95 &lt; 2s including DB read (indexed queries).

---

#### `GET /meetings/{meetingId}`

**Purpose**: Full detail including structured insights.

**Response** `200 OK`:

```json
{
  "meetingId": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Sprint planning",
  "status": "Completed",
  "originalFileName": "planning-2026-03-30.txt",
  "createdAtUtc": "2026-03-30T14:00:00Z",
  "updatedAtUtc": "2026-03-30T14:00:05Z",
  "insights": {
    "summary": "Team aligned on sprint goal...",
    "actionItems": [
      { "text": "Update API docs", "assignee": "Jordan", "dueDate": "2026-04-02" }
    ],
    "decisions": [
      { "text": "We will postpone feature X to next sprint." }
    ]
  },
  "errorMessage": null
}
```

**Errors**: `404` if not found.

---

#### `GET /health`

**Purpose**: Liveness for .NET API.

**Response** `200 OK`: `{ "status": "Healthy" }`

---

### 8.3 Internal / service-to-service (optional MVP)

- **.NET → Python (Polars)**: `POST /analyze` with JSON body `{ "meetingId", "text" }` returning structured JSON matching `insights` (or raw AI output before normalization). Use shared secret or internal network only.

---

## 9. High-level architecture

### 9.1 Overview

| Layer | Technology | Responsibility |
|-------|------------|----------------|
| **Frontend** | React (TypeScript) | Upload UI, dashboard, meeting detail, calls REST API |
| **Backend API** | **.NET** (ASP.NET Core) | Auth boundary, uploads, validation, orchestration, persistence, job status |
| **Analytics / NLP helper** | **Python** service using **Polars** | Optional preprocessing (chunking, tabular stats, text normalization) before/after LLM call; batch-friendly transforms |
| **AI** | LLM API (OpenAI-compatible or Azure OpenAI) | Extraction of summary, actions, decisions (invoked from .NET or Python per team preference) |
| **Data** | Relational DB (e.g. PostgreSQL) | Meetings, insights, optional job table |

### 9.2 Request flow (MVP)

```text
[React] --multipart--> [.NET API] --> save transcript metadata + blob/path
                           |
                           +--> [Python Polars service] (optional preprocess)
                           |
                           +--> [LLM] --> structured JSON
                           |
                           +--> persist insights --> [DB]
[React] <---- JSON ---- [.NET API]  (GET list / GET detail)
```

### 9.3 Component notes (practical MVP)

- **.NET** owns **all external contracts** (public REST), **file storage** (local disk or blob in phase 2), and **transactions** when saving meeting + insights.
- **Python + Polars**: use for **fast dataframe-style operations** if you later add batch reprocessing, quality metrics, or columnar exports; for MVP it can be a thin HTTP service that **delegates** to the LLM after light text cleanup.
- **React**: single-page app; use React Query (or similar) for caching list/detail to help meet **&lt; 2s** perceived load on revisits.

### 9.4 Deployment sketch

- Containers: `api` (.NET), `web` (React static + nginx), `worker` (optional separate .NET worker or same API with background queue), `python-analytics` (Python).
- MVP on one VM or small Kubernetes: 1 replica each; scale API replicas when read traffic grows; scale workers when AI queue depth grows.

---

## 10. Database schema

Rationale: normalized core entities + JSON for flexible AI fields (MVP speed); migrate to dedicated tables if reporting needs grow.

### 10.1 Tables

#### `Meetings`

| Column | Type | Notes |
|--------|------|--------|
| `Id` | UUID, PK | |
| `Title` | NVARCHAR(500) | Nullable; default from filename |
| `OriginalFileName` | NVARCHAR(500) | |
| `StoragePath` | NVARCHAR(1024) | Or blob URI |
| `Status` | TINYINT or VARCHAR | Pending, Processing, Completed, Failed |
| `ErrorMessage` | NVARCHAR(2000) | Nullable |
| `CreatedAtUtc` | DATETIME2 | |
| `UpdatedAtUtc` | DATETIME2 | |
| `CreatedByUserId` | UUID, nullable | If auth enabled |

**Indexes**: `IX_Meetings_CreatedAtUtc DESC`, `IX_Meetings_Status`.

---

#### `MeetingInsights`

| Column | Type | Notes |
|--------|------|--------|
| `MeetingId` | UUID, PK, FK → Meetings.Id | One row per meeting for MVP |
| `Summary` | NVARCHAR(MAX) | |
| `ActionItemsJson` | NVARCHAR(MAX) | JSON array |
| `DecisionsJson` | NVARCHAR(MAX) | JSON array |
| `ModelName` | NVARCHAR(200) | For traceability |
| `PromptVersion` | NVARCHAR(50) | Optional |
| `GeneratedAtUtc` | DATETIME2 | |

---

#### `ProcessingJobs` (optional but recommended for async MVP)

| Column | Type | Notes |
|--------|------|--------|
| `Id` | UUID, PK | |
| `MeetingId` | UUID, FK | |
| `State` | VARCHAR(32) | Queued, Running, Succeeded, Failed |
| `Attempts` | INT | |
| `StartedAtUtc` | DATETIME2, nullable | |
| `CompletedAtUtc` | DATETIME2, nullable | |

---

### 10.2 Example JSON payloads in columns

**ActionItemsJson**

```json
[
  { "text": "Ship MVP dashboard", "assignee": "Alex", "dueDate": "2026-04-05" },
  { "text": "Review PRD", "assignee": null, "dueDate": null }
]
```

**DecisionsJson**

```json
[
  { "text": "Use PostgreSQL for MVP database." }
]
```

---

## 11. Risks and mitigations

| Risk | Impact | Mitigation (MVP) |
|------|--------|------------------|
| **LLM hallucinations** | Wrong actions/decisions | Prompt constraints (“only from transcript”); show disclaimer in UI; optional human edit (P1); store `modelName` / `promptVersion`. |
| **PII / sensitive data in transcripts** | Compliance / leaks | Minimize logs; encrypt at rest if cloud blob; access control on API; document “do not upload classified data” for pilot. |
| **AI latency &gt; 2s** | User confusion | Use **async** upload (`202`) + polling or SignalR later; keep **read** APIs under 2s. |
| **Cost spikes (token usage)** | Budget | Cap file size; truncate very long inputs with clear error or chunking strategy; rate limit uploads per user/key. |
| **Python service failure** | Processing stuck | Timeouts from .NET; retry with backoff; mark `Failed` with message; health checks. |
| **Single DB bottleneck** | Scale limits | Indexed queries; connection pooling; read replicas later if needed. |
| **Schema drift in JSON** | Client breaks | Version field in JSON or API; validate on write in .NET. |

---

## 12. Milestones (suggested MVP timeline)

1. **M1**: .NET API + DB schema + file upload + stubbed insights (fixed JSON).
2. **M2**: Real LLM integration + persist `MeetingInsights`.
3. **M3**: Python Polars service wired (preprocess or post-process).
4. **M4**: React dashboard (list + detail + upload).
5. **M5**: Hardening—health checks, basic metrics, error states, README for local run.

---

## 13. Open questions are answered with resolved decisions - 

1. LLM provider and deployment
For MVP, use a hosted open-weight LLM API to minimize infrastructure complexity and accelerate delivery. Support Ollama locally for development/testing. Defer private/self-hosted deployment until compliance or scale requirements justify the operational overhead.

2. Raw transcript storage
For MVP, store raw transcript text in the relational database with strict file-size limits and secure handling. As scale grows, move raw transcript storage to object storage and keep metadata and structured insights in the database.

3. Authentication
Implement API key authentication from day one. This is sufficient for MVP protection of transcript upload and AI-processing endpoints, while full user auth and RBAC are deferred to a later phase.

---

*End of PRD*
