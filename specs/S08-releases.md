# S08 Releases

## Scope
How teams plan what ships and when. It covers:
- Releases with a fixed status and a date range
- Release tracks (folders for releases)
- Many-to-many assignment of features to releases
- Archiving
- Size roll-up into a release
- How releases appear in views

Related modules:
- Features and slices: [S02](S02-product-tree.md)
- Statuses and date ranges as fields: [S03](S03-fields-statuses.md)
- Size: [S06](S06-prioritization.md)
- Check-ins and progress of features: [S07](S07-strategy.md)
- Release columns, lanes and timeline: [S09](S09-views.md)
- Roles: [S01](S01-workspace-roles.md)

## Entities

| entity | field | type | required | remarks |
|---|---|---|---|---|
| release | id, workspace_id, name | | yes | |
| | description | rich text | no | |
| | status | enum `planned \| active \| shipped` | yes | default `planned`; not customizable (R3) |
| | owner | member | no | |
| | date_range | start date, end date, precision `day \| month \| quarter \| year` | no | R5 |
| | release_track_id | uuid or null | no | R9 |
| | size | decimal ≥ 0 or null, mode `from_children \| manual` | | R16 |
| | archived | boolean | yes | default false |
| release_track | id, workspace_id, name | | yes | |
| | description | rich text | no | |
| | archived | boolean | yes | default false |
| feature_release_assignment | feature_id, release_id | | | many-to-many (R10) |
| | assigned_by, assigned_at | member, timestamp | | |

## Rules

### Releases
- **R1** A release has a name, optional description, owner, status, date range, optional release track, size and an archived flag. [src:4][src:7][src:8]
- **R2** Releases sit outside the product tree. They exist beside it, and features from any product or module can be in the same release. [src:7]
- **R3** Release status has three fixed values: `planned`, `active`, `shipped`. Workspaces cannot add, remove or rename them. [src:4][src:9]
- **R4** A new release starts as `planned`. Status is set by hand; it does not change on its own when dates pass. Any change in any direction is allowed (for example shipped back to active). [decision]
- **R5** Date range is a start date and an end date, each optional, with a precision (day, month, quarter, year). The end date cannot be before the start date. [src:8][decision on the validation]
- **R6** Release names do not need to be unique. Two releases can have the same name, even in the same track. [decision]
- **R7** Deleting a release removes it and all its feature assignments. The UI asks for confirmation and shows how many assignments will be lost. Features are not deleted. Archiving (R12) is the safe alternative. [decision]

### Release tracks
- **R8** A release track is a folder. It holds releases and has a name, description and archived flag. [src:4][src:9]
- **R9** A release is in at most one release track, or in none. Tracks cannot be nested. [src:8][decision on one-level only]
- **R10a** A track has no status and no date range of its own. [decision]
- **R10b** Deleting a track is refused while it holds releases that are not archived. Moving or archiving them first is required. [decision]
- **R10c** Release tracks are a paid-tier feature. On lower tiers all releases are ungrouped. [src:9]

### Feature assignment
- **R10** A feature can be assigned to many releases, and a release can hold many features. A given feature and release can be paired only once. [src:4][src:9][src:8]
- **R11** Slices use the same rule as features. Modules and products cannot be assigned. [src:8][decision on slices]
- **R11a** Assigning and unassigning can be done from the release or from the feature. Both sides show the same data. [decision]
- **R11b** Moving a feature under another parent does not change its assignments. [decision]
- **R11c** Every assignment change is written to the feature's history, and triggers the "assignment updated" event in the API (S13). [src:8]

### Archiving
- **R12** A release or a release track can be archived and restored. [src:8]
- **R13** An archived release keeps its assignments, history and links. It is hidden from default lists, view columns, pickers and roll-ups, and cannot receive new assignments. An "include archived" filter shows it again. [decision on details; sources give only the archived flag]
- **R14** Archiving a track archives all its releases. Restoring a track does not restore them automatically; each release is restored on its own. [decision]
- **R15** A release cannot be restored into an archived track; the user restores the track first or picks another track. [decision]

