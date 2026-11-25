# Architecture

## Principles
- **Thin client.** The browser loads only what a view needs, page by page. Never the whole workspace.
- **Modular monolith.** One deployable backend, with one module per spec area (S01–S15) and strict boundaries between modules. Split it up only when there is a real reason.
- **Server-authoritative real-time.** The server pushes "item X changed". Clients refetch what they show. Pages are the one exception: they use a CRDT for co-editing.
- **One public API.** The web app, integrations, the AI assistant and the MCP server all use the same API.

## Stack

| Layer | Choice |
|---|---|
| Frontend | React + TypeScript, TanStack Query, virtualized table, dnd-kit, own design system |
| Rich text and co-editing | Tiptap / ProseMirror + Yjs, with a Hocuspocus server |
| Backend | Python 3.13, FastAPI, SQLAlchemy |
| Database | Postgres: `jsonb` for custom field values, full-text search + `pg_trgm`, `pgvector` for semantic search |
| Jobs and events | Postgres-backed queue, plus an outbox table for webhooks and integrations |
| Real-time | WebSocket gateway (e.g. Centrifugo) |
| Auth | OIDC / SAML, SCIM endpoint, OAuth2 provider for the public API |
| AI | Claude API, a tool-calling agent over the public API, and an MCP server reusing the same tools |
| Dev setup | Docker Compose |

## Core data model

Item kinds are configuration, not hard-coded tables. This keeps custom fields, renamed kinds and new kinds cheap.

```
workspace ─┬─ member(role: admin|editor|collaborator|reader) ── team ── space(visibility, access)
           ├─ item_kind(key, label, allowed_parents, fields[])       # product, module, feature, slice,
           │                                                         # bet, objective, key_result, release_track, release
           ├─ field_def(kind, options, kinds[], expression?)          # incl. criterion, computed
           ├─ status(label, category, kinds[])
           ├─ item(id, kind, title, body_page, status, owner, date_range, size, values jsonb, archived_at)
           ├─ relation(from_item, to_item, kind: child_of|related|blocks, values jsonb)
           ├─ check_in(item, rag: green|amber|red|none, comment, mode, author, at)
           ├─ feedback(kind: message|thread|deal, body, origin{tool, external_id, url}, contact|account, owner, state: new|triaged, tags)
           ├─ evidence(feedback, item, excerpt_range, need: none|low|medium|high|unspecified)
           ├─ account / contact / cohort(rule) / account_field
           ├─ view(kind: table|timeline|kanban|feedback_view|account|contact, config jsonb, sharing)
           ├─ page(yjs_state, sharing) / comment / mention / watcher
           ├─ public_roadmap ── roadmap_section ── roadmap_list ── roadmap_entry(item, public fields) ── vote
           ├─ integration(kind, config) ── external_link(item, external_id, sync_state)
           ├─ webhook / api_client / oauth_token
           └─ audit_event / item_history
```

Scores (demand score, criteria, computed fields, size roll-ups) are calculated on the server and cached per item. A change event recalculates them up the tree.

## API conventions
- REST + JSON. A single object is returned as is. Lists are `{items: [], next_cursor, has_more}`, paged with `?cursor=`.
- One generic `/items` resource, plus `/schema` describing each workspace's kinds and fields.
- Partial updates: `PATCH` with a JSON Patch list (RFC 6902).
- Errors: problem details (RFC 9457, `application/problem+json`).
- OAuth2 with scopes per resource (for example `items.read`), and a rate limit per token. Unstable routes need the `Wareslate-Preview` header.
- Thin webhooks (`{event, item_id, changed_fields, occurred_at}`), signed with `Wareslate-Signature`: the receiver fetches the full object.

## Milestones

| M | Scope | Specs |
|---|---|---|
| M0 | Skeleton: sign-in, workspace, members, roles, CI, Docker Compose, API conventions | S01, S13 |
| M1 | Product tree, custom fields, statuses, table view | S02, S03, S09 |
| M2 | Feedback, evidence, accounts and contacts, demand score; email and CSV import; feedback view | S04, S05, S06 |
| M3 | Public API, webhooks, importers from other tools | S13 |
| M4 | Objectives, key results, bets, check-ins, releases, dependencies, criteria, computed fields | S06, S07, S08 |
| M5 | Timeline and kanban views, sharing, export | S09 |
| M6 | Pages with live co-editing, comments, mentions, notifications | S10 |
| M7 | Public roadmap with voting | S11 |
| M8 | Integrations: issue trackers (two-way), chat and support tools (inbound) | S12 |
| M9 | AI assistant, clusters, patterns, spec drafting, MCP server | S15 |
| M10 | Enterprise: SAML, SCIM, MFA, custom roles, audit log | S14 |

MVP = M0–M3.
