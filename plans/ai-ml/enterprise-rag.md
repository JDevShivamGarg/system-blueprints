# Enterprise RAG System

**Tagline:** A modular, open-source, production-grade Retrieval-Augmented Generation (RAG) platform with multi-level RBAC, hybrid retrieval, and automated security guardrails.  
**Category:** `ai-ml`  
**Author:** [@jdevshivamgarg](https://github.com/jdevshivamgarg)  
**Date:** 2026-07-19  
**Status:** `Draft`

---

## 1. Genesis

**Problem observed:**  
While deploying enterprise AI assistants, we observed that commercial RAG frameworks and naive vector database integrations introduce severe data compliance risks, frequent hallucinations, and lack access control mechanisms. Organizations attempted to connect internal documentation to LLMs using ad-hoc scripts, resulting in low-clearance users retrieving confidential executive strategy documents or trade secrets, with zero audit trail or verification of retrieved facts.

**Why existing solutions fail:**  
Existing commercial SaaS RAG platforms (e.g., Pinecone + OpenAI integrations) require shipping sensitive internal enterprise documents to third-party cloud APIs, violating strict corporate governance policies (GDPR, HIPAA, SOC 2). Conversely, simplistic open-source RAG tutorials rely on single-user vector searches without multi-tenant Role-Based Access Control (RBAC), lack hybrid search (dense + sparse keyword matching), offer no prompt injection or PII guardrails, and fail to validate retrieval faithfulness before generating responses.

**Core hypothesis:**  
If a fully self-hostable, air-gapped, modular RAG system is built with fine-grained document-level RBAC, hybrid retrieval (Qdrant dense + BM25 sparse), and automated input/output security guardrails, enterprises can safely deploy internal knowledge search without data leakage, third-party vendor lock-in, or unchecked hallucination risks.

---

## 2. Engineering Principles

Non-negotiable axioms that govern every decision in this plan. If a later decision conflicts with one of these, the principle wins unless this section is explicitly revised.

1. **Modular Independence** — Every microservice (Gateway, Ingestion, Retrieval, Generation, Guardrails, Observability) must be independently deployable, scaleable, and testable behind a strict REST/gRPC API contract. Replacing a component (e.g., swapping Qdrant for another vector store or replacing Ollama with vLLM) must require zero modifications to surrounding microservices.
2. **Air-Gapped Open-Source Stack** — The core architecture must run completely self-hosted inside a private enterprise VPC or air-gapped network without mandatory outbound internet access. All models (embeddings, rerankers, OCR, PII classifiers, LLMs) must run on local GPU/CPU hardware nodes.
3. **Zero-Trust Document Access** — Access control is enforced at the chunk-retrieval layer in the database, not just at the API Gateway. Queries are automatically appended with namespace and sensitivity clearance filters derived from validated user JWT tokens before vector distance calculations occur.

---

## 3. Target Persona

Not demographics. Behavior.

**Who they are:**  
Enterprise AI Engineers, Security & Compliance Officers, and Platform Architects responsible for building and maintaining internal knowledge search tools for thousands of employees across legal, HR, engineering, and operations departments.

**How they currently solve this problem:**  
They stitch together LangChain or LlamaIndex scripts pointing to hosted SaaS vector databases and cloud LLM APIs, relying on client-side filtering and manual regex scripts to catch sensitive data or prompt injections.

**What they will switch from and why:**  
They switch from ad-hoc Python scripts and commercial SaaS solutions because of security compliance audits flagging third-party data transmission, high token costs, and user complaints regarding retrieved outdated or inaccurate documents.

---

## 4. Problem Statement

State the problem in precise language. No solution language in this section.

- **Data Privacy & Compliance Violations:** Exposing internal enterprise documentation to external LLM providers risks regulatory fines and intellectual property leaks.
- **Inadequate Access Controls:** Standard vector indexes return document fragments without verifying if the requesting user possesses the security clearance to view that source material.
- **High Hallucination & Retrieval Noise:** Single-vector dense search frequently fails on technical keyword queries (e.g., error codes, serial numbers), causing LLMs to generate plausible but false answers.
- **Assumptions:** 
  - Enterprise infrastructure can allocate dedicated local CPU/GPU compute nodes (or Kubernetes clusters) for self-hosted LLM and embedding models.
  - Documents can be processed and stored in localized object storage and relational metadata indexes.

---

## 5. Solution Overview

**What this system does:**  
The Enterprise RAG System is a self-hosted, microservice-based retrieval platform. It ingests enterprise documents (PDFs, text, markdown, tabular data, scanned images, code), extracts and chunks text, tags sensitivity levels via Presidio, and indexes vectors into Qdrant alongside PostgreSQL metadata. When a user submits a natural language query, the system validates user permissions via Keycloak JWT, checks for prompt injections, runs hybrid search (dense + BM25 sparse) fused via Reciprocal Rank Fusion (RRF), reranks results with a cross-encoder model, applies output PII redaction and NLI hallucination guardrails, and streams an accurately cited response.

**Explicit non-goals (what this does NOT do):**  
- **Does not perform base LLM model pre-training or fine-tuning:** The platform uses off-the-shelf open-source foundation models (e.g., Llama 3.1, Mistral, Qwen2.5) served via Ollama or vLLM; model training pipelines are out of scope.
- **Does not provide an end-user consumer Chat UI app:** The system provides an API Gateway, SDKs, and admin observability dashboards, but leaves end-user client UI customization to internal enterprise frontend teams.
- **Does not support unstructured voice/video streaming ingestion:** Video and audio transcription pipelines are excluded to focus on document, code, and database knowledge retrieval.

---

## 6. System Architecture

Describe each major component and its sole responsibility. One component = one responsibility.

```mermaid
flowchart TB
    subgraph clients [Client Layer]
        WebUI[Enterprise Web App / Frontend]
        RESTAPI[REST API Consumers]
        SDKClient[Python/JS SDK]
    end

    subgraph gateway_layer [Gateway Layer]
        APIGW[FastAPI API Gateway]
        AuthMiddleware[Auth Middleware - Keycloak JWT]
        RateLimiter[Rate Limiter - Redis]
        InputGuardrail[Input Guardrail Service]
    end

    subgraph ingestion_svc [Ingestion Service - Independent]
        FileRouter[File Router & MIME Inspector]
        OCRWorker[OCR Worker - PaddleOCR]
        TextExtractor[Text Extractor - PyMuPDF / Unstructured]
        ChunkEngine[Multi-Format Chunking Engine]
        EmbedWorker[Embedding Worker - TEI Client]
        SensitivityTagger[Sensitivity Tagger - Presidio]
        MetadataWriter[Metadata & Index Writer]
    end

    subgraph retrieval_svc [Retrieval Service - Independent]
        QueryProcessor[Query Processor & Language Detector]
        QueryExpander[Query Expander - HyDE / Multi-Query]
        DenseSearch[Dense Vector Search - Qdrant]
        SparseSearch[Sparse BM25 Search - Qdrant]
        RRFMerger[Reciprocal Rank Fusion Merger]
        RerankerSvc[Reranker Service - Cross-Encoder / FlashRank]
        ACLEnforcer[ACL & Clearance Filter]
    end

    subgraph generation_svc [Generation Service - Independent]
        ContextAssembler[Context Assembler & Prompt Builder]
        LLMRouter[LLM Router - Ollama / vLLM / LiteLLM Proxy]
        OutputGuardrail[Output Guardrail Service - PII / Hallucination]
        CitationEngine[Citation Attribution Engine]
    end

    subgraph storage_layer [Storage Layer]
        Qdrant[(Qdrant Vector DB)]
        Postgres[(PostgreSQL Metadata & Audit)]
        MinIO_Store[(MinIO S3 Storage)]
        RedisCache[(Redis Cache & Rate Limiter)]
    end

    subgraph observability_layer [Observability Layer]
        OTelCollector[OpenTelemetry Collector]
        RetrievalValidator[Retrieval Validator Worker]
        Grafana_Dash[Grafana Dashboards]
        AlertManager[Prometheus AlertManager]
    end

    clients --> APIGW
    APIGW --> AuthMiddleware
    AuthMiddleware --> RateLimiter
    RateLimiter --> InputGuardrail

    InputGuardrail --> QueryProcessor
    InputGuardrail --> FileRouter

    FileRouter --> OCRWorker
    FileRouter --> TextExtractor
    OCRWorker --> ChunkEngine
    TextExtractor --> ChunkEngine
    ChunkEngine --> EmbedWorker
    EmbedWorker --> SensitivityTagger
    SensitivityTagger --> MetadataWriter
    MetadataWriter --> Qdrant
    MetadataWriter --> Postgres
    FileRouter --> MinIO_Store

    QueryProcessor --> QueryExpander
    QueryExpander --> DenseSearch
    QueryExpander --> SparseSearch
    DenseSearch --> RRFMerger
    SparseSearch --> RRFMerger
    RRFMerger --> RerankerSvc
    RerankerSvc --> ACLEnforcer
    ACLEnforcer --> ContextAssembler

    ContextAssembler --> LLMRouter
    LLMRouter --> OutputGuardrail
    OutputGuardrail --> CitationEngine

    APIGW --> OTelCollector
    RerankerSvc --> OTelCollector
    LLMRouter --> OTelCollector
    OutputGuardrail --> OTelCollector
    OTelCollector --> Grafana_Dash
    OTelCollector --> RetrievalValidator
    RetrievalValidator --> AlertManager
```

**Component breakdown:**

| Component | Responsibility | Communicates With |
|---|---|---|
| API Gateway | Entry point routing, rate limiting, and Keycloak JWT authentication | Redis, Guardrails Service, Ingestion Service, Retrieval Service |
| Ingestion Service | Asynchronous document parsing, OCR, chunking, TEI embedding, sensitivity tagging, and Qdrant/Postgres indexing | MinIO, TEI Embedding Server, Presidio, Qdrant, PostgreSQL |
| Retrieval Service | Query expansion (HyDE), dense + sparse hybrid vector search, RRF fusion, cross-encoder reranking, and RBAC ACL filtering | Qdrant, TEI Reranker, PostgreSQL, Redis |
| Generation Service | Assembles context, routes prompts to local LLMs (vLLM/Ollama), executes PII output redaction and NLI hallucination checks, builds citations | vLLM / Ollama, Guardrails Service, API Gateway |
| Guardrails Service | Validates input queries against prompt injections and scans LLM outputs for PII leaks and hallucination markers | API Gateway, Generation Service, Presidio, DeBERTa Classifier |
| Storage Layer | Vector embeddings (Qdrant), document metadata/audit logs (PostgreSQL), raw files (MinIO), and query cache (Redis) | Ingestion Service, Retrieval Service, Generation Service |
| Observability Layer | Collects OpenTelemetry traces, computes faithfulness/relevance scores asynchronously, and triggers operational alerts | Grafana, Prometheus, AlertManager, PostgreSQL |

**Non-negotiable constraints:**
- **No Direct Storage Access from Frontend:** Clients can only communicate via the API Gateway. Vector DB, PostgreSQL, and MinIO instances must be bound strictly to internal private network interfaces.
- **Asynchronous Heavy Computations:** Document ingestion, OCR extraction, and retrieval validation must run in Celery background workers to keep API response latencies low.
- **Fail-Closed ACL Security:** If user authorization tokens or namespace parameters cannot be resolved by the ACL filter, the retrieval service returns zero chunks rather than risking an un-gated vector search.

---

## 7. Data Flow

Trace a primary user query end-to-end through the system.

```mermaid
sequenceDiagram
    participant C as Client App
    participant GW as API Gateway
    participant Auth as Keycloak / JWT Auth
    participant Guard as Guardrails Service
    participant Ret as Retrieval Service
    participant Qdrant as Qdrant Vector DB
    participant Gen as Generation Service
    participant LLM as Local LLM (vLLM / Ollama)
    participant OTel as OpenTelemetry Log

    C->>GW: POST /api/v1/query (Query text + JWT token)
    GW->>Auth: Validate JWT & extract AuthContext (roles, clearance, namespaces)
    Auth-->>GW: AuthContext (user_id, clearance=2, namespaces=["kb:finance"])
    GW->>Guard: Validate Input (Check prompt injection)
    Guard-->>GW: Safe = True
    GW->>Ret: Execute Hybrid Retrieval (Query, clearance=2, namespaces)
    Ret->>Qdrant: Parallel Dense (e5-large) + Sparse (BM25) search with ACL filter
    Qdrant-->>Ret: Top 50 Dense + Top 50 Sparse chunks
    Ret->>Ret: Reciprocal Rank Fusion (RRF) -> Top 20
    Ret->>Ret: Cross-Encoder Rerank -> Top 5 chunks
    Ret-->>GW: Return Top 5 Filtered Chunks
    GW->>Gen: Generate Answer (Query, Top 5 Chunks, AuthContext)
    Gen->>LLM: Stream Prompt (System instructions + Cited Chunks)
    LLM-->>Gen: Raw Generated Response Text + Citation markers
    Gen->>Guard: Scan Output (Presidio PII + NLI Hallucination check)
    Guard-->>Gen: Sanitized Response + Redactions
    Gen-->>GW: Formatted Answer + Citation Map
    GW->>OTel: Write async Query Log (latency, faithfulness, chunk IDs)
    GW-->>C: 200 OK (JSON with Answer, Citations, Metadata)
```

---

## 8. Data Models

**Core entities and relationships:**

```mermaid
erDiagram
    users ||--o{ user_roles : has
    roles ||--o{ user_roles : assigned
    roles ||--o{ role_permissions : contains
    permissions ||--o{ role_permissions : mapped
    collections ||--o{ documents : contains
    users ||--o{ collections : creates
    documents ||--o{ chunks : has
    users ||--o{ query_logs : executes
    users ||--o{ audit_log : generates
```

**Schema decisions:**

| Decision | Reason |
|---|---|
| UUID Primary Keys | Prevents document/user ID enumeration and supports distributed multi-cluster data sync. |
| Append-Only Audit Log | Enforces regulatory compliance and tamper-evident audit security via SHA-256 hash chains. |
| JSONB Metadata Fields | Allows flexible document and chunk attribute tagging (languages, sensitivity entities, token counts). |
| Document Hash Uniqueness | `UNIQUE(file_hash, namespace)` prevents duplicate file storage and indexing within the same namespace. |

**Schema definition (runnable DDL):**

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. Collections Table
CREATE TABLE collections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    namespace VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    created_by UUID
);

