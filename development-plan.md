# Vector Database as a Service — Phased Development Plan

> Project: 192-vector-database-as-a-service · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Rust | The vector engine (HNSW indexing, quantization, ANN search) is the performance-critical core. Qdrant demonstrates Rust's suitability for this domain: memory-safe, zero-cost abstractions, sub-millisecond query latency. The control plane uses Python for rapid iteration. |
| Vector engine framework | Custom Rust crate (`vectrus-engine`) | No reusable vector engine library exists at the quality level needed. HNSW, quantization, and payload filtering must be tightly integrated. Builds on the HNSW algorithm (Malkov & Yashunin, 2018). |
| Control plane language | Python 3.12+ | Control plane (tenant management, billing, RBAC, API gateway) is CRUD-heavy and benefits from rapid iteration. FastAPI provides automatic OpenAPI 3.1 generation required by the spec. |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 spec generation, async support for concurrent queries, Pydantic v2 for request/response validation. Matches the REST+OpenAPI pattern used by Pinecone, Qdrant, and Weaviate. |
| gRPC framework | tonic (Rust) + grpcio (Python) | High-throughput vector ingestion requires binary serialization. Protocol Buffers (proto3) provide 3-10x throughput over JSON for bulk upserts. gRPC is standard across Qdrant, Weaviate, and Milvus. |
| Control plane database | PostgreSQL 16 | Hybrid relational + JSONB model (Data Model Suggestion 3) provides relational integrity for stable entities and JSONB flexibility for variable config. GIN indexes on JSONB enable efficient containment queries. pgvector extension available if needed for internal embeddings. |
| Task queue | Celery 5.4 + Redis 7 | Async workloads: index building, embedding inference, drift detection, snapshot creation. Redis doubles as a cache layer for hot API key lookups and rate limiting. |
| Object storage | S3-compatible (MinIO for self-hosted) | Vector segment files, snapshots, and cold-tier data stored on object storage. S3 API is universal across AWS, GCP (via interop), Azure (via gateway), and MinIO. |
| Frontend | React 19 + TypeScript 5.5 + Vite 6 | Dashboard for collection management, query playground, usage analytics. React ecosystem has mature charting (Recharts) and table (TanStack Table) libraries. |
| CSS framework | Tailwind CSS 4 + shadcn/ui | Consistent, accessible UI components without custom design system overhead. shadcn/ui provides the dashboard patterns (data tables, forms, charts) needed. |
| Containerisation | Docker + Docker Compose (dev), Helm chart (production K8s) | Self-hosted deployment is a key differentiator vs. Pinecone/Turbopuffer. Helm chart targets Kubernetes for production clusters. |
| Testing (Python) | pytest 8 + pytest-asyncio + httpx (async test client) | Standard Python testing stack. httpx provides async test client for FastAPI. |
| Testing (Rust) | cargo test + criterion (benchmarks) | Built-in Rust test framework plus criterion for HNSW performance benchmarks. |
| Testing (Frontend) | Vitest + React Testing Library + Playwright | Unit/component tests with Vitest, E2E with Playwright for dashboard flows. |
| Code quality (Python) | Ruff (linting + formatting) + mypy (type checking) | Ruff replaces flake8+isort+black in a single fast tool. mypy for static type safety. |
| Code quality (Rust) | clippy + rustfmt | Standard Rust code quality tooling. |
| Observability | Prometheus client + OpenTelemetry SDK + Grafana dashboards | Matches the observability stack used by Qdrant and Milvus. Prometheus metrics endpoint is table-stakes per features.md. |
| Authentication | API key (primary) + JWT (dashboard sessions) + OAuth 2.0 (SSO) | API key auth follows Pinecone/Qdrant patterns. JWT for web dashboard sessions per RFC 7519. OAuth 2.0 (RFC 6749) for enterprise SSO. |
| SDK generation | OpenAPI Generator 7 | Auto-generate Python and TypeScript SDKs from the OpenAPI 3.1 spec. Pinecone and Qdrant both publish OpenAPI specs for SDK generation. |
| Embedding providers | OpenAI SDK + Sentence Transformers | Built-in inference API for embedding generation. OpenAI Embeddings API is the de-facto convention; Sentence Transformers for self-hosted models. |
| Package manager (Python) | uv | Fast, modern Python package manager with lockfile support. Replaces pip + pip-tools. |
| Package manager (JS) | pnpm 9 | Fast, disk-efficient, workspace-aware package manager for the dashboard and SDKs. |

---

### Project Structure

```
vectrus/
├── pyproject.toml                      # Python control plane package config
├── Cargo.toml                          # Rust workspace root
├── Dockerfile.engine                   # Vector engine container
├── Dockerfile.api                      # Control plane API container
├── Dockerfile.dashboard                # Dashboard container
├── docker-compose.yml                  # Local development stack
├── helm/
│   └── vectrus/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── engine-deployment.yaml
│           ├── api-deployment.yaml
│           ├── dashboard-deployment.yaml
│           ├── postgres-statefulset.yaml
│           ├── redis-deployment.yaml
│           └── ingress.yaml
├── proto/
│   ├── vectrus/
│   │   ├── vectors.proto               # Vector upsert/query/delete messages
│   │   ├── collections.proto           # Collection management messages
│   │   └── health.proto                # Health check messages
│   └── buf.yaml
├── engine/                             # Rust vector engine
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── hnsw/
│   │   │   ├── mod.rs
│   │   │   ├── graph.rs                # HNSW graph construction and search
│   │   │   ├── node.rs                 # Graph node with vector + payload
│   │   │   └── distance.rs             # Distance metric implementations
│   │   ├── quantization/
│   │   │   ├── mod.rs
│   │   │   ├── scalar.rs               # Scalar quantization (int8)
│   │   │   ├── binary.rs               # Binary quantization
│   │   │   └── product.rs              # Product quantization
│   │   ├── filter/
│   │   │   ├── mod.rs
│   │   │   ├── parser.rs               # Filter expression parser
│   │   │   └── evaluator.rs            # Payload filter evaluation
│   │   ├── storage/
│   │   │   ├── mod.rs
│   │   │   ├── segment.rs              # Immutable vector segment
│   │   │   ├── wal.rs                  # Write-ahead log
│   │   │   └── snapshot.rs             # Snapshot creation/restore
│   │   ├── sparse/
│   │   │   ├── mod.rs
│   │   │   └── inverted_index.rs       # Sparse vector inverted index (BM25)
│   │   ├── server/
│   │   │   ├── mod.rs
│   │   │   ├── grpc.rs                 # gRPC service implementation
│   │   │   └── http.rs                 # REST API (Axum)
│   │   └── config.rs                   # Engine configuration
│   ├── benches/
│   │   ├── hnsw_build.rs
│   │   └── hnsw_search.rs
│   └── tests/
│       ├── hnsw_test.rs
│       ├── filter_test.rs
│       └── integration_test.rs
├── api/                                # Python control plane
│   ├── src/
│   │   └── vectrus_api/
│   │       ├── __init__.py
│   │       ├── main.py                 # FastAPI application entry point
│   │       ├── config.py               # Settings via pydantic-settings
│   │       ├── models/
│   │       │   ├── __init__.py
│   │       │   ├── organization.py     # Organization Pydantic models
│   │       │   ├── collection.py       # Collection Pydantic models
│   │       │   ├── api_key.py          # API key models
│   │       │   ├── user.py             # User/membership models
│   │       │   └── billing.py          # Plan, usage, invoice models
│   │       ├── routes/
│   │       │   ├── __init__.py
│   │       │   ├── collections.py      # /v1/collections endpoints
│   │       │   ├── vectors.py          # /v1/collections/{id}/vectors endpoints
│   │       │   ├── query.py            # /v1/query endpoint
│   │       │   ├── organizations.py    # /v1/organizations endpoints
│   │       │   ├── api_keys.py         # /v1/api-keys endpoints
│   │       │   ├── users.py            # /v1/users endpoints
│   │       │   └── billing.py          # /v1/billing endpoints
│   │       ├── services/
│   │       │   ├── __init__.py
│   │       │   ├── collection_service.py
│   │       │   ├── vector_service.py
│   │       │   ├── auth_service.py
│   │       │   ├── billing_service.py
│   │       │   ├── embedding_service.py
│   │       │   └── engine_client.py    # gRPC client to Rust engine
│   │       ├── middleware/
│   │       │   ├── __init__.py
│   │       │   ├── auth.py             # API key + JWT auth middleware
│   │       │   ├── rate_limit.py       # Redis-backed rate limiting
│   │       │   └── telemetry.py        # OpenTelemetry middleware
│   │       ├── db/
│   │       │   ├── __init__.py
│   │       │   ├── session.py          # SQLAlchemy async session
│   │       │   ├── models.py           # SQLAlchemy ORM models
│   │       │   └── migrations/         # Alembic migrations
│   │       │       ├── env.py
│   │       │       └── versions/
│   │       ├── tasks/
│   │       │   ├── __init__.py
│   │       │   ├── index_build.py      # Celery task: trigger index build
│   │       │   ├── snapshot.py         # Celery task: create/restore snapshots
│   │       │   ├── usage_rollup.py     # Celery task: hourly usage aggregation
│   │       │   └── drift_detection.py  # Celery task: embedding drift analysis
│   │       └── ai/
│   │           ├── __init__.py
│   │           ├── index_tuner.py      # AI-driven index parameter tuning
│   │           ├── schema_advisor.py   # LLM-assisted payload schema design
│   │           ├── drift_detector.py   # Embedding distribution drift detection
│   │           └── nl_query.py         # Natural language query translation
│   └── tests/
│       ├── conftest.py
│       ├── test_collections.py
│       ├── test_vectors.py
│       ├── test_auth.py
│       ├── test_billing.py
│       └── test_integration.py
├── dashboard/                          # React web dashboard
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── ui/                     # shadcn/ui components
│   │   │   ├── collections/
│   │   │   ├── query-playground/
│   │   │   └── analytics/
│   │   ├── hooks/
│   │   ├── lib/
│   │   │   ├── api-client.ts           # Generated from OpenAPI spec
│   │   │   └── auth.ts
│   │   └── pages/
│   │       ├── Dashboard.tsx
│   │       ├── Collections.tsx
│   │       ├── QueryPlayground.tsx
│   │       ├── ApiKeys.tsx
│   │       ├── Settings.tsx
│   │       └── Usage.tsx
│   └── tests/
│       ├── collections.spec.ts
│       └── query-playground.spec.ts
├── sdks/
│   ├── python/                         # Auto-generated + hand-tuned Python SDK
│   │   ├── pyproject.toml
│   │   ├── src/vectrus/
│   │   │   ├── __init__.py
│   │   │   ├── client.py
│   │   │   ├── models.py
│   │   │   └── async_client.py
│   │   └── tests/
│   └── typescript/                     # Auto-generated + hand-tuned TS SDK
│       ├── package.json
│       ├── src/
│       │   ├── index.ts
│       │   ├── client.ts
│       │   └── types.ts
│       └── tests/
├── mcp-server/                         # MCP server for AI agent integration
│   ├── pyproject.toml
│   └── src/vectrus_mcp/
│       ├── __init__.py
│       ├── server.py
│       └── tools.py
├── grafana/
│   └── dashboards/
│       ├── engine-metrics.json
│       └── api-metrics.json
└── docs/
    ├── openapi.yaml                    # Published OpenAPI 3.1 spec
    └── architecture.md
```

---

## Phase 1: Foundation — Project Scaffold and Core Configuration

### Purpose

Establish the project skeleton, build tooling, CI configuration, and core configuration types that every subsequent phase depends on. After this phase, the project compiles, tests run (even if empty), Docker images build, and the development workflow is functional.

### Tasks

#### 1.1 — Rust Workspace and Engine Crate Scaffold

**What**: Initialize the Cargo workspace with the `engine` crate, basic module structure, and build tooling.

**Design**:

```rust
// engine/src/config.rs
use serde::Deserialize;
use std::path::PathBuf;

#[derive(Debug, Clone, Deserialize)]
pub struct EngineConfig {
    /// Directory for vector segment storage
    pub data_dir: PathBuf,
    /// gRPC listen address
    pub grpc_addr: String,      // default: "0.0.0.0:50051"
    /// REST API listen address
    pub http_addr: String,      // default: "0.0.0.0:8081"
    /// Maximum vector dimension supported
    pub max_dimension: u32,     // default: 65536
    /// WAL segment size in bytes
    pub wal_segment_bytes: u64, // default: 64 * 1024 * 1024 (64 MB)
    /// Number of threads for index building
    pub index_build_threads: usize, // default: num_cpus / 2
    /// Snapshot storage path
    pub snapshot_dir: PathBuf,
}

impl Default for EngineConfig {
    fn default() -> Self {
        Self {
            data_dir: PathBuf::from("./data"),
            grpc_addr: "0.0.0.0:50051".to_string(),
            http_addr: "0.0.0.0:8081".to_string(),
            max_dimension: 65536,
            wal_segment_bytes: 64 * 1024 * 1024,
            index_build_threads: num_cpus::get() / 2,
            snapshot_dir: PathBuf::from("./snapshots"),
        }
    }
}

// engine/src/hnsw/distance.rs
/// Distance metric applied to vector pairs during search.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum DistanceMetric {
    Cosine,
    Euclidean,
    DotProduct,
}

impl DistanceMetric {
    /// Compute distance between two vectors. Lower is closer for Cosine/Euclidean;
    /// higher is closer for DotProduct (negated internally for min-heap compatibility).
    pub fn compute(&self, a: &[f32], b: &[f32]) -> f32 {
        match self {
            Self::Cosine => cosine_distance(a, b),
            Self::Euclidean => euclidean_distance(a, b),
            Self::DotProduct => -dot_product(a, b), // negate for min-heap
        }
    }
}
```

```toml
# Cargo.toml (workspace root)
[workspace]
members = ["engine"]
resolver = "2"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
tonic = "0.12"
prost = "0.13"
axum = "0.8"
tracing = "0.1"
tracing-subscriber = "0.3"
num_cpus = "1"
```

