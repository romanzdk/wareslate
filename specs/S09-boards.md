# S09 Boards

## Scope
Boards are saved views over workspace data. A board stores only its setup (type, items, columns, filters, sort, grouping, display options). It never stores a copy of the data. This module covers:
- Grid board
- Timeline board
- Columns board
- Insights board (a saved filter over feedback) and customer boards (companies, users)
- Rules shared by all boards: stored config, filtering engine, new-item behavior, private boards, sharing, links, visit list, favorites, duplicate, move, global search

Related modules:
- Roles and teamspaces: [S01](S01-workspace-roles.md)
- Item hierarchy and item types: [S02](S02-hierarchy.md)
- Fields, statuses, health: [S03](S03-fields-statuses.md)
- Feedback and insights: [S04](S04-feedback-insights.md)
- Companies, users, segments: [S05](S05-customers.md)
- Impact score, drivers, formulas, effort columns: [S06](S06-prioritization.md)
- Objectives, initiatives, key results: [S07](S07-strategy.md)
- Releases and timeframes: [S08](S08-releases.md)
- Public portal and published roadmaps: [S11](S11-portal.md)
- Semantic search, AI assistant: [S15](S15-ai.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| board | id, workspace_id | uuid | yes | |
| | kind | enum `grid \| timeline \| columns \| insights \| companies \| users` | yes | cannot change after creation |
| | name | text | yes | |
| | space | enum `personal \| teamspace \| workspace` plus `teamspace_id` | yes | new boards start in `personal` (R59) |
| | owner_id | member | yes | creator |
| | config | JSON | yes | see "Board config" below (R1) |
| | saved_config_version | int | yes | grows on each Save |
| | access_overrides | list of {principal, level} | no | per-board overrides of space access (R63) |
| | link_mode | enum `off \| private_link \| public_link` | yes | default `off` (R66) |
| | link_token | random string | when link on | |
| | created_at, updated_at | timestamp | yes | |
| board_favorite | board_id, member_id | | | one row per star |
| board_visit | board_id, member_id, last_visited_at | | | for the visit list (R69) |
| milestone | id, workspace_id, name, description, timeframe (date or date range), color | | yes | workspace-wide, not owned by a board |
| board_milestone | board_id, milestone_id | | | which timeline boards show it (R31) |

### Board config (stored as data)
| key | used by | meaning |
|---|---|---|
| `items` | grid, timeline, columns | ordered list of item types. First = main type, others = nested child types |
| `columns` | grid | ordered list of `{field, width, visible, sort, filter}` |
| `combined_columns` | grid | which of release / objective / initiative use one shared column |
| `grouping` | grid, timeline, columns | list of 0–2 field ids (timeline: swimlanes) |
| `sort` | grid | map `item_type_level → {column, direction}` |
| `filters` | all | list of filter rows (R8) |
| `hide_empty` | grid, timeline | boolean |
| `granularity`, `zoom`, `time_horizon` | timeline | see R34–R38 |
| `card_attributes`, `color_by` | timeline, columns | see R41, R42 |
| `column_source` | columns | datatype and hidden values (R45) |
| `milestones` | timeline | milestone ids (R31) |

## Rules

### Common to all boards
- **R1** A board is stored as a config record. Saving a board writes the config only. Items, feedback, companies and users are never copied into it. [src:4][src:12]
- **R2** Changing the config of an open board makes it "modified". A modified board offers Save (overwrite), Save as new (new board, same space, name + " copy" unless typed), and Discard (go back to the last saved config). Leaving the board without saving discards the changes. [src:4]
- **R3** Changes to the data (status, owner, text, links) show at once on every board that includes the item. They do not need a save. [src:4]
- **R4** Saving changes the board for everyone who can open it. Only people who can edit the board (R62) can Save. A viewer of the board can still change filters and sort for themselves; those changes stay in their own session and cannot be saved (R2 discard rules apply). [src:4; session-only edits for read-only users are a decision]
- **R5** Filtering engine: filter rows on a board are combined with AND. An item or feedback is shown only if it matches every row. [src:4][src:12]
- **R6** There is one filtering engine for all boards. It supports a filter in the board header, a filter on one column, and a filter in the board controls. All three write to the same `filters` list. A column filter is the same as a board filter on that field. [src:1][decision on "same list"]
- **R7** Operators per field type: [decision, sources only say "filter per board or per column"]

  | field type | operators |
  |---|---|
  | single select, status, health, owner, member, team, item type | is any of, is none of, is empty, is not empty |
  | multi select, tags, linked items (release, objective, initiative, parent) | has any of, has all of, has none of, is empty, is not empty |
  | text, title, note content | contains, does not contain, is empty, is not empty. Wrapping words in double quotes matches the exact phrase |
  | number, effort, driver, impact score, formula | =, ≠, >, ≥, <, ≤, between, is empty, is not empty |
  | date, timeframe | is, before, after, between, in the last N days, in the next N days, is empty, is not empty |
  | boolean | is true, is false |

- **R8** A filter row is `{field, operator, values[]}`. Rows with an empty value list (except is empty / is not empty) are ignored while editing and not saved. [decision]
- **R9** Within one row, multiple values are OR-ed for "is any of" and "has any of", AND-ed for "has all of". Across rows it is always AND. [src:5 AND rule from sources; within-row meaning is a decision]
- **R10** "Filter by me" is a shortcut for member/owner columns. It adds a row "is any of [current member]". "Filter by my teams" does the same for team columns with the current member's teams. [src:1]
- **R11** "Filter by me" stores the member id, not the word "me". When a board is saved and another member opens it, it shows the original member's filter. A row may be saved as "current viewer" instead; then it resolves per viewer. [decision]
- **R12** A filter on a field applies to the main items and to nested child items by this rule: a main item is shown when it matches, or when at least one of its nested children matches; non-matching children are hidden and non-matching main items stay visible as context. [decision, sources silent]
- **R13** New-item behavior: when a user creates a main item on a board that has a single-value filter (one status, one owner, one team, one tag), the new item gets that value. A notice names the values that were filled. [src:1][src:2][src:3]
- **R14** If active filters are multi-value or on a type that cannot be auto-filled (effort, date, number), the new item may not match. It is still created. An alert says it is hidden by filters and offers: show it (clear the blocking rows), or keep the filters. [src:1][src:2][src:3; two buttons are a decision]
- **R15** Search results inside a board go to the item's place on that board, expanding collapsed parents. Words in double quotes search for an exact phrase. [src:2]
- **R16** Pickers that link an item (release, objective, initiative, parent) show a "Suggested" section of up to 5 items before the user types. It prefers items the user linked before, recently opened items, and items the user owns. It leaves out the item itself, already-linked items, archived items and items the user cannot see. Typing hides the section and searches the full list. [src:1][src:2]
- **R17** Inline create: grid and timeline have "create" at the bottom (or on empty space) to add a main item. Hovering a main item offers "add item" for a nested child. [src:1][src:2][src:3]
- **R18** Board kinds visible per role: viewers cannot open grid boards or insights boards. They can open shared timeline and columns boards in read-only form, and published roadmaps. [src:11][decision on read-only timeline/columns; sources say viewers see shared roadmaps]

### Grid board
- **R19** At creation the user picks one main item type (objective, initiative, product, component, feature, release, or another type the workspace has). More types can be added below it as nested child types. The first type in `items` is the main type. The available types depend on the plan. [src:1]
- **R20** Nested child rows appear under their main item. Child types must be allowed under the parent type by the hierarchy rules of [S02](S02-hierarchy.md). [src:1][decision on S02 link]
- **R21** Columns:
  - Add a column from a side panel with the default fields and every other field/element type, with search. A toggle shows or hides the column.
  - Reorder by dragging the column header.
  - Resize by dragging the divider between headers. A column has a minimum width of 80 px.
  - Available column kinds: item fields, custom fields, hierarchy (parent), portal, tasks, health, timeframe as a small bar chart (mini-Gantt), end date, impact score and company/segment columns, drivers, formulas, effort ([S06](S06-prioritization.md)). [src:1][src:12]
  - Column order and widths are part of the board config only. [src:1][decision on "only"]
- **R22** Combined columns: release, objective and initiative links can be shown in one column that lists all linked values, per element kind ("Assigned [kind]" switch). It is available only for those three kinds. [src:1]
- **R23** Inline edit: clicking a cell edits it in place. Link cells open the picker (R16). To remove a link, hover the value in the cell and press "Remove". Read-only columns (computed values, rollups) do not open an editor. Editing is allowed only if the member has edit rights on the item ([S01](S01-workspace-roles.md)). [src:1][decision on rights and read-only]
- **R24** Grouping: a grid can be grouped by 0, 1 or 2 attributes. A second attribute makes subgroups inside each first-level group. Items with no value for a grouping attribute go to a "Not assigned" group, placed last. Grouping applies to the main item level. [src:1][decision on main level]
- **R25** Drag inside a group changes the manual order. Drag to another group changes the item's field value: for a first-level group it sets the first attribute; for a subgroup it sets both attributes (the first from the group, the second from the subgroup). Dropping into "Not assigned" clears the attribute (both when dropped in a subgroup of it). [src:1][decision on clearing]
- **R26** For multi-value attributes (release, tags), dropping into a group replaces the value of the group it was dragged out of with the target group's value and leaves other values alone. [decision]
- **R27** If one of the two updates in R25 fails (for example, a required field or no edit right), neither is applied and the item snaps back with an error message. [decision]
- **R28** Sort: a column's menu sets ascending or descending. Not all columns can sort (for example the mini-Gantt). Sort is kept per nesting level: main items use the main-level sort, each nested type uses its own. One sort at a time per level; setting a new one replaces the old. Empty values sort last in both directions. Sort works inside groups. [src:1][decision on one-per-level, empty-last]
- **R29** Hide empty: the "hide empty items" switch removes main items that have no nested children of the shown child types. It counts children after filters. [src:2][decision on exact meaning]
- **R30** CSV export: exports the grid as currently filtered, sorted and grouped, with the visible columns in column order, nested rows below their parent with a level column. Multi-value cells are joined with "; ". Dates are ISO 8601. Cells that start with `= + - @` are prefixed with `'` to block spreadsheet formula injection. A grid with more than 50,000 rows exports in the background and the user gets a download notice. [src:12 says export to CSV exists][decision for all details]

### Timeline board
- **R31** Milestones are workspace-wide records (name, description, timeframe, color). Creating one on a timeline also assigns it to that board. A milestone can be assigned to many boards. Assigning needs a Save. Only admins and makers can create, edit, assign, remove and delete milestones. Deleting removes it from every board and needs confirmation. Milestones show as colored diamonds with a label. Close milestones are grouped; clicking the group shows its members. The milestone menu splits them into past and future by the today marker. [src:5]
- **R32** Nesting: `items` holds up to 4 item types, one per level. The first is the main type. Rows can collapse and expand. Hide empty works as in R29. [src:2]
- **R33** Swimlanes: grouping puts rows into horizontal lanes by an attribute. A second grouping makes nested lanes inside the first. Maximum 2 levels, same "Not assigned" rule as R24. Custom fields can be used. [src:2][decision on max 2 matching the grid]
- **R34** Granularity: week, month or quarter. It sets the column unit of the time axis. [src:2]
- **R35** Zoom changes how much time is visible within the chosen granularity. It does not change item dates. Zoom is saved with the board config. [src:2][decision on saving]
- **R36** Items appear on the timeline only if they have a timeframe. Items without one are listed in an "unscheduled" area; dragging one onto the axis sets its timeframe. [src:2 first part][decision on unscheduled area]
- **R37** Time horizons (snap) are visual only. They stretch each card to whole units of the granularity and do not change stored dates. Modes:
  - Duration (default): the card covers every unit it touches (12 Sep–20 Oct on a month axis covers Sep and Oct).
  - End date: the card covers only the last unit it touches (Oct).
  - Flat list (child types only): the card takes its parent's horizon and hides its own timeframe.
  [src:6]
- **R38** Horizon mode limits by level: the top type uses Duration or End date, never Flat list. If the top type is End date, all lower types are Flat list. If the top type is Duration, level 2 may use any mode and level 3 and 4 are Flat list only. A child linked straight to a level-1 item (skipping level 2) uses the level-2 mode. While time horizons are on, dragging cards to change dates is turned off. [src:6]
- **R39** Drag and resize: dragging a card moves its timeframe. Dragging an edge changes the start or the end. The new dates are written to the item's timeframe field ([S08](S08-releases.md) for releases). While dragging, dates snap to whole days; with snapping on, to the start/end of the granularity unit. The write happens on drop. If the write fails the card returns. A member without edit rights cannot drag. Dependencies and parent bounds are not enforced in v1. [src:2][decision on snapping, rights, no enforcement]
- **R40** Vertical-only drag: holding Shift while dragging locks movement to up/down. This changes row order or lane and never changes dates. If the card is dropped in another swimlane, the grouping attribute changes as in R25. [src:2][decision on lane drop]
- **R41** Card attributes: a card customization menu picks which fields show on cards, for all item types. Default: name only. The parent name can be shown on cards. [src:2][src:12][decision on default]
- **R42** Color coding: the main item type's cards can be colored by health (on track, at risk, blocked), a single-select custom field, a multi-select custom field, status, or task completion. Works on timeline and columns boards only. Items with no value use a neutral grey. A multi-select item with several values uses the first value in field order. The board shows a legend. [src:7][decision on neutral color, first value, legend]
- **R43** Health on cards: when health is chosen as a card attribute, the card shows the latest health value and its date ([S03](S03-fields-statuses.md)). [src:12][decision on details]
- **R44** PNG export: available on timeline and columns boards. The user picks a date range and display options and gets an image. The board does not need to be saved first. Unsaved config changes are used for the image and can be discarded after. [src:9]

### Columns board
- **R45** A columns board has one set of columns. The set comes from a datatype chosen in board controls: release, objective, initiative, product, component, feature, or a single-select/multi-select custom field. It can be changed at any time. [src:3][src:12]
- **R46** Each column value can be hidden or shown (eye toggle). An empty column can be added so items can be assigned to it. [src:3]
- **R47** Swimlanes work as R33. Local and global order is described in R50. [src:3][decision on the order]
- **R48** Card attributes and color coding work as R41 and R42. [src:3][src:7]
- **R49** Column reorder can be done by dragging in board controls, by dragging a column header on the board, or from the column's menu. [src:3]
- **R50** For columns that are items (release, objective, initiative, product, component, feature), reordering changes the order of those items for the whole workspace, so every board that uses them sees the new order ("global order"). For custom-field columns the order is the field option order, also global. Hiding a column is local to the board. Order inside a swimlane is local to the board. [src:3][decision on custom field and lane order]
- **R51** Dropping a card into another column changes the item's value for the column field (as in R25 and R26). Sources do not describe this, so it follows the grid rule. Dropping into a hidden-column area is not possible. [decision]
- **R52** Items can be added from an empty area ("click to add item") for a main item, and from the column menu "Add" for new or existing nested items if the board shows them. R13 and R14 apply. [src:3]

### Insights board and customer boards
- **R53** An insights board is a saved filter over feedback ([S04](S04-feedback-insights.md)). It shows feedback items that match all its filters (R5). It stores only the filter list. [src:4]
- **R54** Every workspace has a default insights board with preset filters. It can be changed only by admins; others can duplicate it. [src:4][decision on admin-only edits]
- **R55** Filter fields for feedback: status (processed/unprocessed), owner, tags, company, user, source, created date range, linked item, importance, text. Operators follow R7. The filter list is the same as the feedback list filters in S04. [src:4][src:12][decision on the exact list]
- **R56** Feedback list export to CSV follows R30 (what is shown, in order). [src:12][decision]
- **R57** Customer boards: a companies board and a users board list those entities ([S05](S05-customers.md)) as rows, with columns and filters under the same engine. Companies board columns include name, segment, custom company fields, aggregated customer fields (for example total revenue of linked companies), user count and linked feedback count. Users board columns include name, email, company and custom user fields. Inline edit and CSV export work as in the grid. No nesting, no grouping beyond 1 level. [src:12][decision on columns, 1 level]
- **R58** Contributors can open and use default and existing boards of these kinds. Makers and admins can create and edit them. Viewers cannot open insights boards. [src:4][src:11]

### Private boards, sharing, links
- **R59** A new board is created in the creator's personal space and is private. Only the owner (and workspace admins, see R60) can see it. Creating it from home inside a teamspace or folder puts it there. [src:4]
- **R60** Admins can see all boards for management and can move or delete them. They cannot read the content of a private board without the owner's sharing it. [decision, favoring privacy; revisit with S14]
- **R61** Board access levels: view, can edit content, edit, owner. A board in a teamspace follows the teamspace role by default. [src:12][decision on mapping, see S01]
- **R62** Edit rights on the board config (Save, rename, delete, move) need level edit or owner. "Can edit content" lets the member change item data inside the board but not the config. View lets them read only. [src:12][decision on meaning]
- **R63** Per-board overrides can raise or lower access for a person or teamspace compared to the space. The effective level is the highest of the space level and the override for that principal, unless an override explicitly says "none" (then access is removed). [src:12][decision on resolution]
- **R64** Sharing: a board can be shared with named people, with a teamspace, or with the whole workspace. Sharing a personal board does not move it. The board list has "Shared by me" and "Shared with me" tabs. [src:12]
- **R65** Moving a board to a teamspace makes it visible to everyone with access to that teamspace. A personal board stays private until it is moved or shared. [src:4]
- **R66** Links: a board has link mode `off`, `private_link` or `public_link`. A private link needs a signed-in workspace member with access. A public link shows a read-only view of the published config to anyone with the URL and needs no sign-in; it is available for timeline and columns boards only. The URL contains a random token (at least 128 bits). Turning the link off or regenerating the token stops the old URL at once. [src:10][decision on private link meaning, token strength]
- **R67** An admin can turn off public board links for the whole workspace. Existing public links then stop working and are shown as "off" (they come back if the switch is turned on again, unless regenerated). [src:10][decision on restore]
- **R68** A public link shows the board's saved config, not the viewer's changes, hides internal-only fields (those not marked public in S03) and cannot be edited. Public timeline and columns views are the same as published roadmaps in [S11](S11-portal.md). [src:10][decision on hidden fields]
- **R69** Visit list: the board header shows who looked at the board in the last 30 days, most recent first, with a mark for people viewing it now. Every workspace member can open this list, whatever their role. Anonymous public-link views are counted as one number, not listed by name. [src:8][decision on anonymous count]
- **R70** Favorites: a member stars a board to add it to their own favorites list. Unstar removes it. Favorites are per member and never shown to others. Any member who can open the board can star it. [src:4; research is inconsistent about who may star, so "any member who can open it" is a decision]
- **R71** Duplicate: creates a copy of the config in the same space, owned by the person who duplicated. The copy starts private to that person if the original was in a personal space, otherwise it takes the space of the original. Shares, overrides, links, favorites and visit history are not copied. Milestone assignments are copied (milestones themselves are shared). The member needs view access to the original. [src:4][decision on what is copied]
- **R72** Move to teamspace: changes `space`. The owner stays. Access overrides are cleared. Member shares are kept only if the member has access to the target teamspace. Links stay as they are. Needs edit or owner level on the board and member rights in the target teamspace. [src:4][decision on shares and overrides]
- **R73** Delete: only owner, board editors and admins. Deleting a board does not delete items, feedback or milestones. [decision]

### Global search
- **R74** Global search finds boards by name and by owner, together with other objects. Results respect board access (R61). A private board never shows in another member's results. [src:12][decision]
- **R75** Search combines keyword and semantic matching as defined in [S15](S15-ai.md); this module only sets the access check on results. [src:12]

## User actions

| action | admin | maker | contributor | viewer |
|---|---|---|---|---|
| Create grid, timeline, columns, insights boards | yes | yes | no | no |
| Create companies / users boards | yes | yes | no | no |
| Open grid and insights boards | yes | yes | yes | no |
| Open shared timeline / columns boards, published roadmaps | yes | yes | yes | yes (read only) |
| Change filter, sort, zoom (session only) | yes | yes | yes | yes |
| Save board config | if board editor | if board editor | if board editor | no |
| Edit item data inline, drag cards to change dates | yes | yes | only items they may edit | no |
| Create / edit / assign milestones | yes | yes | no | no |
| Share, set links, move, delete a board | board owner or editor, and admin | board owner or editor | no | no |
| Star a board, see visit list | yes | yes | yes | yes |
| Turn public board links on or off for the workspace | yes | no | no | no |
| Export CSV / PNG | yes | yes | yes | PNG only if board is shared with them |

Contributor and viewer details are [decision] where sources are silent. See [S01](S01-workspace-roles.md) for the role model.

## Acceptance tests
- **T1 (R1, R2)** Given a user changes a column filter and leaves the board without saving, when they come back, then the board shows the last saved config. When they change it and press Save as new, then a second board exists and the first is unchanged.
- **T2 (R3)** Given feature F appears on two boards, when its status changes on board A, then board B shows the new status with no save.
- **T3 (R5, R9)** Given filters status is any of [Planned, In progress] and owner is any of [Ana], when the grid loads, then only items with one of those statuses and owner Ana show.
- **T4 (R10, R11)** Given a user presses "Filter by me" on the owner column, when another member opens the saved board, then they see Ana's items (the original user), not their own, unless the row was saved as "current viewer".
- **T5 (R13)** Given a status filter with one value "Planned", when a user creates a feature inline, then the feature has status Planned and a notice says so.
- **T6 (R14)** Given an effort filter "> 5", when a user creates a feature inline, then the feature exists, is hidden, and an alert offers to clear the blocking row.
- **T7 (R16)** Given a user clicks a release cell, then up to 5 suggestions appear that exclude the item's current releases and archived releases. When they type, then suggestions hide and search results show.
- **T8 (R22, R23)** Given the combined release/objective/initiative column is on, when the user clicks Remove on one objective value, then only that link is removed.
- **T9 (R24, R25)** Given a grid grouped by owner (first) and status (second), when the user drags item X into the subgroup "Ben / Done", then X's owner is Ben and X's status is Done.
- **T10 (R24)** Given an item has no owner, then it appears in "Not assigned" at the end of the groups.
- **T11 (R27)** Given dropping X into a subgroup would set a status X is not allowed to take, when dropped, then neither field changes and an error shows.
- **T12 (R28)** Given sort by name ascending on main items and by effort descending on subfeatures, then main rows are ordered by name and the subfeatures inside each parent by effort, with empty effort last.
- **T13 (R29)** Given hide empty is on, a feature with no subfeatures disappears and a feature whose only subfeature was removed by a filter also disappears.
- **T14 (R30)** Given a grid with two filters and 3 visible columns, when exported, then the CSV has only the filtered rows and 3 columns in board order, and a cell "=SUM(A1)" is written as "'=SUM(A1)".
- **T15 (R37)** Given a feature from 12 Sep to 20 Oct on a month axis with Duration, then the card covers Sep and Oct. With End date it covers only Oct. The stored dates do not change in both cases.
- **T16 (R38)** Given the top type is End date, when a user sets the second type to Duration, then the change is refused.
- **T17 (R39)** Given a card from 1 Mar to 31 Mar, when the user drags its right edge to 15 Apr and drops, then the item's end date is 15 Apr.
- **T18 (R40)** Given Shift is held while dragging a card to another lane, then dates do not change and the lane attribute changes.
- **T19 (R31)** Given a contributor opens the milestone menu, then they cannot create a milestone. Given a maker deletes a milestone used by two boards after confirming, then it disappears from both.
- **T20 (R42)** Given cards are colored by health, an item without health shows grey and the legend lists on track, at risk, blocked.
- **T21 (R44)** Given a timeline with unsaved zoom changes, when the user exports PNG with a date range, then the image uses the unsaved view and the saved board is unchanged.
- **T22 (R50)** Given a columns board by release, when a maker drags release R2 before R1, then another columns board using releases shows R2 before R1. Hiding R1 on the first board does not hide it on the other.
- **T23 (R53)** Given an insights board with filters tag = "billing" and status = unprocessed, then only feedback with both shows. Changing a feedback's status removes it from the board at once without saving.
- **T24 (R18, R58)** Given a viewer opens the link of a grid board, then access is refused. Given the same viewer opens a shared timeline, then it is read only.
- **T25 (R59, R65)** Given a new board in a personal space, then only its owner can find it in search. When moved to a teamspace, then members of that teamspace can open it.
- **T26 (R66)** Given a public link on a columns board, when an anonymous visitor opens it, then they see the saved view and cannot edit. When the owner regenerates the token, then the old URL returns "not found".
- **T27 (R69)** Given three members visited in the last 30 days, then the visit list shows them, newest first, and a member viewing now has a mark. A visit 31 days ago is not listed.
- **T28 (R70)** Given a member stars a board, then it shows in their favorites only.
- **T29 (R71)** Given a shared board with a public link is duplicated, then the copy has no shares or links and is owned by the person who duplicated.
- **T30 (R72)** Given a board shared with member M is moved to a teamspace M cannot access, then M loses access.
- **T31 (R74)** Given a private board named "Roadmap Q3", when another member searches "Q3", then it does not appear.
- **T32 (R57)** Given a companies board filtered by segment = "Enterprise", then only those companies show and CSV export matches.

## Tier
Grid, timeline, columns, insights and customer boards, sharing, favorites: all tiers. PNG export: all tiers. Public board links and per-board access overrides: paid tiers. Visit list: all tiers. [decision]

## Open questions
1. Should filtering a nested child hide its non-matching main items (R12) instead of keeping them as context?
2. Should a workspace-wide limit on rows per grid be set (for speed), and what number?
3. Should timeline drag enforce dependency and parent date bounds, or only warn? (R39)
4. Can admins read private boards for compliance? (R60). Decide together with S14.
5. Should the public link of a grid board be allowed later? (R66)
6. Should column order for custom fields be global (R50) or per board?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S09).
