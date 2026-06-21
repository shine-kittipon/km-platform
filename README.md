# OpenMetadata Cookbook for the KM Platform

**Purpose:** Practical architectural guide mapping OpenMetadata concepts, patterns, and capabilities to our Enterprise Knowledge Management Platform design.  
**Audience:** Platform architects and engineers building the KM Platform  
**Date:** 2026-06-21  
**Scope:** Architecture, concepts, and trade-offs — not installation or UI walkthroughs  
**References:** OpenMetadata v1.12+, OpenMetadata Standards v1.13.0

---

## 1. Executive Summary

### What OpenMetadata Is

OpenMetadata is an open-source metadata platform that originated from Uber's Databook project. It positions itself as the "Open Context Layer for Data and AI" — a centralized metadata graph that connects technical metadata, data quality signals, lineage, ownership, classifications, glossary terms, policies, data contracts, and (more recently) memory nuggets into a unified knowledge graph. It supports 130+ connectors for ingesting metadata from databases, data warehouses, BI tools, pipelines, ML platforms, messaging systems, storage, APIs, and SaaS applications.

### Core Philosophy

OpenMetadata operates on three principles that align closely with our platform's design:

**Schema-first architecture.** Every entity, relationship, and API contract is defined by JSON Schema specifications (700+ schemas). These schemas are the source of truth — the API layer, Java backend, and Python ingestion framework are all generated from them. This eliminates drift between what the system stores and what it exposes.

**Metadata only, not content.** Like our platform, OpenMetadata does not replicate source data. It harvests metadata — schemas, owners, lineage, quality signals, classifications — and stores that metadata in a centralized graph. The actual data stays in source systems.

**Unified metadata graph.** Rather than treating metadata as isolated records, OpenMetadata links everything into a graph: assets connect to owners, owners belong to teams, teams map to domains, domains contain glossary terms, glossary terms connect to policies, policies govern classifications. This graph structure is what makes traversal and contextual discovery possible.

### Why It Matters for AI/Agent-Based KM Systems

OpenMetadata has explicitly pivoted toward AI and agent consumption. Its recent releases (v1.8+) introduced a native MCP (Model Context Protocol) server, an AI SDK, "memory nuggets" (governed knowledge fragments tied to assets), and semantic search capabilities. The platform's stated position is that "AI does not need another raw database connector — AI needs context + memory."

This matters for our platform because OpenMetadata has already solved several problems we need to solve: how to model diverse asset types under a single schema, how to ingest metadata from heterogeneous sources, how to attach governance and lineage to every entity, and how to expose that context to AI agents via structured APIs and MCP. We do not need to reinvent these patterns. We need to understand which ones to adopt, which to simplify, and which to extend for our broader knowledge scope.

### Critical Scope Distinction

OpenMetadata is fundamentally a **data catalog** — its entity model, connectors, and lineage are optimized for structured data assets: tables, columns, pipelines, dashboards, ML models, and topics. Our KM platform must handle a broader knowledge spectrum: documents, wikis, knowledge articles, API specifications, business processes, and semantic concepts that do not fit neatly into OpenMetadata's data-centric entity hierarchy. This distinction runs through every section of this cookbook.

---

## 2. Metadata Model

### Entity Model

OpenMetadata defines every metadata object as a typed entity with a JSON Schema contract. The core entity types most relevant to our platform:

| OpenMetadata Entity | Description | KM Platform Equivalent |
|---|---|---|
| Table | Database table with columns, schema, constraints | Data asset (type: table) in Metadata Store (3.1) |
| Dashboard | BI dashboard with charts and data sources | Knowledge asset (type: dashboard) |
| Pipeline | ETL/ELT pipeline with tasks and schedule | Not primary — pipeline metadata is lineage context |
| Topic | Kafka/messaging topic with schema | Knowledge asset (type: event_stream) if needed |
| ML Model | Machine learning model with features, algorithm | Knowledge asset (type: model) if needed |
| API Endpoint | REST/GraphQL endpoint with request/response | Knowledge asset (type: api) |
| Container | Cloud storage container (S3, GCS, ADLS) | Knowledge asset (type: storage) |
| Glossary Term | Business term with definition, synonyms, related terms | Glossary term in Enterprise Knowledge Semantic Layer (3.16) |
| Domain | Business domain grouping | Domain node in Knowledge Graph (3.14) |
| Data Product | Curated, governed data offering | Not Phase 1 — but conceptually similar to governed knowledge packages |
| User / Team | People and organizational structure | Ownership Registry (3.6) |
| Policy | Access control rules | Policy Store (3.7) |
| Metric | Business metric definition | Analytics Semantic Layer (3.17) — MetricFlow-scoped |
| Data Contract | Schema, quality, freshness guarantees | Maps to Governance Engine sub-components (3.5–3.9) |

**Key architectural insight:** OpenMetadata collapses what our architecture separates. In OpenMetadata, an entity's metadata record, its semantic annotations (tags, glossary terms), its governance attributes (owner, classification), and its graph relationships all live on the same Entity object. In our architecture, these are distributed across the Metadata Store (3.1), Enterprise Knowledge Semantic Layer (3.16), Governance Engine (3.5–3.9), and Knowledge Graph (3.13–3.15). Neither approach is inherently wrong — OpenMetadata's unified model is simpler to operate; our separated model is more flexible for independent evolution of each concern.

### Fully Qualified Names (FQN)

