# Standards & API Reference

> Project: Vector Database as a Service · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 25010:2023 — Systems and Software Quality Requirements and Evaluation (SQuaRE)**
- URL: https://www.iso.org/standard/78176.html
- Defines quality characteristics for software products including performance efficiency, reliability, and security. Relevant to benchmarking vector database recall, latency SLAs, and availability guarantees in a managed service offering.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The primary international standard for information security management. Enterprise buyers of vector-database-as-a-service increasingly require ISO 27001 certification alongside SOC 2 Type II as a vendor qualification criterion.

**ISO/IEC 27017:2015 — Code of Practice for Information Security Controls for Cloud Services**
- URL: https://www.iso.org/standard/43757.html
- Cloud-specific extension to ISO 27001. Directly applicable to managed vector database providers operating on shared cloud infrastructure.

**ISO/IEC 27018:2019 — Protection of Personally Identifiable Information (PII) in Public Cloud**
- URL: https://www.iso.org/standard/76559.html
- Governs PII handling in cloud environments. Relevant when vector embeddings are derived from personal data (user profiles, medical records, documents) and stored in a managed service.

---

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics (2022)**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- Defines HTTP/1.1 and HTTP/2 request/response semantics. All REST APIs for vector databases build on HTTP; adherence ensures correct status codes, caching headers, and content negotiation.

**RFC 9114 — HTTP/3 (2022)**
- URL: https://www.rfc-editor.org/rfc/rfc9114
- HTTP over QUIC. Relevant for low-latency vector query APIs, particularly for high-frequency agent workloads where connection overhead matters.

**RFC 8288 — Web Linking (2017)**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines `Link` header usage for pagination, filtering, and related resource discovery in REST APIs. Applicable to paginated vector search result sets and collection listing endpoints.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Standard token format for authentication and authorisation. Used by Weaviate, Milvus, and Qdrant Cloud for bearer token authentication in their REST and gRPC APIs.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- De-facto standard for delegated authorisation. Enterprise vector database deployments integrate with corporate identity providers (Okta, Azure AD) via OAuth 2.0 Client Credentials and Authorization Code flows.

**gRPC / Protocol Buffers (Google / CNCF)**
- URL: https://grpc.io/ and https://protobuf.dev/
- Binary RPC protocol and serialisation format used as the high-performance transport layer by Qdrant, Weaviate, and Milvus alongside their REST APIs. Reduces serialisation overhead for high-throughput vector ingestion.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.0**
- URL: https://spec.openapis.org/oas/v3.1.0 and https://swagger.io/specification/
- The industry standard for describing RESTful APIs in YAML or JSON. Pinecone, Qdrant, and Weaviate publish OpenAPI 3.x specifications enabling client SDK auto-generation. Full JSON Schema Draft 2020-12 compatibility introduced in OAS 3.1.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used for validating vector payload/metadata schemas in all major vector databases. Weaviate collection schemas and Qdrant payload index definitions follow JSON Schema conventions.

**Protocol Buffers (proto3)**
- URL: https://protobuf.dev/programming-guides/proto3/
- Binary serialisation format used in gRPC APIs for Qdrant, Weaviate, and Milvus. Offers 3–10x throughput improvement over JSON for bulk vector upsert operations.

**ANN-Benchmarks methodology**
- URL: https://ann-benchmarks.com/ and https://github.com/erikbern/ann-benchmarks
- Community-standard benchmarking framework for approximate nearest-neighbor algorithms. Defines standardised datasets (SIFT, GIST, GloVe, NYTimes) and recall@k measurement methodology. Used as the reference baseline by Qdrant, Weaviate, Milvus, and pgvector in their published benchmarks.

**HNSW Algorithm Specification**
- Reference: Malkov & Yashunin (2018). IEEE TPAMI. https://arxiv.org/abs/1603.09320
- The dominant ANN indexing algorithm implemented across Pinecone, Qdrant, Weaviate, Milvus, pgvector, and LanceDB. Not a formal standards-body specification, but functions as a de-facto standard for graph-based ANN indexing.

