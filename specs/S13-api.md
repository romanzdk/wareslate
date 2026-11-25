# S13 Public API

## Scope
The public REST API, its login methods, webhooks, the MCP server, and importers from other tools. The web app, integrations ([S12](S12-integrations.md)), the AI assistant ([S15](S15-ai.md)) and the MCP server all call this same API.

The API is built on public standards: JSON Patch (RFC 6902) for partial updates, problem details (RFC 9457) for errors, OAuth 2.0 with PKCE, and HMAC-signed webhooks.

Related modules:
- Items, kinds, fields: [S02](S02-product-tree.md), [S03](S03-fields-statuses.md)
- Feedback entries and evidence: [S04](S04-feedback.md)
- Accounts and contacts: [S05](S05-customers.md)
- Roles, workspace: [S01](S01-workspace-roles.md)
- Security, SSO, audit: [S14](S14-security.md)

## Entities

| entity | field | type | required | remarks |
|---|---|---|---|---|
| api_token | id, workspace_id, name, scopes[], created_by, last_used_at, expires_at | | yes | secret shown once, stored as a hash |
| oauth_client | id, name, redirect_uris[], kind `public \| confidential`, scopes[] | | yes | |
| oauth_grant | client_id, member_id, scopes[], access_expires_at, refresh_expires_at | | yes | |
| webhook_subscription | id, name, events[], kinds[], url, auth_header (write-only), status, created_by | | yes | `kinds` is an optional filter on item kinds |
| webhook_delivery | id, subscription_id, event_id, attempt, status_code, at | | yes | kept 14 days |
| item (API view) | id, kind, fields{}, relations{}, origin{tool, external_id, url}, created_at, updated_at, urls{self, web} | | yes | one shape for every item kind, including accounts and contacts |
| schema_kind | kind, fields[], relation_rules[], filters[] | | yes | read-only description of this workspace |
| relation | kind `parent \| child \| related \| blocked_by \| blocks`, target{id, kind}, fields{} | | | rules per kind: allowed targets, `one \| many`, mandatory |
| feedback (API view) | id, kind `message \| thread \| deal`, fields{}, customer, evidence[], origin, created_at, updated_at | | yes | |
| team, member | see R30–R32 | | | |
| import_job | id, source_tool, status, phase, counts{}, report[] | | yes | R70–R79 |

## Rules

### Conventions
- **R1** Base path is `/api/v1`. Bodies are JSON. A single object is returned as is, with no wrapper. A list is `{items: [...], next_cursor, has_more}`. [src:1][src:10]
- **R2** The version is in the path. A feature that is not yet stable is switched on per request with the header `Wareslate-Preview: <name>[,<name>]`. Without it, a preview route answers 404. Unknown names are ignored and echoed in the response header `Wareslate-Preview-Ignored`. [src:1; header name and 404 behavior are decisions]
- **R3** A breaking change needs a new path version. A route that is being retired answers with `Deprecation` and `Sunset` headers for at least 6 months before it goes away. [src:14; 6 months is a decision]
- **R4** Lists use cursors. A page returns `next_cursor` (an opaque string) and `has_more`. The client sends the cursor back as `?cursor=`. On the last page `has_more` is `false` and `next_cursor` is `null`. Clients must not build cursors. Default page size is 100, and `limit` goes up to 200. Long relation lists inside an item use the same list shape. [src:10]
- **R5** A cursor stays valid for 24 hours. A page may repeat or skip an item that changed during paging. Order is stable by `(created_at, id)`. [decision]
- **R6** List filters are plain query parameters: `kind`, `owner_id`, `owner_email`, `status`, `team_id`, `parent_id`, `archived`, `origin_tool`, `origin_external_id`. Different filters combine with AND. A parameter repeated with several values combines them with OR (for example `?kind=feature&kind=slice`). [src:1]
- **R7** Search endpoints take `POST /items/search` or `POST /feedback/search` with a structured filter body (kind, id, created or updated ranges, field values, relations, origin) and an `include` list. Feedback search also takes a full-text `query`. The filters a kind supports are listed in its schema entry. [src:1][src:2]
- **R8** Field selection: `?fields=<id>,<id>` or `?fields=*`. By default only fields that have a value are returned. [src:1]
- **R9** Errors use problem details (RFC 9457) with content type `application/problem+json`: `{type, title, status, detail, request_id, errors: [{pointer, reason}]}`. `type` is a path such as `/problems/missing_field`, and it is the stable code clients should match on. `errors` lists each bad field as a JSON Pointer into the request body, or a query parameter name. Status codes used: 400, 401, 403, 404, 408, 409, 415, 422, 429, 500. A merged or moved item answers 308 with `Location`. [src:1]
- **R10** Every response carries `X-Request-Id`, the same value as `request_id` in problem bodies. [decision]
- **R11** Rate limit: each token may make 50 calls in any sliding 1-second window. Responses carry `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset`. Past the limit the answer is 429 with `Retry-After` in seconds. [src:9; sliding window and header names are decisions]
- **R12** Writes accept an `Idempotency-Key` header. The same key with the same body within 24 hours returns the first result. [decision]
- **R13** Times are ISO 8601 UTC. IDs are UUIDs. Field names are `snake_case`. [decision]

