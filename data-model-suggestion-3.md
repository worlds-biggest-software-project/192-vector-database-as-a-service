# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Vector Database as a Service · Created: 2026-05-20

## Philosophy

This model uses PostgreSQL relational tables for the stable, well-understood parts of the domain (identity, billing, RBAC) and JSONB columns for everything that varies across tenants, regions, index types, or feature releases. The core insight is that a Vector Database as a Service platform has a dual nature: the control plane is a standard SaaS app (users, orgs, billing), but the data plane configuration is highly variable -- different index types have different parameters, different embedding providers have different config schemas, different tenants need different metadata schemas, and new features constantly add new configuration knobs.

Rather than creating a new relational table and running a migration every time a new index parameter or embedding provider is added, this model stores variable configuration in JSONB columns with JSON Schema validation at the application layer. The relational structure provides the foreign key graph and query performance for common operations, while JSONB provides the extensibility for rapid iteration.

This approach is used in production by companies like Stripe (payments have a core relational structure with JSONB metadata), Notion (page properties are JSONB), and most modern SaaS platforms that need to ship configuration changes without database migrations. PostgreSQL's JSONB indexing (GIN indexes, `@>` containment, `jsonb_path_ops`) makes this performant for the query patterns a VDBaaS control plane needs.

**Best for:** Teams that want to iterate fast on features without migration overhead, need to support tenant-specific configurations, or are building an MVP that will evolve rapidly. Also ideal when the platform spans multiple cloud providers and regions with different capability sets.

**Trade-offs:**
- Pro: New configuration fields added without schema migration
- Pro: Tenant-specific and region-specific customization via JSONB
- Pro: Fewer tables (~18-20 vs ~27 in fully normalized model)
- Pro: PostgreSQL GIN indexes on JSONB are efficient for containment queries
- Pro: Natural fit for storing variable index parameters, embedding configs, payload schemas
- Con: No database-level referential integrity on JSONB field references
- Con: Application must validate JSONB structure (JSON Schema validation layer required)
- Con: Complex JSONB queries can be harder to optimize than relational JOINs
- Con: Schema drift risk if validation is not enforced consistently
- Con: JSONB columns are opaque to standard SQL reporting tools without extraction

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JSON Schema (Draft 2020-12) | Validates all JSONB column structures at the application layer |
| OpenAPI 3.1 | Collection configs and index params documented as OpenAPI component schemas |
| ISO/IEC 27001:2022 | Audit log retains full JSONB change diffs for security review |
| GDPR (EU) 2016/679 | `data_governance` JSONB on organizations tracks residency and retention policies |
| OAuth 2.0 / JWT | Auth config stored as JSONB on organizations for SSO flexibility |
| OWASP API Security | RBAC permissions stored as structured JSONB arrays with validation |

---

## Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'cancelled')),
    plan_tier       VARCHAR(20) NOT NULL DEFAULT 'free'
                    CHECK (plan_tier IN ('free', 'starter', 'pro', 'enterprise')),

    -- Billing configuration (varies by payment provider)
    billing         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "email": "billing@acme.com",
    --   "stripe_customer_id": "cus_...",
    --   "payment_method": "card",
    --   "currency": "USD",
    --   "tax_id": "EU123456789",
    --   "billing_address": {
    --     "country": "DE", "city": "Berlin", "postal_code": "10115"
    --   }
    -- }

    -- Plan limits and feature flags (varies by plan and custom enterprise deals)
    plan_limits     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "max_collections": 10,
    --   "max_dimensions": 4096,
    --   "max_vectors_total": 10000000,
    --   "max_replicas": 3,
    --   "included_read_units": 1000000,
    --   "included_write_units": 500000,
    --   "included_storage_gb": 10,
    --   "features": {
    --     "hybrid_search": true,
    --     "gpu_indexing": false,
    --     "custom_embedding_models": true,
    --     "sso": false,
    --     "dedicated_nodes": false
    --   }
    -- }

    -- Data governance and compliance settings
    data_governance JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "allowed_regions": ["eu-west-1", "eu-central-1"],
    --   "data_residency": "EU",
    --   "retention_days": 365,
    --   "encryption_at_rest": true,
    --   "encryption_key_arn": "arn:aws:kms:...",
    --   "gdpr_dpo_email": "dpo@acme.com",
    --   "hipaa_baa_signed": false
    -- }

    -- SSO/auth configuration (varies by provider)
    auth_config     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "sso_provider": "okta",
    --   "saml_entity_id": "https://acme.okta.com/...",
    --   "saml_sso_url": "https://acme.okta.com/app/.../sso/saml",
    --   "saml_certificate": "MIID...",
    --   "allowed_email_domains": ["acme.com"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organizations_slug ON organizations(slug);
CREATE INDEX idx_organizations_status ON organizations(status);
CREATE INDEX idx_organizations_plan ON organizations(plan_tier);
-- GIN index for querying organizations by governance settings
CREATE INDEX idx_organizations_governance ON organizations
    USING GIN (data_governance jsonb_path_ops);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),

    -- Auth provider details (varies by provider)
    auth            JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "provider": "github",
    --   "provider_id": "12345",
    --   "avatar_url": "https://...",
    --   "mfa_enabled": true,
    --   "mfa_method": "totp"
    -- }

    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"timezone": "Europe/Berlin", "theme": "dark", "notifications": {"email": true, "slack": false}}

    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users(email);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(30) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'member', 'viewer')),

    -- Per-member permission overrides and scoping
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "collection_access": {
    --     "550e8400-...": {"can_query": true, "can_upsert": true, "can_delete": false},
    --     "660e8400-...": {"can_query": true, "can_upsert": false, "can_delete": false}
    --   },
    --   "project_scopes": ["proj-a", "proj-b"]
    -- }

    invited_by      UUID REFERENCES users(id),
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);
CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);
```

## Collections & Index Configuration

```sql
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    project_slug    VARCHAR(100),
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'creating'
                    CHECK (status IN ('creating', 'ready', 'indexing', 'error', 'deleting')),

    -- Core vector configuration (relational -- queried frequently)
    dimension       INTEGER NOT NULL CHECK (dimension > 0 AND dimension <= 65536),
    distance_metric VARCHAR(30) NOT NULL DEFAULT 'cosine'
                    CHECK (distance_metric IN ('cosine', 'euclidean', 'dot_product')),
    vector_count    BIGINT NOT NULL DEFAULT 0,
    storage_bytes   BIGINT NOT NULL DEFAULT 0,

    -- Deployment configuration
    deployment      JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "region": "us-east-1",
    --   "cloud_provider": "aws",
    --   "shard_count": 2,
    --   "replica_count": 1,
    --   "node_type": "query",
    --   "placement": {
    --     "shards": [
    --       {"shard_id": 0, "primary_node": "node-abc", "replicas": ["node-def"]},
    --       {"shard_id": 1, "primary_node": "node-ghi", "replicas": ["node-jkl"]}
    --     ]
    --   }
    -- }

    -- Index configuration (JSONB because parameters vary by index type)
    index_config    JSONB NOT NULL DEFAULT '{}',
    -- HNSW example: {
    --   "type": "hnsw",
    --   "status": "ready",
    --   "params": {
    --     "m": 16,
    --     "ef_construction": 200,
    --     "ef_search": 100
    --   },
    --   "quantization": {
    --     "type": "scalar",
    --     "bits": 8,
    --     "rescoring": true
    --   },
    --   "build_started_at": "2026-05-20T10:00:00Z",
    --   "build_completed_at": "2026-05-20T10:45:00Z"
    -- }
    --
    -- IVF example: {
    --   "type": "ivf_pq",
    --   "status": "building",
    --   "params": {
    --     "nlist": 1024,
    --     "nprobe": 16,
    --     "pq_segments": 8,
    --     "pq_bits": 8
    --   }
    -- }
    --
    -- DiskANN example: {
    --   "type": "diskann",
    --   "status": "ready",
    --   "params": {
    --     "max_degree": 64,
    --     "search_list_size": 100,
    --     "pq_dims": 128
    --   }
    -- }

    -- Payload (metadata) schema for this collection
    payload_schema  JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "fields": {
    --     "category": {"type": "string", "indexed": true, "required": false},
    --     "price": {"type": "float", "indexed": true, "required": false},
    --     "tags": {"type": "string_array", "indexed": true},
    --     "location": {"type": "geo", "indexed": true},
    --     "created_date": {"type": "datetime", "indexed": true},
    --     "metadata": {"type": "json", "indexed": false}
    --   },
    --   "strict_mode": false
    -- }

    -- Embedding model configuration (if using built-in inference)
    embedding_config JSONB DEFAULT NULL,
    -- Example: {
    --   "provider": "openai",
    --   "model": "text-embedding-3-large",
    --   "dimension": 1536,
    --   "api_key_ref": "vault://embedding-keys/openai-prod",
    --   "batch_size": 100,
    --   "matryoshka_dim": null
    -- }

    -- Hybrid search configuration
    hybrid_config   JSONB DEFAULT NULL,
    -- Example: {
    --   "sparse_model": "bm25",
    --   "fusion_method": "rrf",
    --   "alpha": 0.7,
    --   "sparse_index_status": "ready"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, project_slug, name)
);
CREATE INDEX idx_collections_org ON collections(organization_id);
CREATE INDEX idx_collections_status ON collections(status);
CREATE INDEX idx_collections_dimension ON collections(dimension);
-- GIN index for querying by index type or deployment region
CREATE INDEX idx_collections_index_config ON collections
    USING GIN (index_config jsonb_path_ops);
