# S12 Integrations

## Scope
How Wareslate connects to outside tools. Three families:
1. **Issue trackers (two-way):** Jira in depth, then Azure DevOps, GitHub Issues, Trello, Shortcut, GitLab, Linear.
2. **Inbound feedback and data:** Slack, MS Teams, Zendesk, Intercom, Gong, email, browser extension, Salesforce, Zapier, analytics cohorts, design embeds.
3. **External actions:** a generic model so any outside service can show its link state on an item.

Related modules:
- Items, parents, children: [S02](S02-product-tree.md)
- Fields, statuses: [S03](S03-fields-statuses.md)
- Feedback entries and evidence: [S04](S04-feedback.md)
- Accounts, contacts, cohorts: [S05](S05-customers.md)
- Size (maps to story points): [S06](S06-prioritization.md)
- Releases (map to fix versions): [S08](S08-releases.md)
- Public API, webhooks, OAuth used by integrations: [S13](S13-api.md)
- SSO and SCIM (not covered here): [S14](S14-security.md)

## Entities

| entity | field | type | required | remarks |
|---|---|---|---|---|
| integration | id, workspace_id, kind, name | | yes | one row per connection to an outside account or project |
| | status | enum `enabled \| disabled \| error` | yes | |
| | config | jsonb | yes | project, mapping, rules; secrets are stored encrypted and the API never returns them |
| | authorized_by | member id | yes | the member whose credentials the connection uses |
| integration_link | integration_id, item_id | | yes | one item to one external record |
| | external_id, external_key, external_url | text | yes | |
| | sync_state | enum `synced \| pending \| error \| unlinked` | yes | |
| | last_synced_at, last_error | | no | |
| field_mapping | integration_id, local_field, external_field | | yes | |
| | direction | enum `sync \| seed` | yes | default `seed` (R6) |
| | value_map | jsonb | no | for status, select and priority values |
| status_rule | integration_id, external_statuses[], local_status | | yes | external → local status (R12) |
| sync_event | link_id, field, side `local \| external`, old, new, at, outcome | | yes | audit log kept 30 days |
| inbound_source | id, kind, config, filter | | yes | a connected chat, helpdesk, call or CRM source |
| external_action | id, type (dotted domain-style text), name, status, initial_state, endpoint{url, auth_header} | | yes | R50–R56; the auth header is write-only |
| external_action_link | action_id, item_id | | yes | |
| | state | enum `idle \| working \| linked \| failed` | yes | |
| | label, hover_label, tooltip, color, target_url | | when `linked` | |
| | message, debug_detail | | when `failed` | |

## Rules

### Common to all integrations
- **R1** An integration belongs to one workspace. Only an admin can connect one. An Editor can use a connected integration, and can edit its mapping if an admin allows it. [src:1][decision on the Editor part]
- **R2** Only an admin can switch on `sync` direction for an integration. Switching it on changes nothing until a mapped field is edited. [src:1]
- **R3** Credentials belong to the member who authorized the connection. If that member is removed or the token is revoked, the integration moves to `error` and stops syncing. It never deletes data. [decision]
- **R4** Every sync action is written to `sync_event`. The item shows the last result and the last error text. [decision]
- **R5** Failures retry with growing waits (1 min, 5 min, 30 min, 2 h, 12 h). After that the link goes to `error` and an admin is told. [decision]

### Jira: send, link, import
- **R6** Each mapped field has a direction: `sync` (kept equal at all times) or `seed` (copied only when sending, linking or importing). A new mapping starts as `seed`. [src:1; default is a decision]
- **R7** Send: a chosen item becomes a new Jira issue in the selected project, using the mapping. The link is stored. [src:1]
- **R8** Link: an item is tied to an existing Jira issue, found by key or by search. The mapped `seed` and `sync` fields are copied once at link time. The user picks the direction of this first copy (item to issue, or issue to item). [src:1; the choice is a decision]
- **R9** Import: chosen Jira issues become new items of a chosen item type, each already linked. Items without a link get no later updates, so they need a one-time import or a send. [src:1]
- **R10** Field mapping works with one Jira project at a time. Jira fields marked required must be mapped before a send can create an issue. [src:1]
- **R11** A mapping is allowed only when both fields have the same kind. A Jira custom field pairs only with a Wareslate custom field of the matching kind (text, number, one-of list, many-of list, date, member, url, label list). [src:1]

