# S10 Documents

## Scope
Rich-text documents with a block editor, real-time co-editing, comments, mentions, embeds, and turning text into items. The same editor is used for rich-text descriptions on items. It covers:
- Block types, slash menu, drag handles
- Co-editing with presence
- Comments with resolve
- @mentions and embeds
- Convert text into items
- Document location (personal or teamspace), sharing, who can create and edit

Related modules:
- Roles, teamspaces, sharing model: [S01](S01-workspace-roles.md)
- Items and hierarchy (convert targets, item descriptions): [S02](S02-hierarchy.md)
- Fields and dates (mention targets): [S03](S03-fields-statuses.md)
- Boards (mention targets): [S09](S09-boards.md)
- Customers (mention targets): [S05](S05-customers.md)
- AI assistant creating comments: [S15](S15-ai.md)
- Notifications for mentions: [S14](S14-security.md) for audit; delivery is a platform feature

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| document | id, workspace_id, title | | yes | |
| | location | enum `personal \| teamspace` | yes | |
| | teamspace_id | uuid | if teamspace | null for personal |
| | owner_id | uuid | yes | the creator; never changes on share |
| | content | CRDT state (binary) | yes | see R4 |
| | created_at, updated_at, deleted_at | timestamp | | soft delete |
| document_share | document_id, target_type, target_id, access | | | target: `user \| teamspace \| workspace`; access: `view \| comment \| edit` |
| block | id (stable), type, attrs, children | inside content | yes | see R1 |
| comment_thread | id, document_id, anchor, resolved, resolved_by, resolved_at | | yes | anchor = text range, block ids, or mention id |
| comment | id, thread_id, author_id, body, created_at, edited_at | | yes | body is rich text with mentions |
| mention | document_id, target_type, target_id | | | target: `user \| team \| board \| document \| item \| field \| date` (plus customer, segment, release as item-like targets) |
| embed | block id, provider, url | | | provider: `figma \| figjam \| miro \| mural \| loom \| youtube \| chart` |
| presence | document_id, user_id, caret, color | ephemeral | | never stored |
| item_description | item_id, content | same format as `content` | no | same editor and storage as a document |

## Rules

### Blocks and editing
- **R1** A document is a list of blocks. Block types: paragraph, headings (levels 1–3), bullet list, numbered list, to-do (checkbox), toggle, quote, horizontal rule, code block, table, image (caption, left/center/right alignment), embed, and 2-, 3- and 4-column layouts. [src:1] Heading levels limited to 1–3 is a [decision].
- **R2** Typing `/` at the start of an empty block opens the slash menu. It lists the block types in R1 and filters as you type. [src:1]
- **R3** Hovering a block shows a handle in the left margin. The handle drags the block (also into and out of columns) and offers: copy link to block, duplicate, change type, set color, comment, delete. Block colors come from a fixed palette of 12. [src:1]
- **R4** Content is stored as a CRDT document, not as plain HTML. Every block has a stable id so links and comments can point to it. Export to Markdown and HTML is produced from the CRDT. [decision] (Architecture note in docs/architecture.md says documents use a CRDT.)
- **R5** Selecting text shows a format bar: italic, bold, text color, link, change block type, comment, convert. [src:1] Bold is a [decision].
- **R6** Every document has an outline of its headings and a find-in-document search. Find also searches inside collapsed toggles. [src:1]
- **R7** Pasting a link offers "paste as link" or "paste as preview". Preview is available for Figma, FigJam, Miro, Mural, Loom and YouTube. Other links stay plain links. [src:1]
- **R8** A chart from analytics can be embedded as a live block. It shows current data, honors the viewer's data access, and is empty for viewers without access to its data. [src:1 says charts can be embedded; access behavior is a decision]
- **R9** Embeds are shown only from an allow-list of provider hosts. The embed never runs scripts of the host page. [decision]
- **R10** Autosave: no save button. Changes are kept every few seconds and on close. Version history keeps snapshots and allows restore by makers and admins. [decision]

### Co-editing and presence
- **R11** Several people can edit one document at once. Edits merge without conflict and without losing text. [src:1 (co-editing); merge guarantee comes from R4]
- **R12** Each editor shows a caret in their own color. Selections of others are not shown. [src:1]
- **R13** A small avatar group beside a row shows who is editing that row. [src:1]
- **R14** If a client is offline, it keeps editing locally and merges on reconnect. [decision]
- **R15** Users with only view or comment access can see presence but cannot change text. The server rejects edits from them. [decision]

### Comments
- **R16** A comment thread can be anchored to selected text, one block, several blocks, or a mention (item or date). [src:1]
- **R17** Replies are threaded. The author can edit or delete their own comment. [src:1]
- **R18** Anyone with comment access can resolve or reopen a thread. [src:1]
- **R19** Resolved threads are hidden by default. Each user has a personal setting per document: display (expanded, inline, hidden), filter (unresolved, resolved, all), sort. [src:1] Defaults are a [decision] except "unresolved" [src:1].
- **R20** If the anchor text is deleted, the thread stays and shows a note that the original content was removed. [src:1]
- **R21** Comments are not part of the CRDT text. They are stored separately and anchored by relative positions. [decision]
- **R22** The AI assistant can create comment threads for the user who asked. It changes resolve state only when asked. [src:1]

