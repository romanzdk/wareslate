# S04 Feedback and insights

## Scope
How customer feedback gets into Wareslate, how it is triaged, and how parts of it are linked to hierarchy items as insights. It covers:
- Feedback items (text, conversation, opportunity), sources, owner, processed state, tags
- Highlights and insights, re-linking, bulk link
- Automation rules, feedback forms, CSV import and export
- Suggested highlights (the AI part is in S15)

Related modules:
- Importance points and the impact score built from insights: [S06](S06-prioritization.md) (this spec only stores the importance value)
- Companies and customer users: [S05](S05-customers.md)
- Hierarchy items that insights link to: [S02](S02-hierarchy.md)
- Tags and field definitions: [S03](S03-fields-statuses.md)
- Insights boards (saved filters over feedback): [S09](S09-boards.md)
- Integration connectors (Zendesk, Slack…): [S12](S12-integrations.md)
- AI features: [S15](S15-ai.md). API and webhooks: [S13](S13-api.md). Roles: [S01](S01-workspace-roles.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| feedback_item | id, workspace_id | uuid | yes | |
| | type | enum `text \| conversation \| opportunity` | yes | fixed after create |
| | title | string | yes | falls back to first 80 characters of content |
| | content | rich text (text); list of parts (conversation); key-value list (opportunity) | yes | see R3–R5 |
| | owner_id | member | no | a workspace member |
| | creator_id | member | no | null for items from outside (form, email, integration) |
| | processed | boolean | yes | default false |
| | archived | boolean | yes | default false |
| | tags | list of tag | no | tags come from S03 |
| | customer | one customer user or one company | no | S05; see R10 |
| | source | `{origin, record_id, url}` | yes | origin is one of R6 |
| | created_at, updated_at | timestamp | yes | |
| conversation_part | external_id, author_name, author_type `customer \| agent`, content, timestamp | | | |
| opportunity_field | name, type `number \| string`, value | | | |
| attachment | id, item_id, filename, size, mime_type | | no | |
| highlight | id, item_id, start, end | int offsets | yes | a marked range of a text or conversation item |
| insight | id, item_id, hierarchy_item_id, highlight_id | uuid | yes | highlight_id is null for a whole-item link |
| | importance | enum from S06 or null | no | defined in S06 R1, not here |
| | importance_set_at, importance_source | | when set | S06 |
| automation_rule | id, name, enabled | | yes | |
| | match | `{tags[], topic}` | yes | at least one condition |
| | target_item_id | uuid | yes | hierarchy item |
| | importance | enum or null | no | applied to created insights |
| feedback_form | id, name, enabled, fields[], target_tag, default_owner_id | | | R25–R28 |
| email_inbox | workspace_id, address | | | R8 |

## Rules

### Feedback items
- **R1** Three item types: text, conversation, opportunity. [src:1]
- **R2** An item has a title, content, tags, an owner, a processed state, an archived state, an optional customer and a source. [src:1][src:4]
- **R3** A text item holds rich text (headings, lists, links, bold, italic, images by link). Content is sanitized: scripts and unknown HTML are removed. [src:1][decision on sanitizing]
- **R4** A conversation item holds ordered parts. Each part has an author name, an author type (customer or agent), text and a time. Parts are shown in time order. [src:1][src:4]
- **R5** An opportunity item holds name/value fields (number or text), for example amount, close date, stage. It comes from a CRM integration. People cannot create or edit it by hand or by API in v1. They can still tag, own, process, archive and link it. [src:1][src:4; UI limit is a decision]
- **R6** Sources of feedback: `manual` (typed in the app), `form` (R25), `email` (R8), `extension` (browser extension, R9), `integration` (name of the connector, S12), `import` (CSV, R30), `api`. [src:1][src:3]
- **R7** Source data is `{origin, record_id, url}`. For items from an integration, `(origin, record_id)` is unique in a workspace. A second push with the same pair updates the existing item. It does not create a new one. [src:1][src:4; update-instead-of-create is a decision]
- **R8** Email-in: each workspace has an inbox address. Mail sent or forwarded to it becomes a text item. The subject is the title. The body is the content. The sender address is matched to a customer user (S05 R27). The item is unprocessed. Attachments are stored on the item. [src:3]
- **R9** The browser extension lets a member select text on any page, add a title, tag and customer, and save it as a text item. The page URL is kept as the source URL. [src:3]
- **R10** A feedback item has at most one customer: one customer user, or one company. If it has a user that belongs to a company, the company is shown too, without a second link. It can be set by id or by email. Setting an email that matches nobody creates a new customer user (S05 R27). [src:1][src:4; auto-create is a decision]
- **R11** Content of a text item cannot be edited while it has highlights linked to hierarchy items. The change is refused with an error that says why. Removing the links first makes it editable. [src:4]
- **R12** Archiving hides an item from default lists but keeps its insights. Archived items can be restored. Deleting an item deletes its highlights and insights for good, after a confirmation that shows how many insights go away. [decision]
- **R13** Attachments: up to 10 per item and 25 MB each. [decision]

### Processing and triage
- **R14** New items start unprocessed. A member marks an item processed when it has been reviewed. They can mark it unprocessed again. [src:1][src:2]
- **R15** An item from outside (form, email, extension, integration) gets the owner set by its channel (form default owner, R26). If none is set, the owner is empty. [decision]
- **R16** The inbox view lists unprocessed items, newest first. It can filter by source, owner, tag, customer, company and date. A member can set owner, tags, customer and processed in bulk for the selected items. [src:2][decision on bulk]
- **R17** Triage flow: open an unprocessed item → set customer → highlight the useful parts → link each highlight to hierarchy items → set importance → mark processed. Marking processed does not need a link. [src:2][decision on "no link required"]
- **R18** A processed item with no links is allowed and stays searchable. [decision]
- **R19** Full-text search covers title, content, conversation parts, tags and customer names. [src:4]

### Highlights and insights
- **R20** A member can highlight a range of text in a text or conversation item and link the highlight to one or more hierarchy items. Each link is an insight. An item can also be linked as a whole, without a highlight. [src:1][src:2]
- **R21** The pair (item, hierarchy item, highlight) is unique. Linking the same highlight to the same hierarchy item twice does nothing. [decision]
- **R22** An insight can carry an importance value from S06 R1, or none. This spec only stores it. Points and the effect on scores are in S06. [src:2][src:6]
- **R23** Re-linking moves an insight to another hierarchy item and keeps its highlight and importance. Moving it onto an item that already has the same highlight merges the two, and keeps the stronger importance. Scores of both items update (S06 R10). [src:1][decision on merge]
- **R24** Bulk link: select several feedback items (or insights) and link all of them to one hierarchy item in one step. Each item gets one whole-item insight. The result tells how many were linked and how many were skipped as duplicates (R21). An optional importance applies to all new insights. Bulk unlink works the same way. [src:1][decision on details]
- **R24a** An insight links to hierarchy items of any type that S02 allows as a feedback target (product, component, feature, subfeature). Archived items cannot get new links. [decision]
- **R24b** Insights keep a link to their feedback item. Open an item to see its insights. Open a hierarchy item to see its insights with the customer for each. [src:1][src:4]

### Suggested highlights
- **R24c** For a text item, an AI assistant may suggest highlights. They show as pending marks and become real highlights only when a member accepts them. Rejected ones are dropped. A suggestion never creates an insight by itself. Details, limits and credits: [S15](S15-ai.md). [src:5][decision on never auto-linking]

### Automation rules
- **R25a** An automation rule links incoming feedback to a hierarchy item when the item has a matching tag or topic. [src:2][src:7]
- **R25b** A rule matches when the item has all of the rule's tags (AND), or when its topic equals the rule's topic. If both are given, both must match. Topic is a tag-like label set by a member or by AI grouping (S15). [decision on AND]
- **R25c** Rules run when an item is created and when tags or topic change. They create a whole-item insight with the rule's importance (or none). They never remove links and never change existing importance. Running twice makes no duplicates (R21). [decision]
- **R25d** A rule can be run once on existing items ("apply to existing"). It shows the number of matches first. [decision]
- **R25e** Rules are listed in order. Several rules can match one item. All of them apply. [decision]
- **R25f** A disabled rule does nothing. Deleting a rule keeps insights it created. [decision]

### Feedback forms
- **R25** A form is a public page that collects feedback. A maker or admin builds it from fields: title, description, name, email, company (optional), file attachment, extra text fields. The form gets a share link and can be embedded. [src:2][src:3]
- **R26** Each form has a default owner and a tag added to every submission. [decision]
- **R27** A submission creates an unprocessed text item with source `form`. The email field is matched to a customer user (S05 R27). [src:2][decision]
- **R28** Forms have spam protection (a hidden field and a rate limit of 10 submissions per IP per hour). Disabling a form makes its link show a "closed" message. [decision]

### CSV import and export
- **R29** Export: any filtered list of feedback (inbox or an insights board) exports to CSV. Columns: id, type, title, content (plain text), owner email, processed, archived, tags (semicolon list), customer email, company, source origin, source url, created at, linked items (names, semicolon list). [src:1][decision on columns]
- **R30** Import: a member uploads a CSV and maps columns to title, content, owner email, tags, customer email, company, created at, source url, processed. Title and content are required. Imported items are text items with source `import`. Up to 5,000 rows per file. [src:1][decision on limits]
- **R31** Import checks all rows first and shows a report (rows ok, rows with errors and why). The member can import the valid rows or cancel. Re-importing a row with the same source url updates that item (R7). [decision]
- **R32** Import from another feedback tool through its CSV export uses the same mapping step. [src:1]

## User actions

- Admin and maker: everything in this spec, including automation rules, forms, email-in setup, import, export, delete.
- Contributor: create feedback (manual, extension, form), edit items they created, add tags, highlight, link insights and set importance on them. Cannot run bulk edits on others' items, edit rules or forms, delete items they did not create.
- Viewer: read only. Can see feedback they have access to.
- Anyone outside the workspace: can submit through a form, if a maker has published it.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R1, R4)** Given a conversation item with parts at 10:02, 10:00 and 10:01, when it is opened, then parts show in order 10:00, 10:01, 10:02.
- **T2 (R7)** Given an integration pushes record "A-1" twice with new content the second time, then there is one item with the new content.
- **T3 (R8, R10)** Given an email from known user u@x.com, when it arrives at the inbox, then an unprocessed text item exists with u as customer. Given an unknown sender, then a new customer user is created.
- **T4 (R11)** Given a text item with a linked highlight, when a member edits its content, then the edit is refused with a clear message.
- **T5 (R14, R17, R18)** Given an unprocessed item with no links, when a member marks it processed, then it leaves the inbox and still shows in search.
- **T6 (R20, R21)** Given a highlight already linked to feature F, when a member links it to F again, then no second insight exists.
- **T7 (R22)** Given an insight with importance "critical", then the stored value is "critical" and no points are stored in this module.
- **T8 (R23)** Given two insights on F and G for the same highlight (important and critical), when the one on F is re-linked to G, then G has one insight with importance critical and F has none.
- **T9 (R24)** Given 10 items where 2 already link to F, when a member bulk links all 10 to F, then 8 are added, and the report says 8 linked, 2 skipped.
- **T10 (R24c)** Given an AI suggestion on an item, when nobody accepts it, then no highlight and no insight exist.
- **T11 (R25b, R25c)** Given a rule "tags: api AND export → feature F", when an item is created with tag api only, then nothing links. When tag export is added, then one insight to F is created. Adding export again changes nothing.
- **T12 (R25d)** Given a new rule that matches 40 existing items, when a maker clicks "apply to existing", then the count 40 is shown first, and 40 insights exist after confirming.
- **T13 (R25, R27)** Given a published form with default owner O and tag "form-a", when someone submits it with an email, then an unprocessed item with source `form`, owner O and tag "form-a" is created.
- **T14 (R30, R31)** Given a CSV with 100 rows where 3 have no content, when imported, then the report shows 97 ok and 3 errors, and only valid rows are imported after confirming.
- **T15 (R29)** Given a filtered list of 12 items, when exported, then the CSV has 12 rows and the listed columns.
- **T16 (R5)** Given an opportunity item, when a member opens it, then they cannot edit its fields but can tag and link it.
- **T17 (R10)** Given an item with a user in company C, then it shows C as well, and the item has only one customer link.

## Tier
Feedback items, insights, manual and import: all tiers. Forms, email-in, extension: all tiers. Automation rules and AI suggested highlights: paid tiers. Integration connectors follow S12. [decision]

## Open questions
1. Should importance be settable through the API (the public source is unclear)? Wareslate default: yes, as in R22.
2. Should an item allow more than one customer (for example a call with two companies)?
3. Where does "topic" come from if AI grouping is off? Wareslate default: members set it by hand.
4. Should followers of an item get notified on new insights?
5. Should conversation items be editable (add a part) after creation?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S04).
