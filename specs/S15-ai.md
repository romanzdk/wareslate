# S15 AI assistant

## Scope
The AI assistant is a chat that works on top of the workspace. It can read, create and change objects, run saved playbooks, and study feedback. It covers:
- Chat, sub-agents, object changes with a preview, apply without review, and undo
- Playbooks and recurring runs
- Context library and custom instructions
- Feedback analysis: suggested excerpts, clusters, patterns, problem statements, feedback digest
- Spec writing, hand-off to coding agents, post-launch review
- Suggestions, suggested links, hybrid search
- AI units, roles, data-use policy

Wareslate choices: the model provider is the Claude API. The assistant uses the public API (S13) as its tool, so it can never do more than the user's own token allows. Embeddings live in pgvector (see docs/architecture.md). [decision]

Related modules:
- Roles and permissions: [S01](S01-workspace-roles.md)
- Items and product tree: [S02](S02-product-tree.md)
- Fields and statuses: [S03](S03-fields-statuses.md)
- Feedback, excerpts, evidence: [S04](S04-feedback.md)
- Accounts and contacts: [S05](S05-customers.md)
- Demand score: [S06](S06-prioritization.md)
- Strategy (objectives): [S07](S07-strategy.md)
- Views: [S09](S09-views.md)
- Pages and version history: [S10](S10-pages.md)
- Slack and other integrations: [S12](S12-integrations.md)
- Public API and the MCP server for coding agents: [S13](S13-api.md)
- Audit log and data security: [S14](S14-security.md)

## Entities

| entity | field | type | required | remark |
|---|---|---|---|---|
| ai_settings | workspace_id, enabled | bool | yes | default off; an admin switches it on (R1) |
| | max_subagents | int | yes | default 6 (R12) |
| | apply_without_review_limit | int | yes | default 1000 (R17) |
| chat | id, owner_id, title, created_at | | yes | seen only by its owner (R5) |
| chat_message | chat_id, role `user \| assistant \| tool`, content, mentions, attachments | | yes | |
| | reasoning | text | no | shown to the user (R6) |
| | followups | list of up to 3 strings | no | |
| | state | enum `queued \| running \| done \| stopped` | yes | R9 |
| ai_run | id, chat_id or recurring_run_id, subagent_of, status, ai_units_used | | yes | one per assistant turn or sub-agent |
| change_set | id, chat_id, mode `review \| apply_without_review`, status `previewed \| applied \| partly_reverted \| reverted \| rejected` | | yes | the object changes that came from one request |
| change_item | change_set_id, object_id, op `create \| update \| delete`, before (json), after (json), state | | yes | `before` is what makes undo possible (R19) |
| playbook | id, name, instructions, scripts, scope `mine \| workspace \| built_in`, owner_id | | yes | |
| scheduled_run | id, playbook_id, schedule (cron + timezone), output `slack \| email`, target, owner_id, enabled | | yes | the schedule of a recurring run |
| recurring_run | scheduled_run_id, started_at, status, output_ref | | | one row per execution |
| context_source | id, kind `url \| text \| file`, title, content, tag `persona \| competitor \| other`, added_by | | yes | context library (R30) |
| instruction | scope `mine \| space \| workspace`, scope_id, text | | yes | custom instructions (R32) |
| embedding | object_kind, object_id, model, vector, content_hash | | yes | pgvector; rebuilt when content changes |
| cluster | id, name, summary, status `suggested \| accepted \| dismissed`, created_by `ai \| user` | | yes | a group of feedback entries |
| cluster_member | cluster_id, feedback_id, score | | | |
| pattern | id, statement, cluster_id, created_at | | yes | |
| citation | owner_kind `pattern \| digest_claim \| problem`, owner_id, feedback_id, quote_range | | yes | at least one per claim (R36) |
| problem | id, statement, feedback_count, account_count, evidence_score, created_by `ai \| user` | | yes | a problem statement, R39 |
| digest | id, scope (filter), generated_at, claims[] | | yes | each claim has citations (R38) |
| suggestion | object_id, kind `change \| link \| excerpt`, payload, status `open \| applied \| dismissed` | | yes | R46 |
| ai_units_ledger | workspace_id, member_id, run_id, units, at | | yes | R48 |
| ai_units_plan | tier, monthly_units_per_editor, extra_pack | | yes | R49 |

