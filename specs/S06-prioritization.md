# S06 Prioritization

## Scope
How Wareslate turns customer feedback and team judgment into numbers for sorting a backlog. It covers:
- Customer impact score, with per-company and per-segment views of it
- Drivers
- Formula fields
- Effort

Related modules:
- Insights (note → item links with importance): [S04](S04-feedback-insights.md)
- Companies, users, segments: [S05](S05-customers.md)
- Field definitions and item types: [S03](S03-fields-statuses.md)
- How scores show on boards: [S09](S09-boards.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| insight (from S04) | note_id, item_id | uuid | yes | link from a feedback note to a hierarchy item |
| | importance | enum `not_important \| nice_to_have \| important \| critical \| null` | no | null = not set |
| | importance_set_at | timestamp | when set | |
| | importance_source | enum `app \| portal \| slack \| api \| import` | when set | |
| | customer_key | derived | yes | see R4 |
| driver | id, workspace_id, name | | yes | a field definition of kind `driver` (S03) |
| | assigned_item_types | list | no | empty = unusable (R12) |
| driver_value | driver_id, item_id, value | int 0–5 or null | | |
| formula | id, name, expression (AST, stored as JSON) | | yes | a field definition of kind `formula` |
| | assigned_item_types | list | no | empty = unusable (R16) |
| effort | item_id, value | decimal ≥ 0 or null | | |
| | mode | enum `rollup \| manual` | yes | default `rollup` |
| score_cache | item_id, impact_total, impact_direct, computed_at | | | derived, rebuilt on change |

## Rules

### Customer impact score
- **R1** Importance points: not_important = 0, nice_to_have = 1, important = 2, critical = 3. An insight with no importance set counts 1. [src:1]
- **R2** Direct score of an item: for each distinct customer with at least one insight on that item, take that customer's *effective importance* (R3) and add up the points. [src:1]
- **R3** Effective importance of one customer on one item is the importance on their most recently changed insight, among values set in the app or API. A value from the portal or Slack is used only when the customer has no earlier value on that item. A later portal or Slack value never replaces an existing one. [src:1]
- **R4** Customer key: the note's user if it has one, otherwise its company. A note with neither counts as its own anonymous customer, so each such note counts separately. [decision]
- **R5** Total score of an item = its direct score + the total scores of its child items (product → component → feature → subfeature). One customer can count again at a parent if they gave feedback on two different children. [src:1][src:6]; no dedupe across children is a [decision]
- **R6** Item types with impact score: product, component, feature, subfeature. [src:6] The same calculation may be switched on for other item types later. [decision]
- **R7** Company column for company C on item X: the sum of points from C's users and C-level notes on X and on X's children, capped at **3**. [src:1] *(The cap applies only to the column. The total score in R2/R5 is not capped by company.)* [decision; sources describe the cap only for company columns]
- **R8** Segment column for segment S on item X: the same sum over customers in S, **with no cap**. [src:1][src:5]
- **R9** "Not important" (0) still records the request. It appears in the item's insight list and in the customer count, but adds 0 points. [src:1]
- **R10** Scores update within 5 seconds of any change to an insight, importance, note customer, company membership, segment membership or hierarchy move. [decision]

### Drivers
- **R11** A driver is an integer score from 0 to 5, or empty. [src:2 title "zero to five"; range enforcement is a decision]
- **R12** A driver must be assigned to at least one item type before it can be used. A new driver has no types. [src:2]
- **R13** Creating a driver from a grid board adds it as a column on that board. Creating it in settings adds it to no board. [src:2]
- **R14** Deleting a driver removes it and all its values from the workspace for good. The UI asks for confirmation and shows how many values will be lost. [src:2; confirmation is a decision]

### Formula fields
- **R15** Allowed in a formula: numbers, `+ − × ÷`, brackets, and these inputs: impact score (total), effort, drivers, number custom fields. No functions in v1. [src:3]
- **R16** A formula must be assigned to item types to be usable. If an input field is not assigned to one of the formula's item types, show a warning, and that input reads as 0 for that type. [src:3; read-as-0 is a decision]
- **R17** An empty input reads as 0. [src:3]
- **R18** Division by zero gives an empty result, shown as "—" and sorted last. [decision]
- **R19** Results are decimals, shown with up to 2 decimal places. Sorting uses the full value. [decision]
- **R20** The editor checks the expression as you type and saves only valid expressions. A formula cannot reference another formula in v1, which avoids cycles. [src:3 shows validation; no-nesting is a decision]
- **R21** Adding a formula column to a board also adds its input columns. Removing the formula column leaves them in place. [src:3]

### Effort
- **R22** Item types with effort: product, component, feature, subfeature, release, initiative, objective. [src:4]
- **R23** New items start in `rollup` mode. The value is the **sum** of the children's effort, with empty children counting as 0. An item with no children and mode `rollup` shows empty. [src:4 rollup default; sum is a decision]
- **R24** Typing a value switches the item to `manual`. It then ignores children. A "reset to rollup" action switches it back and recalculates at once. [src:4]
- **R25** The effort input suggests Fibonacci values (1, 2, 3, 5, 8, 13, 21) and also accepts any number ≥ 0. [src:4]
- **R26** In the Jira integration, effort maps to story points (S12). [src:4]

## User actions
- Maker or admin: set importance on insights, create, edit and delete drivers and formulas, set driver and effort values.
- Contributor: set importance on insights they create (S04). Cannot edit drivers, formulas or effort.
- Viewer: read only.
- Any role that can see a board can sort and filter by impact score, company, segment, driver, formula and effort columns.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R1, R2)** Given feature F with insights from users A (critical), B (no importance) and C (not important), when the score is computed, then F's direct score is 3 + 1 + 0 = 4.
- **T2 (R3)** Given user A gives F "nice_to_have" in the app on day 1 and "critical" in the app on day 2, then A adds 3 points. Given instead that A's day-2 value came from the portal, then A adds 1 point.
- **T3 (R5)** Given component K with features F1 (direct 4) and F2 (direct 2), and K's own direct score is 1, then K's total is 7.
- **T4 (R7)** Given company C has 3 users who each give F "critical", then C's column on F shows 3. F's direct score includes 9 points.
- **T5 (R8)** Given segment S holds the same 3 users, then S's column on F shows 9.
- **T6 (R4)** Given two notes with no user and no company are linked to F with no importance, then F gets 2 points.
- **T7 (R17, R18)** Given formula `(impact score * Impact) / Effort`, with item X having impact score 10, Impact empty and Effort 2, then X's result is 0. Given item Y has Effort 0, then Y's result is empty and Y sorts last.
- **T8 (R23, R24)** Given feature F (rollup) with subfeatures of effort 3, empty and 5, then F shows 8. When a user types 13 on F, then F is manual and shows 13. When the subfeature with 5 changes to 8, F still shows 13. When the user resets F to rollup, F shows 11.
- **T9 (R12, R16)** Given a new driver or formula with no assigned item types, then it does not appear as a column option on any board.

## Tier
Impact score, drivers, formulas and effort: all tiers. Segments: paid tiers. [decision]

## Open questions
1. Should the total score dedupe customers across the subtree (each customer counts once per parent) instead of R5? Make it a workspace setting later if users ask.
2. Formula functions (min, max, if, round): add in v2 of formulas?
3. Should the anonymous-note rule (R4) be configurable, for example treating all anonymous notes as one customer?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S06).