### Items and schema
- **R14** `/items` is one generic resource for every item kind (products, modules, features, slices, bets, objectives, releases, accounts, contacts and so on). `kind` is an open string, so a new kind needs no API change. [src:1][src:12]
- **R15** `GET /schema` lists every kind in the workspace. `GET /schema/{kind}` returns that kind's fields (id, name, type, options), relation rules and supported filters. Clients read it and do not hard-code fields. [src:1]
- **R16** Custom field values are read and written inline in `fields`, keyed by field id. Allowed values of select, status and tag fields are listed at `/schema/fields/{id}/options`, and appear inline when a field has 1000 options or fewer. [src:1]
- **R17** Creating an item needs `kind` and the fields that are mandatory for that kind. Mandatory relations (for example a parent) are set in the same request. A missing one gives 422 with a pointer at the relation. [src:1]
- **R18** Partial update is `PATCH /items/{id}` with content type `application/json-patch+json` and a JSON Patch (RFC 6902) list. Supported operations are `add`, `remove` and `replace`, on paths such as `/fields/<field_id>`. List-valued fields (tags, multi select, teams) act as sets keyed by option id: `add` on `/fields/<field_id>/-` appends a value, and `remove` on `/fields/<field_id>/<option_id>` takes one away. `replace` of a whole field sets it, and `remove` of a whole field clears it. Other operations, or a body that is not a patch list, give 400 or 415. A patch applies fully or not at all. [src:1]
- **R19** Status, select, tag and team values can be given by id or by name, and members by id or email. A name that matches nothing, or more than one value, gives 422. [src:1; ambiguity rule is a decision]
- **R20** Delete removes the item for good. Archiving is a field change, not a delete. Delete needs the `items.delete` scope. [src:1]
- **R21** Field value types in the API: text, number, boolean, date, url, rich_text (HTML), status, member, teams, tags, select, multi_select, date_range (`start`, `end`, `precision` of day, month, quarter or year), check_in (`status` of green, amber, red or none, plus `comment`), progress (`value`, `mode` of `manual`, `from_status` or `computed`), computed (read-only). [src:1]
- **R22** A value may carry the markers `set_by` (`member`, `integration` or `computed`) and `read_only`. Writing a `computed` or `read_only` value gives 422. [src:1]
- **R23** Relations live under `/items/{id}/relations`. They can be listed, added and removed, and a relation can carry its own field values. [src:1][src:13]
- **R24** Relation rules per kind: allowed target kinds, cardinality `one` or `many`, and a mandatory flag. A parent change is checked against these rules (S02). [src:1]
- **R25** A read-only `GET /items/{id}/demand_score` returns the item's demand score total (S06), for kinds that have one. Preview. [src:11]
- **R26** `origin{tool, external_id, url}` can be set on create. Importers and syncs find what they made through the `origin_tool` and `origin_external_id` filters (R6). [src:1]

