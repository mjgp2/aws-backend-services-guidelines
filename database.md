---
title: Database
permalink: /database/
---

## 1. Default Choice

Aurora PostgreSQL is my default primary datastore for backend services.

I default to it because most startup backends need:

- strong transactional guarantees
- relational integrity
- predictable query semantics
- mature tooling and operator familiarity
- enough flexibility to support both product data and workflow coordination

Aurora also gives a practical managed-service upside: storage and compute are
decoupled, scaling readers and writers is easier than with self-managed
PostgreSQL, and zero-ETL integrations give you a cleaner path into analytics
without bolting bespoke pipelines onto the transactional database.

A different datastore should be chosen only when the workload clearly does not
fit PostgreSQL.

## 2. What The Database Is For

Use Aurora PostgreSQL for:

- source-of-truth relational data
- transactional workflow state
- idempotency and uniqueness enforcement
- audit and compliance metadata
- bounded metadata for objects stored elsewhere

Do not use PostgreSQL as:

- an object store for large files
- a general-purpose event bus
- an unbounded cache for arbitrarily shaped payloads

## 3. Schema Design Rules

### 3.1 Model ownership and access patterns explicitly

- Put tenant, account, or ownership columns on rows where ownership matters.
- Design keys and indexes around real query patterns, not theoretical purity.
- Make high-risk uniqueness guarantees explicit in the schema.

If the product is multi-tenant, the data model must make tenant scoping easy to
enforce and easy to query.

Authorization and isolation rules:

- authorization-scoped queries should start from owned or tenant-scoped rows,
not fetch broadly and filter late in application code
- keep tenant or owner columns in the keys and indexes that serve primary read
and mutation paths
- do not treat object hashes, object keys, or other storage identifiers as a
  substitute for resource authorization

### 3.2 Separate operational state from historical state

- Keep hot workflow tables small.
- Move append-only history into dedicated audit or event tables.
- Use retention and archive policies deliberately.

Operational coordination data and historical reporting data should not fight
over the same indexes and vacuum budget.

### 3.3 Prefer append-only history and HOT-friendly live state

- favor append-only tables for audit history, immutable events, and immutable
derived records
- favor HOT-friendly updates for mutable operational state that genuinely needs
in-place change
- do not distort the domain model just to chase HOT updates

Use append-only design when the data is historical by nature. Use mutable
current-state tables when the system needs a directly queryable source of truth
for the latest status.

For mutable hot tables:

- keep rows narrow
- avoid indexing churn-heavy columns unless necessary
- tune fillfactor and maintenance for update-heavy paths
- move historical detail into append-only side tables when possible

### 3.4 Partition intentionally, and usually plan it up front

If a table is likely to need partitioning, bias toward designing and often
implementing it up front.

Design-time checks:

- identify which tables could become very large or retention-heavy
- identify whether access is naturally tenant-scoped, time-scoped, or
identity-scoped
- decide whether the likely future partition key should already appear in the
primary read paths, ownership model, and index strategy
- avoid schema and query patterns that would make later repartitioning
unnecessarily painful

Retrofitting partitioning later is often painful because the partition key
usually leaks into primary keys, uniqueness rules, foreign keys, query shapes,
backfill plans, and operational runbooks. Do not assume it will be a cheap
future cleanup.

Partition when:

- table growth is expected to be very large
- access patterns align with a stable partition key
- retention or archival operations benefit from partition boundaries

Do not partition just because a table might eventually become large. Partition
because the access pattern or lifecycle justifies it.

Practical bias:

- if a table is likely to become large, retention-heavy, or tenant- or
  time-sharded, put the partition key into the design from day one
- if the table is very likely to need partitioning later, default to doing it
  early rather than betting on an easy retrofit
- keep obviously small, bounded operational tables unpartitioned until there is
  a real reason to add the complexity

Useful examples:

- tenant-key partitioning for large tenant-scoped entities
- time partitioning for append-only audit or event history
- identity-based partitioning for very large deduplicated object families

Practical rule:

- small and medium operational tables should usually stay unpartitioned at first
- large history, audit, event, or retention-managed tables should be evaluated
for partitioning during the initial schema design, not after they are already
painful to operate

Operational side-table pattern:

- if the source-of-truth table is large, partitioned, and history-heavy, do not
  force operational workers to discover “what is waiting now?” by scanning that
  full table
- keep a small side table for the hot operational subset, for example rows that
  are ready, pending, retryable, or otherwise actionable
- update that side table transactionally with the source-of-truth state change
  so workers can query a bounded hot set while the full partitioned table
  remains optimized for retention and history

### 3.5 Use the database to enforce correctness

Prefer schema-level protection for:

- uniqueness
- foreign-key integrity where the operational cost is acceptable
- valid enum-like state domains
- check constraints on critical invariants

If correctness matters, do not leave it as an application convention only.

### 3.6 Keep rows and indexes disciplined

- avoid very wide hot rows
- avoid indexing columns that churn constantly unless necessary
- use JSONB sparingly and intentionally
- add indexes only for known read or write paths

