# S08 Releases

## Scope
How teams plan what ships and when. It covers:
- Releases with a fixed status and a timeframe
- Release groups (folders for releases)
- Many-to-many assignment of features to releases
- Archiving
- Effort roll-up into a release
- How releases appear on boards

Related modules:
- Features and subfeatures: [S02](S02-hierarchy.md)
- Statuses and timeframes as fields: [S03](S03-fields-statuses.md)
- Effort: [S06](S06-prioritization.md)
- Health and progress of features: [S07](S07-strategy.md)
- Release columns, swimlanes and timeline: [S09](S09-boards.md)
- Roles: [S01](S01-workspace-roles.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| release | id, workspace_id, name | | yes | |
| | description | rich text | no | |
| | status | enum `upcoming \| in_progress \| completed` | yes | default `upcoming`; not customizable (R3) |
| | owner | member | no | |
| | timeframe | start date, end date, granularity `day \| month \| quarter \| year` | no | R5 |
| | release_group_id | uuid or null | no | R9 |
| | effort | decimal ≥ 0 or null, mode `rollup \| manual` | | R16 |
| | archived | boolean | yes | default false |
| release_group | id, workspace_id, name | | yes | |
| | description | rich text | no | |
| | archived | boolean | yes | default false |
| feature_release_assignment | feature_id, release_id | | | many-to-many (R10) |
| | assigned_by, assigned_at | member, timestamp | | |

## Rules

### Releases
- **R1** A release has a name, optional description, owner, status, timeframe, optional release group, effort and an archived flag. [src:4][src:7][src:8]
- **R2** Releases are not part of the product hierarchy. They exist beside it, and features from any product or component can be in the same release. [src:7]
- **R3** Release status is fixed to three values: `upcoming`, `in_progress`, `completed`. Workspaces cannot add, remove or rename them. [src:4][src:9]
- **R4** A new release starts as `upcoming`. Status is set by hand; it does not change on its own when dates pass. Any change in any direction is allowed (for example completed back to in progress). [decision]
- **R5** Timeframe is a start date and an end date, each optional, with a granularity (day, month, quarter, year). The end date cannot be before the start date. [src:8][decision on the validation]
- **R6** Release names do not need to be unique. Two releases can have the same name, even in the same group. [decision]
- **R7** Deleting a release removes it and all its feature assignments. The UI asks for confirmation and shows how many assignments will be lost. Features are not deleted. Archiving (R12) is the safe alternative. [decision]

### Release groups
- **R8** A release group is a folder. It holds releases and has a name, description and archived flag. [src:4][src:9]
- **R9** A release is in at most one release group, or in none. Groups cannot be nested. [src:8][decision on one-level only]
- **R10a** A group has no status and no timeframe of its own. [decision]
- **R10b** Deleting a group is refused while it holds releases that are not archived. Moving or archiving them first is required. [decision]
- **R10c** Release groups are a paid-tier feature. On lower tiers all releases are ungrouped. [src:9]

### Feature assignment
- **R10** A feature can be assigned to many releases, and a release can hold many features. The same pair can exist only once. [src:4][src:9][src:8]
- **R11** Subfeatures follow the same rule as features. Components and products cannot be assigned. [src:8][decision on subfeatures]
- **R11a** Assigning and unassigning can be done from the release or from the feature. Both views show the same data. [decision]
- **R11b** Moving a feature to a different parent does not change its assignments. [decision]
- **R11c** Every assignment change is written to the feature's history, and triggers the "assignment updated" event in the API (S13). [src:8]

### Archiving
- **R12** A release or a release group can be archived and restored. [src:8]
- **R13** An archived release keeps its assignments, history and links. It is hidden from default lists, board columns, pickers and roll-ups, and cannot receive new assignments. An "include archived" filter shows it again. [decision on details; sources give only the archived flag]
- **R14** Archiving a group archives all its releases. Restoring a group does not restore them automatically; each release is restored on its own. [decision]
- **R15** A release cannot be restored into an archived group; the user restores the group first or picks another group. [decision]

### Effort roll-up
- **R16** Release effort follows S06: new releases are in `rollup` mode. The value is the sum of the effort of the features assigned to it, with empty effort counting as 0. No assigned features and mode `rollup` shows empty. Typing a value switches to `manual`; "reset to rollup" switches back. [src:4][src:10]; see [S06](S06-prioritization.md) R22–R24
- **R17** A feature's effort counts once per release. A feature in two releases counts in both, so release totals can add up to more than the total of all features. [decision]
- **R18** Subfeature effort is not added again: if a feature has subfeatures in rollup mode, the feature's own value (already the sum of its children) is used. If a subfeature is assigned to a release on its own, its own effort counts, and its parent's does not unless the parent is also assigned. [decision]
- **R19** Archived releases are not rolled into any group or workspace total. A group may show the sum of its non-archived releases' effort for information; it is not stored. [decision]
- **R20** The release effort updates within 5 seconds of an assignment change, an effort change on an assigned feature, or an archive. [decision]

### Releases on boards
- **R21** Boards can show releases as a column (a field with the release names), as a filter, as a card attribute, and as board columns or lanes on the columns board. [src:9]; the S09 spec has the full list
- **R22** On the timeline board a release is drawn as a bar from its start to its end date. A release with no dates is not drawn but still shows in lists. [src:9][decision on undated]
- **R23** On a columns board grouped by release, a feature that is in two releases shows in both columns. Moving a card to another release column removes the card from the source release and adds it to the target. Copy is a separate action. [decision]
- **R24** Release groups can be used to group or collapse releases in lists and in the column picker. [decision]
- **R25** Archived releases do not show on boards unless the board filter includes archived. [decision]

## User actions
- Maker or admin: create, edit, delete, archive and restore releases and groups; change status; assign and unassign features; set effort.
- Contributor: assign features they can edit to releases (R11a). Cannot create or edit releases or groups. [decision]
- Viewer: read only.
- Any role that can see a board can filter and group by release.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R3)** When a user tries to create a status "Blocked" for releases, then it is refused. The only values are upcoming, in progress, completed.
- **T2 (R4)** Given release status `completed`, when a maker sets `in_progress`, then the change is saved.
- **T3 (R5)** When a user sets end date before start date, then the change is refused.
- **T4 (R10)** Given feature F assigned to releases R1 and R2, when F is assigned to R1 again, then nothing changes. R1 and R2 both list F.
- **T5 (R11)** When a user assigns a component to a release, then it is refused.
- **T6 (R9, R10b)** Given group G with active release R, when a maker deletes G, then it is refused.
- **T7 (R13)** Given release R with features, when R is archived, then it disappears from board columns and pickers. When "include archived" is on, then R shows with its features.
- **T8 (R14, R15)** Given group G with releases R1 and R2, when G is archived, then R1 and R2 are archived. When G is restored, then R1 and R2 stay archived. When the user restores R1, then R1 is active.
- **T9 (R16)** Given release R with features of effort 3, empty and 5, then R shows 8. When a user types 20, then R is manual and shows 20. When the user resets, then R shows 8.
- **T10 (R17)** Given F with effort 5 is in releases R1 and R2, then both show 5.
- **T11 (R18)** Given feature F (rollup, subfeatures 2 and 3) is assigned to R, then R's effort counts F as 5. Given only subfeature S1 (2) is assigned, then R counts 2.
- **T12 (R23)** Given F in R1 and R2 on a release-grouped columns board, when F is moved from R1 to R3, then F is in R2 and R3.
- **T13 (R7)** When a release with 4 assignments is deleted, then the confirmation shows 4 and after confirming all features remain.
- **T14 (R22)** Given a release with start 1 Jan and end 31 Mar, then it appears on the timeline from 1 Jan to 31 Mar.

## Tier
- Releases: all tiers. [src:9]
- Release groups: paid tiers. [src:9]

## Open questions
1. Should release status change automatically from dates (for example start date reached → in progress)?
2. Should unassigning a feature from a completed release be allowed, or kept as history?
3. Should groups be allowed to nest in v2?
4. Should a release have its own health and progress (calculated from assigned features), like initiatives in S07?
5. Should the effort roll-up have a "count each feature once across releases" option for group totals?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S08).
