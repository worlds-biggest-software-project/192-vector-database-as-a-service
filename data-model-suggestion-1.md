# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Vector Database as a Service · Created: 2026-05-20

## Philosophy

This model applies classical relational normalization to the **control plane** of a vector database service -- the management layer that handles tenants, billing, RBAC, API keys, collection configuration, and index lifecycle. The data plane (actual vector storage and HNSW indexes) lives in a purpose-built vector engine; this schema governs everything around it.

Every concept gets its own table with strict foreign key relationships. Reference data (regions, distance metrics, index types, quantization strategies) is stored in dedicated lookup tables rather than as free-text strings. This makes the schema self-documenting and ensures referential integrity at the database level rather than relying on application validation.

The normalized approach mirrors how Pinecone, Qdrant Cloud, and Weaviate Cloud structure their control planes internally: the customer-facing API metadata (collections, namespaces, API keys, usage records) is relational, while the vector engine itself operates on its own optimized storage format (segments, shards, HNSW graphs).

**Best for:** Teams prioritizing data integrity, complex cross-entity reporting (billing reconciliation, compliance audits), and a well-defined schema that can be enforced at the database layer.

**Trade-offs:**
- Pro: Maximum referential integrity -- invalid states are prevented by the schema
- Pro: Standard SQL tooling works out of the box (pgAdmin, Metabase, dbt)
- Pro: Easy to add new reference data without schema migration
- Pro: Clean separation of control plane (PostgreSQL) and data plane (vector engine)
- Con: More tables to manage (~35-40 tables for full feature set)
- Con: JOIN-heavy queries for dashboard views and billing aggregation
- Con: Schema migrations needed for structural changes (new entity types)
- Con: Metadata schema flexibility requires additional modeling (see payload_schema tables)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 27001:2022 | Audit log tables and RBAC design follow information security management requirements |
| ISO/IEC 27018:2019 | Data residency tracking via `regions` and `collection_region_assignments` tables |
| OAuth 2.0 (RFC 6749) / JWT (RFC 7519) | `api_keys` and `user_sessions` tables model token-based authentication |
| OpenAPI 3.1 | Collection and index configuration tables mirror the OpenAPI resource model |
| GDPR (EU) 2016/679 | `data_deletion_requests` table tracks right-to-erasure workflows |
| SOC 2 Type II | Comprehensive audit logging via `audit_logs` table |
| OWASP API Security Top 10 | RBAC tables enforce object-level authorization (BOLA prevention) |

---

## Identity & Multi-Tenancy

```sql
-- Organizations are the top-level tenant boundary
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    plan_id         UUID REFERENCES plans(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'cancelled')),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organizations_slug ON organizations(slug);
CREATE INDEX idx_organizations_status ON organizations(status);

-- Users belong to one or more organizations via memberships
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) DEFAULT 'local'
                    CHECK (auth_provider IN ('local', 'google', 'github', 'saml')),
    auth_provider_id VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users(email);

-- Organization membership with role assignment
CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id),
    invited_by      UUID REFERENCES users(id),
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);
CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);

-- Projects group collections within an organization
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);
CREATE INDEX idx_projects_org ON projects(organization_id);
```