Every index is a tax on writes, storage, vacuum, and operational complexity.

Good JSONB use cases:

- sparse or provider-specific metadata that does not justify a full relational
  model yet
- captured request, response, or audit payload fragments that are read
  occasionally but not central to query planning
- bounded ingest buffers where important fields will be extracted into normal
  columns once the contract settles

Bad JSONB use cases:

- core relational fields that need joins, uniqueness, or frequent filtering
- status, ownership, or lifecycle fields that define the main operational query
  paths
- dumping arbitrary application state into one opaque column because schema
  design feels inconvenient

## 4. Aurora PostgreSQL Operational Posture

### 4.0 Ownership boundary

Database ownership needs two levels to stay clear:

- platform/cloud owns the database platform: cluster shape, engine posture,
  backup and restore standards, auth model, connection-management defaults,
  shared observability, and the delivery guardrails around migrations
- product teams own service-local database behavior: schema design, indexes,
  queries, data lifecycle choices, migration contents, and the correctness and
  cost of the workload they run on that allocation

The default model in these docs is a shared Aurora cluster per environment or
domain group, with service-owned allocations inside it. Shared cluster does
not mean shared schema ownership.

Normal boundary rules:

- each service should have its own database allocation, app user, owner user,
  and migration artifact
- a product team should be able to evolve its own schema within the platform
  guardrails without opening a standing ticket to another team
- material schema changes should normally receive database peer review from
  platform/cloud when that team carries the strongest scalability and
  operational expertise for the shared database estate
- platform should own the standards and reusable tooling, not the routine
  content of every product migration
- cross-service direct table writes are the exception case and should be
  treated as an explicit architectural decision, not a convenience default
- if one service's schema or query behavior can materially harm the shared
  cluster, platform gets to define and enforce the safety bar

### 4.1 Treat the database as a precious shared dependency

- use connection pooling
- use RDS Proxy, PgBouncer, or an equivalent pooling layer when runtime
  connection fan-out would otherwise put avoidable pressure on Aurora
- set per-query or per-operation timeouts
- keep transactions short
- avoid holding transactions open during network or object-storage I/O
- measure slow queries early
- prefer IAM-authenticated runtime connections in AWS when the platform already
supports them; keep long-lived admin credentials limited to bootstrap,
emergency access, or other cases that cannot use IAM auth

Aurora scales well, but it is still easy to overwhelm with poor query or
connection behavior.

### 4.2 Scaling defaults

- default to a shared Aurora cluster per environment or domain group for small
service fleets, with service-owned database allocations, users, and migration
ownership inside that cluster
- start with one writer and one replica (HA)
- for read-heavy paths that tolerate replica lag, prefer Aurora reader auto
  scaling before treating replica count as a manual constant
- scale reads with replicas only for paths that tolerate replica lag
- keep read-routing explicit; do not send correctness-critical reads to replicas
  unless the product semantics clearly tolerate stale data
- when read traffic splits into materially different classes, a valid Aurora
  pattern is to put different replicas behind different custom endpoints, for
  example one subset for production reads and another for analytics or operator
  traffic
- use replicas to protect the writer from analytics, read-heavy APIs, and
  operator/reporting traffic before scaling the writer forever
- measure replica lag and fail back to the writer or degrade gracefully when
  lag breaks the service contract
- prefer query and index fixes before buying larger instances forever
- use partitioning and retention to control long-term table growth
- keep queue or outbox tables operationally small

A good shared-infrastructure setup follows this model: each service owns its
own database allocation, app user, owner user, and migration stack inside a
shared Aurora cluster rather than defaulting to one cluster per service.

That is the key boundary: platform can own the shared cluster and its
guardrails, but product still owns the service-local schema and the migrations
that change it.

### 4.3 Migrations

- migrations must be forward-only and automated
- prefer additive changes before destructive ones
- assume mixed-version deploy windows during rollout
- apply the same mixed-version compatibility rule described in
  [Delivery]({{ '/delivery/' | relative_url }}): if older runtimes may still be handling traffic,
  jobs, or rollbacks, the migrated schema must remain backward-compatible
  enough for them to keep operating
- backfills need explicit throttling and observability
- treat the migration tool and migration set as release artifacts, not only as
  source files in the repo
- a dedicated migration image or package is a good default when services are
  deployed from immutable images, because it makes the schema step versioned,
  auditable, and promotable through environments
- Sqitch is a reasonable option when the team wants database-native change
  management with explicit deploy, revert, and verify steps, but the exact tool
  matters less than having a disciplined forward-only delivery model in normal
  rollout paths

If a migration cannot be run safely while older runtimes still exist, it needs
a runbook and likely a phased design. Normal deploys should not depend on a
brief downtime window just because the schema changed.

### 4.4 Audit and retention

- define retention periods explicitly
- keep audit history append-only
- partition historical tables when retention windows are time-based
- verify maintenance jobs and partition creation in production

Retention is an architecture decision, not only a compliance detail.

### 4.5 Data deletion and lifecycle enforcement

