# S14 Security

## Scope
How people log in and how the workspace is protected and audited. It covers:
- Login through Google and through SAML identity providers, and making it mandatory
- SCIM provisioning of users and groups
- Multi-factor login (MFA)
- IP allowlist
- Audit log
- Item history
- Data export and deletion (GDPR)

Related modules:
- Roles, members, teams: [S01](S01-workspace-roles.md)
- Public roadmap visitor login: [S11](S11-public-roadmap.md)
- API tokens and scopes: [S13](S13-api.md)
- Product tree items and their history: [S02](S02-product-tree.md)
- Customer data subject to deletion: [S05](S05-customers.md)

## Entities

| entity | field | type | required | details |
|---|---|---|---|---|
| sso_config | workspace_id, kind | enum `google \| saml` | yes | |
| | idp_name | text | saml | display name |
| | sso_endpoint, slo_endpoint | url | saml, slo optional | |
| | certificate or fingerprint | text | saml | one is enough |
| | status | enum `off \| testing \| enforced` | yes | default `off` |
| | default_role | enum of S01 roles | yes | default `collaborator` |
| scim_config | enabled | bool | yes | |
| | manage_teams | bool | yes | group sync on or off |
| | token | secret | yes | shown once |
| scim_user | external_id, member_id, active | | | link between provider user and member |
| scim_group | external_id, team_id | | | |
| mfa_setting | member_id, enabled, method | | | |
| | workspace_required | bool | | workspace level switch |
| ip_rule | id, workspace_id, cidr, label | | | |
| audit_event | id, workspace_id, at (UTC), actor_member_id, actor_email, ip | | yes | |
| | action | enum `create \| update \| delete \| login \| ...` | yes | see R26 and R27 |
| | entity_type, entity_id, entity_name | | yes | |
| | changes | JSON | no | before and after values |
| item_history | item_id, at, actor, field, old_value, new_value | | | per item, see R26 |
| data_request | id, kind | enum `export \| delete` | yes | |
| | subject_type | enum `member \| contact \| account` | yes | |
| | subject_id, status, requested_by, created_at, done_at | | | |

## Rules

### Single sign-on
- **R1** On all tiers, members can log in with Google. [src:4]
- **R2** SAML SSO works with any provider that speaks SAML 2.0. Documented guides: Okta, Entra ID, OneLogin, Google Workspace. [src:1][src:2]
- **R3** The SAML request uses the email address as the NameID. The audience (entity ID) is a fixed value shown in settings. The provider sends email, first name and last name. [src:1]
- **R4** Setup: the admin enters the provider's endpoint and certificate (or uploads provider metadata), then runs a test login. SSO only moves from `testing` to `enforced` after a successful test login by the admin. [src:1; the test gate is partly a decision]
- **R5** Emails from the provider must match the member's email in Wareslate. If no member matches, a new member is created with the default role. A different spelling would create a duplicate, so emails are compared case-insensitively. [src:1; case-insensitive is a decision]
- **R6** A member created through SAML gets the role `collaborator` unless the provider says otherwise. The provider may send a role attribute (`admin`, `editor`, `collaborator`, `reader`) that sets the role at first sign-in only. After that, admins change roles in Wareslate. Members who already exist keep their role when SSO is turned on. [src:1]
- **R7** When SSO is enforced, all members are signed out once and must use SSO from then on. Password login is off for everyone. [src:1]
- **R8** While SSO is in `testing`, normal login still works, and admins can use email and password to fix settings. [src:1]
- **R9** There is no backup password login after enforcement. If an expired certificate locks everyone out, a Wareslate operator can switch SSO off. Admins also get a visible warning 30 days before a certificate expires. [src:1; warning is a decision]
- **R10** Turning SSO off lets members log in with a password. Members who never had a password get a reset email. [src:1]
- **R11** Google SSO enforcement (only Google accounts of one verified domain may log in) is a paid-tier option. [src:4]
- **R12** A SAML setup can send a workspace list and a role per workspace. A member can only enter the workspaces on their list. Each workspace has its own SSO config. [src:1] (see S01 R41)

### SCIM
- **R13** SCIM needs SAML SSO to be set up and working first. [src:2]
- **R14** SCIM is turned on in workspace settings. Group sync is a separate switch (`manage_teams`). [src:2]
- **R15** The SCIM base address is `<api host>/scim/v2`. It accepts an API access token or OAuth 2. [src:2]
- **R16** For users, SCIM supports list, create and update. There is no delete. Setting `active` to false deactivates the member (S01 R10). Setting it back to true reactivates. [src:2]
- **R17** The provider's own id is stored as `externalId` on the member. [src:2]
- **R18** Groups map to teams. SCIM can list, create, update and delete them. Group sync needs `manage_teams` on. [src:2]
- **R19** Members of a synced group are the members of the team. Hand edits to a synced team's members are refused. [decision]
- **R20** SCIM can set the role through a role attribute using the same values as R6. A change sent by the provider wins over a hand change on the next sync. [src:2 points to role guides; details are a decision]
- **R21** A SCIM call that creates a user with an email that already exists links to the existing member instead of creating a duplicate. [decision]

### MFA
- **R22a** Members can turn on MFA for their own login. It is available on all tiers. [src:4][src:5]
- **R22b** An admin can require MFA for all members who log in with a password. Members logging in through SAML or Google use the provider's MFA instead. [decision]
- **R22c** Method in v1: one-time code from an authenticator app, plus one-time recovery codes shown once at setup. An admin can reset a member's MFA. Every reset is written to the audit log. [decision]

