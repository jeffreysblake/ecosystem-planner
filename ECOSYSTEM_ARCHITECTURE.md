# Autonomous Development Ecosystem - Architecture Plan (2025)

> **Last Updated:** November 14, 2025
> **Vision:** End-to-end business workflow and development process automation using AI agents coordinated through web interface

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Existing Repositories](#existing-repositories)
3. [Recommended New Repositories](#recommended-new-repositories)
4. [System Architecture](#system-architecture)
5. [Technology Stack](#technology-stack)
6. [Integration Patterns](#integration-patterns)
7. [Implementation Roadmap](#implementation-roadmap)
8. [2025 Best Practices & Standards](#2025-best-practices--standards)

---

## Executive Summary

This ecosystem enables automated workflows from high-level business tasks → research → specifications → code → testing → deployment, with:
- **Small team optimization** (2-10 people)
- **Priority focus:** Spec → Code → Test pipeline
- **Deployment:** Sandboxed containers
- **Artifact storage:** Git repositories
- **AI strategy:** Claude for high-level tasks, local models for low-level execution

### Core Workflow
```
Business Task → Research (web search) → Reports →
Spec Generation → Code Implementation → Testing → Deployment
```

---

## Existing Repositories

### 1. **Collector-MCP**
**Type:** MCP Server
**Purpose:** Grant data collection for business opportunities
**Status:** Existing
**Tech Stack:** TypeScript, MCP Protocol

**Key Features:**
- Pull grants of interest for business
- Data aggregation from multiple sources
- MCP server interface for agent consumption

### 2. **Vibes-Director**
**Type:** Planning/Workflow Management
**Purpose:** Orchestrate agent flows for project backlogs
**Status:** Existing
**Tech Stack:** TypeScript

**Key Features:**
- Workflow coordination
- Agent orchestration
- Project backlog management
- Job queue system

**2025 Enhancement Opportunities:**
- Integrate LangGraph for complex multi-agent workflows
- Add visual workflow builder for non-technical users
- Implement observability hooks for monitoring

### 3. **DataMan-MCP**
**Type:** MCP Server + Frontend
**Purpose:** Initial data flow and storage for RAG ecosystem
**Status:** Existing
**Tech Stack:** Python (server), React + TypeScript (frontend)

**Key Features:**
- Data ingestion and management
- Storage coordination
- RAG data preparation
- Web interface for data management

**2025 Enhancement Opportunities:**
- Add OpenTelemetry instrumentation
- Implement document versioning
- Add metadata tagging for better retrieval

---

## Recommended New Repositories

### TIER 1: Critical for Spec→Code→Test Pipeline

#### 1. **Spec-Synthesizer-MCP**
**Priority:** HIGHEST - Start here
**Type:** MCP Server
**Purpose:** Transform research and notes into structured specifications

**Key Features:**
- **Multi-format spec generation:**
  - User stories with acceptance criteria
  - Technical design documents
  - API contracts (OpenAPI 3.1, GraphQL schemas)
  - Database schemas (Prisma, SQL DDL)
  - Test plans with coverage requirements

- **Input sources:**
  - Documents from DataMan-MCP (via MCP protocol)
  - Research reports from Research-Harvester-MCP
  - Project notes from Git repositories
  - Existing codebase analysis

- **Spec validation:**
  - Completeness checking (all required sections present)
  - Internal consistency validation
  - Dependencies and constraints analysis
  - Feasibility scoring

- **Version control:**
  - Git integration for spec storage
  - Diff visualization for spec changes
  - Branch-based spec workflows
  - Spec review process support

**Tech Stack:**
- **Language:** Python 3.12+
- **Validation:** Pydantic v2 for schema validation
- **Templates:** Jinja2 for spec templates
- **Git:** GitPython for repository operations
- **MCP:** Official MCP Python SDK
- **AI:** Claude Sonnet 4.5 for spec generation, local models for formatting

**API Endpoints (MCP Tools):**
```typescript
// Core spec generation
generate_user_stories(research_doc_id: string, project_context: string)
generate_technical_design(user_stories: string[], architecture_preferences: object)
generate_api_contract(design_doc: string, api_style: 'rest' | 'graphql')
generate_database_schema(design_doc: string, db_type: 'postgres' | 'mysql' | 'mongodb')
generate_test_plan(spec: string, coverage_target: number)

// Validation and review
validate_spec(spec_id: string): ValidationReport
check_spec_completeness(spec_id: string): CompletenessScore
review_spec(spec_id: string, reviewer_context: string): ReviewComments

// Version control
save_spec_to_git(spec: object, repo_url: string, branch: string)
get_spec_diff(spec_id_1: string, spec_id_2: string)
```

**Integration Points:**
- **Input:** DataMan-MCP (research documents), Research-Harvester-MCP (web content)
- **Output:** Codesmith-MCP (specs for code generation), Git repositories
- **Orchestration:** Vibes-Director (workflow coordination)

---

#### 2. **Codesmith-MCP**
**Priority:** HIGHEST - Required for core pipeline
**Type:** MCP Server
**Purpose:** Transform specifications into production-ready code

**Key Features:**
- **Spec-to-code generation:**
  - Architecture scaffolding (project structure, config files)
  - Boilerplate code generation (CRUD, API routes, models)
  - Business logic implementation
  - Test file generation
  - Documentation generation (JSDoc, docstrings)

- **Git workflow automation:**
  - Branch creation (feature/spec-{id})
  - Incremental commits with meaningful messages
  - Pull request creation with description
  - Code review coordination

- **Code quality:**
  - Linting integration (ESLint, Pylint, Ruff)
  - Formatting (Prettier, Black)
  - Type checking (TypeScript, mypy)
  - Security scanning (basic patterns)

- **Template system:**
  - REST API templates (Express, FastAPI, Go Fiber)
  - React component templates
  - Database layer templates (Prisma, SQLAlchemy)
  - Microservice templates

- **Incremental updates:**
  - Diff-based code modifications
  - Merge conflict resolution assistance
  - Refactoring suggestions

**Tech Stack:**
- **Language:** TypeScript (for Git/Node integration) + Python (for code analysis)
- **Git:** simple-git (Node.js), GitPython (Python)
- **MCP:** Official MCP TypeScript SDK
- **Code Analysis:** ts-morph (TypeScript), ast (Python)
- **AI:** Claude Sonnet 4.5 for architecture, local models (Qwen-Coder, DeepSeek-Coder) for boilerplate
- **GitHub/GitLab API:** Octokit, python-gitlab

**2025 Best Practices:**
- Use Claude Agent SDK for "spec-to-repo" automation
- Leverage Claude Sonnet 4.5's improved edit capabilities (0% error rate on code editing)
- Implement agentic workflows: spec interpretation → planning → implementation → validation

**API Endpoints (MCP Tools):**
```typescript
// Code generation
generate_project_scaffold(spec: Spec, tech_stack: TechStack): ProjectStructure
generate_api_routes(api_spec: OpenAPISpec): CodeFiles
generate_database_models(schema: DatabaseSchema, orm: 'prisma' | 'sqlalchemy'): CodeFiles
generate_tests(code_files: CodeFiles[], test_framework: string): TestFiles
implement_feature(spec: FeatureSpec, target_repo: string): ImplementationResult

// Git operations
create_feature_branch(repo: string, spec_id: string): BranchInfo
commit_changes(repo: string, files: string[], message: string): CommitHash
create_pull_request(repo: string, branch: string, spec: Spec): PRUrl

// Code quality
run_linter(repo: string, files: string[]): LintResults
format_code(repo: string, files: string[]): void
run_type_check(repo: string): TypeCheckResults

// Review and iteration
request_code_review(pr_url: string, reviewers: string[]): ReviewRequest
apply_review_feedback(pr_url: string, feedback: ReviewComments): UpdatedCode
```

**Integration Points:**
- **Input:** Spec-Synthesizer-MCP (specifications)
- **Output:** Git repositories, Sandbox-Runner-MCP (for testing)
- **Orchestration:** Vibes-Director (multi-step workflows)
- **Monitoring:** Observatory (code generation metrics, AI costs)

---

#### 3. **Sandbox-Runner-MCP**
**Priority:** HIGHEST - Required for safe testing
**Type:** MCP Server
**Purpose:** Containerized execution environment for code and tests

**Key Features:**
- **Container orchestration:**
  - Docker SDK integration
  - Pre-built images for common stacks:
    - Node.js (18, 20, 22)
    - Python (3.10, 3.11, 3.12)
    - Go (1.21, 1.22)
    - Rust (stable, nightly)
    - Multi-language (polyglot projects)

- **Security & isolation:**
  - Resource limits (CPU: 2 cores max, Memory: 4GB max, Timeout: 30min)
  - Network isolation (optional internet access via allowlist)
  - Filesystem isolation (read-only base, writable /tmp)
  - User namespaces (non-root execution)
  - Seccomp profiles for syscall filtering

- **Test execution:**
  - Test framework detection (Jest, Pytest, Go test, Cargo test)
  - Parallel test execution
  - Test result parsing (JUnit XML, TAP)
  - Code coverage reporting
  - Performance benchmarks

- **Logging & artifacts:**
  - Real-time log streaming (WebSocket)
  - stdout/stderr capture
  - Test artifacts collection (screenshots, reports)
  - Build artifacts preservation

- **Multi-container support:**
  - Docker Compose integration
  - Service dependencies (app + DB + Redis)
  - Health checks and readiness probes

**Tech Stack:**
- **Language:** Python 3.12 (async/await for container streaming) or Go 1.22 (better performance)
- **Container Runtime:** Docker Engine API (docker-py for Python, docker SDK for Go)
- **Orchestration:** Docker Compose v2
- **MCP:** Official MCP Python/Go SDK
- **Streaming:** WebSockets for real-time logs
- **Resource Management:** cgroups v2

**2025 Best Practices:**
- Implement sandboxed containers using lightweight VMs (gVisor, Kata Containers) for untrusted code
- Use existing open-source: [code-sandbox-mcp](https://github.com/Automata-Labs-team/code-sandbox-mcp)
- Container image caching for faster startup
- OpenTelemetry instrumentation for observability

**API Endpoints (MCP Tools):**
```typescript
// Container lifecycle
create_sandbox(image: string, resources: ResourceLimits, network: NetworkConfig): SandboxId
start_sandbox(sandbox_id: string): void
stop_sandbox(sandbox_id: string): void
destroy_sandbox(sandbox_id: string): void

// Code execution
execute_code(sandbox_id: string, code: string, language: string, timeout: number): ExecutionResult
run_tests(sandbox_id: string, test_framework: string, test_path: string): TestResults
run_build(sandbox_id: string, build_command: string): BuildResult

// File operations
inject_files(sandbox_id: string, files: FileMap): void
extract_artifacts(sandbox_id: string, paths: string[]): FileMap

// Logging and monitoring
stream_logs(sandbox_id: string): WebSocketStream
get_resource_usage(sandbox_id: string): ResourceStats

// Multi-container
create_environment(compose_file: string, resources: ResourceLimits): EnvironmentId
run_integration_tests(env_id: string, test_command: string): TestResults
```

**Integration Points:**
- **Input:** Codesmith-MCP (generated code), DevOps-Conductor-MCP (CI pipelines)
- **Output:** Test results to Vibes-Director, artifacts to DataMan-MCP
- **Monitoring:** Observatory (container metrics, execution costs)

---

### TIER 2: Supporting Infrastructure

#### 4. **RAG-Engine-MCP**
**Type:** MCP Server
**Purpose:** Vector database, embeddings, and RAG infrastructure (previously unplanned)

**Key Features:**
- **Vector database:**
  - Multiple backend support: Pinecone (cloud), Weaviate (self-hosted), Chroma (lightweight)
  - Hybrid search: semantic (vector) + keyword (BM25)
  - Multi-tenant namespacing for team isolation

- **Embeddings:**
  - Model support: OpenAI text-embedding-3, Cohere Embed v3, local (BGE-M3)
  - Batch embedding processing
  - Embedding cache for cost reduction
  - Fine-tuning support for domain-specific embeddings

- **RAG capabilities:**
  - **Retrieval strategies:**
    - Top-K similarity search
    - MMR (Maximal Marginal Relevance) for diversity
    - Hybrid search with score fusion
    - Contextual compression (re-ranking)
  - **Agentic RAG (2025 trend):**
    - Iterative retrieval with reasoning
    - Self-correction loops
    - Multi-hop reasoning across documents
    - Tool use for external knowledge

- **Document processing:**
  - Chunking strategies (fixed-size, semantic, recursive)
  - Metadata extraction and indexing
  - Document versioning and updates
  - Source attribution and citation

**Tech Stack:**
- **Language:** Python 3.12
- **Vector DB:** Weaviate (self-hosted) or Chroma (lightweight)
- **Embeddings:** sentence-transformers, OpenAI API
- **Framework:** LangChain or LlamaIndex for RAG patterns
- **MCP:** Official MCP Python SDK
- **Search:** Elasticsearch integration for hybrid search

**2025 Best Practices:**
- Implement Agentic RAG with iterative reasoning
- Use hybrid search (semantic + keyword) for better retrieval
- Add multimodal support (text, images, audio)
- Integrate knowledge graphs for structured data
- Support on-device embeddings for privacy

**API Endpoints (MCP Tools):**
```typescript
// Document management
ingest_documents(docs: Document[], namespace: string): IngestResult
update_document(doc_id: string, content: string): void
delete_documents(doc_ids: string[]): void

// Embeddings
generate_embeddings(texts: string[], model: string): Embedding[]
batch_embed(texts: string[], batch_size: number): Embedding[]

// Retrieval
search_similar(query: string, top_k: number, filters: object): SearchResults
hybrid_search(query: string, alpha: number, top_k: number): SearchResults
agentic_retrieval(query: string, reasoning_steps: number): AgenticResult

// RAG workflows
retrieve_and_generate(query: string, context_window: number): RAGResponse
multi_hop_reasoning(query: string, max_hops: number): ReasoningResult
```

**Integration Points:**
- **Input:** DataMan-MCP (documents), Research-Harvester-MCP (web content)
- **Output:** Spec-Synthesizer-MCP (context for specs), Codesmith-MCP (code context)

---

#### 5. **Research-Harvester-MCP**
**Type:** MCP Server
**Purpose:** Automated web research and content aggregation

**Key Features:**
- **Search integration:**
  - SearXNG (self-hosted, privacy-focused, meta-search)
  - Perplexity API (if budget allows, limited beta)
  - Google Scholar (academic research)
  - Arxiv, GitHub, Stack Overflow (specialized sources)

- **Content extraction:**
  - HTML parsing and cleaning (BeautifulSoup, Readability)
  - JavaScript rendering (Playwright for dynamic sites)
  - PDF text extraction
  - Code snippet extraction

- **Content analysis:**
  - Source credibility scoring (domain authority, recency)
  - Duplicate detection (SimHash, MinHash)
  - Summary generation (extractive and abstractive)
  - Citation management (APA, MLA, Chicago)

- **Research workflows:**
  - Task queue with scheduling (Celery or BullMQ)
  - Multi-source aggregation
  - Automatic storage in DataMan-MCP
  - Research report generation

**Tech Stack:**
- **Language:** Python 3.12
- **Scraping:** Playwright (JS rendering), BeautifulSoup4 (parsing)
- **Search:** SearXNG API, Perplexity API (if available)
- **Queue:** Celery + Redis for task management
- **NLP:** spaCy for text processing, sentence-transformers for similarity
- **MCP:** Official MCP Python SDK

**2025 Best Practices:**
- Use Perplexity API for AI-enhanced research (but note: returns natural language, not structured data)
- Implement rate limiting and politeness delays
- Add browser fingerprinting resistance
- Support RSS feeds for monitoring sources
- Integrate with RAG-Engine-MCP for semantic search of research

**API Endpoints (MCP Tools):**
```typescript
// Search operations
web_search(query: string, sources: string[], max_results: number): SearchResults
academic_search(query: string, start_year: number): AcademicPapers
code_search(query: string, languages: string[]): CodeResults

// Content extraction
extract_article(url: string): Article
batch_extract(urls: string[]): Article[]
monitor_source(url: string, schedule: string): MonitorId

// Research workflows
create_research_task(topic: string, sources: string[], depth: 'quick' | 'thorough'): TaskId
get_task_status(task_id: string): TaskStatus
generate_research_report(task_id: string, format: 'markdown' | 'pdf'): Report
```

**Integration Points:**
- **Output:** DataMan-MCP (store research), RAG-Engine-MCP (embed content)
- **Orchestration:** Vibes-Director (research → spec workflows)

---

#### 6. **DevOps-Conductor-MCP**
**Type:** MCP Server
**Purpose:** CI/CD orchestration and deployment management

**Key Features:**
- **Pipeline orchestration:**
  - Git webhook listeners (push, PR, tag)
  - Pipeline definitions (YAML or code)
  - Build → Test → Deploy workflows
  - Integration with Sandbox-Runner-MCP for CI

- **Environment management:**
  - Multi-environment support (dev, staging, prod)
  - Environment variables and config management
  - Secrets management (encrypted, rotated)
  - Database migrations

- **Deployment strategies:**
  - Rolling updates (zero-downtime)
  - Blue-green deployments
  - Canary releases (gradual rollout)
  - Rollback capabilities (automatic on failure)

- **Notifications:**
  - Slack/Discord webhooks
  - Email alerts
  - GitHub/GitLab status updates

**Tech Stack:**
- **Language:** Python 3.12 or Go 1.22
- **Webhooks:** FastAPI or Fiber for webhook server
- **Container Orchestration:** Docker Compose, Kubernetes (optional)
- **Secrets:** SOPS, age encryption
- **MCP:** Official MCP SDK

**API Endpoints (MCP Tools):**
```typescript
// Pipeline management
create_pipeline(repo: string, config: PipelineConfig): PipelineId
trigger_pipeline(pipeline_id: string, branch: string): RunId
get_pipeline_status(run_id: string): PipelineStatus

// Deployment
deploy(service: string, version: string, environment: 'dev' | 'staging' | 'prod', strategy: DeployStrategy): DeploymentId
rollback(deployment_id: string): void
get_deployment_status(deployment_id: string): DeploymentStatus

// Environment management
set_env_var(environment: string, key: string, value: string, encrypted: boolean): void
run_migration(environment: string, migration_files: string[]): MigrationResult
```

**Integration Points:**
- **Input:** Git webhooks, Codesmith-MCP (code commits)
- **Uses:** Sandbox-Runner-MCP (CI testing)
- **Monitoring:** Observatory (deployment metrics, failure rates)

---

#### 7. **Gateway-Hub**
**Type:** API Gateway
**Purpose:** Unified entry point, authentication, and service coordination

**Key Features:**
- **Authentication & Authorization (2025 Best Practices):**
  - JWT tokens (access + refresh)
  - OAuth 2.0/2.1 support (MCP spec now supports OAuth 2.1)
  - API key support (for service-to-service)
  - Role-based access control (RBAC)
    - Roles: Admin, Developer, Viewer, AI Agent
    - Permissions per MCP tool

- **Request routing:**
  - MCP server discovery and registration
  - Intelligent routing to appropriate services
  - Load balancing (round-robin, least-connections)
  - Health checks (ping MCP servers)

- **Rate limiting & quotas:**
  - Per-user rate limits (100 req/min, 1000 req/hour)
  - Team-level quotas
  - Cost controls (max AI spend per user/day)
  - Graceful degradation (queue or reject)

- **Observability:**
  - Request/response logging
  - Latency tracking
  - Error aggregation
  - WebSocket support for real-time updates

**Tech Stack:**
- **Language:** Go (high performance) or Node.js with Express/Fastify
- **Auth:** JWT libraries (golang-jwt, jsonwebtoken)
- **Reverse Proxy:** Built-in or use Traefik/Nginx
- **Rate Limiting:** Redis-based (go-redis, ioredis)
- **MCP:** HTTP transport with SSE (new in 2025)

**2025 Best Practices:**
- Centralized authentication at gateway, decentralized authorization in services
- Use OAuth 2.1 for modern auth flows
- Implement RBAC with purpose-built authorization service (not just JWT claims)
- Support MCP HTTP transport with Server-Sent Events (SSE)

**API Endpoints:**
```typescript
// Authentication
POST /auth/login (email, password) → { access_token, refresh_token }
POST /auth/refresh (refresh_token) → { access_token }
POST /auth/logout (token) → void
POST /auth/create-api-key (name, permissions) → { api_key }

// Service routing (proxies to MCP servers)
POST /mcp/:server_name/:tool_name (args) → tool_result

// Admin
GET /admin/services → [RegisteredService]
GET /admin/usage → UsageStats
POST /admin/set-quota (user_id, quota) → void
```

**Integration Points:**
- **Frontend:** Control-Hub-Web (web interface)
- **Backend:** All MCP servers (Vibes-Director, DataMan, Collector, etc.)
- **Monitoring:** Observatory (auth events, API usage)

---

### TIER 3: Operations & Visibility

#### 8. **Observatory**
**Type:** Monitoring & Analytics Platform
**Purpose:** Centralized observability for the entire ecosystem

**Key Features:**
- **Logging:**
  - Centralized log aggregation (from all services)
  - Structured logging (JSON format)
  - Log parsing and indexing
  - Full-text search
  - Retention policies (30 days default)

- **Metrics:**
  - API latency (p50, p95, p99)
  - Throughput (requests per second)
  - Error rates (by service, by endpoint)
  - AI model costs (by service, by user, by model)
  - Container resource usage (CPU, memory, disk)
  - Workflow success/failure rates

- **Alerting:**
  - Threshold-based alerts (latency > 1s, error rate > 5%)
  - Anomaly detection (ML-based)
  - Alert channels: Slack, email, webhooks
  - Alert routing by severity
  - On-call schedules

- **Dashboards:**
  - Service health overview
  - Cost tracking (AI spend per day/week/month)
  - Performance trends
  - User activity
  - Custom dashboards

**Tech Stack (2025 Recommendations):**
- **Option 1 - OpenObserve:**
  - 140x lower storage costs than Elasticsearch
  - Unified logs, metrics, traces
  - SQL and PromQL querying
  - Self-hosted, open-source

- **Option 2 - SigNoz:**
  - OpenTelemetry-native
  - Combines metrics, traces, logs
  - Self-hosted or cloud
  - Free community edition

- **Option 3 - Grafana Stack (LGTM):**
  - Loki (logs), Prometheus (metrics), Tempo (traces), Grafana (visualization)
  - Most flexible, requires more setup
  - Fully self-hosted

**Recommended:** OpenObserve for small teams (easier setup, lower cost)

**API Endpoints:**
```typescript
// Metrics
POST /metrics (metric_name, value, tags) → void
GET /metrics/query (query, time_range) → MetricData[]

// Logs
POST /logs (log_entry) → void
GET /logs/search (query, time_range, filters) → LogEntries[]

// Alerts
POST /alerts/create (alert_rule) → AlertId
GET /alerts/active → Alert[]

// Dashboards
GET /dashboards → Dashboard[]
POST /dashboards/create (dashboard_config) → DashboardId
```

**Integration Points:**
- **Input:** All services (via OpenTelemetry SDK)
- **Output:** Control-Hub-Web (dashboard embed), Slack (alerts)

---

#### 9. **Control-Hub-Web**
**Type:** Web Application
**Purpose:** Unified web interface for ecosystem management

**Key Features:**
- **Dashboard overview:**
  - Active workflows (running, queued, completed)
  - Recent grants (from Collector-MCP)
  - Document statistics (from DataMan-MCP)
  - Cost tracking (AI spend today/this week/this month)
  - System health (service status)

- **Service management:**
  - Start/stop services (if self-hosted)
  - View logs (integrated from Observatory)
  - Configuration management (env vars, secrets)
  - Service dependency graph

- **Workflow builder:**
  - Visual workflow editor (drag-and-drop)
  - YAML editor with syntax highlighting
  - Workflow templates (Research→Spec→Code, Grant Analysis)
  - Test workflow execution

- **Team management:**
  - User accounts and permissions
  - Role assignment (Admin, Developer, Viewer)
  - Usage reports per user
  - API key management

- **Project browser:**
  - List all specs (from Spec-Synthesizer-MCP)
  - View generated code (GitHub/GitLab integration)
  - Track deployments (from DevOps-Conductor-MCP)
  - Git integration viewer

- **Real-time features:**
  - Live workflow progress (WebSocket)
  - Notifications (workflow completion, errors)
  - Log streaming

**Tech Stack:**
- **Frontend:** React 18 + TypeScript
- **State Management:** Zustand or TanStack Query
- **UI Library:** TailwindCSS + shadcn/ui
- **Visualization:** Recharts (dashboards), ReactFlow (workflow builder)
- **Real-time:** WebSocket (Socket.io)
- **Build:** Vite
- **Backend:** Node.js + Express (or use Gateway-Hub directly)

**Integration Points:**
- **Backend:** Gateway-Hub (all API calls proxied)
- **Auth:** Gateway-Hub (JWT tokens)
- **Real-time:** Direct WebSocket to services for logs/updates

---

## System Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       PRESENTATION LAYER                        │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Control-Hub-Web (React App)                  │ │
│  │     Dashboard │ Workflows │ Projects │ Team │ Logs        │ │
│  └───────────────────────┬───────────────────────────────────┘ │
│                          │ HTTPS/WSS                           │
└──────────────────────────┼─────────────────────────────────────┘
                           │
┌──────────────────────────┼─────────────────────────────────────┐
│                    API GATEWAY LAYER                            │
│                          │                                      │
│  ┌───────────────────────▼───────────────────────────────────┐ │
│  │                   Gateway-Hub                             │ │
│  │  Authentication (JWT, OAuth 2.1, API Keys)                │ │
│  │  Authorization (RBAC)                                     │ │
│  │  Rate Limiting (100 req/min per user)                     │ │
│  │  Request Routing (to MCP servers)                         │ │
│  │  Cost Controls (AI spend quotas)                          │ │
│  └───────────────────────┬───────────────────────────────────┘ │
│                          │ MCP Protocol (HTTP+SSE)             │
└──────────────────────────┼─────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼──────┐  ┌────────▼───────┐  ┌──────▼───────┐
│ ORCHESTRATION│  │  DATA LAYER    │  │ BUSINESS MCP │
│              │  │                │  │   SERVERS    │
│ ┌──────────┐ │  │ ┌────────────┐ │  │ ┌──────────┐ │
│ │  Vibes-  │ │  │ │  DataMan-  │ │  │ │Collector-│ │
│ │ Director │ │  │ │    MCP     │ │  │ │   MCP    │ │
│ │          │ │  │ │            │ │  │ │          │ │
│ │ Workflow │ │  │ │ Document   │ │  │ │  Grant   │ │
│ │ Engine   │ │  │ │ Storage    │ │  │ │  Data    │ │
│ └────┬─────┘ │  │ └────┬───────┘ │  │ └──────────┘ │
│      │       │  │      │         │  │              │
└──────┼───────┘  └──────┼─────────┘  └──────────────┘
       │                 │
       │ orchestrates    │ stores/retrieves
       │                 │
┌──────▼─────────────────▼──────────────────────────────────────┐
│                    CORE PIPELINE LAYER                        │
│                  (Spec → Code → Test)                         │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │Research-        │  │Spec-Synthesizer-│  │Codesmith-    │ │
│  │Harvester-MCP    │→ │MCP              │→ │MCP           │ │
│  │                 │  │                 │  │              │ │
│  │Web Search       │  │Requirements →   │  │Specs →       │ │
│  │Content Extract  │  │Specifications   │  │Code          │ │
│  └────────┬────────┘  └────────┬────────┘  └──────┬───────┘ │
│           │                    │                   │         │
│           │ stores research    │ stores specs      │ commits │
│           ▼                    ▼                   ▼         │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              RAG-Engine-MCP                             │ │
│  │  Vector DB │ Embeddings │ Semantic Search              │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │Sandbox-Runner-  │  │DevOps-Conductor-│  │Git Repos     │ │
│  │MCP              │  │MCP              │  │(GitHub/      │ │
│  │                 │  │                 │  │ GitLab)      │ │
│  │Container        │  │CI/CD            │  │              │ │
│  │Testing          │  │Deployment       │  │Code Storage  │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└───────────────────────────────────────────────────────────────┘
                           │
                           │ metrics, logs, traces
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  OBSERVABILITY LAYER                            │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   Observatory                             │ │
│  │  (OpenObserve or SigNoz)                                  │ │
│  │                                                           │ │
│  │  Centralized Logs │ Metrics │ Traces │ Alerts            │ │
│  │  Cost Dashboards │ Performance Monitoring                │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow: Research → Spec → Code → Test → Deploy

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. RESEARCH PHASE                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ User creates task → Vibes-Director                              │
│        ↓                                                        │
│ Vibes-Director → Research-Harvester-MCP                         │
│        ↓                                                        │
│ Web search (SearXNG, Perplexity) → Extract content             │
│        ↓                                                        │
│ Store in DataMan-MCP + embed in RAG-Engine-MCP                  │
│        ↓                                                        │
│ Generate research report                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. SPECIFICATION PHASE                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Vibes-Director → Spec-Synthesizer-MCP                           │
│        ↓                                                        │
│ Retrieve research from RAG-Engine-MCP (agentic retrieval)       │
│        ↓                                                        │
│ Generate specs:                                                 │
│   - User stories                                                │
│   - Technical design                                            │
│   - API contracts                                               │
│   - Database schema                                             │
│   - Test plan                                                   │
│        ↓                                                        │
│ Validate spec completeness                                      │
│        ↓                                                        │
│ Save to Git repo (branch: spec/{task-id})                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. CODE GENERATION PHASE                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Vibes-Director → Codesmith-MCP                                  │
│        ↓                                                        │
│ Read spec from Git                                              │
│        ↓                                                        │
│ Generate architecture scaffold                                  │
│        ↓                                                        │
│ Generate code:                                                  │
│   - Claude Sonnet 4.5 (complex logic, architecture)             │
│   - Local models (boilerplate, repetitive code)                 │
│        ↓                                                        │
│ Create feature branch (feature/{task-id})                       │
│        ↓                                                        │
│ Commit code with meaningful messages                            │
│        ↓                                                        │
│ Run linter + formatter                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. TESTING PHASE                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Vibes-Director → Sandbox-Runner-MCP                             │
│        ↓                                                        │
│ Create isolated container (with dependencies)                   │
│        ↓                                                        │
│ Inject code from feature branch                                 │
│        ↓                                                        │
│ Run tests (unit, integration, e2e)                              │
│        ↓                                                        │
│ Collect results (JUnit XML, coverage)                           │
│        ↓                                                        │
│ IF tests pass → proceed                                         │
│ IF tests fail → Codesmith-MCP fixes → retry                     │
│        ↓                                                        │
│ Destroy container                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. REVIEW & MERGE PHASE                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Codesmith-MCP → Create PR (with spec summary)                   │
│        ↓                                                        │
│ Notify team (Slack, email)                                      │
│        ↓                                                        │
│ Code review (human or AI agent)                                 │
│        ↓                                                        │
│ Apply feedback → Codesmith-MCP updates                          │
│        ↓                                                        │
│ Merge to main branch                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. DEPLOYMENT PHASE                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Git webhook (merge to main) → DevOps-Conductor-MCP              │
│        ↓                                                        │
│ Trigger CI pipeline:                                            │
│   - Build (Docker image)                                        │
│   - Test (via Sandbox-Runner-MCP)                               │
│   - Security scan                                               │
│        ↓                                                        │
│ Deploy to environment (dev first, then staging, then prod)      │
│        ↓                                                        │
│ Run database migrations                                         │
│        ↓                                                        │
│ Health check (readiness probe)                                  │
│        ↓                                                        │
│ IF healthy → complete                                           │
│ IF unhealthy → rollback                                         │
│        ↓                                                        │
│ Notify team (deployment complete)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### MCP Protocol Communication (2025 Standard)

All services communicate using the **Model Context Protocol (MCP)**:

**Transport:** HTTP with Server-Sent Events (SSE)
- Replaces older stdio/SSE-only transports
- Supports cloud deployments (AWS Lambda, Google Cloud Run)
- Bi-directional communication
- Chunked transfer encoding for streaming

**Authentication:** OAuth 2.1 (new in June 2025 spec)
- Authorization Code Flow with PKCE
- Client credentials for service-to-service
- Token refresh support

**Message Format:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "method": "tools/call",
  "params": {
    "name": "generate_spec",
    "arguments": {
      "research_doc_id": "doc-456",
      "project_context": "E-commerce platform"
    }
  }
}
```

**Service Discovery:**
- MCP servers register with Gateway-Hub on startup
- Gateway-Hub maintains service registry
- Health checks every 30 seconds

---

## Technology Stack

### Programming Languages

| Service | Language | Justification |
|---------|----------|---------------|
| Collector-MCP | TypeScript | Existing (maintain consistency) |
| Vibes-Director | TypeScript | Existing (maintain consistency) |
| DataMan-MCP | Python | Existing (maintain consistency) |
| Spec-Synthesizer-MCP | Python | NLP libraries, Pydantic validation |
| Codesmith-MCP | TypeScript | Git integration, code generation |
| Sandbox-Runner-MCP | Go | Performance, Docker SDK |
| RAG-Engine-MCP | Python | ML/AI libraries, vector DB clients |
| Research-Harvester-MCP | Python | Scraping libraries, async support |
| DevOps-Conductor-MCP | Python | Flexibility, webhook handling |
| Gateway-Hub | Go | Performance, concurrent connections |
| Observatory | Go/Python | Depends on chosen platform |
| Control-Hub-Web | TypeScript | React, full-stack JS |

### Key Dependencies

**AI Models:**
- **Claude Sonnet 4.5:** High-level reasoning, spec generation, complex code
- **Local models (Qwen-Coder, DeepSeek-Coder):** Boilerplate, low-level code
- **Embedding models:** OpenAI text-embedding-3 or BGE-M3 (local)

**Databases:**
- **PostgreSQL:** Relational data (specs, workflows, users)
- **Redis:** Caching, rate limiting, job queues
- **Vector DB:** Weaviate or Chroma (for RAG)
- **Elasticsearch:** (Optional) Hybrid search with RAG

**Infrastructure:**
- **Docker:** Containerization
- **Docker Compose:** Local development
- **Kubernetes:** (Optional) Production scaling
- **Git:** Code versioning (GitHub or GitLab)

**Observability:**
- **OpenObserve or SigNoz:** Logs, metrics, traces
- **OpenTelemetry SDK:** Instrumentation

**Frontend:**
- **React 18 + TypeScript**
- **TailwindCSS + shadcn/ui**
- **Vite**

---

## Integration Patterns

### 1. **Service-to-Service Communication**

**Pattern:** MCP Protocol over HTTP+SSE
- All services expose MCP endpoints
- Gateway-Hub acts as proxy/router
- Services register on startup

**Example Flow:**
```
Control-Hub-Web (browser)
  → Gateway-Hub (validates JWT, checks RBAC)
    → Vibes-Director (orchestrates workflow)
      → Spec-Synthesizer-MCP (generates spec)
        → RAG-Engine-MCP (retrieves context)
```

### 2. **Authentication Flow**

**Pattern:** Centralized authentication, decentralized authorization

```
1. User logs in → Gateway-Hub
2. Gateway-Hub validates credentials → JWT (access + refresh)
3. Browser stores JWT
4. Every request includes JWT in Authorization header
5. Gateway-Hub validates JWT signature
6. Gateway-Hub checks RBAC (can user call this tool?)
7. If authorized → forward to MCP server
8. MCP server performs additional authorization (resource-level)
```

### 3. **Workflow Orchestration**

**Pattern:** Event-driven with Vibes-Director as central coordinator

```
1. User creates task in Control-Hub-Web
2. Task → Gateway-Hub → Vibes-Director
3. Vibes-Director creates workflow steps:
   - Step 1: Research-Harvester-MCP.web_search()
   - Step 2: Spec-Synthesizer-MCP.generate_spec()
   - Step 3: Codesmith-MCP.implement_feature()
   - Step 4: Sandbox-Runner-MCP.run_tests()
   - Step 5: DevOps-Conductor-MCP.deploy()
4. Each step emits events (started, progress, completed, failed)
5. Vibes-Director tracks state and progresses workflow
6. Control-Hub-Web subscribes to events via WebSocket
```

### 4. **Data Storage**

**Pattern:** Domain-specific storage with DataMan-MCP as coordinator

- **Research documents:** DataMan-MCP (file storage) + RAG-Engine-MCP (vectors)
- **Specifications:** Git repositories (source of truth) + PostgreSQL (metadata)
- **Code:** Git repositories
- **Metrics/Logs:** Observatory (time-series DB)
- **User data:** PostgreSQL

### 5. **Error Handling**

**Pattern:** Retry with exponential backoff + circuit breaker

```
1. Service A calls Service B (MCP tool)
2. Service B fails (timeout, 5xx error)
3. Service A retries:
   - Retry 1: after 1 second
   - Retry 2: after 2 seconds
   - Retry 3: after 4 seconds
4. If all retries fail → circuit breaker opens
5. Circuit breaker: fail fast for 60 seconds
6. After 60 seconds → half-open (try one request)
7. If success → close circuit, resume normal
8. If failure → keep circuit open
```

### 6. **Cost Management**

**Pattern:** Usage tracking + quotas + budgets

```
1. Every AI model call is logged:
   - User ID
   - Service
   - Model (Claude Sonnet 4.5, gpt-4, etc.)
   - Input tokens
   - Output tokens
   - Cost (calculated)
2. Observatory aggregates costs per user/day
3. Gateway-Hub checks quota before allowing expensive operations:
   - If user spent > $50 today → reject (configurable)
4. Daily reports sent to admins
```

---

## Implementation Roadmap

### Phase 1: MVP Pipeline (Months 1-3)
**Goal:** Working Spec→Code→Test pipeline for a single project

#### Month 1: Foundation
**Week 1-2: Infrastructure Setup**
- [ ] Set up development environment
  - Docker Compose for local services
  - PostgreSQL + Redis
  - Git repositories (GitHub or GitLab organization)
- [ ] Implement Gateway-Hub (basic version)
  - JWT authentication
  - Service registry
  - Request routing (no RBAC yet)
- [ ] Set up Observatory (OpenObserve)
  - Docker deployment
  - Basic dashboards (service health, request count)

**Week 3-4: Spec-Synthesizer-MCP**
- [ ] Implement MCP server scaffold
- [ ] Build spec generation:
  - User story template
  - Technical design template
  - API contract template (OpenAPI)
- [ ] Git integration (save specs to repo)
- [ ] OpenTelemetry instrumentation

**Deliverable:** Can generate specs from text descriptions

---

#### Month 2: Code Generation
**Week 1-2: Codesmith-MCP (Part 1)**
- [ ] Implement MCP server scaffold
- [ ] Build project scaffold generator:
  - Node.js/Express template
  - Python/FastAPI template
  - React template
- [ ] Git workflow automation:
  - Branch creation
  - Commits
- [ ] Claude Sonnet 4.5 integration

**Week 3-4: Codesmith-MCP (Part 2)**
- [ ] Implement code generation:
  - API routes from OpenAPI spec
  - Database models from schema
  - Basic CRUD operations
- [ ] Linting and formatting integration
- [ ] PR creation

**Deliverable:** Can generate working code from specs

---

#### Month 3: Testing & Integration
**Week 1-2: Sandbox-Runner-MCP**
- [ ] Implement MCP server scaffold
- [ ] Build container orchestration:
  - Docker SDK integration
  - Pre-built images (Node, Python)
  - Resource limits
- [ ] Test execution:
  - Jest/Pytest integration
  - Result parsing (JUnit XML)
- [ ] Log streaming

**Week 3-4: Vibes-Director Integration**
- [ ] Update Vibes-Director for workflow orchestration
- [ ] Create "Spec→Code→Test" workflow template
- [ ] Implement event streaming (for UI updates)
- [ ] Error handling and retries

**Deliverable:** End-to-end automated pipeline working

---

### Phase 2: Production Ready (Months 4-6)
**Goal:** Secure, observable, deployable system for small team

#### Month 4: Security & Access Control
- [ ] Enhance Gateway-Hub:
  - RBAC implementation
  - Rate limiting (Redis-based)
  - Cost quotas
  - OAuth 2.1 support (MCP spec compliance)
- [ ] Secrets management in DevOps-Conductor-MCP
- [ ] Security audit of all services

#### Month 5: Deployment & Observability
- [ ] DevOps-Conductor-MCP implementation:
  - Git webhooks
  - CI/CD pipelines
  - Deployment strategies (rolling update)
- [ ] Enhanced Observatory:
  - Cost dashboards
  - Performance monitoring
  - Alerting (Slack integration)
- [ ] Improve OpenTelemetry coverage (all services)

#### Month 6: Control-Hub-Web
- [ ] Build React application:
  - Authentication (login, logout)
  - Dashboard (workflows, costs, services)
  - Workflow builder (visual editor)
  - Team management
- [ ] WebSocket integration (real-time updates)
- [ ] Responsive design (mobile-friendly)

**Deliverable:** Production-ready system with web interface

---

### Phase 3: Full Automation (Months 7-9)
**Goal:** Automated research and RAG capabilities

#### Month 7: RAG Engine
- [ ] RAG-Engine-MCP implementation:
  - Weaviate setup (self-hosted)
  - Embedding generation (BGE-M3)
  - Vector search
- [ ] Integrate with DataMan-MCP (auto-embed documents)
- [ ] Agentic RAG implementation (iterative retrieval)

#### Month 8: Research Automation
- [ ] Research-Harvester-MCP implementation:
  - SearXNG integration (self-hosted)
  - Web scraping (Playwright)
  - Content extraction
- [ ] Task queue (Celery + Redis)
- [ ] Research report generation

#### Month 9: End-to-End Workflows
- [ ] "Research→Spec→Code→Test→Deploy" workflow in Vibes-Director
- [ ] Automated grant analysis workflow (Collector-MCP → Research)
- [ ] Documentation generation
- [ ] Performance optimization

**Deliverable:** Fully automated development pipeline

---

### Phase 4: Polish & Scale (Months 10-12)
**Goal:** Refinement, team onboarding, scaling preparation

#### Month 10: UX Improvements
- [ ] Workflow templates library
- [ ] Code review assistance (AI reviewer agent)
- [ ] Improved error messages and debugging
- [ ] User onboarding flow

#### Month 11: Team Features
- [ ] Multi-tenancy (if needed)
- [ ] Team collaboration features
- [ ] Usage analytics per user
- [ ] Cost allocation

#### Month 12: Scaling Prep
- [ ] Kubernetes deployment manifests (optional)
- [ ] Load testing
- [ ] Backup and disaster recovery
- [ ] Documentation (user guides, API docs)

**Deliverable:** Polished, scalable system ready for team expansion

---

## 2025 Best Practices & Standards

### 1. MCP Protocol Compliance
- ✅ Use HTTP transport with SSE (2025 spec)
- ✅ Implement OAuth 2.1 for authentication
- ✅ Follow tool naming conventions: `service_action` (e.g., `spec_generate`, `code_create`)
- ✅ Provide detailed tool descriptions for AI models
- ✅ Version MCP server APIs (e.g., `/v1/tools`)

### 2. AI Agent Orchestration
- ✅ Use LangGraph for complex workflows (conditional logic, loops)
- ✅ Implement agentic patterns: reasoning → action → observation → repeat
- ✅ Combine Claude (high-level) with local models (low-level) for cost efficiency
- ✅ Track AI costs per operation and user
- ✅ Use Claude Agent SDK for "spec-to-repo" workflows

### 3. Container Security
- ✅ Run containers as non-root user
- ✅ Use read-only filesystems (except /tmp)
- ✅ Apply resource limits (CPU, memory, timeout)
- ✅ Network isolation (no internet access unless explicitly allowed)
- ✅ Consider gVisor or Kata Containers for untrusted code
- ✅ Scan images for vulnerabilities (Trivy, Grype)

### 4. RAG Architecture
- ✅ Use hybrid search (semantic + keyword) for better retrieval
- ✅ Implement agentic RAG (iterative retrieval with reasoning)
- ✅ Chunk documents semantically (not just fixed-size)
- ✅ Add metadata for filtering (date, source, author)
- ✅ Re-rank results for relevance
- ✅ Cache embeddings to reduce costs

### 5. API Gateway & Auth
- ✅ Centralize authentication at gateway
- ✅ Decentralize authorization to services (resource-level)
- ✅ Use JWT (short-lived access tokens + refresh tokens)
- ✅ Implement RBAC (roles: Admin, Developer, Viewer, Agent)
- ✅ Rate limit per user and per endpoint
- ✅ Log all authentication events for audit

### 6. Observability
- ✅ Use OpenTelemetry SDK in all services
- ✅ Structured logging (JSON format)
- ✅ Trace distributed requests (trace ID propagation)
- ✅ Monitor AI costs separately (by model, by user)
- ✅ Set up alerts for anomalies (latency spikes, error rates)
- ✅ Self-host observability platform (OpenObserve or SigNoz)

### 7. Git Workflows
- ✅ Use conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
- ✅ Branch naming: `feature/{task-id}`, `spec/{task-id}`
- ✅ PR templates with checklist (tests, docs, security)
- ✅ Require PR reviews (human or AI)
- ✅ Automate changelog generation from commits

### 8. Cost Management
- ✅ Track AI model usage (tokens in/out, cost)
- ✅ Set daily/monthly quotas per user
- ✅ Use local models for repetitive tasks (boilerplate code)
- ✅ Cache AI responses when possible
- ✅ Monitor and alert on unusual spending patterns

### 9. Documentation
- ✅ API documentation (OpenAPI specs for all services)
- ✅ Architecture decision records (ADRs)
- ✅ Runbooks for common operations
- ✅ User guides for non-technical team members
- ✅ Inline code comments for complex logic

### 10. Testing
- ✅ Unit tests (80%+ coverage target)
- ✅ Integration tests (service-to-service)
- ✅ E2E tests (critical workflows)
- ✅ Load tests (performance benchmarks)
- ✅ Security tests (OWASP top 10)

---

## Appendix

### A. Repository Naming Convention
- `{name}-mcp` for MCP servers (e.g., `spec-synthesizer-mcp`)
- `{name}-web` for web applications (e.g., `control-hub-web`)
- `{name}-service` for non-MCP services (e.g., `gateway-hub`)

### B. Environment Variables Standard
All services should support:
- `PORT` - HTTP port (default: 3000)
- `LOG_LEVEL` - Logging verbosity (default: info)
- `OTEL_ENDPOINT` - OpenTelemetry collector endpoint
- `DATABASE_URL` - PostgreSQL connection string
- `REDIS_URL` - Redis connection string
- `MCP_SERVER_NAME` - Service identifier for registry
- `GATEWAY_URL` - Gateway-Hub endpoint (for service-to-service calls)

### C. Cost Estimates (Small Team, 5 Users)
**Cloud AI APIs (monthly):**
- Claude Sonnet 4.5: ~$200-500 (for high-level tasks)
- Embeddings (OpenAI): ~$20-50
- **Total AI:** ~$250-600/month

**Self-Hosted Infrastructure (monthly):**
- VPS/server (16GB RAM, 8 vCPU): ~$50-100
- Backups: ~$20
- **Total Infra:** ~$70-120/month

**Total Estimated Monthly Cost:** $320-720 (~$64-144 per user)

### D. Alternative Tech Stacks Considered
- **Rust instead of Go:** More memory-safe but steeper learning curve
- **Deno instead of Node.js:** Better security but smaller ecosystem
- **Qdrant instead of Weaviate:** Simpler but fewer features
- **Grafana Stack instead of OpenObserve:** More flexible but complex setup

### E. Future Enhancements (Post-MVP)
- Multi-modal RAG (images, audio, video)
- Fine-tuned local models for specific domains
- Automated code refactoring agent
- Visual workflow designer (no-code)
- Integration marketplace (Zapier-like connectors)
- Mobile app (iOS/Android)

---

## Getting Started

To begin implementation:

1. **Review this document** with your team
2. **Set up Git organization** (GitHub or GitLab)
3. **Provision infrastructure** (VPS, databases)
4. **Follow Phase 1 roadmap** (start with Spec-Synthesizer-MCP)
5. **Iterate based on feedback** (adapt architecture as needed)

**Questions or feedback?** Update this document as the architecture evolves.

---

**Document Version:** 1.0
**Last Updated:** 2025-11-14
**Maintained By:** Development Team
**Next Review:** After Phase 1 completion
