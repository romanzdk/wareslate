# S04 Feedback and evidence

## Scope
How customer feedback gets into Wareslate, how it is triaged, and how parts of it are tied to items in the product tree as evidence. It covers:
- Feedback entries (message, thread, deal), channels, owner, triage state, tags
- Excerpts and evidence, re-linking, bulk link
- Automation rules, feedback forms, CSV import and export
- Suggested excerpts (the AI part is in S15)

Related modules:
- Need points and the demand score built from evidence: [S06](S06-prioritization.md) (this spec only stores the need level)
- Accounts and contacts: [S05](S05-customers.md)
- Product tree items that evidence links to: [S02](S02-product-tree.md)
- Tags and field definitions: [S03](S03-fields-statuses.md)
- Feedback views (saved filters over feedback): [S09](S09-views.md)
- Integration connectors (Zendesk, Slack…): [S12](S12-integrations.md)
- AI features: [S15](S15-ai.md). API and webhooks: [S13](S13-api.md). Roles: [S01](S01-workspace-roles.md)

## Entities

| entity | field | type | required | remarks |
|---|---|---|---|---|
| feedback | id, workspace_id | uuid | yes | |
| | kind | enum `message \| thread \| deal` | yes | fixed after create |
| | title | string | yes | falls back to first 80 characters of content |
| | content | rich text (message); list of parts (thread); key-value list (deal) | yes | see R3–R5 |
| | owner_id | member | no | a workspace member |
| | creator_id | member | no | null for entries from outside (form, email, integration) |
| | state | enum `new \| triaged` | yes | default `new` |
| | archived | boolean | yes | default false |
| | tags | list of tag | no | tags come from S03 |
| | customer | one contact or one account | no | S05; see R10 |
| | channel | enum, see R6 | yes | |
| | origin | `{tool, external_id, url}` | no | where the entry came from |
| | created_at, updated_at | timestamp | yes | |
| thread_part | external_id, author_name, author_type `customer \| agent`, content, timestamp | | | |
| deal_field | name, type `number \| string`, value | | | |
| attachment | id, feedback_id, filename, size, mime_type | | no | |
| excerpt | id, feedback_id, start, end | int offsets | yes | a marked range of a message or thread |
| evidence | id, feedback_id, target_id, excerpt_id | uuid | yes | target_id is a product tree item; excerpt_id is null for a whole-entry link |
| | need | enum from S06 or null | no | defined in S06 R1, not here |
| | need_set_at, need_set_by | | when set | S06 |
| automation_rule | id, name, enabled | | yes | |
| | match | `{tags[], topic}` | yes | at least one condition |
| | target_id | uuid | yes | product tree item |
| | need | enum or null | no | applied to created evidence |
| feedback_form | id, name, enabled, fields[], target_tag, default_owner_id | | | R25–R28 |
| email_inbox | workspace_id, address | | | R8 |

## Rules

### Feedback entries
- **R1** A feedback entry has one of three kinds: message, thread or deal. [src:1]
- **R2** An entry has a title, content, tags, an owner, a state (`new` or `triaged`), an archived flag, an optional customer and a channel. [src:1][src:4]
- **R3** A message holds rich text (headings, lists, links, bold, italic, images by link). Content is sanitized: scripts and unknown HTML are removed. [src:1][decision on sanitizing]
- **R4** A thread holds ordered parts. Each part has an author name, an author type (customer or agent), text and a time. Parts are shown oldest first. [src:1][src:4]
- **R5** A deal holds name/value fields (number or text), for example amount, close date, stage. It comes from a CRM integration. People cannot create or edit it by hand or by API in v1. They can still tag it, assign an owner, triage it, archive it and link it. [src:1][src:4; UI limit is a decision]
- **R6** Channels: `manual` (typed in the app), `form` (R25), `email` (R8), `extension` (browser extension, R9), `integration` (the origin tool is the connector name, S12), `import` (CSV, R30), `api`. [src:1][src:3]
- **R7** Origin data is `{tool, external_id, url}`. For entries from an integration, `(tool, external_id)` is unique in a workspace. A second push with the same pair updates the existing entry and does not add a new one. [src:1][src:4; update-instead-of-create is a decision]
- **R8** Email-in: each workspace has an inbox address. Mail sent or forwarded to it becomes a message. The subject is the title and the body is the content. The sender address is matched to a contact (S05 R27). The entry starts as `new`. Attachments are stored on the entry. [src:3]
- **R9** With the browser extension, a member selects text on any web page, adds a title, tags and a customer, and saves it as a message. The page URL is kept as the origin URL. [src:3]
- **R10** A feedback entry has at most one customer: one contact, or one account. If the contact belongs to an account, that account is shown as well, with no second link. The customer can be set by id or by email. An email that matches nobody creates a new contact (S05 R27). [src:1][src:4; auto-create is a decision]
- **R11** The content of a message cannot be edited while it has excerpts linked to product tree items. The edit is refused with an error that gives the reason. Once the links are removed, it can be edited. [src:4]
- **R12** Archiving hides an entry from default lists but keeps its evidence. Archived entries can be restored. Deleting an entry deletes its excerpts and evidence for good, after a confirmation that shows how many evidence links will go. [decision]
- **R13** Attachments: up to 10 per entry and 25 MB each. [decision]