OpenMetadata uses a hierarchical naming convention as the primary identifier for every entity: `service.database.schema.table.column`. This is analogous to our platform URI scheme (`km://assets/uuid-abc`) but serves a different purpose — FQNs encode the containment hierarchy (a column belongs to a table, which belongs to a schema, which belongs to a database, which belongs to a service), while our platform URIs are flat identifiers with mappings managed by the Reference Registry (3.2).

**Recommendation:** Adopt the FQN concept for source-system-native references (it makes connector development simpler), but retain our platform URI scheme for stable cross-system identification. Map between the two in the Reference Registry.

### Relationships

OpenMetadata stores relationships as edges in its entity graph. Relationship types include: `contains`, `owns`, `uses`, `appliedTo`, `has`, `upstream`, `downstream`, `relatedTo`, and custom relationship types. Relationships are first-class citizens with their own properties — they can carry attributes like `lineageDetails` (for data lineage edges) or `confidence` scores.

This maps directly to our Knowledge Graph Service (3.13–3.15), which stores relationships as typed edges with properties including confidence scores and extraction methods. The key difference: OpenMetadata's relationships are tightly coupled to its entity store (MySQL/Postgres), whereas our design separates the relational metadata (Metadata Store in PostgreSQL) from the graph structure (Graph Store in Neo4j/Neptune).

### Glossary and Taxonomy

OpenMetadata's glossary model is hierarchical: Glossary → Glossary Term → child Glossary Terms. Each term has a definition, synonyms, related terms, reviewers, and can be tagged to any entity. Terms go through an approval workflow (Draft → Approved → Deprecated). Multiple glossaries can coexist for different business domains.

This maps well to our Enterprise Knowledge Semantic Layer (3.16), which manages the business glossary, taxonomy, and ontology. OpenMetadata's glossary is simpler than what we've designed — it does not include a formal ontology layer (entity type constraints, relationship type constraints, cardinality rules). OpenMetadata relies on its entity schema to define what types of entities and relationships are valid, whereas our ontology layer allows runtime definition of new concept types and relationship constraints without schema changes.

**Recommendation:** Use OpenMetadata's glossary model as the starting implementation for our Phase 1 glossary. Defer the full ontology layer to Phase 2 — it adds significant complexity and Phase 1 can operate with a simpler concept hierarchy.

### Ownership and Tags

OpenMetadata treats ownership as a direct attribute on every entity — every asset has an `owner` field that references either a User or a Team entity. This is simple and effective. Tags and classifications are applied via a `tags` array on each entity, where each tag references a Classification (the category) and a Tag (the specific label within that category). PII, Tier (critical/important/etc.), and custom classifications are all modeled this same way.

This maps to our Ownership Registry (3.6) and Classification Engine (3.4). OpenMetadata's approach of embedding ownership and classification directly on the entity record (rather than in separate registries) is architecturally simpler. It trades off independent lifecycle management — in our design, ownership rules and classification rules can evolve independently of the metadata store schema.

---

## 3. Metadata Ingestion

### Connector Architecture

OpenMetadata's ingestion framework is Python-based and follows a consistent pattern across all 130+ connectors:

1. **Source** — connects to the source system API and extracts raw metadata
2. **Processor** — transforms raw metadata into OpenMetadata entity models
3. **Sink** — writes processed entities to the OpenMetadata API
4. **Stage** (optional) — writes intermediate results for debugging

Each connector runs as a workflow, orchestrated by Apache Airflow (default), the OpenMetadata UI, or any cron-compatible scheduler. Workflows are defined in YAML configuration files that specify the source type, connection details (credentials reference a secrets manager), and processing options.

**Mapping to our architecture:**

| OpenMetadata Component | KM Platform Component |
|---|---|
| Source (connector) | Source Connector (Layer 5 adapter) |
| Processor | Metadata Harvester (4.1) normalization logic |
| Sink | Metadata Store (3.1) write path |
| Workflow config | Connector Manager (4.4) registration records |
| Airflow orchestration | Scheduler / Event Trigger |

**Key patterns to adopt:**

**Pull-based ingestion with cursor tracking.** OpenMetadata connectors track their position using cursors (page offsets, timestamps, version numbers) to support incremental extraction. This is exactly our WF1 (Metadata Harvesting) approach — track `last_harvested_at` per source and use source-system pagination cursors for incremental sync.

**Connector configuration as code.** OpenMetadata stores connector configs as YAML — source type, connection URL, credentials reference, ingestion schedule. Our Connector Manager (4.4) should adopt this same pattern. Don't build a complex admin UI for connector management in Phase 1; YAML files in a Git repository give you version control for free.

**Separation of metadata types per workflow.** OpenMetadata runs different workflows for different metadata types: metadata ingestion (schemas, tables, columns), usage ingestion (query logs, access patterns), lineage ingestion (query parsing for upstream/downstream), profiling (data statistics, distributions), and dbt ingestion (models, tests, lineage). Each workflow produces different types of metadata and runs on different schedules. We should adopt this pattern — our WF1 (Harvesting) and WF2 (Change Detection) are already separated, but within WF1, we should consider separating structural metadata extraction from classification, governance evaluation, and graph construction into independently schedulable phases.

### Change Events

OpenMetadata publishes change events for every entity modification. The change event schema captures: entity type, entity FQN, event type (entityCreated, entityUpdated, entityDeleted, entitySoftDeleted), the user/bot that made the change, a timestamp, and a `changeDescription` object that records field-level diffs (fieldsAdded, fieldsUpdated, fieldsDeleted, with previous and new values).

