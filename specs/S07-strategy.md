# S07 Strategy

## Scope
How teams write down goals and track whether work is on course. It covers:
- Objectives and the objective hierarchy
- Key results (measurable outcomes of an objective)
- Initiatives (groups of work linked to objectives and features)
- Progress (manual, status-based, calculated)
- Health updates and the health roll-up rule
- Followers and notifications for health updates

Related modules:
- Feature hierarchy and subfeatures: [S02](S02-hierarchy.md)
- Statuses, owners, timeframes, teams: [S03](S03-fields-statuses.md)
- Effort roll-up (initiatives and objectives have effort): [S06](S06-prioritization.md)
- Releases: [S08](S08-releases.md)
- Showing objectives, initiatives and health on boards: [S09](S09-boards.md)
- Roles and teamspaces: [S01](S01-workspace-roles.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| objective | id, workspace_id, name | | yes | |
| | description | rich text | no | |
| | parent_objective_id | uuid or null | no | objective hierarchy (R1) |
| | status | status id (S03) | no | |
| | owner | member | no | |
| | timeframe | start date, end date, granularity `day \| month \| quarter \| year` | no | |
| | progress | `{value 0–100, mode}` | derived or manual | R16 |
| | health | see health record | derived or manual | R22 |
| | archived | boolean | yes | default false |
| key_result | id, name, objective_id | | yes | belongs to exactly one objective (R6) |
| | description | rich text | no | |
| | start_value, current_value, target_value | decimal | yes | R8 |
| | status | status id | no | |
| | owner | member | no | |
| | team | team (S01) | no | |
| | health | health record | | |
| | archived | boolean | yes | default false |
| initiative | id, name | | yes | |
| | description, status, owner, timeframe, archived | | | as objective |
| | progress | `{value 0–100, mode}` | | R16 |
| | health | health record | | |
| objective_initiative_link | objective_id, initiative_id | | | many-to-many (R11) |
| initiative_feature_link | initiative_id, feature_id | | | many-to-many (R12) |
| objective_feature_link, key_result_feature_link | | | | many-to-many (R13) |
| feature (from S02) | progress | `{value 0–100, mode}` | | R16 |
| | health | health record | | features can have health too (R18) |
| health_update | id, entity_type, entity_id | | yes | one row per update; history is kept |
| | status | enum `on_track \| at_risk \| off_track` | yes | `not_set` is the state before any update |
| | mode | enum `manual \| calculated` | yes | |
| | comment | rich text | no | |
| | created_by, created_at | member, timestamp | yes | |
| | previous_status | enum | no | status before this update |
| follower | entity_type, entity_id, member_id | | | R26 |

## Rules

### Objectives
- **R1** Objectives can be nested: an objective may have one parent objective, and any number of child objectives. The tree has no cycles; a move that would make an objective its own ancestor is refused. [src:5][decision on no-cycles, depth limit none]
- **R2** Moving an objective moves its whole subtree. Deleting an objective with children is refused until the children are moved or deleted. [decision]
- **R3** Objective fields: name, description, status, owner, timeframe, progress, health, effort. Effort follows S06 R22–R24. [src:5][src:9]
- **R4** The objective hierarchy is a paid-tier feature. Flat objectives are available on lower tiers. [src:9]
- **R5** An archived objective stays visible in history and in its links, but is hidden from default lists, boards and roll-ups. [src:8][decision on roll-up exclusion]

### Key results
- **R6** A key result belongs to exactly one objective. It cannot be created without one. Deleting its objective deletes it (after confirmation). [src:5][src:6]
- **R7** Key result fields: name, description, start, current and target value, status, owner, team, health, archived. [src:1][src:6]
- **R8** Start, current and target are numbers (decimals allowed). Start and target must differ. Target may be lower than start (for example "reduce churn from 8 to 5"). [src:1][src:6]; the rules on start ≠ target and downward targets are [decision]
- **R9** Key result progress in percent = (current − start) ÷ (target − start) × 100. It is not capped by the data, but display and roll-up clamp it between 0 and 100. [decision; formula follows the start/current/target fields in src 6]
- **R10** Changing the current value recalculates progress at once and records the change in history. [decision]

### Links
- **R11** An initiative can be linked to many objectives, and an objective to many initiatives. A link is not a parent-child relation: it does not move, nest or delete anything. [src:2][src:9]
- **R12** An initiative can be linked to many features, and a feature to many initiatives. Same rule: link only. [src:2][src:9]
- **R13** Objectives and key results can also be linked directly to features. [src:7]
- **R14** Removing a link or archiving one side never deletes the other side. [decision]
- **R15** Links are created and removed from either side (for example from the feature or from the initiative). [decision]

### Progress
- **R16** Progress is a whole number 0–100 with a mode. Allowed modes by type: [src:6]
  - Objective: `manual`, `calculated`
  - Initiative: `manual`, `calculated`
  - Feature: `manual`, `status_based`, `calculated`
  - Subfeature: `manual`, `status_based`
  - Key result: calculated from values only (R9)
- **R17** Calculated progress:
  - Objective: average of the progress of its key results and its child objectives. An objective with neither shows empty. [decision on average; sources only name the mode]
  - Initiative: average of the progress of its linked features. No linked features shows empty. [decision]
  - Feature: average of the progress of its subfeatures. [decision]
  - Children with empty progress are skipped, not counted as 0. [decision]
- **R17a** Status-based progress: each status in S03 carries a progress value from 0 to 100 (for example To do = 0, Done = 100). The item takes the value of its current status. [src:6]; the per-status value is a [decision]
- **R17b** Typing a value sets the mode to `manual`. A "use calculated" or "use status-based" action switches the mode back and recalculates at once. [decision]

### Health
- **R18** Health updates can be posted on objectives, key results, initiatives and features. [src:3][src:9]
- **R19** A health update has a status (`on_track`, `at_risk`, `off_track`) and an optional comment. Before the first update, health is `not_set`. [src:3][src:6]; the comment being optional is a [decision]
- **R20** Each update is stored with its author, time and the previous status. The latest update is the current health. Older updates stay in the history list and are never edited. [src:6]
- **R21** A user with edit rights on the item can post an update. Posting a manual update sets mode `manual`. [src:3][src:6]
- **R22** Health roll-up. A parent takes its health from its children when more than half of its children have a status other than `not_set`. [src:3][src:9]
  - Children that count as a parent's children: objective → key results and child objectives; initiative → linked features; feature → subfeatures. [decision on which links count; sources describe parents and children in general]
  - The roll-up result is the worst child status: any `off_track` gives `off_track`; otherwise any `at_risk` gives `at_risk`; otherwise `on_track`. [src:9]
  - Children with `not_set` and archived children are ignored when picking the worst status, but archived children are also left out of the count. [decision]
  - "More than half" means strictly more than 50 percent of the non-archived children. [src:9]
- **R23** A manual update overrides the roll-up and sets mode `manual`. The roll-up does not change the value while the mode is `manual`. [src:9]
- **R24** A "reset to calculated" action removes the override: mode becomes `calculated` and health is recomputed at once. If fewer than half the children have a status, health becomes `not_set`. [decision]
- **R25** A calculated health change is written to the history as an update with mode `calculated`, with no author and no comment. It does not trigger follower notifications (R27). [decision]
- **R26** Anyone who can see an item can follow it. A person who creates an item, or becomes its owner, follows it automatically. [src:9]; auto-follow is a [decision]
- **R27** When a manual health update is posted, every follower except the author is notified (in-app and email, per their notification settings). The message has the item name, new status, previous status, comment and a link. [decision; sources list followers and health but do not describe notifications]
- **R28** Roll-up runs again within 5 seconds of: a child's health change, a child added, removed, archived or linked, or a link change. [decision]

## User actions
- Maker or admin: create, edit, archive and delete objectives, key results and initiatives; link and unlink; set progress mode; post health updates; reset to calculated.
- Contributor: post health updates and comments on items they own; follow items. Cannot create or edit objectives, key results or initiatives. [decision]
- Viewer: read only; can follow items.
- Updating a key result's current value: its owner, a maker or an admin. [decision]

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R1)** Given objective A is parent of B, when a user sets B as A's parent, then the move is refused.
- **T2 (R6)** When a user creates a key result with no objective, then it is refused.
- **T3 (R9)** Given start 0, target 50, current 20, then progress is 40. Given start 8, target 5, current 6, then progress is 66.67 (shown as 67).
- **T4 (R9, R17)** Given a key result with current value above target, then its own progress shows above 100 in detail but counts as 100 in the objective's calculated progress.
- **T5 (R11, R12)** Given initiative I linked to objectives O1 and O2 and to feature F, when O1 is deleted, then I and F still exist and I stays linked to O2.
- **T6 (R17)** Given an objective with key results at 40 and 80 and a child objective at 60, then its calculated progress is 60.
- **T7 (R17a)** Given a feature in status-based mode and status "Done" = 100, when the status changes to Done, then progress is 100.
- **T8 (R17b)** Given a calculated feature, when a user types 30, then mode is `manual`. When the user picks "use calculated", then progress is recomputed.
- **T9 (R22)** Given an initiative with 4 linked features and only 2 have a status, then it shows `not_set`. Given 3 of 4 have a status (one `at_risk`, two `on_track`), then it shows `at_risk` and mode `calculated`.
- **T10 (R22)** Given the same initiative and a later `off_track` on one feature, then the initiative shows `off_track`.
- **T11 (R23)** Given a calculated `off_track` initiative, when a maker posts `on_track` with a comment, then it shows `on_track` in `manual` mode, and a later `off_track` child does not change it.
- **T12 (R24)** Given the initiative from T11, when a user resets to calculated, then it shows the rolled-up status.
- **T13 (R27, R25)** Given followers X and Y on an objective, when X posts a manual update, then Y is notified and X is not. When a roll-up changes health, then nobody is notified.
- **T14 (R5, R22)** Given an objective with 2 key results (one `off_track`) and 1 archived child objective, then the archived child is left out and the objective is `off_track`.

## Tier
- Key results, initiatives, health updates: paid tiers. [src:9][decision]
- Objective hierarchy: highest tier only. [src:9]

## Open questions
1. Should calculated progress be a weighted average (for example by effort) instead of a simple average?
2. Should a health roll-up also use the child objective's own owner comment? (v1: no comment on calculated updates.)
3. Should notifications go out on calculated changes, behind a workspace setting?
4. Is there a depth limit for the objective tree?
5. Should key results allow a check-in cadence and reminders to update the current value?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S07).