### Feedback
- **R27** Feedback kinds: `message` (rich text), `thread` (a list of parts, each with `external_id`, `author_name`, `author_side` of `customer` or `agent`, `body`, `sent_at`) and `deal` (key-value fields, read-only). [src:2]
- **R28** A feedback entry has exactly one `customer` (an account or a contact, given by id or email) and many `evidence` links to items. Each evidence link carries a `need` level (S04). `PUT /feedback/{id}/customer` replaces the customer. [src:2]
- **R29** The content of an entry cannot be changed (422) while excerpts inside it are linked to items. Other fields can. [src:2]

### Teams, members
- **R30** Teams support create, read, update and delete, plus `/teams/{id}/members`. [src:4]
- **R31** Members are read-only: `id`, `username`, `display_name`, `email`, `role`, `disabled`, `invite_pending`, `team_ids`. Invites and role changes are done in the app or through SCIM (S14), not here. [src:5]
- **R32** Without the personal-data scope (R41), `display_name` and `email` of members come back as `"[hidden]"`. [src:5][src:7]

### Login and scopes
- **R33** Every request sends `Authorization: Bearer <token>`. The token is an API token, an OAuth2 access token, or a server-to-server token. [src:7]
- **R34** API token: an admin creates it in settings with a name, chosen scopes and an optional expiry. The secret is shown once, and the token can be revoked at any time. A token acts as a service account, not as a member. [src:7; service-account and expiry details are decisions]
- **R35** OAuth2 auth-code grant, protected by PKCE. PKCE is required for public clients and allowed for confidential ones. Only an admin can authorize an app for the workspace. Endpoints: authorize, token, introspect, revoke. [src:8; required for public is a decision since research says optional]
- **R36** An access token lives 1 day and a refresh token 180 days. Using a refresh token issues a new one, and the old one keeps working for 10 more minutes, so a lost response does not lock the client out. [src:8]
- **R37** Server-to-server flow: the client signs a short-lived JWT with its private key and trades it for an access token (RFC 7523). No member takes part, and scopes are limited to those granted to the client. [src:7]
- **R38** MCP clients log in with the PKCE code flow and register themselves with dynamic client registration (RFC 7591). [src:7]
- **R39** Scopes are per resource and action: `items.read|write|delete`, `schema.write`, `feedback.read|write|delete`, `members.read`, `teams.read|write|delete`, `webhooks.read|write|delete`, `external_actions.read|write|delete`, `jira_links.read`, `reports.read`. A call without the scope it needs gets 403 with problem type `/problems/missing_scope` and the needed scope in `required_scope`. [src:7][src:8]
- **R40** Scope names use one pattern, `<resource>.<action>`. Other spellings are not accepted as aliases. [decision]
- **R41** One personal-data scope, `people.personal_data`, unlocks the names and emails of members, contacts and accounts' people. Without it these values read as `"[hidden]"` and cannot be used as filter values (filter by id instead). Writing these fields needs only the normal write scope. [src:5][src:7]
- **R42** A token never gets more rights than the member who authorized it has in the app (S01). [decision]
- **R43** Revoking a token or grant takes effect within 60 seconds. [decision]

