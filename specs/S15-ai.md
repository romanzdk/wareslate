# S15 AI assistant

## Scope
The AI assistant: a chat over the workspace that can read, create and change entities, run saved workflows, and analyse feedback. It covers:
- Chat, sub-agents, entity changes with preview, auto-accept and undo
- Skills and scheduled tasks
- Company knowledge and custom instructions
- Feedback analysis: highlights, themes, findings, opportunities, voice-of-customer report
- Spec writing, hand-off to coding agents, post-launch review
- Suggestions, suggested links, hybrid search
- Credits, roles, data-use policy

Wareslate choices: Claude API is the model provider. The assistant calls the public API as a tool (S13), so it can never do more than the user's own token. Embeddings are stored in pgvector (see docs/architecture.md). [decision]

Related modules:
- Roles and permissions: [S01](S01-workspace-roles.md)
- Items and hierarchy: [S02](S02-hierarchy.md)
- Fields and statuses: [S03](S03-fields-statuses.md)
- Feedback, highlights, links: [S04](S04-feedback-insights.md)
- Customers: [S05](S05-customers.md)
- Impact score: [S06](S06-prioritization.md)
- Strategy (objectives): [S07](S07-strategy.md)
- Boards: [S09](S09-boards.md)
- Documents and version history: [S10](S10-documents.md)
- Slack and other integns: [S12](S12-integrations.md)
- Public API and the MCP server for coding agents: [S13](S13-api.md)
- Audit log and data security: [S14](S14-security.md)

## Entities

| entity | field | type | required | notes |
|---|---|---|---|---|
| ai_settings | workspace_id, enabled | bool | yes | default off; an admin turns it on (R1) |
| | max_subagents | int | yes | default 6 (R12) |
| | auto_accept_limit | int | yes | default 1000 (R17) |
| chat | id, owner_id, title, created_at | | yes | visible only to the owner (R5) |
| chat_message | chat_id, role `user \| assistant \| tool`, content, mentions, attachments | | yes | |
| | reasoning | text | no | shown to the user (R6) |
| | followups | list of up to 3 strings | no | |
| | state | enum `queued \| running \| done \| stopped` | yes | R9 |
| ai_run | id, chat_id or task_run_id, subagent_of, status, credits_used | | yes | one per assistant turn or sub-agent |
| change_set | id, chat_id, mode `manual \| auto`, status `previewed \| applied \| partly_reverted \| reverted \| rejected` | | yes | a group of entity changes from one request |
| change_item | change_set_id, entity_id, op `create \| update \| delete`, before (json), after (json), state | | yes | `before` makes undo possible (R19) |
| skill | id, name, instructions, scripts, scope `personal \| workspace \| built_in`, owner_id | | yes | |
| scheduled_task | id, skill_id, schedule (cron + timezone), output `slack \| email`, target, owner_id, enabled | | yes | |
| task_run | scheduled_task_id, started_at, status, output_ref | | | |
| knowledge_source | id, kind `url \| text \| file`, title, content, tag `persona \| competitor \| other`, added_by | | yes | company knowledge (R30) |
| instruction | scope `personal \| teamspace \| workspace`, scope_id, text | | yes | custom instructions (R32) |
| embedding | entity_kind, entity_id, model, vector, content_hash | | yes | pgvector; rebuilt when content changes |
| theme | id, name, summary, status `suggested \| accepted \| dismissed`, created_by `ai \| user` | | yes | cluster of feedback |
| theme_member | theme_id, note_id, score | | | |
| finding | id, statement, theme_id, created_at | | yes | |
| citation | owner_kind `finding \| report_claim \| opportunity`, owner_id, note_id, quote_range | | yes | at least one per claim (R36) |
| opportunity | id, problem_statement, feedback_count, customer_count, evidence_score, created_by `ai \| user` | | yes | R39 |
| voc_report | id, scope (filter), generated_at, claims[] | | yes | each claim has citations (R38) |
| suggestion | entity_id, kind `change \| link \| highlight`, payload, status `open \| applied \| dismissed` | | yes | R46 |
| credit_ledger | workspace_id, member_id, run_id, credits, at | | yes | R48 |
| credit_plan | tier, monthly_credits_per_member, extra_pack | | yes | R49 |

