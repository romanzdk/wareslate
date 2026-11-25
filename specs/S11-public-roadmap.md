# S11 Public roadmap

## Scope
A public roadmap website where visitors see chosen items, vote on them, and suggest ideas. It covers:
- Structure: sections, lists, entries
- Voting, idea suggestions, email confirmation
- How votes and ideas turn into feedback
- Updates to voters, moderation
- Visibility, custom domain, iframe, SSO, branding, localization
- Number of public roadmaps by tier

Related modules:
- Roles and who can manage public roadmaps: [S01](S01-workspace-roles.md)
- Items shown as entries: [S02](S02-product-tree.md)
- Feedback entries and evidence: [S04](S04-feedback.md)
- How need counts in scores: [S06](S06-prioritization.md)
- Accounts and contacts for visitors: [S05](S05-customers.md)
- SSO and security for public roadmaps: [S14](S14-security.md)
- Entry descriptions use the editor from [S10](S10-pages.md)

## Entities

| entity | field | type | required | details |
|---|---|---|---|---|
| public_roadmap | id, workspace_id, name | | yes | |
| | visibility | enum `hidden \| members \| secret_link \| public` | yes | default `hidden` |
| | link_token | string | if secret_link | random, can be rotated |
| | custom_domain | hostname | no | |
| | ideas_enabled | bool | yes | default true |
| | show_vote_counts | bool | yes | default true |
| | iframe_allowed_origins | list | no | empty = embedding off |
| | sso_enabled | bool | yes | default false |
| | branding | logo, colors (font, primary button, entry background, page background), hide_badge | | |
| | default_locale | string | yes | |
| | moderation_mode | enum `off \| ideas \| ideas_and_votes` | yes | default `ideas` |
| roadmap_section | id, roadmap_id, name, position | | yes | |
| roadmap_list | id, section_id, name, position | | yes | |
| roadmap_entry | id, list_id, item_id, position | | yes | one item once per public roadmap |
| | public_name, public_description, image | | no | empty name or description = fall back, see R7 |
| | public_status | enum or null | no | shown on the entry, see R8 |
| vote_scale | roadmap_id, question, options (4) | | yes | each option: label, need level (R14) |
| vote | id, entry_id, visitor_id, option, created_at | | | one per visitor per entry, changeable |
| visitor | id, email, email_verified_at, sso_subject | | | |
| idea | id, roadmap_id, visitor_id, title, body, state | enum `pending \| approved \| rejected` | | |
| entry_update | id, entry_id, body, author_id, created_at, notified_at | | | |
| roadmap_translation | roadmap_id, locale, key, text | | | UI texts only |

## Rules

### Structure
- **R1** A public roadmap has sections. A section has lists. A list has entries. Any number of sections and lists. [src:1]
- **R2** Admins and editors can rename, move and delete sections and lists. Deleting a section or list that has entries asks where the entries should go. If a list is removed, its entries move to the list above it, or to the first list when there is none above. [src:1] Asking for a destination when a section is deleted is [src:1]; the "first list" fallback is a [decision].
- **R3** An entry is an item of type product, module, feature or slice. [src:1] Other item types (bet, release) may be allowed later. [decision]
- **R4** An item can be an entry only once per public roadmap, but it can appear in several roadmaps. [decision]
- **R5** Entries are added by picking items. Adding leaves the item as it was. Removing an entry leaves the item and its feedback as they were. [decision]

### Entry content
- **R6** An entry has its own public name, public description and image, separate from the internal ones. [src:1]
- **R7** If the public name is empty, the item name is shown. If the public description is empty, the entry shows no description. The internal description is never shown. [decision]
- **R8** An entry can show the item's status when an editor switches that on for the entry. Only the status label is shown, no internal fields. [decision]
- **R9** Public descriptions use the page editor (S10) with a limited set of blocks: text, headings, lists, links, images. Embeds that run scripts are not allowed. HTML is sanitized when saved and when served. [decision]
- **R10** No data from an item is public unless the item is an entry in a public roadmap that the visitor is allowed to see. [decision]

