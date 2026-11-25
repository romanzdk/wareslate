# S02 Hierarchy

## Scope
The tree of work items a product team plans: product > component > feature > subfeature. It covers:
- Item types and which parent each type may have
- Creating items (quick-create, CSV import), moving, converting, archiving, deleting
- Common item properties: owner, timeframe, description, followers, links
- Tasks inside an item
- Dependencies (blocking / blocked by) and date-conflict detection
- Renaming item types

Related modules:
- Roles and teams: [S01](S01-workspace-roles.md)
- Custom fields, statuses, tags, teams field: [S03](S03-fields-statuses.md)
- Impact score and effort that roll up this tree: [S06](S06-prioritization.md)
- Releases and initiatives (linked to items, not parents): [S08](S08-releases.md), [S07](S07-strategy.md)
- Boards that show the tree: [S09](S09-boards.md)
- API shape: [S13](S13-api.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| item | id | uuid | yes | |
| | type | enum `product \| component \| feature \| subfeature` | yes | fixed key; the display name can be renamed (R50) |
| | name | text, 1–255 chars | yes | |
| | description | rich text, up to 1 MiB | no | |
| | parent_id | uuid or null | see R2 | null only for product |
| | position | integer | yes | order among siblings |
| | owner_id | member id or null | no | a workspace member (S01) |
| | timeframe | `{start_date, end_date, granularity}` or null | no | see R28–R30 |
| | archived | boolean | yes | default false |
| | archived_at | timestamp | when archived | |
| | created_at, updated_at | timestamp | yes | |
| | custom field values, status, tags, teams | | | defined in [S03](S03-fields-statuses.md) |
| follower | item_id, member_id | | yes | one row per pair |
| item_link | item_id, target_type, target_id | | yes | link to a release, initiative, objective or key result |
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
- **R1** The four item types are product, component, feature and subfeature, in that order from top to bottom. [src:1][src:3][src:6]
- **R2** Allowed parents: product has none (it is a root; a workspace may have many products). Component: a product or another component. Feature: a product or a component. Subfeature: a feature. Every item except a product must have a parent. [src:6]
- **R3** A subfeature cannot have children. [src:9]
- **R4** Components can nest. The depth limit is 10 levels of components under a product. [src:3]; the limit is a [decision]
- **R5** A move or create that breaks R2 or R4 is rejected with the error `hierarchy.invalid_parent`. [decision]

### Creating
- **R6** Quick-create: pick a product or component, choose "add feature", type a name and press Enter. A new row opens for the next name. Clicking away stops. Each name creates one feature with no other fields set. [src:1]
- **R7** Quick-create works under any allowed parent (R2). It can also add a subfeature under a feature, and a component under a product or component. [decision; research only describes features]
- **R8** A new item goes to the end of its siblings. Names do not need to be unique. [decision]
- **R9** The creator is not made owner automatically. A new item has no owner until one is set. [decision]
- **R10** The creator automatically follows the item (R25). [decision]

### Moving and converting
- **R11** To move an item, set its parent (a picker shows up to 5 suggested parents, then the full tree). The same change is available on board cards that show the parent. [src:1]
- **R12** Moving an item moves its whole subtree. The impact score and effort of the old and new ancestors are recomputed (S06 R10). [src:1; recompute from S06]
- **R13** An item cannot be moved under itself or under one of its own descendants. [decision]
- **R14** Many items can be moved at once by selecting them on a board and setting the parent field. All moved items must be allowed under the new parent (R2), or the whole bulk move is rejected. [src:1]; all-or-nothing is a [decision]
- **R15** Convert feature ↔ component ↔ subfeature from the item's "more actions" menu. Rules: a component with no child components or features can become a feature. A feature can become a component or a subfeature. A feature that has subfeatures, even archived ones, cannot become a subfeature. When a component becomes a feature, its child features become subfeatures. [src:1][src:3]
- **R16** A conversion keeps id, name, description, owner, timeframe, followers, links, tasks and dependencies. Fields and statuses that do not exist for the new type are dropped, and the user is told which ones before confirming. [decision]
- **R17** Changing an item's position among siblings (drag to reorder) changes only `position`. [decision]

### Archiving and deleting
- **R18** Components, features and subfeatures can be archived. A product cannot be archived. [src:2]
- **R19** Archiving an item archives all its descendants too. [src:2]
- **R20** Archived items are hidden from boards by default. A board setting can show archived items only, or active and archived together. [src:2]
- **R21** Unarchive is done on one item. It restores that item. Its children stay archived and can be unarchived one by one. Research does not say whether children come back, so this is a [decision]. An item cannot be unarchived while its parent is archived. [decision]
- **R22** Archived items are left out of the scores and effort roll-ups of S06. They keep their links, insights, dependencies and history. [decision]
- **R23** Delete removes an item for good. Only an admin or a maker can delete. If the item has descendants, the dialog shows how many items, tasks and insight links will be removed, and asks for confirmation. Deleting removes the whole subtree, its tasks and its dependencies. Insights (S04) stay in the feedback inbox but lose the link. [decision]
- **R24** A product with children cannot be deleted until an admin confirms the subtree count. Only admins can delete a product. [decision]

### Properties
- **R25** Followers: any workspace member who can see the item can follow or unfollow it. Followers get a notification when the item's status, owner or timeframe changes or a comment is added. A maker or admin can add or remove other followers. [decision]
- **R26** Owner is one workspace member or empty. Disabled members cannot be set as owner. If an owner is disabled later, the owner is kept but marked inactive. [decision]
- **R27** Description is rich text. It is stored as sanitized HTML. Scripts and unsafe attributes are removed. [src:5]; sanitizing is a [decision]
- **R28** Timeframe has a start date, an end date and a granularity: day, month, quarter or year. [src:5]
- **R29** The end date must not be before the start date. Dates are calendar dates without time. [decision]
- **R30** With a granularity larger than day, the stored dates are snapped to the start of the first period and the end of the last period (for example, quarter granularity with a start in February stores 1 January). [decision]
- **R31** Item links to releases, initiatives, objectives and key results are many-to-many. A link is not a parent-child relation. [src:6]
- **R32** Tasks belong to exactly one item. A task has a title, a done flag and an optional assignee. Tasks do not nest and are not items in the tree. A user who can edit the item can add, rename, reorder, complete and delete tasks. Deleting the item deletes its tasks. [decision; tasks are listed in research only as an entity]

### CSV import
- **R33** Import accepts a UTF-8 CSV file with a header row. Recognised columns: `type`, `name` (required), `parent` (a parent's name or id), `description`, `owner_email`, `status`, `start_date`, `end_date`, `granularity`, `tags` (separated by `;`) and any custom field by its name. Unknown columns are ignored and listed in the result. [decision; research covers no columns]
- **R34** A file may have up to 5,000 rows and 5 MB. [decision]
- **R35** Before saving, the user sees a preview with the number of rows to create and a list of errors by row. A row fails if the name is empty, the parent is missing or ambiguous, the type or parent breaks R2, a date is invalid, or a value does not fit its field type (S03). [decision]
- **R36** Rows that pass are created. Rows that fail are skipped and downloadable as a CSV with an `error` column. [decision]
- **R37** A parent given by name must match exactly one item, or a parent created earlier in the same file. A row with no `parent` goes under a product named in the import dialog. [decision]
- **R38** Import never updates or deletes existing items. [decision]

### Dependencies
- **R39** A dependency links two items. Seen from one side it is "blocking", from the other "blocked by". Creating one on item A as "blocking B" also shows on B as "blocked by A". [src:4][src:6]
- **R40** Allowed types: feature and subfeature in any pair, and initiative. [src:4]; research is unclear about initiatives, so allowing them is a [decision]
- **R41** A subfeature cannot depend on its own parent feature. In the same spirit, an item cannot depend on any of its ancestors or descendants. [src:4]; the wider rule is a [decision]
- **R42** An item cannot depend on itself. The same pair cannot be linked twice. A new dependency that would create a cycle (A blocks B blocks C blocks A) is rejected. [decision]
- **R43** Dependencies are created and removed from the item's detail panel, which has a picker with up to 5 suggested items. Removing one removes it from both sides. [src:4]; remove is a [decision]
- **R44** A dependency has a conflict when both items have a timeframe and the blocked item starts on or before the blocking item's end date. [decision]; a looser rule (blocked item starts on or before the blocking item's start) is the alternative, see open question 1
- **R45** For R44, a timeframe with a granularity larger than day is read as its full range (R30). [decision]
- **R46** If either item has no timeframe, there is no conflict. Archived items are ignored for conflicts. [decision]
- **R47** A conflict is shown in the detail panel, on card link icons and in a filter on boards. It is recomputed whenever a timeframe changes. It is never a hard error, and the user can save dates that create one. [src:4]
- **R48** Dependencies are not sent to Jira (S12). [src:4]
- **R49** Dependencies require a paid tier (see Tier). [src:4]

### Item types, display names
- **R50** An admin can rename the four item types, for example "feature" to "epic". Only the display name changes. API keys, URLs and exports keep the fixed type key. Names are plural and singular, 1–40 characters, unique among item types. [src:7]; limits are a [decision]
- **R51** Renaming applies everywhere in the workspace UI at once, including boards, imports (R33 accepts both the key and the display name) and notifications. [decision]

## User actions
- Admin: everything below, plus delete products and rename item types.
- Maker: create, edit, move, convert, archive, unarchive and delete components, features and subfeatures. Manage dependencies, tasks, followers, import CSV.
- Contributor: view items, comment, follow or unfollow, and add feedback (S04). Cannot edit items. [src:8]
- Viewer: read only.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R2, R5)** Given a subfeature S, when a user moves a feature under S, then the request is rejected with `hierarchy.invalid_parent`.
- **T2 (R4)** Given a chain of 10 nested components, when a user adds an 11th component under the last one, then it is rejected.
- **T3 (R6)** Given product P, when a maker quick-creates "A", "B", "C" in one session, then P gets three features with those names in that order and no other fields set.
- **T4 (R12)** Given feature F with direct impact 4 under component K1 (total 7), when F is moved to component K2, then K1's total is 3 and K2's total goes up by 4.
- **T5 (R13)** Given component K with child feature F, when a user moves K under F, then it is rejected.
- **T6 (R14)** Given a bulk move of two features and one component under a feature, then nothing moves and the error names the component.
- **T7 (R15)** Given feature F with an archived subfeature, when a user converts F to a subfeature, then it is refused. Given a component with no children, then it can become a feature.
- **T8 (R18)** Given a product, when a user tries to archive it, then it is refused.
- **T9 (R19, R21)** Given component K with features F1 and F2, when K is archived, then F1 and F2 are archived. When K is unarchived, then K is active and F1 and F2 stay archived.
- **T10 (R22)** Given an archived feature with direct score 4 under component K, then K's total no longer includes the 4.
- **T11 (R23)** Given component K with 3 features and 5 insight links, when a maker deletes K, then the dialog shows 3 items and 5 links, and after confirmation the items are gone and the notes remain unlinked.
- **T12 (R30)** Given a timeframe with quarter granularity and a start of 14 February 2027, then the stored start is 1 January 2027.
- **T13 (R33–R36)** Given a CSV with 100 rows where 3 have an unknown parent, when imported, then 97 items are created and a downloadable file lists the 3 failed rows with an error text.
- **T14 (R41)** Given subfeature S under feature F, when a user adds "S blocked by F", then it is rejected.
- **T15 (R42)** Given A blocks B and B blocks C, when a user adds "C blocks A", then it is rejected as a cycle.
- **T16 (R44, R46)** Given feature A (ends 30 June) blocks feature B (starts 15 June), then the dependency shows a conflict. When B is moved to start on 1 July, then the conflict goes away. Given B has no timeframe, then there is no conflict.
- **T17 (R50)** Given the display name of "feature" is changed to "epic", then boards show "epic", and the API still returns type `feature`.
- **T18 (R39)** Given A is "blocking" B, then B's panel lists A under "blocked by".

## Tier
Hierarchy, tasks, followers, CSV import: all tiers. Dependencies: paid tiers. [src:4]. Renaming item types: top tier only. [src:7]

## Open questions
1. Conflict rule (R44): keep the stricter "starts on or before the blocking item's end", or use the looser "starts on or before the blocking item's start"?
2. Should deleting be soft (trash with 30-day restore) instead of immediate?
3. Should unarchiving a parent offer to unarchive children too?
4. Should the component depth limit (R4) be removed?
5. Should dependencies also allow objectives and releases later?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S02).