This is precisely our WF2 (Metadata Change Detection) pattern. OpenMetadata's change event schema is well-designed and worth adopting as a reference for our own event schema — particularly the `changeDescription` structure that captures field-level deltas with before/after values.

**Change events can be consumed via:**
- Elasticsearch (for search index updates — our Search Index 3.3)
- Webhooks (for external system notifications)
- Event subscriptions (filtered by entity type, event type, or field changes)

**Recommendation:** Adopt OpenMetadata's change event schema as the basis for our internal event bus messages. It is mature, field-tested, and captures the right granularity for driving downstream updates to the search index, knowledge graph, and lineage engine.

### Incremental Ingestion

OpenMetadata supports incremental ingestion primarily through two mechanisms:

1. **Timestamp-based.** The connector stores the last successful ingestion timestamp and requests only metadata modified after that timestamp on the next run. This depends on the source system exposing a `modified_after` or equivalent filter in its API.

2. **Change-event-driven.** For sources that support webhooks or event streams (Kafka, CDC), OpenMetadata can receive real-time change notifications rather than polling. This is more efficient but requires webhook infrastructure.

Both mechanisms map directly to our architecture: timestamp-based corresponds to our scheduled harvester (WF1), and change-event-driven corresponds to our webhook-triggered change detection (WF2).

**Gap to note:** OpenMetadata does not currently implement a sophisticated reconciliation mechanism for detecting drift between the source system state and the catalog state (e.g., deleted pages that were never reported via webhook). Our Reference Registry (3.2) health-check mechanism — proactively verifying that source URIs are still accessible — fills this gap. This was the architectural finding from our previous OpenMetadata analysis: OpenMetadata only surfaces connector-level service health (is Confluence reachable?), not per-asset URI health (is this specific page still at this URL?).

---

## 4. Lineage and Knowledge Graph

### Data Lineage

OpenMetadata tracks lineage at two levels:

**Table-level lineage.** Shows upstream and downstream dependencies between tables, dashboards, pipelines, and topics. For example: `raw_customers` → (transformed by) `dbt_model_customers` → (visualized in) `customer_dashboard`. Each lineage edge carries a pipeline reference (what process created this dependency) and optionally a SQL query or transformation definition.

**Column-level lineage.** Shows which columns flow into which columns across transformations. For example: `raw_customers.email` → `clean_customers.email_normalized`. This is derived by parsing SQL queries in the lineage ingestion workflow.

**Mapping to our architecture:** OpenMetadata's lineage is narrower than ours. It tracks data flow lineage — how data moves between tables, pipelines, and dashboards. Our Lineage Engine (3.10–3.12) tracks four distinct dimensions: source lineage (where the asset came from), reference lineage (URI mappings), governance lineage (classification and ownership changes), and knowledge lineage (semantic and graph relationships). OpenMetadata's lineage is one dimension within our model.

**Recommendation:** Adopt OpenMetadata's lineage edge model (source entity → edge with pipeline/process reference → target entity) for our source lineage dimension. Do not try to force our governance lineage (classification change trails) or knowledge lineage (concept mappings) into OpenMetadata's lineage model — these are different concerns that need different data structures.

### Entity Graph

OpenMetadata's entity graph is stored in its relational database (MySQL or PostgreSQL) as entity records with JSON fields and relationship records linking them. It is not a native graph database — traversals are implemented as recursive SQL queries or application-level walks. Elasticsearch is used for discovery-time search, not for graph traversal.

This is a significant architectural difference from our design. Our Knowledge Graph Service (3.13–3.15) uses a dedicated graph database (Neo4j or Neptune) for native graph traversal. The trade-off:

| Approach | Advantage | Disadvantage |
|---|---|---|
| OpenMetadata (relational + search) | Simpler operations — one database to manage. Entity consistency is easier with relational constraints. | Multi-hop traversals are slow in SQL. Graph queries require application-level logic. |
| Our design (relational + graph DB) | Native multi-hop traversals in milliseconds. Graph query language (Cypher/Gremlin) is expressive. | Two databases to operate, keep in sync, and version. Eventual consistency between relational and graph stores. |

**Recommendation:** For Phase 1 (Walk milestone), start with a relational-only approach like OpenMetadata — store relationships in PostgreSQL and handle traversals in application code. Introduce Neo4j in Phase 3 (Run milestone) when graph traversal complexity justifies the operational overhead. This reduces Phase 1 infrastructure to a single PostgreSQL database.

### How Agents Can Utilize Lineage and Graph

OpenMetadata's MCP server exposes lineage and graph traversal as tools that AI agents can invoke via natural language. An agent connected to OpenMetadata's MCP server can ask questions like "show me the lineage of this dashboard" or "what tables does this metric depend on" and receive structured responses from the metadata graph.

For our platform, agent utilization of lineage and graph structures should follow the same pattern but through our own API surface:

1. **Discovery queries** — "find all knowledge assets related to customer onboarding" should traverse the knowledge graph from the "Customer" concept through "Onboarding" process nodes to connected knowledge assets (this is WF3 step 7 in our workflow specification).

2. **Provenance queries** — "where did this information come from" should follow the lineage chain from the knowledge asset back through the harvest job to the source system (WF5 provenance chain).

3. **Impact analysis** — "what would be affected if this Confluence space is deprecated" should traverse all downstream dependencies: assets in the space → agents consuming those assets → processes documented by those assets.

4. **Confidence-weighted results** — our graph edges carry confidence scores (0.0–1.0) based on extraction method (source-derived: 0.95–1.0, pattern-derived: 0.7–0.9, ontology-inferred: 0.5–0.8). Agents can use these scores to weight their responses — preferring high-confidence relationships over inferred ones.