---

### Security & Authentication Standards

**NIST SP 800-207 — Zero Trust Architecture (2020)**
- URL: https://csrc.nist.gov/pubs/sp/800/207/final
- Framework defining a "never trust, always verify" security model using Policy Engines, Policy Administrators, and Policy Enforcement Points. Relevant for enterprise vector database deployments that serve multiple tenants or cross organisational boundaries.

**NIST SP 800-207A — Zero Trust Architecture for Cloud-Native Applications**
- URL: https://csrc.nist.gov/pubs/sp/800/207/a/final
- Extension of SP 800-207 for multi-cloud environments. Directly applicable to managed vector database services operating across AWS, GCP, and Azure.

**FIPS 140-3 — Security Requirements for Cryptographic Modules**
- URL: https://csrc.nist.gov/publications/detail/fips/140/3/final
- US federal standard for cryptographic module validation. Required for vector database deployments in US government and defence contexts. Pinecone and Turbopuffer advertise FIPS-compatible encryption at rest and in transit.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Defines the top API security risks including Broken Object Level Authorization (BOLA), Broken Authentication, and Excessive Data Exposure. Directly applicable to the design of vector database REST and gRPC APIs, particularly around namespace isolation and metadata filtering bypass risks.

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- US auditing standard for SaaS providers covering security, availability, processing integrity, confidentiality, and privacy. Pinecone, Turbopuffer, Weaviate Cloud, and Zilliz Cloud hold or are pursuing SOC 2 Type II certification.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- Requires data residency controls, right-to-erasure mechanisms, and audit logging when vectors derived from EU personal data are stored. EU AI Act penalties (up to 35M EUR or 7% global revenue) amplify compliance urgency for cloud-only managed services.

**HIPAA — Health Insurance Portability and Accountability Act**
- URL: https://www.hhs.gov/hipaa/
- US healthcare data regulation requiring Business Associate Agreements (BAAs) and encryption at rest/in transit for PHI. Pinecone and Turbopuffer offer HIPAA-compliant plans with BAAs.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic Open Standard (2024)**
- URL: https://modelcontextprotocol.io/ and https://github.com/anthropics/mcp
- Open protocol for connecting AI assistants to external data sources and tools. Vector databases are a natural MCP retrieval surface: an agent calls a `search` tool, the MCP server performs ANN retrieval, and results are returned as tool output. By 2026, Qdrant and Milvus ship official MCP servers; MindsDB provides a unified MCP server covering multiple vector stores.
- Qdrant MCP server: https://github.com/qdrant/mcp-server-qdrant
- Milvus MCP server: https://milvus.io/docs/milvus_and_mcp.md

---

## Similar Products — Developer Documentation & APIs

### Pinecone

- **Description:** Fully managed, serverless vector database purpose-built for AI applications. Supports dense, sparse, and hybrid search with an integrated Inference API for embedding generation and reranking.
- **API Documentation:** https://docs.pinecone.io/reference/api/introduction
- **SDKs/Libraries:**
  - Python: https://sdk.pinecone.io/python/index.html (package: `pinecone`)
  - JavaScript/TypeScript: https://github.com/pinecone-io/pinecone-node-client
  - Go: https://github.com/pinecone-io/go-pinecone
  - Java: https://github.com/pinecone-io/pinecone-java-client
- **Developer Guide:** https://docs.pinecone.io/guides/get-started/quickstart
- **OpenAPI Specs:** https://github.com/pinecone-io/pinecone-api
- **Standards:** REST/JSON, OpenAPI 3.x, versioned API headers (e.g., `X-Pinecone-API-Version: 2025-10`)
- **Authentication:** API Key (per-project key, passed as `Authorization: Bearer <key>` header)

---

### Qdrant