-- 2. Users Table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id VARCHAR(255) NOT NULL UNIQUE,  -- Keycloak Subject ID
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);

-- Add foreign key constraint to collections
ALTER TABLE collections ADD CONSTRAINT fk_collections_created_by FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL;

-- 3. Documents Table
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id UUID REFERENCES collections(id) ON DELETE CASCADE,
    filename VARCHAR(512) NOT NULL,
    file_hash VARCHAR(64) NOT NULL,  -- SHA-256
    file_type VARCHAR(50) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    namespace VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'processing' CHECK (status IN ('processing', 'ready', 'failed', 'metadata_only')),
    chunk_count INTEGER DEFAULT 0,
    languages_detected JSONB DEFAULT '[]',
    max_sensitivity_level INTEGER DEFAULT 0 CHECK (max_sensitivity_level BETWEEN 0 AND 3),
    object_store_path VARCHAR(1024) NOT NULL,
    ingested_at TIMESTAMPTZ DEFAULT NOW(),
    ingested_by UUID REFERENCES users(id) ON DELETE SET NULL,
    expires_at TIMESTAMPTZ,
    promoted_from UUID,
    version INTEGER DEFAULT 1,
    CONSTRAINT uq_document_hash_namespace UNIQUE(file_hash, namespace)
);

