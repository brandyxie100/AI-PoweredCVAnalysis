# AI-Powered CV Analysis — PR Feature Plan

This document defines all Pull Requests required to complete the AI-Powered CV Analysis platform. Each PR follows the conventions in [01-git-workflow-guide.md](../01-git-workflow-guide.md) and implements sections of [02-system-architecture.md](../02-system-architecture.md).

**Rules applied to every PR:**

- Branch from `develop`, target `develop`.
- Branch naming: `feature/<scope>-<description>`.
- PR title: Conventional Commits `feat(scope): description`.
- Size: < 400 lines of changed code.
- Tests included (test-first workflow).
- PR description includes Summary, Motivation, Changes, Testing.
- Squash and merge when approved.

---

## Implementation Status

| Area | Status | Notes |
|------|--------|-------|
| Backend — config & models | ✅ Implemented | AppConfig singleton, Pydantic schemas |
| Backend — document loader | ✅ Implemented | Factory pattern, PDF/DOCX/TXT |
| Backend — text chunker | ✅ Implemented | LangChain RecursiveCharacterTextSplitter |
| Backend — cv extractor | ✅ Implemented | ChatAnthropic + structured output |
| Backend — job matcher | ✅ Implemented | FAISS + HuggingFaceEmbeddings |
| Backend — recommender | ✅ Implemented | LCEL chain |
| Backend — agent | ✅ Implemented | LangGraph ReAct + 4 tools |
| Backend — cv analyzer & API | ✅ Implemented | Pipeline orchestrator, FastAPI routes |
| Frontend — API client & types | ✅ Implemented | lib/api.ts, types/index.ts |
| Frontend — components & pages | ✅ Implemented | FileUpload, AnalysisResults, page.tsx |
| Docker & deployment | ✅ Implemented | docker-compose, Dockerfiles |

---

## Dependency Graph

```
                    BACKEND (Python / FastAPI / LangChain)

  B1 (config + schemas)
    ├── B2 (document loader)
    │     └── B7 (cv analyzer) ◄─────────────────────────┐
    ├── B3 (text chunker) ──────────────────────────────┤
    ├── B4 (cv extractor) ──────────────────────────────┤
    ├── B5 (job matcher) ───────────────────────────────┤
    ├── B6 (recommender) ───────────────────────────────┤
    │     └─────────────────────────────────────────────┤
    └── B8 (agent + tools) ── depends on B7 ───────────┤
              │                                           │
              └──────────────────► B9 (main + routes) ◄──┘
                                        │
                                        ▼
                    FRONTEND (Next.js / TypeScript / Tailwind)

  F1 (api client + types) ──► F2 (upload + layout)
                                    └──► F3 (results + page)
```

---

## Timeline (Aligns with Waterfall / Agile)

| Week | PRs |
|------|-----|
| **Sprint 0** (Requirements & Design) | — |
| **Sprint 1** (Implementation) | B1 → B2, B3, B4, B5, B6 (parallel) → B7 → B8 → B9 ; F1 → F2 → F3 |
| **Sprint 2** (Testing & Deployment) | T1 (tests), T2 (Docker) |

---

# Part 1: Backend PRs

## B1 — Config & Schemas

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-config-schemas` |
| **PR Title** | `feat(backend): add AppConfig singleton and Pydantic schemas` |
| **Target** | `develop` |
| **Est. Lines** | ~280 |
| **Depends On** | — |

### PR Description

```markdown
## Summary
- Implements AppConfig singleton (loads .env via python-dotenv)
- Implements get_llm() (ChatAnthropic) and get_embeddings() (HuggingFaceEmbeddings)
- Implements Pydantic schemas: CVUploadResponse, CVAnalysisResult, AgentQueryRequest/Response, HealthResponse, sub-models (ExtractedSkill, WorkExperience, JobMatch, Recommendation)

## Motivation
Foundation for all backend services. Config provides LLM and embeddings; schemas enforce API contract.

## Changes
- `Backend/app/config.py` — AppConfig singleton
- `Backend/app/models/schemas.py` — All request/response models
- `Backend/app/models/__init__.py`
- `Backend/tests/conftest.py` — Pytest config, AppConfig.reset() fixture

