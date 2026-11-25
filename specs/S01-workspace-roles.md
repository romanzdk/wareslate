# S01 Workspace and roles

## Scope
Who can do what in Wareslate. It covers:
- Workspaces and members (invite, change role, deactivate)
- Roles: admin, maker, contributor, viewer and custom roles
- Teamspaces (open, closed, private, personal) and access levels
- Sharing boards and documents
- Teams
- More than one workspace per organization

Related modules:
- Login, SSO, SCIM, audit log: [S14](S14-security.md)
- Boards and sharing of boards: [S09](S09-boards.md)
- Documents: [S10](S10-documents.md)
- Public roadmap portal and its visitors: [S11](S11-portal.md)
- Fields and statuses (custom roles limit field access): [S03](S03-fields-statuses.md)
- Hierarchy items: [S02](S02-hierarchy.md); feedback: [S04](S04-feedback-insights.md); customers: [S05](S05-customers.md)
- API (members and teams endpoints): [S13](S13-api.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| workspace | id, name, slug | | yes | one data space. Slug is part of its address |
| member | id, workspace_id, email, name, username | | yes | a person with a login. Not a customer user (S05) |
| | role_id | ref to role | yes | built-in or custom |
| | disabled | bool | yes | default false. See R10 |
| | invitation_pending | bool | yes | true until the invite is accepted |
| | team_ids | list | no | |
| role | id, name, kind | enum `admin \| maker \| contributor \| viewer \| custom` | yes | the four built-in roles cannot be deleted |
| | base_role | enum of built-in roles | custom only | the custom role starts from this one (R6) |
| | field_access | map field_id → `none \| read \| write` | custom only | |
| teamspace | id, name, type | enum `open \| closed \| private` | yes | |
| | owner_member_id | ref | yes | exactly one |
| | default_contributor_access | enum `view \| edit_content` | yes | default `view` |
| | archived | bool | yes | |
| teamspace_member | teamspace_id, member_id, access | enum `view \| edit_content \| edit \| owner` | yes | |
| personal_space | member_id | | yes | one per member, see R19 |
| share | resource_type (`board \| document`), resource_id | | yes | |
| | target_type | enum `member \| teamspace \| workspace \| link` | yes | |
| | target_id | uuid | for member, teamspace | |
| | access | enum `view \| edit` | yes | |
| | link_mode | enum `private_link \| public_link` | for link | |
| board_override | board_id, member_id, access | | no | beats the teamspace access for that board |
| team | id, name, handle, description, avatar_url | | yes | a named group of members, used for assignment and mentions |
| | member_ids | list | no | |
| invite | id, email, role_id, invited_by, created_at, accepted_at | | yes | |

## Rules

### Workspace and members
- **R1** A workspace is the top-level container. All data (items, notes, boards, documents, customers) belongs to exactly one workspace. [src:1][decision on the exact wording]
- **R2** Every member has exactly one role in a workspace. [src:1][src:3]
- **R3** Only an admin can invite members, change roles, deactivate members, manage billing, manage integrations, create portals, change item statuses and delete the workspace. [src:1]
- **R4** An admin changing a role on a paid plan can change the bill. Wareslate shows a warning with the change before the admin confirms. [src:1; the confirmation step is a decision]
- **R5** A member can see their own role in their account settings. [src:1]

### Built-in roles
- **R6** Role abilities, per area (Y = allowed, N = not allowed). [src:1]

| area | admin | maker | contributor | viewer |
|---|---|---|---|---|
| Members, roles, invites, billing, delete workspace | Y | N | N | N |
| Integrations: set up and manage | Y | N | N | N |
| Statuses: customize | Y | N | N | N |
| Portals: create, change settings | Y | N | N | N |
| Hierarchy items and notes: create, edit, delete | Y | Y | N | N |
| Item owner: can be set as owner | Y | Y | N | N |
| Notes: create, tag; add customer users and companies; set note owner | Y | Y | Y | N |
| Portal cards: create, edit, post card updates | Y | Y | N | N |
| Boards: create, edit, change settings; create teamspaces | Y | Y | N | N |
| Documents: create, edit, manage | Y | Y | Y | N |
| Comment on items | Y | Y | Y | N |
| View insights boards and grids | Y | Y | Y | N |
| View timelines, columns boards, documents, shared portals | Y | Y | Y | Y |

- **R7** A contributor can add feedback and work on documents. On hierarchy items they can only comment. [src:1]
- **R8** A viewer sees only what is shared with them: timelines, columns boards, documents and portals. A viewer cannot open grid boards or insights boards, even inside an open teamspace. [src:1][src:2]
- **R9** Viewer is never raised above `view` access in any teamspace (R16). [src:2]
- **R9a** People who give feedback through a portal are not members and do not need a login to submit. [src:1] (see S11)
- **R9b** A role is a ceiling. A teamspace or share can lower what a member can do but never raise it above the role. [src:2]

### Invites and deactivation
- **R10** Deactivating a member (`disabled = true`) blocks login at once and ends their sessions. Their name stays on items, notes and history. The member no longer counts as a paid seat. Reactivating restores their old role and teamspace access. [decision; sources show the `disabled` flag only]
- **R11** An invite holds an email and a role. The member shows as `invitation_pending` until accepted. An admin can resend or cancel the invite. Invite links expire after 14 days. [src:3 for pending flag; the rest is a decision]
- **R12** Inviting an email that already belongs to a member in this workspace is an error. [decision]
- **R13** The last active admin cannot be deactivated, demoted or removed. [decision]
- **R14** Members cannot be deleted through the API or UI in v1. Only deactivated. [decision; the public API treats members as read-only] [src:3]

### Custom roles
- **R15** A custom role starts from one built-in role (its base role) and can restrict access to chosen fields and master data (for example, hide a field or make it read-only). It can never grant more than its base role. [src:6; the "never more than base" limit is a decision]
- **R15a** Deleting a custom role needs a replacement role for its members. [decision]

### Teamspaces
- **R16** Teamspace types: [src:2]
  - **open**: everyone in the workspace sees it and its boards. Anyone can join.
  - **closed**: everyone sees that it exists, but only members see its boards. Joining needs an invite or owner approval of a request.
  - **private**: hidden from non-members. Membership only by creating it or by invite from a member.
- **R17** Access levels in a teamspace [src:2]:
  - `view`: see boards, cannot change or create them.
  - `edit_content`: create, rename, duplicate, delete and move documents. Cannot create or change other board types. For contributors.
  - `edit`: create, change and delete boards. For makers.
  - `owner`: edit, plus manage settings and members, archive, and hand over ownership. One owner per teamspace.
- **R18** Default access: accepting an invite or an approved request gives `edit`. Joining an open teamspace on your own gives `view`. A contributor gets `edit_content` or `view` by the teamspace setting `default_contributor_access`. A viewer is always `view`. [src:2]
- **R19** Each member has a personal space that is private to them. Admins cannot see it. [src:2]
- **R20** Admins have full access to open teamspaces. For closed and private teamspaces an admin must be invited like anyone else, and then has `owner` level. An admin who is not a member sees no boards in it. [src:2]
- **R21** An admin can change a teamspace type. For closed and private teamspaces the admin must be a member. Content, members and access levels stay. Moving to a stricter type applies at once. [src:2]
- **R22** Closed teamspaces can grant `view` or `edit` on named hierarchy items (products) to members. [src:2]
- **R23** A board moved into a teamspace takes that teamspace's visibility and edit rules. [src:2]
- **R24** Teamspace features for visibility of data, folders and hubs are later work. [src:6][decision to defer]
- **R25** Closed and private teamspaces are paid-tier only. [src:2]
- **R26** Who may create open and closed teamspaces: admins and makers. Anyone who can create a teamspace may create a private one. [src:1 says admin and maker create teamspaces; the rest is a decision]

### Sharing boards and documents
- **R27** A board or document can be shared with: one member, one teamspace, the whole workspace, or by link. Each share has an access of `view` or `edit`. [src:5][decision on link modes]
- **R28** Boards and documents can be private (visible to the owner only) until shared. [src:5]
- **R29** A board override sets one member's access to one board and wins over the teamspace default. It still respects R9b. [src:2]
- **R30** Link sharing has two modes: `private_link` (needs a login and a role that can see the resource) and `public_link` (anyone with the link, read only). Public links can be turned off for the whole workspace by an admin. [src:5][workspace switch is a decision]
- **R31** Public links never give `edit`. [decision]
- **R32** The share list for a resource shows who has access and from where (direct, teamspace, workspace, link). Each user can see "Shared by me" and "Shared with me". [src:5]
- **R33** The owner of a board can see a list of who opened it. [src:5]
- **R34** Removing a member from a teamspace removes teamspace-based access, but keeps direct shares. [decision]

### Teams
- **R35** A team has a name, handle, description and avatar, and a list of members. Teams can be created, read, updated and deleted. [src:3]
- **R36** A team can be mentioned and used as a target in owner and team fields (S03). Deleting a team clears it from those fields. [src:3 for team fields; clearing is a decision]
- **R37** Teams can be filled by hand or by group sync from the identity provider (S14). Groups synced from the provider cannot be edited by hand. [decision]
- **R38** A team has no access rights of its own in v1. Access is by role, teamspace or share. [decision]

### Multiple workspaces
- **R39** One organization can have more than one workspace, each with its own members, roles, data and settings. This is paid-tier only. [src:4][src:5]
- **R40** A person has one login but a separate member record and role per workspace. [decision]
- **R41** When the identity provider sends a project list, a member can only enter the listed workspaces, with a role per workspace. See S14. [src:4]

## User actions
- Admin: R3 list, plus everything a maker can do. Cannot see other people's personal spaces or non-member closed and private teamspaces (R19, R20).
- Maker: create and edit items, boards, teamspaces, documents; share boards and documents they can edit.
- Contributor: add notes, work on documents, comment, view insights boards. Shares only what they own.
- Viewer: view what is shared with them.
- Portal visitor: not a member (R9a).
- Member of any role: see own role, mine and shared lists, leave a teamspace they joined.

## Acceptance tests
- **T1 (R3, R6)** Given a maker, when they try to invite a member, then it is refused. Given an admin, then it works.
- **T2 (R6, R7)** Given a contributor, when they create a note, it works. When they edit a feature's name, it is refused. When they comment on the feature, it works.
- **T3 (R8, R9)** Given a viewer who joins an open teamspace, when they open a grid board, then access is denied. When they open a columns board shared with them, they see it with `view`.
- **T4 (R16, R20)** Given a private teamspace and an admin who was not invited, when the admin lists teamspaces, then it does not appear. After an invite, the admin has `owner` level.
- **T5 (R18)** Given a contributor joins an open teamspace on their own and `default_contributor_access` is `edit_content`, then they can create documents but not boards.
- **T6 (R19)** Given a member's personal space with a document, when an admin searches the workspace, then the document is not found.
- **T7 (R10, R13)** Given two admins, when one is deactivated, then they cannot log in and the other is still admin. When the other admin is deactivated, then it is refused.
- **T8 (R29, R9b)** Given a viewer with a board override of `edit`, then they still only get `view`.
- **T9 (R30, R31)** Given a document with a `public_link`, when a not-logged-in person opens the link, they can read it but not edit. When the admin turns public links off, then the link stops working.
- **T10 (R21)** Given an open teamspace with 5 members, when an admin changes it to private (after joining it), then boards and members stay and non-members lose sight of it at once.
- **T11 (R11, R12)** Given a pending invite for a@x.com, when the same email is invited again, then it is refused with an error. After 14 days the invite is expired.
- **T12 (R15)** Given a custom role based on contributor with field "Budget" set to `none`, then a member with that role does not see "Budget" on items and cannot set it through the API.
- **T13 (R41)** Given a member with the provider project list limited to workspace A, when they try to enter workspace B, then access is refused.

## Tier
- Four built-in roles, open and personal teamspaces, teams, sharing: all tiers.
- Custom roles, closed and private teamspaces, multiple workspaces: top tier only. [src:2][src:5]
- Viewer and contributor seat counts may be limited on lower tiers. [src:5][decision on limits]

## Open questions
1. Should custom roles be able to add abilities (for example "can set statuses") on top of a base role, or only remove them?
2. Should a team be usable as a share target, not only a teamspace?
3. Should invites expire after 14 days, or never?
4. Who may create open and closed teamspaces: makers too, or admins only?
5. Should an admin be able to take over a deactivated member's personal space after a set time?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S01).
