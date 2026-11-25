# Glossary

Wareslate's own terms. Specs, code, API and UI use exactly these words.

| Term | Identifier | Meaning |
|---|---|---|
| Workspace | `workspace` | One organization's data and settings |
| Admin / Editor / Collaborator / Reader | `admin`, `editor`, `collaborator`, `reader` | Member roles, from most to least rights (S01) |
| Space | `space` | A group of members with its own views and pages. Visibility `listed`, `invite_only` or `hidden` |
| My space | `my_space` | A member's own private space |
| Access | `read`, `write`, `manage`, `owner` | A member's level in a space or on a shared object |
| Product tree | | Product > Module > Feature > Slice |
| Module | `module` | A part of a product; modules can nest |
| Slice | `slice` | A smaller piece of a feature |
| Bet | `bet` | A cross-cutting piece of work linked to objectives and features |
| Objective / Key result | `objective`, `key_result` | Goals and their measurable results |
| Release / Release track | `release`, `release_track` | A planned delivery, and a set of releases. Release status: `planned`, `active`, `shipped` |
| Date range | `date_range` | Start and end date with a `precision` (day, month, quarter, year) |
| Size | `size` | Estimated work. Mode `from_children` or `manual` |
| Check-in | `check_in` | A status report: `green`, `amber`, `red` or `none`, plus a comment |
| Feedback entry | `feedback` | A piece of customer input. Kinds: `message`, `thread`, `deal`. State: `new` or `triaged` |
| Excerpt | `excerpt` | A marked part of a feedback entry |
| Evidence | `evidence` | A link from feedback (or an excerpt) to an item, with a need level |
| Need | `need` | How much the customer needs it: `none`, `low`, `medium`, `high`, or `unspecified` |
| Demand score | `demand_score` | The score built from evidence (S06) |
| Criterion | `criterion` | A 0–5 team scoring field (S06) |
| Computed field | `computed` | A field calculated from other fields (S06) |
| Account / Contact | `account`, `contact` | A customer organization and a person at it |
| Cohort | `cohort` | A saved group of accounts or contacts |
| Account total | `account_total` | A sum or other roll-up of an account field over the accounts asking for an item |
| View | `view` | A saved way to look at data: `table`, `timeline`, `kanban`, `feedback_view`, account view, contact view |
| Lanes | `lanes` | Horizontal grouping rows on timeline and kanban views |
| Page | `page` | A rich-text page that can be edited by several people at once |
| Watcher | `watcher` | A member who gets notified about an object |
| Public roadmap | `public_roadmap` | A website showing chosen items to customers: sections > lists > entries, with votes and idea suggestions |
| External action | `external_action` | A button on an item, served by an outside app. State `idle`, `working`, `linked`, `failed` |
| Sync / Seed | `sync`, `seed` | Integration field direction: kept in step both ways, or copied once |
| Context library | `context_source` | Background material the AI assistant uses |
| Playbook | `playbook` | A saved AI workflow |
| Recurring run | `recurring_run` | A playbook run on a schedule |
| Apply without review | `apply_without_review` | AI changes applied without a preview step, with undo |
| Pattern / Problem statement / Cluster / Digest | `pattern`, `problem`, `cluster`, `digest` | AI analysis results over feedback |
| AI units | `ai_units` | Metering for AI usage |