## Rules

### Access, switch and data use
- **R1** The assistant is off for a new workspace. A workspace admin turns it on in settings. While it is off, no AI feature runs and no data goes to the model provider. [src:2]
- **R2** Viewers can neither see nor use the assistant. Contributors can use chat but only inside what their own role allows (R3). Makers and admins can use everything. [src:1][src:4]
- **R3** The assistant acts as the signed-in user. It calls the public API (S13) with that user's permissions, so it cannot read or change anything the user cannot. [decision]
- **R4** Customer data is never used to train models, by Wareslate or by the model provider. The provider contract must forbid training and must not allow storing data longer than needed to answer. Data sent to the provider is limited to what the request needs. [src:1]
- **R5** Chats are private to their owner. They cannot be shared. Admins cannot read them in the product. Deleting a chat deletes its messages. [src:1] Admin access and deletion rules are a [decision]
- **R5a** Every assistant change and every scheduled run is written to the audit log (S14) with the user, the run and the entities touched. [decision]

### Chat
- **R6** The user can type, dictate by voice, and attach files. The answer shows its reasoning steps and the tools it used. [src:1] Which file types are allowed (text, PDF, CSV, images) and the size limit (10 MB per file) are a [decision]
- **R7** Typing `@` lets the user mention feedback notes, boards and templates. A mention puts that object's content in the context of the request, only if the user can see it. [src:1] Mentioning other entity types (items, companies, documents) is a [decision]
- **R8** After each answer the assistant offers up to 3 suggested follow-up prompts. An empty chat shows 3 suggested starting prompts. [src:1]
- **R9** While the assistant is answering, the user can queue another message (it runs after the current turn) or steer (it is added to the current turn and the assistant adjusts). The user can also stop the turn. [src:1]
- **R10** Answers about workspace data must be based on tool results from the API. If the assistant cannot find data, it says so. It must not invent entity names, numbers or quotes. [decision]
- **R11** A chat has a context limit. When it is near the limit, old messages are summarized and the user sees a notice. [decision]

### Sub-agents
- **R12** For a large task the assistant may split work into sub-tasks that run in parallel as sub-agents. The workspace maximum is **6** at one time, an admin can lower it (1–6). Extra sub-tasks wait in a queue. [src:1] Lowering is a [decision]
- **R13** The user can see which sub-agents started and read their reasoning. Sub-agents are not separate chats. [src:1]
- **R14** A sub-agent has the same permissions as its parent and cannot start sub-agents itself. It counts its credits to the same chat. [decision]
- **R15** If a sub-agent fails, the parent is told and says in the answer which part failed. It does not hide the gap. [decision]

### Creating and changing entities
- **R16** The user can ask in plain language to create, update, move, link or archive entities. By default (manual mode) the assistant first shows a preview of every change: entity, field, old value, new value. Nothing is saved until the user accepts. The user can accept all, accept some, or reject. [src:1]
- **R17** Auto-accept mode applies changes without a preview. It is chosen per chat, is off by default, and turns off when the chat ends. One request can change at most **1,000** entities. If the plan is larger, the assistant refuses and asks to split it or use manual mode. [src:4][decision on per-chat scope and the stop]
- **R18** In auto-accept mode the chat shows a before-and-after view of what changed. [src:1]
- **R19** Every change set can be undone. The system saves the old value of each field it touched. "Revert" restores them for the whole set or for single entities. [src:1][src:4] Whole-set and per-entity choice is a [decision]
- **R20** Revert is blocked for an entity when someone else changed the same field after the assistant did. It is shown as "conflict" and left as is. The rest of the set is reverted. [decision]
- **R21** Reverting a create archives the new entity. Reverting a delete restores it. Deleting stays a soft delete (archive) for 30 days after any assistant change. [decision]
- **R22** The assistant checks the same validation rules as the API (S02, S03). A change that fails is shown as failed in the preview and is not applied. The rest of the set still can be. [decision]
- **R23** Document edits from the assistant are marked in the document. In manual mode they stay highlighted until the user accepts or rejects. The document's own version history (S10) can also restore the old text. [src:1]

