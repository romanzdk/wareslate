# S06 Prioritization

## Scope
How Wareslate turns customer feedback and team judgment into numbers for sorting a backlog. It covers:
- Demand score, with per-account and per-cohort columns
- Criteria
- Computed fields
- Size

Related modules:
- Feedback entries and evidence (links from feedback to items, with a need level): [S04](S04-feedback.md)
- Accounts, contacts, cohorts: [S05](S05-customers.md)
- Field definitions and item types: [S03](S03-fields-statuses.md)
- How scores show in views: [S09](S09-views.md)

## Entities

| entity | field | type | required | remarks |
|---|---|---|---|---|
| evidence (from S04) | feedback_id, item_id | uuid | yes | link from a feedback entry to a product tree item |
| | need | enum `none \| low \| medium \| high \| unspecified` | yes | default `unspecified` |
| | need_set_at | timestamp | when set | |
| | need_source | enum `app \| public_roadmap \| slack \| api \| import` | when set | |
| | customer_key | derived | yes | see R4 |
| criterion | id, workspace_id, name | | yes | a field definition of kind `criterion` (S03) |
| | assigned_item_types | list | no | empty = unusable (R12) |
| criterion_value | criterion_id, item_id, value | int 0–5 or null | | |
| computed field | id, name, expression (AST, stored as JSON) | | yes | a field definition of kind `computed` |
| | assigned_item_types | list | no | empty = unusable (R16) |
| size | item_id, value | decimal ≥ 0 or null | | |
| | mode | enum `from_children \| manual` | yes | default `from_children` |
| score_cache | item_id, demand_total, demand_direct, computed_at | | | derived, rebuilt on change |

## Rules

### Demand score
- **R1** Need points: `none` = 0, `low` = 1, `medium` = 2, `high` = 3. Evidence with need `unspecified` counts 1. [src:1]
- **R2** Direct score of an item: for each distinct customer with at least one piece of evidence on that item, find that customer's *effective need* (R3) and add up the points. [src:1]
- **R3** Effective need of one customer on one item comes from their most recently changed evidence, looking only at values set in the app or API. A value from the public roadmap or Slack is used only when the customer has no earlier value on that item. A later public roadmap or Slack value never replaces an existing one. [src:1]
- **R4** Customer key: the contact of the feedback entry if it has one, otherwise its account. An entry with neither counts as its own anonymous customer, so each such entry counts separately. [decision]
- **R5** Total score of an item = its direct score + the total scores of its child items (product → module → feature → slice). One customer can count again at a parent if they gave feedback on two different children. [src:1][src:6]; no dedupe across children is a [decision]
- **R6** Item types with a demand score: product, module, feature, slice. [src:6] The same calculation may be switched on for other item types later. [decision]
- **R7** Account column for account C on item X: the sum of points from C's contacts and account-level entries on X and on X's children, capped at **3**. [src:1] *(The cap applies only to the column. The total score in R2/R5 is not capped by account.)* [decision; sources describe the cap only for account columns]
- **R8** Cohort column for cohort S on item X: the same sum over customers in S, **with no cap**. [src:1][src:5]
- **R9** Need `none` (0) still records the request. It appears in the item's evidence list and in the customer count, but adds 0 points. [src:1]
- **R10** Scores update within 5 seconds of any change to evidence, need, the customer on a feedback entry, account membership, cohort membership or a product tree move. [decision]

### Criteria
- **R11** A criterion value is a whole number from 0 to 5, or empty. [src:2 title "zero to five"; range enforcement is a decision]
- **R12** A new criterion starts unlinked to any item kind, and stays unusable until it is linked to one or more kinds. [src:2]
- **R13** Creating a criterion from a table view adds it as a column of that view. Creating it in settings adds it to no view. [src:2]
- **R14** Deleting a criterion removes it, and every value it holds, from the workspace for good. The UI asks for confirmation and shows how many values will be lost. [src:2; confirmation is a decision]

### Computed fields
- **R15** A computed field may use numbers, `+ − × ÷`, brackets, and these inputs: demand score (total), size, criteria, and number custom fields. No functions in v1. [src:3]
- **R16** A computed field needs item types to be usable. If an input field is not assigned to one of its item types, show a warning, and that input reads as 0 for that type. [src:3; read-as-0 is a decision]
- **R17** An empty input reads as 0. [src:3]
- **R18** Division by zero gives an empty result, shown as "—" and sorted last. [decision]
- **R19** Results are decimals, shown with up to 2 decimal places. Sorting uses the full value. [decision]
- **R20** The editor checks the expression as you type and saves only valid expressions. A computed field cannot reference another computed field in v1, which avoids cycles. [src:3 shows validation; no-nesting is a decision]
- **R21** Adding a computed column to a view also adds the columns of its inputs. Removing the computed column leaves them in place. [src:3]

### Size
- **R22** Item types with a size: product, module, feature, slice, release, bet, objective. [src:4]
- **R23** New items start in `from_children` mode. The value is the **sum** of the children's size, with empty children counting as 0. An item with no children in `from_children` mode shows empty. [src:4 default mode; sum is a decision]
- **R24** Typing a value switches the item to `manual`, and children are then ignored. A "reset to from children" action switches it back and recalculates at once. [src:4]
- **R25** The size input suggests Fibonacci values (1, 2, 3, 5, 8, 13, 21) and also accepts any number ≥ 0. [src:4]
- **R26** In the Jira integration, size maps to story points (S12). [src:4]

## User actions
- Editor or admin: set need on evidence, create, edit and delete criteria and computed fields, set criterion and size values.
- Collaborator: set need on evidence they create (S04). Cannot edit criteria, computed fields or size.
- Reader: read only.
- Any role that can see a view can sort and filter by demand score, account, cohort, criterion, computed field and size columns.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R1, R2)** Given feature F with evidence from contacts A (`high`), B (`unspecified`) and C (`none`), when the score is computed, then F's direct score is 3 + 1 + 0 = 4.
- **T2 (R3)** Given contact A gives F `low` in the app on day 1 and `high` in the app on day 2, then A adds 3 points. Given instead that A's day-2 value came from the public roadmap, then A adds 1 point.
- **T3 (R5)** Given module K with features F1 (direct 4) and F2 (direct 2), and K's own direct score is 1, then K's total is 7.
- **T4 (R7)** Given account C has 3 contacts who each give F `high`, then C's column on F shows 3. F's direct score includes 9 points.
- **T5 (R8)** Given cohort S holds the same 3 contacts, then S's column on F shows 9.
- **T6 (R4)** Given two feedback entries with no contact and no account are linked to F with need `unspecified`, then F gets 2 points.
- **T7 (R17, R18)** Given computed field `(demand score * Impact) / Size`, with item X having demand score 10, Impact empty and Size 2, then X's result is 0. Given item Y has Size 0, then Y's result is empty and Y sorts last.
- **T8 (R23, R24)** Given feature F (`from_children`) with slices of size 3, empty and 5, then F shows 8. When a user types 13 on F, then F is manual and shows 13. When the slice with 5 changes to 8, F still shows 13. When the user resets F to from children, F shows 11.
- **T9 (R12, R16)** Given a new criterion or computed field with no assigned item types, then it does not appear as a column option on any view.

## Tier
Demand score, criteria, computed fields and size: all tiers. Cohorts: paid tiers. [decision]

## Open questions
1. Should the total score count each customer once across the whole subtree, instead of R5? Make it a workspace setting later if users ask.
2. Computed field functions (min, max, if, round): add in v2?
3. Should the anonymous-entry rule (R4) be configurable, for example treating all anonymous entries as one customer?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S06).
