# S02 Product tree

## Scope
The tree of work items a product team plans: product > module > feature > slice. It covers:
- Item types and which parent each type may have
- Creating items (quick-create, CSV import), moving, converting, archiving, deleting
- Common item properties: owner, date range, description, watchers, links
- Tasks inside an item
- Dependencies (blocking / blocked by) and date-conflict detection
- Renaming item types

Related modules:
- Roles and teams: [S01](S01-workspace-roles.md)
- Custom fields, statuses, tags, teams field: [S03](S03-fields-statuses.md)
- Demand score and size that roll up this tree: [S06](S06-prioritization.md)
- Releases and bets (linked to items, not parents): [S08](S08-releases.md), [S07](S07-strategy.md)
- Views that show the tree: [S09](S09-views.md)
- API shape: [S13](S13-api.md)

## Entities

| entity | field | type | required | details |
|---|---|---|---|---|
| item | id | uuid | yes | |
| | type | enum `product \| module \| feature \| slice` | yes | fixed key; the display name can be renamed (R50) |
| | name | text, 1–255 chars | yes | |
| | description | rich text, up to 1 MiB | no | |
| | parent_id | uuid or null | see R2 | null only for product |
| | position | integer | yes | order among siblings |
| | owner_id | member id or null | no | a workspace member (S01) |
| | date range | `{start_date, end_date, precision}` or null | no | see R28–R30 |
| | archived | boolean | yes | default false |
| | archived_at | timestamp | when archived | |
| | created_at, updated_at | timestamp | yes | |
| | custom field values, status, tags, teams | | | defined in [S03](S03-fields-statuses.md) |
| watcher | item_id, member_id | | yes | one row per pair |
| item_link | item_id, target_type, target_id | | yes | link to a release, bet, objective or key result |
| task | id, item_id | | yes | |
| | title | text, 1–255 chars | yes | |
| | done | boolean | yes | default false |
| | assignee_id | member id or null | no | |
| | position | integer | yes | |
| dependency | id | uuid | yes | |
| | blocking_item_id | uuid | yes | the item that must finish first |
| | blocked_item_id | uuid | yes | the item that waits |
| item_type_label | type, display_name | | yes | per workspace, see R50 |
| import_job | id, file, status, rows_total, rows_ok, rows_failed, errors[] | | | see R33–R38 |

## Rules

### Types and parents
- **R1** The four item types are product, module, feature and slice, in that order from top to bottom. [src:1][src:3][src:6]
- **R2** Allowed parents: product has none (it is a root; a workspace may have many products). Module: a product or another module. Feature: a product or a module. Slice: a feature. Every item except a product must have a parent. [src:6]
- **R3** A slice cannot have children. [src:9]
- **R4** Modules can nest. The depth limit is 10 levels of modules under a product. [src:3]; the limit is a [decision]
- **R5** A move or create that breaks R2 or R4 is rejected with the error `tree.invalid_parent`. [decision]

### Creating
- **R6** Quick-create: select a product or module and choose "add feature". Each Enter keystroke saves the typed title and opens a blank row for another one. Clicking elsewhere ends the session. Each name creates one feature with no other fields set. [src:1]
- **R7** Quick-create works under any allowed parent (R2). It can also add a slice under a feature, and a module under a product or module. [decision; research only describes features]
- **R8** A new item goes to the end of its siblings. Names do not need to be unique. [decision]
- **R9** The creator is not made owner automatically. A new item has no owner until one is set. [decision]
- **R10** The creator automatically watches the item (R25). [decision]

### Moving and converting
- **R11** To move an item, set its parent (a picker shows up to 5 suggested parents, then the full tree). The same change is available on view cards that show the parent. [src:1]
- **R12** Moving an item moves its whole subtree. The demand score and size of the old and new ancestors are recomputed (S06 R10). [src:1; recompute from S06]
- **R13** An item cannot be moved under itself or under one of its own descendants. [decision]
- **R14** Many items can be moved at once by selecting them on a view and setting the parent field. All moved items must be allowed under the new parent (R2), or the whole bulk move is rejected. [src:1]; all-or-nothing is a [decision]
- **R15** Convert feature ↔ module ↔ slice from the item's "more actions" menu. Rules: a module with no child modules or features can become a feature. A feature can become a module or a slice. A feature that has slices, even archived ones, cannot become a slice. When a module becomes a feature, its child features become slices. [src:1][src:3]
- **R16** A conversion keeps id, name, description, owner, date range, watchers, links, tasks and dependencies. Fields and statuses that do not exist for the new type are dropped, and the user is told which ones before confirming. [decision]
- **R17** Changing an item's position among siblings (drag to reorder) changes only `position`. [decision]