**Testing**:
- `Unit: EngineConfig::default() returns valid config with expected field values`
- `Unit: EngineConfig deserialization from TOML file with all fields set`
- `Unit: EngineConfig deserialization with missing optional fields uses defaults`
- `Unit: DistanceMetric::Cosine compute on identical vectors returns 0.0`
- `Unit: DistanceMetric::Euclidean compute on known vectors returns expected distance`
- `Unit: DistanceMetric::DotProduct compute on orthogonal vectors returns 0.0`

#### 1.2 — Python Control Plane Scaffold

**What**: Initialize the Python project with FastAPI application, configuration management, and database connection.

**Design**:

```python
# api/src/vectrus_api/config.py
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    """Application settings loaded from environment variables."""

    # Database
    database_url: str = Field(
        default="postgresql+asyncpg://vectrus:vectrus@localhost:5432/vectrus",
        description="PostgreSQL connection string"
    )
    database_pool_size: int = Field(default=20)
    database_max_overflow: int = Field(default=10)

    # Redis
    redis_url: str = Field(default="redis://localhost:6379/0")

    # Engine gRPC
    engine_grpc_url: str = Field(default="localhost:50051")

    # API
    api_host: str = Field(default="0.0.0.0")
    api_port: int = Field(default=8080)
    api_key_hash_algorithm: str = Field(default="argon2")

    # Auth
    jwt_secret_key: str = Field(default="change-me-in-production")
    jwt_algorithm: str = Field(default="HS256")
    jwt_expiry_minutes: int = Field(default=60)

    # Observability
    otel_exporter_endpoint: str | None = Field(default=None)
    prometheus_enabled: bool = Field(default=True)

    model_config = {"env_prefix": "VECTRUS_"}


# api/src/vectrus_api/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from vectrus_api.config import Settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: init DB pool, Redis, gRPC channel
    yield
    # Shutdown: close connections

def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or Settings()
    app = FastAPI(
        title="Vectrus API",
        version="0.1.0",
        lifespan=lifespan,
        openapi_url="/v1/openapi.json",
        docs_url="/v1/docs",
    )
    app.state.settings = settings
    return app
```

```toml
# pyproject.toml (api/)
[project]
name = "vectrus-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.34",
    "pydantic>=2.10",
    "pydantic-settings>=2.7",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.30",
    "alembic>=1.14",
    "redis>=5.2",
    "celery>=5.4",
    "grpcio>=1.68",
    "grpcio-tools>=1.68",
    "argon2-cffi>=23.1",
    "pyjwt>=2.9",
    "httpx>=0.28",
    "opentelemetry-api>=1.29",
    "opentelemetry-sdk>=1.29",
    "opentelemetry-instrumentation-fastapi>=0.50b",
    "prometheus-client>=0.21",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3",
    "pytest-asyncio>=0.24",
    "ruff>=0.8",
    "mypy>=1.13",
]
```

**Testing**:
- `Unit: Settings() loads defaults when no env vars set`
- `Unit: Settings() overrides database_url from VECTRUS_DATABASE_URL env var`
- `Unit: create_app() returns FastAPI instance with correct title and version`
- `Integration: GET /v1/docs returns 200 with Swagger UI HTML`
- `Integration: GET /v1/openapi.json returns valid OpenAPI 3.1 JSON`

#### 1.3 — Docker Compose Development Stack

**What**: Create Docker Compose configuration for the full local development stack (PostgreSQL, Redis, engine, API).

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: vectrus
      POSTGRES_USER: vectrus
      POSTGRES_PASSWORD: vectrus
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U vectrus"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s

  engine:
    build:
      context: .
      dockerfile: Dockerfile.engine
    ports:
      - "50051:50051"
      - "8081:8081"
    volumes:
      - engine_data:/data
    environment:
      VECTRUS_DATA_DIR: /data
      VECTRUS_GRPC_ADDR: "0.0.0.0:50051"
      VECTRUS_HTTP_ADDR: "0.0.0.0:8081"

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      engine:
        condition: service_started
    environment:
      VECTRUS_DATABASE_URL: "postgresql+asyncpg://vectrus:vectrus@postgres:5432/vectrus"
      VECTRUS_REDIS_URL: "redis://redis:6379/0"
      VECTRUS_ENGINE_GRPC_URL: "engine:50051"

volumes:
  pgdata:
  engine_data:
```

**Testing**:
- `E2E: docker compose up --build completes without errors`
- `E2E: docker compose ps shows all services healthy`
- `E2E: curl http://localhost:8080/v1/docs returns 200`
- `E2E: curl http://localhost:8081/health returns 200 from engine`

#### 1.4 — Protocol Buffer Definitions

**What**: Define the gRPC service contracts for communication between the Python control plane and Rust engine.

**Design**:

```protobuf
// proto/vectrus/vectors.proto
syntax = "proto3";
package vectrus.vectors;

service VectorService {
    rpc Upsert(UpsertRequest) returns (UpsertResponse);
    rpc Query(QueryRequest) returns (QueryResponse);
    rpc Delete(DeleteRequest) returns (DeleteResponse);
    rpc Count(CountRequest) returns (CountResponse);
}

message Vector {
    string id = 1;
    repeated float values = 2;
    map<string, Value> metadata = 3;
    optional SparseVector sparse_values = 4;
}

message SparseVector {
    repeated uint32 indices = 1;
    repeated float values = 2;
}

message Value {
    oneof kind {
        string string_value = 1;
        int64 int_value = 2;
        double float_value = 3;
        bool bool_value = 4;
    }
}

message UpsertRequest {
    string collection_id = 1;
    string namespace = 2;
    repeated Vector vectors = 3;
}

message UpsertResponse {
    uint64 upserted_count = 1;
}

message QueryRequest {
    string collection_id = 1;
    string namespace = 2;
    repeated float vector = 3;
    optional SparseVector sparse_vector = 4;
    uint32 top_k = 5;
    optional FilterExpression filter = 6;
    bool include_values = 7;
    bool include_metadata = 8;
    optional float alpha = 9; // hybrid search weight (0.0 = sparse only, 1.0 = dense only)
}

message FilterExpression {
    oneof expr {
        FieldCondition field = 1;
        CombinedCondition combined = 2;
    }
}

message FieldCondition {
    string field = 1;
    string operator = 2;  // eq, ne, gt, gte, lt, lte, in, nin
    Value value = 3;
    repeated Value values = 4; // for in/nin operators
}

message CombinedCondition {
    string operator = 1; // and, or
    repeated FilterExpression conditions = 2;
}

message QueryResponse {
    repeated ScoredVector results = 1;
    string namespace = 2;
}

message ScoredVector {
    string id = 1;
    float score = 2;
    repeated float values = 3;
    map<string, Value> metadata = 4;
}

message DeleteRequest {
    string collection_id = 1;
    string namespace = 2;
    repeated string ids = 3;
    optional FilterExpression filter = 4;
}

message DeleteResponse {
    uint64 deleted_count = 1;
}

message CountRequest {
    string collection_id = 1;
    string namespace = 2;
}

message CountResponse {
    uint64 count = 1;
}

// proto/vectrus/collections.proto
syntax = "proto3";
package vectrus.collections;

service CollectionService {
    rpc CreateCollection(CreateCollectionRequest) returns (CreateCollectionResponse);
    rpc DeleteCollection(DeleteCollectionRequest) returns (DeleteCollectionResponse);
    rpc GetCollectionInfo(GetCollectionInfoRequest) returns (CollectionInfo);
    rpc BuildIndex(BuildIndexRequest) returns (BuildIndexResponse);
    rpc CreateSnapshot(CreateSnapshotRequest) returns (CreateSnapshotResponse);
}

message CreateCollectionRequest {
    string collection_id = 1;
    uint32 dimension = 2;
    string distance_metric = 3; // cosine, euclidean, dot_product
    IndexConfig index_config = 4;
}

message IndexConfig {
    string index_type = 1; // hnsw, flat
    uint32 hnsw_m = 2;     // default: 16
    uint32 hnsw_ef_construction = 3; // default: 200
    string quantization = 4; // none, scalar, binary, product
    uint32 quant_bits = 5;   // for scalar: 8; for product: code_size
}

message CreateCollectionResponse {
    string collection_id = 1;
    string status = 2;
}

message DeleteCollectionRequest {
    string collection_id = 1;
}

message DeleteCollectionResponse {
    bool success = 1;
}

message GetCollectionInfoRequest {
    string collection_id = 1;
}

message CollectionInfo {
    string collection_id = 1;
    uint32 dimension = 2;
    string distance_metric = 3;
    uint64 vector_count = 4;
    uint64 storage_bytes = 5;
    string index_status = 6;
    IndexConfig index_config = 7;
}

message BuildIndexRequest {
    string collection_id = 1;
    IndexConfig config = 2;
}

message BuildIndexResponse {
    string status = 1; // building, completed, error
}

message CreateSnapshotRequest {
    string collection_id = 1;
}

message CreateSnapshotResponse {
    string snapshot_id = 1;
    string path = 2;
    uint64 size_bytes = 3;
}

// proto/vectrus/health.proto
syntax = "proto3";
package vectrus.health;

service HealthService {
    rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
}

message HealthCheckRequest {}

message HealthCheckResponse {
    string status = 1; // serving, not_serving
    string version = 2;
    uint64 uptime_seconds = 3;
    uint64 total_vectors = 4;
    uint32 collection_count = 5;
}
```

**Testing**:
- `Unit: protoc compiles all .proto files without errors`
- `Unit: generated Rust types serialize and deserialize correctly`
- `Unit: generated Python stubs import without errors`
- `Unit: QueryRequest with FilterExpression round-trips through protobuf`

---

## Phase 2: Vector Engine Core — HNSW Index and Storage

### Purpose

Implement the heart of the system: the HNSW graph-based ANN index, distance metric computations, write-ahead log, and segment-based vector storage. After this phase, the engine can store vectors, build HNSW indexes, and execute nearest-neighbor queries locally via unit tests. No network layer yet.

### Tasks

#### 2.1 — Distance Metric Implementations

**What**: Implement SIMD-accelerated cosine, Euclidean, and dot-product distance functions.

**Design**:

```rust
// engine/src/hnsw/distance.rs

/// Cosine distance: 1 - (a . b) / (||a|| * ||b||)
/// Returns 0.0 for identical vectors, 2.0 for opposite vectors.
pub fn cosine_distance(a: &[f32], b: &[f32]) -> f32 {
    debug_assert_eq!(a.len(), b.len());
    let mut dot = 0.0f32;
    let mut norm_a = 0.0f32;
    let mut norm_b = 0.0f32;
    for i in 0..a.len() {
        dot += a[i] * b[i];
        norm_a += a[i] * a[i];
        norm_b += b[i] * b[i];
    }
    let denom = norm_a.sqrt() * norm_b.sqrt();
    if denom == 0.0 { return 1.0; }
    1.0 - (dot / denom)
}

/// Squared Euclidean distance: sum((a_i - b_i)^2)
pub fn euclidean_distance(a: &[f32], b: &[f32]) -> f32 {
    debug_assert_eq!(a.len(), b.len());
    a.iter().zip(b.iter())
        .map(|(x, y)| (x - y).powi(2))
        .sum()
}

/// Dot product: sum(a_i * b_i). Higher = more similar.
pub fn dot_product(a: &[f32], b: &[f32]) -> f32 {
    debug_assert_eq!(a.len(), b.len());
    a.iter().zip(b.iter())
        .map(|(x, y)| x * y)
        .sum()
}
```

**Testing**:
- `Unit: cosine_distance on identical unit vectors returns ~0.0`
- `Unit: cosine_distance on orthogonal vectors returns ~1.0`
- `Unit: cosine_distance on opposite vectors returns ~2.0`
- `Unit: cosine_distance on zero vector returns 1.0 (not NaN)`
- `Unit: euclidean_distance on identical vectors returns 0.0`
- `Unit: euclidean_distance([1,0,0], [0,1,0]) returns 2.0`
- `Unit: dot_product on orthogonal vectors returns 0.0`
- `Unit: dot_product([1,2,3], [4,5,6]) returns 32.0`
- `Benchmark: cosine_distance on 1536-dim vectors (criterion) — baseline for future SIMD optimization`

#### 2.2 — HNSW Graph Construction

**What**: Implement the HNSW insert algorithm (Malkov & Yashunin, 2018) with configurable M and ef_construction parameters.

**Design**:

```rust
// engine/src/hnsw/graph.rs
use std::collections::BinaryHeap;

/// HNSW graph parameters matching the algorithm specification.
#[derive(Debug, Clone)]
pub struct HnswParams {
    /// Max number of connections per node per layer (M in the paper).
    pub m: usize,           // default: 16
    /// Max connections for layer 0 (2 * M).
    pub m_max0: usize,      // default: 32
    /// Size of dynamic candidate list during construction.
    pub ef_construction: usize, // default: 200
    /// Normalization factor for layer assignment: 1/ln(M).
    pub ml: f64,
}

impl Default for HnswParams {
    fn default() -> Self {
        let m = 16;
        Self {
            m,
            m_max0: m * 2,
            ef_construction: 200,
            ml: 1.0 / (m as f64).ln(),
        }
    }
}

/// A node in the HNSW graph.
pub struct HnswNode {
    pub id: String,
    pub vector: Vec<f32>,
    pub payload: serde_json::Value,
    /// Connections per layer. connections[layer] = Vec<node_index>
    pub connections: Vec<Vec<usize>>,
    /// Maximum layer this node exists on.
    pub max_layer: usize,
}

/// The HNSW graph index.
pub struct HnswIndex {
    pub params: HnswParams,
    pub distance_metric: DistanceMetric,
    pub dimension: usize,
    pub nodes: Vec<HnswNode>,
    /// Index of the entry point node.
    pub entry_point: Option<usize>,
    /// Maximum layer in the graph.
    pub max_level: usize,
}

impl HnswIndex {
    pub fn new(dimension: usize, distance_metric: DistanceMetric, params: HnswParams) -> Self;

    /// Insert a vector into the index. Returns the assigned internal index.
    /// Algorithm: INSERT from Malkov & Yashunin (2018), Section 4.
    pub fn insert(&mut self, id: String, vector: Vec<f32>, payload: serde_json::Value) -> usize;

    /// Search for the top_k nearest neighbors.
    /// Algorithm: SEARCH-LAYER with greedy traversal from top to layer 1,
    /// then ef-bounded search on layer 0.
    pub fn search(
        &self,
        query: &[f32],
        top_k: usize,
        ef_search: usize,
    ) -> Vec<SearchResult>;

    /// Search with payload filtering. Filter is applied post-retrieval with
    /// oversampling to maintain recall.
    pub fn search_with_filter(
        &self,
        query: &[f32],
        top_k: usize,
        ef_search: usize,
        filter: &dyn Fn(&serde_json::Value) -> bool,
    ) -> Vec<SearchResult>;

    /// Select the random layer for a new node: floor(-ln(uniform(0,1)) * ml)
    fn random_level(&self) -> usize;

    /// Find ef nearest neighbors on a given layer (SEARCH-LAYER algorithm).
    fn search_layer(
        &self,
        query: &[f32],
        entry_points: &[usize],
        ef: usize,
        layer: usize,
    ) -> BinaryHeap<Candidate>;

    /// Select M neighbors using the heuristic from the paper (SELECT-NEIGHBORS-HEURISTIC).
    fn select_neighbors_heuristic(
        &self,
        query: &[f32],
        candidates: BinaryHeap<Candidate>,
        m: usize,
    ) -> Vec<usize>;
}

pub struct SearchResult {
    pub id: String,
    pub score: f32,
    pub payload: Option<serde_json::Value>,
}

struct Candidate {
    pub index: usize,
    pub distance: f32,
}
```

