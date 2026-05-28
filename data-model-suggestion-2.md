# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Vector Database as a Service · Created: 2026-05-20

## Philosophy

This model treats every mutation to the system as an immutable event appended to a single event store. The current state of any entity (organization, collection, index, API key) is derived by replaying its event stream. Materialized read models (projections) are maintained asynchronously for fast querying -- this is the CQRS (Command Query Responsibility Segregation) pattern.

The motivation is twofold. First, a vector database service operating in regulated environments (GDPR, HIPAA, SOC 2) needs a complete, tamper-evident audit trail of every change. Event sourcing provides this as a first-class architectural property rather than a bolted-on logging layer. Second, the platform needs to answer temporal questions: "What was the collection's index configuration on March 15th?", "When was this API key last rotated?", "What was the organization's plan tier when this usage spike occurred?" -- all of which are trivial with event replay but require complex bi-temporal modeling in a traditional relational schema.

Real-world precedents include event-sourced banking ledgers, Marten (PostgreSQL event store for .NET), and AWS EventBridge-backed SaaS architectures. The event store itself uses PostgreSQL for durability and ACID guarantees on event appends, while projections can target PostgreSQL read tables, Redis for hot caches, or Elasticsearch for search.

**Best for:** Teams that require full audit trails for compliance, need temporal queries across the lifecycle of every entity, or plan to build AI-powered analytics on change patterns (e.g., detecting anomalous usage before billing disputes).

**Trade-offs:**
- Pro: Complete, immutable audit trail -- every state change is recorded forever
- Pro: Temporal queries are trivial (replay to any point in time)
- Pro: Event replay enables schema evolution without data migration
- Pro: Natural fit for usage metering (usage events are already append-only)
- Pro: Decoupled read models can be independently optimized and rebuilt
- Con: Higher write amplification (event + projection update per mutation)
- Con: Eventual consistency between event store and projections
- Con: More complex application code (command handlers, event handlers, projectors)
- Con: Event schema evolution requires careful versioning (upcasting)
- Con: Aggregate reconstruction from thousands of events can be slow without snapshots

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SOC 2 Type II | Event store IS the audit log -- 100% mutation coverage by design |
| ISO/IEC 27001:2022 | Immutable event stream satisfies information security audit requirements |
| GDPR (EU) 2016/679 | Right-to-erasure modeled as a `DataRedacted` event with crypto-shredding |
| HIPAA | BAA compliance via encrypted event payloads and immutable access records |
| OWASP API Security | Every API mutation recorded with actor, IP, request context |
| NIST SP 800-207 | Zero-trust audit: every action verified and logged regardless of origin |

---

## Event Store (Core)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,
    -- e.g. 'Organization', 'Collection', 'ApiKey', 'Index', 'User'
    aggregate_id    UUID NOT NULL,
    sequence_num    BIGINT NOT NULL,
    -- Monotonically increasing per aggregate; used for optimistic concurrency
    event_type      VARCHAR(100) NOT NULL,
    -- e.g. 'OrganizationCreated', 'CollectionDimensionChanged',
    --      'ApiKeyRevoked', 'IndexBuildCompleted'
    event_version   SMALLINT NOT NULL DEFAULT 1,
    -- Schema version of this event type (for upcasting)
    payload         JSONB NOT NULL,
    -- Event-specific data (see examples below)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Context: user_id, api_key_id, ip_address, request_id, user_agent
    organization_id UUID NOT NULL,
    -- Denormalized for partition pruning and tenant isolation
    caused_by       UUID,
    -- event_id of the command/event that triggered this event
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, sequence_num)
) PARTITION BY RANGE (created_at);

-- Partition per month for retention management
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_num);
CREATE INDEX idx_events_org ON events(organization_id, created_at);
CREATE INDEX idx_events_type ON events(event_type, created_at);