---

## 5. Search and Discovery

### Metadata Indexing

OpenMetadata uses Elasticsearch (or OpenSearch) as its search backend. Every entity is indexed with its metadata fields — name, description, display name, FQN, tags, owners, domain, service, and type-specific fields (columns for tables, charts for dashboards, etc.). The index is kept in sync through change events: whenever an entity is created, updated, or deleted, a corresponding update is applied to the search index.

This maps directly to our Search Index (3.3) component. OpenMetadata's approach — event-driven index sync from the entity store — is the right pattern. Our workflow specifications already define this: the Metadata Repository publishes change events, and the Search Index listens and updates.

### Semantic Search

OpenMetadata recently introduced semantic search capabilities that go beyond keyword matching. Instead of requiring exact term matches, semantic search understands conceptual similarity — a search for "customer data" can surface assets tagged with "client information" even if the words don't overlap. This is powered by embedding-based search layered on top of Elasticsearch.

**Important boundary for our platform:** Semantic search in OpenMetadata operates over metadata text fields (names, descriptions, tags). It does NOT embed or search document content. This aligns with our Phase 1 constraint — no embeddings, no vector search, no RAG indexing. However, our Enterprise Knowledge Semantic Layer (3.16) achieves similar semantic expansion through the glossary synonym mechanism rather than embeddings: "customer" maps to a glossary term with synonyms ["client", "account holder"], and discovery queries are expanded with those synonyms before hitting the search index (WF3, step 6).

**Recommendation:** For Phase 1, use glossary-based synonym expansion (our current design) rather than embedding-based semantic search. It is deterministic, explainable, and does not require ML infrastructure. If Phase 2 introduces embeddings, OpenMetadata's approach of layering semantic search on top of Elasticsearch is a reasonable reference architecture.

### Faceted Search

OpenMetadata supports faceted search across standard dimensions: entity type (table, dashboard, pipeline), service (Snowflake, Confluence), domain, owner, tag, tier, and classification. Users can combine facets to narrow results — for example, "all tables in the Finance domain tagged PII owned by the analytics team."

Our Discovery API (2.1) should support the same faceting pattern. The facet dimensions relevant for our broader knowledge scope:

| Facet | Purpose | OpenMetadata Equivalent |
|---|---|---|
| Asset type | Filter by document, API, dashboard, data asset, etc. | Entity type |
| Source system | Filter by Confluence, Git, Snowflake, etc. | Service |
| Domain | Filter by business domain | Domain |
| Owner | Filter by owning team or individual | Owner |
| Classification | Filter by public/internal/confidential/restricted | Tag (PII tier) |
| Quality score | Filter by metadata quality threshold | Not natively supported |
| Status | Filter by active/archived/draft | Lifecycle status |

### What Is Useful for Agents vs Humans

OpenMetadata's search interface is UI-optimized — autocomplete, visual facets, clickable cards. Agents don't need any of this. What agents need from search:

**Structured query parameters.** Not a search box — agents need typed filter parameters they can construct programmatically: `{query: "onboarding", filters: {domain: "retail-banking", classification: ["public", "internal"], status: "active"}, limit: 20}`.

**Ranked, scored results.** Each result should carry a relevance score that the agent can use in its reasoning — "I found 14 assets, the top 3 have relevance scores above 0.85."

**Metadata, not content.** Agents need enough metadata to decide whether to retrieve the full content (via Context API), not the content itself in search results.

**Governance pre-filtering.** Results should already be filtered by the agent's access permissions — the agent should never see assets it cannot access. This is our WF3 governance filtering step (step 10) and is non-negotiable.

OpenMetadata's MCP server handles some of these requirements, but its API is still oriented around the same entity model used for human UI interactions. Our Discovery API (2.1) should be purpose-built for agent consumption with structured query/response contracts.

---

## 6. Governance

### Policies and Access Control

OpenMetadata implements a hybrid RBAC+ABAC model, drawing from NIST specifications for both role-based and attribute-based access control. The model works as follows:

**Roles** are assigned to users and teams. Each role contains one or more policies. Standard roles include Data Steward, Data Consumer, and Admin.

**Policies** contain rules. Each rule specifies: an effect (allow or deny), a list of operations (ViewAll, EditAll, EditDescription, EditTags, etc.), a list of resources (table, dashboard, pipeline, or wildcard), and an optional condition expression. Conditions can reference subject attributes (isOwner(), hasRole(), inTeam()) and object attributes (matchAnyTag(), matchDomain()).

**Rule evaluation** uses a priority system: the most specific rule wins, and deny overrides allow at the same specificity level.

**Mapping to our architecture:**

| OpenMetadata Concept | KM Platform Component |
|---|---|
| Role | Agent trust_level + scopes in Agent Registry (1.6) |
| Policy | Policy rules in Policy Store (3.7) |
| Rule evaluation | ABAC Policy Engine (3.5) |
| Condition expressions | ABAC attribute evaluation |
| Operations (ViewAll, EditAll) | API scope checks in Gateway Authorization (1.2) |

**Key difference:** OpenMetadata's access control governs UI and API operations on metadata entities (who can edit a table's description, who can add tags). Our ABAC model governs agent access to knowledge assets and their content — a broader scope. Our policy engine evaluates: agent trust level, asset classification, domain membership, time-of-day, and request volume — attributes that OpenMetadata's condition expressions partially support but don't fully cover.