## RBAC (Role-Based Access Control)

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- System roles: owner, admin, member, viewer

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   VARCHAR(50) NOT NULL
                    CHECK (resource_type IN ('organization', 'project', 'collection', 'api_key', 'member')),
    action          VARCHAR(50) NOT NULL
                    CHECK (action IN ('create', 'read', 'update', 'delete', 'query', 'upsert', 'manage')),
    description     TEXT,
    UNIQUE (resource_type, action)
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- Per-collection permission overrides (field-level access)
CREATE TABLE collection_access_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id),
    can_query       BOOLEAN NOT NULL DEFAULT true,
    can_upsert      BOOLEAN NOT NULL DEFAULT false,
    can_delete      BOOLEAN NOT NULL DEFAULT false,
    can_manage      BOOLEAN NOT NULL DEFAULT false,
    field_filter    JSONB DEFAULT NULL,
    -- Example field_filter: {"allowed_metadata_keys": ["category", "source"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_col_access_collection ON collection_access_policies(collection_id);
```

## API Key Management

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(8) NOT NULL,       -- First 8 chars shown in UI
    key_hash        VARCHAR(255) NOT NULL,      -- bcrypt/argon2 hash of full key
    role_id         UUID NOT NULL REFERENCES roles(id),
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    -- e.g. ARRAY['collections:read', 'collections:write', 'query']
    rate_limit_rps  INTEGER DEFAULT 100,
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_keys_org ON api_keys(organization_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
CREATE INDEX idx_api_keys_project ON api_keys(project_id);
```

## Collection & Index Configuration

```sql
-- Reference: distance metrics
CREATE TABLE distance_metrics (
    id              VARCHAR(30) PRIMARY KEY,
    -- 'cosine', 'euclidean', 'dot_product'
    description     TEXT
);

-- Reference: index types
CREATE TABLE index_types (
    id              VARCHAR(30) PRIMARY KEY,
    -- 'hnsw', 'ivf_flat', 'ivf_pq', 'flat', 'diskann'
    description     TEXT
);

-- Reference: quantization strategies
CREATE TABLE quantization_types (
    id              VARCHAR(30) PRIMARY KEY,
    -- 'none', 'scalar', 'binary', 'product'
    description     TEXT
);

-- Deployment regions
CREATE TABLE regions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cloud_provider  VARCHAR(20) NOT NULL CHECK (cloud_provider IN ('aws', 'gcp', 'azure')),
    region_code     VARCHAR(30) NOT NULL,      -- e.g. 'us-east-1', 'eu-west-1'
    display_name    VARCHAR(100) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (cloud_provider, region_code)
);

-- Collections: the primary user-facing resource
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    dimension       INTEGER NOT NULL CHECK (dimension > 0 AND dimension <= 65536),
    distance_metric VARCHAR(30) NOT NULL REFERENCES distance_metrics(id),
    region_id       UUID NOT NULL REFERENCES regions(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'creating'
                    CHECK (status IN ('creating', 'ready', 'indexing', 'error', 'deleting')),
    vector_count    BIGINT NOT NULL DEFAULT 0,
    storage_bytes   BIGINT NOT NULL DEFAULT 0,
    replica_count   INTEGER NOT NULL DEFAULT 1,
    shard_count     INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name)
);
CREATE INDEX idx_collections_project ON collections(project_id);
CREATE INDEX idx_collections_status ON collections(status);

-- Namespaces for logical partitioning within a collection
CREATE TABLE namespaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    vector_count    BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (collection_id, name)
);
CREATE INDEX idx_namespaces_collection ON namespaces(collection_id);

-- Index configurations per collection
CREATE TABLE collection_indexes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    index_type      VARCHAR(30) NOT NULL REFERENCES index_types(id),
    quantization    VARCHAR(30) NOT NULL REFERENCES quantization_types(id) DEFAULT 'none',
    -- HNSW parameters
    hnsw_m          INTEGER DEFAULT 16,
    hnsw_ef_construction INTEGER DEFAULT 200,
    hnsw_ef_search  INTEGER DEFAULT 100,
    -- IVF parameters
    ivf_nlist       INTEGER,
    -- Quantization parameters
    quant_bits      INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'building'
                    CHECK (status IN ('building', 'ready', 'rebuilding', 'error')),
    build_started_at TIMESTAMPTZ,
    build_completed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_col_indexes_collection ON collection_indexes(collection_id);

-- Payload (metadata) schema definitions per collection
CREATE TABLE payload_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    field_name      VARCHAR(255) NOT NULL,
    field_type      VARCHAR(30) NOT NULL
                    CHECK (field_type IN ('string', 'integer', 'float', 'boolean',
                                          'datetime', 'geo', 'string_array', 'json')),
    is_indexed      BOOLEAN NOT NULL DEFAULT false,
    is_required     BOOLEAN NOT NULL DEFAULT false,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (collection_id, field_name)
);
CREATE INDEX idx_payload_schemas_collection ON payload_schemas(collection_id);
```

## Embedding Model Configuration

```sql
-- Embedding model providers (OpenAI, Cohere, Hugging Face, etc.)
CREATE TABLE embedding_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    api_base_url    VARCHAR(500),
    auth_type       VARCHAR(30) CHECK (auth_type IN ('api_key', 'oauth2', 'none')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Specific embedding models
CREATE TABLE embedding_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider_id     UUID NOT NULL REFERENCES embedding_providers(id),
    model_name      VARCHAR(255) NOT NULL,
    dimension       INTEGER NOT NULL,
    max_tokens      INTEGER,
    supports_matryoshka BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider_id, model_name)
);

-- Per-collection embedding configuration (for built-in inference)
CREATE TABLE collection_embedding_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    embedding_model_id UUID NOT NULL REFERENCES embedding_models(id),
    api_key_encrypted BYTEA,    -- Encrypted customer API key for the provider
    batch_size      INTEGER DEFAULT 100,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (collection_id)
);
```

## Billing & Usage Metering

```sql
-- Subscription plans
CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,
    slug            VARCHAR(50) NOT NULL UNIQUE,
    tier            VARCHAR(20) NOT NULL CHECK (tier IN ('free', 'starter', 'pro', 'enterprise')),
    monthly_price_cents INTEGER,
    included_storage_bytes BIGINT,
    included_read_units BIGINT,
    included_write_units BIGINT,
    max_collections INTEGER,
    max_dimensions  INTEGER DEFAULT 4096,
    max_replicas    INTEGER DEFAULT 1,
    features        JSONB DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Usage metering (append-only, partitioned by month)
CREATE TABLE usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    collection_id   UUID REFERENCES collections(id),
    metric_type     VARCHAR(30) NOT NULL
                    CHECK (metric_type IN ('read_units', 'write_units', 'storage_bytes',
                                           'query_count', 'upsert_count', 'delete_count',
                                           'inference_tokens')),
    quantity        BIGINT NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions
-- CREATE TABLE usage_records_2026_05 PARTITION OF usage_records
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_usage_org_metric ON usage_records(organization_id, metric_type, recorded_at);
CREATE INDEX idx_usage_collection ON usage_records(collection_id, recorded_at);

-- Monthly billing invoices
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    billing_period_start DATE NOT NULL,
    billing_period_end   DATE NOT NULL,
    plan_id         UUID NOT NULL REFERENCES plans(id),
    base_amount_cents INTEGER NOT NULL,
    overage_amount_cents INTEGER NOT NULL DEFAULT 0,
    total_amount_cents INTEGER NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'finalized', 'paid', 'overdue', 'void')),
    stripe_invoice_id VARCHAR(255),
    issued_at       TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_invoices_org ON invoices(organization_id);
CREATE INDEX idx_invoices_period ON invoices(billing_period_start, billing_period_end);
```

## Observability & Audit

```sql
-- Audit log (append-only)
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id         UUID,
    api_key_id      UUID,
    action          VARCHAR(100) NOT NULL,
    -- e.g. 'collection.created', 'api_key.revoked', 'member.invited'
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    ip_address      INET,
    user_agent      TEXT,
    request_id      UUID,
    changes         JSONB,
    -- e.g. {"before": {"name": "old"}, "after": {"name": "new"}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org_time ON audit_logs(organization_id, created_at);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_user ON audit_logs(user_id, created_at);

-- Cluster / node health (for managed infrastructure)
CREATE TABLE cluster_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    region_id       UUID NOT NULL REFERENCES regions(id),
    hostname        VARCHAR(255) NOT NULL,
    node_type       VARCHAR(30) NOT NULL CHECK (node_type IN ('query', 'index', 'coordinator')),
    status          VARCHAR(20) NOT NULL DEFAULT 'healthy'
                    CHECK (status IN ('healthy', 'degraded', 'offline', 'draining')),
    capacity_vectors BIGINT,
    used_vectors    BIGINT,
    last_heartbeat  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_nodes_region ON cluster_nodes(region_id);
CREATE INDEX idx_nodes_status ON cluster_nodes(status);

-- Collection-to-node placement
CREATE TABLE collection_placements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES cluster_nodes(id),
    shard_id        INTEGER NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (collection_id, node_id, shard_id)
);
CREATE INDEX idx_placements_collection ON collection_placements(collection_id);
CREATE INDEX idx_placements_node ON collection_placements(node_id);
```

## GDPR & Data Governance

```sql
-- Track data deletion requests for GDPR right-to-erasure
CREATE TABLE data_deletion_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    scope           VARCHAR(30) NOT NULL
                    CHECK (scope IN ('collection', 'namespace', 'vector_ids', 'organization')),
    collection_id   UUID REFERENCES collections(id),
    namespace_id    UUID REFERENCES namespaces(id),
    vector_ids      UUID[],
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    reason          TEXT,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_deletion_org ON data_deletion_requests(organization_id);
CREATE INDEX idx_deletion_status ON data_deletion_requests(status);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organizations, users, organization_members, projects |
| RBAC | 4 | roles, permissions, role_permissions, collection_access_policies |
| API Keys | 1 | api_keys |
| Collections & Indexes | 5 | collections, namespaces, collection_indexes, payload_schemas, distance_metrics + 2 ref tables |
| Embedding Configuration | 3 | embedding_providers, embedding_models, collection_embedding_configs |
| Billing & Metering | 3 | plans, usage_records, invoices |
| Observability & Infrastructure | 3 | audit_logs, cluster_nodes, collection_placements |
| Data Governance | 1 | data_deletion_requests |
| Reference Data | 3 | distance_metrics, index_types, quantization_types |
| **Total** | **~27** | Plus monthly partitions for usage_records and audit_logs |

---

## Key Design Decisions

1. **Control plane only.** This schema models the management layer (tenants, billing, RBAC, collection metadata) -- not the vector data itself. Actual vectors, HNSW graphs, and payload data live in the vector engine's own storage format. This is how every major VDBaaS (Pinecone, Qdrant Cloud, Weaviate Cloud) structures their architecture.

2. **UUID primary keys everywhere.** Avoids sequential ID leakage (OWASP BOLA), works across distributed systems, and is the standard for modern SaaS.

3. **Partitioned append-only tables for metering and audit.** `usage_records` and `audit_logs` are partitioned by month for efficient time-range queries and partition-level retention management. Old partitions can be detached and archived.

4. **Reference tables for enums.** Distance metrics, index types, and quantization strategies are stored in lookup tables rather than CHECK constraints. This allows adding new options without schema migrations.

5. **Encrypted API key storage.** Only the key prefix (first 8 chars) and a bcrypt/argon2 hash are stored. The raw key is never persisted, following industry best practice.

6. **Payload schema as structured metadata.** Rather than allowing arbitrary metadata, `payload_schemas` defines the schema per collection, enabling the system to build appropriate indexes and validate payloads at ingestion time.

7. **Billing tied to usage partitions.** Monthly invoices are computed from `usage_records` partitions, enabling simple billing reconciliation: query the partition for the billing period and aggregate by metric type.

8. **Region-aware placement.** Collections are assigned to regions at creation, and `collection_placements` tracks which physical nodes host each shard. This supports data residency requirements (GDPR, EU AI Act) by ensuring vectors stay in the designated region.