### IP allowlist
- **R23** An admin can set a list of allowed network ranges (CIDR). When the list is not empty, logins and API calls from other addresses are refused with 403. [src:4; behavior is a decision]
- **R24** The list cannot be saved if it would exclude the admin's own current address. [decision]
- **R25** Allowlist is a paid-tier option. [src:4]

### Audit log
- **R26** The audit log records, per event: time (UTC), actor email, actor IP, entity type, entity name, action (create, update or delete) and the changes. [src:3]
- **R27** v1 also records: login, failed login, role change, member invite and deactivation, SSO and SCIM setting changes, IP allowlist changes, MFA reset, share and public link changes, and data requests. [decision; sources list entity changes only]
- **R28** Not recorded: changes to the order of columns in a view, changes to the order of saved views, and edits that only affect how a feedback view is laid out. [src:3]
- **R29** Events are kept for 90 days. Older events are deleted and cannot be exported. [src:3]
- **R30** Only admins can see or export the audit log. [src:3]
- **R31** Export gives a CSV file. It is made in the background and sent as an email with a download link. The file uses UTF-8 and commas. [src:3; sources say the file uses semicolons, Wareslate uses commas: decision]
- **R32** Export filters: number of days, action, entity type, date from, date to, timezone. Dates are UTC unless a timezone is given. [src:3]
- **R33** Audit events cannot be edited or deleted by anyone, including admins, before the 90 days end. [decision]
- **R34** Audit log is a paid-tier option. [src:3]

### Item history
- **R35** Every product tree item keeps a history of changes: who, when, which field, old and new value. [src:4]
- **R36** Anyone who can see the item can see its history. Field values hidden from a member by a custom role (S01 R15) are hidden in history too. [decision]
- **R37** Item history is available on all tiers and has no time limit. [src:4][decision on no limit]

### Data export and deletion
- **R38** An admin can request a deletion of data about one contact, one account or one member. The request removes personal fields (name, email, other personal text) and keeps the counts so demand scores stay stable. [src:4; behavior is a decision]
- **R39** An admin can request an export of everything stored about one person. The result is a JSON file. [decision]
- **R40** Both are also available by API (S13), with a scope that is separate from normal data scopes. [src:4][decision on scope]
- **R41** Data requests finish within 30 days and are written to the audit log. [decision, based on the usual legal limit]
- **R42** Deleting a workspace (admin only, S01 R3) removes all its data. The admin must type the workspace name to confirm. Backups are removed within 30 days. [decision]

## User actions
- Admin: set up SSO, SCIM, MFA rules, IP list; view and export the audit log; run data requests.
- Editor, collaborator, reader: turn on own MFA; log in by SSO. Editors and admins can see item history on items they can see.
- Identity provider (machine): create, update and deactivate members; sync groups.

## Acceptance tests
- **T1 (R4, R7)** Given SSO in `testing`, when the admin finishes a successful test login and sets `enforced`, then every member is signed out and password login is refused.
- **T2 (R6)** Given a provider sending role `reader` for a new person, when they first log in, then a member is created as a reader. Given no role attribute, then collaborator. Given an existing editor, then their role stays editor.
- **T3 (R5)** Given member `Ann@x.com` exists, when the provider sends `ann@x.com`, then no duplicate is created.
- **T4 (R16)** Given SCIM sets `active: false` on a member, then they cannot log in, and DELETE on the user returns an error.
- **T5 (R13)** Given SAML is not configured, when an admin turns SCIM on, then it is refused with a message.
- **T6 (R18, R19)** Given `manage_teams` on and a group with 3 users, then a team with those 3 members exists. When an admin tries to add a 4th by hand, then it is refused. When the group is deleted at the provider, then the team is deleted.
- **T7 (R24)** Given an admin at 10.0.0.5, when saving a list with only 192.168.0.0/16, then it is refused.
- **T8 (R23)** Given an allowlist with 192.168.0.0/16, when an API call comes from 10.0.0.9, then it gets 403.
- **T9 (R29)** Given an event 91 days old, when an admin exports, then it is not in the file. An event 89 days old is.
- **T10 (R30)** Given an editor, when they open the audit log or export, then it is refused.
- **T11 (R26, R32)** Given 5 edits to features and 3 removals in the last 7 days, when the admin exports with action `delete`, then the CSV has 3 rows with time, email, IP, entity, and changes.
- **T12 (R35, R36)** Given a feature whose status changed twice and a field hidden from a member by a custom role, then the member sees both status changes but no history for the hidden field.
- **T13 (R38)** Given a contact linked to 4 feedback entries, when a deletion is run, then the name and email are cleared, the 4 entries remain, and demand scores stay the same.
- **T14 (R22b, R22c)** Given an admin requires MFA, when a password user without MFA logs in, then they must set it up first. When an admin resets a member's MFA, then an audit event exists.

## Tier
- Google login, MFA, item history, basic data requests: all tiers. [src:4]
- SAML SSO, SSO enforcement, SCIM, IP allowlist, audit log, multiple workspaces: top tier. [src:1][src:2][src:3][src:4]

## Open questions
1. Should audit retention be longer than 90 days on request (for example 1 year)?
2. Which MFA methods beyond authenticator apps (passkeys, hardware keys)?
3. Should SCIM be allowed to delete members, with a safe delay?
4. Should the audit log also be sent out through a webhook or log stream?
5. Should data deletion keep a hashed record so a person is not re-imported by a later sync?

## Sources
Provenance for `[src:N]` tags is kept in private research files (S14).