**Recommendation:** Adopt OpenMetadata's policy/rule/condition structure as the schema for our Policy Store (3.7). Extend the condition expression language to support our agent-specific attributes (trust_level, max_classification, knowledge_domains). Do not adopt OpenMetadata's role model — our Agent Registry already captures agent attributes more precisely than roles can express.

### Data Quality Metadata

OpenMetadata supports data quality through its Test Framework: Test Definitions (what to check), Test Cases (check applied to a specific entity), and Test Suites (groups of test cases). Results include status (pass/fail/aborted), timestamps, and sample data. Quality signals are displayed alongside entity metadata, giving consumers immediate visibility into data trustworthiness.

For our KM platform, data quality metadata translates to **knowledge quality signals** — but the quality dimensions are different:

| Data Quality (OpenMetadata) | Knowledge Quality (KM Platform) |
|---|---|
| Null percentage | Metadata completeness (does the asset have a description, owner, classification?) |
| Freshness (time since last update) | Content staleness (how old is the source content?) |
| Schema drift | Reference health (is the source URI still valid?) |
| Row count anomaly | Not applicable |
| Custom SQL test | Not applicable |
| Column value range | Not applicable |

**Recommendation:** Adopt the concept of quality test definitions and test cases, but define our own quality dimensions appropriate for knowledge assets. The quality_score field on our metadata records should be computed from: metadata completeness, reference health, content freshness, and classification confidence.

### Stewardship

OpenMetadata supports stewardship through glossary term review workflows (terms go through Draft → Approved → Deprecated), ownership assignment (any entity can have an owner), and collaborative features (conversations, tasks, announcements tied to entities). Stewards are assigned via roles and team membership.

This maps to our Stewardship Workflow (3.9). OpenMetadata's approach is UI-centric — stewards review and approve through the web interface. Our design should support notification-based stewardship (Slack/email notifications with approve/reject actions) since our Phase 1 does not include a steward UI.

### Auditability

OpenMetadata tracks entity version history through its change event system. Every field-level change to an entity is recorded with: who made the change, when, what the previous value was, and what the new value is. This provides a complete audit trail for any entity.

Our Audit Log (3.8) and Metadata Lineage (3.12) serve the same purpose. OpenMetadata's version history is entity-centric (all changes to an entity are grouped under that entity's version history). Our audit log is event-centric (each change is an independent immutable event). Both approaches provide auditability — the event-centric approach is better for compliance reporting ("show me all classification changes in the last quarter across all assets") while the entity-centric approach is better for entity investigation ("show me the full history of this specific asset").

**Recommendation:** Implement both: entity version history on metadata records (increment version, track field changes) AND an independent append-only audit event log. The version history supports entity-level investigation; the audit log supports cross-entity compliance queries.

---

## 7. Open Standards

### OpenMetadata Standard

OpenMetadata has published its metadata specifications as a separate open-source project (OpenMetadata Standards, v1.13.0 as of April 2026). The standard includes 700+ JSON Schemas covering entities, types, API contracts, events, and configurations. It also includes RDF ontologies (RDF/OWL), JSON-LD contexts, and SHACL shapes for validation.

The standard is aligned with several W3C and industry specifications: DCAT (Data Catalog Vocabulary), DPROD (Data Product), PROV-O (Provenance Ontology), OpenLineage, ODCS (Open Data Contract Standard), RDF/OWL, JSON-LD, and SHACL.

### JSON Schema

OpenMetadata's schema-first approach means that the JSON Schema definitions are the authoritative specification — code is generated from schemas, not the other way around. This has two practical benefits for us:

1. **Reusable entity schemas.** We can adopt OpenMetadata's entity schemas (Table, Dashboard, Pipeline, etc.) as a starting point for our Metadata Store (3.1) type_metadata JSONB extensions. Rather than designing our own metadata model for data assets from scratch, we can reference the OpenMetadata schema for those entity types and extend it for our additional knowledge asset types (documents, wikis, APIs, processes).

2. **API contract generation.** If we adopt OpenMetadata's schemas, we can generate API clients, validators, and documentation from the same schema definitions — reducing drift between our API contracts and our storage model.

### Event Schema

OpenMetadata's change event schema is well-structured:

```
{
  "id": "uuid",
  "eventType": "entityCreated | entityUpdated | entityDeleted | entitySoftDeleted",
  "entityType": "table | dashboard | ...",
  "entityId": "uuid",
  "entityFullyQualifiedName": "service.database.schema.table",
  "userName": "who made the change",
  "timestamp": 1718700000,
  "changeDescription": {
    "fieldsAdded": [{"name": "fieldName", "newValue": "..."}],
    "fieldsUpdated": [{"name": "fieldName", "oldValue": "...", "newValue": "..."}],
    "fieldsDeleted": [{"name": "fieldName", "oldValue": "..."}],
    "previousVersion": 1.2
  }
}
```

**Recommendation:** Adopt this schema (or a simplified version of it) as the internal event format on our event bus. It captures exactly the information our downstream consumers need: what entity changed, how it changed (field-level delta), who changed it, and when.

### API-First Architecture

OpenMetadata exposes every operation through a REST API with consistent endpoint patterns: `GET /api/v1/{entityType}`, `GET /api/v1/{entityType}/{id}`, `GET /api/v1/{entityType}/name/{fqn}`, `PUT /api/v1/{entityType}`, `PATCH /api/v1/{entityType}/{id}`, `DELETE /api/v1/{entityType}/{id}`. Every endpoint returns the entity's JSON Schema-defined structure.