### Skills and scheduled tasks
- **R24** A skill is a saved workflow: a name, instructions and optional scripts. Its scope is personal, workspace or built-in. Built-in skills are shipped with Wareslate and cannot be edited. [src:1]
- **R25** The skill list shows personal skills first, then workspace skills, then built-in ones. A user can promote a personal skill to workspace scope. Only makers and admins can create or edit workspace skills. [src:1] Promotion and who may edit are a [decision]
- **R26** Scripts in a skill run in a sandbox with no network except the public API (S13), a time limit of 60 seconds and a memory limit. A script cannot read secrets. [decision]
- **R27** A scheduled task is one skill plus a schedule (cron with a time zone). Its result goes to Slack (S12) or email. Example: a weekly feedback summary in a Slack channel. [src:1]
- **R28** A scheduled task runs with the permissions of the user who created it. If that user is removed, made a viewer, or loses the needed access, the task pauses and its owner and the admins are told. [decision]
- **R29** A task run uses credits like a chat (R48). If credits run out, the run is skipped and the owner is told. A task never runs more often than once per hour. Each run is logged with its status. [decision]
- **R29a** Scheduled tasks run in auto-accept mode only if their skill is marked read-only. A skill that changes entities needs manual review: the task saves the change set as `previewed` and notifies the owner. [decision]

### Company knowledge and custom instructions
- **R30** Company knowledge holds public web pages, pasted text, files, personas and competitors. The assistant can use it in every request. Only makers and admins edit it. [src:1][src:2]
- **R31** A knowledge source from a URL is fetched once when added and refreshed weekly. The editor can refresh it by hand. Content is limited to 100 000 characters per source. [decision]
- **R32** Custom instructions are text that changes how the assistant answers. There are three scopes: personal, teamspace and workspace. Anyone with the assistant can edit their personal one. Teamspace admins edit their teamspace's. Workspace admins edit the workspace one. [src:1] Who edits is a [decision]
- **R33** If instructions conflict, the more specific one wins: personal over teamspace over workspace. Instructions can never turn off the rules in this spec (permissions, data use, citations). [decision]

### Feedback analysis
- **R34** AI-suggested highlights: for a feedback note, the assistant marks key customer quotes as suggested highlights. The user accepts one (it becomes a normal highlight in S04) or dismisses it. Nothing becomes a highlight without a user action. [src:2] Accept/dismiss is a [decision]
- **R35** Themes group related feedback by meaning (embedding clustering, with a Claude-written name and summary). A theme shows how many notes and customers it holds. Users can rename, merge, split, accept or dismiss a theme, and move notes between them. Themes are recomputed daily and when a user asks. [src:1][src:4] Editing and schedule are a [decision]
- **R36** Findings are statements the assistant draws from feedback, for example "Admins struggle to find export". Each finding must have at least one citation: a feedback note and the quoted part. A statement with no citation is dropped. [src:1]
- **R37** A citation opens the original note at the quoted text. If the note is deleted or the user cannot see it, the citation shows "source not available" and does not show the text. [decision]
- **R38** A voice-of-customer report covers a filter (time, segment, theme, product area). Every claim in it has at least one citation (R37). A claim that fails the check is removed before the report is shown. The report states how many notes it used. [src:1] The check and the count are a [decision]
- **R39** An opportunity is a problem statement with its evidence: number of feedback notes, number of distinct customers, and the sum of impact points from linked notes (S06). Opportunities are ranked by evidence. They can be made by the assistant or by hand, and edited by hand. [src:1][src:2] Ranking by that exact formula is a [decision]
- **R40** Opportunities can use feedback, entered competitor facts (R30) and analytics data (R42). The assistant keeps the source of each fact. [src:2]
- **R41** The assistant can link an opportunity to an item, an objective (S07) or a theme. Linking is a normal change set (R16). [decision]

