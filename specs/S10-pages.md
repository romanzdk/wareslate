# S10 Pages

## Scope
Rich-text pages with a block editor, live editing by several people, comments, mentions, embeds, and a way to turn text into items. Item descriptions use the same editor. This spec covers:
- Block types, the slash menu, block handles
- Live co-editing with presence
- Comment threads that can be resolved
- @mentions and embeds
- Turning text into items
- Where a page lives (My space or a space), sharing, who can create and edit

Related modules:
- Roles, spaces, sharing model: [S01](S01-workspace-roles.md)
- Items and the product tree (convert targets, item descriptions): [S02](S02-product-tree.md)
- Fields and dates (mention targets): [S03](S03-fields-statuses.md)
- Views (mention targets): [S09](S09-views.md)
- Accounts and contacts (mention targets): [S05](S05-customers.md)
- AI assistant that writes comments: [S15](S15-ai.md)
- Audit of mention notifications: [S14](S14-security.md); delivery itself is a platform feature

## Entities

| entity | field | type | required | details |
|---|---|---|---|---|
| page | id, workspace_id, title | | yes | |
| | location | enum `my_space \| space` | yes | |
| | space_id | uuid | if space | null for My space |
| | owner_id | uuid | yes | the creator; never changes on share |
| | content | CRDT state (binary) | yes | see R4 |
| | created_at, updated_at, deleted_at | timestamp | | soft delete |
| page_share | page_id, target_type, target_id, access | | | target: `user \| space \| workspace`; access: `read \| comment \| write` |
| block | id (stable), type, attrs, children | inside content | yes | see R1 |
| comment_thread | id, page_id, anchor, resolved, resolved_by, resolved_at | | yes | anchor = text range, block ids, or mention id |
| comment | id, thread_id, author_id, body, created_at, edited_at | | yes | body is rich text with mentions |
| mention | page_id, target_type, target_id | | | target: `user \| team \| view \| page \| item \| field \| date` (plus account, contact, cohort, release as item-like targets) |
| embed | block id, provider, url | | | provider: `youtube \| loom \| miro \| mural \| figjam \| figma \| chart` |
| presence | page_id, user_id, caret, color | ephemeral | | never stored |
| item_description | item_id, content | same format as `content` | no | same editor and storage as a page |

## Rules

### Blocks and editing
- **R1** A page is a list of blocks. Block types: paragraph, headings (levels 1–3), bullet list, numbered list, to-do (checkbox), toggle, quote, divider, code block, table, image (caption; left, center or right alignment), embed, and column layouts with 2, 3 or 4 columns. [src:1] Limiting headings to levels 1–3 is a [decision].
- **R2** Typing `/` in an empty block opens the slash menu. It offers the block types from R1 and narrows the list as the user types. [src:1]
- **R3** Pointing at a block shows a handle in the left margin. Dragging the handle moves the block, including into and out of columns. The handle's menu has: link to this block, clone, turn into another type, color, comment, delete. Block colors come from a fixed palette of 12. [src:1]
- **R4** Content is stored as a CRDT, not as plain HTML. Every block has a stable id, so links and comments can point at it. Markdown and HTML export are built from the CRDT. [decision] (docs/architecture.md says pages use a CRDT.)
- **R5** Selecting text shows a format bar: italic, bold, text color, link, turn into another block type, comment, convert. [src:1] Bold is a [decision].
- **R6** Every page has an outline made from its headings, and a search box to locate text in the page. Search also looks inside closed toggles. [src:1]
- **R7** Pasting a link offers two choices: keep it as a link, or show a preview. Preview works for YouTube, Loom, Miro, Mural, FigJam and Figma. Other links stay plain links. [src:1]
- **R8** A chart from analytics can be embedded as a live block. It shows current data and respects the reader's data access. A reader without access to the data sees an empty block. [src:1 says charts can be embedded; the access behavior is a decision]
- **R9** Embeds load only from an allow-list of provider hosts. An embed never runs scripts from the host page. [decision]
- **R10** Autosave: there is no save button. Changes are stored every few seconds and when the page closes. Version history keeps snapshots; editors and admins can restore one. [decision]

### Co-editing and presence
- **R11** Several people can edit one page at the same time. Edits merge without conflicts and without losing text. [src:1 (co-editing); the merge guarantee comes from R4]
- **R12** Each editor sees the others' carets, each in its own color. Text selections of others are not shown. [src:1]
- **R13** A small group of avatars next to a row shows who is editing that row. [src:1]
- **R14** A client that is offline keeps editing locally and merges when it reconnects. [decision]
- **R15** Users with only `read` or `comment` access can see presence but cannot change text. The server rejects their edits. [decision]

### Comments
- **R16** A comment thread can be anchored to selected text, one block, several blocks, or a mention (item or date). [src:1]
- **R17** Replies are threaded. Authors can edit or delete their own comments. [src:1]
- **R18** Anyone with `comment` access can resolve or reopen a thread. [src:1]
- **R19** Out of the box, a closed thread is out of sight. Each user has a personal setting per page: layout (expanded, inline, hidden), filter (unresolved, resolved, all) and sort order. [src:1] The defaults are a [decision], except "unresolved" [src:1].
- **R20** If the anchored text is deleted, the thread stays and shows a message that the original content was removed. [src:1]
- **R21** Comments are not part of the CRDT text. They are stored on their own and anchored by relative positions. [decision]
- **R22** The AI assistant can start comment threads for the user who asked. It resolves or reopens a thread only when asked. [src:1]