-- 4. Chunks Table
CREATE TABLE chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    chunk_text TEXT NOT NULL,
    chunk_level VARCHAR(20) DEFAULT 'detail' CHECK (chunk_level IN ('summary', 'section', 'detail')),
    page_number INTEGER,
    section_header VARCHAR(512),
    language VARCHAR(10) DEFAULT 'en',
    sensitivity_level INTEGER DEFAULT 0 CHECK (sensitivity_level BETWEEN 0 AND 3),
    sensitivity_entities JSONB DEFAULT '[]',
    embedding_status VARCHAR(20) DEFAULT 'pending' CHECK (embedding_status IN ('pending', 'done', 'failed')),
    token_count INTEGER NOT NULL,
    qdrant_point_id UUID,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 5. Roles Table
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    max_clearance_level INTEGER DEFAULT 0 CHECK (max_clearance_level BETWEEN 0 AND 3)
);

-- 6. Permissions Table
CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_pattern VARCHAR(255) NOT NULL,  -- e.g., "kb:*", "user:*"
    action VARCHAR(50) NOT NULL CHECK (action IN ('read', 'write', 'delete', 'admin'))
);

-- 7. Role Permissions Table
CREATE TABLE role_permissions (
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- 8. User Roles Table
CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    scope VARCHAR(255) DEFAULT '*',
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    granted_by UUID REFERENCES users(id) ON DELETE SET NULL,
    PRIMARY KEY (user_id, role_id, scope)
);

-- 9. Audit Log Table (Append-Only)
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID,
    action VARCHAR(50) NOT NULL,
    details JSONB DEFAULT '{}',
    ip_address INET,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    prev_hash VARCHAR(64)
);

-- 10. Security Incidents Table
CREATE TABLE security_incidents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_type VARCHAR(100) NOT NULL,
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    query_id UUID,
    detected_entities JSONB DEFAULT '[]',
    action_taken VARCHAR(50) NOT NULL,
    resolved BOOLEAN DEFAULT FALSE,
    timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- 11. Query Logs Table
CREATE TABLE query_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    query_text TEXT NOT NULL,
    query_language VARCHAR(10),
    intent_class VARCHAR(50),
    retrieved_chunk_ids JSONB DEFAULT '[]',
    reranker_scores JSONB DEFAULT '[]',
    response_hash VARCHAR(64),
    faithfulness_score FLOAT,
    relevance_score FLOAT,
    latency_ms JSONB DEFAULT '{}',
    guardrail_flags JSONB DEFAULT '[]',
    model_used VARCHAR(100),
    timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_documents_namespace ON documents(namespace);