### Spec writing and post-launch review
- **R42** Spec writing: from an opportunity or a short brief, the assistant writes a spec as a document (S10). It can use strategy, company knowledge and, if a repository is connected (GitHub, GitLab, Bitbucket or Azure DevOps), the code to name real modules and files. [src:1][src:2][src:3]
- **R43** Repository access is read-only, per workspace, set by an admin with a token of their own. The assistant reads only what it needs per request and does not store a copy of the code except short excerpts used in the spec. [decision]
- **R44** Hand-off: a spec can be sent to a coding agent. Wareslate gives the agent a link to the spec and the MCP server (S13). The agent can then read the spec, update item status and add comments. Wareslate does not run the coding agent. [src:3][src:5] Link-based hand-off and "does not run" are a [decision]
- **R45** Post-launch review: for a released item, the assistant builds an impact document 7, 14 and 30 days after release. It compares analytics (connected tool or CSV) with feedback received since launch, and cites both. If there is no analytics data, it uses feedback only and says so. [src:1][src:2] CSV fallback is a [decision]

### Suggestions and search
- **R46** For an entity, the assistant can suggest field changes and links to other entities. Suggestions are shown beside the entity. Creating suggestions costs no credits. Applying one by the assistant costs credits (R48), and applying it by hand costs none. [src:1]
- **R47** Search is hybrid: keyword search (Postgres full-text and trigram) and meaning search (pgvector) are merged with reciprocal rank fusion into one list. It respects permissions. Embeddings are updated within 5 minutes of a content change. Keyword-only search must work when the assistant is off. [src:2] Merge method, delay and the off-switch fallback are [decision]
- **R47a** The suggested-link feature needs an item hierarchy to exist. With no items, it shows nothing. [src:2]

### Credits and limits
- **R48** AI use is metered in credits. One credit is a fixed amount of model tokens, with a price table kept in config. Each chat turn, sub-agent, scheduled run, report, theme run and spec uses credits. The user sees credits used on each answer. [src:4] Exact meaning of a credit is a [decision]
- **R49** Each tier gives a monthly number of credits per maker (see Tier). Credits are counted per workspace pool and per member. They reset on the first day of the month. Unused credits do not carry over. [src:4] Pool and carry-over are a [decision]
- **R50** An admin can see usage by member and by feature and can set a monthly cap per member. At 80% of the pool, admins are told. At 100%, chat and tasks stop with a clear message, and the rest of the product keeps working. [decision]
- **R51** Admins can buy extra credit packs. The pack size and price are set per tier. Pack credits are used after the monthly credits and expire after 12 months. [src:4] Expiry is a [decision]
- **R52** An unfinished turn charges only the tokens already used. A failed turn that the system caused (provider error) charges nothing. [decision]
- **R53** In a self-hosted install the operator can turn off credit limits. Usage is still recorded. [decision]

## User actions

| action | viewer | contributor | maker | admin |
|---|---|---|---|---|
| see or use the assistant | no | yes (R2) | yes | yes |
| chat, mention, attach files | no | yes | yes | yes |
| accept or reject a preview | no | only for changes they may do | yes | yes |
| auto-accept mode | no | no | yes | yes |
| revert a change set | no | own sets | own sets | any set |
| personal skills, personal instructions | no | yes | yes | yes |
| workspace skills, company knowledge | no | no | yes | yes |
| scheduled tasks | no | own, read-only skills | yes | yes |
| themes, findings, opportunities: view | no | yes | yes | yes |
| themes, opportunities: edit | no | no | yes | yes |
| voice-of-customer report | no | run | run | run |
| spec writing and hand-off | no | no | yes | yes |
| connect a repository, turn AI on, set caps, buy credits | no | no | no | yes |

Contributor limits in this table are a [decision]. See [S01](S01-workspace-roles.md) for the role model and teamspace admins.