### Webhooks
- **R44** A subscription has a name, a list of events, an optional `kinds` filter, a public https URL, and an optional `auth_header` value that Wareslate sends with each delivery. The header is write-only. [src:3]
- **R45** Events: `item.created`, `item.updated`, `item.deleted`; `feedback.created`, `feedback.updated`, `feedback.deleted`; `evidence.created`, `evidence.deleted`; `relation.updated`; `field_value.updated`; `release_assignment.updated`. Account and contact changes arrive as `item.*` events with their kind. [src:3; account and contact events are a decision since sources do not list them]
- **R46** The payload is thin: `{event, item_id, changed_fields[], occurred_at}`. The receiver fetches the full object from the API. Feedback events carry `feedback_id` instead of `item_id`. Evidence events carry both. [src:3]
- **R47** A webhook belongs to the app that made it. One app cannot read or delete another app's subscriptions (404). [src:3]
- **R48** A subscription can be edited (URL, events, kinds, header). [decision]
- **R49** Delivery order is not promised. Every delivery has a unique `Wareslate-Event-Id` header, so receivers can drop repeats. [decision]
- **R50** Retries (decision): a delivery counts as done on any 2xx within 10 seconds. Otherwise it is retried 8 times, waiting 10 s, 1 min, 5 min, 30 min, 1 h, 3 h, 6 h, 12 h (about 23 hours in all). After that the event is dropped and recorded as `failed`. [decision]
- **R51** After 3 days with no successful delivery, the subscription is set to `disabled` and its creator is told. Switching it back on is manual. [decision]
- **R52** Signatures (decision): each delivery has `Wareslate-Signature: t=<unix time>,v1=<hex HMAC-SHA256 of "t.body">`, using a per-subscription secret shown at creation. Receivers should reject a timestamp older than 5 minutes. The `auth_header` (R44) stays as a second, optional check. [decision]
- **R53** Events are written to an outbox table in the same database transaction as the change, so no change is lost if the server stops. [decision; see architecture]
- **R54** `GET /webhooks/{id}/deliveries` lists recent deliveries with status codes. A failed delivery can be sent again. [decision]

### MCP server
- **R55** Wareslate runs an MCP server with tools built on this API: search items and feedback, read an item with its relations, create and update items, create feedback entries, read the schema. It has no rights beyond the API. [src:1; scope of the tool list is a decision]
- **R56** The MCP server logs in as in R38. Tool calls are limited by the granted scopes. Tools that write are marked, so clients can ask for approval. [decision]
- **R57** MCP calls count against the same rate limit as the token (R11). [decision]

### Other endpoints
- **R58** External action endpoints (S12 R50–R55) need the `external_actions.*` scopes. Jira link endpoints are read-only. [src:6]
- **R59** `GET /reports/member_activity` returns one row per member per day: role, active flag, and counts of items, feedback entries and views created or opened. It needs `reports.read`. [src:15]
- **R60** Not in the v1 API: view settings, comments, audit log, workspace settings, member invites, and defining new custom fields. These stay in the app. Field definitions can be added later as a decision. [src:1; list is from research]

### Importers from other tools
- **R70** Importers read the other tool through its public API, with a key or OAuth token given by an admin. Nothing is scraped, and no login screen is automated. The key is used for the import only and is deleted when the job ends. [decision]
- **R71** An import is a job that runs four phases in this fixed order: (1) schema, (2) items, (3) feedback, (4) relations. [decision]
- **R72** Phase 1, schema: read the source's item kinds, custom fields, statuses, teams and members. Show a plan of what would be created and what would match existing configuration. Fields match by name and type. The admin confirms or edits the plan before phase 2. [decision]
- **R73** Phase 2, items: create items parent first. Each gets `origin{tool, external_id}` set to the source id. [decision]
- **R74** Phase 3, feedback: create feedback entries with their customer. Accounts and contacts are created first, or matched by email or external id (S05). [decision]
- **R75** Phase 4, relations: links between items, feedback to items, and dependencies, built from the source ids. A link whose target was not imported is skipped and reported. [decision]
- **R76** A job can be run again. Items are matched by `(origin.tool, origin.external_id)` and updated, not duplicated. A dry run only builds the plan and the report. [decision]
- **R77** The final report lists everything that was not imported, each with source id, kind and reason (unsupported field type, missing parent, name clash, rate limited, source denied access). Counts are shown per phase: read, created, updated, skipped, failed. The report can be downloaded as CSV. [decision]
- **R78** Importers respect the source's rate limits, and stop with status `paused` when blocked. [decision]
- **R79** Source tools supported first: those with a public API for items and feedback (issue trackers and other product tools). Each one is an adapter onto the same four phases. [decision]

