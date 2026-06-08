# Graph Report - /workspaces/opencode-edumanage-nemotron  (2026-06-08)

## Corpus Check
- cluster-only mode — file stats not available

## Summary
- 51 nodes · 44 edges · 16 communities (9 shown, 7 thin omitted)
- Extraction: 98% EXTRACTED · 2% INFERRED · 0% AMBIGUOUS · INFERRED: 1 edges (avg confidence: 0.8)
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `a281f6ea`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- [[_COMMUNITY_Community 0|Community 0]]
- [[_COMMUNITY_Community 1|Community 1]]
- [[_COMMUNITY_Community 2|Community 2]]
- [[_COMMUNITY_Community 3|Community 3]]
- [[_COMMUNITY_Community 4|Community 4]]
- [[_COMMUNITY_Community 5|Community 5]]
- [[_COMMUNITY_Community 6|Community 6]]
- [[_COMMUNITY_Community 7|Community 7]]
- [[_COMMUNITY_Community 8|Community 8]]
- [[_COMMUNITY_Community 9|Community 9]]
- [[_COMMUNITY_Community 10|Community 10]]
- [[_COMMUNITY_Community 11|Community 11]]
- [[_COMMUNITY_Community 13|Community 13]]
- [[_COMMUNITY_Community 14|Community 14]]
- [[_COMMUNITY_Community 15|Community 15]]

## God Nodes (most connected - your core abstractions)
1. `Prisma Schema` - 5 edges
2. `Blueprint — Sprint 004: Financial Management & Mpesa` - 4 edges
3. `Blueprint — Sprint 005: Communications, Calendar & Procurement` - 4 edges
4. `Blueprint — Sprint 007: AI Automation Layer` - 4 edges
5. `Builder Handoff — Sprint 004: Financial Management & Mpesa` - 3 edges
6. `Builder Handoff — Sprint 005: Communications, Calendar & Procurement` - 3 edges
7. `Builder Handoff — Sprint 006: Library & Resources` - 3 edges
8. `Builder Handoff — Sprint 007: AI Automation Layer` - 3 edges
9. `Blueprint — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR` - 3 edges
10. `Builder Handoff — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR` - 3 edges

## Surprising Connections (you probably didn't know these)
- `Blueprint — Sprint 006: Library & Resources` --implements--> `Prisma Schema`  [EXTRACTED]
  planning/sprints/006-library-resources/blueprint.md → prisma/schema.prisma
- `Blueprint — Sprint 007: AI Automation Layer` --implements--> `Prisma Schema`  [EXTRACTED]
  planning/sprints/007-ai-automation/blueprint.md → prisma/schema.prisma
- `Software Architect Agent` --references--> `Sprint 002: Auth & Multi-Tenancy`  [EXTRACTED]
  AGENTS.md → planning/sprints/002-auth-multitenancy/requirements.md
- `Software Architect Agent` --references--> `Sprint 003: Academic Core`  [EXTRACTED]
  AGENTS.md → planning/sprints/003-academic-core/requirements.md
- `Next.js Expert Agent` --references--> `tRPC API Reference`  [EXTRACTED]
  AGENTS.md → docs/API_REFERENCE.md

## Import Cycles
- None detected.

## Communities (16 total, 7 thin omitted)

### Community 0 - "Community 0"
Cohesion: 0.27
Nodes (10): Acceptance Criteria — Sprint 006: Library & Resources, Blueprint — Sprint 004: Financial Management & Mpesa, Blueprint — Sprint 005: Communications, Calendar & Procurement, Blueprint — Sprint 006: Library & Resources, Blueprint — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR, Builder Handoff — Sprint 006: Library & Resources, Inngest Functions, Prisma Schema (+2 more)

### Community 1 - "Community 1"
Cohesion: 0.29
Nodes (7): Clerk Auth Agent, DevOps Agent, Software Architect Agent, Clerk Organization as Tenant, Inngest Jobs Master List, Sprint 002: Auth & Multi-Tenancy, Sprint 003: Academic Core

### Community 2 - "Community 2"
Cohesion: 0.50
Nodes (4): Acceptance Criteria — Sprint 007: AI Automation Layer, Blueprint — Sprint 007: AI Automation Layer, Builder Handoff — Sprint 007: AI Automation Layer, Requirements — Sprint 007: AI Automation Layer

### Community 3 - "Community 3"
Cohesion: 0.67
Nodes (3): Acceptance Criteria — Sprint 004: Financial Management & Mpesa, Builder Handoff — Sprint 004: Financial Management & Mpesa, Requirements — Sprint 004: Financial Management & Mpesa

### Community 4 - "Community 4"
Cohesion: 0.67
Nodes (3): Acceptance Criteria — Sprint 005: Communications, Calendar & Procurement, Builder Handoff — Sprint 005: Communications, Calendar & Procurement, Requirements — Sprint 005: Communications, Calendar & Procurement

### Community 5 - "Community 5"
Cohesion: 0.67
Nodes (3): Acceptance Criteria — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR, Builder Handoff — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR, Requirements — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR

### Community 6 - "Community 6"
Cohesion: 0.67
Nodes (3): Next.js Expert Agent, tRPC API Reference, Role Permissions Matrix

### Community 7 - "Community 7"
Cohesion: 0.67
Nodes (3): System Architecture, Prisma Database Schema, Postgres Row-Level Security

## Knowledge Gaps
- **26 isolated node(s):** `$schema`, `plugin`, `@opencode-ai/plugin`, `Next.js Expert Agent`, `React Frontend Agent` (+21 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **7 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Prisma Schema` connect `Community 0` to `Community 2`?**
  _High betweenness centrality (0.093) - this node is a cross-community bridge._
- **Why does `Blueprint — Sprint 007: AI Automation Layer` connect `Community 2` to `Community 0`?**
  _High betweenness centrality (0.050) - this node is a cross-community bridge._
- **Why does `Blueprint — Sprint 004: Financial Management & Mpesa` connect `Community 0` to `Community 3`?**
  _High betweenness centrality (0.050) - this node is a cross-community bridge._
- **What connects `$schema`, `plugin`, `@opencode-ai/plugin` to the rest of the system?**
  _26 weakly-connected nodes found - possible documentation gaps or missing edges._