## Acceptance tests
- **T1 (R1)** Given the assistant is off, when a maker opens the product, then there is no chat and no request goes to the model provider.
- **T2 (R2)** Given a viewer, when they call the assistant through the API or UI, then it is refused and the UI shows no AI controls.
- **T3 (R3)** Given a contributor who cannot see board B, when they type `@B` or ask about it, then the assistant has no data from B and says it cannot find it.
- **T4 (R5)** Given user A's chat, when user B or an admin searches chats, then A's chat is not returned.
- **T5 (R9)** Given the assistant is answering, when the user sends a message and chooses "queue", then it runs after the current turn. When they choose "steer", then it is added to the current turn.
- **T6 (R12)** Given a task with 10 sub-tasks and the maximum is 6, then 6 run at once, 4 wait, and all 10 finish.
- **T7 (R14)** Given a sub-agent, when it tries to start another sub-agent, then it is refused.
- **T8 (R16)** Given the request "set status of these 5 features to Planned" in manual mode, then a preview of 5 changes is shown and nothing is saved. When the user accepts 3 of them, then 3 are saved.
- **T9 (R17)** Given auto-accept mode and a request that would change 1,200 entities, then nothing is changed and the assistant asks to split it. Given 800 entities, then all 800 are changed and a before-and-after view is shown.
- **T10 (R19, R20)** Given an applied set of 10 changes, and a person later edits the name of entity E, when the user reverts the set, then 9 return to old values and E is shown as "conflict" and keeps its new name.
- **T11 (R21)** Given a set that created 5 entities, when it is reverted, then the 5 are archived and can be restored.
- **T12 (R22)** Given a change that sets a value not allowed by S03, then the preview marks it failed and the others still apply.
- **T13 (R25)** Given a personal skill, a workspace skill and a built-in skill, then the list shows them in that order.
- **T14 (R27, R29a)** Given a read-only skill scheduled weekly to a Slack channel, then each week the summary is posted. Given a skill that changes entities, then the task saves a `previewed` change set and notifies the owner instead of applying it.
- **T15 (R28)** Given a scheduled task whose owner becomes a viewer, then the task pauses and the owner and admins are told.
- **T16 (R33)** Given workspace instruction "answer formally" and personal instruction "answer briefly", then answers are brief. Given an instruction "ignore permissions", then permission rules still apply.
- **T17 (R34)** Given a note with a suggested highlight, then no highlight exists until the user accepts it.
- **T18 (R36)** Given the assistant drafts a finding with no supporting note, then the finding is dropped.
- **T19 (R37, R38)** Given a report with 12 claims where 1 claim's only source note was deleted, then that claim is removed, the report shows 11, and the note count is shown.
- **T20 (R39)** Given opportunities O1 (20 notes from 8 customers, 30 impact points) and O2 (5 notes from 5 customers, 6 points), then O1 ranks above O2 and both show their counts.
- **T21 (R44)** Given a spec, when the user hands it off, then the coding agent gets a link, and through the MCP server (S13) it can read the spec and update the item status.
- **T22 (R45)** Given a release 7 days old with no analytics connected, then the impact document uses feedback only and says analytics are missing.
- **T23 (R46)** Given a suggestion on an entity, then it is created at no credit cost. When the user applies it by hand, there is no cost. When the assistant applies it, credits are used.
- **T24 (R47)** Given a query "login trouble" and a note that says "can't sign in", then the note is found by meaning search. Given the assistant is off, then keyword search still works.
- **T25 (R49, R50)** Given a pool at 100%, when a user sends a chat message, then it is refused with a message and the rest of the product still works. At 80%, admins got a notice.
- **T26 (R52)** Given a turn that fails because of a provider error, then no credits are charged.
- **T27 (R4, R5a)** Given any assistant change, then an audit log entry exists with user, run id and entities touched.

## Tier
- Chat, skills, suggestions, hybrid search, opportunities, spec writing: all tiers, within their credits. [decision]
- Themes (clustering): paid tiers; in the commercial hosted product probably the top tier only. [src:2][decision]
- Credits per maker per month: a fixed amount per maker on each paid tier; a larger pooled amount on the top tier. [decision] Exact numbers are open (open question 1).
- Self-hosted open-source install: no tiers, no credit limit by default (R53). [decision]

## Open questions
1. What is one credit worth, and how many credits does a typical chat turn, report or spec use? Run a measurement on real prompts first.
2. Should auto-accept be allowed for contributors, or for scheduled tasks that change data (R29a)?
3. Should admins be able to read chats for legal reasons (R5), with a visible audit entry?
4. Should we allow other model providers later for self-hosters? The architecture only assumes Claude API.
5. Should themes be live (updated as notes arrive) instead of daily (R35)?

## Sources
Provenance for `[src:N]` tags is kept in private research notes (S15).