## Rules

### Access, switch and data use
- **R1** A new workspace starts with the assistant off. A workspace admin switches it on in settings. While it is off, no AI feature runs and nothing is sent to the model provider. [src:2]
- **R2** Readers cannot see or use the assistant. Collaborators can chat, but only within what their own role allows (R3). Editors and admins can use all of it. [src:1][src:4]
- **R3** The assistant acts as the signed-in user. It calls the public API (S13) with that user's rights, so it cannot read or change anything the user cannot. [decision]
- **R4** Customer data does not train any model, neither Wareslate's nor the provider's. The provider contract must ban training and must not let the provider keep data longer than it takes to answer. Only the data a request needs is sent. [src:1]
- **R5** Chats belong to their owner and cannot be shared. Admins cannot open them in the product. Deleting a chat deletes its messages. [src:1] The admin rule and the deletion rule are a [decision]
- **R5a** Each assistant change and each recurring run is written to the audit log (S14), with the user, the run and the objects touched. [decision]

### Chat
- **R6** The user can type, speak, or attach files. Each answer lists its reasoning steps and the tools it called. [src:1] Allowed file types (text, PDF, CSV, images) and the size cap (10 MB per file) are a [decision]
- **R7** Typing `@` opens a picker for feedback entries, views and templates. A picked object adds its content to the request, but only if the user can see it. [src:1] Picking other kinds of objects (items, accounts, pages) is a [decision]
- **R8** Under each answer the assistant offers up to 3 next-step prompts. A new empty chat offers 3 starter prompts. [src:1]
- **R9** While an answer is running, the user has two choices for a new message: queue it (it starts when the current turn ends) or steer (it joins the running turn and the assistant adjusts). The user can also stop the turn. [src:1]
- **R10** Answers about workspace data must rest on results returned by the API. When the data is not found, the assistant says that. It must not make up names, numbers or quotes. [decision]
- **R11** A chat has a size limit. Close to it, older messages are condensed and the user is told. [decision]

### Sub-agents
- **R12** For a big task the assistant may split the work into parts that run side by side as sub-agents. At most **6** run at once; an admin can lower this (1–6). Parts beyond the limit wait in line. [src:1] Lowering the limit is a [decision]
- **R13** The user can see which sub-agents started and read their reasoning. A sub-agent is not a separate chat. [src:1]
- **R14** A sub-agent has the same rights as its parent and cannot start sub-agents. Its AI units are counted on the same chat. [decision]
- **R15** When a sub-agent fails, the parent is told and names the failed part in its answer. It does not paper over the gap. [decision]

### Creating and changing objects
- **R16** The user can ask in plain words to create, update, move, link or archive objects. By default (review mode) the assistant first shows a preview of every change: object, field, old value, new value. Nothing is saved until the user accepts. The user can accept everything, accept part, or reject. [src:1]
- **R17** Apply without review saves changes with no preview step. It is chosen per chat, starts off, and switches off again when the chat ends. One request can change at most **1,000** objects. A bigger plan is refused, and the assistant asks the user to split it or use review mode. [src:4][decision on per-chat scope and the stop]
- **R18** With apply without review, the chat shows a side-by-side diff of what changed. [src:1]
- **R19** Every change set can be undone. The system keeps the old value of each field it touched. "Revert" puts them back, for the whole set or for single objects. [src:1][src:4] Choosing between whole set and single objects is a [decision]
- **R20** Reverting an object is blocked when someone else edited the same field after the assistant. It is reported as "conflict" and left alone. The rest of the set is reverted. [decision]
- **R21** Reverting a create archives the new object. Reverting a delete brings it back. After any assistant change, deletion stays a soft delete (archive) for 30 days. [decision]
- **R22** The assistant runs the same validation as the API (S02, S03). A change that fails is marked failed in the preview and is not applied; the rest of the set can still be. [decision]
- **R23** Edits the assistant makes to a page are marked on the page. In review mode they stay marked until the user accepts or rejects them. The page's own version history (S10) can also bring back the old text. [src:1]