### Mentions
- **R23** Typing `@` opens a mention menu. Targets: members, teams, boards, documents, folders, items, fields, dates. Each result shows its parent. [src:1]
- **R24** Mentioning a person notifies them. Mentioning a team notifies all its members, in the app and by email. [src:1]
- **R25** A mention of an item shows as an inline chip. Click opens the item in a side panel. Boards show as links. Item attributes can show on hover. This hover is on by default and can be turned off per document. [src:1]
- **R26** Only items that belong to the document's teamspace can be mentioned. [src:1]
- **R27** A notification is sent only when a mention is first added, not each time the document is edited. A person who cannot open the document gets no notification, and the editor warns "this person cannot see this document" with a one-click share. [decision]
- **R28** Mentions are kept as links. If the target is deleted, the chip shows "deleted". [decision]

### Convert to items
- **R29** Selecting text and choosing Convert asks for an item type and, when needed, its parent. [src:1]
- **R30** For several lines: the first line is the name, the rest is the description. For a list: each list item becomes its own item, and sub-items become that item's description. [src:1]
- **R31** After convert, the text stays and links to the new item. The new item records the document as its source. Convert checks the user's right to create that item type (S01). [decision]

### Location, sharing, roles
- **R32** A document lives in a teamspace or in the creator's personal space. Personal documents are visible only to the owner until shared. [src:1]
- **R33** A document can be shared with a person, a teamspace, or the whole workspace, even from a personal, closed or private location. Sharing does not move it or change its owner. Recipients see it under "Shared with me". [src:1]
- **R34** Share access levels are view, comment, edit. Access in a teamspace comes from the teamspace role plus any shares; the highest wins. [decision]
- **R35** Admins and makers can create and delete documents. Contributors can create and delete documents in their personal space, and in a teamspace where they have "can edit content". Without it they can view and comment only. Viewers can view only. [src:1] Delete of others' documents by makers is a [decision].
- **R36** Linking a document to an item does not change who can see it. [src:1]
- **R37** Deleting a document is a soft delete for 30 days, then permanent. [decision]

### Item descriptions
- **R38** Item descriptions use the same editor and storage as documents, with the same blocks, mentions, embeds and co-editing. Comments on items use S02, not thread anchors. Block types that need a document (columns, chart embeds) may be turned off for descriptions. [decision]
- **R39** The API reads and writes rich text as HTML or Markdown, converted from the CRDT. Writes through the API replace the CRDT content as one change. [decision]

## User actions

| action | admin | maker | contributor | viewer |
|---|---|---|---|---|
| read documents they can access | yes | yes | yes | yes |
| create documents (teamspace) | yes | yes | only with "can edit content" | no |
| create documents (personal) | yes | yes | yes | no |
| edit text | yes | yes | where edit access | no |
| comment, resolve | yes | yes | yes | where comment access |
| share | yes | yes | owner only | no |
| convert to item | yes | yes | only item types they can create | no |
| delete | yes | yes | own documents | no |

See [S01](S01-workspace-roles.md) for the full role model. Viewer comment access is a [decision].

## Acceptance tests
- **T1 (R1, R2)** Given an empty block, when the user types `/3col`, then the menu offers a 3-column layout and inserting it gives three empty columns.
- **T2 (R3)** Given a paragraph, when the user drags its handle into column 2 of a layout, then the block moves there and keeps its id.
- **T3 (R11, R12)** Given users A and B edit the same paragraph at the same time, when both type, then both texts remain and each sees the other's caret.
- **T4 (R14)** Given A goes offline and types "x" while B types "y", when A reconnects, then both "x" and "y" are in the document.
- **T5 (R15, R35)** Given a viewer opens a document, when the client sends an edit, then the server rejects it and the text is unchanged.
- **T6 (R16, R18, R19)** Given a thread on a text range, when someone resolves it, then it disappears from the default view and shows with the "resolved" filter.
- **T7 (R20)** Given a thread anchored on text, when that text is deleted, then the thread stays with the note "original content was removed".
- **T8 (R24, R27)** Given a user types `@Team Alpha`, when the mention is added, then each member gets one notification. When the document is edited later, then no new notification.
- **T9 (R26)** Given an item from another teamspace, when the user types `@` and its name, then it is not in the list.
- **T10 (R30)** Given a bullet list with 3 items, the first with 1 sub-item, when the user converts them to features, then 3 features exist and the first has the sub-item text as description.
- **T11 (R33)** Given a personal document is shared with a teamspace, when a teamspace member opens "Shared with me", then it is there and its location is still personal.
- **T12 (R35)** Given a contributor without "can edit content" in teamspace T, when they try to create a document there, then it is refused. In their personal space it succeeds.
- **T13 (R7)** Given a pasted YouTube link, when the user picks "paste as preview", then an embed block is created. A link from an unknown host stays a link.
- **T14 (R38)** Given an item description is edited by two users at once, then both edits merge as in T3.

## Tier
Documents, comments, mentions: all tiers. Version history and chart embeds: paid tiers. [decision]

## Open questions
1. Does the CRDT library we pick give stable block ids and relative positions for comments (R4, R21)?
2. Should contributors without edit rights be able to suggest changes (suggestion mode)?
3. How long should version history be kept (R10)?
4. Do we need public document links (outside the workspace)? Not in v1.
5. Item descriptions as CRDT may make large items heavy. Do we compact the CRDT history?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S10).
