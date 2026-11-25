# S09 Views

## Scope
A view is a saved way to look at workspace data. It keeps only its setup (kind, item types, columns, filters, sort, grouping, display options) and never holds a copy of the data. This module covers:
- Table view
- Timeline view
- Kanban view
- Feedback view (a saved filter over feedback) and the account and contact views
- Rules every view shares: stored config, filter engine, behavior of newly created items, private views, sharing, links, seen-by list, stars, duplicate, move, global search

Related modules:
- Roles and spaces: [S01](S01-workspace-roles.md)
- Product tree and item types: [S02](S02-product-tree.md)
- Fields, statuses, check-ins: [S03](S03-fields-statuses.md)
- Feedback and evidence: [S04](S04-feedback.md)
- Accounts, contacts, cohorts: [S05](S05-customers.md)
- Demand score, criteria, computed fields, size columns: [S06](S06-prioritization.md)
- Objectives, bets, key results: [S07](S07-strategy.md)
- Releases and date ranges: [S08](S08-releases.md)
- Public roadmap: [S11](S11-public-roadmap.md)
- Semantic search, AI assistant: [S15](S15-ai.md)

## Entities

| entity | field | type | required | remarks |
|---|---|---|---|---|
| view | id, workspace_id | uuid | yes | |
| | kind | enum `table \| timeline \| kanban \| feedback_view \| account \| contact` | yes | fixed after creation |
| | name | text | yes | |
| | space | enum `my_space \| space \| workspace` plus `space_id` | yes | a new view starts in `my_space` (R59) |
| | owner_id | member | yes | the creator |
| | config | JSON | yes | see "View config" below (R1) |
| | saved_config_version | int | yes | goes up on each Save |
| | access_overrides | list of {principal, level} | no | per-view changes to the space access (R63) |
| | link_mode | enum `off \| private_link \| public_link` | yes | default `off` (R66) |
| | link_token | random string | when link on | |
| | created_at, updated_at | timestamp | yes | |
| view_star | view_id, member_id | | | one row per star |
| view_visit | view_id, member_id, last_visited_at | | | feeds the seen-by list (R69) |
| milestone | id, workspace_id, name, description, date_range (date or range), color | | yes | workspace-wide, not owned by a view |
| view_milestone | view_id, milestone_id | | | the timeline views that show it (R31) |

### View config (stored as data)
| key | used by | meaning |
|---|---|---|
| `items` | table, timeline, kanban | ordered list of item types. The first is the root type, the rest are nested types |
| `columns` | table | ordered list of `{field, width, visible, sort, filter}` |
| `merged_columns` | table | which of release / objective / bet share one column |
| `grouping` | table, timeline, kanban | list of 0–2 field ids (timeline: lanes) |
| `sort` | table | map `item_type_level → {column, direction}` |
| `filters` | all | list of filter rows (R8) |
| `hide_childless` | table, timeline | boolean |
| `precision`, `zoom`, `rounding` | timeline | see R34–R38 |
| `card_fields`, `color_by` | timeline, kanban | see R41, R42 |
| `column_source` | kanban | field choice and hidden values (R45) |
| `milestones` | timeline | milestone ids (R31) |

## Rules

### Common to all views
- **R1** A view is one stored config record. Saving writes the config and nothing else. Items, feedback, accounts and contacts are never copied into it. [src:4][src:12]
- **R2** Any change to the config of an open view marks it "unsaved". An unsaved view offers Save (overwrite), Save as copy (a new view in the same space, named with " copy" added unless the user types a name), and Revert (back to the last saved config). Leaving without saving drops the changes. [src:4]
- **R3** Edits to data (status, owner, text, links) appear at once in every view that shows the item. No save is needed. [src:4]
- **R4** Saving changes the view for everyone who can open it. Only members with manage or owner access (R62) can Save. A reader of the view can still change filters and sort for themselves; those changes live in their session only and cannot be saved (the revert rules of R2 apply). [src:4; session-only edits for read-only users are a decision]
- **R5** Filter engine: the filter rows of a view are joined with AND. An item or feedback entry shows only if it matches every row. [src:4][src:12]
- **R6** One filter engine serves all views. A filter can be set in the view header, on one column, or in the view controls. All three write to the same `filters` list, so a column filter is the same as a view filter on that field. [src:1][decision on "same list"]
- **R7** Operators per field type: [decision, sources only say "filter per view or per column"]

  | field type | operators |
  |---|---|
  | single select, status, check-in, owner, member, team, item type | is any of, is none of, is empty, is not empty |
  | multi select, tags, linked items (release, objective, bet, parent) | has any of, has all of, has none of, is empty, is not empty |
  | text, title, body text | contains, does not contain, is empty, is not empty. Double quotes around words match that exact phrase |
  | number, size, criterion, demand score, computed | =, ≠, >, ≥, <, ≤, between, is empty, is not empty |
  | date, date range | is, before, after, between, in the last N days, in the next N days, is empty, is not empty |
  | boolean | is true, is false |

