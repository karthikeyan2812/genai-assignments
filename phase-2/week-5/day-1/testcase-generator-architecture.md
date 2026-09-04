# AI Testcase Generator Architecture

## 1. Purpose

The AI Testcase Generator ingests product and engineering knowledge, indexes it as searchable embeddings and lexical records, and generates traceable test cases from natural-language or structured requests.

Supported inputs:

- BRDs, user stories, Jira, Azure DevOps, TestRail, Xray, and Zephyr
- Figma files and UI specifications
- Confluence and wiki documentation
- Release notes
- Excel and PDF files
- Grooming-session recordings
- Developer code repositories
- Swagger/OpenAPI specifications
- Defect databases

## 2. Architecture Decisions

| Area | Decision |
|---|---|
| Deployment | Hybrid cloud and on-premises deployment |
| Authentication | Email/password and enterprise SSO |
| Isolation | Organization tenants containing multiple projects |
| Ingestion | Manual upload/connectors plus scheduled synchronization |
| Sources | All listed source categories are first-release targets |
| Test coverage | Functional, integration, API, UI, regression, and negative tests |
| LLMs | Provider adapter for OpenAI, Groq, and Anthropic |
| Embeddings | Configurable provider, starting with Mistral AI embeddings |
| Vector store | MongoDB with deployment-independent repository abstraction |
| Retrieval | Hybrid BM25 and vector search, reranking, deduplication, summarization |
| Review | Approve, reject, or modify with comments |
| Outputs | Markdown, Excel, CSV, JSON, and tool-specific formats |
| Versioning | Source version history plus replacement of outdated indexed records |
| Processing | Event-driven queues and worker services |
| Confidence | Retrieval relevance and generation quality scores |
| Voice | Full voice conversation with interruption handling and history |
| UI | Chat-first experience with optional forms and dashboards |
| Integrations | Ingestion, retrieval, generation, export, review, and webhooks |
| Operations | Logs, metrics, traces, alerts, and audit trails |

## 3. Logical Architecture

```text
[React Web App]
      |
      v
[API Gateway / Express BFF]
      |
      +--> Identity and Tenant Service
      +--> Project and Source Service
      +--> Query and Generation Service
      +--> Review and Export Service
      +--> Voice Service
      |
      v
[Event Bus / Message Queue]
      |
      +--> Ingestion Workers
      |      +--> Connectors and File Parsers
      |      +--> OCR / Speech-to-Text
      |      +--> Chunking and Metadata Extraction
      |      +--> Mistral Embedding Adapter
      |      +--> MongoDB Index Writer
      |
      +--> Generation Workers
      |      +--> Query Preprocessor
      |      +--> Hybrid Retriever
      |      +--> Reranker
      |      +--> Deduplicator
      |      +--> Context Summarizer
      |      +--> LLM Provider Adapter
      |
      +--> Notification and Webhook Workers

[MongoDB]
  +--> Tenant, user, project, job, review, and audit collections
  +--> Source documents, chunks, metadata, versions, and embeddings
  +--> Vector Search and lexical/BM25 indexes

[Observability]
  +--> Central logs, metrics, distributed traces, alerts, audit events
```

## 4. Core Flows

### 4.1 Ingestion Pipeline

1. A user uploads a file, connects a source, or a scheduled job detects a change.
2. The source adapter authenticates and retrieves the source content.
3. The parser extracts text, tables, metadata, links, and source identifiers.
4. PDF and image content can pass through OCR; recordings pass through speech-to-text.
5. Content is normalized, classified, and split into semantically meaningful chunks.
6. Each chunk receives tenant, project, source, version, access-control, and lineage metadata.
7. The embedding adapter generates a Mistral embedding by default.
8. MongoDB stores the chunk, embedding, lexical fields, and version information.
9. Changed records replace stale active records while preserving historical versions.
10. The job emits progress, completion, failure, and webhook events.

### 4.2 Retrieval and Generation Pipeline

1. The API receives a chat, form, or voice query.
2. Query preprocessing normalizes text and expands abbreviations and synonyms.
3. Access control limits retrieval to the requesting tenant and authorized projects.
4. Hybrid search combines BM25 results with MongoDB vector similarity results.
5. A reranker orders the candidate records by semantic and lexical relevance.
6. Duplicate and near-duplicate records are removed.
7. Relevant context is summarized within the configured token budget.
8. The prompt contains the user query, project instructions, retrieved context, and output schema.
9. The LLM adapter calls the selected OpenAI, Groq, or Anthropic provider.
10. The response is validated against the test-case schema and receives retrieval and generation confidence scores.
11. The result is saved as a draft and presented for approval, rejection, or modification.

### 4.3 Human Review Flow

A generated result has one of these states:

`DRAFT -> APPROVED | REJECTED | MODIFIED`