### Playbooks and recurring runs
- **R24** A playbook is a saved workflow made of a name, instructions and optional scripts. Its scope is mine, workspace or built-in. Built-in playbooks ship with Wareslate and cannot be edited. [src:1]
- **R25** In the list, the order is: the user's own playbooks, then workspace ones, then built-in ones. A user can promote one of their own to workspace scope. Only editors and admins can create or change workspace playbooks. [src:1] Promotion and who may edit are a [decision]
- **R26** Scripts in a playbook run in a sandbox. The only network they can reach is the public API (S13). They have a 60-second time limit and a memory limit, and they cannot read secrets. [decision]
- **R27** A recurring run is one playbook plus a schedule (cron with a time zone). Its output goes to Slack (S12) or email. Example: every Monday a digest of new feedback is posted to a channel. [src:1]
- **R28** A recurring run uses the rights of the user who set it up. If that user is removed, becomes a reader, or loses the access the run needs, the run pauses and its owner and the admins are told. [decision]
- **R29** A recurring run spends AI units like a chat (R48). When units run out, the run is skipped and the owner is told. A run never fires more than once an hour. Every execution is logged with its status. [decision]
- **R29a** A recurring run may apply without review only if its playbook is marked read-only. A playbook that changes objects needs a manual check: the run stores the change set as `previewed` and notifies the owner. [decision]

### Context library and custom instructions
- **R30** The context library holds public web pages, pasted text, files, personas and competitors. The assistant can draw on it in any request. Only editors and admins change it. [src:1][src:2]
- **R31** A source added from a URL is fetched when added and then refreshed every week. The person who owns it can refresh it by hand. A source is capped at 100 000 characters. [decision]
- **R32** Custom instructions are text that shapes how the assistant answers. They come in three scopes: mine, space and workspace. Anyone who has the assistant edits their own. Space admins edit their space's. Workspace admins edit the workspace one. [src:1] Who may edit is a [decision]
- **R33** When instructions disagree, the narrower scope wins: mine over space over workspace. No instruction can switch off the rules of this spec (rights, data use, citations). [decision]

### Feedback analysis
- **R34** Suggested excerpts: for a feedback entry, the assistant marks the key customer quotes as suggested excerpts. The user accepts one (it becomes a normal excerpt, see S04) or dismisses it. No excerpt exists without a user action. [src:2] Accept and dismiss are a [decision]
- **R35** Clusters group related feedback by meaning (embedding clustering, with a name and summary written by Claude). A cluster shows how many feedback entries and accounts it holds. Users can rename, merge, split, accept or dismiss a cluster, and move entries between clusters. Clusters are recomputed every day and on request. [src:1][src:4] Editing and schedule are a [decision]
- **R36** Patterns are statements the assistant draws from feedback, such as "Admins struggle to find export". Each pattern needs at least one citation: a feedback entry and the quoted part. A statement without a citation is dropped. [src:1]
- **R37** A citation opens the source entry at the quoted text. If the entry is deleted or the user cannot see it, the citation reads "source not available" and shows no text. [decision]
- **R38** A feedback digest covers a filter (time, cohort, cluster, product area). Each claim in it needs at least one citation (R37). A claim that fails this check is taken out before the digest is shown. The digest states how many feedback entries it used. [src:1] The check and the count are a [decision]
- **R39** A problem statement describes a problem and its evidence: the number of feedback entries, the number of distinct accounts, and the sum of demand score points from linked feedback (S06). Problem statements are ranked by evidence. The assistant or a person can create them, and people can edit them. [src:1][src:2] Ranking by this exact calculation is a [decision]
- **R40** Problem statements can draw on feedback, competitor facts entered in the context library (R30) and analytics data (R42). The assistant keeps the source of every fact. [src:2]
- **R41** The assistant can link a problem statement to an item, an objective (S07) or a cluster. Linking is an ordinary change set (R16). [decision]