- **R8** A filter row is `{field, operator, values[]}`. While the user edits, a row with no values (except is empty / is not empty) is ignored and is not saved. [decision]
- **R9** Inside one row, values are OR-ed for "is any of" and "has any of", and AND-ed for "has all of". Between rows it is always AND. [src:5 AND rule from sources; within-row meaning is a decision]
- **R10** "Only mine" is a shortcut on member and owner columns. It adds the row "is any of [current member]". "Only my teams" does the same on team columns, using the teams of the current member. [src:1]
- **R11** "Only mine" stores the member id, not the word "mine". When a saved view is opened by another member, it still shows the first member's filter. A row can be saved as "current reader" instead, and then it resolves for each person who opens it. [decision]
- **R12** A filter on a field covers root items and nested items like this: a root item shows when it matches itself or when at least one nested item matches. Nested items that do not match are hidden. Root items that do not match stay visible as context. [decision, sources silent]
- **R13** Creating an item: when a user creates a root item in a view where a filter has exactly one value (one status, one owner, one team, one tag), the new item gets that value. A message lists the values that were filled in. [src:1][src:2][src:3]
- **R14** If the active filters have several values, or are on a field that cannot be filled in automatically (size, date, number), the new item may not match. It is still created. An alert says it is hidden by the filters and offers two choices: show it (clear the rows that hide it) or keep the filters. [src:1][src:2][src:3; two buttons are a decision]
- **R15** A search hit inside a view jumps to the item's spot in that view and opens any collapsed parents on the way. Double quotes search for an exact phrase. [src:2]
- **R16** Pickers that link an item (release, objective, bet, parent) show a "Quick picks" list of up to 5 items before the user types. It favors items the user linked before, items opened recently, and items the user owns. It leaves out the item itself, items already linked, archived items and items the user cannot see. Typing hides the list and searches everything. [src:1][src:2]
- **R17** Inline create: table and timeline views have a "create" line at the bottom (or on empty space) for a new root item. Hovering a root item shows "add item" for a nested one. [src:1][src:2][src:3]
- **R18** View kinds by role: readers cannot open table views or feedback views. They can open shared timeline and kanban views read-only, and public roadmaps. [src:11][decision on read-only timeline/kanban; sources say readers see shared roadmaps]

### Table view
- **R19** When creating a table view, the user picks one root item type (objective, bet, product, module, feature, release, or another type the workspace has). More types can be added below it as nested types. The first type in `items` is the root type. Which types are offered depends on the plan. [src:1]
- **R20** Nested rows sit under their root item. A nested type must be allowed under its parent type by the product tree rules in [S02](S02-product-tree.md). [src:1][decision on S02 link]
- **R21** Columns:
  - A side panel lists the default fields and all other field types, with search. A switch shows or hides each column.
  - Drag a column header to reorder.
  - Drag the divider between headers to resize. The minimum width is 80 px.
  - Column kinds: item fields, custom fields, parent, public roadmap, tasks, check-in, date range shown as a small bar chart, end date, demand score, account and cohort columns, criteria, computed fields, size ([S06](S06-prioritization.md)). [src:1][src:12]
  - Column order and widths belong to the view config only. [src:1][decision on "only"]
