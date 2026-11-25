# S05 Customers

## Scope
The people and organizations that feedback comes from, and how they are grouped. It covers:
- Accounts and contacts (people outside the workspace)
- Contact ↔ account link, custom account fields
- Account totals on product tree items
- Cohorts (hand-picked contact cohorts, rule-based account cohorts)
- Duplicate merge, CSV import and export, matching by email or external id

Related modules:
- Feedback entries and evidence that point at customers: [S04](S04-feedback.md)
- How accounts and cohorts feed scores: [S06](S06-prioritization.md)
- Custom field definitions: [S03](S03-fields-statuses.md)
- Product tree items that carry account totals: [S02](S02-product-tree.md)
- Views that show customer columns: [S09](S09-views.md)
- CRM and other sync: [S12](S12-integrations.md). API: [S13](S13-api.md). Roles: [S01](S01-workspace-roles.md)

## Entities

| entity | field | type | required | remarks |
|---|---|---|---|---|
| account | id, workspace_id | uuid | yes | |
| | name | string | yes | not unique |
| | description | rich text | no | |
| | domain | string | no | for matching contacts (R9) |
| | external_id | string | no | unique per workspace when set (R25) |
| | archived | boolean | yes | default false |
| | custom fields | map field_id → value | no | R12–R15 |
| | origin | `{tool, external_id}` | no | set by integrations and API |
| contact | id, workspace_id | uuid | yes | not a workspace member (S01) |
| | name | string | no | |
| | email | string | no | unique per workspace, case-insensitive, when set |
| | external_id | string | no | unique per workspace when set |
| | account_id | account | no | at most one account (R8) |
| | description | rich text | no | |
| | archived | boolean | yes | default false |
| | origin | `{tool, external_id}` | no | |
| account_field | id, name, type `number \| text \| boolean \| date \| url \| single_select` | | yes | a custom field definition for accounts (S03) |
| account_total | id, name, source_field_id, function `sum \| avg \| min \| max \| count`, item_types[] | | | R17–R21 |
| cohort | id, name, kind `contacts \| accounts` | | yes | R22 |
| | members | list of contact (manual) | | kind contacts |
| | rule | `{op: AND\|OR, conditions[]}` | | kind accounts, R24 |
| condition | field_id, operator, value | | | R24 |
| merge_log | id, kept_id, removed_id, merged_by, merged_at, summary | | | R31 |

## Rules

### Accounts and contacts
- **R1** An account is a customer or prospect organization. It has a name, a description, custom fields and an archived flag. [src:1][src:3]
- **R2** A contact is a person who gives feedback. Contacts are not workspace members and cannot sign in to the workspace. [src:2][src:3]
- **R3** Contacts and accounts are full records. Each can be created, edited, archived and listed in its own view. [src:3][src:4]
- **R4** Both can be created by hand, from a CSV (R28), through the API, or by an integration. Integrations fill `origin`. [src:1][src:2]
- **R5** Archiving hides an account or contact from pickers and default lists. Their feedback and scores stay. Archived records can be restored. [src:1][src:2]
- **R6** Deleting an account or contact asks for confirmation and shows how many feedback entries point at it. Those entries keep their text and lose the customer link. Deleting an account does not delete its contacts. They stay, with no account. [decision]
- **R7** Admins can delete in bulk through the API and in the list. The same rules as R6 apply. [src:3][decision on list]
- **R8** A contact belongs to at most one account. Setting the account moves the contact. An account can have many contacts. [src:1][src:2]
- **R9** Domain matching: if a contact has an email and no account, and an account has a domain equal to the email's domain, the contact is placed in that account. This runs when the contact is created or the email changes. It never moves a contact who already has an account. Free-mail domains (gmail.com, outlook.com and similar) are ignored. [src:1][decision on free-mail and no-move]
- **R10** An account with no domain is never matched by R9. If two accounts share a domain, no match is made and the contact stays without an account. [decision]
- **R11** Names are not used to pick an account. [decision]