Every decision stores the reviewer, timestamp, comments, previous content hash, and resulting version. Approved test cases can be exported or published through an integration adapter.

## 5. Test Case Schema

Each generated test case should contain:

- Test case ID and title
- Requirement and source references
- Preconditions
- Test data
- Steps
- Expected results
- Test type and priority
- Severity and component
- Positive or negative classification
- Environment requirements
- Automation candidate flag
- Retrieval confidence
- Generation confidence
- Review status and reviewer comments

## 6. API Endpoints

All endpoints are versioned under `/api/v1`, require tenant authorization unless marked public, and return a correlation ID.

### Identity and Tenant Management

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/auth/register` | Register an email/password user |
| `POST` | `/auth/login` | Authenticate a user |
| `GET` | `/auth/sso/{provider}/start` | Start enterprise SSO |
| `GET` | `/auth/sso/{provider}/callback` | Complete enterprise SSO |
| `GET` | `/me` | Get the current user |
| `GET` | `/organizations` | List organizations available to the user |
| `POST` | `/organizations` | Create an organization |
| `GET` | `/organizations/{organizationId}/members` | List members |
| `POST` | `/organizations/{organizationId}/members` | Invite a member |

### Projects and Sources

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/projects` | List authorized projects |
| `POST` | `/projects` | Create a project |
| `GET` | `/projects/{projectId}` | Get project details |
| `PATCH` | `/projects/{projectId}` | Update project settings |
| `DELETE` | `/projects/{projectId}` | Archive a project |
| `GET` | `/projects/{projectId}/sources` | List connected sources |
| `POST` | `/projects/{projectId}/sources` | Connect a source or upload metadata |
| `PATCH` | `/projects/{projectId}/sources/{sourceId}` | Update source configuration |
| `DELETE` | `/projects/{projectId}/sources/{sourceId}` | Disconnect a source |
| `POST` | `/projects/{projectId}/sources/{sourceId}/sync` | Start manual synchronization |
| `GET` | `/projects/{projectId}/sources/{sourceId}/versions` | List source versions |
| `GET` | `/projects/{projectId}/documents` | List ingested documents |
| `GET` | `/projects/{projectId}/documents/{documentId}` | Get document and lineage details |
| `DELETE` | `/projects/{projectId}/documents/{documentId}` | Remove a document from active retrieval |

### Jobs and Ingestion

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/projects/{projectId}/ingestion-jobs` | Start an ingestion job |
| `GET` | `/ingestion-jobs/{jobId}` | Get job status and progress |
| `POST` | `/ingestion-jobs/{jobId}/cancel` | Cancel a queued or running job |
| `GET` | `/ingestion-jobs/{jobId}/errors` | Get item-level ingestion errors |

### Retrieval and Test Generation

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/projects/{projectId}/search` | Run authorized hybrid search |
| `POST` | `/projects/{projectId}/queries` | Save a query and retrieval context |
| `POST` | `/projects/{projectId}/test-cases/generate` | Create a test-case generation job |
| `GET` | `/generation-jobs/{jobId}` | Get generation status and output reference |
| `GET` | `/projects/{projectId}/test-cases` | List generated test cases |
| `GET` | `/projects/{projectId}/test-cases/{testCaseId}` | Get a test case and citations |
| `PATCH` | `/projects/{projectId}/test-cases/{testCaseId}` | Modify a draft or reviewed test case |
| `POST` | `/projects/{projectId}/test-cases/{testCaseId}/regenerate` | Regenerate using feedback or new context |
| `GET` | `/projects/{projectId}/test-cases/{testCaseId}/versions` | List test-case versions |

### Review, Export, and Integrations

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/test-cases/{testCaseId}/review` | Approve, reject, or modify with comments |
| `GET` | `/projects/{projectId}/reviews` | List pending and completed reviews |
| `POST` | `/projects/{projectId}/exports` | Export test cases to a selected format |
| `GET` | `/exports/{exportId}` | Get export status and download metadata |
| `POST` | `/projects/{projectId}/integrations` | Configure Jira, ADO, TestRail, Xray, Zephyr, or defect systems |
| `GET` | `/projects/{projectId}/integrations` | List integrations |
| `POST` | `/integrations/{integrationId}/publish` | Publish approved test cases |
| `POST` | `/webhooks/{provider}` | Receive provider change events |
| `POST` | `/projects/{projectId}/webhook-subscriptions` | Register a project webhook |
| `DELETE` | `/webhook-subscriptions/{subscriptionId}` | Remove a webhook |

### Voice

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/projects/{projectId}/voice/sessions` | Start a voice conversation |
| `POST` | `/voice/sessions/{sessionId}/audio` | Stream or submit audio input |
| `POST` | `/voice/sessions/{sessionId}/interrupt` | Interrupt the current response |
| `POST` | `/voice/sessions/{sessionId}/end` | End a voice conversation |
| `GET` | `/voice/sessions/{sessionId}/transcript` | Get the conversation transcript |

