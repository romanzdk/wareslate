# S01 Workspace and roles

## Scope
Who can do what in Wareslate. It covers:
- Workspaces and members (invite, change role, deactivate)
- Roles: admin, editor, collaborator, reader and custom roles
- Spaces (listed, invite-only, hidden, my space) and access levels
- Sharing views and pages
- Teams
- More than one workspace per organization

Related modules:
- Login, SSO, SCIM, audit log: [S14](S14-security.md)
- Views and sharing of views: [S09](S09-views.md)
- Pages: [S10](S10-pages.md)
- Public roadmap and its visitors: [S11](S11-public-roadmap.md)
- Fields and statuses (custom roles limit field access): [S03](S03-fields-statuses.md)
- Product tree items: [S02](S02-product-tree.md); feedback: [S04](S04-feedback.md); customers: [S05](S05-customers.md)
- API (members and teams endpoints): [S13](S13-api.md)

## Entities

| entity | field | type | required | details |
|---|---|---|---|---|
| workspace | id, name, slug | | yes | one organization's data. The slug is part of its address |
| member | id, workspace_id, email, name, username | | yes | a person with a login. Not a customer contact (S05) |
| | role_id | ref to role | yes | built-in or custom |
| | disabled | bool | yes | default false. See R10 |
| | invitation_pending | bool | yes | true until the invite is accepted |
| | team_ids | list | no | |
| role | id, name, kind | enum `admin \| editor \| collaborator \| reader \| custom` | yes | the four built-in roles cannot be deleted |
| | base_role | enum of built-in roles | custom only | the custom role starts from this one (R15) |
| | field_access | map field_id → `none \| read \| write` | custom only | |
| space | id, name, visibility | enum `listed \| invite_only \| hidden` | yes | |
| | owner_member_id | ref | yes | exactly one |
| | default_collaborator_access | enum `read \| write` | yes | default `read` |
| | archived | bool | yes | |
| space_member | space_id, member_id, access | enum `read \| write \| manage \| owner` | yes | |
| my_space | member_id | | yes | one per member, see R19 |
| share | resource_type (`view \| page`), resource_id | | yes | |
| | target_type | enum `member \| space \| workspace \| link` | yes | |
| | target_id | uuid | for member, space | |
| | access | enum `read \| write` | yes | |
| | link_mode | enum `login_link \| public_link` | for link | |
| view_override | view_id, member_id, access | | no | beats the space access for that view |
| team | id, label, slug, about, image_url | | yes | a named group of members, used for assignment and mentions |
| | member_ids | list | no | |
| invite | id, email, role_id, invited_by, created_at, accepted_at | | yes | |

## Rules

### Workspace and members
- **R1** A workspace is the top-level container. All data (items, feedback entries, views, pages, customers) belongs to exactly one workspace. [src:1][decision on the exact wording]
- **R2** Every member has exactly one role in a workspace. [src:1][src:3]
- **R3** Only an admin can invite members, change roles, deactivate members, manage billing, manage integrations, create public roadmaps, change item statuses and delete the workspace. [src:1]
- **R4** Changing a role on a paid plan can change the bill. Before the admin confirms, Wareslate shows a warning that lists the change. [src:1; the confirmation step is a decision]
- **R5** A member can see their own role in their account settings. [src:1]

### Built-in roles
- **R6** What each role may do, per area (Y = allowed, N = not allowed). [src:1]

| area | admin | editor | collaborator | reader |
|---|---|---|---|---|
| Members, roles, invites, billing, delete workspace | Y | N | N | N |
| Integrations: set up and manage | Y | N | N | N |
| Statuses: customize | Y | N | N | N |
| Public roadmaps: create, change settings | Y | N | N | N |
| Product tree items and feedback entries: create, edit, delete | Y | Y | N | N |
| Item owner: can be set as owner | Y | Y | N | N |
| Feedback entries: create, tag; add contacts and accounts; set entry owner | Y | Y | Y | N |
| Public roadmap entries: create, edit, post updates | Y | Y | N | N |
| Views: create, edit, change settings; create spaces | Y | Y | N | N |
| Pages: create, edit, manage | Y | Y | Y | N |
| Comment on items | Y | Y | Y | N |
| Open feedback views and table views | Y | Y | Y | N |
| Open timeline views, kanban views, pages, shared public roadmaps | Y | Y | Y | Y |