CREATE INDEX idx_collections_deployment ON collections
    USING GIN (deployment jsonb_path_ops);

-- Namespaces within a collection
CREATE TABLE namespaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    vector_count    BIGINT NOT NULL DEFAULT 0,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (collection_id, name)
);
CREATE INDEX idx_namespaces_collection ON namespaces(collection_id);
```

### Querying JSONB Configuration

```sql
-- Find all collections using HNSW indexing
SELECT id, name, index_config
FROM collections
WHERE index_config @> '{"type": "hnsw"}';

-- Find all collections in EU regions
SELECT id, name, deployment->>'region' AS region
FROM collections
WHERE deployment @> '{"cloud_provider": "aws"}'
  AND deployment->>'region' LIKE 'eu-%';

-- Find collections with scalar quantization enabled
SELECT id, name,
       index_config->'quantization'->>'type' AS quant_type,
       index_config->'quantization'->>'bits' AS quant_bits
FROM collections
WHERE index_config->'quantization' @> '{"type": "scalar"}';

-- Find organizations with GDPR data residency requirements
SELECT id, name, data_governance->>'data_residency' AS residency
FROM organizations
WHERE data_governance @> '{"data_residency": "EU"}';

-- Find collections where a specific payload field is indexed
SELECT id, name
FROM collections
WHERE payload_schema->'fields'->'category'->>'indexed' = 'true';
```

## API Keys

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(8) NOT NULL,
    key_hash        VARCHAR(255) NOT NULL,
    role            VARCHAR(30) NOT NULL DEFAULT 'member',

    -- Scoping and rate limiting (JSONB for flexibility)
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "scopes": ["collections:read", "query"],
    --   "project_scope": "proj-a",
    --   "collection_ids": ["550e8400-...", "660e8400-..."],
    --   "rate_limit_rps": 200,
    --   "rate_limit_burst": 500,
    --   "allowed_ips": ["10.0.0.0/8", "192.168.1.0/24"],
    --   "expires_at": "2027-05-20T00:00:00Z"
    -- }

    last_used_at    TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_keys_org ON api_keys(organization_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
CREATE INDEX idx_api_keys_config ON api_keys USING GIN (config jsonb_path_ops);
```

## Usage Metering & Billing