### Jira: field mapping
- **R12** Status: rules pair one or more Jira statuses with one Wareslate status. A rule acts only on changes made after it is switched on, not on values that already exist. Linking an existing issue leaves the item status alone until the issue's status changes. A Jira status with no rule is ignored. [src:2]
- **R13** Status flows from Jira to Wareslate, for sent and linked items and for bets. A status change in Wareslate is not written to Jira. [src:2; scope wording is from the research inventory; no write-back is a decision]
- **R14** A rule that points at a deleted Jira status or a deleted Wareslate status is shown as broken and stays off until fixed. A rule that is not complete cannot be saved. [src:2]
- **R15** Name maps to summary. Description maps to the Jira description. Formatting is kept only if the Jira field takes rich text; otherwise plain text is sent. [src:1]
- **R16** Date range start and end map to the Jira start date and due date. [src:1]
- **R17** Release maps to fix version, matched by name. If the version is missing, it is created in the Jira project on send only when the admin allowed this in the mapping. Otherwise the field stays empty and a warning is logged. [src:1; the create option is a decision]
- **R18** Owner maps to reporter, only when a Jira user with the same email exists. Otherwise the field stays empty. [src:1]
- **R19** Size maps to story points. A size goes to Jira only when the item's size is in `manual` mode (S06). A value coming from Jira always arrives as a manual size, even if Jira shows a summed number. [src:1] (see also S06 R26)
- **R20** Tags map to Jira labels. Team maps to a Jira multi-select or group-picker field. A select field can map to Jira priority through a value map. [src:1]
- **R21** Fields that Jira controls itself (for example components, versions, sprint, and anything Jira calculates) are read-only in Wareslate and change only in Jira. Assigning a value still syncs both ways. Mapping such a Jira multi-select to a Wareslate multi-select overwrites the values already in Wareslate, and the UI warns before saving. One of these fields cannot be used in a second Jira integration. [src:1]
- **R22** Jira components tie an integration to one project. Remove the component mapping before changing the project. [src:1]
- **R23** A member-picker mapping needs the authorizing Jira user to have the permission to browse users. [src:1]

### Jira: detection, conflicts, deletes
- **R24** Preferred way to detect changes: Jira webhooks sent to Wareslate. Each webhook is checked with a secret in the URL path or in a header. [src:1 says webhooks are needed; details are a decision]
- **R25** Fallback when webhooks cannot reach Wareslate (self-hosted Jira behind a firewall): poll every 2 minutes with a "changed since" query, one query per integration. [src:1 mentions a no-webhook mode; interval is a decision]
- **R26** Wareslate changes reach Jira within 10 seconds of the edit, through the job queue. Several quick edits to one field are merged into one write. [decision]
- **R27** Echo guard: a change that Wareslate wrote to Jira is not written back when Jira reports it. The check compares the value with the last synced value kept on the link. [decision]
- **R28** Conflict rule for a `sync` field changed on both sides before a sync runs: the later timestamp wins, and the losing value is saved in `sync_event` with outcome `overwritten`. [decision; sources are silent]
- **R29** For a `seed` field, later changes on either side are never copied. [src:1]
- **R30** Never delete: deleting an item removes only the link, and the Jira issue stays. Deleting a Jira issue marks the link `unlinked` and leaves the item as it is. Archiving touches neither side. [src:1 for the first sentence; the rest is a decision]
- **R31** A member can unlink at any time. Unlinking keeps both records and stops sync. [decision]
- **R32** Connection limits per plan are open (see Tier). The count is per Jira connection. [decision]