- **R7** A collaborator can add feedback and work on pages. On product tree items they can only comment. [src:1]
- **R8** A reader sees only what is shared with them: timeline views, kanban views, pages and public roadmaps. A reader cannot open table views or feedback views, even inside a listed space. [src:1][src:2]
- **R9** A reader never gets more than `read` access in any space (R16). [src:2]
- **R9a** People who send feedback through a public roadmap are not members and need no login to do it. [src:1] (see S11)
- **R9b** A role is a ceiling. A space or share can lower what a member may do but never raise it above the role. [src:2]

### Invites and deactivation
- **R10** Deactivating a member (`disabled = true`) blocks login at once and ends their sessions. Their name stays on items, feedback entries and history. The member stops counting as a paid seat. Reactivating brings back their old role and space access. [decision; sources show the `disabled` flag only]
- **R11** An invite holds an email and a role. The member shows as `invitation_pending` until the invite is accepted. An admin can resend or cancel it. Invite links expire after 14 days. [src:3 for pending flag; the rest is a decision]
- **R12** Inviting an email that already belongs to a member of this workspace is an error. [decision]
- **R13** The last active admin cannot be deactivated, demoted or removed. [decision]
- **R14** In v1 members cannot be deleted through the API or UI, only deactivated. [decision; the public API treats members as read-only] [src:3]

### Custom roles
- **R15** A custom role starts from one built-in role (its base role). It can limit access to chosen fields and master data, for example hide a field or make it read-only. It can never grant more than its base role. [src:6; the "never more than base" limit is a decision]
- **R15a** Deleting a custom role needs a replacement role for its members. [decision]

### Spaces
- **R16** Space visibility: [src:2]
  - **listed**: everyone in the workspace sees it and its views. Anyone can join.
  - **invite_only**: everyone sees that it exists, but only members see its views. To join, a person needs an invite or the owner's approval of their request.
  - **hidden**: not shown to non-members. A person becomes a member only by creating the space or by an invite from a member.
- **R17** Access levels in a space [src:2]:
  - `read`: see views, cannot change or create them.
  - `write`: add pages, and copy, retitle, relocate or remove them. Cannot create or change views. Meant for collaborators.
  - `manage`: create, change and delete views. Meant for editors.
  - `owner`: everything in `manage`, plus space settings, members, archiving and handing over ownership. One owner per space.
- **R18** Default access: accepting an invite or an approved request gives `manage`. Joining a listed space on your own gives `read`. A collaborator gets `write` or `read` as set by the space setting `default_collaborator_access`. A reader always gets `read`. [src:2]
- **R19** Each member has a my space that only they can use. Admins cannot see it. [src:2]
- **R20** In a listed space, an admin can do everything. For invite-only and hidden spaces an admin must be invited like anyone else, and then gets `owner` level. An admin who is not a member sees no views in it. [src:2]
- **R21** An admin can change a space's visibility. This needs membership when the space is invite-only or hidden. Nothing inside is lost: pages, views, people and their levels are kept. A move to a stricter visibility takes effect at once. [src:2]
- **R22** Invite-only spaces can give `read` or `write` on named products in the product tree to members. [src:2]
- **R23** A view moved into a space uses that space's visibility and edit rules. [src:2]
- **R24** Space features for data visibility, folders and hubs are later work. [src:6][decision to defer]
- **R25** Invite-only and hidden spaces are paid-tier only. [src:2]
- **R26** Admins and editors may create listed and invite-only spaces. Anyone who can create a space may also create a hidden one. [src:1 says admins and editors create spaces; the rest is a decision]

### Sharing views and pages
- **R27** A view or page can be shared with one member, one space, the whole workspace, or through a link. Each share has an access of `read` or `write`. [src:5][decision on link modes]
- **R28** Views and pages are private (only the owner sees them) until shared. [src:5]
- **R29** A view override sets one member's access to one view and wins over the space default. It still obeys R9b. [src:2]
- **R30** Link sharing has two modes: `login_link` (needs a login and a role that may see the resource) and `public_link` (anyone with the link, read only). An admin can turn public links off for the whole workspace. [src:5][workspace switch is a decision]
- **R31** Public links never give `write`. [decision]
- **R32** The share list of a resource shows who has access and through which route (direct, space, workspace, link). Each member also has two lists: "I shared" and "Shared to me". [src:5]
- **R33** The owner of a view can see who opened it. [src:5]
- **R34** Removing a member from a space removes the access that came from the space, but keeps direct shares. [decision]