```sql
-- Usage records (append-only, partitioned)
CREATE TABLE usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    collection_id   UUID REFERENCES collections(id),

    -- Core metric (relational for fast aggregation)
    metric_type     VARCHAR(30) NOT NULL
                    CHECK (metric_type IN ('read_units', 'write_units', 'storage_bytes',
                                           'query_count', 'upsert_count', 'delete_count',
                                           'inference_tokens')),
    quantity        BIGINT NOT NULL,

    -- Extended metrics (JSONB for flexibility as new metrics are added)
    details         JSONB DEFAULT '{}',
    -- Example for query: {
    --   "latency_ms": 12,
    --   "result_count": 10,
    --   "namespace": "production",
    --   "filter_complexity": "medium",
    --   "used_hybrid": true,
    --   "embedding_model": "text-embedding-3-large"
    -- }

    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_usage_org_metric ON usage_records(organization_id, metric_type, recorded_at);

-- Invoices
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    billing_period_start DATE NOT NULL,
    billing_period_end   DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'finalized', 'paid', 'overdue', 'void')),

    -- Line items and totals (JSONB for flexible line item structure)
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"description": "Pro plan base", "amount_cents": 2500, "quantity": 1},
    --   {"description": "Read units overage (450K)", "amount_cents": 1800, "quantity": 450000, "unit_price": 0.004},
    --   {"description": "Storage overage (5.2 GB)", "amount_cents": 520, "quantity": 5200, "unit": "MB"}
    -- ]

    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_ref     JSONB DEFAULT '{}',
    -- Example: {"stripe_invoice_id": "in_...", "paid_at": "2026-05-01T..."}

    issued_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_invoices_org ON invoices(organization_id);
```

## Audit Log

```sql
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_type      VARCHAR(20) NOT NULL CHECK (actor_type IN ('user', 'api_key', 'system')),
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,

    -- Full change details (JSONB captures arbitrary before/after)
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "python-sdk/2.1.0",
    --   "request_id": "req-abc-123",
    --   "changes": {
    --     "before": {"index_config": {"type": "hnsw", "params": {"m": 16}}},
    --     "after": {"index_config": {"type": "hnsw", "params": {"m": 32}}}
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org_time ON audit_logs(organization_id, created_at);
CREATE INDEX idx_audit_action ON audit_logs(action, created_at);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
```

## MCP Server & Integration Configuration

```sql
-- Integration configurations per organization (webhooks, MCP, etc.)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    integration_type VARCHAR(30) NOT NULL
                    CHECK (integration_type IN ('webhook', 'mcp_server', 'langchain',
                                                 'llamaindex', 'slack', 'prometheus')),

    -- Configuration varies entirely by integration type
    config          JSONB NOT NULL DEFAULT '{}',
    -- Webhook example: {
    --   "url": "https://api.acme.com/webhooks/vdb",
    --   "events": ["collection.ready", "index.build_completed", "usage.threshold"],
    --   "secret": "whsec_...",
    --   "retry_count": 3
    -- }
    --
    -- MCP server example: {
    --   "enabled": true,
    --   "allowed_tools": ["search", "upsert", "describe_collection"],
    --   "default_collection_id": "550e8400-...",
    --   "max_results": 20,
    --   "auth_method": "api_key"
    -- }
    --
    -- Prometheus example: {
    --   "enabled": true,
    --   "scrape_path": "/metrics",
    --   "labels": {"env": "production", "team": "ml-platform"}
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integrations_org ON integrations(organization_id);
CREATE INDEX idx_integrations_type ON integrations(integration_type);
```

## AI-Native Features: Index Tuning & Drift Detection