### Archiving and deleting
- **R18** Modules, features and slices can be archived. A product cannot be archived. [src:2]
- **R19** Archiving an item archives all its descendants too. [src:2]
- **R20** Views leave out archived items by default. A view setting can switch to archived items only, or to active and archived together. [src:2]
- **R21** Unarchive is done on one item. It restores that item. Its children stay archived and can be unarchived one by one. Research does not say whether children come back, so this is a [decision]. An item cannot be unarchived while its parent is archived. [decision]
- **R22** Archived items are left out of the scores and size roll-ups of S06. They keep their links, evidence, dependencies and history. [decision]
- **R23** Delete removes an item for good. Only an admin or an editor can delete. If the item has descendants, the dialog shows how many items, tasks and evidence links will be removed, and asks for confirmation. Deleting removes the whole subtree, its tasks and its dependencies. Evidence (S04) stay in the feedback inbox but lose the link. [decision]
- **R24** A product with children cannot be deleted until an admin confirms the subtree count. Only admins can delete a product. [decision]

### Properties
- **R25** Watchers: any workspace member who can see the item can watch or unwatch it. Watchers get a notification when the item's status, owner or date range changes or a comment is added. An editor or admin can add or remove other watchers. [decision]
- **R26** Owner is one workspace member or empty. Disabled members cannot be set as owner. If an owner is disabled later, the owner is kept but marked inactive. [decision]
- **R27** Description is rich text. It is stored as sanitized HTML. Scripts and unsafe attributes are removed. [src:5]; sanitizing is a [decision]
- **R28** Date range has a start date, an end date and a precision: day, month, quarter or year. [src:5]
- **R29** The end date must not be before the start date. Dates are calendar dates without time. [decision]
- **R30** With a precision larger than day, the stored dates are snapped to the start of the first period and the end of the last period (for example, quarter precision with a start in February stores 1 January). [decision]
- **R31** Item links to releases, bets, objectives and key results are many-to-many. A link is not a parent-child relation. [src:6]
- **R32** Tasks belong to exactly one item. A task has a title, a done flag and an optional assignee. Tasks do not nest and are not items in the tree. A user who can edit the item can add, rename, reorder, complete and delete tasks. Deleting the item deletes its tasks. [decision; tasks are listed in research only as an entity]

### CSV import
- **R33** Import accepts a UTF-8 CSV file with a header row. Recognised columns: `type`, `name` (required), `parent` (a parent's name or id), `description`, `owner_email`, `status`, `start_date`, `end_date`, `precision`, `tags` (separated by `;`) and any custom field by its name. Unknown columns are ignored and listed in the result. [decision; research covers no columns]
- **R34** A file may have up to 5,000 rows and 5 MB. [decision]
- **R35** Before saving, the user sees a preview with the number of rows to create and a list of errors by row. A row fails if the name is empty, the parent is missing or ambiguous, the type or parent breaks R2, a date is invalid, or a value does not fit its field type (S03). [decision]
- **R36** Rows that pass are created. Rows that fail are skipped and downloadable as a CSV with an `error` column. [decision]
- **R37** A parent given by name must match exactly one item, or a parent created earlier in the same file. A row with no `parent` goes under a product named in the import dialog. [decision]
- **R38** Import never updates or deletes existing items. [decision]

### Dependencies
- **R39** A dependency links two items. Seen from one side it is "blocking", from the other "blocked by". Creating one on item A as "blocking B" also shows on B as "blocked by A". [src:4][src:6]
- **R40** Allowed types: feature and slice in any pair, and bet. [src:4]; research is unclear about bets, so allowing them is a [decision]
- **R41** A slice cannot depend on its own parent feature. In the same spirit, an item cannot depend on any of its ancestors or descendants. [src:4]; the wider rule is a [decision]
- **R42** An item cannot depend on itself. The same pair cannot be linked twice. A new dependency that would create a cycle (A blocks B blocks C blocks A) is rejected. [decision]
- **R43** Dependencies are created and removed from the item's detail panel, which has a picker with up to 5 suggested items. Removing one removes it from both sides. [src:4]; remove is a [decision]
- **R44** A dependency has a conflict when both items have a date range and the blocked item starts on or before the blocking item's end date. [decision]; a looser rule (blocked item starts on or before the blocking item's start) is the alternative, see open question 1
- **R45** For R44, a date range with a precision larger than day is read as its full range (R30). [decision]
- **R46** If either item has no date range, there is no conflict. Archived items are ignored for conflicts. [decision]
- **R47** A conflict is shown in the detail panel, on card link icons and in a filter on views. It is recomputed whenever a date range changes. It is never a hard error, and the user can save dates that create one. [src:4]
- **R48** Dependencies are not sent to Jira (S12). [src:4]
- **R49** Dependencies require a paid tier (see Tier). [src:4]