### Spec writing and post-launch review
- **R42** Spec writing: from a problem statement or a short brief, the assistant writes a spec as a page (S10). It can use strategy, the context library and, when a code repository is connected (Azure DevOps, Bitbucket, GitHub or GitLab), the code, to name real modules and files. [src:1][src:2][src:3]
- **R43** Repository access is read-only and set per workspace by an admin, using the admin's own token. The assistant reads only what a request needs. It keeps no copy of the code except short excerpts that appear in the spec. [decision]
- **R44** Hand-off: a spec can be passed to a coding agent. Wareslate gives the agent a link to the spec and the MCP server (S13). The agent can then read the spec, change item status and add comments. Wareslate does not run the coding agent. [src:3][src:5] Link-based hand-off and "does not run" are a [decision]
- **R45** Post-launch review: for a shipped item, the assistant builds a review page at three checkpoints: day 7, day 14 and day 30 after release. It sets analytics (a connected tool or CSV) next to feedback that came in since launch, and cites both. With no analytics data it uses feedback alone and says so. [src:1][src:2] The CSV fallback is a [decision]

### Suggestions and search
- **R46** For an object, the assistant can suggest field changes and links to other objects. Suggestions appear next to the object. Making a suggestion costs no AI units. If the assistant applies one, it costs AI units (R48); if a person applies it, it costs none. [src:1]
- **R47** Search is hybrid: keyword search (Postgres full-text and trigram) and search by meaning (pgvector) are joined by reciprocal rank fusion into one list. It respects access rights. Embeddings are refreshed within 5 minutes of a content change. Keyword search must keep working while the assistant is off. [src:2] The join method, the delay and the fallback are a [decision]
- **R47a** Suggested links need a product tree. With no items, nothing is suggested. [src:2]

### AI units and limits
- **R48** AI use is metered in AI units. One unit stands for a fixed amount of model tokens, with a price table in config. Each chat turn, sub-agent, recurring run, digest, cluster run and spec spends units. Each answer shows the units it used. [src:4] The exact size of a unit is a [decision]
- **R49** Each tier gives a monthly number of AI units per editor (see Tier). Units are tracked per workspace pool and per member. They reset on the first day of each month, and unused units do not roll over. [src:4] The pool and the no-rollover rule are a [decision]
- **R50** An admin can see use by member and by feature, and can set a monthly cap per member. At 80% of the pool, admins are told. At 100%, chat and recurring runs stop with a clear message while the rest of the product keeps working. [decision]
- **R51** Admins can buy extra packs of AI units. Pack size and price depend on the tier. Pack units are spent after the monthly units and expire after 12 months. [src:4] Expiry is a [decision]
- **R52** A turn that is cut short is charged only for the tokens already spent. A turn that fails through the system's fault (provider error) is not charged. [decision]
- **R53** On a self-hosted install, the operator can switch off the unit limits. Use is still recorded. [decision]

## User actions

| action | reader | collaborator | editor | admin |
|---|---|---|---|---|
| see or use the assistant | no | yes (R2) | yes | yes |
| chat, mention, attach files | no | yes | yes | yes |
| accept or reject a preview | no | only for changes they may make | yes | yes |
| apply without review | no | no | yes | yes |
| revert a change set | no | own sets | own sets | any set |
| my playbooks, my instructions | no | yes | yes | yes |
| workspace playbooks, context library | no | no | yes | yes |
| recurring runs | no | own, read-only playbooks | yes | yes |
| clusters, patterns, problem statements: view | no | yes | yes | yes |
| clusters, problem statements: edit | no | no | yes | yes |
| feedback digest | no | run | run | run |
| spec writing and hand-off | no | no | yes | yes |
| connect a repository, switch AI on, set caps, buy AI units | no | no | no | yes |

The collaborator limits in this table are a [decision]. See [S01](S01-workspace-roles.md) for the role model and space admins.