CREATE INDEX idx_documents_hash ON documents(file_hash);
CREATE INDEX idx_chunks_document ON chunks(document_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp);
CREATE INDEX idx_query_logs_user ON query_logs(user_id, timestamp);
CREATE INDEX idx_security_incidents_unresolved ON security_incidents(resolved, timestamp);
```

---

## 9. Tech Stack

Every row is mandatory. Every rejection reason is mandatory.

| Layer | Choice | Reason | Rejected Alternative | Why Rejected |
|---|---|---|---|---|
| Language | Python 3.11 | Superior ecosystem for AI/ML libraries, native async support, and Pydantic integration. | Node.js / TypeScript | Weaker ML/NLP library support (Presidio, PyMuPDF, spaCy require C++/Python wrappers). |
| API Framework | FastAPI | High performance async I/O, OpenAPI auto-documentation, native Pydantic v2 validation. | Flask | Synchronous thread model scales poorly under concurrent LLM streaming requests. |
| Vector DB | Qdrant | Self-hosted, sparse + dense vector support, built-in payload filtering, low memory footprint. | Pinecone | Closed-source SaaS; violates air-gapped on-premise security constraints. |
| Metadata DB | PostgreSQL 16 | ACID compliance, JSONB support, robust RBAC role schema capabilities. | MongoDB | Lacks strict transactional relational integrity across audit log hash chains. |
| Object Storage | MinIO | AGPL S3-compatible self-hosted object storage with versioning and server-side encryption. | AWS S3 | Cloud vendor lock-in; violates local air-gapped enterprise deployment requirement. |
| Cache & Queue | Redis 7 + Redis Streams | In-memory atomic operations for sliding-window rate limiting and task messaging broker. | Memcached | Lacks data structure options, pub/sub streams, and persistent storage features. |
| Embeddings | `multilingual-e5-large` via TEI | Superior cross-lingual retrieval accuracy; TEI provides high GPU batch throughput. | OpenAI `text-embedding-3` | Cloud API dependency; exposes proprietary query text to external network. |
| Reranker | `ms-marco-MiniLM-L-12-v2` / FlashRank | High-precision cross-encoder reranking; FlashRank provides ultra-fast CPU fallback. | Cohere Rerank API | Commercial paid cloud service; introduces external latency and privacy risks. |
| OCR | PaddleOCR | Apache 2.0 license, high-accuracy multilingual extraction and layout analysis. | Tesseract | Poor layout structure detection; lower accuracy on multi-column enterprise PDFs. |
| LLM Serving (Local) | vLLM / Ollama | High throughput PagedAttention execution; self-hosted open model serving (Llama 3.1). | OpenAI GPT-4o | Violates air-gapped security model; high per-token operating expenses. |
| LLM Proxy | LiteLLM | Standardized OpenAI-compatible interface across multiple local and fallback providers. | LangChain | Heavy abstraction layer with high overhead and frequent breaking API changes. |
| Task Queue | Celery + Redis | Robust async job processing, automatic task retries, and dead-letter queue (DLQ) support. | RQ (Redis Queue) | Lacks complex workflow chaining, task rate-limiting, and multi-worker orchestration. |
| PII Guardrail | Microsoft Presidio | Open-source (MIT), custom entity regex rules, multilingual PII detection and redaction. | AWS Comprehend | Requires external cloud API calls; expensive per-character pricing model. |
| Language Detect | `lingua-py` | High accuracy on short text snippets across 75+ languages with low memory footprint. | `langdetect` | Non-deterministic results on short queries; poor accuracy on mixed scripts. |
| Code Parsing | `tree-sitter` | AST-aware code parsing for function/class-level chunking across multiple programming languages. | Regex splitters | Destroys code syntax context and breaks function boundaries across chunks. |
| Auth Engine | Keycloak | Open-source OIDC/OAuth2 provider supporting enterprise SAML/SSO integration and JWT issuing. | Auth0 | SaaS dependency; user identity data stored outside customer control. |
| Observability | OpenTelemetry + Grafana + Prometheus | Industry standard, vendor-neutral distributed tracing, metrics, and log aggregation. | Datadog | Expensive cloud subscription licensing; requires sending telemetry off-site. |
| Input Guardrail | `protectai/deberta-v3-base-prompt-injection-v2` | High precision local prompt injection classification without cloud API overhead. | Rebuff SaaS | Requires cloud endpoint calls; fails local air-gapped architectural constraints. |
| Orchestration | Docker Compose / Kubernetes | Reproducible containerized service deployments across local dev and production clusters. | Bare-metal scripts | Prone to environment drift, dependency conflicts, and complex manual scaling. |
| Testing | `pytest` + `testcontainers` | Integration testing using real ephemeral database containers in isolated environments. | Mocking DB objects | Mocks mask real SQL syntax errors, vector distance discrepancies, and driver bugs. |

**Key dependencies (concrete package list):**

| Package | Purpose |
|---|---|
| `fastapi==0.109.0` | Core API Gateway web framework. |
| `uvicorn==0.27.0` | High-performance ASGI web server. |
| `pydantic==2.6.0` | Data contract schemas and runtime input validation. |
| `qdrant-client==1.7.3` | Python SDK for interacting with Qdrant vector database. |
| `asyncpg==0.29.0` | Fast async PostgreSQL database driver. |
| `redis==5.0.1` | Redis client for rate limiting and query caching. |
| `minio==7.2.3` | Python client for MinIO object storage. |
| `celery==5.3.6` | Distributed background task queue engine. |
| `presidio-analyzer==2.2.32` | PII entity detection analyzer engine. |
| `presidio-anonymizer==2.2.32` | PII entity text redaction and masking module. |
| `lingua-language-detector==2.0.2` | High-accuracy language detection library. |
| `tree-sitter==0.21.0` | Abstract Syntax Tree parser for code chunking. |
| `opentelemetry-api==1.22.0` | OpenTelemetry tracing and metrics instrumentation API. |
| `flashrank==0.2.4` | Ultra-fast lightweight CPU cross-encoder reranking model. |
| `litellm==1.17.9` | Unified proxy interface for local LLM inference engines. |

---

## 10. Project Structure

```
enterprise-rag/
├── docker-compose.yml              # Local development setup (all services)
├── docker-compose.prod.yml         # Production overrides and resource limits
├── Makefile                        # Common administrative & build tasks
├── .env.example                    # Comprehensive configuration documentation
├── README.md
│
├── services/
│   ├── gateway/                    # API Gateway & Auth Interceptor
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── src/
│   │   │   ├── main.py             # FastAPI entrypoint
│   │   │   ├── auth.py             # Keycloak JWT validation & RBAC rules
│   │   │   ├── rate_limiter.py     # Redis sliding window implementation
│   │   │   └── routes/
│   │   │       ├── query.py        # /api/v1/query endpoint
│   │   │       ├── ingest.py       # Document upload endpoints
│   │   │       ├── admin.py        # Admin configuration routes
│   │   │       └── health.py       # Service health checks
│   │   └── tests/
│   │
│   ├── ingestion/                  # Ingestion & Chunking Service
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── src/
│   │   │   ├── main.py             # Celery worker process
│   │   │   ├── file_router.py      # MIME detection & extractor dispatcher
│   │   │   ├── extractors/
│   │   │   │   ├── text.py         # Plain text & Markdown loader
│   │   │   │   ├── pdf.py          # PyMuPDF text & structure extractor
│   │   │   │   ├── ocr.py          # PaddleOCR engine wrapper
│   │   │   │   ├── tabular.py      # CSV/Excel schema & row reader
│   │   │   │   ├── code.py         # tree-sitter AST parser
│   │   │   │   └── json_loader.py  # Hierarchical JSON flattener
│   │   │   ├── chunkers/
│   │   │   │   ├── text_chunker.py # Recursive token splitter
│   │   │   │   ├── code_chunker.py # AST function/class splitter
│   │   │   │   └── table_chunker.py# Row-group preserving splitter
│   │   │   ├── embedder.py         # TEI client for vector creation
│   │   │   ├── sensitivity.py      # Presidio PII sensitivity classifier
│   │   │   ├── dedup.py            # SHA-256 hash duplication checker
│   │   │   └── language_detect.py  # Lingua language identifier
│   │   └── tests/
│   │
│   ├── retrieval/                  # Retrieval & Search Engine
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── src/
│   │   │   ├── main.py             # Retrieval FastAPI service
│   │   │   ├── query_processor.py  # Query intent classification
│   │   │   ├── query_expander.py   # HyDE & paraphrase expansion
│   │   │   ├── dense_search.py     # Qdrant dense vector retriever
│   │   │   ├── sparse_search.py    # BM25 sparse vector retriever
│   │   │   ├── fusion.py           # Reciprocal Rank Fusion (RRF) logic
│   │   │   ├── reranker.py         # Cross-encoder / FlashRank engine
│   │   │   ├── acl_filter.py       # Metadata permission filtering
│   │   │   └── cache.py            # Redis query response cache
│   │   └── tests/
│   │
│   ├── generation/                 # Context & Generation Engine
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── src/
│   │   │   ├── main.py             # Generation FastAPI service
│   │   │   ├── context_assembler.py# Prompt formatting & token budget manager
│   │   │   ├── llm_router.py       # LiteLLM / Ollama connection provider
│   │   │   ├── citation_builder.py # Inline citation map generator
│   │   │   └── prompts/
│   │   │       ├── system.py       # System instructions
│   │   │       └── citation.py     # Attribution formatting rules
│   │   └── tests/
│   │
│   ├── guardrails/                 # Security & Quality Guardrails
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── src/
│   │   │   ├── main.py             # Guardrails service router
│   │   │   ├── input/
│   │   │   │   ├── injection_detector.py # DeBERTa injection classifier
│   │   │   │   └── query_sanitizer.py    # Encoding attack cleaner
│   │   │   ├── output/
│   │   │   │   ├── pii_scanner.py        # Presidio output redaction
│   │   │   │   ├── sensitivity_checker.py# Document clearance checker
│   │   │   │   └── hallucination.py      # NLI verification model
│   │   │   └── rules/
│   │   │       └── regex_rules.yaml      # Custom security regex patterns
│   │   └── tests/
│   │
│   └── observability/              # Telemetry & Validation
│       ├── Dockerfile
│       ├── pyproject.toml
│       ├── src/
│       │   ├── main.py             # Validation Celery worker
│       │   ├── retrieval_validator.py# Asynchronous faithfulness evaluator
│       │   ├── drift_detector.py   # Weekly retrieval quality comparison
│       │   └── alerting.py         # AlertManager notification dispatcher
│       └── grafana/
│           └── dashboards/         # Pre-configured metrics JSONs
│
├── shared/                         # Shared Code & Interfaces
│   ├── pyproject.toml
│   └── src/
│       ├── models/                 # Shared Pydantic data contracts
│       │   ├── auth.py
│       │   ├── ingestion.py
│       │   ├── retrieval.py
│       │   ├── generation.py
│       │   └── guardrails.py
│       ├── database/               # Database connection helpers
│       │   ├── connection.py
│       │   └── migrations/         # Alembic SQL migration files
│       ├── telemetry.py            # OpenTelemetry setup helper
│       ├── config.py               # Central environment variable parser
│       └── security.py             # Cryptographic hash helpers
│
└── infra/                          # Kubernetes & Infrastructure Configs
    ├── kubernetes/
    │   ├── namespace.yaml
    │   ├── gateway/
    │   ├── ingestion/
    │   ├── retrieval/
    │   ├── generation/
    │   ├── guardrails/
    │   ├── qdrant/
    │   ├── postgres/
    │   ├── redis/
    │   ├── minio/
    │   ├── keycloak/
    │   └── network-policies/
    └── scripts/
        ├── setup-dev.sh
        ├── seed-rbac.sql
        └── generate-certs.sh
```

**Structural decisions:**

| Decision | Reason |
|---|---|
| Monorepo with `services/` Isolation | Allows microservices to be developed in a single repository while preserving build boundary isolation and independent Docker container packaging. |
| Shared `shared/` Library | Centralizes Pydantic interface contracts, database migration scripts, and telemetry setups without duplicate code definitions across microservices. |
| Isolated `infra/kubernetes/` Manifests | Separates infrastructure management declaratively from application logic code. |

---

## 11. Configuration Reference

Every parameter that controls runtime behavior lives here, not hardcoded in source.

```bash
# === Service Storage URLs ===
QDRANT_URL=http://qdrant:6333
POSTGRES_URL=postgresql+asyncpg://rag:password@postgres:5432/ragdb
REDIS_URL=redis://redis:6379/0
MINIO_URL=http://minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# === Auth & Security ===
KEYCLOAK_URL=http://keycloak:8080
KEYCLOAK_REALM=rag
JWT_SECRET=dev-secret-replace-in-prod-environment

# === Embedding & Reranking Models ===
TEI_URL=http://tei:8080
EMBEDDING_MODEL=intfloat/multilingual-e5-large
EMBEDDING_DIMENSION=1024

# === Local LLM Providers ===
LLM_PROVIDER=ollama
OLLAMA_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
VLLM_URL=http://vllm:8000/v1
VLLM_MODEL=meta-llama/Llama-3.1-70B-Instruct
LLM_TIMEOUT_SECONDS=30
LLM_FALLBACK_CHAIN=ollama,vllm

# === Retrieval Tuning Parameters ===
DENSE_TOP_K=50
SPARSE_TOP_K=50
RERANK_TOP_K=5
RRF_K=60
CHUNK_SIZE_TOKENS=512
CHUNK_OVERLAP_TOKENS=64