### Custom account fields
- **R12** An admin or editor can add custom fields to accounts: number, text, boolean, date, url, single select. [src:1][src:3]
- **R13** Typical uses are revenue, plan, industry, region, renewal date. [src:1]
- **R14** A field's type cannot change once values exist. Deleting a field deletes its values, after confirmation. [decision]
- **R15** A value can be set by hand, import, API or integration. The record remembers who set it last (a member id or an integration name). [decision]
- **R16** Custom fields on contacts are not part of v1. [decision]

### Account totals on items
- **R17** An account total shows, on a product tree item, one value built from a number field of the accounts that asked for the item. [src:5]
- **R18** "Asked for the item" means the account has at least one evidence record on the item or on its children (S04, S02). The evidence can come from feedback linked to the account, or from feedback linked to one of its contacts. [src:5][decision on children]
- **R19** Each account counts once per item, however many evidence records it has. [decision]
- **R20** Functions: sum, average, minimum, maximum, count of accounts. Accounts with an empty value are skipped for average, minimum and maximum, and count as 0 for sum. Example: total revenue of the accounts that asked. [src:5][decision on functions]
- **R21** Account totals can be shown as columns in views (S09), sorted, filtered, and used as inputs to computed fields (S06 R15). They are read-only. They are recalculated within 5 seconds after a change to evidence, an account value, a contact-account link or a move in the product tree. [src:5][decision on timing]

### Cohorts
- **R22** A cohort is a saved group of customers: either contacts (picked by hand) or accounts (picked by rule). [src:6]
- **R23** Hand-picked contact cohort: a member adds and removes contacts one by one or from a CSV of emails. A contact can be in many cohorts. [src:6]
- **R24** Rule-based account cohort: it has one rule with conditions joined by AND or OR (a single operator for the whole rule). Each condition tests one account field. Number fields: =, ≠, >, ≥, <, ≤, is empty, is not empty. Text fields: is, is not, contains, does not contain, is empty, is not empty. [src:6][decision on exact operator list]
- **R25a** Members of a rule-based cohort are recalculated when an account value changes, or an account is created or archived. Members are never added by hand. [src:6][decision on timing]
- **R25b** Archived accounts are not in rule-based cohorts. [decision]
- **R26a** Nested groups of conditions are not in v1. [decision]
- **R26b** A cohort shows its member count. Cohort score columns are defined in S06 R8. [src:6]
- **R26c** Deleting a cohort never deletes accounts or contacts. Views that used it lose its column. [decision]
- **R26d** Cohort names are unique per workspace. [decision]

### Matching and duplicates
- **R25** `external_id` is unique per workspace for accounts, and separately for contacts. Email is unique for contacts, ignoring case. A create or update that breaks this returns a conflict error that names the existing record. [src:3][decision]
- **R27** Matching order for feedback and import: external id first, then email (contacts only). On a match, the existing record is used. With no match, a new record is created. For an account, a name without an external id never matches. [src:3][src:4][decision on order]
- **R28** CSV import of accounts: columns for name, external id, domain, description and one column per custom field. Rows are matched by external id (R27). A matched row updates only the columns that are present and not empty. [src:3][decision on details]
- **R29** CSV import of contacts: name, email, external id, account (by account external id or exact account name). An account name that matches two accounts or none is an error for that row. [decision]
- **R30** Before anything is saved, import shows a report of rows created, updated and failed, with reasons. Up to 10,000 rows per file. [decision]
- **R31** Merge duplicates: an admin or editor picks two or more records of the same kind and one to keep. Their data moves to the kept record and the others are removed. [src:7]
- **R32** On merge, all feedback links, evidence, cohort memberships and (for accounts) contacts move to the kept record. Empty fields on the kept record are filled from the others, in the order picked. Fields that already have a value are not overwritten. Domains and emails of removed records stay on the kept record as extra matching aliases. [src:7][decision on field rules and aliases]
- **R33** Merged ids still work: an API request for a removed id gets a redirect to the kept record. [src:3]
- **R34** Each merge is written to a merge log. It cannot be undone in v1. Scores are recalculated after the merge, and the kept account counts once per item (R19, S06 R4). [decision]
- **R35** Two contacts from different accounts can be merged. The kept contact's account wins. [decision]