## Testing
- [x] Unit test: AppConfig returns same instance (singleton)
- [x] Unit test: AppConfig.reset() creates new instance
- [x] Unit test: default config values sensible
```

### Files Touched

- `Backend/app/config.py`
- `Backend/app/models/schemas.py`
- `Backend/app/models/__init__.py`
- `Backend/tests/conftest.py`

---

## B2 — Document Loader (Factory)

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-document-loader` |
| **PR Title** | `feat(backend): add document loader with Factory pattern` |
| **Target** | `develop` |
| **Est. Lines** | ~180 |
| **Depends On** | B1 |

### PR Description

```markdown
## Summary
- Implements BaseDocumentLoader ABC, PDFLoader, DocxLoader, TxtLoader
- Implements DocumentLoaderFactory.create_loader() for extension-based selection
- Supports PDF, DOCX, TXT

## Motivation
Pipeline stage 1 — load. Factory pattern allows adding formats without changing callers.

## Changes
- `Backend/app/services/document_loader.py` — All loader classes + factory
- `Backend/app/services/__init__.py`
- `Backend/tests/test_loader.py` — Factory selection, TxtLoader, DocxLoader, unsupported format, missing file

## Testing
- [x] Factory returns correct loader for .pdf, .docx, .txt
- [x] Factory raises ValueError for unsupported format
- [x] Factory raises FileNotFoundError for missing file
- [x] TxtLoader reads content correctly
- [x] DocxLoader extracts paragraphs
```

### Files Touched

- `Backend/app/services/document_loader.py`
- `Backend/app/services/__init__.py`
- `Backend/tests/test_loader.py`

---

## B3 — Text Chunker

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-text-chunker` |
| **PR Title** | `feat(backend): add text chunker with LangChain RecursiveCharacterTextSplitter` |
| **Target** | `develop` |
| **Est. Lines** | ~130 |
| **Depends On** | B1 |

### PR Description

```markdown
## Summary
- Implements TextChunker wrapping RecursiveCharacterTextSplitter
- Configurable chunk_size and chunk_overlap via AppConfig
- split() and split_with_metadata() methods

## Motivation
Pipeline stage 2 — chunk. Prepares text for embedding and LLM context.

## Changes
- `Backend/app/services/text_chunker.py`
- `Backend/tests/test_analyzer.py` — Add TextChunker tests (split produces chunks, preserves content, metadata)

## Testing
- [x] split() produces multiple chunks for long text
- [x] All original content appears in combined chunks
- [x] split_with_metadata() returns dicts with content + metadata
- [x] Empty text returns empty list
```

### Files Touched

- `Backend/app/services/text_chunker.py`
- `Backend/tests/test_analyzer.py`

---

## B4 — CV Extractor

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-cv-extractor` |
| **PR Title** | `feat(backend): add CV extractor with Claude structured output` |
| **Target** | `develop` |
| **Est. Lines** | ~150 |
| **Depends On** | B1 |

### PR Description

```markdown
## Summary
- Implements CVExtractorService with ChatAnthropic + with_structured_output
- Extracts candidate_name, email, summary, skills, experience, education, overall_quality_score
- Uses CVExtraction Pydantic schema

## Motivation
Pipeline stage 3 — extract. Produces typed structured data from raw CV text.

## Changes
- `Backend/app/services/cv_extractor.py` — CVExtractorService, CVExtraction schema
- `Backend/app/models/schemas.py` — Ensure ExtractedSkill, WorkExperience, Education compatible

## Testing
- [x] Unit test with mocked LLM response (integration test in B9)
```

### Files Touched

- `Backend/app/services/cv_extractor.py`
- `Backend/app/models/schemas.py`

---

## B5 — Job Matcher (FAISS)

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-job-matcher` |
| **PR Title** | `feat(backend): add job matcher with FAISS vector store` |
| **Target** | `develop` |
| **Est. Lines** | ~230 |
| **Depends On** | B1 |

### PR Description

```markdown
## Summary
- Implements JobMatcherService with FAISS + HuggingFaceEmbeddings
- Builds index from JOB_CATALOGUE (10 job roles)
- match() returns top-k JobMatch with similarity scores

## Motivation
Pipeline stage 4 — match. Semantic similarity between CV and job descriptions.

## Changes
- `Backend/app/services/job_matcher.py` — JobMatcherService, JOB_CATALOGUE

## Testing
- [x] build_index() creates FAISS index
- [x] match() returns list of JobMatch with valid scores
- [x] match() normalises distance to 0–1 similarity
```

### Files Touched

- `Backend/app/services/job_matcher.py`

---

## B6 — Recommender

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-recommender` |
| **PR Title** | `feat(backend): add recommender with LCEL chain` |
| **Target** | `develop` |
| **Est. Lines** | ~190 |
| **Depends On** | B1 |