### Triage
- **R14** New entries start as `new`. A member sets an entry to `triaged` after reviewing it, and can set it back to `new`. [src:1][src:2]
- **R15** An entry from outside (form, email, extension, integration) gets its owner from its channel (for forms, the default owner, R26). If the channel has none, there is no owner. [decision]
- **R16** The inbox view lists `new` entries, newest first. It can filter by channel, owner, tag, customer, account and date. A member can set owner, tags, customer and state for all selected entries at once. [src:2][decision on bulk]
- **R17** Triage flow: open a `new` entry, set the customer, mark the useful parts as excerpts, link each excerpt to product tree items, set the need level, set the state to `triaged`. A link is not needed to triage. [src:2][decision on "no link required"]
- **R18** A `triaged` entry with no links is allowed and stays searchable. [decision]
- **R19** Full-text search covers title, content, thread parts, tags and customer names. [src:4]

### Excerpts and evidence
- **R20** A member can mark a range of text in a message or thread as an excerpt and link it to one or more product tree items. Each link is one evidence record. A whole entry can also be linked, with no excerpt. [src:1][src:2]
- **R21** The triple (entry, tree item, excerpt) is unique. Linking the same excerpt to the same tree item again does nothing. [decision]
- **R22** An evidence record can carry a need level from S06 R1, or none. This spec only stores it. Points and their effect on scores are in S06. [src:2][src:6]
- **R23** Re-linking moves an evidence record to another tree item and keeps its excerpt and need. If the target already has evidence for the same excerpt, the two are merged and the higher need is kept. Scores of both items update (S06 R10). [src:1][decision on merge]
- **R24** Bulk link: select several feedback entries (or evidence records) and link all of them to one tree item in a single step. Each entry gets one whole-entry evidence record. The result says how many were linked and how many were skipped as duplicates (R21). An optional need level applies to all new records. Bulk unlink works the same way. [src:1][decision on details]
- **R24a** Evidence can link to tree items of any type that S02 allows as a feedback target (product, module, feature, slice). Archived items cannot get new links. [decision]
- **R24b** Every evidence record keeps a link back to its feedback entry. Open an entry to see its evidence. Open a tree item to see its evidence, with the customer for each. [src:1][src:4]

### Suggested excerpts
- **R24c** For a message, the AI assistant can suggest excerpts. They show as pending marks and turn into real excerpts only when a member accepts them. Rejected ones are dropped. A suggestion never creates evidence by itself. Details, limits and AI units: [S15](S15-ai.md). [src:5][decision on never auto-linking]

### Automation rules
- **R25a** An automation rule links new feedback to a tree item when the entry has a matching tag or topic. [src:2][src:7]
- **R25b** A rule matches when the entry has all of the rule's tags (AND), or when its topic equals the rule's topic. If both are given, both must match. A topic is a tag-like label set by a member or by AI grouping (S15). [decision on AND]
- **R25c** Rules run when an entry is created and when its tags or topic change. They create a whole-entry evidence record with the rule's need level (or none). They never remove links and never change an existing need. Running twice makes no duplicates (R21). [decision]
- **R25d** A rule can be run once over existing entries ("apply to existing"). It shows the number of matches first. [decision]
- **R25e** Rules are kept in order. Several rules can match one entry, and all of them apply. [decision]
- **R25f** A disabled rule does nothing. Deleting a rule keeps the evidence it created. [decision]

### Feedback forms
- **R25** A form is a public page that collects feedback. An editor or admin builds it from fields: title, description, name, email, account (optional), file attachment, extra text fields. The form gets a share link and can be embedded. [src:2][src:3]
- **R26** Each form has a default owner and a tag that is added to every submission. [decision]
- **R27** A submission creates a `new` message with channel `form`. The email field is matched to a contact (S05 R27). [src:2][decision]
- **R28** Forms have spam protection (a hidden field and a limit of 10 submissions per IP per hour). A disabled form shows a "closed" message at its link. [decision]