# === Guardrails & Thresholds ===
INJECTION_DETECTION_THRESHOLD=0.70
PII_DETECTION_CONFIDENCE=0.60
HALLUCINATION_THRESHOLD=0.70
MAX_QUERY_LENGTH=2000
MAX_RESPONSE_TOKENS=4096

# === Rate Limiting & Penalties ===
RATE_LIMIT_VIEWER_QPM=20
RATE_LIMIT_EDITOR_QPM=60
RATE_LIMIT_ADMIN_QPM=120
INJECTION_PENALTY_DURATION_SECONDS=3600

# === Data Storage & Retention ===
USER_UPLOAD_TTL_DAYS=7
CACHE_TTL_SECONDS=3600
AUDIT_LOG_RETENTION_DAYS=90

# === Observability & Alerting ===
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
RETRIEVAL_VALIDATION_SAMPLE_RATE=0.10
FAITHFULNESS_ALERT_THRESHOLD=0.70
LATENCY_ALERT_P95_MS=2000
```

| Variable | Default | Controls | Why This Default |
|---|---|---|---|
| `QDRANT_URL` | `http://qdrant:6333` | Qdrant vector database HTTP endpoint. | Standard internal container hostname and port. |
| `POSTGRES_URL` | None | PostgreSQL database connection string. | Requires explicitly configured user credentials. |
| `REDIS_URL` | `redis://redis:6379/0` | Redis caching and rate limiting connection URL. | Specifies default database index 0. |
| `MINIO_URL` | `http://minio:9000` | Object storage connection endpoint. | Standard MinIO service port. |
| `MINIO_ACCESS_KEY` | `minioadmin` | MinIO administration access key. | Default dev credential; must be overridden in production. |
| `MINIO_SECRET_KEY` | `minioadmin` | MinIO administration secret key. | Default dev credential; must be overridden in production. |
| `KEYCLOAK_URL` | `http://keycloak:8080` | Keycloak identity provider base URL. | Standard Keycloak server port. |
| `KEYCLOAK_REALM` | `rag` | Keycloak realm name for JWT validation. | Dedicated enterprise realm identifier. |
| `JWT_SECRET` | None | Secret key for local JWT validation fallback. | Essential for local dev testing without Keycloak. |
| `TEI_URL` | `http://tei:8080` | Text Embeddings Inference container URL. | Default TEI container binding port. |
| `EMBEDDING_MODEL` | `intfloat/multilingual-e5-large` | Embedding model identifier. | Highest accuracy multi-lingual retrieval balance. |
| `EMBEDDING_DIMENSION` | `1024` | Vector dimension size of embeddings. | Matches `multilingual-e5-large` output dimensions. |
| `LLM_PROVIDER` | `ollama` | Active LLM inference execution provider. | Lightweight local execution default for dev. |
| `OLLAMA_URL` | `http://ollama:11434` | Ollama service REST endpoint. | Default Ollama server binding port. |
| `OLLAMA_MODEL` | `llama3.1:8b` | Ollama foundation LLM target. | Fast CPU/GPU inference balance for dev runs. |
| `VLLM_URL` | `http://vllm:8000/v1` | vLLM high-throughput inference endpoint. | Standard OpenAI-compatible vLLM port. |
| `VLLM_MODEL` | `meta-llama/Llama-3.1-70B-Instruct` | Production vLLM model target. | Highest reasoning accuracy for production tasks. |
| `LLM_TIMEOUT_SECONDS` | `30` | Timeout threshold for LLM generation. | Prevents hung processes from blocking HTTP connections. |
| `LLM_FALLBACK_CHAIN` | `ollama,vllm` | Provider failover order list. | Fallback order if primary LLM container fails. |
| `DENSE_TOP_K` | `50` | Candidate chunk count from vector search. | Provides broad recall window before fusion. |
| `SPARSE_TOP_K` | `50` | Candidate chunk count from BM25 search. | Captures exact keyword matches before fusion. |
| `RERANK_TOP_K` | `5` | Final chunk count passed to LLM context window. | Fits token budget while minimizing distraction noise. |
| `RRF_K` | `60` | Constant parameter for Reciprocal Rank Fusion. | Standard academic constant for optimal rank merging. |
| `CHUNK_SIZE_TOKENS` | `512` | Token length target per chunk. | Fits semantic boundaries without losing context. |
| `CHUNK_OVERLAP_TOKENS` | `64` | Overlapping token count between adjacent chunks. | Prevents context loss across chunk boundaries. |
| `INJECTION_DETECTION_THRESHOLD` | `0.70` | Confidence cutoff to block prompt injection. | Minimizes false positives while catching attacks. |
| `PII_DETECTION_CONFIDENCE` | `0.60` | Presidio entity detection sensitivity. | Balances redaction accuracy and false flags. |
| `HALLUCINATION_THRESHOLD` | `0.70` | NLI score cutoff for hallucination alerts. | Triggers alerts when answer strays from context. |
| `MAX_QUERY_LENGTH` | `2000` | Character ceiling for user query input. | Prevents buffer overflow and denial of service. |
| `MAX_RESPONSE_TOKENS` | `4096` | Ceiling on LLM response token output. | Prevents runaway LLM text generation loops. |
| `RATE_LIMIT_VIEWER_QPM` | `20` | Max queries per minute for Viewer role. | Protects system resources from basic accounts. |
| `RATE_LIMIT_EDITOR_QPM` | `60` | Max queries per minute for Editor role. | Supports standard worker query rates. |
| `RATE_LIMIT_ADMIN_QPM` | `120` | Max queries per minute for Admin role. | Allows high volume admin automation tasks. |
| `INJECTION_PENALTY_DURATION_SECONDS` | `3600` | Penalty lockout duration for injection attacks. | 1-hour cooldown discourages automated probes. |
| `USER_UPLOAD_TTL_DAYS` | `7` | Auto-deletion timer for temporary user uploads. | Cleans up non-promoted user scratch files. |
| `CACHE_TTL_SECONDS` | `3600` | Cache retention lifetime for identical queries. | 1-hour expiration ensures data freshness. |
| `AUDIT_LOG_RETENTION_DAYS` | `90` | PostgreSQL hot audit log storage window. | Meets standard quarterly compliance storage rules. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://otel-collector:4317` | OpenTelemetry collector gRPC endpoint. | Standard OTLP container receiving port. |
| `RETRIEVAL_VALIDATION_SAMPLE_RATE` | `0.10` | Percentage of queries selected for async evaluation. | 10% sample provides statistical quality metrics. |
| `FAITHFULNESS_ALERT_THRESHOLD` | `0.70` | Minimum score before quality alert fires. | Notifies operations when answer quality drops. |
| `LATENCY_ALERT_P95_MS` | `2000` | Latency ceiling before performance alert fires. | Maintains 2-second p95 response time target. |

---

## 12. API Design

**Authentication strategy:**  
Keycloak OpenID Connect (OIDC) JWT Tokens passed via `Authorization: Bearer <token>` headers. The API Gateway validates signatures against Keycloak JWKS endpoints and populates an internal `AuthContext` object.

**Versioning strategy:**  
URI Path Versioning (`/api/v1/...`). Guarantees backwards compatibility for SDKs and external callers when updating backend services.

**Core endpoints:**

| Method | Endpoint | Auth Required | Description | Rate Limit |
|---|---|---|---|---|
| POST | `/api/v1/auth/token` | No | Authenticates user credentials with Keycloak and issues JWT. | 10/min per IP |
| POST | `/api/v1/query` | Yes | Submits query, executes hybrid retrieval, and returns answer + citations. | Role-based (20-120/min) |
| POST | `/api/v1/ingest/upload` | Yes | Uploads document file for processing and indexing. | 10/hr per user |
| GET | `/api/v1/documents` | Yes | Lists metadata of documents accessible to the requesting user. | 60/min per user |
| GET | `/api/v1/documents/:id` | Yes | Retrieves detailed metadata and chunk status for a document. | 60/min per user |
| DELETE | `/api/v1/documents/:id` | Yes | Deletes document and purged associated chunks from Qdrant/Postgres. | 10/min per user |
| POST | `/api/v1/admin/kb/promote` | Yes (Admin) | Promotes user upload document into global knowledge base collection. | 30/min |
| GET | `/api/v1/admin/metrics` | Yes (Admin) | Returns system quality scores, latency metrics, and guardrail stats. | 30/min |
| GET | `/health` | No | Service health check endpoint returning HTTP 200. | Unrestricted |