-- Aggregate version tracking for optimistic concurrency control
CREATE TABLE aggregate_versions (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    current_version BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- Snapshots for aggregates with many events (performance optimization)
CREATE TABLE aggregate_snapshots (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,
    -- Serialized aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, version)
);
```

### Event Payload Examples

```sql
-- OrganizationCreated
-- payload: {
--   "name": "Acme Corp",
--   "slug": "acme-corp",
--   "billing_email": "billing@acme.com",
--   "plan_id": "550e8400-...",
--   "plan_tier": "pro"
-- }

-- CollectionCreated
-- payload: {
--   "collection_name": "product-embeddings",
--   "project_id": "550e8400-...",
--   "dimension": 1536,
--   "distance_metric": "cosine",
--   "region": "us-east-1",
--   "cloud_provider": "aws",
--   "shard_count": 2,
--   "replica_count": 1
-- }

-- IndexBuildStarted
-- payload: {
--   "collection_id": "550e8400-...",
--   "index_type": "hnsw",
--   "hnsw_m": 16,
--   "hnsw_ef_construction": 200,
--   "quantization": "scalar",
--   "quant_bits": 8,
--   "vector_count_at_start": 1500000
-- }

-- ApiKeyCreated
-- payload: {
--   "key_name": "production-rag-pipeline",
--   "key_prefix": "vdb_pk_8x",
--   "key_hash": "$argon2id$...",
--   "scopes": ["collections:read", "query"],
--   "project_id": "550e8400-...",
--   "rate_limit_rps": 200,
--   "expires_at": "2027-05-20T00:00:00Z"
-- }

-- UsageRecorded
-- payload: {
--   "collection_id": "550e8400-...",
--   "metric_type": "read_units",
--   "quantity": 150,
--   "query_latency_ms": 12,
--   "result_count": 10
-- }

-- DataRedactionRequested (GDPR right-to-erasure)
-- payload: {
--   "scope": "collection",
--   "collection_id": "550e8400-...",
--   "reason": "GDPR Article 17 request",
--   "requested_by_user_id": "550e8400-...",
--   "redaction_key_id": "crypto-shred-key-42"
-- }
```

## Projections (Materialized Read Models)

```sql
-- ============================================================
-- PROJECTION: Organizations (current state)
-- Rebuilt by replaying Organization* events
-- ============================================================
CREATE TABLE proj_organizations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    plan_id         UUID,
    plan_tier       VARCHAR(20),
    status          VARCHAR(20) NOT NULL,
    member_count    INTEGER NOT NULL DEFAULT 0,
    collection_count INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    -- Tracks the last processed event for idempotent projection updates
    last_event_at   TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- PROJECTION: Collections (current state)