**Testing**:
- `Unit: insert 1000 random 128-dim vectors, search returns exact nearest neighbor in top-1 with >99% probability`
- `Unit: insert 5000 vectors, recall@10 >= 0.95 with default params (ef_search=100)`
- `Unit: empty index search returns empty results`
- `Unit: single-vector index search returns that vector as top-1`
- `Unit: random_level() distribution follows geometric with parameter ml`
- `Unit: search with top_k > vector_count returns all vectors`
- `Unit: insert with different dimensions panics (debug_assert)`
- `Benchmark: insert 100K 768-dim vectors — measure build time`
- `Benchmark: search 1000 queries against 100K vectors — measure QPS and recall@10`

#### 2.3 — Write-Ahead Log (WAL)

**What**: Implement an append-only WAL for durability of upsert and delete operations before they are committed to segments.

**Design**:

```rust
// engine/src/storage/wal.rs
use std::fs::File;
use std::io::{BufWriter, Write};
use std::path::PathBuf;

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub enum WalEntry {
    Upsert {
        collection_id: String,
        namespace: String,
        id: String,
        vector: Vec<f32>,
        metadata: serde_json::Value,
        timestamp: u64,
    },
    Delete {
        collection_id: String,
        namespace: String,
        ids: Vec<String>,
        timestamp: u64,
    },
}

pub struct WriteAheadLog {
    dir: PathBuf,
    current_segment: File,
    current_segment_id: u64,
    segment_size_bytes: u64,
    current_size: u64,
    writer: BufWriter<File>,
}

impl WriteAheadLog {
    /// Open or create a WAL in the given directory.
    pub fn open(dir: PathBuf, segment_size_bytes: u64) -> std::io::Result<Self>;

    /// Append an entry and flush (fsync) for durability.
    pub fn append(&mut self, entry: &WalEntry) -> std::io::Result<u64>; // returns LSN

    /// Rotate to a new segment file when current exceeds segment_size_bytes.
    fn rotate_segment(&mut self) -> std::io::Result<()>;

    /// Read all entries from a segment file (for replay during recovery).
    pub fn read_segment(path: &PathBuf) -> std::io::Result<Vec<(u64, WalEntry)>>;

    /// Truncate WAL entries up to and including the given LSN (after segment flush).
    pub fn truncate_before(&mut self, lsn: u64) -> std::io::Result<()>;
}
```

**Testing**:
- `Unit: append entry and read_segment returns the same entry`
- `Unit: append 1000 entries, read_segment returns all in order`
- `Unit: segment rotation occurs when size exceeds segment_size_bytes`
- `Unit: truncate_before removes old segments, preserves newer ones`
- `Integration: crash-recovery simulation — write entries, truncate writer mid-entry, recover valid prefix`
- `Benchmark: append 100K entries — measure throughput (entries/sec)`

#### 2.4 — Segment-Based Vector Storage

**What**: Implement immutable segment files that store vectors, metadata, and the serialized HNSW graph for a collection.

**Design**:

```rust
// engine/src/storage/segment.rs

/// A segment is an immutable file containing vectors and their HNSW graph.
/// New upserts go to the WAL and an in-memory buffer; periodically the buffer
/// is flushed to a new segment.
pub struct Segment {
    pub id: String,
    pub collection_id: String,
    pub vector_count: u64,
    pub dimension: u32,
    pub distance_metric: DistanceMetric,
    pub hnsw_index: HnswIndex,
    pub created_at: u64,
    pub size_bytes: u64,
}

/// Manages the lifecycle of segments for a single collection.
pub struct CollectionStorage {
    collection_id: String,
    dimension: u32,
    distance_metric: DistanceMetric,
    /// In-memory buffer for recent upserts (not yet flushed to segment).
    write_buffer: HnswIndex,
    /// Immutable flushed segments (searched in parallel).
    segments: Vec<Segment>,
    /// WAL for durability.
    wal: WriteAheadLog,
    /// Flush threshold: number of vectors in write_buffer before creating a new segment.
    flush_threshold: usize, // default: 50_000
}

impl CollectionStorage {
    pub fn open(
        collection_id: String,
        dimension: u32,
        distance_metric: DistanceMetric,
        data_dir: PathBuf,
    ) -> std::io::Result<Self>;

    pub fn upsert(&mut self, id: String, vector: Vec<f32>, metadata: serde_json::Value)
        -> Result<(), StorageError>;

    pub fn delete(&mut self, ids: &[String]) -> Result<u64, StorageError>;

    pub fn search(
        &self,
        query: &[f32],
        top_k: usize,
        ef_search: usize,
        filter: Option<&dyn Fn(&serde_json::Value) -> bool>,
    ) -> Vec<SearchResult>;

    pub fn count(&self) -> u64;

    /// Flush the write buffer to a new immutable segment.
    pub fn flush(&mut self) -> Result<(), StorageError>;

    /// Merge multiple small segments into one larger segment (compaction).
    pub fn compact(&mut self) -> Result<(), StorageError>;

    /// Create a snapshot of all segments to the snapshot directory.
    pub fn create_snapshot(&self, snapshot_dir: &PathBuf) -> Result<SnapshotInfo, StorageError>;

    /// Restore from a snapshot.
    pub fn restore_snapshot(snapshot_path: &PathBuf, data_dir: &PathBuf)
        -> Result<Self, StorageError>;
}

pub struct SnapshotInfo {
    pub snapshot_id: String,
    pub collection_id: String,
    pub vector_count: u64,
    pub segment_count: u32,
    pub size_bytes: u64,
    pub created_at: u64,
}
```

**Testing**:
- `Unit: upsert 100 vectors, count() returns 100`
- `Unit: upsert then search returns the upserted vector`
- `Unit: delete a vector, search no longer returns it`
- `Unit: upsert beyond flush_threshold triggers automatic segment creation`
- `Unit: search spans both write_buffer and flushed segments, merges results correctly`
- `Integration: upsert 60K vectors (triggering flush), verify segment file exists on disk`
- `Integration: create_snapshot produces valid snapshot, restore_snapshot recovers identical state`
- `Integration: compact merges 3 small segments into 1, search results unchanged`

---

## Phase 3: Engine Network Layer — gRPC and REST APIs

### Purpose

Expose the vector engine's capabilities over the network via gRPC (for control plane communication) and REST (for direct client access). After this phase, the engine is a running server that accepts upsert, query, and delete requests over the wire.

### Tasks

#### 3.1 — gRPC Service Implementation

**What**: Implement the tonic gRPC server matching the proto definitions from Phase 1.

**Design**:

```rust
// engine/src/server/grpc.rs
use tonic::{Request, Response, Status};

pub struct VectorServiceImpl {
    /// Map of collection_id -> CollectionStorage
    collections: Arc<RwLock<HashMap<String, CollectionStorage>>>,
    config: EngineConfig,
}

#[tonic::async_trait]
impl VectorService for VectorServiceImpl {
    async fn upsert(
        &self,
        request: Request<UpsertRequest>,
    ) -> Result<Response<UpsertResponse>, Status> {
        let req = request.into_inner();
        let collections = self.collections.read().await;
        let storage = collections.get(&req.collection_id)
            .ok_or_else(|| Status::not_found("Collection not found"))?;
        // ... upsert vectors
    }

    async fn query(
        &self,
        request: Request<QueryRequest>,
    ) -> Result<Response<QueryResponse>, Status> {
        let req = request.into_inner();
        // Validate dimension matches collection
        // Build filter closure from FilterExpression
        // Execute search on CollectionStorage
        // Return scored results
    }

    async fn delete(
        &self,
        request: Request<DeleteRequest>,
    ) -> Result<Response<DeleteResponse>, Status> {
        // Delete by IDs or by filter
    }

    async fn count(
        &self,
        request: Request<CountRequest>,
    ) -> Result<Response<CountResponse>, Status> {
        // Return vector count for collection
    }
}

pub struct CollectionServiceImpl {
    collections: Arc<RwLock<HashMap<String, CollectionStorage>>>,
    config: EngineConfig,
}

#[tonic::async_trait]
impl CollectionService for CollectionServiceImpl {
    async fn create_collection(
        &self,
        request: Request<CreateCollectionRequest>,
    ) -> Result<Response<CreateCollectionResponse>, Status>;

    async fn delete_collection(
        &self,
        request: Request<DeleteCollectionRequest>,
    ) -> Result<Response<DeleteCollectionResponse>, Status>;

    async fn get_collection_info(
        &self,
        request: Request<GetCollectionInfoRequest>,
    ) -> Result<Response<CollectionInfo>, Status>;

    async fn build_index(
        &self,
        request: Request<BuildIndexRequest>,
    ) -> Result<Response<BuildIndexResponse>, Status>;

    async fn create_snapshot(
        &self,
        request: Request<CreateSnapshotRequest>,
    ) -> Result<Response<CreateSnapshotResponse>, Status>;
}
```

**Testing**:
- `Integration: gRPC CreateCollection then GetCollectionInfo returns correct dimension and metric`
- `Integration: gRPC Upsert 100 vectors then Query returns correct top-k results`
- `Integration: gRPC Query on non-existent collection returns NOT_FOUND status`
- `Integration: gRPC Upsert with wrong dimension returns INVALID_ARGUMENT status`
- `Integration: gRPC Delete by IDs, subsequent Query no longer returns deleted vectors`
- `Integration: gRPC Count returns accurate vector count after upserts and deletes`
- `Integration: gRPC BuildIndex triggers async index build, status transitions from building to ready`
- `Load test: 100 concurrent gRPC Query requests — verify no deadlocks or panics`

#### 3.2 — REST API via Axum

**What**: Implement a lightweight REST API on the engine for health checks, metrics, and direct vector operations (alternative to gRPC for simpler clients).

**Design**:

```rust
// engine/src/server/http.rs
use axum::{Router, Json, extract::State};

pub fn create_router(
    collections: Arc<RwLock<HashMap<String, CollectionStorage>>>,
) -> Router {
    Router::new()
        .route("/health", get(health_check))
        .route("/metrics", get(prometheus_metrics))
        .route("/collections", post(create_collection))
        .route("/collections/:id", get(get_collection).delete(delete_collection))
        .route("/collections/:id/vectors/upsert", post(upsert_vectors))
        .route("/collections/:id/vectors/query", post(query_vectors))
        .route("/collections/:id/vectors/delete", post(delete_vectors))
        .route("/collections/:id/count", get(count_vectors))
        .with_state(collections)
}

// Health check response
#[derive(serde::Serialize)]
struct HealthResponse {
    status: String,    // "serving"
    version: String,   // env!("CARGO_PKG_VERSION")
    uptime_seconds: u64,
    total_vectors: u64,
    collection_count: u32,
}

// REST query request body (mirrors gRPC but JSON)
#[derive(serde::Deserialize)]
struct QueryBody {
    vector: Vec<f32>,
    #[serde(default)]
    sparse_vector: Option<SparseVectorBody>,
    top_k: u32,
    #[serde(default)]
    filter: Option<serde_json::Value>,
    #[serde(default)]
    include_values: bool,
    #[serde(default)]
    include_metadata: bool,
    #[serde(default)]
    namespace: String,
    #[serde(default)]
    alpha: Option<f32>,
}
```

**Testing**:
- `Integration: GET /health returns 200 with status "serving"`
- `Integration: POST /collections creates collection, returns 201`
- `Integration: POST upsert then POST query returns matching results as JSON`
- `Integration: POST query with invalid dimension returns 400 with error message`
- `Integration: GET /metrics returns Prometheus-format text`
- `Integration: DELETE /collections/:id removes collection, subsequent GET returns 404`

#### 3.3 — Metadata Filter Parser and Evaluator

**What**: Implement the filter expression language for payload-based filtering during search (matching the Pinecone-style JSON filter syntax).

**Design**:

```rust
// engine/src/filter/parser.rs

/// Filter expression DSL matching Pinecone's filter language:
/// {"category": {"$eq": "electronics"}}
/// {"price": {"$gt": 10, "$lt": 100}}
/// {"$and": [{"category": {"$eq": "electronics"}}, {"price": {"$lt": 50}}]}

#[derive(Debug, Clone)]
pub enum FilterExpr {
    Field {
        field: String,
        operator: FilterOp,
        value: FilterValue,
    },
    And(Vec<FilterExpr>),
    Or(Vec<FilterExpr>),
}

#[derive(Debug, Clone)]
pub enum FilterOp {
    Eq,   // $eq
    Ne,   // $ne
    Gt,   // $gt
    Gte,  // $gte
    Lt,   // $lt
    Lte,  // $lte
    In,   // $in
    Nin,  // $nin
}

#[derive(Debug, Clone)]
pub enum FilterValue {
    String(String),
    Int(i64),
    Float(f64),
    Bool(bool),
    Array(Vec<FilterValue>),
}

/// Parse a JSON filter expression into FilterExpr.
pub fn parse_filter(json: &serde_json::Value) -> Result<FilterExpr, FilterError>;

// engine/src/filter/evaluator.rs

/// Evaluate a FilterExpr against a metadata payload.
/// Returns true if the payload matches the filter.
pub fn evaluate(expr: &FilterExpr, payload: &serde_json::Value) -> bool;
```