### CSV import and export
- **R29** Export: any filtered list of feedback (the inbox or a feedback view) exports to CSV. Columns: id, kind, title, content (plain text), owner email, state, archived, tags (semicolon list), customer email, account, origin tool, origin url, created at, linked items (names, semicolon list). [src:1][decision on columns]
- **R30** Import: a member uploads a CSV and maps columns to title, content, owner email, tags, customer email, account, created at, origin url, state. Title and content are required. Imported entries are messages with channel `import`. Up to 5,000 rows per file. [src:1][decision on limits]
- **R31** Import checks every row first and shows a report (rows ok, rows with errors and the reason). The member can import the valid rows or cancel. Importing a row again with the same origin url updates that entry (R7). [decision]
- **R32** Import from another feedback tool through its CSV export uses the same mapping step. [src:1]

## User actions

- Admin and Editor: everything in this spec, including automation rules, forms, email-in setup, import, export, delete.
- Collaborator: create feedback (manual, extension, form), edit entries they created, add tags, mark excerpts, link evidence and set the need level on them. Cannot bulk edit other people's entries, edit rules or forms, or delete entries they did not create.
- Reader: read only. Can see feedback they have access to.
- Anyone outside the workspace: can submit through a form, if an editor has published it.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R1, R4)** Given a thread with parts at 10:02, 10:00 and 10:01, when it is opened, then parts show in order 10:00, 10:01, 10:02.
- **T2 (R7)** Given an integration pushes record "A-1" twice with new content the second time, then there is one entry with the new content.
- **T3 (R8, R10)** Given an email from known contact u@x.com, when it arrives at the inbox, then a `new` message exists with u as customer. Given an unknown sender, then a new contact is created.
- **T4 (R11)** Given a message with a linked excerpt, when a member edits its content, then the edit is refused with a clear message.
- **T5 (R14, R17, R18)** Given a `new` entry with no links, when a member sets it to `triaged`, then it leaves the inbox and still shows in search.
- **T6 (R20, R21)** Given an excerpt already linked to feature F, when a member links it to F again, then no second evidence record exists.
- **T7 (R22)** Given an evidence record with need `high`, then the stored value is `high` and no points are stored in this module.
- **T8 (R23)** Given two evidence records on F and G for the same excerpt (`medium` and `high`), when the one on F is re-linked to G, then G has one record with need `high` and F has none.
- **T9 (R24)** Given 10 entries where 2 already link to F, when a member bulk links all 10 to F, then 8 are added, and the report says 8 linked, 2 skipped.
- **T10 (R24c)** Given an AI suggestion on an entry, when nobody accepts it, then no excerpt and no evidence exist.
- **T11 (R25b, R25c)** Given a rule "tags: api AND export → feature F", when an entry is created with tag api only, then nothing links. When tag export is added, then one evidence record on F is created. Adding export again changes nothing.
- **T12 (R25d)** Given a new rule that matches 40 existing entries, when an editor clicks "apply to existing", then the count 40 is shown first, and 40 evidence records exist after confirming.
- **T13 (R25, R27)** Given a published form with default owner O and tag "form-a", when someone submits it with an email, then a `new` entry with channel `form`, owner O and tag "form-a" is created.
- **T14 (R30, R31)** Given a CSV with 100 rows where 3 have no content, when imported, then the report shows 97 ok and 3 errors, and only valid rows are imported after confirming.
- **T15 (R29)** Given a filtered list of 12 entries, when exported, then the CSV has 12 rows and the listed columns.
- **T16 (R5)** Given a deal, when a member opens it, then they cannot edit its fields but can tag and link it.
- **T17 (R10)** Given an entry with a contact in account C, then it shows C as well, and the entry has only one customer link.

## Tier
Feedback entries, evidence, manual and import: all tiers. Forms, email-in, extension: all tiers. Automation rules and AI suggested excerpts: paid tiers. Integration connectors: see S12. [decision]

## Open questions
1. Should the need level be settable through the API (the public source is unclear)? Wareslate default: yes, as in R22.
2. Should an entry allow more than one customer (for example a call with two accounts)?
3. Where does "topic" come from if AI grouping is off? Wareslate default: members set it by hand.
4. Should watchers of an entry get notified about new evidence?
5. Should threads be editable (add a part) after creation?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S04).
