# Architecture

## Principles
- **Thin client.** The browser loads only what a view needs, page by page. Never the whole workspace.
- **Modular monolith.** One deployable backend, with one module per spec area (S01–S15) and strict boundaries between modules. Split it up only when there is a real reason.
- **Server-authoritative real-time.** The server pushes "entity X changed". Clients refetch what they show. Documents are the one exception: they use a CRDT for co-editing.
- **One public API.** The web app, integrations, the AI assistant and the MCP server all use the same API.

## Stack

| Layer | Choice |
|---|---|
| Frontend | React + TypeScript, TanStack Query, virtualized grid, dnd-kit, own design system |
| Rich text and co-editing | Tiptap / ProseMirror + Yjs, with a Hocuspocus server |
| Backend | Python 3.13, FastAPI, SQLAlchemy |
| Database | Postgres: `jsonb` for custom field values, full-text search + `pg_trgm`, `pgvector` for semantic search |
| Jobs and events | Postgres-backed queue, plus an outbox table for webhooks and integrations |
| Real-time | WebSocket gateway (e.g. Centrifugo) |
| Auth | OIDC / SAML, SCIM endpoint, OAuth2 provider for the public API |
| AI | Claude API, a tool-calling agent over the public API, and an MCP server reusing the same tools |
| Dev setup | Docker Compose |

## Core data model

Item types are configuration, not hard-coded tables. This keeps custom fields, renamed types and new types cheap.

```
workspace ─┬─ member(role) ── team ── teamspace(access)
           ├─ entity_type(key, name, allowed_parents, fields[])      # product, component, feature, subfeature,
           │                                                         # initiative, objective, key_result, release, release_group
           ├─ field_definition(kind, options, assigned_types[], formula?)
           ├─ status(name, category, assigned_types[])
           ├─ entity(id, type, name, description_doc, status, owner, timeframe, fields jsonb, archived)
           ├─ relationship(source, target, kind: parent|link|blocks, fields jsonb)
           ├─ health_update(entity, status, comment, mode, author, at)
           ├─ note(type, content, source{system, record_id, url}, customer_ref, owner, processed, tags)
           ├─ insight(note, entity, highlight_range, importance 0..3)
           ├─ company / customer_user / segment(rule) / company_field
           ├─ board(kind: grid|timeline|columns|insights|customers, config jsonb, sharing)
           ├─ document(yjs_state, sharing) / comment / mention / follower
           ├─ portal ── portal_section ── portal_card(entity, public fields) ── vote
           ├─ integration(kind, config) ── integration_link(entity, external_id, sync_state)
           ├─ webhook_subscription / api_client / oauth_token
           └─ audit_event / entity_history
```

Scores (impact score, drivers, formulas, roll-ups) are computed by the backend and cached per entity. A change event recalculates them up the tree.

## API conventions
- REST + JSON. Bodies are wrapped as `{data: …}`, and lists as `{data: [], links: {next}}` with cursor pagination.
- One generic `/entities` resource, plus `/entities/configurations` describing each workspace's types and fields.
- Updates: `PATCH` with either `fields` (replace) or `patch` operations (`set`, `addItems`, `removeItems`, `clear`).
- Errors: `{id, errors: [{code, title, detail, source}]}`.
- OAuth2 with scopes per resource, and a rate limit per token.
- Thin webhooks (`{id, eventType, links, updatedAttributes}`): the receiver fetches the full object.

## Milestones

| M | Scope | Specs |
|---|---|---|
| M0 | Skeleton: auth, workspace, members, roles, CI, Docker Compose, API conventions | S01, S13 |
| M1 | Hierarchy, custom fields, statuses, grid board | S02, S03, S09 |
| M2 | Feedback, insights, customers, impact score; email and CSV import; insights board | S04, S05, S06 |
| M3 | Public API, webhooks, importers from other tools | S13 |
| M4 | Objectives, key results, initiatives, health roll-up, releases, dependencies, drivers, formulas | S06, S07, S08 |
| M5 | Timeline and columns boards, sharing, export | S09 |
| M6 | Documents with co-editing, comments, mentions, notifications | S10 |
| M7 | Public roadmap portal with voting | S11 |
| M8 | Integrations: issue trackers (two-way), chat and support tools (inbound) | S12 |
| M9 | AI assistant, themes, findings, spec writing, MCP server | S15 |
| M10 | Enterprise: SAML, SCIM, MFA, custom roles, audit log | S14 |

MVP = M0–M3.
