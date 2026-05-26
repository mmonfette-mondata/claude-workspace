# Understanding Techspecs in SDLC and md.eco.sg Project

**Date:** 2026-05-26

## Question

I have difficulty understanding what a 'techspec' is in the context of the SDLC and the md.eco.sg project.

## Answer

### What is a Techspec?

A **techspec** (technical specification) is a **shared, implementation-level document** that defines the technical contracts and structures used across multiple features within a domain. Think of it as the "blueprint" layer between high-level feature requirements and actual code.

### Why Techspecs Exist: The Separation of Concerns

Your SDLC follows a clear philosophy: **"Humans own requirements and guidelines. Claude owns implementation."** This creates a natural hierarchy:

```
1. Domain Definition (docs/domain.md)
   ↓
2. Feature Specifications (docs/features/F001-*.md)
   ↓
3. Technical Specifications (docs/techspecs/*.md) ← TECHSPECS
   ↓
4. Implementation (apps/, migrations/, proto/)
```

**Techspecs solve a specific problem:** When multiple features need to reference the same technical details (database schema, API contracts, proto definitions), you don't want to duplicate that information in every feature file. Instead, **feature files link to techspecs**—they don't embed them.

### What Goes in a Techspec vs. a Feature File?

| **Feature File** (`docs/features/F001-*.md`) | **Techspec** (`docs/techspecs/*.md`) |
|----------------------------------------------|--------------------------------------|
| **What** the feature does (requirements) | **How** it's implemented technically |
| Functional and non-functional requirements | Database tables, columns, indexes |
| Test cases (Given/When/Then) | Proto message definitions |
| Which components are affected | API routes and contracts |
| **References** techspecs | Shared across multiple features |

### Real Examples from md.eco.sg

Looking at `domains/ms-entities/docs/techspecs/`, here are concrete examples:

#### 1. database-schema.md

Defines tables like `tenants` and `outbox` with complete structure:

```sql
-- tenants table
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | UUID | PK DEFAULT gen_random_uuid() | Internal surrogate key |
| ms_tenant_id | TEXT | UNIQUE NOT NULL | Microsoft 365 Tenant ID |
| display_name | TEXT | NOT NULL | Human-readable tenant name |
| state | tenant_state | NOT NULL DEFAULT 'pending' | PostgreSQL enum type |
| encryption_key_version | INT | NOT NULL DEFAULT 0 | Active AES key version |
| created_at | TIMESTAMPTZ | NOT NULL DEFAULT now() | System-set at creation |
| updated_at | TIMESTAMPTZ | NOT NULL DEFAULT now() | Refreshed on every mutation |
```

**Why this is a techspec:** Multiple features need this information (F002 creates tenants, F026 adds encryption key rotation, F027 adds security groups). Rather than duplicating table definitions in each feature file, they all reference `database-schema.md`.

#### 2. proto-public.md

Defines inter-domain gRPC services and lifecycle event contracts:

```protobuf
message Tenant {
  string ms_tenant_id = 1;                    // Microsoft 365 Tenant ID
  string display_name = 2;                    // Human-readable name
  TenantState state = 3;                      // Current state
  google.protobuf.Timestamp created_at = 4;
  google.protobuf.Timestamp updated_at = 5;
}

service PublicTenantService {
  // Returns active tenants assigned to a worker shard via MD5 hash modulo.
  rpc ListActiveTenants(ListActiveTenantsRequest) returns (ListActiveTenantsResponse);
}
```

**Why this is a techspec:** When Feature F003 talks about "public protos," it references this techspec rather than duplicating all the proto definitions. Any downstream domain consuming these protos needs this contract.

#### 3. bff-routes.md

Defines HTTP API contracts with complete specifications:

```
## GET /api/v1/tenants

List tenants with optional filtering, search, sorting, and cursor pagination.

### Request Parameters
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| page_size | integer | no | 20 | Items per page (max 100) |
| page_token | string | no | | Opaque cursor from previous response |
| state | string | no | | Filter by state: pending, active, suspended, deleted |
| search | string | no | | Free-text search on Display Name and MS Tenant ID |

### Response — 200 OK
{
  "items": [...],
  "next_page_token": "opaque-cursor-string",
  "total_count": 42
}

### gRPC Mapping
| HTTP | gRPC |
|------|------|
| page_size | ListTenantsRequest.page_size |
| state | ListTenantsRequest.state — map string to TenantState enum |
```

**Why this is a techspec:** Any feature that adds or modifies API endpoints updates this single document. The BFF implementation reads this techspec to know exactly what HTTP contracts to implement.

### The Deeper Principle: Single Source of Truth

Techspecs embody a core software engineering principle: **avoid duplicating authoritative information**.

- **Bad approach:** Every feature file describes database columns → When a column changes, you update 5 feature files.
- **Good approach (techspec):** Feature files say "see database-schema.md for table structure" → When a column changes, you update one place.