Our platform should adopt this consistency. Our API surface (Layer 2) currently defines four specialized APIs (Discovery, Semantic, Context, Governance). These are the right abstractions for agent-facing interactions, but the internal APIs between Layer 3 components should follow the same RESTful consistency that OpenMetadata uses — predictable URL patterns, schema-validated request/response bodies, and standard HTTP semantics.

### Interoperability Considerations

OpenMetadata's adoption of W3C standards (RDF, OWL, JSON-LD, SHACL) creates theoretical interoperability with other metadata systems that speak the same standards. In practice, interoperability between metadata platforms is still nascent — most organizations run a single metadata catalog, not a federated mesh of catalogs.

For our platform, the most practical interoperability concern is with OpenMetadata itself. If the organization already runs OpenMetadata for data catalog purposes, our KM platform should be able to consume metadata from OpenMetadata as a source system — treating OpenMetadata the same way we treat Confluence or Snowflake. This means building a Source Connector for the OpenMetadata API, which is straightforward given OpenMetadata's well-documented REST API.

**Recommendation:** Do not invest in W3C RDF/OWL interoperability for Phase 1. It adds complexity without a clear consumer. Instead, build an OpenMetadata source connector as one of the early connector implementations — this lets organizations that use OpenMetadata for data assets federate their data catalog metadata into our broader knowledge platform.

---

## 8. Architecture Patterns We Should Adopt

### Must-Have: Adopt Directly

**Schema-first entity model.** Define our knowledge asset types as JSON Schemas before writing code. Generate validators, API contracts, and documentation from those schemas. This eliminates the class of bugs where the storage model, API model, and documentation diverge. OpenMetadata has proven this pattern at scale with 700+ schemas.

**Change event pipeline.** Publish a structured change event for every metadata mutation. Use these events to drive the search index, knowledge graph, lineage engine, and any future consumers. This is the architectural backbone that keeps all downstream projections (search, graph, lineage) eventually consistent with the metadata store. Our WF4 (Graph Construction) already defines this as an event-driven pipeline — make it the universal pattern for all downstream updates.

**Connector abstraction with consistent interface.** Every source connector should implement the same interface: connect, extract_metadata, normalize, and track_cursor. OpenMetadata's connector framework proves that this consistency enables rapid development of new connectors — they have 130+ connectors because each one follows the same pattern.

**Glossary with approval workflow.** OpenMetadata's glossary model (term → definition → synonyms → related terms → approval status) is exactly what our Enterprise Knowledge Semantic Layer (3.16) needs for Phase 1. Don't over-engineer the ontology layer — start with glossary terms and add formal ontology constraints when real use cases demand them.

**RBAC+ABAC hybrid access control.** OpenMetadata's approach of combining role-based grouping with attribute-based condition evaluation is the right model for our Governance Engine. Adopt their policy/rule/condition schema structure and extend it with our agent-specific attributes.

**Entity versioning with field-level change tracking.** Every entity mutation should increment a version number and record which fields changed, what the old values were, and why. OpenMetadata stores this as part of the entity record. We should do the same in our Metadata Store and additionally persist it in the Audit Log for cross-entity compliance queries.

### Nice-to-Have: Adopt When Needed

**Data contracts.** OpenMetadata's Data Contract model defines schema contracts (expected columns and types), quality contracts (expected test results), freshness contracts (maximum staleness), and availability contracts (uptime expectations). This concept is valuable for our platform but not critical for Phase 1. When we add data asset connectors (Snowflake, BigQuery), data contracts become relevant for governing the quality expectations of those assets. For Phase 1 knowledge assets (Confluence pages, wikis), the equivalent is our metadata quality score — a simpler mechanism that achieves the same goal of signaling trustworthiness.

**MCP server integration.** OpenMetadata's native MCP server is a compelling reference for how to expose metadata context to AI agents. Our platform should eventually expose an MCP server as well — but only after the core metadata services are stable. MCP is an activation channel, not a core architectural component. Build it as a facade over our existing APIs (Discovery, Semantic, Context, Governance) rather than a parallel access path.

**Memory nuggets.** OpenMetadata's concept of governed "memory nuggets" — structured knowledge fragments derived from conversations, decisions, and agent interactions — is relevant to our platform's future evolution. For Phase 1, we do not need this. When multi-agent collaboration becomes a requirement, memory nuggets provide a model for how agents can persist and share learned knowledge within the governance framework.

**Data products.** OpenMetadata's Data Product entity models curated, governed data offerings that package multiple assets with a defined schema, SLA, and access policy. This concept maps to a future "Knowledge Product" abstraction in our platform — curated collections of knowledge assets packaged for specific agent use cases. Defer to Phase 2 or later.

### Components to Simplify

**Graph storage.** OpenMetadata does not use a dedicated graph database — it stores entity relationships in its relational database and handles traversals in application code. For our Phase 1, this is sufficient. We should start with PostgreSQL-stored relationships and only introduce Neo4j when graph query complexity justifies the operational overhead (likely in our Run milestone when we implement multi-hop semantic traversals).

**Search infrastructure.** OpenMetadata uses Elasticsearch for both search and some internal operations. For Phase 1, we can start with PostgreSQL full-text search (using `tsvector` and `GIN` indexes) for a small catalog. Introduce OpenSearch/Elasticsearch when the asset count exceeds a few thousand or when faceted search performance becomes an issue.

**Ingestion orchestration.** OpenMetadata defaults to Apache Airflow for workflow orchestration. This is heavyweight for Phase 1. Start with a simple cron-based scheduler for harvest jobs. Introduce Airflow (or a lighter alternative like Dagster) when you need dependency management between ingestion pipelines.

