# S12 Integrations

## Scope
How Wareslate connects to outside tools. Three families:
1. **Issue trackers (two-way):** Jira in depth, then Azure DevOps, GitHub Issues, Trello, Shortcut, GitLab, Linear.
2. **Inbound feedback and data:** Slack, MS Teams, Zendesk, Intercom, Gong, email, browser extension, Salesforce, Zapier, analytics cohorts, design embeds.
3. **Integration button:** a generic plugin model so any external service can show its connection state on an item.

Related modules:
- Items, parents, children: [S02](S02-hierarchy.md)
- Fields, statuses: [S03](S03-fields-statuses.md)
- Notes and insights: [S04](S04-feedback-insights.md)
- Companies, users, segments: [S05](S05-customers.md)
- Effort (maps to story points): [S06](S06-prioritization.md)
- Releases (map to fix versions): [S08](S08-releases.md)
- Public API, webhooks, OAuth used by integrations: [S13](S13-api.md)
- SSO and SCIM (not covered here): [S14](S14-security.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| integration | id, workspace_id, kind, name | | yes | one row per connection to an outside account or project |
| | status | enum `enabled \| disabled \| error` | yes | |
| | config | jsonb | yes | project, mapping, rules; secrets are stored encrypted and never returned by the API |
| | authorized_by | member id | yes | the member whose credentials the connection uses |
| integration_link | integration_id, item_id | | yes | one item to one external record |
| | external_id, external_key, external_url | text | yes | |
| | sync_state | enum `synced \| pending \| error \| unlinked` | yes | |
| | last_synced_at, last_error | | no | |
| field_mapping | integration_id, local_field, external_field | | yes | |
| | direction | enum `two_way \| set_once` | yes | default `set_once` (R6) |
| | value_map | jsonb | no | for status, select and priority values |
| status_rule | integration_id, external_statuses[], local_status | | yes | external → local status (R12) |
| sync_event | link_id, field, side `local \| external`, old, new, at, outcome | | yes | audit log kept 30 days |
| inbound_source | id, kind, config, filter | | yes | a connected chat, helpdesk, call or CRM source |
| plugin_integration | id, type (dotted domain-style text), name, status, initial_state, action{url, auth_header} | | yes | integration button (R50–R56); auth header is write-only |
| plugin_connection | plugin_id, item_id | | yes | |
| | state | enum `initial \| progress \| connected \| error` | yes | |
| | label, hover_label, tooltip, color, target_url | | when `connected` | |
| | message, debug_detail | | when `error` | |

## Rules

### Common to all integrations
- **R1** An integration belongs to one workspace. Only an admin can connect one. A maker can use a connected integration and edit its mapping if an admin allows it. [src:1][decision on the maker part]
- **R2** Only an admin maker can switch on two-way sync on an integration. Switching it on changes nothing until a mapped field is edited. [src:1]
- **R3** Credentials belong to the member who authorized. If that member is removed or the token is revoked, the integration moves to `error` and stops syncing. It never deletes data. [decision]
- **R4** Every sync action is written to `sync_event`. The item shows the last result and the last error text. [decision]
- **R5** Failures retry with growing waits (1 min, 5 min, 30 min, 2 h, 12 h), then the link goes to `error` and an admin is told. [decision]

### Jira: push, link, import
- **R6** Each mapped field has a direction: `two_way` (kept equal at all times) or `set_once` (applied only during push, link or import). The default for a new mapping is `set_once`. [src:1; default is a decision]
- **R7** Push: a chosen item becomes a new Jira issue in the selected project, using the mapping. The link is stored. [src:1]
- **R8** Link: an item is tied to an existing Jira issue, found by key or by search. The mapped `set_once` and `two_way` fields are applied once at link time, in the direction the user chooses for the first copy (item to issue or issue to item). [src:1; the choice is a decision]
- **R9** Import: chosen Jira issues become new items of a chosen item type, each already linked. Items with no link are not updated by later sync, so they need a one-time import or a push. [src:1]
- **R10** Field mapping works with one Jira project at a time. Required Jira fields must be mapped before a push can create an issue. [src:1]
- **R11** A mapping is allowed only between fields of the same type. A custom field in Jira pairs only with a Wareslate custom field of the matching type (text, number, date, single select, multi select, member, url, label list). [src:1]

### Jira: field mapping
- **R12** Status: rules pair one or more Jira statuses with one Wareslate status. Rules apply only to changes made after the rule is enabled, not to existing values. A link made to an existing issue does not change status until the issue's status changes. Statuses with no rule are ignored. [src:2]
- **R13** Status flows from Jira to Wareslate, for pushed and linked items and for initiatives. Wareslate status changes do not write to Jira. [src:2; scope wording is from the research inventory; no write-back is a decision]
- **R14** A rule that points at a deleted Jira status or deleted Wareslate status is shown as broken and stays inactive until fixed. An incomplete rule cannot be saved. [src:2]
- **R15** Name maps to summary. Description maps to the description, and keeps formatting only if the Jira field supports rich text; otherwise it is sent as plain text. [src:1]
- **R16** Timeframe start and end map to Jira start date and due date. [src:1]
- **R17** Release maps to fix version. Matching is by name. A missing version is created in the Jira project on push only if the admin allowed it in the mapping, else the field is left empty and a warning is logged. [src:1; the create option is a decision]
- **R18** Owner maps to reporter, only when a Jira user with the same email exists. Otherwise the field is left empty. [src:1]
- **R19** Effort maps to story points. Effort goes to Jira only when the item's effort is in manual mode (S06). A Jira value always arrives as manual effort, even if Jira shows a rolled-up number. [src:1] (see also S06 R26)
- **R20** Tags map to Jira labels. Team maps to a Jira multi-select or group-picker field. A select field can map to Jira priority, with a value map. [src:1]
- **R21** Fields controlled by Jira itself (for example components, versions, sprint, and other fields Jira computes) are read-only in Wareslate and change only in Jira. Assigning a value still syncs both ways. Mapping a managed multi-select to a Wareslate multi-select overwrites existing Wareslate values; the UI warns before saving. A managed field cannot be used in a second Jira integration. [src:1]
- **R22** Components tie an integration to one project. The component mapping must be removed before the project is changed. [src:1]
- **R23** A user-picker mapping needs the authorizing Jira user to have permission to browse users. [src:1]

### Jira: detection, conflicts, deletes
- **R24** Preferred way to detect changes: Jira webhooks sent to Wareslate. Each webhook is checked by a secret in the URL path or header. [src:1 says webhooks are needed; details are a decision]
- **R25** Fallback when webhooks are not possible (self-hosted Jira behind a firewall): polling every 2 minutes with a "changed since" query, using one query per integration. [src:1 mentions a no-webhook mode; interval is a decision]
- **R26** Wareslate changes go to Jira within 10 seconds of the edit, through the job queue. A burst of edits to one field in a short time is merged into one write. [decision]
- **R27** Echo guard: a change that Wareslate wrote to Jira is not written back when Jira reports it. This is done by comparing the value with the last synced value stored on the link. [decision]
- **R28** Conflict rule for a `two_way` field changed on both sides before a sync: the change with the later timestamp wins, and the losing value is recorded in `sync_event` with outcome `overwritten`. [decision; sources are silent]
- **R29** For a `set_once` field, later changes on either side are never copied. [src:1]
- **R30** Never delete: deleting an item removes only the link; the Jira issue stays. Deleting a Jira issue marks the link `unlinked` and leaves the item as is. Archiving does not delete either side. [src:1 for the first sentence; the rest is a decision]
- **R31** A member can unlink at any time. Unlinking keeps both records and stops sync. [decision]
- **R32** Connection limits per plan are open (see Tier). The count is per Jira connection. [decision]

### Azure DevOps
- **R33** Same model as Jira: push, link, import. Per-field direction `two_way` or `set_once`. [src:4]
- **R34** Mapped: name, description, assignee, dates, parent/child links (for example epic to feature to story). Wareslate also adds a hyperlink and a comment on the work item that point back to the item. [src:4]
- **R35** Status comes back through rules, as in R12–R14. [src:4]
- **R36** Work items are never deleted by Wareslate. [src:4]
- **R37** Change detection is by service hooks, with polling as the fallback (R25). [decision]

### GitHub Issues, Trello, Shortcut
- **R38** GitHub: push creates an issue from an item. After that, name, description, labels and open/closed state sync in both directions. Existing issues cannot be linked or imported in v1. [src:5; two-way is unconfirmed in sources, so it is a decision]
- **R39** Trello: push creates a card from an item. The card status shows back on the item. Name and description do not sync. [src:7]
- **R40** Shortcut: push creates an epic from an item. Epic state flows back to the item status through rules. [src:6]
- **R41** None of these three deletes anything on the other side (R30). [decision]

### GitLab and Linear (Wareslate decision)
- **R42** GitLab and Linear are supported through the public API and webhooks, not through a hand-built native sync in v1. Both get push and status-back, using the same engine as R6–R31 with `set_once` and `two_way` fields. The engine is the one shipped for GitHub (R38), with a different adapter. [src:8; the choice is a decision because research show no native GitLab or Linear sync]
- **R43** Until the adapters ship, users can connect GitLab or Linear with the integration button (R50) or with Zapier plus the API. [decision]

### Inbound feedback
- **R44** Every inbound source creates **notes** (S04) and sets `source{system, record_id, url}` on each one. A record with the same `system` and `record_id` updates the note instead of creating a second one. [src:9][src:10]
- **R45** Slack and MS Teams: a message in a channel the admin picked becomes a note. Private channels work only if the admin adds the app. The note keeps a link to the message. Teams can also show the note's processed state in a tab and send notifications back. [src:9]
- **R46** Zendesk: a ticket becomes a note, by manual push, by tag, or for all tickets. Later replies update the note. When a processed note gets a new reply, it goes back to unprocessed. [src:10]
- **R47** Intercom: a conversation becomes a note, for all or by tag. User and company are matched by external ID or email. [src:11]
- **R48** Gong: a call transcript becomes a note linked to the customer of the call. [src:14]
- **R49** Email: a forwarded message to the workspace's address becomes a note. The forwarding member is the note owner. [src:15]
- **R49a** Browser extension: highlighted text on any page becomes a note with the page URL. Needs the contributor role or higher. [src:16]
- **R49b** Salesforce: an account becomes a company, with custom fields mapped. An open opportunity becomes a read-only note, refreshed about every 15 minutes. Lookup fields are not mapped. [src:12][src:13]
- **R49c** Zapier: templates create notes or item ideas from any app through the public API (S13). [src:8]
- **R49d** Analytics cohorts (Amplitude, Mixpanel) become user segments (S05). The cohort is re-read on a schedule and membership is replaced to match. Segments from a cohort cannot be edited by hand. [src:8; schedule and lock are decisions]
- **R49e** Design embeds (Figma, Miro, Mural): a share link pasted into a description shows as an embed. Only the link is stored. A sticky note from a whiteboard can become a note when the tool supports it. [src:8]
- **R49f** Matching of inbound people: first by external ID, then by email, then create a new user under the matched company. Never merge two users by name alone. [decision]
- **R49g** Inbound sources can be paused. A paused source keeps old notes and does not fetch new ones. [decision]

### Integration button (plugin model)
- **R50** An admin registers a plugin integration with a domain-style `type` (for example `com.example.tool`), a name, an initial label, and an action URL with an authorization header. The header is write-only. [src:17]
- **R51** On creation, Wareslate first sends a GET probe to the action URL with a validation token. The URL must answer correctly or registration fails. Only public https URLs are allowed. [src:17]
- **R52** Each item shows a button for every enabled plugin that applies. States: `initial`, `progress`, `connected`, `error`. [src:17]
- **R53** When a member clicks, Wareslate sends a POST to the action URL with the plugin id, the trigger (`button.push`, `button.unlink`, `button.dismiss`) and the item id with a link to it. The service answers with the new connection state. [src:17]
- **R54** A `connected` state carries a label, hover label, tooltip, color and target URL shown on the item. An `error` state carries a message and debug detail. [src:17]
- **R55** The service can also change a connection later through the API with the `plugin-integrations` scope (S13). [src:17]
- **R56** A plugin call has a 10 second timeout. A timeout sets `error` with a message. Retries are manual, by clicking again. [decision]

### Read-only Jira connection data
- **R57** The API exposes, for each Jira-linked item, the Jira issue key and id, read-only. The mapping itself is edited only in the app. [src:18]

## User actions
- Admin: connect and remove integrations, switch on two-way sync, register plugins, pick inbound channels and tags.
- Maker: push, link, import and unlink items; edit mappings where allowed (R1).
- Contributor: send notes with the browser extension and email; click an integration button on items they can edit.
- Viewer: see link state and external links, nothing else.

See [S01](S01-workspace-roles.md) for the role model.

## Acceptance tests
- **T1 (R6, R29)** Given name is `two_way` and description is `set_once`, when both are edited in Jira after linking, then the name changes in Wareslate and the description does not.
- **T2 (R12)** Given a rule "Done, Closed → Shipped" made today, when an issue that was already Done is linked, then the item status stays; when the issue later moves to Closed, the item becomes Shipped.
- **T3 (R19)** Given an item with effort 8 in rollup mode, when pushed, then story points are left empty. Given manual effort 8, then story points are 8.
- **T4 (R27)** Given Wareslate writes owner to Jira, when Jira sends the change back, then no second write is made.
- **T5 (R28)** Given a `two_way` field edited in Wareslate at 10:00:00 and in Jira at 10:00:05 before sync, then Jira's value wins and a `sync_event` shows `overwritten`.
- **T6 (R30)** Given a linked item, when it is deleted, then the Jira issue still exists.
- **T7 (R11)** Given a Jira number field, when a user maps it to a Wareslate text field, then the mapping is refused.
- **T8 (R18)** Given an owner whose email has no Jira user, when pushed, then reporter is empty and the push succeeds.
- **T9 (R44)** Given a Zendesk ticket was imported, when the same ticket is sent again, then one note exists, updated.
- **T10 (R46)** Given a processed note from Zendesk, when the customer replies, then the note is unprocessed.
- **T11 (R49b)** Given a Salesforce account with 3 custom fields mapped, when connected, then a company exists with those fields.
- **T12 (R51)** Given a plugin URL that does not answer the probe, then registration fails and nothing is stored.
- **T13 (R53, R54)** Given a plugin, when a member clicks and the service answers `connected` with a label, then the item shows that label and target URL.
- **T14 (R25)** Given no webhook can be reached, then changes appear within 2 minutes by polling.
- **T15 (R38)** Given a GitHub issue created by push, when it is closed on GitHub, then the item shows the closed state.
- **T16 (R3)** Given the authorizing member is removed, then the integration is `error` and no data is deleted.

## Tier
Jira and Azure DevOps: limited number of connections on lower paid plans, unlimited on top; self-hosted versions and Salesforce on the top tier. Chat, helpdesk, email, extension, analytics, API and plugin button: all paid tiers. Other issue trackers: all paid tiers. [decision; plan details in sources are unconfirmed]

## Open questions
1. What exactly counts toward the connection limit (R32): connections, projects, or linked items?
2. Should Wareslate write status to Jira (R13)? Needs a workflow-aware transition step.
3. Native GitLab and Linear adapters in v1, or API only (R42)?
4. Is last-write-wins (R28) enough, or do users need a "needs review" queue?
5. Webhook secret method for each tracker (path secret vs signed header).

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S12).