- **Description:** High-performance open-source vector search engine written in Rust, with managed cloud and self-hosted deployment options. Supports dense and sparse vectors, rich payload filtering, and multiple quantization strategies.
- **API Documentation:** https://api.qdrant.tech/api-reference
- **SDKs/Libraries:**
  - Python: https://github.com/qdrant/qdrant-client (package: `qdrant-client`)
  - JavaScript/TypeScript: https://github.com/qdrant/qdrant-js
  - Rust: https://github.com/qdrant/rust-client
  - Go: https://github.com/qdrant/go-client
  - .NET: https://github.com/qdrant/qdrant-dotnet
  - Java: https://github.com/qdrant/java-client
- **Developer Guide:** https://qdrant.tech/documentation/quickstart/
- **Full Documentation:** https://qdrant.tech/documentation/
- **Standards:** REST/JSON, gRPC (Protocol Buffers), OpenAPI v3 spec published, OpenTelemetry traces
- **Authentication:** API Key (header `api-key`); mTLS supported for self-hosted deployments

---

### Weaviate

- **Description:** Open-source vector database combining object storage with vector search, supporting multi-modal data, GraphQL/REST/gRPC APIs, a rich ML module ecosystem, and native multi-tenancy at million-tenant scale.
- **API Documentation:** https://weaviate.io/developers/weaviate/api
- **SDKs/Libraries:**
  - Python: https://weaviate-python-client.readthedocs.io/ (package: `weaviate-client`)
  - JavaScript/TypeScript: https://github.com/weaviate/typescript-client
  - Go: https://github.com/weaviate/weaviate-go-client
  - Java: https://github.com/weaviate/java-client
- **Developer Guide:** https://weaviate.io/developers/weaviate/quickstart
- **Full Documentation:** https://docs.weaviate.io/weaviate
- **Standards:** REST/JSON, GraphQL, gRPC (high-performance query path), OpenAPI, Prometheus metrics endpoint
- **Authentication:** API Key; OIDC/OAuth2 (OpenID Connect) for enterprise deployments; RBAC built-in

---

### Milvus / Zilliz Cloud

- **Description:** CNCF-graduated cloud-native vector database designed for billion-scale ANN search. Zilliz Cloud is the fully managed offering with the Cardinal search engine for 10x retrieval speedup.
- **API Documentation:**
  - Milvus REST API: https://milvus.io/api-reference/restful/v2.5.x/About.md
  - Zilliz Cloud API: https://docs.zilliz.com/reference/restful/authentication
- **SDKs/Libraries:**
  - Python: https://milvus.io/docs/install-pymilvus.md (package: `pymilvus`)
  - Java: https://milvus.io/docs/install-java.md
  - Go: https://milvus.io/docs/install-go.md
  - Node.js: https://milvus.io/docs/install-node.md
  - Zilliz Cloud SDK install: https://docs.zilliz.com/docs/install-sdks
- **Developer Guide:** https://milvus.io/docs/quickstart.md
- **Full Documentation:** https://milvus.io/docs/
- **Standards:** REST/JSON, gRPC (Protocol Buffers), Prometheus metrics, OpenTelemetry
- **Authentication:** API Key for Zilliz Cloud; username/password + RBAC for self-hosted Milvus

---

### Chroma

- **Description:** Lightweight, embeddable open-source vector store optimised for LLM application prototyping. Runs in-process (EphemeralClient), on-disk (PersistentClient), or as a server (HttpClient). Chroma Cloud provides serverless managed hosting.
- **API Documentation:** https://docs.trychroma.com/reference/py-client
- **SDKs/Libraries:**
  - Python: https://github.com/chroma-core/chroma (package: `chromadb`)
  - JavaScript/TypeScript: https://github.com/chroma-core/chroma (package: `chromadb`)
- **Developer Guide:** https://docs.trychroma.com/getting-started
- **Full Documentation:** https://docs.trychroma.com/
- **Standards:** REST/JSON (HttpClient mode); no gRPC; no formal OpenAPI spec published
- **Authentication:** API Key (Chroma Cloud); no auth in default self-hosted mode