-- Rebuilt by replaying Collection* events
-- ============================================================
CREATE TABLE proj_collections (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    dimension       INTEGER NOT NULL,
    distance_metric VARCHAR(30) NOT NULL,
    region_code     VARCHAR(30) NOT NULL,
    cloud_provider  VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    vector_count    BIGINT NOT NULL DEFAULT 0,
    storage_bytes   BIGINT NOT NULL DEFAULT 0,
    shard_count     INTEGER NOT NULL DEFAULT 1,
    replica_count   INTEGER NOT NULL DEFAULT 1,
    index_type      VARCHAR(30),
    index_status    VARCHAR(20),
    quantization    VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_collections_org ON proj_collections(organization_id);
CREATE INDEX idx_proj_collections_project ON proj_collections(project_id);

-- ============================================================
-- PROJECTION: API Keys (current state, excluding revoked)
-- ============================================================
CREATE TABLE proj_api_keys (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    project_id      UUID,
    name            VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(8) NOT NULL,
    key_hash        VARCHAR(255) NOT NULL,
    scopes          TEXT[] NOT NULL,
    rate_limit_rps  INTEGER,
    expires_at      TIMESTAMPTZ,
    is_revoked      BOOLEAN NOT NULL DEFAULT false,
    revoked_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_keys_org ON proj_api_keys(organization_id);
CREATE INDEX idx_proj_keys_prefix ON proj_api_keys(key_prefix);

-- ============================================================
-- PROJECTION: Users & Memberships (current state)
-- ============================================================
CREATE TABLE proj_users (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    auth_provider   VARCHAR(50),
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_memberships (
    organization_id UUID NOT NULL,
    user_id         UUID NOT NULL,
    role            VARCHAR(50) NOT NULL,
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (organization_id, user_id)
);

-- ============================================================
-- PROJECTION: Usage Dashboard (hourly aggregates)
-- Rebuilt by replaying UsageRecorded events with time bucketing
-- ============================================================
CREATE TABLE proj_usage_hourly (
    organization_id UUID NOT NULL,
    collection_id   UUID,
    metric_type     VARCHAR(30) NOT NULL,
    hour_bucket     TIMESTAMPTZ NOT NULL,
    total_quantity  BIGINT NOT NULL DEFAULT 0,
    event_count     INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  FLOAT,
    PRIMARY KEY (organization_id, collection_id, metric_type, hour_bucket)
);
CREATE INDEX idx_usage_hourly_time ON proj_usage_hourly(hour_bucket);

-- ============================================================
-- PROJECTION: Billing Summary (monthly aggregates)
-- ============================================================
CREATE TABLE proj_billing_monthly (
    organization_id UUID NOT NULL,
    billing_month   DATE NOT NULL,
    plan_id         UUID,
    plan_tier       VARCHAR(20),
    read_units      BIGINT NOT NULL DEFAULT 0,
    write_units     BIGINT NOT NULL DEFAULT 0,
    storage_byte_hours BIGINT NOT NULL DEFAULT 0,
    query_count     BIGINT NOT NULL DEFAULT 0,
    inference_tokens BIGINT NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (organization_id, billing_month)
);

-- ============================================================
-- PROJECTION: Cluster Infrastructure (current state)
-- ============================================================
CREATE TABLE proj_cluster_nodes (
    id              UUID PRIMARY KEY,
    region_code     VARCHAR(30) NOT NULL,
    cloud_provider  VARCHAR(20) NOT NULL,
    hostname        VARCHAR(255) NOT NULL,
    node_type       VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    capacity_vectors BIGINT,
    used_vectors    BIGINT,
    last_heartbeat  TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL
);
```

## Projection Tracking

```sql
-- Tracks the checkpoint of each projection for replay/rebuild
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    -- e.g. 'proj_organizations', 'proj_usage_hourly'
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'rebuilding', 'paused', 'error')),
    error_message   TEXT,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that fail projection processing
CREATE TABLE projection_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_dead_letters_projection ON projection_dead_letters(projection_name);
```

## GDPR Crypto-Shredding Support

```sql
-- Encryption keys per organization for crypto-shredding
-- When a GDPR deletion is requested, the key is destroyed,
-- rendering all encrypted event payloads for that org unreadable
CREATE TABLE encryption_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    key_purpose     VARCHAR(30) NOT NULL
                    CHECK (key_purpose IN ('event_payload', 'pii_fields', 'api_key')),
    encrypted_key   BYTEA NOT NULL,
    -- The actual encryption key, wrapped with a KMS master key
    key_version     INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    destroyed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, key_purpose, key_version)
);
CREATE INDEX idx_enc_keys_org ON encryption_keys(organization_id);
```

---

## Example Queries

### Reconstruct collection state at a specific point in time

```sql
-- What was collection X's configuration on March 15, 2026?
SELECT payload
FROM events
WHERE aggregate_type = 'Collection'
  AND aggregate_id = '550e8400-...'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY sequence_num ASC;

-- Application code replays these events to reconstruct the state
```

### Audit trail for a specific API key

```sql
SELECT event_type, payload, metadata, created_at
FROM events
WHERE aggregate_type = 'ApiKey'
  AND aggregate_id = '550e8400-...'
ORDER BY sequence_num ASC;