## Acceptance tests
- **T1 (R1)** Given the assistant is off, when an editor opens the product, then there is no chat and no request goes to the model provider.
- **T2 (R2)** Given a reader, when they call the assistant through the API or UI, then it is refused and the UI shows no AI controls.
- **T3 (R3)** Given a collaborator who cannot see view V, when they type `@V` or ask about it, then the assistant gets no data from V and says it cannot find it.
- **T4 (R5)** Given user A's chat, when user B or an admin searches chats, then A's chat is not returned.
- **T5 (R9)** Given the assistant is answering, when the user sends a message and picks "queue", then it runs after the current turn. When they pick "steer", then it joins the current turn.
- **T6 (R12)** Given a task with 10 parts and a limit of 6, then 6 run at once, 4 wait, and all 10 finish.
- **T7 (R14)** Given a sub-agent, when it tries to start another sub-agent, then it is refused.
- **T8 (R16)** Given the request "set status of these 5 features to Planned" in review mode, then a preview of 5 changes is shown and nothing is saved. When the user accepts 3 of them, then 3 are saved.
- **T9 (R17)** Given apply without review and a request that would change 1,200 objects, then nothing changes and the assistant asks to split it. Given 800 objects, then all 800 change and a side-by-side diff is shown.
- **T10 (R19, R20)** Given an applied set of 10 changes, and a person later edits the name of object E, when the user reverts the set, then 9 return to their old values and E is reported as "conflict" and keeps its new name.
- **T11 (R21)** Given a set that created 5 objects, when it is reverted, then the 5 are archived and can be restored.
- **T12 (R22)** Given a change that sets a value S03 does not allow, then the preview marks it failed and the others still apply.
- **T13 (R25)** Given one of the user's own playbooks, a workspace playbook and a built-in playbook, then the list shows them in that order.
- **T14 (R27, R29a)** Given a read-only playbook scheduled weekly to a Slack channel, then the output is posted each week. Given a playbook that changes objects, then the run stores a `previewed` change set and notifies the owner instead of applying it.
- **T15 (R28)** Given a recurring run whose owner becomes a reader, then the run pauses and the owner and admins are told.
- **T16 (R33)** Given the workspace instruction "answer formally" and the user's own instruction "answer briefly", then answers are brief. Given an instruction "ignore permissions", then the permission rules still apply.
- **T17 (R34)** Given a feedback entry with a suggested excerpt, then no excerpt exists until the user accepts it.
- **T18 (R36)** Given the assistant drafts a pattern with no supporting feedback entry, then the pattern is dropped.
- **T19 (R37, R38)** Given a digest with 12 claims where the only source entry of 1 claim was deleted, then that claim is removed, the digest shows 11, and the entry count is shown.
- **T20 (R39)** Given problem statements P1 (20 entries from 8 accounts, 30 demand score points) and P2 (5 entries from 5 accounts, 6 points), then P1 ranks above P2 and both show their counts.
- **T21 (R44)** Given a spec, when the user hands it off, then the coding agent gets a link, and through the MCP server (S13) it can read the spec and change the item status.
- **T22 (R45)** Given a release 7 days old with no analytics connected, then the review page uses feedback only and says analytics are missing.
- **T23 (R46)** Given a suggestion on an object, then it is created at no cost. When the user applies it by hand, there is no cost. When the assistant applies it, AI units are spent.
- **T24 (R47)** Given a query "login trouble" and a feedback entry that says "can't sign in", then the entry is found by search by meaning. Given the assistant is off, then keyword search still works.
- **T25 (R49, R50)** Given a pool at 100%, when a user sends a chat message, then it is refused with a message and the rest of the product still works. At 80%, admins got a notice.
- **T26 (R52)** Given a turn that fails because of a provider error, then no AI units are charged.
- **T27 (R4, R5a)** Given any assistant change, then an audit log entry exists with the user, the run id and the objects touched.

## Tier
- Chat, playbooks, suggestions, hybrid search, problem statements, spec writing: all tiers, within their AI units. [decision]
- Clusters: paid tiers; in the commercial hosted product probably the top tier only. [src:2][decision]
- AI units per editor per month: a fixed amount per editor on each paid tier; a larger pooled amount on the top tier. [decision] Exact numbers are open (open question 1).
- Self-hosted open-source install: no tiers, and no unit limit by default (R53). [decision]

## Open questions
1. What is one AI unit worth, and how many units does a typical chat turn, digest or spec use? Measure on real prompts first.
2. Should apply without review be allowed for collaborators, or for recurring runs that change data (R29a)?
3. Should admins be able to read chats for legal reasons (R5), with a visible audit entry?
4. Should self-hosters be able to use other model providers later? The architecture only assumes the Claude API.
5. Should clusters be live (updated as feedback arrives) instead of daily (R35)?

## Sources
Provenance for `[src:N]` tags is kept in private research files (S15).