This is similar to why we use:
- **Proto files** instead of hand-writing serialization code
- **Database migrations** instead of manually documenting schema changes
- **OpenAPI specs** instead of documenting API contracts in comments

### Example: How Feature Files Reference Techspecs

From `F003-proto-split.md`:

```markdown
## Techspec References

- Existing proto techspec from the PoC repo (migrated in F002) will be split into 
  `domains/ms-entities/docs/techspecs/proto-public.md` and 
  `domains/ms-entities/docs/techspecs/proto-private.md` — no content changes, 
  structural split only
```

The feature file doesn't include the proto definitions—it just references where to find them.

### Common Techspecs in md.eco.sg

Based on the actual project structure:

| Techspec File | Purpose | Shared Across |
|---------------|---------|---------------|
| `database-schema.md` | Table definitions, columns, indexes, constraints, migration order | All features touching the database |
| `proto-public.md` | Inter-domain protos (LCEs, cross-domain gRPC) | Features that expose domain boundaries |
| `proto-private.md` | Intra-domain protos (BFF-consumed services, internal RPCs) | Features within the domain |
| `bff-routes.md` | HTTP API contracts, request/response formats, validation | All features with UI/API components |
| `lce-contracts.md` | Lifecycle event definitions and contracts | Features producing/consuming domain events |
| `architecture.md` | Domain-level architectural decisions | All features needing context |
| `tech-stack.md` | Technologies, libraries, versions used | All implementations |
| `deployment.md` | Deployment configuration and procedures | DevOps and deployment features |

### When to Create or Update a Techspec

From the domain-development guideline:

> "Once a feature is fully specified (description, techspecs, test cases all reviewed): Commit the feature file and any new/updated techspecs"

**You create/update techspecs during Phase 2 (Feature Specification)** when you realize:

- A new database table is needed → Update `techspecs/database-schema.md`
- A new proto message is needed → Update `techspecs/proto-public.md` or `proto-private.md`
- A new API endpoint is needed → Update `techspecs/bff-routes.md`
- A new lifecycle event is published → Update `techspecs/lce-contracts.md`

Then your feature file simply references: 

```markdown
## Techspec References

- `docs/techspecs/database-schema.md` — tenants table structure
- `docs/techspecs/proto-public.md` — TenantSnapshot message definition
```

### The 4-Phase Development Flow

```
Phase 1: Domain Definition
  └─ Define entities, state machines, inputs/outputs
     Output: docs/domain.md

Phase 2: Feature Specification
  ├─ Write feature requirements (F001-*.md)
  │  - Functional requirements (FR1, FR2, ...)
  │  - Non-functional requirements (NFR1, NFR2, ...)
  │  - Test cases (F001-TC001, F001-TC002, ...)
  │  - Components affected
  │  - Dependencies
  │
  └─ Update/create techspecs as needed
     Output: docs/features/F001-*.md + docs/techspecs/*.md

Phase 3: Checkpoint
  └─ Commit feature + techspecs, clear context

Phase 4: Implementation
  └─ Read feature file → Read techspecs → Write code
     - domain-service (gRPC handlers, database)
     - bff (HTTP/JSON gateway)
     - mfe-* (UI components)
     Output: apps/, migrations/, proto/, test/
```

### Why Techspecs are Separate from Code

**Techspecs are documentation, not implementation.** They exist to:

1. **Communicate intent** before writing code
2. **Enable review** of technical contracts without reading code
3. **Provide a stable reference** that multiple features can link to
4. **Separate "what to build" from "how it's built"**

During implementation (Phase 4), Claude reads:
- The feature file (requirements and test cases)
- The referenced techspecs (technical contracts)
- The pattern guidelines (implementation structure)

Then writes:
- Database migrations
- Proto definitions
- Application code
- Tests

The techspec bridges the gap between "we need to store tenants" (feature requirement) and the actual SQL migration file.

### Key Insight: Techspecs vs. Generated Artifacts

| Techspec | Generated Artifact |
|----------|-------------------|
| `techspecs/database-schema.md` | `migrations/003_create_tenants.sql` |
| `techspecs/proto-public.md` | `proto/public/v1/tenant.proto` |
| `techspecs/bff-routes.md` | `apps/bff/internal/handler/tenant.go` |

The techspec describes **what** needs to exist. The artifact is the **actual implementation**. 

- Techspecs can include rationale, constraints, and cross-references
- Artifacts are executable/compilable code
- Techspecs are reviewed by humans; artifacts are validated by compilers/tests

### Summary

A **techspec** is:
- A **shared technical contract document** (database, API, proto, events)
- Lives in `docs/techspecs/*.md`
- Referenced by multiple feature files
- Created/updated during Phase 2 (Feature Specification)
- Read by Claude during Phase 4 (Implementation)
- The single source of truth for technical structures

It solves the problem of **not duplicating technical details across feature files** while maintaining a clear separation between requirements (what) and implementation (how).
