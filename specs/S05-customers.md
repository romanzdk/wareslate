# S05 Customers

## Scope
The people and accounts that feedback comes from, and how they are grouped. It covers:
- Companies and customer users (people outside the workspace)
- User ↔ company link, custom company fields
- Aggregated customer fields on hierarchy items
- Segments (manual user segments, dynamic company segments)
- Duplicate merge, CSV import and export, matching by email or external id

Related modules:
- Feedback items and insights that point at customers: [S04](S04-feedback-insights.md)
- How companies and segments feed scores: [S06](S06-prioritization.md)
- Custom field definitions: [S03](S03-fields-statuses.md)
- Hierarchy items that carry aggregated fields: [S02](S02-hierarchy.md)
- Boards that show customer columns: [S09](S09-boards.md)
- CRM and other sync: [S12](S12-integrations.md). API: [S13](S13-api.md). Roles: [S01](S01-workspace-roles.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| company | id, workspace_id | uuid | yes | |
| | name | string | yes | not unique |
| | description | rich text | no | |
| | domain | string | no | for matching users (R9) |
| | external_id | string | no | unique per workspace when set (R25) |
| | archived | boolean | yes | default false |
| | custom fields | map field_id → value | no | R12–R15 |
| | source | `{system, record_id}` | no | set by integrations and API |
| customer_user | id, workspace_id | uuid | yes | not a workspace member (S01) |
| | name | string | no | |
| | email | string | no | unique per workspace, case-insensitive, when set |
| | external_id | string | no | unique per workspace when set |
| | company_id | company | no | at most one company (R8) |
| | description | rich text | no | |
| | archived | boolean | yes | default false |
| | source | `{system, record_id}` | no | |
| company_field | id, name, type `number \| text \| boolean \| date \| url \| single_select` | | yes | a custom field definition for companies (S03) |
| aggregated_field | id, name, source_field_id, function `sum \| avg \| min \| max \| count`, item_types[] | | | R17–R21 |
| segment | id, name, kind `users \| companies` | | yes | R22 |
| | members | list of customer_user (manual) | | kind users |
| | rule | `{op: AND\|OR, conditions[]}` | | kind companies, R24 |
| condition | field_id, operator, value | | | R24 |
| merge_log | id, kept_id, removed_id, merged_by, merged_at, summary | | | R31 |

## Rules

### Companies and users
- **R1** A company is an account (customer or prospect) with a name, a description, custom fields and an archived flag. [src:1][src:3]
- **R2** A customer user is a person who gives feedback. They are not workspace members and cannot log in to the workspace. [src:2][src:3]
- **R3** Customer users and companies are full records. Each can be created, edited, archived and listed on its own board. [src:3][src:4]
- **R4** Both can be created by hand, by CSV (R28), by API, or by an integration. Integrations set `source`. [src:1][src:2]
- **R5** Archiving hides a company or user from pickers and default lists. Their feedback and scores stay. Archived records can be restored. [src:1][src:2]
- **R6** Deleting a company or user asks for confirmation and shows how many feedback items point at them. Those items keep their text and lose the customer link. Deleting a company does not delete its users. They become users without a company. [decision]
- **R7** Bulk delete is available by API and in the list for admins. It uses the same rules as R6. [src:3][decision on list]
- **R8** A user belongs to at most one company. Setting the company moves the user. A company has many users. [src:1][src:2]
- **R9** Domain matching: when a user has an email and no company, and a company has a domain equal to the email's domain, the user is put in that company. This runs when the user is created or the email changes. It never moves a user who already has a company. Free-mail domains (gmail.com, outlook.com and similar) are ignored. [src:1][decision on free-mail and no-move]
- **R10** A company without a domain is never matched by R9. If two companies share a domain, no match is made, and the user stays without a company. [decision]
- **R11** Name matching is not used for assignment. [decision]

### Custom company fields
- **R12** An admin or maker can add custom fields to companies: number, text, boolean, date, url, single select. [src:1][src:3]
- **R13** Typical uses are revenue, plan, industry, region, renewal date. [src:1]
- **R14** Changing a field type is not allowed once values exist. Deleting a field deletes its values after confirmation. [decision]
- **R15** A value can be set by hand, import, API or integration. The record keeps who set it last (member id or integration name). [decision]
- **R16** Custom fields on customer users are not part of v1. [decision]

### Aggregated customer fields on items
- **R17** An aggregated field shows, on a hierarchy item, a value built from a number field of the companies that requested the item. [src:5]
- **R18** "Requested the item" means the company has at least one insight on the item or on its children (S04, S02), either from a note linked to the company, or from a note linked to one of its users. [src:5][decision on children]
- **R19** Each company counts once per item, however many insights it has. [decision]
- **R20** Functions: sum, average, minimum, maximum, count of companies. Companies with an empty value are skipped for average, minimum and maximum, and count as 0 for sum. Example: total revenue of requesting companies. [src:5][decision on functions]
- **R21** Aggregated fields can be shown as columns on boards (S09), sorted and filtered, and used as inputs to formulas (S06 R15). They are read-only. They recalculate within 5 seconds of a change to an insight, company value, user-company link or hierarchy move. [src:5][decision on timing]

### Segments
- **R22** A segment is a saved group of customers, either of users (manual) or of companies (dynamic). [src:6]
- **R23** Manual user segment: a member adds and removes users by hand or by CSV of emails. A user can be in many segments. [src:6]
- **R24** Dynamic company segment: it has a rule with conditions joined by AND or OR (one operator for the whole rule). A condition checks one company field. Number fields: =, ≠, >, ≥, <, ≤, is empty, is not empty. Text fields: is, is not, contains, does not contain, is empty, is not empty. [src:6][decision on exact operator list]
- **R25a** Membership of a dynamic segment is recalculated when a company value changes, a company is created or archived. Members are never added by hand. [src:6][decision on timing]
- **R25b** Archived companies are not in dynamic segments. [decision]
- **R26a** Nested groups of conditions are not in v1. [decision]
- **R26b** A segment shows how many members it has. Segment score columns follow S06 R8. [src:6]
- **R26c** Deleting a segment never deletes companies or users. Boards that used it lose its column. [decision]
- **R26d** Segment names are unique per workspace. [decision]

### Matching and duplicates
- **R25** `external_id` is unique per workspace for companies, and separately for users. Email is unique for users, ignoring case. A create or update that breaks this returns a conflict error naming the existing record. [src:3][decision]
- **R27** Matching order for feedback and import: first external id, then email (users only). If a match is found, the existing record is used. If no match, a new record is created. For a company, a name without an external id never matches. [src:3][src:4][decision on order]
- **R28** CSV import of companies: columns for name, external id, domain, description and one column per custom field. Rows are matched by external id (R27). A matched row updates only the columns that are present and not empty. [src:3][decision on details]
- **R29** CSV import of users: name, email, external id, company (by company external id or exact company name). A company name that matches two companies or none is an error for that row. [decision]
- **R30** Import shows a report of rows created, updated and failed, with reasons, before anything is saved. Up to 10,000 rows per file. [decision]
- **R31** Merge duplicates: an admin or maker picks two or more records of the same kind and one to keep. Their data moves to the kept record, and the others are removed. [src:7]
- **R32** On merge, all feedback links, insights, segment memberships and, for companies, users move to the kept record. Empty fields on the kept record are filled from the others, in the order picked. Non-empty fields are not overwritten. Domains and emails of removed records are kept as extra matching aliases on the kept record. [src:7][decision on field rules and aliases]
- **R33** Merged ids still work: API requests for a removed id get a redirect to the kept record. [src:3]
- **R34** Merge is recorded in a merge log. It cannot be undone in v1. Scores are recalculated after the merge, and the kept company counts once per item (R19, S06 R4). [decision]
- **R35** Two users from different companies can be merged. The kept user's company wins. [decision]

### Export
- **R36** Companies and users lists export to CSV with the current filter. Columns: all standard fields, custom fields, segments (semicolon list), feedback count. Export of a segment exports its members. [src:3][src:4][decision on columns]

## User actions
- Admin and maker: create, edit, archive, delete, import, export, merge, manage company fields, aggregated fields and segments.
- Contributor: create customer users and companies while adding feedback (S04), edit users and companies they created. Cannot merge, delete, import, or edit fields and segments.
- Viewer: read only.
- Customer users: have no access to the workspace. They only appear as a customer on feedback.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R8)** Given user U in company A, when a member sets U's company to B, then U is in B only.
- **T2 (R9)** Given company "Acme" with domain acme.com, when user ann@acme.com is created with no company, then Ann is in Acme. Given user bob@gmail.com, then Bob stays without a company.
- **T3 (R9)** Given user cy@acme.com already in company Zed, when the domain rule runs, then Cy stays in Zed.
- **T4 (R10)** Given two companies with domain dup.com, when a user dan@dup.com is created, then Dan has no company.
- **T5 (R17–R20)** Given feature F with insights from companies A (revenue 100, two users), B (revenue 50) and C (revenue empty), when "total revenue" (sum) is shown, then F shows 150, and the company count shows 3.
- **T6 (R20)** Given the same data with average, then F shows 75.
- **T7 (R18)** Given an insight on a subfeature of F from company D (revenue 20), then F's total revenue includes 20.
- **T8 (R21)** Given company A's revenue changes from 100 to 200, then F's total revenue updates within 5 seconds.
- **T9 (R24)** Given the rule "revenue > 100 AND plan is Pro", when company A has revenue 200 and plan Pro, then A is a member. When A's plan becomes Free, then A leaves within 5 seconds.
- **T10 (R24)** Given the rule "revenue > 100 OR plan contains Pro", then a company with revenue 10 and plan "Pro Plus" is a member.
- **T11 (R23)** Given a manual segment, when a member adds the same user twice, then the user is in it once.
- **T12 (R25)** Given user ann@acme.com exists, when another user is created with ANN@acme.com, then a conflict error names the existing user.
- **T13 (R27, R28)** Given a CSV row with external id E1 matching company A and a new revenue value but an empty plan, then A's revenue changes and A's plan stays.
- **T14 (R29)** Given a user row whose company name matches two companies, then that row fails with a reason and others import.
- **T15 (R31, R32)** Given duplicate companies A (keep) and B, B has 2 users and 3 feedback items, when merged, then A has those users and items, B no longer exists, and B's domain still matches A.
- **T16 (R33)** Given B was merged into A, when the API reads B's id, then it gets a redirect to A.
- **T17 (R34)** Given A and B both gave F "critical" and were merged, then the merged company counts once on F.
- **T18 (R6)** Given a company with 4 feedback items and 3 users, when it is deleted, then the confirmation says 4 items, the items remain without a customer, and the users remain without a company.
- **T19 (R36)** Given a dynamic segment with 5 members, when exported, then the CSV has 5 rows.

## Tier
Companies, users, custom fields, import, export, merge: all tiers. Aggregated fields and segments: paid tiers. CRM sync follows S12. [decision]

## Open questions
1. Should one user be allowed in many companies (consultants)?
2. Should merge be undoable for a short time (for example 7 days)?
3. Should custom fields on users come in v2?
4. Should aggregated fields be able to count insights instead of companies, or weight by importance?
5. Should dynamic segments allow nested AND/OR groups?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S05).
