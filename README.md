# EduManage

EduManage is a professional-grade, multi-tenant SaaS school management platform built for Kenyan private schools operating on CBC and IGCSE curricula.

## Why EduManage

Kenyan private schools rely on fragmented workflows: WhatsApp groups for communication, Excel spreadsheets for fees and attendance, paper registers for records, and manual approvals for procurement and payroll. EduManage replaces this friction with a unified platform that connects administration, academics, finance, operations, boarding, and parent engagement in one secure system.

## What it does

EduManage consolidates the core school operations into a single digital platform:

- Academic management: timetables, attendance, exams, assignments, quizzes, and student progress
- Financial operations: fee structures, Mpesa payments, payroll/PAYE, tax reporting, budgeting, and subscription billing
- Procurement workflows: digital requisitions, approvals, supplier orders, and inventory tracking
- Communications and calendar: direct messaging, group notices, events, notifications, and approvals
- Library and resources: digital resource management, subject libraries, and school library controls
- Boarding & operations: dorm allocation, transport scheduling, nursing records, kitchen management, and HR workflows
- AI automation: report generation, anomaly detection, timetable conflict checks, and smart suggestions for premium tiers

## Architecture & Technology

EduManage is built with modern full-stack patterns and a secure multi-tenant foundation:

- Next.js App Router for server and client rendering
- tRPC for typed API procedures and secure backend operations
- Prisma ORM with PostgreSQL and strict row-level security (RLS)
- Clerk for authentication, org-based tenancy, and role metadata
- Inngest for async jobs, notifications, reports, and AI processing
- AWS S3 presigned uploads for secure file storage
- Safaricom Daraja Mpesa integration for STK Push and C2B workflows
- Stripe for subscription billing and payment webhooks
- Gemini API for AI automation and insights

## Multi-Tenancy & Security

EduManage enforces tenant separation at every layer:

- Every database table includes `school_id UUID NOT NULL`
- PostgreSQL RLS policies prevent cross-tenant data access
- Middleware and tRPC protect routes and procedures by role
- Clerk JWT payloads carry `orgId`, `role`, and tenant metadata
- Sensitive data is never logged; audit logs record IDs only
- External webhooks are validated for Mpesa, Stripe, and Clerk

## Product Focus

EduManage is designed for Kenyan private schools with the following product focus:

- Primary users: Headteachers, accountants, teachers, parents, and students
- Initial market: private schools in Kenya using CBC and IGCSE
- Core value: replace manual, disconnected school systems with one reliable school operating system
- Subscription strategy: tiered features with a premium AI automation layer

## Documentation

Key documentation for the codebase and architecture:

- `docs/ARCHITECTURE.md` — overall system architecture
- `docs/PRD.md` — product requirements and market positioning
- `docs/ROLE_PERMISSIONS.md` — permission model and enforcement
- `docs/DATABASE_SCHEMA.md` — Prisma and PostgreSQL schema
- `docs/API_REFERENCE.md` — tRPC API and route reference
- `docs/INNGEST_JOBS.md` — asynchronous worker jobs
- `docs/ENVIRONMENT_VARIABLES.md` — runtime environment requirements

## Graphify Knowledge Graph

This repository includes a generated Graphify knowledge graph that captures the project’s architectural hubs and implementation relationships. The latest graph report is available at `graphify-out/GRAPH_REPORT.md`.

### Key knowledge graph highlights

- **Core abstractions:** `Prisma Schema`, `Blueprint — Sprint 004: Financial Management & Mpesa`, `Blueprint — Sprint 005: Communications, Calendar & Procurement`, and `Blueprint — Sprint 007: AI Automation Layer`
- **Cross-cutting themes:** multi-tenancy, financial operations, academic core, and AI automation
- **Documentation value:** Graphify surfaces connections between sprint blueprints, backlog handoffs, and implementation artifacts
- **Update note:** run `graphify update .` after code changes to refresh the graph and maintain architecture visibility

## Development status

EduManage is under active development with a sprint-driven architecture. The repository combines early product discovery, multi-tenant auth, academic core features, financial Mpesa integration, communications tooling, library resources, and AI automation planning.

## Contributing

Contributions are welcome. Use the documentation above to understand the architecture and feature scope before making changes. For development work:

1. Review the existing docs in `docs/`
2. Keep tenant and role enforcement in mind for all backend changes
3. Preserve the RLS principle: every table must include `school_id`
4. Refresh the knowledge graph with `graphify update .` when adding new modules

---

`README.md` is intended to be the repository landing page for contributors, maintainers, and reviewers. It should be updated as the project evolves and new docs are added.