### Azure DevOps
- **R33** Same model as Jira: send, link, import, with a direction per field (`sync` or `seed`). [src:4]
- **R34** Mapped: name, description, assignee, dates, parent/child links (for example epic to feature to story). Wareslate also adds a hyperlink and a comment on the work item that point back to the item. [src:4]
- **R35** Status comes back through rules, as in R12–R14. [src:4]
- **R36** Wareslate never deletes work items. [src:4]
- **R37** Changes are detected through service hooks, with polling as the fallback (R25). [decision]

### GitHub Issues, Trello, Shortcut
- **R38** GitHub: send creates an issue from an item. After that, name, description, labels and the open/closed state sync both ways. Existing issues cannot be linked or imported in v1. [src:5; two-way is unconfirmed in sources, so it is a decision]
- **R39** Trello: send creates a card from an item. The card status shows on the item. Name and description do not sync. [src:7]
- **R40** Shortcut: send creates an epic from an item. The epic state flows back to the item status through rules. [src:6]
- **R41** None of these three deletes anything on the other side (R30). [decision]

### GitLab and Linear (Wareslate decision)
- **R42** In v1, GitLab and Linear work through the public API and webhooks, without a hand-built native sync. Both get send and status-back, using the same engine as R6–R31 with `seed` and `sync` fields. It is the engine built for GitHub (R38), with a different adapter. [src:8; the choice is a decision because research shows no native GitLab or Linear sync]
- **R43** Until the adapters ship, users can connect GitLab or Linear with an external action (R50) or with Zapier plus the API. [decision]

### Inbound feedback
- **R44** Every inbound source creates **feedback entries** (S04) and sets `origin{tool, external_id, url}` on each one. A record with the same `tool` and `external_id` updates the existing entry instead of making a second one. [src:9][src:10]
- **R45** Slack and MS Teams: a message in a channel the admin picked becomes a feedback entry. Private channels work only after the admin adds the app. The entry keeps a link to the message. Teams can also show the entry's triage state in a tab and send notifications back. [src:9]
- **R46** Zendesk: a ticket becomes a feedback entry by manual send, by tag, or for all tickets. Later replies update the entry. When a triaged entry gets a new reply, it goes back to `new`. [src:10]
- **R47** Intercom: a conversation becomes a feedback entry, for all conversations or by tag. The contact and account are matched by external ID or email. [src:11]
- **R48** Gong: a call transcript becomes a feedback entry linked to the customer on the call. [src:14]
- **R49** Email: a message forwarded to the workspace's address becomes a feedback entry. The member who forwarded it is the entry owner. [src:15]
- **R49a** Browser extension: text selected on any page becomes a feedback entry with the page URL. Needs the Collaborator role or higher. [src:16]
- **R49b** Salesforce: a Salesforce account becomes an account in Wareslate, with custom fields mapped. An open opportunity becomes a read-only `deal` entry, refreshed about every 15 minutes. Lookup fields are not mapped. [src:12][src:13]
- **R49c** Zapier: templates create feedback entries or idea items from any app through the public API (S13). [src:8]
- **R49d** Analytics cohorts (Amplitude, Mixpanel) become Wareslate cohorts (S05). The cohort is re-read on a schedule and its members are replaced to match. A cohort made this way cannot be edited by hand. [src:8; schedule and lock are decisions]
- **R49e** Design embeds (Figma, Miro, Mural): a share link pasted into a description shows as an embed. Only the link is stored. A sticky note from a whiteboard can become a feedback entry when the tool allows it. [src:8]
- **R49f** Matching inbound people: first by external ID, then by email, then create a new contact under the matched account. Never merge two contacts by name alone. [decision]
- **R49g** An inbound source can be paused. A paused source keeps its old entries and fetches no new ones. [decision]

