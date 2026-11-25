# S11 Portal

## Scope
A public roadmap portal where visitors see selected items, vote on them, and submit ideas. It covers:
- Structure: tabs, sections, cards
- Voting, idea submission, email confirmation
- How votes and ideas become feedback
- Updates to voters, moderation
- Visibility, custom domain, iframe, SSO, branding, localization
- Number of portals by tier

Related modules:
- Roles and who can manage portals: [S01](S01-workspace-roles.md)
- Items shown as cards: [S02](S02-hierarchy.md)
- Feedback notes and importance: [S04](S04-feedback-insights.md)
- How importance counts in scores: [S06](S06-prioritization.md)
- Customers and companies of visitors: [S05](S05-customers.md)
- Portal SSO and security: [S14](S14-security.md)
- Card descriptions use the editor from [S10](S10-documents.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| portal | id, workspace_id, name | | yes | |
| | visibility | enum `hidden \| company \| private_link \| public` | yes | default `hidden` |
| | link_token | string | if private_link | random, can be rotated |
| | custom_domain | hostname | no | |
| | ideas_enabled | bool | yes | default true |
| | show_vote_counts | bool | yes | default true |
| | iframe_allowed_origins | list | no | empty = embedding off |
| | sso_enabled | bool | yes | default false |
| | branding | logo, colors (font, primary button, card background, page background), hide_badge | | |
| | default_locale | string | yes | |
| | moderation_mode | enum `off \| ideas \| ideas_and_votes` | yes | default `ideas` |
| portal_tab | id, portal_id, name, position | | yes | |
| portal_section | id, tab_id, name, position | | yes | |
| portal_card | id, section_id, item_id, position | | yes | one item once per portal |
| | public_name, public_description, image | | no | empty name or description = fall back, see R7 |
| | public_status | enum or null | no | shown on the card, see R8 |
| vote_scale | portal_id, question, options (4) | | yes | each option: label, points (R14) |
| vote | id, card_id, visitor_id, option, created_at | | | one per visitor per card, changeable |
| visitor | id, email, email_verified_at, sso_subject | | | |
| idea | id, portal_id, visitor_id, title, body, state | enum `pending \| approved \| rejected` | | |
| card_update | id, card_id, body, author_id, created_at, notified_at | | | |
| portal_translation | portal_id, locale, key, text | | | UI texts only |

## Rules

### Structure
- **R1** A portal has tabs. A tab has sections. A section has cards. Any number of tabs and sections. [src:1]
- **R2** Admins and makers can rename, move and delete tabs and sections. Deleting a tab or section that has cards asks where the cards go. A removed section moves its cards to the section above, or the first section if there is none above. [src:1] Asking for a destination on tab delete is [src:1]; "first section" fallback is a [decision].
- **R3** A card is an item of type product, component, feature or subfeature. [src:1] Other item types (initiative, release) may be allowed later. [decision]
- **R4** One item can be a card once per portal. It can be in many portals. [decision]
- **R5** Cards are added by picking items. Adding does not change the item. Removing a card does not change the item or its feedback. [decision]

### Card content
- **R6** A card has its own public name, public description and image, set apart from the internal ones. [src:1]
- **R7** If the public name is empty, the item name is shown. If the public description is empty, the card shows no description. The internal description is never shown. [decision]
- **R8** A card can show the item's status, if the maker turns it on for that card. Only the status label, not internal fields. [decision]
- **R9** Public descriptions use the document editor (S10) with a limited block set: text, headings, lists, links, images. Embeds that run scripts are not allowed. HTML is sanitized on save and on output. [decision]
- **R10** Nothing from an item is public unless it is on a card of a portal that is visible to the visitor. [decision]

### Voting
- **R11** A visitor votes on a card by choosing one option on an importance scale. Default scale: critical, important, nice-to-have, not important. [src:1]
- **R12** Admins can rename the four options and the question text. They cannot add or remove options. [src:1 says rename; fixed count is a decision]
- **R13** One vote per visitor per card. Voting again changes the vote. [decision]
- **R14** Each option maps to importance points as in [S06](S06-prioritization.md) R1 (0 to 3). Renaming does not change the points. [decision]
- **R15** Admins can show or hide the number of votes on cards. When hidden, visitors see only their own vote. The count stays visible in the workspace. [src:1]
- **R16** Visitors can vote only after email confirmation (R19), or SSO (R35). [src:1]

### Idea submission
- **R17** Visitors can submit an idea (title and text) that is not tied to a card. It is not linked to any item on its own. [src:1]
- **R18** An admin can turn idea submission off: the button is hidden and the server refuses new ideas, while voting on cards still works. [src:1]
- **R19** Email confirmation: before a vote or idea counts, the visitor enters an email and clicks a link sent to it. The confirmation holds for the browser session, so it is asked once per session. The portal logo shows in this email. [src:1] Session length is a [decision]: 30 days.
- **R20** Votes and ideas from an unconfirmed email are kept as pending for 24 hours and then deleted. [decision]
- **R21** Rate limits: at most 20 votes and 3 ideas per visitor per hour; at most 100 confirmation emails per portal per hour. [decision]

### Feedback
- **R22** A confirmed vote creates a feedback note in the workspace, linked to the card's item as an insight with the importance from R14. The note's source is `portal`, and its customer is the visitor's user (found or created from the email, S05). [src:1][decision on the customer match]
- **R23** Changing a vote updates the same insight. Removing a vote removes the insight. [decision]
- **R24** Portal importance follows the rule in S06 R3: it counts only when the customer has no earlier value on that item from the app or API. [src:2]
- **R25** A confirmed, approved idea becomes an unprocessed feedback note, not linked to an item. Triage is in S04. [src:1]
- **R26** The visitor's company is set when the email domain matches a known company (S05). Otherwise the visitor has no company. [decision]

### Updates to voters
- **R27** A maker can post an update on a card, with text. The update shows on the card. [src:1]
- **R28** On posting, the maker can choose to notify the card's voters by email. Emails go only to confirmed voters who did not unsubscribe. Each mail has an unsubscribe link. [src:1 for closing the loop; details are a decision]
- **R29** A card can also notify voters when its item reaches a chosen status, for example "shipped". It is sent once per status change. [decision]

### Moderation
- **R30** Moderation mode `ideas`: new ideas are `pending` until a maker approves or rejects them. `ideas_and_votes`: votes are also held. `off`: nothing is held. [decision; sources say moderation exists but not how]
- **R31** Rejected ideas are hidden and create no feedback. A held vote creates no insight until approved. [decision]
- **R32** Makers can block a visitor email or domain. Blocked visitors cannot vote or submit. Their old votes are removed from the portal view and flagged in the workspace. [decision]
- **R33** Public ideas are not shown to other visitors in v1. Only the submitter's own ideas show to them. [decision]

### Visibility, domain, embed, SSO
- **R34** Visibility, per portal and independent of other portals: [src:1]
  - `hidden`: only admins and makers (the default).
  - `company`: signed-in workspace members of any role can browse, but cannot vote or submit as visitors.
  - `private_link`: anyone with the link; search engines are told not to index it.
  - `public`: anyone; search engines may index it.
- **R35** Portal SSO: when enabled, visitors sign in with the company identity provider (S14). A signed-in visitor skips email confirmation and their votes use their SSO identity. [src:1]
- **R36** A custom domain can be set. The admin proves ownership with a DNS record; Wareslate issues and renews a TLS certificate. Until verified, the default address works. [src:1 lists custom domain; method is a decision]
- **R37** A portal can be embedded in an iframe on listed origins only. Others are refused with a frame-ancestors header. [src:1; origin list is a decision]
- **R38** Branding: logo (best at about 72 by 72 px; also used as favicon), and colors for font, primary buttons, card background and page background. [src:1]
- **R39** The "powered by" badge can be hidden only on the top tier. [src:1]
- **R40** Localization: the portal's fixed UI texts (search box, buttons, vote question) can be translated per locale. Card names, descriptions and updates are not translated automatically. [src:1] Visitors get the locale from the browser, and a picker if more than one exists. [decision]

### Roles and tiers
- **R41** Admins manage portals (create, delete, visibility, domain, SSO, moderation). Makers edit tabs, sections, cards, updates and moderate. Contributors and viewers can browse a portal in `company` view and cannot edit. [src:1]
- **R42** Number of portals: 2 on the middle tier, unlimited on the top tier. Creating a portal over the limit is refused with a message. [src:2]

## User actions

| action | admin | maker | contributor | viewer | visitor |
|---|---|---|---|---|---|
| create, delete portal; visibility, domain, SSO | yes | no | no | no | no |
| branding, vote scale, idea switch | yes | yes | no | no | no |
| edit tabs, sections, cards | yes | yes | no | no | no |
| post card update, moderate | yes | yes | no | no | no |
| browse (when visible to them) | yes | yes | company view | company view | public views |
| vote, submit idea | yes | yes | no | no | after confirmation |

See [S01](S01-workspace-roles.md) for the full role model. Makers changing branding and the vote scale is a [decision].

## Acceptance tests
- **T1 (R6, R7)** Given a card with public name "Dark mode" and an empty public description, when a visitor opens it, then the name is "Dark mode" and no description shows.
- **T2 (R2)** Given a tab with 4 cards, when a maker deletes the tab, then the system asks where to move the cards and moves them.
- **T3 (R11, R13, R14)** Given a confirmed visitor votes "important" and then "critical" on a card, then one vote exists and the insight has 3 points.
- **T4 (R15)** Given vote counts are hidden, when a visitor opens a card, then no count shows, and the maker still sees the count in the workspace.
- **T5 (R18)** Given idea submission is off, when a visitor posts an idea through the API, then it is refused. Voting still works.
- **T6 (R19, R20)** Given a new visitor votes, when they have not clicked the email link, then the vote is pending and no insight exists. After 24 hours it is deleted.
- **T7 (R22, R25)** Given a visitor confirms and votes on feature F, then a note with source `portal` exists, linked to F. Given an approved idea, then a note exists with no item link.
- **T8 (R24)** Given user A already set "nice_to_have" on F in the app, when A votes "critical" in the portal, then A still counts 1 point.
- **T9 (R28)** Given 5 confirmed voters, one has unsubscribed, when the maker posts an update with notify on, then 4 emails are sent.
- **T10 (R30, R31)** Given mode `ideas`, when a visitor submits an idea, then it is pending, hidden from the visitor view and no note exists. When a maker approves it, then the note is created.
- **T11 (R34)** Given a portal is `hidden`, when an anonymous user opens its link, then access is denied. Given `private_link`, then it opens and the page asks search engines not to index it.
- **T12 (R35)** Given SSO is on, when a signed-in visitor votes, then no email confirmation is asked.
- **T13 (R37)** Given allowed origin A, when the portal is embedded from origin B, then the browser refuses it.
- **T14 (R42)** Given a workspace on the middle tier with 2 portals, when an admin creates a third, then it is refused.
- **T15 (R10)** Given an item that is not on any card, when anyone requests it through the portal API, then the answer is "not found".
- **T16 (R40)** Given locale `cs` has a translation for the button text, when a browser asks for `cs`, then the button shows the translation and card text is unchanged.

## Tier
Portal basics, voting, branding: paid tiers. 2 portals on the middle tier, unlimited on the top tier. Hide "powered by", SSO and moderation of votes: top tier. [src:2] for the counts and badge; the rest is a [decision]

## Open questions
1. Should visitors see other visitors' ideas and vote on them (public idea board)?
2. Should portals support more than four vote options, or numeric scales?
3. How should visitor email match an existing customer user, to avoid duplicates (R22, R26)?
4. Should card updates support scheduling?
5. Does a free tier get one portal?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S11).