- **R22** Merged columns: release, objective and bet links can share one column that lists every linked value, switched on per link kind. Only these three kinds can be merged. [src:1]
- **R23** Inline edit: a click on a cell edits it where it is. Link cells open the picker (R16). To remove a link, hover the value in the cell and press "Remove". Read-only columns (calculated values, roll-ups) open no editor. A member can edit only items they have write access to ([S01](S01-workspace-roles.md)). [src:1][decision on rights and read-only]
- **R24** Grouping: a table can group by 0, 1 or 2 attributes. A second attribute makes subgroups inside each first-level group. Items with no value for a grouping attribute go to a "No value" group, placed last. Grouping applies to the root item level. [src:1][decision on root level]
- **R25** Dragging inside a group changes the manual order. Dragging to another group changes the item's field value: into a first-level group it sets the first attribute; into a subgroup it sets both (the first from the group, the second from the subgroup). Dropping into "No value" clears the attribute (both attributes when dropped into a subgroup of it). [src:1][decision on clearing]
- **R26** For multi-value attributes (release, tags), dropping into a group swaps the value of the group the item came from for the value of the target group and leaves other values as they are. [decision]
- **R27** If one of the two updates in R25 fails (for example a required field, or missing write access), neither is applied. The item snaps back and an error message shows. [decision]
- **R28** Sort: the column menu sets ascending or descending. Some columns cannot sort (for example the bar chart). Sort is stored per nesting level: root items use the root-level sort and each nested type has its own. Each level has one sort at a time; a new one replaces the old one. Empty values come last in both directions. Sort works inside groups. [src:1][decision on one-per-level, empty-last]
- **R29** Hide childless: this switch removes root items that have no nested items of the shown nested types. Children are counted after filters. [src:2][decision on exact meaning]
- **R30** CSV export: writes the table as currently filtered, sorted and grouped, with the visible columns in column order. Nested rows come below their parent with a level column. Multi-value cells are joined with "; ". Dates use ISO 8601. A cell that starts with `= + - @` gets a `'` in front, so spreadsheet apps do not run it as a command. A table with more than 50,000 rows exports in the background and the user gets a download message. [src:12 says CSV export exists][decision for all details]

### Timeline view
- **R31** Milestones are workspace-wide records (name, description, date range, color). A milestone created on a timeline is also assigned to that view. One milestone can be assigned to many views. Assigning needs a Save. Only admins and editors can create, edit, assign, remove and delete milestones. Deleting removes it from every view and asks for confirmation. A milestone shows as a colored diamond with a label. Milestones close together are grouped, and a click on the group lists them. The milestone menu splits them into past and future around the today line. [src:5]
- **R32** Nesting: `items` holds up to 4 item types, one per level. The first is the root type. Rows collapse and expand. Hide childless works as in R29. [src:2]
- **R33** Lanes: grouping places rows into horizontal lanes by an attribute. A second grouping makes nested lanes inside the first. At most 2 levels, with the same "No value" rule as R24. Custom fields can be used. [src:2][decision on max 2 matching the table]
- **R34** Precision: week, month or quarter. It sets the unit of the time axis. [src:2]
- **R35** Zoom sets how much time is visible within the chosen precision. It never changes item dates. Zoom is saved in the view config. [src:2][decision on saving]
- **R36** An item appears on the timeline only if it has a date range. Items without one are listed in an "undated" tray. Dragging one onto the axis gives it a date range. [src:2 first part][decision on undated tray]
- **R37** Date rounding is visual only. It stretches each card to whole units of the precision and leaves stored dates alone. Modes:
  - `span` (default): the card covers every unit it touches (12 Sep–20 Oct on a month axis covers Sep and Oct).
  - `end_only`: the card covers just the last unit it touches (Oct).
  - `inherit` (nested types only): the card takes the rounding of its parent and hides its own date range.
  [src:6]
- **R38** Mode limits by level: the root type uses `span` or `end_only`, never `inherit`. If the root type uses `end_only`, every lower type uses `inherit`. If the root type uses `span`, level 2 may use any mode, and levels 3 and 4 use `inherit` only. A child linked straight to a level-1 item (skipping level 2) uses the level-2 mode. While rounding is on, dragging cards to change dates is turned off. [src:6]
- **R39** Drag and resize: dragging a card moves its date range, and dragging an edge moves the start or the end. The new dates are written to the item's date range field ([S08](S08-releases.md) for releases). While dragging, dates snap to whole days; with rounding on, to the start or end of the precision unit. The write happens on drop. If it fails, the card goes back. A member without write access cannot drag. Dependencies and parent limits are not enforced in v1. [src:2][decision on snapping, rights, no enforcement]
- **R40** Vertical-only drag: holding Shift while dragging locks the move to up and down. It changes row order or lane and never dates. A drop into another lane changes the grouping attribute as in R25. [src:2][decision on lane drop]
- **R41** Card fields: a menu on the view picks which fields show on cards, for every item type. Default: name only. The parent name can be added. [src:2][src:12][decision on default]
- **R42** Color coding: cards of the root type can be colored by check-in (green, amber, red), a single-select custom field, a multi-select custom field, status, or task completion. Only timeline and kanban views have it. Items with no value are neutral grey. An item with several values in a multi-select field takes the first one in field order. The view shows a legend. [src:7][decision on neutral color, first value, legend]
- **R43** Check-in on cards: when check-in is chosen as a card field, the card shows the latest check-in value and its date ([S03](S03-fields-statuses.md)). [src:12][decision on details]
- **R44** PNG export: offered on timeline and kanban views. The user chooses the period to cover and what to show, then downloads an image. The view need not be saved first: unsaved config changes are used for the image and can be reverted afterwards. [src:9]