### Voting
- **R11** A visitor votes on an entry by picking one option on a need scale. The default options are: must have, would help, small gain, not needed. [src:1]
- **R12** Admins can rename the four options and the question text. They cannot add or remove options. [src:1 says rename; the fixed count is a decision]
- **R13** Each visitor has one vote per entry. Voting again replaces the earlier vote. [decision]
- **R14** Each option maps to a need level (`high`, `medium`, `low`, `none`), which S06 turns into demand points (0 to 3, see S06 R1). Renaming an option does not change its level. [decision]
- **R15** Admins can show or hide vote counts on entries. When hidden, visitors see only their own vote. The count stays visible inside the workspace. [src:1]
- **R16** A visitor can vote only after email confirmation (R19) or SSO (R35). [src:1]

### Idea suggestions
- **R17** A visitor can suggest an idea (title and text) that is not tied to any entry. On its own it is not linked to any item. [src:1]
- **R18** An admin can switch idea suggestions off: the button disappears and the server refuses new ideas, while voting on entries keeps working. [src:1]
- **R19** Email confirmation: before a vote or idea counts, the visitor gives an email address and clicks the link sent there. The confirmation lasts for the browser session, so it is asked once per session. The roadmap logo appears in that email. [src:1] The session length is a [decision]: 30 days.
- **R20** Votes and ideas from an unconfirmed email stay pending for 24 hours and are then deleted. [decision]
- **R21** Rate limits: at most 20 votes and 3 ideas per visitor per hour, and at most 100 confirmation emails per public roadmap per hour. [decision]

### Feedback
- **R22** A confirmed vote creates a feedback entry in the workspace (kind `message`), linked to the entry's item as evidence with the need level from R14. The feedback source is `public_roadmap`, and its contact is the visitor (found or created from the email, S05). [src:1][the contact match is a decision]
- **R23** Changing a vote updates the same evidence. Removing a vote removes the evidence. [decision]
- **R24** Need from the public roadmap uses the rule in S06 R3: it counts only when the account has no earlier value on that item from the app or API. [src:2]
- **R25** A confirmed, approved idea becomes a feedback entry in state `new`, not linked to any item. Triage is in S04. [src:1]
- **R26** The visitor's account is set when the email domain matches a known account (S05). Otherwise the visitor has no account. [decision]

### Updates to voters
- **R27** An editor can post a text update on an entry. The update shows on the entry. [src:1]
- **R28** When posting, the editor can choose to email the entry's voters. Emails go only to confirmed voters who have not unsubscribed. Every email has an unsubscribe link. [src:1 for closing the loop; the details are a decision]
- **R29** An entry can also notify its voters when its item reaches a chosen status, for example "shipped". This is sent once per status change. [decision]

### Moderation
- **R30** Moderation mode `ideas`: new ideas are `pending` until an editor approves or rejects them. Mode `ideas_and_votes`: votes are held too. Mode `off`: nothing is held. [decision; sources say moderation exists but not how it works]
- **R31** Rejected ideas are hidden and create no feedback. A held vote creates no evidence until it is approved. [decision]
- **R32** Editors can block a visitor email or a whole domain. Blocked visitors cannot vote or suggest ideas. Their earlier votes disappear from the public view and are flagged inside the workspace. [decision]
- **R33** In v1, ideas are not shown to other visitors. A visitor sees only their own ideas. [decision]

### Visibility, domain, embed, SSO
- **R34** Visibility is set per public roadmap, independent of other roadmaps: [src:1]
  - `hidden`: only admins and editors can open it (the default).
  - `members`: signed-in workspace members of any role can browse, but cannot vote or suggest ideas as visitors.
  - `secret_link`: anyone who has the link; search engines are told not to index it.
  - `public`: anyone; search engines may index it.
- **R35** SSO: when enabled, visitors sign in with the workspace's identity provider (S14). A signed-in visitor skips email confirmation, and their votes use their SSO identity. [src:1]
- **R36** A custom domain can be set. The admin proves ownership with a DNS record, and Wareslate issues and renews the TLS certificate. Until the domain is verified, the default address keeps working. [src:1 lists custom domain; the method is a decision]
- **R37** A public roadmap can be embedded in an iframe only on the listed origins. Other origins are refused with a frame-ancestors header. [src:1; the origin list is a decision]
- **R38** Branding: a logo (best at about 72 by 72 px; also used as favicon), and colors for font, primary buttons, entry background and page background. [src:1]
- **R39** The "powered by" badge can be hidden only on the top tier. [src:1]
- **R40** Localization: the fixed UI texts (search box, buttons, vote question) can be translated per locale. Entry names, descriptions and updates are not translated automatically. [src:1] The locale comes from the visitor's browser, and a picker appears when more than one locale exists. [decision]