## User actions
- Admin: create API tokens and OAuth clients, authorize apps, manage webhooks, run importers, grant `people.personal_data`.
- Editor: create tokens for themselves with scopes no greater than their own (R42); run importers for their own items if allowed.
- Collaborator, Reader: use the API only through apps an admin authorized, within those apps' permissions.

See [S01](S01-workspace-roles.md) for the role model.

## Acceptance tests
- **T1 (R1, R4)** Given 250 items and `limit` 100, when the client pages with `next_cursor`, then it gets 100, 100, 50 and the last page has `has_more` false and `next_cursor` null.
- **T2 (R9)** Given a create with a missing mandatory field, then 422 `application/problem+json` with an `errors` entry whose `pointer` names that field and a `type` of `/problems/missing_field`.
- **T3 (R11)** Given 60 calls in one second from one token, then the extra calls get 429 with `Retry-After`.
- **T4 (R15, R16)** Given a workspace with a custom select field, then its schema entry lists the field and the options endpoint lists its values.
- **T5 (R18)** Given a PATCH with content type `application/json` and a plain object body, then 415; given a patch list with an unsupported operation, then 400 and nothing is changed.
- **T6 (R19)** Given status "Planned" exists once, when set by name, then it works; given a name that matches two values, then 422.
- **T7 (R32, R41)** Given a token without `people.personal_data`, when it reads members, then names and emails are `"[hidden]"`; with the scope they are real values.
- **T8 (R35)** Given a public client that skips the PKCE challenge, then the authorization is refused.
- **T9 (R36)** Given a refresh token used once, when it is used again 5 minutes later, then it still works; after 11 minutes it does not.
- **T10 (R39)** Given a token with `feedback.read`, when it creates a feedback entry, then 403 with `required_scope` `feedback.write`.
- **T11 (R46)** Given a feature update, then the receiver gets `{event, item_id, changed_fields, occurred_at}` and no field values.
- **T12 (R50, R51)** Given a receiver that always answers 500, then 9 attempts are made over about 23 hours, the event is `failed`, and after 3 days the subscription is `disabled`.
- **T13 (R52)** Given a delivery with a timestamp 10 minutes old, then a correct receiver rejects it.
- **T14 (R29)** Given a feedback entry with a linked excerpt, when its content is edited, then 422.
- **T15 (R6)** Given the filters `origin_tool=x` and `origin_external_id=7`, then at most the matching items return.
- **T16 (R71, R77)** Given an import where 3 items have an unsupported field type, then phases run in order and the report lists the 3 items with reasons.
- **T17 (R76)** Given an import run twice, then the second run creates 0 items and updates the changed ones.
- **T18 (R2)** Given a preview route, when called without `Wareslate-Preview`, then 404; with it, 200.
- **T19 (R12)** Given two POSTs with the same idempotency key and body, then one item is created.
- **T20 (R47)** Given app A's subscription, when app B reads it, then 404.

## Tier
API, webhooks and importers: all paid plans. MCP server: all paid plans. Higher rate limits and a longer delivery log: top plan. [decision]

## Open questions
1. Retries and signatures (R50, R52) are our own picks. Are 8 retries over 23 hours and HMAC enough for the first release?
2. Can `need` on an evidence link be written through the API (R28)? Research did not settle this.
3. Should API tokens act as a service account (R34), or as the member who made them?
4. Should field definitions (custom fields, statuses) be creatable by API in v1 (R60)?
5. Which source tools get an importer first (R79)?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S13).
