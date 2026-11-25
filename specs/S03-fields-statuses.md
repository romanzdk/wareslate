# S03 Fields and statuses

## Scope
How a workspace adds its own data to items, and how work moves through states. It covers:
- Custom field types and their values
- Assigning fields to item types
- Bulk edit
- Custom statuses per item type, with categories
- Required fields before a status change
- Tags and the teams field

Related modules:
- Items and item types: [S02](S02-product-tree.md)
- Roles and teams: [S01](S01-workspace-roles.md)
- Computed fields and criteria (they use the field system here): [S06](S06-prioritization.md)
- Release and bet statuses: [S08](S08-releases.md), [S07](S07-strategy.md)
- Field columns on views: [S09](S09-views.md)
- API value shapes: [S13](S13-api.md)

## Entities

| entity | field | type | required | details |
|---|---|---|---|---|
| field_definition | id, workspace_id | uuid | yes | |
| | name | text, 1–255 chars | yes | unique per workspace, case-insensitive |
| | kind | enum `text \| number \| single_select \| multi_select \| member \| date \| rich_text \| url \| computed \| criterion` | yes | fixed after creation |
| | assigned_item_types | list of item types | no | empty = unusable (R3) |
| | options | list of field_option | select kinds only | |
| | computed_id | uuid | computed kind only | points to a S06 computed |
| field_option | id, name | | yes | name unique within the field |
| | color | enum (fixed palette) | yes | default `gray` |
| | position | integer | yes | |
| field_value | item_id, field_id | | yes | one value per item and field |
| | value | by kind, see R6–R14 | | null = empty |
| | updated_at, updated_by, source | | | source: `app \| api \| import \| bulk` |
| status_definition | id, item_type_group | | yes | `item_type_group` is one status list (R25) |
| | name | text, 1–60 chars | yes | unique within its list |
| | category | enum `not_started \| in_progress \| done \| cancelled` | yes | |
| | color | enum (fixed palette) | yes | |
| | position | integer | yes | |
| | progress_percent | integer 0–100 or null | no | feature list only (R30) |
| | is_default | boolean | yes | exactly one per list (R28) |
| lifecycle_rule | id, name, enabled | | yes | required fields, see R34–R42 |
| | product_ids | list | yes | min 1 |
| | transitions | list of `{status_id, required_field_ids[]}` | yes | |
| tag | id, name | | yes | workspace-wide, unique case-insensitive |
| team (from S01) | id, name | | | used by the teams field (R44) |
| bulk_edit_job | id, item_ids, operations, result per item | | | see R18–R23 |

## Rules

### Custom field types
- **R1** Only admins and editors may create, change or remove custom fields. [src:1]
- **R2** Supported kinds: text, number, single select, multi select, member, date, rich text, URL, and computed (R15). [src:1][src:4]
- **R3** A new field is assigned to no item type. Until it is tied to an item type it cannot be used. Item types without the field never show it, and no value can be written for them. [src:1]
- **R4** Assigning a field to more item types adds it to those items with empty values. Removing an assignment hides the field on that type, but keeps the stored values. Assigning it again shows the old values. [decision]
- **R5** The kind of a field cannot change after creation. [decision]

### Field values
- **R6** Text: a string up to 255 characters, plain text. [src:4]; the limit is a [decision] copied from the item name limit
- **R7** Number: integer or decimal, positive or negative. Stored as a decimal with up to 6 decimal places. [src:4]; precision is a [decision]
- **R8** Single select: one option of the field, or empty. [src:4] The value read back holds the option's id, name and color. [src:4]
- **R9** Multi select: a set of options of the field, without repeats. Reading gives the full option objects. [src:4]
- **R10** Member: one workspace member. Reading gives the member id and email. Writing accepts a member id or an email. [src:4]
- **R11** Date: a calendar date without time (`YYYY-MM-DD`). [src:4] Custom dates are not drawn as milestones on timeline views in v1. [src:1]
- **R12** Rich text: sanitized HTML, up to 1 MiB. [src:4][src:6]
- **R13** URL: a string that must start with `http://` or `https://`. It shows as a clickable link. A text field whose value starts with one of those also shows a link. [src:1][src:4]
- **R14** On writes (API and import), a select value may be sent as an option id or an option name, and a member as an id or an email address. Ids are preferred, because names can change. A name that matches no option, or more than one, is an error. [src:4]
- **R15** Computed field: a read-only field whose value comes from a S06 computed. Users and the API cannot set it, and it is not part of bulk edit. It is shown as `{value, mode: calculated}`. The computed is chosen when the field is created. [src:6]; wiring to S06 is a [decision]
- **R16** Select options: admins and editors can add, rename, recolor and reorder options. Order is workspace-wide. A field has at most 1,000 options. Deleting an option that items use needs a replacement option or "clear the value", chosen in the dialog. [src:6]; the delete flow and the limit are [decision]
- **R17** Deleting a field removes it and all its values for good. The dialog shows how many values will be lost and asks for confirmation. Computed fields that use the field show an error until fixed (S06 R16). [decision]