**UI.** OpenMetadata includes a full web UI for data discovery, governance, and collaboration. Our platform is agent-first — we do not need a human-facing UI in Phase 1. Build APIs only. If a steward UI is needed for governance review workflows, use existing tools (Slack notifications, a simple admin dashboard) rather than building a custom catalog UI.

### Suggested Architecture Alignment

Mapping our build milestones to OpenMetadata adoption:

**Walk (Milestone 1):** Use OpenMetadata's JSON Schema patterns for our entity model. Implement a single Confluence connector following their connector interface pattern. Store metadata in PostgreSQL with the same FQN-based identification they use. Publish change events on entity mutations. Use PostgreSQL full-text search instead of Elasticsearch.

**Jog (Milestone 2):** Adopt OpenMetadata's policy/rule schema for our Policy Store. Implement their glossary model for our semantic layer. Add their change event schema for our event bus. Introduce Elasticsearch if search volume demands it.

**Run (Milestone 3):** Introduce graph database for knowledge graph traversals. Build an MCP server facade over our APIs. Consider adopting OpenMetadata's data contract model for data asset governance. Evaluate memory nuggets for multi-agent knowledge sharing.

---

## 9. Features OpenMetadata Doesn't Support Well That We Need

### Agent-Native APIs

OpenMetadata's API is entity-CRUD-oriented — `GET /api/v1/tables/{id}`, `GET /api/v1/glossaryTerms/name/{fqn}`. Agents don't think in entity CRUD operations. They think in intents: "find me knowledge about X," "explain what Y means in our organization," "give me everything I need to answer a question about Z." Our API surface (Discovery, Semantic, Context, Governance) is purpose-built for these agent intents. OpenMetadata's MCP server bridges this gap partially by letting agents express natural language queries, but the underlying API is still entity-centric.

**What we need to build:** Intent-based APIs that combine search, semantic expansion, graph traversal, governance filtering, and context assembly into single request/response cycles — exactly what our WF3 (Knowledge Discovery) workflow describes. OpenMetadata requires multiple API calls to achieve the same outcome.

### Agent Registry and Trust Model

OpenMetadata authenticates bots (automated processes) using JWT tokens and assigns them the same role-based permissions as human users. There is no concept of agent trust levels, agent capability declarations, knowledge domain scoping, or risk classification. A bot in OpenMetadata is a user that doesn't use a browser — it has a role and policies, nothing more.

**What we need to build:** Our Agent Registry (1.6) with: agent_type, trust_level, allowed scopes, knowledge_domains, max_classification, rate_limit, and owner_team. This is agent-native metadata that has no equivalent in OpenMetadata.

### Multi-Agent Collaboration Metadata

OpenMetadata does not model interactions between multiple AI agents. It has no concept of agent sessions, agent-to-agent handoffs, shared context between agents, or collaborative knowledge construction by multiple agents working on the same task.

**What we should consider for future phases:**
- Agent session metadata: which agent accessed which assets in which order within a session
- Shared context manifests: pre-assembled context packages that one agent creates for another
- Agent provenance chains: when Agent A's output becomes Agent B's input, track that chain
- Collaborative knowledge assertions: when multiple agents agree on a knowledge claim, track the consensus

### Reasoning Traces and Provenance

When an agent uses knowledge from our platform to generate a response, OpenMetadata has no mechanism to capture what knowledge was used, how the agent reasoned about it, or what confidence the agent had in its answer. OpenMetadata's memory nuggets capture static knowledge fragments, not dynamic reasoning traces.

**What we need to build:** Our Agent Access Lineage (3.11) already addresses the "what knowledge was used" part. Beyond that, future phases should consider:
- Reasoning trace storage: a structured record of the agent's decision process (which assets it considered, which it filtered, which it used, and why)
- Answer provenance: linking agent responses back to specific metadata records and source URIs
- Confidence propagation: if a knowledge asset has a quality_score of 0.7 and a graph edge has a confidence of 0.8, the agent's answer confidence should reflect these upstream scores

### Prompt and Tool Catalog

OpenMetadata catalogs data assets, not AI operational assets. It does not model prompts (system prompts, prompt templates, prompt chains), tools (MCP tool definitions, function calling schemas), or agent configurations (model selection, temperature, context window budgets) as first-class entities.

**What we should consider:** Treating prompts, tools, and agent configurations as knowledge assets in our catalog — with the same metadata, classification, ownership, lineage, and governance as any other asset. An agent should be able to discover "what prompts are approved for customer-facing interactions in the retail-banking domain" through the same Discovery API it uses for documents.

### Semantic Contracts

OpenMetadata's data contracts define expectations about schema, quality, freshness, and availability. Our platform needs a broader concept: **semantic contracts** — agreements about the meaning, accuracy, and validity of knowledge. For example: "the definition of 'customer churn' in the retail-banking domain means involuntary account closure within 90 days of last activity, as approved by the analytics team, validated quarterly."

Semantic contracts would govern: term definitions (who can change them, what approval is needed), classification assertions (under what conditions can a classification be overridden), and knowledge validity windows (this policy document is valid until December 2026).

### Knowledge Confidence Scores

OpenMetadata does not assign confidence scores to metadata — metadata is either present or absent. Our platform needs graduated confidence: auto-classified assets have lower confidence than human-reviewed ones; pattern-derived graph relationships have lower confidence than source-derived ones; stale assets have lower confidence than freshly verified ones.