### Kanban view
- **R45** A kanban view has one set of columns. The set comes from a field chosen in the view controls: release, objective, bet, product, module, feature, or a single-select or multi-select custom field. It can be changed at any time. [src:3][src:12]
- **R46** Each column value can be hidden or shown (eye switch). An empty column can be added so items can be placed in it. [src:3]
- **R47** Lanes work as in R33. Local and global order is described in R50. [src:3][decision on the order]
- **R48** Card fields and color coding work as in R41 and R42. [src:3][src:7]
- **R49** A column can be moved by dragging it in the view controls, by dragging its header on the view, or from the column menu. [src:3]
- **R50** When the columns are items (release, objective, bet, product, module, feature), moving one changes the order of those items for the whole workspace, so every view that uses them shows the new order ("global order"). For custom-field columns the order is the field's option order, also global. Hiding a column affects only the current view. Order inside a lane is local to the view. [src:3][decision on custom field and lane order]
- **R51** Dropping a card into another column changes the item's value for the column field (as in R25 and R26). Sources do not describe this, so it copies the table rule. Dropping into a hidden-column area is not possible. [decision]
- **R52** Items can be added from an empty area ("click to add item") for a root item, and from the column menu "Add" for new or existing nested items if the view shows them. R13 and R14 apply. [src:3]

### Feedback view and customer views
- **R53** A feedback view is a saved filter over feedback ([S04](S04-feedback.md)). It lists the feedback entries that match all its filters (R5). It stores only the filter list. [src:4]
- **R54** Each workspace has a default feedback view with preset filters. Only admins can change it; others can duplicate it. [src:4][decision on admin-only edits]
- **R55** Filter fields for feedback: state (new/triaged), owner, tags, account, contact, source, created date range, linked item, need, text. Operators come from R7. The filter list matches the feedback list filters in S04. [src:4][src:12][decision on the exact list]
- **R56** Exporting the feedback list to CSV uses R30 (what is shown, in order). [src:12][decision]
- **R57** Customer views: an account view and a contact view list those entities ([S05](S05-customers.md)) as rows, with columns and filters on the same engine. Account view columns: name, cohort, custom account fields, account totals (for example total revenue of linked accounts), contact count and linked feedback count. Contact view columns: name, email, account and custom contact fields. Inline edit and CSV export work as in the table view. No nesting, and grouping goes 1 level deep at most. [src:12][decision on columns, 1 level]
- **R58** Collaborators can open and use the default and existing views of these kinds. Only editors and admins may set them up or change them. Readers cannot open feedback views. [src:4][src:11]