### Bulk edit
- **R18** Bulk edit changes one or more fields on up to 1,000 selected items in one action. [src:1]
- **R19** All field kinds can be bulk edited except rich text. Computed fields cannot be bulk edited either (R15). [src:1]
- **R20** Operations: set, clear, and for multi-value fields (multi select, tags, teams) also add items and remove items. [src:6]
- **R21** Selecting more than 1,000 items is refused before anything changes. [src:1]; refusing up front is a [decision]
- **R22** Each item is checked on its own. An item fails if the field is not assigned to its type (R3), the value is invalid, or a required-field rule blocks a status change (R39). Failed items are skipped. The others are saved. The result lists each failed item with a reason. [decision]
- **R23** A bulk edit counts as one history entry per item, with source `bulk`. [decision]

### Statuses
- **R24** Item types with a status: feature, slice, objective, bet, key result and release. [src:2]
- **R25** Each item type has one status list. Feature and slice share one list. [src:2]
- **R26** Admins can add, rename, recolor, reorder and delete statuses. Reordering applies workspace-wide. There is no fixed limit on how many statuses a list can have. [src:2]; a soft limit of 50 per list is a [decision]
- **R27** Every status belongs to exactly one category: not started, in progress, done or cancelled. Categories let reports and roll-ups treat custom names the same way. [decision; research does not describe categories]
- **R28** Every list needs a status in `not_started` and a status in `done`. Exactly one status is the default for new items, and it must be in `not_started`. [decision]
- **R29** Seed lists for a new workspace: not started = "New", "Planned"; in progress = "In progress"; done = "Done"; cancelled = "Won't do". Default = "New". [decision]
- **R30** In the feature list, a status can have a progress percentage (0–100). Items with progress mode "status based" (S07) take it from their status. Other lists have no percentage. [src:2][src:5]
- **R31** Deleting a status that items use needs a replacement status in the same list, picked in the confirm dialog. All affected items move to it in one step. A status that no item uses is deleted after confirmation. The default status and the last status of a category cannot be deleted. [src:2]; the last two limits are [decision]
- **R32** Changing an item's status is allowed from any status to any other. There is no fixed workflow order. [decision]
- **R33** Some integrations only sync the default bet statuses. Custom bet statuses are not synced to Jira, and restoring the default names restores the sync. [src:2] Mapping details belong to [S12](S12-integrations.md).

### Required fields before a status change
- **R34** An admin can create a lifecycle rule: for each target status, a list of fields that must be filled before an item moves to that status. One rule may cover several statuses. [src:3]
- **R35** A rule is tied to one or more products and applies to that product and everything below it. If two rules apply to the same item, the fields of both are required. [src:3]; combining rules is a [decision]
- **R36** In v1 the rule applies to features. The engine is built so other item types can be added. [src:3]; slices are left for later as a [decision]
- **R37** Any of these can be required: name, description, owner, tags, teams, custom fields, tasks, date range, size, check-in and criteria. A field counts as filled when its value is not empty. For tasks, at least one task must exist. [src:3]; the "filled" definition is a [decision]
- **R38** When a user moves an item to a status with required fields, the status change waits. The user sees the missing fields, fills them, and the change completes together with the new values. The user can cancel instead, and the status stays as it was. [src:3]
- **R39** The same check runs for bulk edit, API updates and imports. The change for that item is refused with the error `status.required_fields_missing` and a list of the missing field ids. [decision; research describes only the UI]
- **R40** Rules can be edited, duplicated, disabled and enabled. A disabled rule is ignored. Deleting a rule asks for confirmation. [src:3]; the confirmation is a [decision]
- **R41** Rules do not look back. Items already in the target status stay there even if they miss fields. [decision]
- **R42** Only admins can manage lifecycle rules. [src:3] They are a top-tier feature. [src:3]