**Testing**:
- `Unit: parse {"category": {"$eq": "electronics"}} -> Field(category, Eq, String("electronics"))`
- `Unit: parse {"$and": [...]} -> And([...])`
- `Unit: parse invalid operator {"field": {"$invalid": 1}} -> FilterError`
- `Unit: evaluate $eq match returns true`
- `Unit: evaluate $eq non-match returns false`
- `Unit: evaluate $gt on numeric field`
- `Unit: evaluate $in with array of values`
- `Unit: evaluate nested $and with two conditions`
- `Unit: evaluate $or with one matching condition returns true`
- `Unit: evaluate against payload missing the filter field returns false`
- `Integration: search_with_filter only returns vectors matching the filter`

---

## Phase 4: Control Plane — Database, Auth, and Collection Management API

### Purpose

Build the Python control plane that manages tenants, authentication, RBAC, and collection lifecycle. This is the customer-facing REST API layer. After this phase, users can create accounts, generate API keys, create collections, and manage their resources through the REST API.

### Tasks

#### 4.1 — Database Schema and Migrations

**What**: Implement the PostgreSQL schema based on Data Model Suggestion 3 (Hybrid Relational + JSONB) using SQLAlchemy ORM and Alembic migrations.

**Design**:

```python
# api/src/vectrus_api/db/models.py
from sqlalchemy import Column, String, Integer, BigInteger, Boolean, DateTime, ForeignKey, CheckConstraint
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import DeclarativeBase, relationship
from datetime import datetime
import uuid

class Base(DeclarativeBase):
    pass

class Organization(Base):
    __tablename__ = "organizations"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False)
    slug = Column(String(100), nullable=False, unique=True)
    status = Column(
        String(20), nullable=False, default="active",
        info={"check": CheckConstraint("status IN ('active', 'suspended', 'cancelled')")}
    )
    plan_tier = Column(
        String(20), nullable=False, default="free",
        info={"check": CheckConstraint("plan_tier IN ('free', 'starter', 'pro', 'enterprise')")}
    )
    billing = Column(JSONB, nullable=False, default=dict)
    plan_limits = Column(JSONB, nullable=False, default=dict)
    data_governance = Column(JSONB, nullable=False, default=dict)
    auth_config = Column(JSONB, nullable=False, default=dict)
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)
    updated_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)

    members = relationship("OrganizationMember", back_populates="organization")
    collections = relationship("Collection", back_populates="organization")
    api_keys = relationship("ApiKey", back_populates="organization")


class User(Base):
    __tablename__ = "users"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email = Column(String(255), nullable=False, unique=True)
    display_name = Column(String(255))
    password_hash = Column(String(255))
    auth = Column(JSONB, nullable=False, default=dict)
    preferences = Column(JSONB, nullable=False, default=dict)
    last_login_at = Column(DateTime(timezone=True))
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)
    updated_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)

    memberships = relationship("OrganizationMember", back_populates="user")


class OrganizationMember(Base):
    __tablename__ = "organization_members"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organization_id = Column(UUID(as_uuid=True), ForeignKey("organizations.id", ondelete="CASCADE"), nullable=False)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    role = Column(String(30), nullable=False, default="member")
    permissions = Column(JSONB, nullable=False, default=dict)
    invited_by = Column(UUID(as_uuid=True), ForeignKey("users.id"))
    accepted_at = Column(DateTime(timezone=True))
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)

    organization = relationship("Organization", back_populates="members")
    user = relationship("User", back_populates="memberships", foreign_keys=[user_id])


class Collection(Base):
    __tablename__ = "collections"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organization_id = Column(UUID(as_uuid=True), ForeignKey("organizations.id", ondelete="CASCADE"), nullable=False)
    project_slug = Column(String(100))
    name = Column(String(255), nullable=False)
    status = Column(String(20), nullable=False, default="creating")
    dimension = Column(Integer, nullable=False)
    distance_metric = Column(String(30), nullable=False, default="cosine")
    vector_count = Column(BigInteger, nullable=False, default=0)
    storage_bytes = Column(BigInteger, nullable=False, default=0)
    deployment = Column(JSONB, nullable=False, default=dict)
    index_config = Column(JSONB, nullable=False, default=dict)
    payload_schema = Column(JSONB, nullable=False, default=dict)
    embedding_config = Column(JSONB)
    hybrid_config = Column(JSONB)
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)
    updated_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)

    organization = relationship("Organization", back_populates="collections")
    namespaces = relationship("Namespace", back_populates="collection")


class Namespace(Base):
    __tablename__ = "namespaces"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    collection_id = Column(UUID(as_uuid=True), ForeignKey("collections.id", ondelete="CASCADE"), nullable=False)
    name = Column(String(255), nullable=False)
    vector_count = Column(BigInteger, nullable=False, default=0)
    metadata = Column(JSONB, default=dict)
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)

    collection = relationship("Collection", back_populates="namespaces")


class ApiKey(Base):
    __tablename__ = "api_keys"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organization_id = Column(UUID(as_uuid=True), ForeignKey("organizations.id", ondelete="CASCADE"), nullable=False)
    name = Column(String(255), nullable=False)
    key_prefix = Column(String(8), nullable=False)
    key_hash = Column(String(255), nullable=False)
    role = Column(String(30), nullable=False, default="member")
    config = Column(JSONB, nullable=False, default=dict)
    last_used_at = Column(DateTime(timezone=True))
    revoked_at = Column(DateTime(timezone=True))
    created_by = Column(UUID(as_uuid=True), ForeignKey("users.id"))
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)

    organization = relationship("Organization", back_populates="api_keys")


class UsageRecord(Base):
    __tablename__ = "usage_records"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organization_id = Column(UUID(as_uuid=True), ForeignKey("organizations.id"), nullable=False)
    collection_id = Column(UUID(as_uuid=True), ForeignKey("collections.id"))
    metric_type = Column(String(30), nullable=False)
    quantity = Column(BigInteger, nullable=False)
    details = Column(JSONB, default=dict)
    recorded_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)


class AuditLog(Base):
    __tablename__ = "audit_logs"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organization_id = Column(UUID(as_uuid=True), nullable=False)
    actor_type = Column(String(20), nullable=False)
    actor_id = Column(UUID(as_uuid=True))
    action = Column(String(100), nullable=False)
    resource_type = Column(String(50), nullable=False)
    resource_id = Column(UUID(as_uuid=True))
    context = Column(JSONB, nullable=False, default=dict)
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)


class Integration(Base):
    __tablename__ = "integrations"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organization_id = Column(UUID(as_uuid=True), ForeignKey("organizations.id", ondelete="CASCADE"), nullable=False)
    integration_type = Column(String(30), nullable=False)
    config = Column(JSONB, nullable=False, default=dict)
    is_active = Column(Boolean, nullable=False, default=True)
    created_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)
    updated_at = Column(DateTime(timezone=True), nullable=False, default=datetime.utcnow)
```

**Testing**:
- `Unit: Alembic migration generates all tables without errors`
- `Unit: Alembic upgrade head + downgrade base round-trips cleanly`
- `Integration: insert Organization and query by slug`
- `Integration: insert Collection with JSONB index_config and query with @> containment`
- `Integration: cascade delete Organization removes all child records`

#### 4.2 — Authentication and API Key Management

**What**: Implement API key creation, hashing (Argon2), validation middleware, and JWT-based dashboard session authentication.

**Design**:

```python
# api/src/vectrus_api/services/auth_service.py
import secrets
import argon2
from datetime import datetime, timedelta
import jwt

class AuthService:
    """Handles API key and JWT authentication."""

    KEY_PREFIX_LENGTH = 8
    KEY_BYTES = 32  # 256-bit random key

    def __init__(self, settings: Settings, db_session):
        self.settings = settings
        self.db = db_session
        self.hasher = argon2.PasswordHasher()

    async def create_api_key(
        self,
        organization_id: uuid.UUID,
        name: str,
        role: str = "member",
        scopes: list[str] | None = None,
        rate_limit_rps: int = 100,
        expires_at: datetime | None = None,
        created_by: uuid.UUID | None = None,
    ) -> tuple[ApiKey, str]:
        """Create an API key. Returns the DB record and the raw key (shown once)."""
        raw_key = f"vdb_{secrets.token_urlsafe(self.KEY_BYTES)}"
        key_prefix = raw_key[:self.KEY_PREFIX_LENGTH]
        key_hash = self.hasher.hash(raw_key)
        config = {
            "scopes": scopes or ["collections:read", "collections:write", "query"],
            "rate_limit_rps": rate_limit_rps,
        }
        if expires_at:
            config["expires_at"] = expires_at.isoformat()
        api_key = ApiKey(
            organization_id=organization_id,
            name=name,
            key_prefix=key_prefix,
            key_hash=key_hash,
            role=role,
            config=config,
            created_by=created_by,
        )
        self.db.add(api_key)
        await self.db.commit()
        return api_key, raw_key

    async def validate_api_key(self, raw_key: str) -> ApiKey | None:
        """Validate an API key. Returns the ApiKey record or None."""
        prefix = raw_key[:self.KEY_PREFIX_LENGTH]
        # Query by prefix (indexed), then verify hash
        candidates = await self.db.execute(
            select(ApiKey).where(
                ApiKey.key_prefix == prefix,
                ApiKey.revoked_at.is_(None),
            )
        )
        for candidate in candidates.scalars():
            try:
                self.hasher.verify(candidate.key_hash, raw_key)
                # Check expiry
                expires = candidate.config.get("expires_at")
                if expires and datetime.fromisoformat(expires) < datetime.utcnow():
                    return None
                # Update last_used_at
                candidate.last_used_at = datetime.utcnow()
                await self.db.commit()
                return candidate
            except argon2.exceptions.VerifyMismatchError:
                continue
        return None

    def create_jwt(self, user_id: uuid.UUID, org_id: uuid.UUID) -> str:
        """Create a JWT for dashboard session authentication."""
        payload = {
            "sub": str(user_id),
            "org": str(org_id),
            "exp": datetime.utcnow() + timedelta(minutes=self.settings.jwt_expiry_minutes),
            "iat": datetime.utcnow(),
        }
        return jwt.encode(payload, self.settings.jwt_secret_key, algorithm=self.settings.jwt_algorithm)


# api/src/vectrus_api/middleware/auth.py
from fastapi import Request, HTTPException
from fastapi.security import HTTPBearer

class ApiKeyAuth:
    """FastAPI dependency that extracts and validates the API key from the Authorization header."""

    async def __call__(self, request: Request) -> ApiKey:
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            raise HTTPException(status_code=401, detail="Missing or invalid Authorization header")
        raw_key = auth_header[7:]
        auth_service = request.app.state.auth_service
        api_key = await auth_service.validate_api_key(raw_key)
        if not api_key:
            raise HTTPException(status_code=401, detail="Invalid or expired API key")
        return api_key
```

**Testing**:
- `Unit: create_api_key returns raw key starting with "vdb_" and DB record with hashed key`
- `Unit: validate_api_key with correct key returns the ApiKey record`
- `Unit: validate_api_key with wrong key returns None`
- `Unit: validate_api_key with expired key returns None`
- `Unit: validate_api_key with revoked key returns None`
- `Unit: create_jwt returns valid JWT decodable with the secret key`
- `Integration: POST request with valid API key header passes auth middleware`
- `Integration: POST request without Authorization header returns 401`
- `Integration: POST request with invalid key returns 401`
- `Unit: API key prefix indexing — validate_api_key only queries candidates by prefix`

#### 4.3 — Collection Management REST API

**What**: Implement the /v1/collections endpoints for creating, listing, describing, and deleting collections — the primary resource management API.

**Design**:

```python
# api/src/vectrus_api/routes/collections.py
from fastapi import APIRouter, Depends, HTTPException, Query
from pydantic import BaseModel, Field
from uuid import UUID

router = APIRouter(prefix="/v1/collections", tags=["collections"])

class CreateCollectionRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255, pattern=r"^[a-zA-Z0-9_-]+$")
    dimension: int = Field(..., gt=0, le=65536)
    distance_metric: str = Field(default="cosine", pattern=r"^(cosine|euclidean|dot_product)$")
    index_config: dict = Field(default_factory=lambda: {
        "type": "hnsw",
        "params": {"m": 16, "ef_construction": 200, "ef_search": 100},
        "quantization": {"type": "none"},
    })
    payload_schema: dict = Field(default_factory=dict)
    region: str = Field(default="us-east-1")

class CollectionResponse(BaseModel):
    id: UUID
    name: str
    dimension: int
    distance_metric: str
    status: str
    vector_count: int
    storage_bytes: int
    index_config: dict
    created_at: str

class CollectionListResponse(BaseModel):
    collections: list[CollectionResponse]
    total: int

@router.post("", status_code=201, response_model=CollectionResponse)
async def create_collection(
    body: CreateCollectionRequest,
    api_key: ApiKey = Depends(ApiKeyAuth()),
    service: CollectionService = Depends(get_collection_service),
):
    """Create a new vector collection."""
    # Check plan limits (max_collections)
    # Create collection record in control plane DB
    # Send gRPC CreateCollection to engine
    # Return collection metadata

@router.get("", response_model=CollectionListResponse)
async def list_collections(
    api_key: ApiKey = Depends(ApiKeyAuth()),
    limit: int = Query(default=20, ge=1, le=100),
    offset: int = Query(default=0, ge=0),
):
    """List all collections in the organization."""

@router.get("/{collection_id}", response_model=CollectionResponse)
async def get_collection(
    collection_id: UUID,
    api_key: ApiKey = Depends(ApiKeyAuth()),
):
    """Get details of a specific collection."""

@router.delete("/{collection_id}", status_code=204)
async def delete_collection(
    collection_id: UUID,
    api_key: ApiKey = Depends(ApiKeyAuth()),
):
    """Delete a collection and all its vectors."""
    # Mark as 'deleting' in DB
    # Send gRPC DeleteCollection to engine
    # Remove DB record after engine confirms
```