### Private views, sharing, links
- **R59** A new view starts in the creator's own space (`my_space`) and is private. Only the owner (and workspace admins, see R60) can see it. Creating it from home inside a space or folder puts it there. [src:4]
- **R60** Admins can list all views to manage them, and can move or delete them. They cannot read the content of a private view unless the owner shares it. [decision, favoring privacy; revisit with S14]
- **R61** View access levels: read, write, manage, owner. A view in a space starts with the access the member has in that space. [src:12][decision on mapping, see S01]
- **R62** Rights over the view config (Save, rename, delete, move) need manage or owner access. Write access lets the member change item data inside the view but not the config. Read access is look-only. [src:12][decision on meaning]
- **R63** A per-view override can raise or lower access for a person or space compared with the space level. The effective level is the higher of the space level and the override for that principal, unless the override says "none", which removes access. [src:12][decision on resolution]
- **R64** Sharing: a view can be shared with named people, with a space, or with the whole workspace. Sharing a view from `my_space` does not move it. The view list has an "I shared" tab and a "Shared to me" tab. [src:12]
- **R65** Moving a view into a space makes it visible to everyone who can get into that space. A view in `my_space` stays private until it is moved or shared. [src:4]
- **R66** Links: a view has link mode `off`, `private_link` or `public_link`. A private link needs a signed-in workspace member who has access. A public link gives anyone with the URL a read-only page of the published config, with no sign-in; it exists for timeline and kanban views only. The URL carries a random token of at least 128 bits. Turning the link off or regenerating the token ends the old URL at once. [src:10][decision on private link meaning, token strength]
- **R67** An admin can switch off public view links for the whole workspace. Existing public links then stop working and show as "off". They work again if the switch goes back on, unless the token was regenerated. [src:10][decision on restore]
- **R68** A public link shows the saved config of the view, never a visitor's own changes, hides internal-only fields (those not marked public in S03) and cannot be edited. Public timeline and kanban pages are the same thing as public roadmaps in [S11](S11-public-roadmap.md). [src:10][decision on hidden fields]
- **R69** Seen-by list: the view header lists who opened the view within the past 30 days, latest first, with a mark on people looking at it right now. Every workspace member can open this list, whatever their role. Anonymous public-link visits are shown as a single count, not by name. [src:8][decision on anonymous count]
- **R70** Stars: a member stars a view to put it in their own starred list. Unstar removes it. Stars are per member and never shown to others. Any member who can open the view can star it. [src:4; research is inconsistent about who may star, so "any member who can open it" is a decision]
- **R71** Duplicate: makes a copy of the config in the same space, owned by the person who duplicated. If the original was in `my_space`, the copy is private to that person; otherwise it goes into the original's space. Shares, overrides, links, stars and visit history are not copied. Milestone assignments are copied (the milestones themselves stay shared). The member needs read access to the original. [src:4][decision on what is copied]
- **R72** Move to a space: changes `space`. The owner stays. Access overrides are cleared. A member share is kept only if that member can access the target space. Links stay as they are. It needs manage or owner access on the view and membership in the target space. [src:4][decision on shares and overrides]
- **R73** Delete: only the owner, members with manage access and admins. Deleting a view does not delete items, feedback or milestones. [decision]

### Global search
- **R74** Global search finds views by name and by owner, next to other objects. Results respect view access (R61). A private view never appears in another member's results. [src:12][decision]
- **R75** Search mixes keyword and semantic matching as defined in [S15](S15-ai.md). This module only adds the access check on results. [src:12]

## User actions

| action | admin | editor | collaborator | reader |
|---|---|---|---|---|
| Create table, timeline, kanban, feedback views | yes | yes | no | no |
| Create account / contact views | yes | yes | no | no |
| Open table and feedback views | yes | yes | yes | no |
| Open shared timeline / kanban views, public roadmaps | yes | yes | yes | yes (read only) |
| Change filter, sort, zoom (session only) | yes | yes | yes | yes |
| Save view config | with manage access | with manage access | with manage access | no |
| Edit item data inline, drag cards to change dates | yes | yes | only items they may edit | no |
| Create / edit / assign milestones | yes | yes | no | no |
| Share, set links, move, delete a view | view owner or manager, and admin | view owner or manager | no | no |
| Star a view, see the seen-by list | yes | yes | yes | yes |
| Turn public view links on or off for the workspace | yes | no | no | no |
| Export CSV / PNG | yes | yes | yes | PNG only if the view is shared with them |

Collaborator and reader details are [decision] where sources are silent. See [S01](S01-workspace-roles.md) for the role model.