Database deletion policy should be explicit, automated, and reviewable.

Default rules:

- temporary operational rows should expire automatically
- historical rows should age out through partition drops or bounded delete jobs
- user-facing deletion requests should define whether the result is immediate
hard delete, soft delete, or tombstoning followed by asynchronous cleanup
- derived data should be recreated or deleted based on source-of-truth policy,
not retained forever by accident

Prefer:

- partition drops for large time-bounded history
- bounded background delete jobs over huge ad hoc `DELETE` statements
- side tables for short-lived operational state that should be cleaned up fast
- explicit tombstones only when they solve a real product or consistency need

Be careful with:

- soft deletes on every table by default
- cascading deletes across very large relational graphs
- orphaned rows in secondary tables after product-level deletion flows
- retaining "temporary" rows because no cleanup worker owns them

### 4.6 Backup, restore, and cluster recovery

Database recovery posture should be explicit and practiced.

Default rules:

- define expected RPO and RTO for the database, not only for the service as a
whole
- keep automated backups enabled in every non-ephemeral environment
- require deletion protection or equivalent safeguards on canonical production
clusters
- rehearse restore of schema, roles, and service bootstrap, not only raw engine
recovery

Example baseline:

- shared Aurora clusters are configured with deletion protection
- `staging` can reasonably start with `7` days of backup retention
- `production` can reasonably start with `14` days of backup retention
- staging clusters snapshot on stack removal; production clusters retain on
  stack removal

If a team cannot say how to restore a service-owned database allocation into a
working runtime, the operational design is incomplete.

## 5. Default Patterns For High-Scale Services

Recommended defaults:

- transactional outbox for reliable async publication
- uniqueness constraints to make enqueue or create operations idempotent
- side tables for bounded operational discovery paths instead of global scans
- append-only audit trail for important state transitions
- object bytes in S3 and metadata in PostgreSQL
- lifecycle jobs or partition policies for deleting expired data

Detailed S3 policy lives in [S3]({{ '/s3/' | relative_url }}).

## 6. When Aurora PostgreSQL Is Not The Right Default

Consider another datastore when the dominant workload is:

- simple key-value lookups at extremely high scale with little relational logic
- event-log or stream-native processing
- graph-native traversal as the central product primitive
- search-heavy access that really belongs in a search engine

Even then, PostgreSQL may still remain the control-plane or metadata store.

## 6.1 Redis and ElastiCache pattern: cache, ephemeral state, and coordination

If a service needs very fast key-value access, short-lived coordination state,
or cache-backed read shedding, Redis through ElastiCache is a good companion
pattern.

Use Redis for cache, short-lived coordination, rate limits, sessions, and
other state you can rebuild or afford to lose. Do not use it for durable
product state, relational integrity, or anything that falls apart when cache
entries disappear during eviction or failover. The mistake teams keep making is
turning Redis into an unowned dumping ground for arbitrary application state
and then acting surprised when missing invalidation rules become a production
incident.

Keep Aurora PostgreSQL as the canonical source of truth unless the workload
clearly says otherwise, and define cache fill, expiry, and invalidation rules
up front instead of treating Redis as magic shared memory.

## 6.2 Reporting and analytics pattern: Aurora zero-ETL to Redshift

If a service develops serious reporting or analytics requirements, prefer
keeping Aurora PostgreSQL as the transactional source of truth and consider
Aurora zero-ETL integration into Amazon Redshift before inventing bespoke CDC
pipelines or pushing analytical workloads onto the primary database.

Use zero-ETL to Redshift when reporting has turned into read-heavy
aggregation, dashboard load, warehouse-style joins, or historical analysis
that should not be beating on the transactional database. Keep Aurora as the
operational system of record and move only the analytical pressure.

This is not the default for ordinary services. If product-serving queries still
fit inside Aurora with normal indexes and read replicas, stay there. If the
replication path needs in-flight transformation, the source tables do not have
stable primary keys, the replicated Redshift database needs to be writable, the
team is not ready to run a second data platform, or cross-region analytics
replication is a hard requirement on day one, this is the wrong tool. Keep the
replicated scope narrow and deliberate; Redshift is not your second
operational database.

## 6.3 Search pattern: OpenSearch as a derived search system

If the product needs real full-text search, faceting, relevance tuning, or
search-heavy read patterns that no longer fit PostgreSQL cleanly, OpenSearch is
a reasonable companion system.

Use OpenSearch as a derived search layer, not as the canonical source of truth.

- keep PostgreSQL as the operational system of record unless the product
  genuinely centers on search-engine-native behavior
- feed OpenSearch from explicit indexing pipelines or events rather than
  treating it as the place where writes originate
- keep index mappings, reindex strategy, and backfill posture explicit
- be honest about consistency: most OpenSearch-backed product search is
  eventually consistent, and the product needs to tolerate that
- do not move routine product lookups into OpenSearch just because it exists

The mistake to avoid is building a second primary datastore by accident. Use
OpenSearch when search quality or search scale is the real requirement, not as
a vague escape hatch from relational design.