### Teams
- **R35** A team has a name, handle, description, avatar and a list of members. Teams can be created, read, updated and deleted. [src:3]
- **R36** A team can be mentioned and chosen in owner and team fields (S03). Deleting a team clears it from those fields. [src:3 for team fields; clearing is a decision]
- **R37** Teams can be filled by hand or by group sync from the identity provider (S14). Members of a synced team cannot be edited by hand. [decision]
- **R38** In v1 a team has no access rights of its own. Access comes from roles, spaces and shares. [decision]

### Multiple workspaces
- **R39** One organization can have more than one workspace, each with its own members, roles, data and settings. This is paid-tier only. [src:4][src:5]
- **R40** A person has one login but a separate member record and role in each workspace. [decision]
- **R41** When the identity provider sends a workspace list, a member can only enter the listed workspaces, with a role for each. See S14. [src:4]

## User actions
- Admin: everything in R3, plus everything an editor can do. Cannot see other people's my spaces or invite-only and hidden spaces they are not in (R19, R20).
- Editor: create and edit items, views, spaces and pages; share views and pages they can edit.
- Collaborator: add feedback entries, work on pages, comment, open feedback views. Shares only what they own.
- Reader: open what is shared with them.
- Public roadmap visitor: not a member (R9a).
- Member of any role: see own role, see the "I shared" and "Shared to me" lists, leave a space they joined.

## Acceptance tests
- **T1 (R3, R6)** Given an editor, when they try to invite a member, then it is refused. Given an admin, then it works.
- **T2 (R6, R7)** Given a collaborator, when they create a feedback entry, it works. When they edit a feature's name, it is refused. When they comment on the feature, it works.
- **T3 (R8, R9)** Given a reader who joins a listed space, when they open a table view, then access is denied. When they open a kanban view shared with them, they see it with `read`.
- **T4 (R16, R20)** Given a hidden space and an admin who was not invited, when the admin lists spaces, then it does not appear. After an invite, the admin has `owner` level.
- **T5 (R18)** Given a collaborator joins a listed space on their own and `default_collaborator_access` is `write`, then they can create pages but not views.
- **T6 (R19)** Given a member's my space with a page, when an admin searches the workspace, then the page is not found.
- **T7 (R10, R13)** Given two admins, when one is deactivated, then they cannot log in and the other is still admin. When the other admin is deactivated, then it is refused.
- **T8 (R29, R9b)** Given a reader with a view override of `write`, then they still only get `read`.
- **T9 (R30, R31)** Given a page with a `public_link`, when a person who is not logged in opens the link, they can read it but not edit. When the admin turns public links off, then the link stops working.
- **T10 (R21)** Given a listed space with 5 members, when an admin changes it to hidden (after joining it), then views and members stay and non-members lose sight of it at once.
- **T11 (R11, R12)** Given a pending invite for a@x.com, when the same email is invited again, then it is refused with an error. After 14 days the invite is expired.
- **T12 (R15)** Given a custom role based on collaborator with field "Budget" set to `none`, then a member with that role does not see "Budget" on items and cannot set it through the API.
- **T13 (R41)** Given a member whose provider workspace list holds only workspace A, when they try to enter workspace B, then access is refused.

## Tier
- Four built-in roles, listed spaces, my space, teams, sharing: all tiers.
- Custom roles, invite-only and hidden spaces, multiple workspaces: top tier only. [src:2][src:5]
- The number of reader and collaborator seats may be capped on lower tiers. [src:5][decision on limits]

## Open questions
1. Should custom roles be able to add abilities (for example "can set statuses") on top of a base role, or only remove them?
2. Should a team be usable as a share target, not only a space?
3. Should invites expire after 14 days, or never?
4. Who may create listed and invite-only spaces: editors too, or admins only?
5. Should an admin be able to take over a deactivated member's my space after a set time?

## Sources
Provenance for `[src:N]` tags is kept in private research files (S01).