### PR Description

```markdown
## Summary
- Implements RecommenderService with ChatPromptTemplate | ChatAnthropic | StrOutputParser
- Generates JSON array of Recommendation (category, suggestion, priority)
- Parses LLM output, fallback on parse failure

## Motivation
Pipeline stage 5 — recommend. Produces actionable improvement suggestions.

## Changes
- `Backend/app/services/recommender.py` — RecommenderService, _parse_recommendations

## Testing
- [x] Unit test with mocked LLM JSON output
- [x] _parse_recommendations handles malformed JSON gracefully
```

### Files Touched

- `Backend/app/services/recommender.py`

---

## B7 — CV Analyzer (Orchestrator)

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-cv-analyzer` |
| **PR Title** | `feat(backend): add CVAnalyzer pipeline orchestrator` |
| **Target** | `develop` |
| **Est. Lines** | ~220 |
| **Depends On** | B2, B3, B4, B5, B6 |

### PR Description

```markdown
## Summary
- Implements CVAnalyzer orchestrating load → chunk → extract → match → recommend
- upload() uses DocumentLoaderFactory, returns CVUploadResponse
- analyze(file_id) runs full pipeline, returns CVAnalysisResult
- In-memory store for file_id → text, chunks

## Motivation
Central coordinator. Ties all pipeline stages into one API-callable workflow.

## Changes
- `Backend/app/services/cv_analyzer.py` — CVAnalyzer, _build_match_query

## Testing
- [x] Integration test: upload then analyze returns CVAnalysisResult
- [x] analyze() with unknown file_id raises ValueError
```

### Files Touched

- `Backend/app/services/cv_analyzer.py`
- `Backend/tests/test_analyzer.py`

---

## B8 — Agent & Tools

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-agent-tools` |
| **PR Title** | `feat(backend): add LangGraph ReAct agent with CV tools` |
| **Target** | `develop` |
| **Est. Lines** | ~420 |
| **Depends On** | B7 |

### PR Description

```markdown
## Summary
- Implements 4 tools: get_cv_full_text, get_cv_chunks, search_cv_section, analyze_cv_formatting
- Implements CVAgentService with create_react_agent
- set_analyzer() injects CVAnalyzer into tools

## Motivation
Interactive Q&A about uploaded CVs. Agent autonomously selects tools.

## Changes
- `Backend/app/tools/cv_tools.py` — 4 @tool functions, ALL_CV_TOOLS
- `Backend/app/tools/__init__.py`
- `Backend/app/services/agent.py` — CVAgentService, AGENT_SYSTEM_PROMPT

## Testing
- [x] Agent invokes tools when question requires CV data
- [x] query() returns AgentQueryResponse with answer and tool_calls
```

### Files Touched

- `Backend/app/tools/cv_tools.py`
- `Backend/app/tools/__init__.py`
- `Backend/app/services/agent.py`

> **Note:** Slightly over 400 lines. Consider splitting into B8a (tools) and B8b (agent) if strict compliance required.

---

## B9 — FastAPI Main & Routes

| Field | Value |
|-------|-------|
| **Branch** | `feature/backend-api-routes` |
| **PR Title** | `feat(backend): add FastAPI app with upload, analyze, agent endpoints` |
| **Target** | `develop` |
| **Est. Lines** | ~240 |
| **Depends On** | B7, B8 |

### PR Description

```markdown
## Summary
- Implements FastAPI app with lifespan (CVAnalyzer, CVAgentService init)
- Endpoints: GET /health, POST /upload, GET /analyze/{file_id}, POST /agent/query
- CORS middleware for Frontend
- File save to UPLOAD_DIR

## Motivation
Exposes all functionality via REST API. Entry point for Frontend.

## Changes
- `Backend/app/main.py` — app, lifespan, routes, CORS

## Testing
- [x] GET /health returns 200
- [x] POST /upload with unsupported format returns 400
- [x] POST /upload with valid .txt returns 200 + file_id
- [x] GET /analyze/{unknown_id} returns 404
```

### Files Touched

- `Backend/app/main.py`
- `Backend/tests/test_analyzer.py`

---

# Part 2: Frontend PRs

## F1 — API Client & Types

| Field | Value |
|-------|-------|
| **Branch** | `feature/frontend-api-client` |
| **PR Title** | `feat(frontend): add API client and TypeScript types` |
| **Target** | `develop` |
| **Est. Lines** | ~120 |
| **Depends On** | — |