## Acceptance tests
- **T1 (R1, R2)** Given a user changes a column filter and leaves the view without saving, when they come back, then the view shows the last saved config. When they change it and press Save as copy, then a second view exists and the first is unchanged.
- **T2 (R3)** Given feature F appears in two views, when its status changes in view A, then view B shows the new status with no save.
- **T3 (R5, R9)** Given filters status is any of [Planned, In progress] and owner is any of [Ana], when the table loads, then only items with one of those statuses and owner Ana show.
- **T4 (R10, R11)** Given a user presses "Only mine" on the owner column, when another member opens the saved view, then they see Ana's items (the original user), not their own, unless the row was saved as "current reader".
- **T5 (R13)** Given a status filter with one value "Planned", when a user creates a feature inline, then the feature has status Planned and a message says so.
- **T6 (R14)** Given a size filter "> 5", when a user creates a feature inline, then the feature exists, is hidden, and an alert offers to clear the row that hides it.
- **T7 (R16)** Given a user clicks a release cell, then up to 5 quick picks appear that exclude the item's current releases and archived releases. When they type, then the quick picks hide and search results show.
- **T8 (R22, R23)** Given the merged release/objective/bet column is on, when the user clicks Remove on one objective value, then only that link is removed.
- **T9 (R24, R25)** Given a table grouped by owner (first) and status (second), when the user drags item X into the subgroup "Ben / Done", then X's owner is Ben and X's status is Done.
- **T10 (R24)** Given an item has no owner, then it shows in "No value" after the other groups.
- **T11 (R27)** Given dropping X into a subgroup would set a status X is not allowed to take, when dropped, then neither field changes and an error shows.
- **T12 (R28)** Given sort by name ascending on root items and by size descending on slices, then root rows are ordered by name and the slices inside each parent by size, with empty size last.
- **T13 (R29)** Given hide childless is on, a feature with no slices disappears, and so does a feature whose only slice was removed by a filter.
- **T14 (R30)** Given a table with two filters and 3 visible columns, when exported, then the CSV has only the filtered rows and 3 columns in view order, and a cell "=SUM(A1)" is written as "'=SUM(A1)".
- **T15 (R37)** Given a feature from 12 Sep to 20 Oct on a month axis with `span`, then the card covers Sep and Oct. With `end_only` it covers only Oct. In both cases the stored dates do not change.
- **T16 (R38)** Given the root type is `end_only`, when a user sets the second type to `span`, then the change is refused.
- **T17 (R39)** Given a card from 1 Mar to 31 Mar, when the user drags its right edge to 15 Apr and drops, then the item's end date is 15 Apr.
- **T18 (R40)** Given Shift is held while dragging a card to another lane, then dates do not change and the lane attribute changes.
- **T19 (R31)** Given a collaborator opens the milestone menu, then they cannot create a milestone. Given an editor deletes a milestone used by two views after confirming, then it disappears from both.
- **T20 (R42)** Given cards are colored by check-in, an item without a check-in shows grey and the legend lists green, amber, red.
- **T21 (R44)** Given a timeline with unsaved zoom changes, when the user exports PNG with a date range, then the image uses the unsaved view and the saved view is unchanged.
- **T22 (R50)** Given a kanban view by release, when an editor drags release R2 before R1, then another kanban view using releases shows R2 before R1. Hiding R1 in the first view does not hide it in the other.
- **T23 (R53)** Given a feedback view with filters tag = "billing" and state = new, then only feedback with both shows. Changing an entry's state to triaged removes it from the view at once without saving.
- **T24 (R18, R58)** Given a reader opens the link of a table view, then access is refused. Given the same reader opens a shared timeline, then it is read only.
- **T25 (R59, R65)** Given a new view in `my_space`, then only its owner can find it in search. When moved to a space, then members of that space can open it.
- **T26 (R66)** Given a public link on a kanban view, when an anonymous visitor opens it, then they see the saved view and cannot edit. When the owner regenerates the token, then the old URL returns "not found".
- **T27 (R69)** Given three members opened the view within the past 30 days, then the seen-by list shows them, latest first, and a member looking at it now has a mark. A visit 31 days ago is not listed.
- **T28 (R70)** Given a member stars a view, then it shows in their starred list only.
- **T29 (R71)** Given a shared view with a public link is duplicated, then the copy has no shares or links and is owned by the person who duplicated.
- **T30 (R72)** Given a view shared with one member (M) is moved to a space M cannot access, then M loses access.
- **T31 (R74)** Given a private view named "Roadmap Q3", when another member searches "Q3", then it does not appear.
- **T32 (R57)** Given an account view filtered by cohort = "Enterprise", then only those accounts show and the CSV export matches.

## Tier
Table, timeline, kanban, feedback and customer views, sharing, stars: all tiers. PNG export: all tiers. Public view links and per-view access overrides: paid tiers. Seen-by list: all tiers. [decision]

## Open questions
1. Should filtering on a nested item hide its non-matching root items (R12) instead of keeping them as context?
2. Should a workspace-wide limit on rows per table be set (for speed), and what number?
3. Should timeline drag enforce dependency and parent date limits, or only warn? (R39)
4. Can admins read private views for compliance? (R60). Decide together with S14.
5. Should the public link of a table view be allowed later? (R66)
6. Should column order for custom fields be global (R50) or per view?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S09).