### Size roll-up
- **R16** Release size works as in S06: new releases are in `from_children` mode. The value is the sum of the size of the features assigned to it, with an empty size counting as 0. No assigned features in `from_children` mode shows empty. Typing a value switches to `manual`; "reset to from children" switches back. [src:4][src:10]; see [S06](S06-prioritization.md) R22–R24
- **R17** A feature's size counts once per release. A feature in two releases counts in both, so release totals can add up to more than the total of all features. [decision]
- **R18** Slice size is not added twice: if a feature has slices in `from_children` mode, the feature's own value (already the sum of its children) is used. If a slice is assigned to a release on its own, its own size counts, and its parent's does not unless the parent is also assigned. [decision]
- **R19** Archived releases are not rolled into any track or workspace total. A track may show the sum of its non-archived releases' size for information; it is not stored. [decision]
- **R20** The release size updates within 5 seconds of an assignment change, a size change on an assigned feature, or an archive. [decision]

### Releases in views
- **R21** Views can show releases as a column (a field with the release names), as a filter, as a card field, and as columns or lanes on a kanban view. [src:9]; the S09 spec has the full list
- **R22** On a timeline view a release is drawn as a bar from its start to its end date. A release with no dates is not drawn but still shows in lists. [src:9][decision on undated]
- **R23** On a kanban view grouped by release, a feature that is in two releases shows in both columns. Moving a card to another release column removes the card from the source release and adds it to the target. Copy is a separate action. [decision]
- **R24** Release tracks can be used to group or collapse releases in lists and in the column picker. [decision]
- **R25** Archived releases do not show in views unless the view filter includes archived. [decision]

## User actions
- Editor or admin: create, edit, delete, archive and restore releases and tracks; change status; assign and unassign features; set size.
- Collaborator: assign features they can edit to releases (R11a). Cannot create or edit releases or tracks. [decision]
- Reader: read only.
- Any role that can see a view can filter and group by release.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R3)** When a user attempts to add a status "Blocked" for releases, then it is refused. The only values are planned, active, shipped.
- **T2 (R4)** Given release status `shipped`, when an editor sets `active`, then the change is saved.
- **T3 (R5)** When a user sets end date before start date, then the change is refused.
- **T4 (R10)** Given feature F assigned to releases R1 and R2, when F is assigned to R1 again, then nothing changes. R1 and R2 both list F.
- **T5 (R11)** When a user assigns a module to a release, then it is refused.
- **T6 (R9, R10b)** Given track G with non-archived release R, when an editor deletes G, then it is refused.
- **T7 (R13)** Given release R with features, when R is archived, then it disappears from view columns and pickers. When "include archived" is on, then R shows with its features.
- **T8 (R14, R15)** Given track G with releases R1 and R2, when G is archived, then R1 and R2 are archived. When G is restored, then R1 and R2 stay archived. When the user restores R1, then R1 is no longer archived.
- **T9 (R16)** Given release R with features of size 3, empty and 5, then R shows 8. When a user types 20, then R is manual and shows 20. When the user resets, then R shows 8.
- **T10 (R17)** Given F with size 5 is in releases R1 and R2, then both show 5.
- **T11 (R18)** Given feature F (`from_children`, slices 2 and 3) is assigned to R, then R's size counts F as 5. Given only slice S1 (2) is assigned, then R counts 2.
- **T12 (R23)** Given F in R1 and R2 on a kanban view grouped by release, when F is moved from R1 to R3, then F is in R2 and R3.
- **T13 (R7)** When a release with 4 assignments is deleted, then the confirmation shows 4 and after confirming all features remain.
- **T14 (R22)** Given a release with start 1 Jan and end 31 Mar, then it appears on the timeline from 1 Jan to 31 Mar.

## Tier
- Releases: all tiers. [src:9]
- Release tracks: paid tiers. [src:9]

## Open questions
1. Should release status change automatically from dates (for example start date reached → active)?
2. Should unassigning a feature from a shipped release be allowed, or kept as history?
3. Should tracks be allowed to nest in v2?
4. Should a release have its own check-in and progress (calculated from assigned features), like bets in S07?
5. Should the size roll-up have a "count each feature once across releases" option for track totals?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S08).