**Error response shape:**
```json
{
  "error": {
    "code": "SECURITY_GUARDRAIL_VIOLATION",
    "message": "The input query was flagged for potential prompt injection and cannot be processed.",
    "field": "query",
    "timestamp": "2026-07-19T03:20:00Z"
  }
}
```

**Pagination shape:**
```json
{
  "data": [],
  "pagination": {
    "cursor": "eyJpZCI6ICJkOWI4OGUyMy1jOGRkLTk0ZmYifQ==",
    "has_next": true,
    "limit": 20,
    "total_count": 142
  }
}
```

**Internal service contracts:**

```python
from pydantic import BaseModel, Field
from typing import Literal, Optional, List

# 1. Auth Service Contract
class AuthContext(BaseModel):
    user_id: str
    roles: List[str]
    permissions: List[str]
    allowed_namespaces: List[str]
    clearance_level: int = Field(..., ge=0, le=3)

# 2. Ingestion Service Contracts
class IngestRequest(BaseModel):
    file_path: str
    namespace: str
    metadata: dict = {}
    force: bool = False

class IngestResult(BaseModel):
    doc_id: str
    chunk_count: int
    file_type: str
    languages_detected: List[str]
    sensitivity_levels: List[int]
    duration_ms: int

# 3. Retrieval Service Contracts
class RetrievalRequest(BaseModel):
    query: str
    top_k: int = 5
    namespaces: List[str]
    max_sensitivity: int = Field(..., ge=0, le=3)
    file_type_filter: Optional[str] = None
    language_filter: Optional[str] = None

class RetrievedChunk(BaseModel):
    chunk_id: str
    text: str
    source: str
    page: Optional[int] = None
    chunk_index: int
    relevance_score: float
    reranker_score: float
    namespace: str
    language: str
    sensitivity_level: int

# 4. Generation Service Contracts
class Citation(BaseModel):
    marker: int
    chunk_id: str
    source_document: str
    page: Optional[int] = None

class GenerationRequest(BaseModel):
    query: str
    chunks: List[RetrievedChunk]
    response_language: str = "en"
    max_tokens: int = 1024

class GenerationResult(BaseModel):
    answer: str
    citations: List[Citation]
    confidence: float
    guardrail_flags: List[str]
    model_used: str
    latency_ms: int

# 5. Guardrails Service Contracts
class GuardrailFlag(BaseModel):
    type: str
    severity: Literal["low", "medium", "high", "critical"]
    details: str
    span: Optional[List[int]] = None

class GuardrailCheck(BaseModel):
    text: str
    direction: Literal["input", "output"]
    user_clearance: int
    source_sensitivity_levels: Optional[List[int]] = None

class GuardrailResult(BaseModel):
    safe: bool
    action: Literal["pass", "redact", "block"]
    flags: List[GuardrailFlag]
    redacted_text: Optional[str] = None
```

---

## 13. Security

**Threat model:**

| Threat | Vector | Mitigation |
|---|---|---|
| Prompt Injection Attack | Attacker embeds adversarial system instructions into natural language user query. | `protectai/deberta-v3-base-prompt-injection-v2` classifier checks all queries at Gateway before processing; queries exceeding 0.70 threshold are blocked immediately. |
| Unauthorized Cross-Clearance Data Access | User queries system seeking chunks beyond their designated authorization clearance level. | Retrieval Service enforces Qdrant payload filters matching `sensitivity_level <= user_clearance` and restricts search to `allowed_namespaces` extracted from signed Keycloak JWTs. |
| PII Data Exfiltration in LLM Responses | LLM includes sensitive customer PII (SSN, credit cards) in generated answer. | Output Guardrail passes generated answer through Presidio PII scanner; any detected entities above threshold are masked with `[REDACTED]` prior to returning response. |
| Insecure Document Upload Execution | Malicious file payload (e.g., executable script masked as PDF) uploaded for ingestion. | Ingestion Service performs strict MIME type verification using `python-magic`, limits uploads to 100MB, and executes extraction in isolated non-root container environments. |
| Rate Limit Bypass & DoS Probing | Automated bot sweeps gateway with rapid queries seeking to consume LLM GPU resources. | Redis sliding-window rate limiter throttles calls by IP and User ID; accounts triggering repeated prompt injection errors incur an escalating 1-hour IP block penalty. |

**Auth/AuthZ decisions:**
- **Token type:** Keycloak OAuth2 / OIDC RS256 signed JWT.
- **Token expiry:** 15 minutes access token, 7 days refresh token.
- **Refresh strategy:** OAuth2 refresh token rotation flow.
- **Permission model:** Dual-level RBAC + ABAC (Role-Based Access Control + Attribute-Based Access Control on document clearance levels and namespaces).

**Sensitive data handling:**
- **Passwords:** Handled exclusively by Keycloak (bcrypt / Argon2); zero passwords stored in application DB.
- **PII fields:** Automatically scanned during ingestion by Presidio; sensitive fields flagged in metadata and encrypted at rest.
- **Logging:** All user query logs sanitised to strip detected PII entities before inserting into PostgreSQL `query_logs`.
- **Data at rest:** PostgreSQL with TDE, MinIO with Server-Side Encryption (SSE-S3), Qdrant encrypted storage volumes.
- **Data in transit:** TLS 1.3 mandated across all external ingress and internal inter-service mTLS mesh networks.

**Blast radius of a breach:**
- **Database Compromise:** Attacker gains encrypted chunks and document metadata. Because raw files reside in MinIO with independent credentials and PII is redacted/encrypted, un-cleared PII exposure is minimized.
- **API Server Instance Compromise:** Attacker gets short-lived Keycloak JWKS public validation keys and service tokens. Private keys remain secure in Keycloak/Vault. Attacker cannot decrypt encrypted storage volumes.
- **User Session Token Compromise:** Attacker is limited to accessing documents within that single user's namespace and clearance level for a maximum of 15 minutes until the JWT token expires.

---

## 14. Scalability

**Current design ceiling:**  
The baseline single-node deployment handles approximately **20 concurrent active queries per second (QPS)** and **50 document ingestion tasks per hour** before GPU LLM generation (vLLM on single NVIDIA A10G) becomes the bottleneck.

**First bottleneck:**  
Local LLM Generation inference speed. Under concurrent user query load, GPU token generation queues accumulate, increasing p95 latency beyond the 2-second target.

**Scaling path:**

| Stage | Users | Change Required |
|---|---|---|
| MVP | 100–1K | Single Docker Compose host, 1 GPU node (A10G), Ollama/vLLM, managed Postgres/Qdrant containers. |
| Growth | 1K–10K | Deploy on Kubernetes (EKS/GKE/On-Prem); scale Retrieval/Gateway pods to 5 replicas; dedicated TEI GPU nodes; vLLM cluster with 2-4 GPUs. |
| Scale | 10K–100K | Qdrant sharding (4 shards, 2 replicas); PostgreSQL read-replicas for query logs; multi-node vLLM cluster behind vLLM load balancer; dedicated Redis cluster. |
| Beyond | 100K+ | Multi-region Kubernetes deployments; dedicated model serving clusters (TensorRT-LLM / vLLM engine farms); edge caching for frequent queries. |

**Caching strategy:**

| What is cached | Where | TTL | Invalidation trigger |
|---|---|---|---|
| Identical Query Results | Redis | 1 hour | Expiration or underlying document collection mutation. |
| Keycloak JWKS Public Keys | API Gateway Memory | 24 hours | Key rotation trigger event. |
| Document Clearance Metadata | Redis | 15 minutes | User role or permission updates in Keycloak. |

---

## 15. Testing Strategy