Our architecture already defines this through: classification confidence scores in the Classification Engine (3.4), graph edge confidence scores in the Knowledge Graph (3.14), and quality scores on asset metadata. OpenMetadata has no equivalent mechanism.

### Context Assembly for Agents

The most significant gap. OpenMetadata returns entity records — flat JSON objects with metadata fields. Our Context API (2.3) assembles a complete context package: metadata + semantic context (glossary terms, related concepts) + graph context (related entities, processes, policies) + lineage (provenance chain) + governance summary (what the agent can/cannot access, and why). This assembly is a multi-step operation involving calls to the Metadata Store, Knowledge Graph, Semantic Layer, Governance Engine, and Lineage Engine.

OpenMetadata's MCP server can answer individual questions ("what is this table?", "who owns it?", "show me its lineage"), but it cannot assemble a holistic context package optimized for agent consumption in a single operation.

---

## 10. Final Recommendation

### Adopt Directly

| Concept | From OpenMetadata | Apply To |
|---|---|---|
| Schema-first entity definitions | JSON Schema for every entity type | Metadata Store (3.1) — define knowledge_assets schema and type-specific extensions as JSON Schemas before writing code |
| Connector interface pattern | Source → Processor → Sink → Stage | Source Connectors (Layer 5) and Metadata Harvester (4.1) — each connector implements the same extract/normalize/track interface |
| Change event schema | entityCreated/Updated/Deleted with changeDescription | Internal event bus between Metadata Store and downstream consumers (Search Index, Graph, Lineage) |
| Glossary model | Glossary → Term → synonyms/related terms → approval status | Enterprise Knowledge Semantic Layer (3.16) Phase 1 implementation |
| Policy/rule/condition schema | Policy contains rules; rules have effect, operations, resources, conditions | Policy Store (3.7) — adopt the schema, extend conditions for agent attributes |
| Entity versioning | Version increment + field-level change tracking per entity | Metadata Store (3.1) — metadata_version field with change history |
| FQN-based identification | Hierarchical fully qualified names for source-system references | Reference Registry (3.2) — use FQNs for source-side identification, platform URIs for stable cross-system identity |
| Connector configuration as code | YAML configs for source type, connection, credentials, schedule | Connector Manager (4.4) — YAML in Git for Phase 1 |

### Simplify

| What to Simplify | OpenMetadata Approach | Our Simpler Approach (Phase 1) |
|---|---|---|
| Graph storage | Relational DB for entity relationships | Start with PostgreSQL relationships; defer Neo4j to Run milestone |
| Search infrastructure | Elasticsearch required from deployment | Start with PostgreSQL full-text search; add OpenSearch when catalog exceeds thousands of assets |
| Ingestion orchestration | Apache Airflow as default | Cron-based scheduler; introduce Airflow only when pipeline dependencies demand it |
| Steward UI | Full web application | Slack notifications with approve/reject actions |
| Data products | Full data product entity model | Defer entirely — revisit when "knowledge products" become a real requirement |
| W3C semantic standards | RDF/OWL/SHACL/JSON-LD support | Defer — no practical consumer for Phase 1; build pragmatic REST APIs instead |

### Redesign for Agent-Native

| What to Redesign | Why OpenMetadata's Approach Doesn't Fit | Our Approach |
|---|---|---|
| Discovery interface | Entity-CRUD API requires multiple calls to discover knowledge | Single intent-based Discovery API (2.1) that combines search + semantic expansion + graph traversal + governance filtering |
| Agent identity model | Bots are simplified users with roles | Agent Registry (1.6) with trust levels, capability declarations, domain scoping, and risk classification |
| Context assembly | No assembled context — individual entity lookups only | Context API (2.3) that assembles metadata + semantic + graph + lineage + governance into a single response package |
| Lineage model | Data flow lineage only (table → pipeline → dashboard) | Four-dimension lineage: source, reference, governance, knowledge — each tracked independently |
| Confidence scoring | Binary metadata presence (exists or doesn't) | Graduated confidence on classifications (0.0–1.0), graph edges (source/pattern/inferred), and composite quality scores |
| Per-asset health checking | Connector-level service health only | Reference Registry (3.2) with proactive per-asset URI validation on independent schedules |
| Knowledge scope | Data assets only (tables, dashboards, pipelines, models) | Full knowledge spectrum: documents, wikis, APIs, processes, business concepts, policies, dashboards, data assets |

### Implementation Priority

For the Walk milestone (our immediate build target), the following OpenMetadata patterns should be adopted in order:

1. Define the knowledge_assets JSON Schema (adopting OpenMetadata's entity schema pattern)
2. Build the Confluence connector following OpenMetadata's Source → Processor → Sink pattern
3. Implement the Metadata Store with entity versioning and change event publishing
4. Set up the change event pipeline to drive the Search Index
5. Build the Discovery API with structured query/response contracts (agent-native, not entity-CRUD)

For the Jog milestone, add:
1. Glossary model from OpenMetadata for the Enterprise Knowledge Semantic Layer
2. Policy/rule/condition schema for the Governance Engine
3. Agent Registry (no OpenMetadata equivalent — custom build)
4. Audit Log with OpenMetadata's change event schema as the event format

For the Run milestone, add:
1. Graph database for knowledge graph traversals (beyond OpenMetadata's relational approach)
2. Context API for assembled knowledge packages (beyond OpenMetadata's entity-CRUD model)
3. MCP server facade over our APIs (referencing OpenMetadata's MCP implementation as a guide)
4. Evaluate data contracts for data asset governance

---

*End of cookbook.*