### Export
- **R36** Account and contact lists export to CSV with the current filter. Columns: all standard fields, custom fields, cohorts (semicolon list), feedback count. Exporting a cohort exports its members. [src:3][src:4][decision on columns]

## User actions
- Admin and Editor: create, edit, archive, delete, import, export, merge, and manage account fields, account totals and cohorts.
- Collaborator: create contacts and accounts while adding feedback (S04), and edit the ones they created. Cannot merge, delete, import, or edit fields and cohorts.
- Reader: read only.
- Contacts: have no access to the workspace. They only appear as the customer on feedback.

See [S01](S01-workspace-roles.md) for the full role model.

## Acceptance tests
- **T1 (R8)** Given contact U in account A, when a member sets U's account to B, then U is in B only.
- **T2 (R9)** Given account "Acme" with domain acme.com, when contact ann@acme.com is created with no account, then Ann is in Acme. Given contact bob@gmail.com, then Bob stays without an account.
- **T3 (R9)** Given contact cy@acme.com already in account Zed, when the domain rule runs, then Cy stays in Zed.
- **T4 (R10)** Given two accounts with domain dup.com, when contact dan@dup.com is created, then Dan has no account.
- **T5 (R17–R20)** Given feature F with evidence from accounts A (revenue 100, two contacts), B (revenue 50) and C (revenue empty), when "total revenue" (sum) is shown, then F shows 150, and the account count shows 3.
- **T6 (R20)** Given the same data with average, then F shows 75.
- **T7 (R18)** Given an evidence record on a slice of F from account D (revenue 20), then F's total revenue includes 20.
- **T8 (R21)** Given account A's revenue changes from 100 to 200, then F's total revenue updates within 5 seconds.
- **T9 (R24)** Given the rule "revenue > 100 AND plan is Pro", when account A has revenue 200 and plan Pro, then A is a member. When A's plan becomes Free, then A leaves within 5 seconds.
- **T10 (R24)** Given the rule "revenue > 100 OR plan contains Pro", then an account with revenue 10 and plan "Pro Plus" is a member.
- **T11 (R23)** Given a hand-picked cohort, when a member adds the same contact twice, then the contact is in it once.
- **T12 (R25)** Given contact ann@acme.com exists, when another contact is created with ANN@acme.com, then a conflict error names the existing contact.
- **T13 (R27, R28)** Given a CSV row with external id E1 matching account A and a new revenue value but an empty plan, then A's revenue changes and A's plan stays.
- **T14 (R29)** Given a contact row whose account name matches two accounts, then that row fails with a reason and the others import.
- **T15 (R31, R32)** Given duplicate accounts A (keep) and B, B has 2 contacts and 3 feedback entries, when merged, then A has those contacts and entries, B no longer exists, and B's domain still matches A.
- **T16 (R33)** Given B was merged into A, when the API reads B's id, then it gets a redirect to A.
- **T17 (R34)** Given A and B both gave F need `high` and were merged, then the merged account counts once on F.
- **T18 (R6)** Given an account with 4 feedback entries and 3 contacts, when it is deleted, then the confirmation says 4 entries, the entries remain without a customer, and the contacts remain without an account.
- **T19 (R36)** Given a rule-based cohort with 5 members, when exported, then the CSV has 5 rows.

## Tier
Accounts, contacts, custom fields, import, export, merge: all tiers. Account totals and cohorts: paid tiers. CRM sync: see S12. [decision]

## Open questions
1. Should one contact be allowed in many accounts (consultants)?
2. Should merge be undoable for a short time (for example 7 days)?
3. Should custom fields on contacts come in v2?
4. Should account totals be able to count evidence records instead of accounts, or weight by need?
5. Should rule-based cohorts allow nested AND/OR groups?

## Sources
Provenance for `[src:N]` tags is kept in private research material (S05).