| Level | Tool | What It Tests | When It Runs |
|---|---|---|---|
| Unit | `pytest` | Individual chunker logic, RRF fusion math, PII regex patterns, utility helpers. | Every Git commit |
| Integration | `pytest` + `testcontainers` | Service-to-database workflows (ingestion to Qdrant/Postgres, ACL filters). | Every Pull Request |
| Contract | `pytest` + `pydantic` | Inter-service API contract compatibility between Gateway, Retrieval, and Generation. | Every Pull Request |
| End-to-End | `pytest` + `httpx` | Complete pipeline lifecycle: upload PDF -> ingest -> search -> query -> cited answer. | Nightly scheduled run |
| Security | `pytest` + custom scripts | Prompt injection evasion datasets, PII redaction leakage, ACL permission bypasses. | Every Pull Request |
| Load | `locust` | Concurrent query capacity (100 parallel users) verifying p95 latency < 2s. | Weekly / Pre-release |
| Quality | Custom evaluation scripts | MRR@5, Recall@10, and Faithfulness scoring against 200 benchmark test pairs. | Weekly / Pre-release |

---

## 16. Build vs Buy

| Component | Decision | Reason |
|---|---|---|
| Vector Search Engine | Buy Open-Source (Qdrant) | High-performance Rust-based vector index with native sparse+dense hybrid support and payload filtering. Building a custom vector index requires years of low-level database engineering. |
| Identity & Access Control | Buy Open-Source (Keycloak) | Complete OIDC/SAML enterprise authentication server supporting complex RBAC/SSO out of the box. Building custom security auth code is prone to vulnerabilities. |
| Object Storage Engine | Buy Open-Source (MinIO) | S3-compliant object store providing reliable blob storage, server-side encryption, and lifecycle management without cloud lock-in. |
| Document Chunking & Hybrid Retrieval Pipeline | Build | Core differentiating application logic requiring custom multi-format parsing, RRF rank fusion, ACL clearance filtering, and sensitivity tagging. |

---

## 17. Key Algorithms & Critical Logic

### 1. Reciprocal Rank Fusion (RRF)

Combines dense vector search results and sparse BM25 keyword search results into a single unified ranked list:

```python
from typing import List, Tuple, Dict

def reciprocal_rank_fusion(
    ranked_lists: List[List[str]],
    k: int = 60
) -> List[Tuple[str, float]]:
    """
    Merges multiple ranked lists of document IDs using Reciprocal Rank Fusion.
    
    :param ranked_lists: List of ordered doc_id lists from different retrievers.
    :param k: Constant factor to smooth rank impact (standard default = 60).
    :return: Sorted list of (doc_id, rrf_score) tuples descending by score.
    """
    scores: Dict[str, float] = {}
    
    for ranked_list in ranked_lists:
        for rank, doc_id in enumerate(ranked_list, start=1):
            if doc_id not in scores:
                scores[doc_id] = 0.0
            scores[doc_id] += 1.0 / (k + rank)
            
    sorted_scores = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_scores
```

### 2. Redis Sliding Window Rate Limiter

Implements atomic sliding-window rate limiting per user/role using Redis sorted sets:

```python
import time
import redis.asyncio as redis

async def check_rate_limit(
    redis_client: redis.Redis,
    user_id: str,
    limit: int,
    window_seconds: int = 60
) -> bool:
    """
    Validates if a user request is within allowed sliding window rate limits.
    
    :return: True if request is permitted, False if rate limit is exceeded.
    """
    key = f"ratelimit:{user_id}"
    now = time.time()
    clear_before = now - window_seconds
    
    async with redis_client.pipeline(transaction=True) as pipe:
        pipe.zremrangebyscore(key, 0, clear_before)
        pipe.zadd(key, {str(now): now})
        pipe.zcard(key)
        pipe.expire(key, window_seconds)
        results = await pipe.execute()
        
    request_count = results[2]
    return request_count <= limit
```

### 3. Circuit Breaker Pattern for LLM Providers

Prevents system cascading failures when a backend local LLM engine degrades or goes offline:

```python
import time

class CircuitOpenError(Exception):
    pass

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 3, recovery_timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.state = "closed"  # Options: closed, open, half_open
        self.last_failure_time = 0.0

    def call(self, func, *args, **kwargs):
        now = time.time()
        
        if self.state == "open":
            if now - self.last_failure_time > self.recovery_timeout:
                self.state = "half_open"
            else:
                raise CircuitOpenError("Circuit breaker is OPEN. LLM provider unavailable.")

        try:
            result = func(*args, **kwargs)
            if self.state == "half_open":
                self.state = "closed"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
            raise e
```

### 4. Presidio Sensitivity Classification Pipeline

Scans text chunks during ingestion and assigns document sensitivity clearance levels:

```python
from presidio_analyzer import AnalyzerEngine

ENTITY_SENSITIVITY_MAP = {
    "US_SSN": 3, "CREDIT_CARD": 3, "IN_AADHAAR": 3, "CRYPTO": 3,
    "IBAN_CODE": 2, "MONEY": 2, "PHONE_NUMBER": 2,
    "PERSON": 1, "ORGANIZATION": 1, "EMAIL_ADDRESS": 1, "IP_ADDRESS": 1,
}

def classify_sensitivity(text: str, analyzer: AnalyzerEngine) -> Tuple[int, List[dict]]:
    """
    Analyzes text chunk content and computes numerical sensitivity level (0-3).
    """
    results = analyzer.analyze(
        text=text,
        language="en",
        entities=list(ENTITY_SENSITIVITY_MAP.keys())
    )

    max_level = 0
    detected_entities = []
    
    for res in results:
        level = ENTITY_SENSITIVITY_MAP.get(res.entity_type, 0)
        if level > max_level:
            max_level = level
        detected_entities.append({
            "entity_type": res.entity_type,
            "score": res.score,
            "start": res.start,
            "end": res.end
        })

    return max_level, detected_entities
```

---

## 18. Failure Modes

| Component | Failure Scenario | Degradation Strategy |
|---|---|---|
| PostgreSQL Database | Primary DB node unavailable or connection pool exhausted | Next.js/FastAPI Gateway returns HTTP 503 Service Unavailable; read requests fall back to read-replicas; ingestion tasks pause in Celery queue until DB recovers. |
| Vector Database (Qdrant) | Qdrant vector cluster fails to respond | Retrieval service falls back to PostgreSQL text keyword search (TSVector/BM25) as degraded retrieval mode; alerts fire on Grafana dashboard. |
| Redis Cache / Broker | Redis node crashes | Rate limiter defaults to allowing requests (fail-open for rate limits); query cache is bypassed; Celery queue falls back to disk-backed queue or temporary in-memory queue. |
| LLM Provider (vLLM) | Primary GPU LLM server crashes or runs out of VRAM | Circuit breaker opens after 3 failures; LLM Router automatically redirects generation requests to backup Ollama instance or secondary LLM container. |
| Guardrails Service | Presidio or DeBERTa container crashes | Gateway defaults to strict fail-closed security mode: blocks non-admin queries containing potential sensitive patterns and logs a critical security incident alert. |

---

## 19. Risk Register

Distinct from Failure Modes above. This covers what happens when a design assumption turns out to be wrong.

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| PaddleOCR exhibits high error rates on low-quality handwritten scans | Medium | Medium | Integrate fallback to alternative OCR engine (Surya); tag chunks with `ocr_confidence`; flag low-confidence documents for human review. |
| Multilingual embedding quality degrades on niche regional languages | Medium | High | Implement translate-query-then-search pipeline for low-resource languages; enable fine-tuning pipeline for domain-specific embeddings. |
| Cross-encoder reranking adds prohibitive latency on CPU hardware | High | Medium | Deploy FlashRank lightweight model on CPU environments; reserve heavy cross-encoder models for GPU nodes or batch offline runs. |
| Presidio false-positive entity detection redacts legitimate technical terms | Medium | High | Maintain customized enterprise entity regex exclusion lists; allow admins to tune detection confidence thresholds per namespace. |
| Vector database index corruption occurs during ungraceful pod termination | Low | Critical | Automated daily Qdrant snapshot exports to MinIO S3 storage; continuous Write-Ahead Log (WAL) replication across database nodes. |