**Testing**:
- `Integration (mocked engine): POST /v1/collections with valid body returns 201 and collection metadata`
- `Integration (mocked engine): POST /v1/collections with duplicate name returns 409`
- `Integration (mocked engine): POST /v1/collections with dimension 0 returns 422`
- `Integration (mocked engine): POST /v1/collections with invalid name characters returns 422`
- `Integration (mocked engine): GET /v1/collections returns paginated list`
- `Integration (mocked engine): GET /v1/collections/{id} returns collection details`
- `Integration (mocked engine): GET /v1/collections/{non-existent-id} returns 404`
- `Integration (mocked engine): DELETE /v1/collections/{id} returns 204`
- `Integration (mocked engine): DELETE /v1/collections/{id} idempotent (second delete returns 404)`

#### 4.4 — Vector Operations REST API

**What**: Implement the /v1/collections/{id}/vectors and /v1/query endpoints for upserting, querying, and deleting vectors.

**Design**:

```python
# api/src/vectrus_api/routes/vectors.py
router = APIRouter(prefix="/v1/collections/{collection_id}", tags=["vectors"])

class VectorInput(BaseModel):
    id: str = Field(..., min_length=1, max_length=512)
    values: list[float]
    metadata: dict = Field(default_factory=dict)
    sparse_values: dict | None = Field(default=None)
    # sparse_values: {"indices": [1, 5, 100], "values": [0.5, 0.3, 0.8]}

class UpsertRequest(BaseModel):
    vectors: list[VectorInput] = Field(..., min_length=1, max_length=1000)
    namespace: str = Field(default="")

class UpsertResponse(BaseModel):
    upserted_count: int

class QueryRequest(BaseModel):
    vector: list[float]
    sparse_vector: dict | None = None
    top_k: int = Field(default=10, ge=1, le=10000)
    filter: dict | None = None
    include_values: bool = False
    include_metadata: bool = True
    namespace: str = ""
    alpha: float | None = Field(default=None, ge=0.0, le=1.0)

class ScoredVector(BaseModel):
    id: str
    score: float
    values: list[float] | None = None
    metadata: dict | None = None

class QueryResponse(BaseModel):
    results: list[ScoredVector]
    namespace: str

class DeleteRequest(BaseModel):
    ids: list[str] | None = None
    filter: dict | None = None
    namespace: str = ""

class DeleteResponse(BaseModel):
    deleted_count: int

@router.post("/vectors/upsert", response_model=UpsertResponse)
async def upsert_vectors(
    collection_id: UUID,
    body: UpsertRequest,
    api_key: ApiKey = Depends(ApiKeyAuth()),
):
    """Upsert vectors into a collection."""
    # Validate dimension matches collection
    # Forward to engine via gRPC
    # Record usage (write_units)
    # Return count

@router.post("/vectors/query", response_model=QueryResponse)
async def query_vectors(
    collection_id: UUID,
    body: QueryRequest,
    api_key: ApiKey = Depends(ApiKeyAuth()),
):
    """Query vectors by similarity search."""
    # Validate dimension
    # Forward to engine via gRPC
    # Record usage (read_units, query_count)
    # Return scored results

@router.post("/vectors/delete", response_model=DeleteResponse)
async def delete_vectors(
    collection_id: UUID,
    body: DeleteRequest,
    api_key: ApiKey = Depends(ApiKeyAuth()),
):
    """Delete vectors by IDs or filter."""
```

**Testing**:
- `Integration (mocked engine): POST upsert with 10 vectors returns upserted_count=10`
- `Integration (mocked engine): POST upsert with wrong dimension returns 400`
- `Integration (mocked engine): POST upsert with >1000 vectors returns 422`
- `Integration (mocked engine): POST query returns ranked results with scores`
- `Integration (mocked engine): POST query with filter passes filter to engine`
- `Integration (mocked engine): POST query with include_metadata=false omits metadata`
- `Integration (mocked engine): POST delete by IDs returns deleted_count`
- `Integration (mocked engine): POST delete with filter forwards filter to engine`
- `E2E (real engine): upsert 50 vectors, query returns correct nearest neighbors`

#### 4.5 — Rate Limiting Middleware

**What**: Implement Redis-backed sliding window rate limiting per API key.

**Design**:

```python
# api/src/vectrus_api/middleware/rate_limit.py
import redis.asyncio as redis
from fastapi import Request, HTTPException
import time

class RateLimiter:
    """Sliding window rate limiter backed by Redis."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def check_rate_limit(self, api_key_id: str, limit_rps: int) -> bool:
        """
        Returns True if the request is allowed, False if rate-limited.
        Uses a Redis sorted set with timestamps as scores.
        Window: 1 second.
        """
        key = f"ratelimit:{api_key_id}"
        now = time.time()
        window_start = now - 1.0

        pipe = self.redis.pipeline()
        # Remove entries outside the window
        pipe.zremrangebyscore(key, 0, window_start)
        # Count current entries
        pipe.zcard(key)
        # Add new entry
        pipe.zadd(key, {f"{now}:{id(object())}": now})
        # Set TTL
        pipe.expire(key, 2)
        results = await pipe.execute()

        current_count = results[1]
        return current_count < limit_rps


class RateLimitMiddleware:
    """FastAPI middleware that enforces rate limits per API key."""

    async def __call__(self, request: Request, call_next):
        api_key = request.state.api_key  # Set by auth middleware
        limit = api_key.config.get("rate_limit_rps", 100)
        limiter = request.app.state.rate_limiter

        if not await limiter.check_rate_limit(str(api_key.id), limit):
            raise HTTPException(
                status_code=429,
                detail="Rate limit exceeded",
                headers={"Retry-After": "1"},
            )
        return await call_next(request)
```

**Testing**:
- `Unit (mocked Redis): 1 request within limit returns allowed`
- `Unit (mocked Redis): 101 requests within 1 second at limit=100 — last request rejected`
- `Integration (real Redis): burst of 50 requests at limit=100 all succeed`
- `Integration (real Redis): burst of 150 requests at limit=100 — first 100 succeed, rest return 429`
- `Unit: rate limit headers include Retry-After: 1`

---

## Phase 5: Usage Metering, Billing, and Audit Logging

### Purpose

Implement usage tracking, billing plan enforcement, invoice generation, and audit logging. After this phase, every API call is metered, organizations have plan limits enforced, and all mutations are audit-logged for SOC 2 compliance.

### Tasks

#### 5.1 — Usage Recording Service

**What**: Record usage metrics (read_units, write_units, query_count, storage_bytes) for every API call, stored in the partitioned usage_records table.

**Design**:

```python
# api/src/vectrus_api/services/billing_service.py

class UsageService:
    """Records and queries usage metrics."""

    UNIT_COSTS = {
        "read_units": 1,      # 1 read unit per query
        "write_units": 1,     # 1 write unit per upserted vector
    }

    def __init__(self, db_session, redis_client):
        self.db = db_session
        self.redis = redis_client

    async def record_usage(
        self,
        organization_id: uuid.UUID,
        collection_id: uuid.UUID | None,
        metric_type: str,
        quantity: int,
        details: dict | None = None,
    ) -> None:
        """Record a usage event. Buffered in Redis, flushed to DB periodically."""
        record = UsageRecord(
            organization_id=organization_id,
            collection_id=collection_id,
            metric_type=metric_type,
            quantity=quantity,
            details=details or {},
        )
        self.db.add(record)
        # Also increment Redis counter for real-time plan limit checks
        key = f"usage:{organization_id}:{metric_type}:{self._current_period()}"
        await self.redis.incrby(key, quantity)
        await self.redis.expire(key, 86400 * 35)  # 35 days TTL

    async def get_usage_summary(
        self,
        organization_id: uuid.UUID,
        period_start: datetime,
        period_end: datetime,
    ) -> dict[str, int]:
        """Get aggregated usage for a billing period."""
        result = await self.db.execute(
            select(
                UsageRecord.metric_type,
                func.sum(UsageRecord.quantity).label("total"),
            )
            .where(
                UsageRecord.organization_id == organization_id,
                UsageRecord.recorded_at >= period_start,
                UsageRecord.recorded_at < period_end,
            )
            .group_by(UsageRecord.metric_type)
        )
        return {row.metric_type: row.total for row in result}

    async def check_plan_limits(
        self,
        organization_id: uuid.UUID,
        metric_type: str,
        quantity: int,
    ) -> bool:
        """Check if the organization is within plan limits. Uses Redis for speed."""
        org = await self.db.get(Organization, organization_id)
        limit = org.plan_limits.get(f"included_{metric_type}")
        if limit is None:
            return True  # No limit configured
        key = f"usage:{organization_id}:{metric_type}:{self._current_period()}"
        current = int(await self.redis.get(key) or 0)
        return (current + quantity) <= limit
```

**Testing**:
- `Unit: record_usage creates UsageRecord in DB`
- `Unit: record_usage increments Redis counter`
- `Unit: get_usage_summary aggregates correctly across multiple records`
- `Unit: check_plan_limits returns True when under limit`
- `Unit: check_plan_limits returns False when over limit`
- `Integration: record 1000 usage events, get_usage_summary returns correct totals`

#### 5.2 — Audit Logging

**What**: Record all control plane mutations to the audit_logs table for SOC 2 Type II compliance.

**Design**:

```python
# api/src/vectrus_api/services/audit_service.py

class AuditService:
    """Records audit events for all control plane mutations."""

    async def log(
        self,
        organization_id: uuid.UUID,
        actor_type: str,       # "user", "api_key", "system"
        actor_id: uuid.UUID | None,
        action: str,           # e.g., "collection.created", "api_key.revoked"
        resource_type: str,    # e.g., "collection", "api_key", "organization"
        resource_id: uuid.UUID | None,
        context: dict | None = None,
        request: Request | None = None,
    ) -> None:
        ctx = context or {}
        if request:
            ctx["ip_address"] = request.client.host
            ctx["user_agent"] = request.headers.get("user-agent", "")
            ctx["request_id"] = request.headers.get("x-request-id", "")
        log_entry = AuditLog(
            organization_id=organization_id,
            actor_type=actor_type,
            actor_id=actor_id,
            action=action,
            resource_type=resource_type,
            resource_id=resource_id,
            context=ctx,
        )
        self.db.add(log_entry)
        await self.db.commit()

    async def query_logs(
        self,
        organization_id: uuid.UUID,
        action: str | None = None,
        resource_type: str | None = None,
        since: datetime | None = None,
        limit: int = 50,
    ) -> list[AuditLog]:
        """Query audit logs with optional filters."""
        # Build query with filters
        # Return ordered by created_at DESC
```

**Testing**:
- `Unit: log creates AuditLog record with correct fields`
- `Unit: log extracts IP address and user-agent from Request`
- `Unit: query_logs filters by action and resource_type`
- `Unit: query_logs filters by time range`
- `Integration: create collection via API, audit_logs contains "collection.created" entry`
- `Integration: delete API key via API, audit_logs contains "api_key.revoked" entry`

#### 5.3 — Invoice Generation

**What**: Implement monthly invoice generation from usage records, plan base prices, and overage calculations.

**Design**:

```python
# api/src/vectrus_api/services/invoice_service.py
from decimal import Decimal

OVERAGE_RATES = {
    "read_units": Decimal("0.004"),     # $0.004 per 1K read units
    "write_units": Decimal("0.008"),    # $0.008 per 1K write units
    "storage_bytes": Decimal("0.20"),   # $0.20 per GB/month
}

PLAN_BASE_PRICES = {
    "free": 0,
    "starter": 2500,      # $25.00
    "pro": 9900,           # $99.00
    "enterprise": None,    # Custom
}

class InvoiceService:
    async def generate_invoice(
        self,
        organization_id: uuid.UUID,
        billing_period_start: date,
        billing_period_end: date,
    ) -> Invoice:
        """Generate a monthly invoice for an organization."""
        org = await self.db.get(Organization, organization_id)
        usage = await self.usage_service.get_usage_summary(
            organization_id,
            datetime.combine(billing_period_start, time.min),
            datetime.combine(billing_period_end, time.min),
        )
        base_price = PLAN_BASE_PRICES.get(org.plan_tier, 0)
        line_items = [{"description": f"{org.plan_tier.title()} plan", "amount_cents": base_price}]
        overage_total = 0
        for metric, total in usage.items():
            included = org.plan_limits.get(f"included_{metric}", 0)
            overage = max(0, total - included)
            if overage > 0 and metric in OVERAGE_RATES:
                rate = OVERAGE_RATES[metric]
                amount = int(overage * rate)
                line_items.append({
                    "description": f"{metric} overage ({overage:,})",
                    "amount_cents": amount,
                    "quantity": overage,
                })
                overage_total += amount
        invoice = Invoice(
            organization_id=organization_id,
            billing_period_start=billing_period_start,
            billing_period_end=billing_period_end,
            status="draft",
            line_items=line_items,
            total_amount_cents=base_price + overage_total,
        )
        self.db.add(invoice)
        await self.db.commit()
        return invoice
```

**Testing**:
- `Unit: free tier with zero usage generates invoice with total_amount_cents=0`
- `Unit: pro tier with usage under limits generates invoice with base price only`
- `Unit: pro tier with read_units overage includes overage line item`
- `Unit: enterprise tier with custom limits calculates overage correctly`
- `Integration: generate invoice for period with usage records, verify line items match`

---

## Phase 6: Observability — Metrics, Tracing, and Dashboards

### Purpose

Instrument both the engine and control plane with Prometheus metrics, OpenTelemetry tracing, and Grafana dashboard templates. After this phase, operators have full visibility into query latency, throughput, index build progress, and system health.

### Tasks

#### 6.1 — Engine Prometheus Metrics

**What**: Expose Prometheus metrics from the Rust engine covering vector operations, index status, and system resources.

**Design**:

