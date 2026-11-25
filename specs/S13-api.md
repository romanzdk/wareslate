# S13 Public API

## Scope
The public REST API, its login methods, webhooks, the MCP server, and importers from other tools. The web app, integrations ([S12](S12-integrations.md)), the AI assistant ([S15](S15-ai.md)) and the MCP server all use this same API.

Related modules:
- Items, types, fields: [S02](S02-hierarchy.md), [S03](S03-fields-statuses.md)
- Notes: [S04](S04-feedback-insights.md)
- Companies and users: [S05](S05-customers.md)
- Roles, workspace: [S01](S01-workspace-roles.md)
- Security, SSO, audit: [S14](S14-security.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| api_token | id, workspace_id, name, scopes[], created_by, last_used_at, expires_at | | yes | secret shown once, stored as a hash |
| oauth_client | id, name, redirect_uris[], kind `public \| confidential`, scopes[] | | yes | |
| oauth_grant | client_id, member_id, scopes[], access_expires_at, refresh_expires_at | | yes | |
| webhook_subscription | id, name, events[], url, auth_header (write-only), status, created_by | | yes | |
| webhook_delivery | id, subscription_id, event_id, attempt, status_code, at | | yes | kept 14 days |
| entity (API view) | id, type, fields{}, relationships{data[], links.next}, metadata{source{system, recordId, url}}, createdAt, updatedAt, links{self, html} | | yes | one shape for all item types |
| entity_configuration | type, fields[], relationships[], filters[] | | yes | read-only description of this workspace |
| relationship | type `parent \| child \| link \| isBlockedBy \| isBlocking`, target{id, type}, fields{} | | | rules per type: allowed targets, `single \| multiple`, mandatory |
| note (API view) | id, type `text \| conversation \| opportunity`, fields{}, relationships, metadata | | yes | |
| team, member | see R30–R32 | | | |
| import_job | id, source_kind, status, phase, counts{}, report[] | | yes | R70–R77 |

## Rules

### Conventions
- **R1** Base path is `/api/v1`. Bodies are JSON only. A single object comes as `{data: {...}}` and a list as `{data: [...], links: {next}}`. [src:1][src:10]
- **R2** The version is in the path. Features that are not stable are switched on per request with a header `Wareslate-Beta: <name>[,<name>]`. Without the header, a beta route answers 404. Unknown names are ignored and listed in a response header. [src:1; header name and 404 behavior are decisions]
- **R3** Breaking changes need a new path version. A deprecated route answers with `Deprecation` and `Sunset` headers for at least 6 months before it is removed. [src:14; 6 months is a decision]
- **R4** Cursor pagination: a list returns `links.next` as a full URL with an opaque cursor, and no `next` on the last page. Clients must not build cursors. Default page size 100, `limit` up to 200. Long relationship lists inside an entity use the same links. [src:10]
- **R5** Cursors stay valid 24 hours. A page may repeat or skip an item changed while paging; sorting is stable by `(createdAt, id)`. [decision]
- **R6** Filters on lists use GET query parameters in bracket form: `type[]`, `owner[id]`, `owner[email]`, `status[name]`, `teams[id]`, `parent[id]`, `archived`, `metadata[source][system]`, `metadata[source][recordId]`. Several filters combine with AND. Several values in one array filter combine with OR. [src:1]
- **R7** Search endpoints take `POST …/search` with a structured filter body (type, id, created/updated ranges, field values, relationships, metadata) and a `return` selector. Entity search and note search exist; note search adds full-text `query`. The filters a type supports are listed in its configuration. [src:1][src:2]
- **R8** Field selection: `fields[]=<id>` or `fields=all`. By default only fields with a value are returned. [src:1]
- **R9** Errors: `{id: <request id>, errors: [{code, title, detail, source{pointer|parameter}}]}`. `code` is dotted, for example `validation.field.required`. Status codes used: 400, 401, 403, 404, 408, 409, 422, 429, 500. A merged or moved entity answers 308 with `Location`. [src:1]
- **R10** Every response carries `X-Request-Id`, the same value as `id` in errors. [decision]
- **R11** Rate limit: 50 requests per second per token, counted over a sliding 1 second window. Responses carry `X-RateLimit-Limit` and `X-RateLimit-Remaining`. Above the limit: 429 with `Retry-After` in seconds. [src:9; sliding window is a decision]
- **R12** Writes accept an `Idempotency-Key` header. The same key and same body within 24 hours returns the first result. [decision]
- **R13** Times are ISO 8601 UTC. IDs are UUIDs. [decision]

### Entities and configuration
- **R14** `/entities` is one generic resource for every item type (products, components, features, objectives, releases, companies, users and so on). The `type` is an open string, so new types need no API change. [src:1][src:12]
- **R15** `/entities/configurations` lists every type in the workspace. `/entities/configurations/{type}` returns that type's fields (id, name, kind, options), relationship rules and supported filters. Clients must read it instead of hard-coding fields. [src:1]
- **R16** Custom field values are read and written inline in `fields`, keyed by the field id. Allowed values for select, status and tag fields are listed at `/entities/fields/{id}/values`, inline when a field has 1000 values or fewer. [src:1]
- **R17** Creating an entity needs `type` and the fields mandatory for that type. Mandatory relationships (for example a parent) are set in the same request. A missing one gives 422 pointing at the relationship. [src:1]
- **R18** Update is `PATCH` with either `fields` (replace those fields) or `patch` (a list of operations `set`, `addItems`, `removeItems`, `clear` on a path). Sending both gives 400. [src:1]
- **R19** Values can be given by id or by name for status, select, tag and team fields, and members by id or email. A name that matches nothing or more than one value gives 422. [src:1; ambiguity rule is a decision]
- **R20** Deleting removes the entity for good. Archiving is a field change, not a delete. Delete needs the delete scope. [src:1]
- **R21** Field value kinds in the API: text, number, boolean, date, url, rich text (HTML), status, member, teams, tags, single select, multi select, timeframe (start, end, granularity day/month/quarter/year), health (status, mode, comment), progress (value, mode `manual \| statusBased \| calculated`), calculated (read-only). [src:1]
- **R22** Each value may carry `source` and `isViewableOnly` markers. Writing a `calculated` or view-only value gives 422. [src:1]
- **R23** Relationships live under `/entities/{id}/relationships`. They can be listed, added and removed. A relationship can carry its own field values. [src:1][src:13]
- **R24** Relationship rules per type: allowed `targetType[]`, cardinality `single` or `multiple`, mandatory flag. A parent change is checked against these rules (S02). [src:1]
- **R25** A read-only score endpoint returns an item's impact score total (S06) for item types that have one. Beta. [src:11]
- **R26** `metadata.source` (system, recordId, url) can be set on create. A lookup by `(system, recordId)` is available as a filter (R6), so importers and syncs can find what they made. [src:1]

### Notes
- **R27** Note types: text (rich text), conversation (list of parts: external id, author name, author type `customer \| agent`, content, timestamp), and opportunity (key-value fields, read-only). [src:2]
- **R28** A note has exactly one `customer` relationship (user or company, by id or email) and many `link` relationships to items. A link is an insight (S04). `PUT …/relationships/customer` replaces the customer. [src:2]
- **R29** A note's content cannot be changed (422) while highlights inside it are linked to items. Other fields can. [src:2]

### Teams, members
- **R30** Teams have full create, read, update and delete, plus `/teams/{id}/members`. [src:4]
- **R31** Members are read-only: id, name, username, email, role, disabled, invitation pending, teams. Invites and role changes are done in the app or through SCIM (S14), not here. [src:5]
- **R32** Without the PII scopes (R41), `name` and `email` of members come back as `"[redacted]"`. [src:5][src:7]

### Login and scopes
- **R33** Every request sends `Authorization: Bearer <token>`. A token is an API token, an OAuth2 access token, or a server-to-server token. [src:7]
- **R34** API token: created by an admin in settings, with a name, chosen scopes and an optional expiry. The secret is shown once. It can be revoked at any time. Tokens act as a service account, not as a member. [src:7; service-account and expiry details are decisions]
- **R35** OAuth2 authorization code with PKCE. PKCE is required for public clients and allowed for confidential ones. Only an admin can authorize an app for the workspace. Endpoints: authorize, token, token info, revoke. [src:8; required for public is a decision since research says optional]
- **R36** Access token lives 1 day. Refresh token lives 180 days. A used refresh token is replaced, and the old one still works for 10 more minutes so a lost response does not lock the client out. [src:8]
- **R37** Server-to-server flow: the client signs a short JWT with its private key and exchanges it for an access token. No member is involved, and scopes are limited to those granted to the client. [src:7]
- **R38** MCP clients use the authorization code flow with PKCE and dynamic client registration. [src:7]
- **R39** Scopes are per resource and action: `entities:read|write|delete`, `fields:write`, `notes:read|write|delete`, `members:read`, `teams:read|write|delete`, `webhooks:read|write|delete`, `plugin-integrations:read|write|delete`, `jira-integrations:read`, `analytics:read`. A call without the needed scope gets 403 naming the scope. [src:7][src:8]
- **R40** Scope names follow one pattern, `<resource>:<action>`. Aliases with other spellings are not accepted. [decision]
- **R41** PII scopes: `members:pii:read` and `users:pii:read`. They unlock names and emails of members and of customer users. Without them these values are `"[redacted]"` in reads, and cannot be used as filter values (filter by id instead). Writes of these fields need the normal write scope. [src:5][src:7]
- **R42** A token never gets more than the member who authorized it can do in the app (S01). [decision]
- **R43** Revoking a token or grant takes effect within 60 seconds. [decision]

### Webhooks
- **R44** A subscription has a name, a list of event types, a public https URL, and an optional `authorization` header value that Wareslate sends with each delivery. The header is write-only. [src:3]
- **R45** Events: entity created, updated, deleted for every item type (one event name per type and action, for example `feature.created`), note created, updated, deleted, insight created and deleted, relationship updated, custom field value updated, release assignment updated. Company and user events are added for the types that the workspace has. [src:3; company and user events are a decision since sources do not list them]
- **R46** Payload is thin: `{data: {id, eventType, links: {target}, updatedAttributes[]}}`. The receiver fetches the full object with the link. Insight events link to the note and the item. [src:3]
- **R47** Webhooks are scoped to the app that made them. One app cannot read or delete another app's subscriptions (404). [src:3]
- **R48** A subscription can be edited (URL, events, header). [decision]
- **R49** Delivery order is not promised. Each event has a unique `id` so receivers can ignore repeats. [decision]
- **R50** Retries (decision): a delivery succeeds on any 2xx within 10 seconds. Otherwise it is retried 8 times with waits of 10 s, 1 min, 5 min, 30 min, 1 h, 3 h, 6 h, 12 h (about 23 hours in all). After that the event is dropped and recorded as `failed`. [decision]
- **R51** After 3 days with no successful delivery, the subscription is set to `disabled` and its creator is told. Re-enabling is manual. [decision]
- **R52** Signatures (decision): each delivery has `Wareslate-Signature: t=<unix time>,v1=<hmac sha256 of "t.body">`, using a per-subscription secret shown at creation. Receivers should reject a timestamp older than 5 minutes. The `authorization` header (R44) stays as a second, optional check. [decision]
- **R53** Events are written to an outbox table in the same database transaction as the change, so no change is lost if the server stops. [decision; see architecture]
- **R54** `GET /webhooks/{id}/deliveries` lists recent deliveries with status codes. A failed delivery can be re-sent. [decision]

### MCP server
- **R55** Wareslate runs an MCP server that offers tools built on this API: search items and notes, read an item with its relationships, create and update items, create notes, read configuration. It has no extra rights beyond the API. [src:1; scope of the tool list is a decision]
- **R56** The MCP server logs in with R38. Tool calls are limited by the granted scopes. Tools that write are marked as such so clients can ask for approval. [decision]
- **R57** MCP calls count against the same rate limit as the token (R11). [decision]

### Other endpoints
- **R58** Plugin integration endpoints (S12 R50–R55) need the `plugin-integrations` scopes. Jira connection endpoints are read-only. [src:6]
- **R59** An analytics endpoint returns one row per member per day: role, active flag, and counts of items, notes and boards created or opened. It needs `analytics:read`. [src:15]
- **R60** The API does not cover (v1): board and view settings, comments, audit log, workspace settings, member invites, defining new custom fields. These stay in the app. Field definitions can be added later as a decision. [src:1; list is from research]

### Importers from other tools
- **R70** Importers read the other tool through its public API with a key or OAuth token given by an admin. Nothing is scraped, and no login screen is automated. The key is used for the import only and is deleted when the job ends. [decision]
- **R71** An import is a job with phases in this fixed order: (1) configuration, (2) entities, (3) notes, (4) relationships. [decision]
- **R72** Phase 1, configuration: read item types, custom fields, statuses, teams and members of the source. Show a plan of what would be created and what would be matched to existing configuration. Fields are matched by name and kind. The admin confirms or edits the plan before phase 2. [decision]
- **R73** Phase 2, entities: create items in parent-first order. Each gets `metadata.source{system, recordId}` set to the source id. [decision]
- **R74** Phase 3, notes: create notes with their customer. Companies and users are created first or matched by email or external id (S05). [decision]
- **R75** Phase 4, relationships: links between items, notes to items, and dependencies, created from the source ids. A link whose target did not import is skipped and reported. [decision]
- **R76** Jobs can be re-run. Items are matched by `(system, recordId)` and updated instead of duplicated. A dry run only makes the plan and the report. [decision]
- **R77** The final report lists everything not imported, each with source id, kind and reason (unsupported field kind, missing parent, name clash, rate limited, source denied access). Counts are shown per phase: read, created, updated, skipped, failed. The report can be downloaded as CSV. [decision]
- **R78** Importers must respect the source's rate limits, and stop with status `paused` when blocked. [decision]
- **R79** Source tools supported first: those with a public API for items and feedback (issue trackers and other product tools). Each is an adapter that maps to the same four phases. [decision]

## User actions
- Admin: create API tokens and OAuth clients, authorize apps, manage webhooks, run importers, grant PII scopes.
- Maker: create tokens for themselves with scopes no greater than their own (R42); run importers for their own items if allowed.
- Contributor, Viewer: use the API only through apps an admin authorized; limited by their app permissions.

See [S01](S01-workspace-roles.md) for the role model.

## Acceptance tests
- **T1 (R1, R4)** Given 250 items and `limit` 100, when the client follows `links.next`, then it gets 100, 100, 50 and the last page has no `next`.
- **T2 (R9)** Given a create with a missing mandatory field, then 422 with `source.pointer` at that field and a dotted code.
- **T3 (R11)** Given 60 calls in one second from one token, then the extra calls get 429 with `Retry-After`.
- **T4 (R15, R16)** Given a workspace with a custom select field, then its configuration lists the field and the values endpoint lists its options.
- **T5 (R18)** Given a PATCH with both `fields` and `patch`, then 400.
- **T6 (R19)** Given status "Planned" exists once, when set by name, then it works; given a name that matches two values, then 422.
- **T7 (R32, R41)** Given a token without PII scopes, when it reads members, then names and emails are `"[redacted]"`; with `members:pii:read` they are real values.
- **T8 (R35)** Given a public client that skips the PKCE challenge, then the authorization is refused.
- **T9 (R36)** Given a refresh token used once, when it is used again 5 minutes later, then it still works; after 11 minutes it does not.
- **T10 (R39)** Given a token with `notes:read`, when it creates a note, then 403 naming `notes:write`.
- **T11 (R46)** Given a feature update, then the receiver gets a payload with the id, event type and a link, and no field values.
- **T12 (R50, R51)** Given a receiver that always answers 500, then 9 attempts are made over about 23 hours, the event is `failed`, and after 3 days the subscription is `disabled`.
- **T13 (R52)** Given a delivery with a timestamp 10 minutes old, then a correct receiver rejects it.
- **T14 (R29)** Given a note with a linked highlight, when its content is edited, then 422.
- **T15 (R6)** Given a filter `metadata[source][system]=x` and `recordId=7`, then at most the matching items return.
- **T16 (R71, R77)** Given an import where 3 items have an unsupported field kind, then phases run in order and the report lists the 3 items with reasons.
- **T17 (R76)** Given an import run twice, then the second run creates 0 items and updates the changed ones.
- **T18 (R2)** Given a beta route, when called without the beta header, then 404; with it, 200.
- **T19 (R12)** Given two POSTs with the same idempotency key and body, then one entity is created.
- **T20 (R47)** Given app A's subscription, when app B reads it, then 404.

## Tier
API, webhooks and importers: all paid tiers. MCP server: all paid tiers. Higher rate limits, longer delivery log: top tier. [decision]

## Open questions
1. Retries and signatures (R50, R52) are our own picks. Are 8 retries over 23 hours and HMAC enough for the first release?
2. Can importance on an insight be written through the API (R28)? Research did not settle this.
3. Should API tokens act as a service account (R34), or as the member who made them?
4. Should field definitions (custom fields, statuses) be creatable by API in v1 (R60)?
5. Which source tools get an importer first (R79)?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S13).