### Tags and teams
- **R43** Tags are free labels. A tag is created by typing a new name in the tag picker. Names are unique in the workspace ignoring upper/lower case, up to 60 characters. An item can have many tags. Renaming or deleting a tag changes or removes it everywhere. [src:6]; the rules for names are a [decision]
- **R44** The teams field is a set of teams from S01. It can hold many teams. Writing accepts a team id or a team name. A name that matches no team is an error. [src:4]
- **R45** Both tags and teams are available on every item type and can be filtered, grouped and bulk edited. [decision]

## User actions
- Admin: everything, including statuses, lifecycle rules, deleting fields and tags.
- Editor: create, edit and delete custom fields and select options; set field values; use bulk edit; create tags.
- Collaborator: sees field values. Cannot edit items (S02).
- Reader: read only.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R3)** Given a new field with no item types, then it appears in no item panel and an API write for it fails.
- **T2 (R4)** Given a field with values on features, when it is removed from features and later assigned again, then the old values show again.
- **T3 (R7)** Given a number field, when a user saves "-12.5", then it is stored as -12.5. Given "abc", then it is refused.
- **T4 (R13)** Given a URL field, when a user saves "example.com", then it is refused. Given "https://example.com", then it shows as a link.
- **T5 (R14)** Given a single select with options "High" and "Low", when the API sends `{"name": "High"}`, then "High" is set. When it sends `{"name": "Medium"}`, then it is an error.
- **T6 (R10, R14)** Given a member field, when the API sends an email of a workspace member, then the member is set. Given an unknown email, then it is an error.
- **T7 (R15)** Given a computed field, when a user or the API tries to set its value, then it is refused.
- **T8 (R16)** Given 12 items use option "Blocked", when an admin deletes it and picks "clear the value", then the 12 values are empty.
- **T9 (R18, R21)** Given 1,001 selected items, when the user starts a bulk edit, then it is refused and nothing changes.
- **T10 (R19)** Given a rich text field, then it is not offered in bulk edit.
- **T11 (R20)** Given a multi select with {A}, when bulk "add B" runs, then the value is {A, B}. When "remove A" runs, then it is {B}.
- **T12 (R22)** Given a bulk edit of 10 items where 2 are slices and the field is not assigned to slices, then 8 are saved and 2 are listed as failed.
- **T13 (R25)** Given a status "In review" is added to the feature list, then it is also offered on slices.
- **T14 (R28, R31)** Given the default status "New", when an admin tries to delete it, then it is refused.
- **T15 (R31)** Given 5 features are in "Planned", when an admin deletes it and picks "New" as replacement, then all 5 are in "New".
- **T16 (R38)** Given a rule requiring owner and date range for "In progress" on product P, when an editor moves a feature under P to "In progress" without them, then the user is asked for both. After they are filled, the feature is in "In progress". If they cancel, the status stays.
- **T17 (R39)** Given the same rule, when the API sets that status without an owner, then it returns `status.required_fields_missing` listing the owner field.
- **T18 (R35)** Given rule A requires owner and rule B requires tags for the same status and product, then both are required.
- **T19 (R41)** Given a feature already in "In progress", when a rule is added, then it stays in that status.
- **T20 (R43)** Given a tag "Mobile", when a user adds "mobile", then the existing tag is used.
- **T21 (R44)** Given the teams field, when the API sends `[{"id": T1}, {"name": "Design"}]`, then both teams are set.

## Tier
Custom fields, tags, teams field, bulk edit: all tiers. [src:1] Custom statuses: paid tiers. [src:2] Required fields before a status change: top tier only. [src:3]

## Open questions
1. Should the category list (R27) have more values, for example "on hold"?
2. Should select fields allow options to be created on the fly by editors, as tags can?
3. Release statuses are fixed in some sources and customizable in another (R24). What should S08 do?
4. Should required fields also cover slices and other item types in v1?
5. Should deleting a field be soft (restorable for 30 days)?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S03).