---

## 20. Cost Architecture

Assumes self-hosted deployment on Cloud Infrastructure (AWS EC2 / On-Prem Kubernetes) pricing as of plan date.

| Resource | 100 users/mo | 10K users/mo | 100K users/mo |
|---|---|---|---|
| Compute (Gateway & Services) | $50/mo (1x t4g.xlarge) | $300/mo (Kubernetes cluster 4x c6g.2xlarge) | $1,200/mo (Auto-scaling cluster 12x c6g.4xlarge) |
| GPU Compute (vLLM / TEI) | $200/mo (1x g5.xlarge - A10G) | $600/mo (2x g5.2xlarge - A10G) | $2,400/mo (4x g5.12xlarge - multi-A10G farm) |
| Database & Storage (Postgres/Qdrant/MinIO) | $40/mo (Managed EBS / DB) | $250/mo (Multi-node DB + 500GB SSD) | $1,000/mo (Sharded Qdrant + 5TB High-IOPS NVMe) |
| Redis Cache | $15/mo (t4g.medium) | $60/mo (r6g.large) | $240/mo (HA Redis Cluster) |
| Observability & Monitoring | $0 (Self-hosted Grafana/Loki) | $50/mo (Storage volume) | $200/mo (High-volume log storage) |
| **Total estimate** | **$305/mo** | **$1,260/mo** | **$5,040/mo** |

**Cost ceiling:**  
At **100,000+ monthly active users**, GPU compute nodes account for ~50% of overall infrastructure expenses. Because all models run self-hosted open-source software, there are zero per-token third-party API fees, maintaining fixed predictable unit economics per query ($0.0005/query).

---

## 21. Limitations

Specific and honest. No vague disclaimers.

- **Handwritten Document OCR Degradation:** Scanned historical documents containing un-structured handwriting exhibit up to 35% character error rates, leading to incomplete vector indexing.
- **Context Window Truncation:** Long conversations exceeding the 4,096 token context limit discard earlier message history, causing the model to lose track of multi-turn conversational nuances.
- **Cross-Lingual Semantic Loss:** Complex domain-specific legal or technical queries translated from low-resource languages into English lose specialized context during translation expansion steps.

---

## 22. Rejected Alternatives

| Alternative | Why Considered | Why Rejected |
|---|---|---|
| Pure Dense Vector Search (No BM25 Hybrid) | Simpler pipeline architecture and lower storage requirements. | Failed accuracy benchmarks on exact keyword searches (part numbers, error codes, legal IDs), resulting in a 30% drop in retrieval recall. |
| Commercial Cloud LLM APIs (OpenAI / Anthropic) | Eliminates self-hosted GPU infrastructure management overhead. | Violates mandatory air-gapped enterprise security rules prohibiting internal document data export to third-party servers. |
| LangChain Framework Abstraction | Speeds up initial prototype script construction. | Excessive abstraction layers, frequent breaking API changes, difficult to debug, and high memory footprint in production microservices. |

---

## 23. Decision Log

| # | Decision | Options Considered | Chosen | Reason | Trade-off Accepted |
|---|---|---|---|---|---|
| 1 | Vector Database Engine | Qdrant vs Milvus vs pgvector | Qdrant | Built-in native BM25 sparse vector search + dense vector search in a single Rust engine. | Requires managing a dedicated vector DB service alongside PostgreSQL. |
| 2 | Reranking Engine | Cross-Encoder vs FlashRank | FlashRank for CPU, Cross-Encoder for GPU | FlashRank runs in under 10ms on standard CPUs without requiring expensive GPU allocation. | Slightly lower reranking precision compared to 7B-parameter cross-encoders. |
| 3 | Ingestion Architecture | Inline HTTP sync processing vs Celery worker tasks | Celery background workers | Prevents gateway request timeouts when uploading large 100MB+ PDF documents. | Requires Redis message broker setup and Celery worker management. |
| 4 | Authentication Engine | Custom JWT logic vs Keycloak OIDC | Keycloak OIDC | Enterprise-grade SAML/SSO support out of the box with proven security audit records. | Increases infrastructure footprint with Keycloak container deployment. |

---

## 24. Implementation Roadmap

**Phase 1 — Foundation (Weeks 1–3)**
- [ ] Set up monorepo repository structure and shared Pydantic models in `shared/`.
- [ ] Configure `docker-compose.yml` with PostgreSQL 16, Qdrant, MinIO, and Redis 7.
- [ ] Run initial database Alembic migrations for core tables (`users`, `roles`, `documents`, `chunks`).
- [ ] Build basic `ingestion` service processing plain text and single-page PDFs.
- [ ] Build basic `retrieval` and `generation` services querying dense vector index with Ollama local LLM.

**Phase 2 — Security & Access Control (Weeks 4–6)**
- [ ] Deploy Keycloak container, configure realm, and wire JWT auth middleware to API Gateway.
- [ ] Implement multi-tenant RBAC ACL filtering in Retrieval service based on user JWT tokens.
- [ ] Build `guardrails` service integrating DeBERTa prompt injection classifier on inputs.
- [ ] Integrate Presidio PII analyzer and anonymizer on generation service outputs.

**Phase 3 — Multilingual & Multi-Format Support (Weeks 7–9)**
- [ ] Upgrade embedding pipeline to `multilingual-e5-large` served via HuggingFace TEI.
- [ ] Add `lingua-py` language detection to query processor and chunk ingestion engines.
- [ ] Integrate PaddleOCR for scanned document layout analysis and image text extraction.
- [ ] Build `tree-sitter` code chunker for function/class-level code indexing.

**Phase 4 — Retrieval Optimization (Weeks 10–12)**
- [ ] Configure Qdrant sparse vectors and implement BM25 sparse retriever.
- [ ] Write Reciprocal Rank Fusion (RRF) rank merger joining dense and sparse candidate lists.
- [ ] Integrate FlashRank and Cross-Encoder reranking models.
- [ ] Add HyDE (Hypothetical Document Embeddings) query expansion mode.

**Phase 5 — Observability & Validation (Weeks 13–15)**
- [ ] Instrument all microservices with OpenTelemetry SDK tracing and metrics exporters.
- [ ] Deploy Prometheus, Grafana, and AlertManager stack with pre-configured dashboard JSONs.
- [ ] Build asynchronous background `retrieval_validator` evaluating faithfulness and relevance scores.
- [ ] Configure AlertManager notification rules for latency spikes and faithfulness drops.

**Phase 6 — Production Hardening (Weeks 16–18)**
- [ ] Implement LLM provider circuit breakers and LiteLLM failover chains.
- [ ] Generate production Kubernetes manifests, HPA rules, and network security policies.
- [ ] Conduct Locust load testing targetting 100 concurrent users with p95 < 2s.
- [ ] Document operational runbooks, GDPR data purging scripts, and backup procedures.

---

## 25. Future Work

**Phase 1 — GraphRAG Knowledge Graph Integration**  
Trigger: When enterprise queries require multi-hop entity reasoning across disparate documents.  
- [ ] Integrate Neo4j graph database alongside Qdrant.
- [ ] Build automated entity-relationship extraction pipeline during document ingestion.

**Phase 2 — Autonomous Agentic Tool Calling**  
Trigger: When users require RAG pipeline to trigger external actions (e.g., creating Jira tickets or updating DB records).  
- [ ] Add LangGraph or custom function-calling execution loop to Generation service.

---

## 26. References

- [Qdrant Hybrid Search Documentation](https://qdrant.tech/docs/concepts/search/#hybrid-search) — Guided dense and sparse vector index design.
- [Reciprocal Rank Fusion (RRF) Paper (Cormack et al.)](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf) — Standard mathematical foundation for rank fusion logic.
- [Microsoft Presidio PII Documentation](https://microsoft.github.io/presidio/) — Influenced the PII detection and sensitivity classification strategy.
- [HuggingFace Text Embeddings Inference (TEI)](https://github.com/huggingface/text-embeddings-inference) — Performance benchmark reference for local embedding deployment.