-- Returns: ApiKeyCreated -> ApiKeyScopesUpdated -> ApiKeyUsed -> ApiKeyRevoked
```

### Usage analytics with time bucketing

```sql
-- Hourly query volume for the last 7 days (from projection)
SELECT hour_bucket, total_quantity, avg_latency_ms
FROM proj_usage_hourly
WHERE organization_id = '550e8400-...'
  AND metric_type = 'query_count'
  AND hour_bucket >= now() - INTERVAL '7 days'
ORDER BY hour_bucket;
```

### Detect anomalous usage patterns (AI-native feature)

```sql
-- Find organizations whose write volume spiked >5x in the last hour
-- compared to their 7-day average
WITH recent AS (
    SELECT organization_id, SUM(total_quantity) AS last_hour
    FROM proj_usage_hourly
    WHERE metric_type = 'write_units'
      AND hour_bucket >= now() - INTERVAL '1 hour'
    GROUP BY organization_id
),
baseline AS (
    SELECT organization_id,
           AVG(total_quantity) AS avg_hourly
    FROM proj_usage_hourly
    WHERE metric_type = 'write_units'
      AND hour_bucket >= now() - INTERVAL '7 days'
      AND hour_bucket < now() - INTERVAL '1 hour'
    GROUP BY organization_id
)
SELECT r.organization_id, r.last_hour, b.avg_hourly,
       r.last_hour / NULLIF(b.avg_hourly, 0) AS spike_ratio
FROM recent r
JOIN baseline b ON r.organization_id = b.organization_id
WHERE r.last_hour > b.avg_hourly * 5;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (core) | 3 | events (partitioned), aggregate_versions, aggregate_snapshots |
| Projections - Identity | 3 | proj_organizations, proj_users, proj_memberships |
| Projections - Data Plane | 2 | proj_collections, proj_api_keys |
| Projections - Usage/Billing | 2 | proj_usage_hourly, proj_billing_monthly |
| Projections - Infrastructure | 1 | proj_cluster_nodes |
| Projection Management | 2 | projection_checkpoints, projection_dead_letters |
| Security / GDPR | 1 | encryption_keys |
| **Total** | **~14** | Plus monthly partitions for the events table |

---

## Key Design Decisions

1. **Single event table, not per-aggregate.** All events share one partitioned table. This simplifies backup, replication, and cross-aggregate queries (e.g., "show all events for organization X"). The `aggregate_type` + `aggregate_id` + `sequence_num` unique constraint provides per-aggregate ordering.

2. **Optimistic concurrency via sequence numbers.** Before appending an event, the command handler checks that the expected `sequence_num` matches `aggregate_versions.current_version`. This prevents lost updates without pessimistic locking.

3. **Projections are disposable.** Every `proj_*` table can be dropped and rebuilt by replaying events from the event store. The `projection_checkpoints` table tracks replay progress. This makes schema changes to read models trivial -- add a column, rebuild the projection.

4. **Crypto-shredding for GDPR.** Instead of deleting events (which would break the append-only guarantee), PII-containing event payloads are encrypted with per-organization keys. A GDPR deletion request triggers destruction of the encryption key, rendering the payloads unreadable while preserving the event sequence for audit purposes.

5. **Usage events are first-class.** `UsageRecorded` events flow through the same event store as administrative events. The `proj_usage_hourly` and `proj_billing_monthly` projections aggregate these into billing-ready summaries. This ensures billing is always reconcilable against the raw event stream.

6. **Snapshots for hot aggregates.** Organizations or collections with thousands of events use periodic snapshots (`aggregate_snapshots`) to avoid replaying the full history on every read. Snapshots are created every N events (e.g., every 100).

7. **Metadata envelope for context.** Every event carries a `metadata` JSONB field with actor identity (user_id, api_key_id), network context (ip_address, user_agent), and correlation (request_id). This metadata is not part of the domain event but is essential for security auditing.

8. **Dead letter queue for projection failures.** If a projector fails to process an event, it is written to `projection_dead_letters` for investigation rather than blocking the entire projection pipeline. This provides resilience without sacrificing auditability.
