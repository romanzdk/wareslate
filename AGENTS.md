# Rules for anyone (human or agent) working in this repo

## Specs
- One file per module in `specs/`, following `specs/README.md`. Plain, simple language.
- Specs describe behavior and rules, not screens. Write everything in your own words.
- Every rule is tagged `[src:N]` (based on public research; the source list is kept in private notes)
  or `[decision]` (a Wareslate design choice).
- Use generic product-management terms. Do not reference other product-management tools or their branded feature names.
  Third-party tools we integrate with (Jira, Slack, GitHub, …) may be named.
- Keep `specs/PARITY.md` up to date.

## Code
- Backend: Python 3.13, FastAPI, SQLAlchemy, Postgres. One deployable app with strict module boundaries.
- Frontend: React + TypeScript.
- Comments only for a non-obvious "why".