```sql
-- AI-driven index tuning recommendations
CREATE TABLE index_tuning_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    status          VARCHAR(20) NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'completed', 'failed')),

    -- Input: observed query patterns
    input_stats     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "query_count_7d": 150000,
    --   "avg_latency_ms": 18.5,
    --   "p99_latency_ms": 85,
    --   "avg_result_count": 10,
    --   "filter_usage_pct": 65,
    --   "top_filters": ["category", "date_range"],
    --   "recall_estimate": 0.95
    -- }

    -- Output: recommended parameter changes
    recommendations JSONB DEFAULT NULL,
    -- Example: {
    --   "current": {"m": 16, "ef_construction": 200, "ef_search": 100, "quantization": "none"},
    --   "recommended": {"m": 24, "ef_construction": 256, "ef_search": 128, "quantization": "scalar_8bit"},
    --   "expected_improvement": {
    --     "latency_reduction_pct": 15,
    --     "memory_reduction_pct": 40,
    --     "recall_impact": -0.002
    --   },
    --   "reasoning": "High filter usage suggests increasing M for better graph connectivity..."
    -- }

    applied_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tuning_collection ON index_tuning_runs(collection_id);

-- Embedding drift detection
CREATE TABLE drift_detection_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    status          VARCHAR(20) NOT NULL DEFAULT 'running',

    -- Drift analysis results
    results         JSONB DEFAULT NULL,
    -- Example: {
    --   "baseline_period": {"start": "2026-05-01", "end": "2026-05-07"},
    --   "current_period": {"start": "2026-05-14", "end": "2026-05-20"},
    --   "drift_score": 0.15,
    --   "drift_threshold": 0.10,
    --   "drift_detected": true,
    --   "affected_dimensions": [42, 87, 156],
    --   "probable_cause": "upstream_model_update",
    --   "alert_severity": "warning",
    --   "recommendation": "Re-embed vectors using current model version"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_drift_collection ON drift_detection_runs(collection_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organizations, users, organization_members |
| Collections & Indexes | 2 | collections, namespaces |
| API Keys | 1 | api_keys |
| Usage & Billing | 2 | usage_records (partitioned), invoices |
| Audit | 1 | audit_logs (partitioned) |
| Integrations | 1 | integrations |
| AI Features | 2 | index_tuning_runs, drift_detection_runs |
| **Total** | **~12** | Plus monthly partitions for usage_records and audit_logs |

---

## Key Design Decisions

1. **JSONB for variable configuration, relational for stable identity.** The `organizations`, `users`, and `organization_members` tables use relational columns for fields that are queried constantly (slug, email, role). Variable fields (billing details, SSO config, plan limits) live in JSONB because they change shape frequently and vary across tenants.

2. **Collection as a rich document.** The `collections` table packs index configuration, deployment topology, payload schema, embedding config, and hybrid search config into separate JSONB columns. Each column has a well-defined JSON Schema contract validated at the application layer. This means adding support for a new index type (e.g., DiskANN) requires zero schema migrations -- just a new config shape in the `index_config` JSONB.

3. **GIN indexes on JSONB columns.** The `jsonb_path_ops` GIN indexes on `collections.index_config`, `collections.deployment`, and `organizations.data_governance` enable efficient containment queries (`@>`) for filtering by index type, region, or compliance requirements.

4. **Invoice line items as JSONB arrays.** Billing line items vary by plan and overage type. Storing them as a JSONB array in the `invoices` table avoids a separate `invoice_line_items` table and keeps invoice rendering simple (just read the array).

5. **Integration configs as polymorphic JSONB.** Webhooks, MCP servers, Prometheus endpoints, and framework integrations all have completely different configuration shapes. A single `integrations` table with a `config` JSONB column handles all of them, discriminated by `integration_type`.

6. **AI feature results stored as JSONB.** Index tuning recommendations and drift detection results are complex, nested structures that change as the AI models improve. JSONB allows the recommendation format to evolve without migrations.

7. **Fewer tables, faster iteration.** At ~12 tables (versus ~27 in the normalized model), this schema is faster to understand, faster to migrate, and faster to build an MVP on. The trade-off is that data validation shifts from the database to the application layer.

8. **JSON Schema validation contract.** Every JSONB column should have a corresponding JSON Schema definition in the application code. This provides the same structural guarantees as relational constraints, just enforced at a different layer. The schemas should be versioned and published as part of the OpenAPI spec.