### PR Description

```markdown
## Summary
- Implements lib/api.ts: checkHealth, uploadCV, analyzeCV, queryAgent
- Implements types/index.ts matching Backend Pydantic schemas
- NEXT_PUBLIC_API_URL from env, default localhost:8000

## Motivation
Centralised API layer. Types ensure Frontend/Backend contract alignment.

## Changes
- `Frontend/src/lib/api.ts`
- `Frontend/src/types/index.ts`

## Testing
- [x] checkHealth() returns HealthResponse when backend up
- [x] uploadCV() with valid file returns CVUploadResponse
```

### Files Touched

- `Frontend/src/lib/api.ts`
- `Frontend/src/types/index.ts`

---

## F2 — Upload UI & Layout

| Field | Value |
|-------|-------|
| **Branch** | `feature/frontend-upload-ui` |
| **PR Title** | `feat(frontend): add FileUpload component and app layout` |
| **Target** | `develop` |
| **Est. Lines** | ~200 |
| **Depends On** | F1 |

### PR Description

```markdown
## Summary
- Implements FileUpload with react-dropzone (PDF/DOCX/TXT)
- Implements Header component
- Implements layout.tsx, globals.css, Tailwind config

## Motivation
User entry point. Drag-and-drop or click-to-browse CV upload.

## Changes
- `Frontend/src/components/FileUpload.tsx`
- `Frontend/src/components/Header.tsx`
- `Frontend/src/app/layout.tsx`
- `Frontend/src/app/globals.css`
- `Frontend/tailwind.config.ts`

## Testing
- [x] FileUpload renders, accepts valid file types
- [x] Invalid file type shows error or is rejected
```

### Files Touched

- `Frontend/src/components/FileUpload.tsx`
- `Frontend/src/components/Header.tsx`
- `Frontend/src/app/layout.tsx`
- `Frontend/src/app/globals.css`
- `Frontend/tailwind.config.ts`

---

## F3 — Results & Main Page

| Field | Value |
|-------|-------|
| **Branch** | `feature/frontend-results-page` |
| **PR Title** | `feat(frontend): add AnalysisResults, LoadingSpinner, and main page workflow` |
| **Target** | `develop` |
| **Est. Lines** | ~350 |
| **Depends On** | F1, F2 |

### PR Description

```markdown
## Summary
- Implements LoadingSpinner with pipeline stages
- Implements AnalysisResults (score, skills, experience, job matches, recommendations, agent Q&A)
- Implements page.tsx: upload → analyzing → results flow

## Motivation
Complete user journey. Display analysis and allow agent questions.

## Changes
- `Frontend/src/components/LoadingSpinner.tsx`
- `Frontend/src/components/AnalysisResults.tsx`
- `Frontend/src/app/page.tsx`

## Testing
- [x] Upload triggers analyze, shows LoadingSpinner
- [x] AnalysisResults renders all sections when result provided
- [x] Agent Q&A input sends request, displays answer
- [x] "Analyse Another CV" resets state
```

### Files Touched

- `Frontend/src/components/LoadingSpinner.tsx`
- `Frontend/src/components/AnalysisResults.tsx`
- `Frontend/src/app/page.tsx`

---

# Part 3: DevOps & Testing PRs

## T1 — Backend & Frontend Tests

| Field | Value |
|-------|-------|
| **Branch** | `feature/tests-coverage` |
| **PR Title** | `test: add comprehensive backend and frontend tests` |
| **Target** | `develop` |
| **Est. Lines** | ~250 |
| **Depends On** | B9, F3 |

### PR Description

```markdown
## Summary
- Backend: test_loader.py, test_analyzer.py (unit + integration)
- Frontend: Jest/Vitest or E2E for critical flows (optional)

## Motivation
Meet git workflow requirement: tests included in every PR. This PR adds/consolidates coverage.

## Changes
- `Backend/tests/test_loader.py` — Document loader tests
- `Backend/tests/test_analyzer.py` — Config, chunker, endpoints

## Testing
- [x] pytest Backend/tests -v passes
- [x] Coverage report generated
```

### Files Touched

- `Backend/tests/test_loader.py`
- `Backend/tests/test_analyzer.py`
- `Backend/tests/conftest.py`

---

## T2 — Docker & Deployment