### Roles and tiers
- **R41** Admins manage public roadmaps (create, delete, visibility, domain, SSO, moderation). Editors change sections, lists, entries and updates, and moderate. Collaborators and readers can browse a roadmap in `members` visibility and cannot edit. [src:1]
- **R42** Number of public roadmaps: 2 on the middle tier, unlimited on the top tier. Creating one over the limit is refused with a message. [src:2]

## User actions

| action | admin | editor | collaborator | reader | visitor |
|---|---|---|---|---|---|
| create, delete roadmap; visibility, domain, SSO | yes | no | no | no | no |
| branding, vote scale, idea switch | yes | yes | no | no | no |
| edit sections, lists, entries | yes | yes | no | no | no |
| post entry update, moderate | yes | yes | no | no | no |
| browse (when visible to them) | yes | yes | `members` visibility | `members` visibility | public visibility |
| vote, suggest idea | yes | yes | no | no | after confirmation |

See [S01](S01-workspace-roles.md) for the full role model. Letting editors change branding and the vote scale is a [decision].

## Acceptance tests
- **T1 (R6, R7)** Given an entry with public name "Dark mode" and an empty public description, when a visitor opens it, then the name is "Dark mode" and no description shows.
- **T2 (R2)** Given a section with 4 entries, when an editor deletes the section, then the system asks where to move the entries and moves them.
- **T3 (R11, R13, R14)** Given a confirmed visitor votes "would help" and then "must have" on an entry, then one vote exists and the evidence has 3 points.
- **T4 (R15)** Given vote counts are hidden, when a visitor opens an entry, then no count shows, and the editor still sees the count in the workspace.
- **T5 (R18)** Given idea suggestions are off, when a visitor posts an idea through the API, then it is refused. Voting still works.
- **T6 (R19, R20)** Given a new visitor votes, when they have not clicked the email link, then the vote is pending and no evidence exists. After 24 hours it is deleted.
- **T7 (R22, R25)** Given a visitor confirms and votes on feature F, then a feedback entry with source `public_roadmap` exists, linked to F. Given an approved idea, then a feedback entry exists with no item link.
- **T8 (R24)** Given contact A already set need `low` on F in the app, when A votes "must have" on the public roadmap, then A still counts 1 point.
- **T9 (R28)** Given 5 confirmed voters, one has unsubscribed, when the editor posts an update with notify on, then 4 emails are sent.
- **T10 (R30, R31)** Given mode `ideas`, when a visitor suggests an idea, then it is pending, hidden from the visitor view and no feedback entry exists. When an editor approves it, then the feedback entry is created.
- **T11 (R34)** Given a roadmap is `hidden`, when an anonymous user opens its link, then access is denied. Given `secret_link`, then it opens and the page asks search engines not to index it.
- **T12 (R35)** Given SSO is on, when a signed-in visitor votes, then no email confirmation is asked.
- **T13 (R37)** Given allowed origin A, when the public roadmap is embedded from origin B, then the browser refuses it.
- **T14 (R42)** Given a workspace on the middle tier with 2 public roadmaps, when an admin creates a third, then it is refused.
- **T15 (R10)** Given an item that is not an entry anywhere, when anyone requests it through the public roadmap API, then the answer is "not found".
- **T16 (R40)** Given locale `cs` has a translation for the button text, when a browser asks for `cs`, then the button shows the translation and entry text is unchanged.

## Tier
Public roadmap basics, voting, branding: paid tiers. 2 public roadmaps on the middle tier, unlimited on the top tier. Hiding "powered by", SSO and moderation of votes: top tier. [src:2] for the counts and badge; the rest is a [decision]

## Open questions
1. Should visitors see other visitors' ideas and vote on them (a public idea list)?
2. Should public roadmaps support more than four vote options, or numeric scales?
3. How should a visitor email match an existing contact, to avoid duplicates (R22, R26)?
4. Should entry updates support scheduling?
5. Does a free tier get one public roadmap?

## Sources
Provenance for `[src:N]` tags is kept in private research files (S11).
