# Vector Database as a Service

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source managed vector storage, indexing, and similarity search platform that automates the expert tuning incumbents leave to users.

Vector Database as a Service provides scalable approximate-nearest-neighbour (ANN) search with metadata filtering, hybrid retrieval, and multi-tenant isolation. It targets AI/ML engineers building RAG pipelines, platform engineers embedding semantic search into SaaS products, and data teams powering recommendation systems. The core problem: today's vector databases force a hard trade-off between expensive proprietary SaaS (Pinecone, Zilliz Cloud) and operationally complex open-source self-hosting (Milvus, Weaviate at scale).

---

## Why Vector Database as a Service?

- **Pricing bifurcation leaves a gap.** Managed offerings cluster at $25–$350/mo for typical RAG workloads and reach six figures annually for enterprise Pinecone or Zilliz deals, while self-hosting Milvus or Weaviate requires multi-component infrastructure (etcd, MinIO, Pulsar) that small teams cannot operate.
- **No incumbent automates expert knobs.** Index parameter tuning (ef_construction, M, quantization), embedding model selection, and hybrid-search alpha weighting all remain trial-and-error across every surveyed product.
- **Compliance gap for regulated industries.** Pinecone and Turbopuffer have no self-hosted option, blocking EU AI Act and GDPR-constrained deployments that need data residency control.
- **Observability is missing.** No surveyed provider offers automated drift detection on embedding distributions or retrieval quality regression alerts; teams build custom monitoring.
- **ORM and framework tooling lags.** Prisma pgvector support is still incomplete as of 2026, and there is no first-class Django or Rails vector ORM.

---

## Key Features

### Core Vector Search

- Dense vector upsert and ANN search using HNSW indexing
- Distance metrics: cosine, L2 (Euclidean), inner product
- Metadata filtering (key-value, range, boolean) applied server-side before scoring
- Real-time single-record upsert and batch ingestion
- Snapshot/restore for backup and point-in-time recovery

### Hybrid and Multi-Modal Retrieval

- Hybrid search fusing sparse (BM25 / SPLADE) and dense vectors in a single query
- Quantization strategies (scalar, binary, product) for memory and cost reduction
- Multi-modal vector support for image, audio, and video embeddings stored natively
- GPU-accelerated indexing path (NVIDIA CAGRA integration)

### Platform and Multi-Tenancy

- Logical namespace or collection isolation per tenant
- RBAC with collection-level and field-level permissions
- API key authentication with per-key scoping
- Managed cloud deployment alongside self-hosted Docker/Kubernetes option

### Developer Experience

- REST API with OpenAPI 3.1 specification
- Python and JavaScript/TypeScript SDKs
- LangChain and LlamaIndex integrations
- MCP server for AI agent memory integration

### Observability

- Prometheus metrics endpoint
- OpenTelemetry tracing
- Grafana dashboard templates

---

## AI-Native Advantage

The project closes the gaps every incumbent leaves to users. AI capabilities include automated embedding model selection benchmarked against the user's own corpus and query distribution, intelligent index parameter tuning (ef_construction, M, quantization) learned from observed query patterns, LLM-assisted schema design that suggests filterable payload fields from sample documents, embedding-distribution drift detection that alerts before retrieval quality degrades in production, and a natural-language query interface that translates plain-English intent into optimised vector + filter query plans.

---

## Tech Stack & Deployment

Two deployment modes: a managed cloud service and a self-hosted distribution shipped as Docker images and a Helm chart for Kubernetes. The query surface is REST plus gRPC, with a published OpenAPI 3.1 spec and SDKs for Python and JavaScript/TypeScript. Indexing uses HNSW for primary ANN with optional GPU CAGRA acceleration. Standard transports are gRPC and REST, matching the conventions of every surveyed competitor.

---

## Market Context

The Vector Database as a Service market was valued at USD 1.62 billion in 2025 and is projected to reach USD 4.68 billion by 2029 (30.3% CAGR), per Research and Markets (2025). The broader vector database market is forecast at USD 2.38 billion in 2025 growing to USD 18.86 billion by 2035 (SNS Insider, 2025). Incumbent managed pricing spans roughly $25/mo entry tiers (Qdrant Cloud, Weaviate Cloud) to $150–$350/mo for 10M-vector workloads on Zilliz Cloud, with six-figure annual enterprise contracts at Pinecone and Zilliz. Primary buyers are AI/ML engineers, platform engineers, data science teams, and enterprise knowledge management groups.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