---

### pgvector (PostgreSQL extension)

- **Description:** Open-source PostgreSQL extension adding vector similarity search. Enables teams to store and query high-dimensional vectors alongside relational data using standard SQL, with HNSW and IVFFlat indexes.
- **API Documentation:** https://github.com/pgvector/pgvector (README serves as spec)
- **SDKs/Libraries:** Any PostgreSQL client library (psycopg2, asyncpg, pg, node-postgres, JDBC, etc.); no dedicated vector SDK needed
- **Managed Offerings:**
  - Supabase: https://supabase.com/docs/guides/database/extensions/pgvector
  - Neon: https://neon.com/docs/extensions/pgvector
  - AWS RDS: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.PGVECTOR.html
- **Developer Guide:** https://github.com/pgvector/pgvector#getting-started
- **Standards:** Standard PostgreSQL wire protocol; SQL:2023; OpenAPI not applicable
- **Authentication:** Standard PostgreSQL authentication (md5, SCRAM-SHA-256, mTLS)

---

### LanceDB

- **Description:** Embedded, serverless vector database built on the Lance columnar format. Supports multimodal data, automatic versioning, DuckDB SQL integration, and full-text search. LanceDB Cloud is the managed serverless offering.
- **API Documentation:** https://lancedb.github.io/lancedb/
- **SDKs/Libraries:**
  - Python: https://github.com/lancedb/lancedb (package: `lancedb`)
  - JavaScript/TypeScript: https://github.com/lancedb/lancedb (package: `@lancedb/lancedb`)
  - Rust: https://github.com/lancedb/lance
- **Developer Guide:** https://lancedb.github.io/lancedb/basic/
- **Cloud Documentation:** https://docs.lancedb.com/cloud
- **Standards:** Python/JS library API (embedded); REST API for LanceDB Cloud; DuckDB SQL dialect for analytics queries
- **Authentication:** API Key for LanceDB Cloud; no auth in embedded mode

---

### Turbopuffer

- **Description:** Serverless search engine built on object storage, designed for extreme scale (2 trillion+ documents) and cost efficiency (~10x cheaper than in-memory alternatives). Combines vector and full-text search. HIPAA and SOC 2 compliant.
- **API Documentation:** https://turbopuffer.com/docs
- **SDKs/Libraries:**
  - Python: https://github.com/turbopuffer/turbopuffer-python (package: `turbopuffer`)
  - TypeScript: https://github.com/turbopuffer/turbopuffer-typescript (package: `turbopuffer`)
- **Developer Guide:** https://turbopuffer.com/docs/quickstart
- **Standards:** REST/JSON; no gRPC; no published OpenAPI spec
- **Authentication:** API Key (Bearer token)

---

## Notes

**Emerging standard: IEEE P3396 (Vector Database)**
An IEEE working group (P3396) exploring standardisation of vector database interfaces and data models was reported in early 2026. No final specification has been published. Practitioners should monitor https://standards.ieee.org for updates.

**Embedding model interoperability**
There is currently no industry standard for embedding model interfaces. OpenAI's Embeddings API (`POST /v1/embeddings`, response schema `data[].embedding`) has become a de-facto convention that Weaviate, Qdrant, and others accept as a module input format. Hugging Face Inference API follows a similar pattern. A common embedding interface standard would significantly reduce vendor lock-in at the embedding layer.

**Hybrid search scoring**
Reciprocal Rank Fusion (RRF) and Convex Combination (linear alpha blending) are the two dominant hybrid-search fusion strategies. Neither is codified in an official standard; Weaviate, Qdrant, and Milvus each implement their own variants with different default parameters, creating interoperability friction when migrating between systems.

**GDPR / EU AI Act compliance gap**
No vector database provider currently offers a certified, automated right-to-erasure workflow for embeddings derived from personal data. This is an active regulatory gap — embedding vectors may implicitly contain personal information even after the source document is deleted, and no standard mechanism exists to verify deletion completeness at the vector level.