| Field | Value |
|-------|-------|
| **Branch** | `feature/docker-deployment` |
| **PR Title** | `feat(infra): add Dockerfile and docker-compose for Backend and Frontend` |
| **Target** | `develop` |
| **Est. Lines** | ~150 |
| **Depends On** | B9, F3 |

### PR Description

```markdown
## Summary
- Backend Dockerfile (multi-stage Python)
- Frontend Dockerfile (multi-stage Node.js)
- docker-compose.yml (backend + frontend services)
- entrypoint.sh for volume permissions
- .env.example, .gitignore updates

## Motivation
One-command deployment. `docker compose up --build` runs full stack.

## Changes
- `Backend/Dockerfile`
- `Backend/entrypoint.sh`
- `Frontend/Dockerfile`
- `docker-compose.yml`
- `.env.example`
- `.gitignore`

## Testing
- [x] docker compose up --build succeeds
- [x] Frontend at localhost:3000, Backend at localhost:8000
- [x] POST /upload from Frontend works
```

### Files Touched

- `Backend/Dockerfile`
- `Backend/entrypoint.sh`
- `Frontend/Dockerfile`
- `docker-compose.yml`
- `.env.example`
- `.gitignore`

---

# Summary Tables

## Backend PRs

| PR | Branch | Title | Est. Lines | Depends On |
|----|--------|-------|-----------|------------|
| B1 | `feature/backend-config-schemas` | `feat(backend): add AppConfig singleton and Pydantic schemas` | ~280 | — |
| B2 | `feature/backend-document-loader` | `feat(backend): add document loader with Factory pattern` | ~180 | B1 |
| B3 | `feature/backend-text-chunker` | `feat(backend): add text chunker with LangChain RecursiveCharacterTextSplitter` | ~130 | B1 |
| B4 | `feature/backend-cv-extractor` | `feat(backend): add CV extractor with Claude structured output` | ~150 | B1 |
| B5 | `feature/backend-job-matcher` | `feat(backend): add job matcher with FAISS vector store` | ~230 | B1 |
| B6 | `feature/backend-recommender` | `feat(backend): add recommender with LCEL chain` | ~190 | B1 |
| B7 | `feature/backend-cv-analyzer` | `feat(backend): add CVAnalyzer pipeline orchestrator` | ~220 | B2–B6 |
| B8 | `feature/backend-agent-tools` | `feat(backend): add LangGraph ReAct agent with CV tools` | ~420 | B7 |
| B9 | `feature/backend-api-routes` | `feat(backend): add FastAPI app with upload, analyze, agent endpoints` | ~240 | B7, B8 |

**Backend total:** ~2,040 lines across 9 PRs

## Frontend PRs

| PR | Branch | Title | Est. Lines | Depends On |
|----|--------|-------|-----------|------------|
| F1 | `feature/frontend-api-client` | `feat(frontend): add API client and TypeScript types` | ~120 | — |
| F2 | `feature/frontend-upload-ui` | `feat(frontend): add FileUpload component and app layout` | ~200 | F1 |
| F3 | `feature/frontend-results-page` | `feat(frontend): add AnalysisResults, LoadingSpinner, and main page workflow` | ~350 | F1, F2 |

**Frontend total:** ~670 lines across 3 PRs

## DevOps PRs

| PR | Branch | Title | Est. Lines | Depends On |
|----|--------|-------|-----------|------------|
| T1 | `feature/tests-coverage` | `test: add comprehensive backend and frontend tests` | ~250 | B9, F3 |
| T2 | `feature/docker-deployment` | `feat(infra): add Dockerfile and docker-compose` | ~150 | B9, F3 |

**DevOps total:** ~400 lines across 2 PRs

## Combined Total

| Area | PRs | Est. Lines |
|------|-----|-----------|
| Backend | 9 | ~2,040 |
| Frontend | 3 | ~670 |
| DevOps | 2 | ~400 |
| **Total** | **14** | **~3,110** |

---

## Git Workflow Compliance Checklist

| Requirement | Applied |
|-------------|---------|
| Branch from `develop`, target `develop` | ✅ All PRs |
| Branch naming `feature/<scope>-<description>` | ✅ All branches |
| PR title `feat(scope): description` | ✅ All PR titles |
| Size < 400 lines | ⚠️ B8 ~420 (consider split) |
| Tests included | ✅ B2, B3, B7, B9, T1 |
| PR description: Summary, Motivation, Changes, Testing | ✅ Template in each PR |
| Squash and merge | ✅ Per workflow |