### Mentions
- **R23** Typing `@` opens the mention menu. Targets: members, teams, views, pages, folders, items, fields, dates. Each result shows where it lives. [src:1]
- **R24** Mentioning a person notifies that person. Mentioning a team alerts every person on it, both inside Wareslate and through email. [src:1]
- **R25** A mentioned item shows as an inline chip. Clicking it opens the item in a side panel. Views show as plain links. Item fields can appear when the pointer rests on a chip. This is on by default and can be switched off per page. [src:1]
- **R26** Only items from the page's own space can be mentioned. [src:1]
- **R27** A notification goes out only when a mention is first added, not on every later edit. If the mentioned person cannot open the page, no notification is sent, and the editor warns "this person cannot see this page" with a one-click share. [decision]
- **R28** Mentions are stored as links. If the target is deleted, the chip reads "deleted". [decision]

### Convert to items
- **R29** Selecting text and choosing Convert asks for an item type and, when needed, a parent. [src:1]
- **R30** Several lines: the first line becomes the name and the rest becomes the description. A list: every list entry becomes its own item, and an entry's sub-entries become that item's description. [src:1]
- **R31** After converting, the text stays on the page and links to the new item. The new item records the page as its origin. Convert checks that the user may create that item type (S01). [decision]

### Location, sharing, roles
- **R32** A page lives in a space or in its creator's My space. A page in My space is visible only to its owner until it is shared. [src:1]
- **R33** A page can be shared with one person, a space, or the whole workspace, including a page from My space or from a hidden or invite-only space. Sharing leaves the location and owner as they were. Recipients find the page under "Shared to me". [src:1]
- **R34** Share access levels are `read`, `comment` and `write`. Access in a space comes from the member's space access plus any shares; the highest level wins. [decision]
- **R35** Admins and editors can create and delete pages. Collaborators can create and delete pages in My space, and in a space where they have `write`. Without `write` they can read and comment only. Readers can only read. [src:1] Deleting other people's pages by editors is a [decision].
- **R36** Page visibility stays the same when the page gets attached to an item. [src:1]
- **R37** Deleting a page is a soft delete for 30 days; after that it is removed for good. [decision]

### Item descriptions
- **R38** Item descriptions use the same editor and storage as pages, with the same blocks, mentions, embeds and co-editing. Comments on items work as in S02, not as thread anchors. Block types that need a full page (columns, chart embeds) may be switched off in descriptions. [decision]
- **R39** The API reads and writes rich text as HTML or Markdown, converted from the CRDT. An API write replaces the CRDT content as a single change. [decision]

## User actions

| action | admin | editor | collaborator | reader |
|---|---|---|---|---|
| read pages they can access | yes | yes | yes | yes |
| create pages (space) | yes | yes | only with `write` | no |
| create pages (My space) | yes | yes | yes | no |
| edit text | yes | yes | where `write` access | no |
| comment, resolve | yes | yes | yes | where `comment` access |
| share | yes | yes | owner only | no |
| convert to item | yes | yes | only item types they can create | no |
| delete | yes | yes | own pages | no |

See [S01](S01-workspace-roles.md) for the full role model. Comment access for readers is a [decision].

## Acceptance tests
- **T1 (R1, R2)** Given an empty block, when the user types `/3col`, then the menu offers a 3-column layout and inserting it gives three empty columns.
- **T2 (R3)** Given a paragraph, when the user drags its handle into column 2 of a layout, then the block moves there and keeps its id.
- **T3 (R11, R12)** Given users A and B edit the same paragraph at the same time, when both type, then both texts remain and each sees the other's caret.
- **T4 (R14)** Given A goes offline and types "x" while B types "y", when A reconnects, then both "x" and "y" are on the page.
- **T5 (R15, R35)** Given a reader opens a page, when the client sends an edit, then the server rejects it and the text is unchanged.
- **T6 (R16, R18, R19)** Given a thread on a text range, when someone resolves it, then it disappears from the default view and shows with the "resolved" filter.
- **T7 (R20)** Given a thread anchored on text, when that text is deleted, then the thread stays with the message "original content was removed".
- **T8 (R24, R27)** Given a user types `@Team Alpha`, when the mention is added, then each member gets one notification. When the page is edited later, then no new notification is sent.
- **T9 (R26)** Given an item from another space, when the user types `@` and its name, then it is not in the list.
- **T10 (R30)** Given a bullet list with 3 entries, the first with 1 sub-entry, when the user converts them to features, then 3 features exist and the first has the sub-entry text as description.
- **T11 (R33)** Given a page in My space is shared with a space, when a member of that space opens "Shared to me", then the page is there and its location is still `my_space`.
- **T12 (R35)** Given a collaborator without `write` in space T, when they try to create a page there, then it is refused. In their My space it succeeds.
- **T13 (R7)** Given a pasted YouTube link, when the user picks the preview option, then an embed block is created. A link from an unknown host stays a link.
- **T14 (R38)** Given an item description is edited by two users at once, then both edits merge as in T3.

## Tier
Pages, comments, mentions: all tiers. Version history and chart embeds: paid tiers. [decision]

## Open questions
1. Does the CRDT library we pick give stable block ids and relative positions for comments (R4, R21)?
2. Should collaborators without `write` be able to suggest changes (suggestion mode)?
3. How long should version history be kept (R10)?
4. Do we need public page links (outside the workspace)? Not in v1.
5. Item descriptions as CRDT may make large items heavy. Do we compact the CRDT history?

## Sources
Provenance for `[src:N]` tags is kept in private research files (S10).