### Item types, display names
- **R50** An admin can rename the four item types, for example "feature" to "epic". Only the display name changes. API keys, URLs and exports keep the fixed type key. Names are plural and singular, 1–40 characters, unique among item types. [src:7]; limits are a [decision]
- **R51** Renaming applies everywhere in the workspace UI at once, including views, imports (R33 accepts both the key and the display name) and notifications. [decision]

## User actions
- Admin: everything below, plus delete products and rename item types.
- Editor: create, edit, move, convert, archive, unarchive and delete modules, features and slices. Manage dependencies, tasks, watchers, import CSV.
- Collaborator: view items, comment, watch or unwatch, and add feedback (S04). Cannot edit items. [src:8]
- Reader: read only.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R2, R5)** Given a slice S, when a user moves a feature under S, then the request is rejected with `tree.invalid_parent`.
- **T2 (R4)** Given a chain of 10 nested modules, when a user adds an 11th module under the last one, then it is rejected.
- **T3 (R6)** Given product P, when an editor quick-creates "A", "B", "C" in one session, then P gets three features with those names in that order and no other fields set.
- **T4 (R12)** Given feature F with direct impact 4 under module K1 (total 7), when F is moved to module K2, then K1's total is 3 and K2's total goes up by 4.
- **T5 (R13)** Given module K with child feature F, when a user moves K under F, then it is rejected.
- **T6 (R14)** Given a bulk move of two features and one module under a feature, then nothing moves and the error names the module.
- **T7 (R15)** Given feature F with an archived slice, when a user converts F to a slice, then it is refused. Given a module with no children, then it can become a feature.
- **T8 (R18)** Given a product, when a user attempts to archive it, then it is refused.
- **T9 (R19, R21)** Given module K with features F1 and F2, when K is archived, then F1 and F2 are archived. When K is unarchived, then K is active and F1 and F2 stay archived.
- **T10 (R22)** Given an archived feature with direct score 4 under module K, then K's total no longer includes the 4.
- **T11 (R23)** Given module K with 3 features and 5 evidence links, when an editor deletes K, then the dialog shows 3 items and 5 links, and after confirmation the items are gone and the feedback entries remain unlinked.
- **T12 (R30)** Given a date range with quarter precision and a start of 14 February 2027, then the stored start is 1 January 2027.
- **T13 (R33–R36)** Given a CSV with 100 rows where 3 have an unknown parent, when imported, then 97 items are created and a downloadable file lists the 3 failed rows with an error text.
- **T14 (R41)** Given slice S under feature F, when a user adds "S blocked by F", then it is rejected.
- **T15 (R42)** Given A blocks B and B blocks C, when a user adds "C blocks A", then it is rejected as a cycle.
- **T16 (R44, R46)** Given feature A (ends 30 June) blocks feature B (starts 15 June), then the dependency shows a conflict. When B is moved to start on 1 July, then the conflict goes away. Given B has no date range, then there is no conflict.
- **T17 (R50)** Given the display name of "feature" is changed to "epic", then views show "epic", and the API still returns type `feature`.
- **T18 (R39)** Given A is "blocking" B, then B's panel lists A under "blocked by".

## Tier
Product tree, tasks, watchers, CSV import: all tiers. Dependencies: paid tiers. [src:4]. Renaming item types: top tier only. [src:7]

## Open questions
1. Conflict rule (R44): keep the stricter "starts on or before the blocking item's end", or use the looser "starts on or before the blocking item's start"?
2. Should deleting be soft (trash with 30-day restore) instead of immediate?
3. Should unarchiving a parent offer to unarchive children too?
4. Should the module depth limit (R4) be removed?
5. Should dependencies also allow objectives and releases later?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S02).