```rust
// engine/src/server/metrics.rs
use prometheus::{
    Registry, IntCounter, IntGauge, Histogram, HistogramOpts, IntCounterVec, Opts,
};
use lazy_static::lazy_static;

lazy_static! {
    pub static ref REGISTRY: Registry = Registry::new();

    // Query metrics
    pub static ref QUERY_DURATION: Histogram = Histogram::with_opts(
        HistogramOpts::new("vectrus_query_duration_seconds", "Query latency in seconds")
            .buckets(vec![0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0])
    ).unwrap();
    pub static ref QUERY_TOTAL: IntCounterVec = IntCounterVec::new(
        Opts::new("vectrus_query_total", "Total number of queries"),
        &["collection_id", "status"],
    ).unwrap();
    pub static ref QUERY_RESULTS_COUNT: Histogram = Histogram::with_opts(
        HistogramOpts::new("vectrus_query_results_count", "Number of results returned per query")
            .buckets(vec![1.0, 5.0, 10.0, 20.0, 50.0, 100.0])
    ).unwrap();

    // Upsert metrics
    pub static ref UPSERT_TOTAL: IntCounter = IntCounter::new(
        "vectrus_upsert_total", "Total vectors upserted"
    ).unwrap();
    pub static ref UPSERT_DURATION: Histogram = Histogram::with_opts(
        HistogramOpts::new("vectrus_upsert_duration_seconds", "Upsert batch latency")
            .buckets(vec![0.01, 0.05, 0.1, 0.5, 1.0, 5.0, 10.0])
    ).unwrap();

    // Storage metrics
    pub static ref VECTOR_COUNT: IntGauge = IntGauge::new(
        "vectrus_vector_count", "Total number of stored vectors"
    ).unwrap();
    pub static ref STORAGE_BYTES: IntGauge = IntGauge::new(
        "vectrus_storage_bytes", "Total storage used in bytes"
    ).unwrap();
    pub static ref SEGMENT_COUNT: IntGauge = IntGauge::new(
        "vectrus_segment_count", "Number of segments"
    ).unwrap();

    // Index metrics
    pub static ref INDEX_BUILD_DURATION: Histogram = Histogram::with_opts(
        HistogramOpts::new("vectrus_index_build_duration_seconds", "Index build duration")
            .buckets(vec![1.0, 5.0, 10.0, 30.0, 60.0, 300.0, 600.0])
    ).unwrap();
}
```

**Testing**:
- `Integration: after upserting vectors, GET /metrics returns vectrus_upsert_total > 0`
- `Integration: after querying, GET /metrics returns vectrus_query_duration_seconds histogram`
- `Integration: vectrus_vector_count matches actual vector count`
- `Unit: all metrics register without conflicts in the registry`

#### 6.2 — Control Plane OpenTelemetry Instrumentation

**What**: Add OpenTelemetry tracing to FastAPI requests, database queries, Redis calls, and gRPC calls to the engine.

**Design**:

```python
# api/src/vectrus_api/middleware/telemetry.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.grpc import GrpcInstrumentorClient

def setup_telemetry(app, settings: Settings):
    """Configure OpenTelemetry tracing for the control plane."""
    if not settings.otel_exporter_endpoint:
        return

    provider = TracerProvider()
    exporter = OTLPSpanExporter(endpoint=settings.otel_exporter_endpoint)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    FastAPIInstrumentor.instrument_app(app)
    SQLAlchemyInstrumentor().instrument()
    RedisInstrumentor().instrument()
    GrpcInstrumentorClient().instrument()
```

**Testing**:
- `Unit: setup_telemetry with None endpoint does not crash`
- `Integration: API request with OTEL configured produces trace spans`
- `Integration: trace spans include database, redis, and gRPC child spans`

#### 6.3 — Grafana Dashboard Templates

**What**: Create pre-built Grafana dashboard JSON files for engine and API metrics.

**Design**:

Two dashboard JSON files:
- `grafana/dashboards/engine-metrics.json`: Vector count, query latency (p50/p95/p99), QPS, upsert throughput, segment count, storage usage, index build status.
- `grafana/dashboards/api-metrics.json`: HTTP request rate by endpoint, error rate (4xx/5xx), auth failures, rate limit rejections, usage by organization.

Each dashboard uses Prometheus as the data source with template variables for collection_id and time range.

**Testing**:
- `Unit: each dashboard JSON parses as valid Grafana dashboard model`
- `Integration: import dashboard JSON into Grafana API, verify no import errors`
- `Manual: docker compose stack with Grafana, verify dashboards display data`

---

## Phase 7: Quantization and Hybrid Search

### Purpose

Add memory-efficient quantization strategies (scalar, binary, product) and hybrid search combining dense HNSW with sparse BM25/SPLADE vectors. These are differentiating features identified in features.md that move beyond the MVP.

### Tasks

#### 7.1 — Scalar Quantization

**What**: Implement int8 scalar quantization for HNSW vectors, reducing memory by ~75% while maintaining >98% recall.

**Design**:

```rust
// engine/src/quantization/scalar.rs

/// Scalar quantization: map f32 vectors to u8 (0-255) using min-max scaling.
pub struct ScalarQuantizer {
    pub dimension: usize,
    pub min_values: Vec<f32>,  // per-dimension minimum
    pub max_values: Vec<f32>,  // per-dimension maximum
    pub scale: Vec<f32>,       // per-dimension scale factor
}

impl ScalarQuantizer {
    /// Train the quantizer from a sample of vectors.
    /// Computes per-dimension min/max for the scaling range.
    pub fn train(vectors: &[&[f32]], dimension: usize) -> Self;

    /// Quantize a float vector to u8.
    pub fn quantize(&self, vector: &[f32]) -> Vec<u8>;

    /// Dequantize a u8 vector back to approximate f32.
    pub fn dequantize(&self, quantized: &[u8]) -> Vec<f32>;

    /// Compute approximate distance between a query (f32) and a quantized vector (u8).
    /// Uses asymmetric computation: query stays in f32, candidate is dequantized.
    pub fn distance_asymmetric(
        &self,
        query: &[f32],
        quantized: &[u8],
        metric: DistanceMetric,
    ) -> f32;
}
```

**Testing**:
- `Unit: train on 1000 random vectors computes correct min/max per dimension`
- `Unit: quantize then dequantize preserves vector within +-0.01 per dimension`
- `Unit: distance_asymmetric matches f32 distance within 5% relative error`
- `Benchmark: quantized search on 100K vectors — measure recall@10 vs unquantized (target >98%)`
- `Benchmark: quantized search memory usage vs unquantized (target ~25%)`

#### 7.2 — Binary and Product Quantization

**What**: Implement binary quantization (1-bit, ~32x compression) and product quantization (configurable code size).

**Design**:

```rust
// engine/src/quantization/binary.rs

/// Binary quantization: map each f32 component to 1 bit (sign bit).
/// Distance computed via Hamming distance on packed u64 words.
pub struct BinaryQuantizer {
    pub dimension: usize,
    pub packed_words: usize,  // ceil(dimension / 64)
}

impl BinaryQuantizer {
    pub fn new(dimension: usize) -> Self;
    pub fn quantize(&self, vector: &[f32]) -> Vec<u64>;
    pub fn hamming_distance(&self, a: &[u64], b: &[u64]) -> u32;
}

// engine/src/quantization/product.rs

/// Product quantization: split vector into sub-vectors, quantize each with a codebook.
pub struct ProductQuantizer {
    pub dimension: usize,
    pub num_subquantizers: usize, // e.g., 8
    pub code_size: usize,         // bits per sub-code, e.g., 8 (256 centroids)
    pub codebooks: Vec<Vec<Vec<f32>>>, // [subquantizer][centroid][sub_dimension]
}

impl ProductQuantizer {
    /// Train codebooks using k-means on sub-vectors.
    pub fn train(vectors: &[&[f32]], num_subquantizers: usize, code_size: usize) -> Self;
    pub fn quantize(&self, vector: &[f32]) -> Vec<u8>;
    pub fn distance_table(&self, query: &[f32], metric: DistanceMetric) -> Vec<Vec<f32>>;
    pub fn distance_from_table(&self, codes: &[u8], table: &Vec<Vec<f32>>) -> f32;
}
```

**Testing**:
- `Unit: binary quantize and hamming_distance on identical vectors returns 0`
- `Unit: binary quantize on opposite-sign vectors returns max distance`
- `Unit: PQ train on 10K vectors with 8 subquantizers converges`
- `Unit: PQ quantize then distance_from_table approximates true distance within 15%`
- `Benchmark: binary search recall@10 on 100K vectors (target >90%)`
- `Benchmark: PQ search recall@10 on 100K vectors (target >95%)`

#### 7.3 — Sparse Vector Index for Hybrid Search

**What**: Implement an inverted index for sparse vectors (BM25/SPLADE) to enable hybrid dense+sparse retrieval.

**Design**:

```rust
// engine/src/sparse/inverted_index.rs

/// Inverted index for sparse vector retrieval.
/// Each sparse vector is a set of (term_index, weight) pairs.
pub struct InvertedIndex {
    /// posting_lists[term_index] = Vec<(doc_id, weight)>
    posting_lists: HashMap<u32, Vec<(usize, f32)>>,
    /// Document norms for scoring normalization.
    doc_norms: Vec<f32>,
    pub doc_count: usize,
}

impl InvertedIndex {
    pub fn new() -> Self;
    pub fn insert(&mut self, doc_id: usize, sparse_vector: &[(u32, f32)]);
    pub fn delete(&mut self, doc_id: usize);

    /// Score documents by sparse vector similarity (dot product on matching terms).
    pub fn search(&self, query: &[(u32, f32)], top_k: usize) -> Vec<(usize, f32)>;
}

/// Hybrid search: combine dense HNSW results with sparse inverted index results.
pub fn hybrid_search(
    hnsw: &HnswIndex,
    sparse_index: &InvertedIndex,
    dense_query: &[f32],
    sparse_query: &[(u32, f32)],
    top_k: usize,
    ef_search: usize,
    alpha: f32,          // 0.0 = sparse only, 1.0 = dense only
    fusion: FusionMethod,
) -> Vec<SearchResult>;

pub enum FusionMethod {
    /// Reciprocal Rank Fusion: score = sum(1 / (k + rank_i)) for each result list
    ReciprocalRankFusion { k: u32 }, // default k=60
    /// Convex combination: score = alpha * dense_score + (1 - alpha) * sparse_score
    ConvexCombination,
}
```

**Testing**:
- `Unit: insert 100 sparse vectors, search returns matching documents`
- `Unit: hybrid_search with alpha=1.0 returns same results as pure dense search`
- `Unit: hybrid_search with alpha=0.0 returns same results as pure sparse search`
- `Unit: hybrid_search with RRF fusion combines results from both sources`
- `Unit: hybrid_search with ConvexCombination blends scores correctly`
- `Integration: end-to-end hybrid upsert and query through gRPC`

---

## Phase 8: Web Dashboard

### Purpose

Build the React web dashboard for collection management, a query playground, API key administration, and usage analytics. After this phase, users have a visual interface for all platform operations alongside the API.

### Tasks

#### 8.1 — Dashboard Foundation and Authentication

**What**: Set up the React application with routing, JWT-based authentication flow, and layout components.

**Design**:

```typescript
// dashboard/src/lib/auth.ts
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
}

interface User {
  id: string;
  email: string;
  display_name: string;
  organization: {
    id: string;
    name: string;
    slug: string;
    plan_tier: string;
  };
}

interface LoginRequest {
  email: string;
  password: string;
}

interface LoginResponse {
  token: string;
  user: User;
}

// Auth context provider that stores JWT in localStorage
// and attaches it to all API requests via the api-client.

// dashboard/src/lib/api-client.ts
// Generated from OpenAPI spec using openapi-typescript-codegen
// Base URL: /v1
// Auth: Bearer token (JWT for dashboard, API key for programmatic access)
```

Route structure:
- `/login` — Login page
- `/` — Dashboard overview (redirect if not authenticated)
- `/collections` — Collection list
- `/collections/:id` — Collection detail
- `/collections/:id/query` — Query playground for a specific collection
- `/api-keys` — API key management
- `/usage` — Usage analytics
- `/settings` — Organization settings

**Testing**:
- `Unit: AuthProvider stores and clears JWT on login/logout`
- `Unit: api-client attaches Bearer token to requests`
- `Unit: unauthenticated routes redirect to /login`
- `E2E (Playwright): login with valid credentials, see dashboard`
- `E2E (Playwright): login with invalid credentials, see error message`

#### 8.2 — Collection Management UI

**What**: Build the collection list page with create/delete actions and the collection detail page showing configuration, vector count, and index status.

**Design**:

```typescript
// dashboard/src/components/collections/CollectionList.tsx
interface CollectionRow {
  id: string;
  name: string;
  dimension: number;
  distance_metric: string;
  status: "creating" | "ready" | "indexing" | "error" | "deleting";
  vector_count: number;
  storage_bytes: number;
  created_at: string;
}

// Uses TanStack Table for sortable, paginated table
// Create button opens modal with form:
//   - name (text input, validated: ^[a-zA-Z0-9_-]+$)
//   - dimension (number input, 1-65536)
//   - distance_metric (select: cosine, euclidean, dot_product)
//   - index_config.type (select: hnsw, flat)
//   - HNSW params (collapsible advanced section): m, ef_construction

// dashboard/src/components/collections/CollectionDetail.tsx
// Shows: status badge, dimension, metric, vector count, storage
// Tabs: Overview | Index Config (JSON viewer) | Payload Schema | Namespaces
```

**Testing**:
- `Unit: CollectionList renders table with correct columns`
- `Unit: CreateCollectionForm validates name pattern`
- `Unit: CreateCollectionForm validates dimension range`
- `E2E (Playwright): create collection through form, see it appear in list`
- `E2E (Playwright): delete collection, confirm dialog, see it removed from list`

#### 8.3 — Query Playground

**What**: Build an interactive query playground where users can paste a vector, configure search parameters, and see results with scores and metadata.

**Design**:

```typescript
// dashboard/src/components/query-playground/QueryPlayground.tsx
interface QueryPlaygroundState {
  collectionId: string;
  vector: string;        // JSON array of floats (user pastes)
  topK: number;          // slider, 1-100
  filter: string;        // JSON editor for filter expression
  includeMetadata: boolean;
  includeValues: boolean;
  namespace: string;
  alpha: number | null;  // slider for hybrid search weight
}

interface QueryResult {
  id: string;
  score: number;
  metadata: Record<string, unknown> | null;
  values: number[] | null;
}

// Layout:
// Left panel: query input (vector JSON, filter JSON, params)
// Right panel: results list with expandable metadata
// Bottom: query latency, result count, request/response JSON
```

**Testing**:
- `Unit: QueryPlayground validates vector JSON before submit`
- `Unit: QueryPlayground displays results with scores`
- `Unit: QueryPlayground shows raw request/response JSON`
- `E2E (Playwright): enter vector, submit query, see results with scores`
- `E2E (Playwright): add filter, verify results change`