### Operations

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health/live` | Liveness check |
| `GET` | `/health/ready` | Readiness and dependency check |
| `GET` | `/metrics` | Metrics endpoint for monitoring |
| `GET` | `/projects/{projectId}/audit-events` | Query project audit events |

## 7. Proposed Project Structure

```text
testcase-generator/
├── apps/
│   ├── api/                         # Express API and BFF
│   ├── web/                         # React application
│   ├── ingestion-worker/            # Queue consumers and source ingestion
│   ├── generation-worker/           # Retrieval and LLM generation jobs
│   └── notification-worker/         # Webhooks and notifications
├── packages/
│   ├── config/                      # Environment and runtime configuration
│   ├── contracts/                   # OpenAPI, DTOs, and event schemas
│   ├── auth/                        # JWT, SSO, RBAC, and permissions
│   ├── database/                    # MongoDB clients, schemas, repositories
│   ├── connectors/                  # Jira, ADO, Confluence, Git, Swagger, files
│   ├── parsers/                     # PDF, Excel, Figma, code, and transcript parsers
│   ├── ingestion/                   # Chunking, metadata, versioning, embeddings
│   ├── retrieval/                   # Query preprocessing, hybrid search, reranking
│   ├── generation/                  # Prompting, provider adapters, validation
│   ├── review/                      # Review state machine and audit events
│   ├── export/                      # Markdown, Excel, CSV, JSON, and tool exporters
│   ├── voice/                       # Speech recognition, synthesis, session handling
│   └── observability/               # Logging, metrics, traces, and correlation IDs
├── infrastructure/
│   ├── docker/                      # Local service containers
│   ├── kubernetes/                  # Hybrid deployment manifests
│   ├── terraform/                   # Cloud infrastructure modules
│   └── monitoring/                  # Dashboards, alerts, and telemetry config
├── docs/
│   ├── architecture.md
│   ├── api/openapi.yaml
│   ├── security.md
│   └── runbooks/
├── tests/
│   ├── contract/
│   ├── integration/
│   ├── e2e/
│   └── evaluation/                 # Retrieval and generation quality datasets
├── .env.example
├── docker-compose.yml
├── package.json
├── pnpm-workspace.yaml
└── README.md
```

## 8. Security and Governance

- Enforce organization and project-level authorization on every retrieval and generation request.
- Encrypt data in transit and at rest; store provider credentials in a secret manager.
- Apply least-privilege service accounts for connectors and workers.
- Redact secrets and sensitive personal data before sending context to an external LLM.
- Record immutable audit events for source access, generation, review, export, and publication.
- Support configurable data residency and an on-premises LLM route for restricted deployments.
- Validate uploaded files, limit size and type, scan for malware, and sandbox parsers.
- Use rate limits, idempotency keys, retry policies, dead-letter queues, and webhook signatures.

## 9. Quality and Observability

Track retrieval recall, citation coverage, groundedness, duplicate rate, generation latency, token usage, confidence calibration, review acceptance rate, ingestion throughput, and connector error rate.

Every request and asynchronous job should carry a correlation ID and tenant-safe trace context. Alerts should cover dependency failures, queue backlog, repeated connector errors, low retrieval quality, high generation failure rate, and security events.

## 10. Recommended Delivery Phases

1. Foundation: tenant model, authentication, project management, file ingestion, MongoDB repository, chat UI, and Markdown output.
2. Retrieval quality: Mistral embeddings, hybrid search, reranking, citations, deduplication, and confidence scoring.
3. Generation workflow: full test-case schema, review lifecycle, versioning, Excel/CSV/JSON export, and audit trails.
4. Connectors: Jira, ADO, Confluence, Git, Swagger, TestRail, Xray, Zephyr, Figma, and defect databases.
5. Operations and scale: queue workers, scheduled sync, webhooks, traces, alerts, hybrid deployment, and evaluation suite.
6. Voice: speech-to-text, text-to-speech, interruption handling, and conversation history.

## 11. Key Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Hallucinated test cases | Require citations, structured validation, confidence scores, and human review |
| Stale requirements | Version sources, synchronize on schedule, and expose source timestamps |
| Cross-tenant data leakage | Enforce tenant filters in repositories and retrieval metadata |
| Poor retrieval quality | Hybrid search, reranking, evaluation datasets, and feedback capture |
| Connector instability | Isolated adapters, retries, rate limits, and dead-letter queues |
| Sensitive data sent to LLMs | Redaction, provider policy controls, and optional on-premises routing |
| Cost and latency growth | Async jobs, model routing, caching, token budgets, and usage metrics |