### External actions
- **R50** An admin registers an external action with a domain-style `type` (for example `com.example.tool`), a name, an initial label, and an endpoint URL with an authorization header. The header is write-only. [src:17]
- **R51** When one is registered, Wareslate first sends a GET probe with a validation token to the endpoint URL. The URL must answer correctly, or registration fails. Only public https URLs are allowed. [src:17]
- **R52** Each item shows a button for every enabled external action that applies to it. States: `idle`, `working`, `linked`, `failed`. [src:17]
- **R53** When a member clicks the button, Wareslate sends a POST to the endpoint URL. The body holds the action id, the trigger (`link`, `unlink` or `hide`), and the item id with a link to it. The service answers with the new state. [src:17]
- **R54** A `linked` state carries a label, hover label, tooltip, color and target URL, all shown on the item. A `failed` state carries a message and debug detail. [src:17]
- **R55** Later, the service can change a link through the API with the `external_actions.write` scope (S13). [src:17]
- **R56** A call to the service times out after 10 seconds. A timeout sets `failed` with a message. Retrying is manual: the member clicks again. [decision]

### Read-only Jira link data
- **R57** For each Jira-linked item, the API returns the Jira issue key and id, read-only. The mapping itself can be edited only in the app. [src:18]

## User actions
- Admin: connect and remove integrations, switch on `sync` direction, register external actions, pick inbound channels and tags.
- Editor: send, link, import and unlink items; edit mappings where allowed (R1).
- Collaborator: send feedback with the browser extension and email; click an external action button on items they can edit.
- Reader: see link state and external links, nothing else.

See [S01](S01-workspace-roles.md) for the role model.

## Acceptance tests
- **T1 (R6, R29)** Given name is `sync` and description is `seed`, when both are edited in Jira after linking, then the name changes in Wareslate and the description does not.
- **T2 (R12)** Given a rule "Done, Closed → Shipped" made today, when an issue that was already Done is linked, then the item status stays; when the issue later moves to Closed, the item becomes Shipped.
- **T3 (R19)** Given an item with size 8 in `from_children` mode, when sent, then story points are empty. Given a manual size of 8, then story points are 8.
- **T4 (R27)** Given Wareslate writes owner to Jira, when Jira sends the change back, then no second write is made.
- **T5 (R28)** Given a `sync` field edited in Wareslate at 10:00:00 and in Jira at 10:00:05 before the sync runs, then Jira's value wins and a `sync_event` shows `overwritten`.
- **T6 (R30)** Given a linked item, when it is deleted, then the Jira issue still exists.
- **T7 (R11)** Given a Jira number field, when a user maps it to a Wareslate text field, then the mapping is refused.
- **T8 (R18)** Given an owner whose email has no Jira user, when sent, then reporter is empty and the send succeeds.
- **T9 (R44)** Given a Zendesk ticket was imported, when the same ticket arrives again, then one feedback entry exists, updated.
- **T10 (R46)** Given a triaged entry from Zendesk, when the customer replies, then the entry is `new` again.
- **T11 (R49b)** Given a Salesforce account with 3 custom fields mapped, when connected, then a Wareslate account exists with those fields.
- **T12 (R51)** Given an endpoint URL that does not answer the probe, then registration fails and nothing is stored.
- **T13 (R53, R54)** Given an external action, when a member clicks and the service answers `linked` with a label, then the item shows that label and target URL.
- **T14 (R25)** Given no webhook can be reached, then changes appear within 2 minutes by polling.
- **T15 (R38)** Given a GitHub issue created by send, when it is closed on GitHub, then the item shows the closed state.
- **T16 (R3)** Given the authorizing member is removed, then the integration is `error` and no data is deleted.

## Tier
Jira and Azure DevOps: a limited number of connections on lower paid plans, unlimited on the top plan; self-hosted versions and Salesforce on the top plan. Chat, helpdesk, email, extension, analytics, API and external actions: all paid plans. Other issue trackers: all paid plans. [decision; plan details in sources are unconfirmed]

## Open questions
1. What exactly counts toward the connection limit (R32): connections, projects, or linked items?
2. Should Wareslate write status to Jira (R13)? It would need a step that knows Jira workflows.
3. Native GitLab and Linear adapters in v1, or API only (R42)?
4. Is last-write-wins (R28) enough, or do users need a "needs review" queue?
5. Webhook secret method for each tracker (path secret or signed header).

## Sources
Provenance for `[src:N]` tags is kept in private research material (S12).