#### 8.4 — Usage Analytics Dashboard

**What**: Build the usage analytics page showing query volume, write volume, storage growth, and cost projections with time-series charts.

**Design**:

```typescript
// dashboard/src/pages/Usage.tsx
// Time range selector: 7d | 30d | 90d | custom
// Metrics cards (top): total queries, total writes, storage used, estimated cost
// Charts (Recharts):
//   - Query volume over time (bar chart, hourly/daily granularity)
//   - Write volume over time (bar chart)
//   - Storage growth (area chart)
//   - Latency percentiles (line chart: p50, p95, p99)
// Table: per-collection breakdown of usage

interface UsageSummary {
  period: { start: string; end: string };
  metrics: {
    query_count: number;
    read_units: number;
    write_units: number;
    storage_bytes: number;
    inference_tokens: number;
  };
  estimated_cost_cents: number;
}

interface UsageTimeSeries {
  buckets: Array<{
    timestamp: string;
    query_count: number;
    write_count: number;
    storage_bytes: number;
  }>;
}
```

**Testing**:
- `Unit: Usage page renders metrics cards with formatted numbers`
- `Unit: time range selector updates chart data`
- `Unit: per-collection table sorts by usage column`
- `E2E (Playwright): navigate to usage page, verify charts render with data`

---

## Phase 9: SDKs and MCP Server

### Purpose

Generate and publish Python and TypeScript client SDKs from the OpenAPI spec, and build an MCP server for AI agent integration. After this phase, developers can integrate with Vectrus using idiomatic SDK methods, and AI agents can use Vectrus as a retrieval tool.

### Tasks

#### 9.1 — Python SDK

**What**: Auto-generate a Python SDK from the OpenAPI 3.1 spec and add hand-tuned convenience methods.

**Design**:

```python
# sdks/python/src/vectrus/client.py
from httpx import Client, AsyncClient
from typing import Any

class VectrusClient:
    """Synchronous Vectrus client."""

    def __init__(
        self,
        api_key: str,
        base_url: str = "https://api.vectrus.io",
        timeout: float = 30.0,
    ):
        self.api_key = api_key
        self._client = Client(
            base_url=base_url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=timeout,
        )

    def create_collection(
        self,
        name: str,
        dimension: int,
        distance_metric: str = "cosine",
        **kwargs: Any,
    ) -> dict:
        """Create a new vector collection."""
        resp = self._client.post("/v1/collections", json={
            "name": name,
            "dimension": dimension,
            "distance_metric": distance_metric,
            **kwargs,
        })
        resp.raise_for_status()
        return resp.json()

    def upsert(
        self,
        collection_id: str,
        vectors: list[dict],
        namespace: str = "",
    ) -> dict:
        """Upsert vectors into a collection."""
        resp = self._client.post(
            f"/v1/collections/{collection_id}/vectors/upsert",
            json={"vectors": vectors, "namespace": namespace},
        )
        resp.raise_for_status()
        return resp.json()

    def query(
        self,
        collection_id: str,
        vector: list[float],
        top_k: int = 10,
        filter: dict | None = None,
        namespace: str = "",
        include_metadata: bool = True,
        include_values: bool = False,
    ) -> dict:
        """Query a collection by vector similarity."""
        body = {
            "vector": vector,
            "top_k": top_k,
            "namespace": namespace,
            "include_metadata": include_metadata,
            "include_values": include_values,
        }
        if filter:
            body["filter"] = filter
        resp = self._client.post(
            f"/v1/collections/{collection_id}/vectors/query",
            json=body,
        )
        resp.raise_for_status()
        return resp.json()

    def delete(
        self,
        collection_id: str,
        ids: list[str] | None = None,
        filter: dict | None = None,
        namespace: str = "",
    ) -> dict:
        """Delete vectors from a collection."""
        body: dict[str, Any] = {"namespace": namespace}
        if ids:
            body["ids"] = ids
        if filter:
            body["filter"] = filter
        resp = self._client.post(
            f"/v1/collections/{collection_id}/vectors/delete",
            json=body,
        )
        resp.raise_for_status()
        return resp.json()

    def list_collections(self) -> list[dict]:
        resp = self._client.get("/v1/collections")
        resp.raise_for_status()
        return resp.json()["collections"]

    def close(self):
        self._client.close()

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self.close()
```

**Testing**:
- `Unit (mocked httpx): VectrusClient.create_collection sends correct POST body`
- `Unit (mocked httpx): VectrusClient.upsert sends vectors in correct format`
- `Unit (mocked httpx): VectrusClient.query sends filter in request body`
- `Unit (mocked httpx): VectrusClient raises httpx.HTTPStatusError on 4xx/5xx`
- `Integration (real API): full workflow — create collection, upsert, query, delete`
- `Unit: context manager usage closes client`

#### 9.2 — TypeScript SDK

**What**: Auto-generate a TypeScript SDK from the OpenAPI spec with hand-tuned types.

**Design**:

```typescript
// sdks/typescript/src/client.ts
interface VectrusConfig {
  apiKey: string;
  baseUrl?: string;  // default: "https://api.vectrus.io"
  timeout?: number;  // default: 30000
}

interface VectorInput {
  id: string;
  values: number[];
  metadata?: Record<string, unknown>;
  sparseValues?: { indices: number[]; values: number[] };
}

interface QueryOptions {
  vector: number[];
  topK?: number;
  filter?: Record<string, unknown>;
  namespace?: string;
  includeMetadata?: boolean;
  includeValues?: boolean;
  alpha?: number;
}

interface ScoredVector {
  id: string;
  score: number;
  values?: number[];
  metadata?: Record<string, unknown>;
}

class VectrusClient {
  constructor(config: VectrusConfig);
  async createCollection(params: CreateCollectionParams): Promise<Collection>;
  async listCollections(): Promise<Collection[]>;
  async getCollection(id: string): Promise<Collection>;
  async deleteCollection(id: string): Promise<void>;
  async upsert(collectionId: string, vectors: VectorInput[], namespace?: string): Promise<{ upsertedCount: number }>;
  async query(collectionId: string, options: QueryOptions): Promise<{ results: ScoredVector[] }>;
  async deleteVectors(collectionId: string, ids?: string[], filter?: Record<string, unknown>): Promise<{ deletedCount: number }>;
}

export { VectrusClient, VectrusConfig, VectorInput, QueryOptions, ScoredVector };
```

**Testing**:
- `Unit (mocked fetch): createCollection sends correct request`
- `Unit (mocked fetch): query passes filter in body`
- `Unit (mocked fetch): HTTP errors throw VectrusError with status code`
- `Unit: TypeScript types compile without errors`
- `Integration (real API): full workflow — create, upsert, query, delete`

#### 9.3 — MCP Server for AI Agent Integration

**What**: Build a Model Context Protocol server that exposes Vectrus search, upsert, and collection management as tools for AI agents.

**Design**:

```python
# mcp-server/src/vectrus_mcp/tools.py
from mcp.server import Server
from mcp.types import Tool, TextContent

TOOLS = [
    Tool(
        name="vectrus_search",
        description="Search a Vectrus vector collection by similarity. Returns the top-k most similar vectors with their metadata.",
        inputSchema={
            "type": "object",
            "properties": {
                "collection_id": {"type": "string", "description": "Collection ID to search"},
                "query_text": {"type": "string", "description": "Text to search for (will be embedded automatically)"},
                "top_k": {"type": "integer", "default": 10, "description": "Number of results to return"},
                "filter": {"type": "object", "description": "Metadata filter expression"},
                "namespace": {"type": "string", "default": "", "description": "Namespace to search in"},
            },
            "required": ["collection_id", "query_text"],
        },
    ),
    Tool(
        name="vectrus_upsert",
        description="Store text with metadata in a Vectrus collection. Text is automatically embedded.",
        inputSchema={
            "type": "object",
            "properties": {
                "collection_id": {"type": "string"},
                "documents": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "id": {"type": "string"},
                            "text": {"type": "string"},
                            "metadata": {"type": "object"},
                        },
                        "required": ["id", "text"],
                    },
                },
                "namespace": {"type": "string", "default": ""},
            },
            "required": ["collection_id", "documents"],
        },
    ),
    Tool(
        name="vectrus_list_collections",
        description="List all vector collections in the organization.",
        inputSchema={"type": "object", "properties": {}},
    ),
    Tool(
        name="vectrus_describe_collection",
        description="Get details about a specific vector collection.",
        inputSchema={
            "type": "object",
            "properties": {
                "collection_id": {"type": "string"},
            },
            "required": ["collection_id"],
        },
    ),
]
```

**Testing**:
- `Unit: MCP server lists all 4 tools`
- `Unit: vectrus_search tool calls Vectrus query API and returns formatted results`
- `Unit: vectrus_upsert tool embeds text and calls upsert API`
- `Integration: MCP server connects to running Vectrus API, search returns results`

---

## Phase 10: AI-Native Features

### Purpose

Implement the AI-native advantages that differentiate Vectrus from incumbents: automated index tuning, embedding drift detection, LLM-assisted schema design, and natural language query translation. These are the features identified in research.md as the primary innovation opportunity.

### Tasks

#### 10.1 — Automated Index Parameter Tuning

**What**: Analyze observed query patterns (latency, recall, filter usage) and recommend optimal HNSW parameters (M, ef_construction, ef_search, quantization).

**Design**:

```python
# api/src/vectrus_api/ai/index_tuner.py
from pydantic import BaseModel

class QueryPatternStats(BaseModel):
    query_count_7d: int
    avg_latency_ms: float
    p99_latency_ms: float
    avg_top_k: float
    filter_usage_pct: float
    top_filters: list[str]
    recall_estimate: float      # estimated from re-scoring sample queries
    memory_usage_bytes: int
    vector_count: int
    dimension: int

class TuningRecommendation(BaseModel):
    current: dict       # {"m": 16, "ef_construction": 200, ...}
    recommended: dict   # {"m": 24, "ef_construction": 256, ...}
    expected_improvement: dict  # {"latency_reduction_pct": 15, "memory_change_pct": -40}
    reasoning: str      # LLM-generated explanation
    confidence: float   # 0.0-1.0

class IndexTuner:
    """AI-driven index parameter optimizer."""

    def __init__(self, llm_client, engine_client):
        self.llm = llm_client
        self.engine = engine_client

    async def analyze_and_recommend(
        self,
        collection_id: str,
        stats: QueryPatternStats,
        current_config: dict,
    ) -> TuningRecommendation:
        """
        1. Compute heuristic recommendations based on stats:
           - High filter_usage_pct (>50%) -> increase M for better graph connectivity
           - High p99 latency -> increase ef_search or enable quantization
           - Low recall estimate -> increase ef_construction
           - Large vector count with high memory -> recommend scalar quantization
        2. Use LLM to synthesize reasoning and validate recommendations.
        3. Return structured recommendation.
        """

    async def run_tuning(
        self,
        collection_id: str,
    ) -> IndexTuningRun:
        """Collect stats, generate recommendation, store as IndexTuningRun."""
        stats = await self._collect_query_stats(collection_id)
        current_config = await self._get_current_config(collection_id)
        recommendation = await self.analyze_and_recommend(
            collection_id, stats, current_config
        )
        # Store in index_tuning_runs table
        run = IndexTuningRun(
            collection_id=collection_id,
            status="completed",
            input_stats=stats.model_dump(),
            recommendations=recommendation.model_dump(),
        )
        self.db.add(run)
        await self.db.commit()
        return run
```

**Testing**:
- `Unit: high filter_usage_pct triggers M increase recommendation`
- `Unit: high p99 latency triggers ef_search increase or quantization`
- `Unit: large vector count with scalar quantization recommends keeping quantization`
- `Unit: recommendation includes non-empty reasoning string`
- `Integration (mocked LLM): full tuning run produces valid IndexTuningRun record`

#### 10.2 — Embedding Distribution Drift Detection

**What**: Monitor embedding distributions over time and alert when upstream model changes or data distribution shifts degrade retrieval quality.

**Design**:

```python
# api/src/vectrus_api/ai/drift_detector.py
import numpy as np

class DriftDetector:
    """Detects embedding distribution drift between time periods."""

    DRIFT_THRESHOLD = 0.10  # configurable per collection

    async def detect_drift(
        self,
        collection_id: str,
        baseline_start: datetime,
        baseline_end: datetime,
        current_start: datetime,
        current_end: datetime,
        sample_size: int = 1000,
    ) -> DriftResult:
        """
        1. Sample `sample_size` vectors from baseline and current periods.
        2. Compute per-dimension statistics:
           - Mean shift per dimension
           - Variance change per dimension
        3. Compute overall drift score using Maximum Mean Discrepancy (MMD)
           or Kolmogorov-Smirnov test per dimension.
        4. Flag dimensions with statistically significant shifts.
        5. Generate alert severity (info, warning, critical) based on drift_score vs threshold.
        """

    def _compute_mmd(
        self,
        sample_a: np.ndarray,  # (n, d)
        sample_b: np.ndarray,  # (m, d)
        kernel: str = "rbf",
    ) -> float:
        """Maximum Mean Discrepancy between two sample distributions."""

    def _identify_drifted_dimensions(
        self,
        sample_a: np.ndarray,
        sample_b: np.ndarray,
        significance: float = 0.05,
    ) -> list[int]:
        """Per-dimension KS test, returns indices of significantly drifted dimensions."""

class DriftResult(BaseModel):
    baseline_period: dict
    current_period: dict
    drift_score: float
    drift_threshold: float
    drift_detected: bool
    affected_dimensions: list[int]
    probable_cause: str  # "upstream_model_update", "data_distribution_shift", "unknown"
    alert_severity: str  # "info", "warning", "critical"
    recommendation: str
```

**Testing**:
- `Unit: identical distributions produce drift_score near 0.0`
- `Unit: significantly shifted distributions produce drift_score > threshold`
- `Unit: _identify_drifted_dimensions returns correct indices for shifted dimensions`
- `Unit: alert_severity is "warning" when drift_score between 1x and 2x threshold`
- `Unit: alert_severity is "critical" when drift_score > 2x threshold`
- `Integration: Celery task runs drift detection on schedule and stores DriftDetectionRun`

#### 10.3 — LLM-Assisted Payload Schema Design

**What**: Analyze sample documents and suggest optimal payload schema (filterable metadata fields, data types, indexes) for a collection.

**Design**:

```python
# api/src/vectrus_api/ai/schema_advisor.py

class SchemaAdvisor:
    """Uses LLM to recommend payload schema based on sample documents."""

    SYSTEM_PROMPT = """You are a vector database schema advisor. Given sample documents
    that will be stored as vector metadata, recommend:
    1. Which fields should be extracted as filterable payload fields
    2. The data type for each field (string, integer, float, boolean, datetime, geo, string_array)
    3. Which fields should be indexed for filter performance
    4. Which fields are required vs optional

    Output valid JSON matching the payload_schema format."""

    async def suggest_schema(
        self,
        sample_documents: list[dict],
        collection_purpose: str,
        max_fields: int = 20,
    ) -> PayloadSchemaRecommendation:
        """
        1. Send sample_documents + collection_purpose to LLM.
        2. Parse structured output into PayloadSchemaRecommendation.
        3. Validate field types against allowed types.
        4. Return recommendation with explanation.
        """

class PayloadSchemaRecommendation(BaseModel):
    fields: dict[str, FieldRecommendation]
    reasoning: str
    strict_mode: bool

class FieldRecommendation(BaseModel):
    type: str       # string, integer, float, boolean, datetime, geo, string_array
    indexed: bool
    required: bool
    description: str
```

**Testing**:
- `Unit (mocked LLM): sample e-commerce products suggest "category" (string, indexed), "price" (float, indexed)`
- `Unit (mocked LLM): sample news articles suggest "published_date" (datetime, indexed), "tags" (string_array, indexed)`
- `Unit: invalid field type in LLM response is rejected`
- `Unit: output validates against PayloadSchemaRecommendation model`
- `Integration: schema advisor API endpoint returns valid recommendation`

#### 10.4 — Natural Language Query Translation

**What**: Translate plain-English retrieval intent into optimized vector + filter query plans.

**Design**:

```python
# api/src/vectrus_api/ai/nl_query.py

class NLQueryTranslator:
    """Translates natural language queries into vector search + filter plans."""

    SYSTEM_PROMPT = """You are a vector database query planner. Given a natural language
    query and the collection's payload schema, produce:
    1. A reformulated text query for embedding (optimized for vector similarity)
    2. Filter expressions to apply (using the collection's indexed fields)
    3. Recommended top_k and any hybrid search parameters

    Output valid JSON matching the QueryPlan schema."""

    async def translate(
        self,
        natural_query: str,
        collection_id: str,
        payload_schema: dict,
    ) -> QueryPlan:
        """
        1. Fetch collection metadata (dimension, distance_metric, payload_schema).
        2. Send natural_query + schema to LLM.
        3. Parse QueryPlan from response.
        4. Embed the reformulated query text using the collection's embedding config.
        5. Return ready-to-execute QueryPlan.
        """

class QueryPlan(BaseModel):
    reformulated_query: str
    embedding_vector: list[float] | None  # filled after embedding
    filter: dict | None
    top_k: int
    use_hybrid: bool
    alpha: float | None
    reasoning: str
```

**Testing**:
- `Unit (mocked LLM): "find cheap electronics" -> filter: {"category": {"$eq": "electronics"}, "price": {"$lt": 50}}`
- `Unit (mocked LLM): "recent news about AI" -> filter: {"published_date": {"$gte": "2026-01-01"}}`
- `Unit: QueryPlan validates correctly`
- `Unit: empty natural_query returns error`
- `Integration (mocked LLM + real embedding): full translation produces executable query`

---

## Phase 11: Self-Hosted Deployment — Helm Chart and Documentation

### Purpose

Package the entire platform as a Helm chart for Kubernetes deployment, with production-grade configuration, TLS, and operational documentation. This addresses the key differentiator vs. Pinecone and Turbopuffer: self-hosted deployment for regulated industries.

### Tasks

#### 11.1 — Helm Chart

**What**: Create a comprehensive Helm chart with configurable replicas, resource limits, persistence, TLS, and Ingress.

**Design**:

```yaml
# helm/vectrus/values.yaml
global:
  imageRegistry: ""
  imagePullSecrets: []

engine:
  replicaCount: 1
  image:
    repository: vectrus/engine
    tag: "0.1.0"
  resources:
    requests:
      cpu: "1000m"
      memory: "2Gi"
    limits:
      cpu: "4000m"
      memory: "8Gi"
  persistence:
    enabled: true
    storageClass: ""
    size: 50Gi
  config:
    maxDimension: 65536
    walSegmentBytes: 67108864
    indexBuildThreads: 2

api:
  replicaCount: 2
  image:
    repository: vectrus/api
    tag: "0.1.0"
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2000m"
      memory: "2Gi"
  config:
    jwtSecretKey: ""  # Required
    apiKeyHashAlgorithm: argon2

dashboard:
  enabled: true
  replicaCount: 1
  image:
    repository: vectrus/dashboard
    tag: "0.1.0"

postgresql:
  enabled: true
  auth:
    postgresPassword: ""
    database: vectrus
  primary:
    persistence:
      size: 20Gi

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      size: 5Gi

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: vectrus.example.com
      paths:
        - path: /v1
          service: api
        - path: /
          service: dashboard
  tls: []

metrics:
  serviceMonitor:
    enabled: false
    interval: 30s
```

**Testing**:
- `Unit: helm lint passes without errors`
- `Unit: helm template with default values renders valid YAML`
- `Unit: helm template with custom values overrides correctly`
- `Integration: helm install in kind/k3d cluster, all pods reach Ready`
- `Integration: curl API endpoint through Ingress returns 200`

#### 11.2 — Architecture and Operations Documentation

**What**: Write architecture documentation covering deployment topology, scaling guidance, backup/restore procedures, and security hardening.

**Design**:

Documentation covers:
- Architecture diagram: engine <-> API <-> PostgreSQL/Redis
- Deployment modes: single-node Docker Compose vs. multi-node Kubernetes
- Scaling guidance: when to add engine replicas (based on QPS and vector count)
- Backup: snapshot creation via API, PostgreSQL pg_dump for control plane
- Restore: snapshot restore procedure, Alembic migration replay
- Security: TLS configuration, API key rotation, RBAC setup, network policies
- Monitoring: Grafana dashboard import, alerting rules (high latency, disk space, error rates)
- GDPR: data deletion procedure, data residency configuration

**Testing**:
- `Unit: all code examples in documentation are syntactically valid`
- `Manual: follow Quick Start guide on a fresh machine, verify working deployment`

---

## Phase 12: Integration Testing, Benchmarks, and LangChain/LlamaIndex Integration

### Purpose

Build comprehensive end-to-end test suites, publish performance benchmarks following ANN-Benchmarks methodology, and implement first-class LangChain and LlamaIndex integrations. After this phase, the platform is ready for beta users.

### Tasks

#### 12.1 — End-to-End Test Suite

**What**: Build a comprehensive E2E test suite that exercises the full stack: SDK -> API -> engine -> storage -> response.

**Design**:

```python
# tests/e2e/test_full_workflow.py

class TestFullWorkflow:
    """E2E tests against a running Vectrus stack (docker compose)."""

    async def test_collection_lifecycle(self, client: VectrusClient):
        """Create collection -> upsert -> query -> delete -> verify gone."""

    async def test_filtered_search(self, client: VectrusClient):
        """Upsert vectors with metadata, query with filter, verify filtered results."""

    async def test_namespace_isolation(self, client: VectrusClient):
        """Upsert to namespace A and B, query A only returns A's vectors."""

    async def test_hybrid_search(self, client: VectrusClient):
        """Upsert with sparse+dense vectors, hybrid query returns fused results."""

    async def test_snapshot_restore(self, client: VectrusClient):
        """Create snapshot, delete vectors, restore snapshot, verify vectors recovered."""

    async def test_rate_limiting(self, client: VectrusClient):
        """Send requests exceeding rate limit, verify 429 responses."""

    async def test_plan_limit_enforcement(self, client: VectrusClient):
        """Exceed plan vector limit, verify 402 response."""

    async def test_api_key_scoping(self, client: VectrusClient):
        """Create read-only key, verify write operations return 403."""

    async def test_concurrent_upsert(self, client: VectrusClient):
        """50 concurrent upsert requests, verify all succeed without data loss."""
```

**Testing**:
- All tests listed above are the test cases. They run against a Docker Compose stack.

#### 12.2 — ANN-Benchmarks Performance Suite

**What**: Implement benchmarks following the ANN-Benchmarks methodology using standard datasets (SIFT, GloVe, NYTimes).

**Design**:

```python
# benchmarks/ann_benchmark.py

DATASETS = {
    "sift-128-euclidean": {
        "dimension": 128,
        "metric": "euclidean",
        "train_url": "http://ann-benchmarks.com/sift-128-euclidean.hdf5",
        "vector_count": 1_000_000,
    },
    "glove-100-angular": {
        "dimension": 100,
        "metric": "cosine",
        "train_url": "http://ann-benchmarks.com/glove-100-angular.hdf5",
        "vector_count": 1_183_514,
    },
}

class BenchmarkRunner:
    def run(self, dataset: str, params: dict) -> BenchmarkResult:
        """
        1. Load dataset
        2. Insert all train vectors
        3. Build index with given params
        4. Run all test queries
        5. Measure: recall@10, QPS, p50/p95/p99 latency, memory usage
        """

class BenchmarkResult:
    dataset: str
    params: dict
    recall_at_10: float
    qps: float
    p50_latency_ms: float
    p95_latency_ms: float
    p99_latency_ms: float
    memory_mb: float
    build_time_seconds: float
```

**Testing**:
- `Benchmark: SIFT-1M with default HNSW params — recall@10 > 0.95, QPS > 1000`
- `Benchmark: SIFT-1M with scalar quantization — recall@10 > 0.93, memory < 50% of unquantized`
- `Benchmark: GloVe-100 with hybrid search — verify combined retrieval quality`

#### 12.3 — LangChain and LlamaIndex Integrations

**What**: Build first-class vector store integrations for LangChain and LlamaIndex, the two dominant LLM application frameworks.

**Design**:

```python
# LangChain integration
from langchain_core.vectorstores import VectorStore
from langchain_core.documents import Document

class VectrusVectorStore(VectorStore):
    """LangChain VectorStore implementation for Vectrus."""

    def __init__(
        self,
        client: VectrusClient,
        collection_id: str,
        embedding: Embeddings,
        namespace: str = "",
    ):
        self.client = client
        self.collection_id = collection_id
        self.embedding = embedding
        self.namespace = namespace

    def add_texts(
        self,
        texts: list[str],
        metadatas: list[dict] | None = None,
        ids: list[str] | None = None,
        **kwargs,
    ) -> list[str]:
        """Embed texts and upsert into Vectrus."""

    def similarity_search(
        self,
        query: str,
        k: int = 4,
        filter: dict | None = None,
        **kwargs,
    ) -> list[Document]:
        """Embed query and search Vectrus."""

    def similarity_search_with_score(
        self,
        query: str,
        k: int = 4,
        filter: dict | None = None,
    ) -> list[tuple[Document, float]]:
        """Search with relevance scores."""

    @classmethod
    def from_texts(
        cls,
        texts: list[str],
        embedding: Embeddings,
        metadatas: list[dict] | None = None,
        **kwargs,
    ) -> "VectrusVectorStore":
        """Create a VectrusVectorStore from a list of texts."""
```

**Testing**:
- `Unit (mocked client): add_texts embeds and upserts correctly`
- `Unit (mocked client): similarity_search returns Documents with page_content and metadata`
- `Unit (mocked client): similarity_search_with_score returns tuples with scores`
- `Unit (mocked client): from_texts creates collection and upserts`
- `Integration: LangChain RAG chain with Vectrus retriever returns relevant answers`
- `Integration: LlamaIndex VectorStoreIndex with Vectrus backend indexes and queries correctly`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                ─── required by everything
    │
Phase 2: Vector Engine Core       ─── requires Phase 1
    │
Phase 3: Engine Network Layer     ─── requires Phase 2
    │
Phase 4: Control Plane            ─── requires Phase 1 + Phase 3
    │
    ├── Phase 5: Billing & Audit   ─── requires Phase 4
    │
    ├── Phase 6: Observability     ─── requires Phase 3 + Phase 4
    │                                  (can parallel with Phase 5)
    │
    ├── Phase 7: Quantization &    ─── requires Phase 2 + Phase 3
    │   Hybrid Search                  (can parallel with Phases 5, 6, 8)
    │
    └── Phase 8: Web Dashboard     ─── requires Phase 4
                                       (can parallel with Phases 5, 6, 7)
         │
Phase 9: SDKs & MCP Server        ─── requires Phase 4
                                       (can parallel with Phases 5-8)
         │
Phase 10: AI-Native Features      ─── requires Phase 4 + Phase 7
                                       (partially parallel with 8, 9)
         │
Phase 11: Helm & Deployment       ─── requires Phases 3, 4, 6
                                       (can parallel with 9, 10)
         │
Phase 12: E2E Tests & Integrations ─── requires all prior phases
```

**Parallelism opportunities:**
- Phases 5, 6, 7, 8, and 9 can all be developed concurrently after Phase 4 is complete.
- Phase 7 (quantization/hybrid) only depends on the engine (Phases 2-3), not the control plane.
- Phase 11 (Helm) can proceed once the container images are buildable (Phases 3-4, 6).

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specifications.
2. All unit tests pass (`cargo test` for Rust, `pytest` for Python, `vitest` for TypeScript).
3. All integration tests pass against the Docker Compose development stack.
4. `ruff check` and `ruff format --check` pass for all Python code.
5. `mypy` passes with no errors for all Python code.
6. `clippy` passes with no warnings for all Rust code.
7. `rustfmt --check` passes for all Rust code.
8. Docker images build successfully (`docker compose build`).
9. The feature works end-to-end (can be demonstrated via API call, CLI, or dashboard).
10. New configuration options are documented in `values.yaml` (Helm) or environment variable table.
11. New API endpoints appear in the auto-generated OpenAPI spec at `/v1/openapi.json`.
12. Database migrations created via Alembic and tested (upgrade + downgrade).
13. Prometheus metrics for new operations are exposed and visible in Grafana dashboards.
14. Audit log entries are created for all new mutation operations.
