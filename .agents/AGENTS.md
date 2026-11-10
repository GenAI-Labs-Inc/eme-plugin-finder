# AGENTS.md

## Catalog Plugin Guide

### Purpose

`plugins/catalog` holds the shared data model for the whole EME stack: the field/table
definitions, list-of-values, search views, AI configuration, and server-side event scripts that
every other plugin (finder, community, profile, mediadb, ...) reads or extends. It has no Java
`code/` folder — almost everything here is XML/XConf data and Groovy/Velocity event scripts.

### Folder Map

- `html/data/fields/` Table (entity) and property definitions — the schema itself
- `html/data/lists/` List-type field values (dropdown options, enabled flags, endpoint docs, etc.)
- `html/data/views/` Saved search/report views over the tables above
- `html/data/_site.xconf` Default page settings applied to everything under `html/data`
- `html/configuration/` Base templates and system config: `baseentitytemplate.xml` (base for
  `entity*` tables), `baseaiprompt.xml`, `beans.xml`, `embedding.xml`, `elementaljob.xml`
- `html/events/<module>/` Server-side event scripts grouped by domain (asset, billing, blog,
  categories, entities, goaltask, llm, notifications, publishing, users, ...) — these run on
  create/update/delete and other lifecycle hooks for the tables in `html/data/fields`
- `html/images/` Static assets used by catalog-owned pages (e.g. watermark)

### What This Plugin Owns

- The full data dictionary: every table name, its fields, and their types
- List-of-values used by dropdowns and enabled/ordering config across the app (including
  `aiskill`/`automationstep` which drive the AI automation pipeline defined in `plugins/finder`)
- REST endpoint documentation in `html/data/lists/endpoint/*.xml` (sample requests/responses for
  the JSON services implemented in `plugins/mediadb/html/services`)
- Cross-cutting event scripts that fire on data changes, independent of which plugin's UI
  triggered the change

### Editing Rules

- Never hand-write XML under `html/data/{fields,lists,views}` or `html/configuration` — use the
  `catalog-table-creator` skill (`.agents/skills/catalog-table-creator/SKILL.md`) so folder layout
  and cross-file linking conventions are followed correctly.
- Table names starting with `entity` follow the entity template pattern in
  `html/configuration/baseentitytemplate.xml`; anything else is a plain table.
- New list-type fields require a matching file under `html/data/lists/<fieldname>/`.
- Event scripts belong under `html/events/<module>/` grouped by the table/domain they react to;
  keep new scripts in the folder matching their table, not a catch-all location.
- If you add or change a REST-facing table, update the matching endpoint doc in
  `html/data/lists/endpoint/*.xml` so API consumers and AI agents can discover it.

### Validation Checklist

1. Confirm the new/changed field or list XML validates (well-formed XML, correct `id`s).
2. Clear the page cache (or restart) — changes to `html/data` and `html/configuration` are read
   through the xconf cache, not picked up on simple reload like a plain HTML file.
3. Reindex if a field's searchability/type changed, since Elasticsearch mapping is derived from
   these field definitions (see `plugins/system` for the index bootstrap).
4. If you changed any file under `html/data/lists/<table>/`, load it into the running server with the
   `reload-list-data` skill (`.agents/skills/reload-list-data/SKILL.md`): `restoredata` for adds/edits,
   `deleteall` then `restoredata` when rows were removed.
5. Exercise the table from the admin UI (finder) to confirm the field/list renders and saves.

### Notes For Agents

- This plugin is data, not code — most "customization" requests that mention a new table, field,
  dropdown, or automation step land here, even if the feature is surfaced by another plugin's UI.
- `plugins/catalog/html/data/lists/automationstep/*.xml` controls which Java `Skill` classes (see
  `.agents/skills/create-java-ai-skill/SKILL.md` at the repo root — not under `plugins/finder`,
  which currently has no `.agents/` directory) run and in what order.
- Don't hand-dump the whole `aiskill` table into an LLM prompt to let the model "see everything
  available" — it's already embedded/indexed for semantic search precisely so callers can retrieve
  just the relevant few records instead. `plugins/finder/code/org/entermediadb/ai/skills/AgentJobCreatorSkill.java`
  is the reference example: it calls `EmbeddingManager.callFindDocIds(...)` over doc ids prefixed
  `aiskill_<id>` (skills) and `automationscenario_<id>` (automations) to shortlist relevant records
  for a given goal. Note that same class's `loadSkillDocIds()` currently fetches the skill list via
  `getMediaArchive().getList("agentskill")` rather than `"aiskill"` — double-check which list id is
  actually correct before copying that call elsewhere.

## Fulservice project tracking (GitHub Project "Fulservice")

Board: https://github.com/orgs/GenAI-Labs-Inc/projects/1. This section overrides other merge/QA rules in this repo for Fulservice work.

- **Issues live in `GenAI-Labs-Inc/eme-server`**, for every EME repo. Set the project field *Component* to the repo you touch.
- **Flow**: Backlog -> In progress -> In review -> Testing local -> Testing EME server -> Deployed to PROD.
- **Only own issues**: work only on issues assigned to the person who launched the session (`gh issue list -R GenAI-Labs-Inc/eme-server --assignee @me`). Someone else's issue: comment, don't implement.
- **Create**: `gh issue create -R GenAI-Labs-Inc/eme-server --assignee @me --label fulservice --project "Fulservice"`. Fill Priority, Size, Owner type (AI/Human/Pair), Agent, Component.
- **Start**: Status -> In progress. Branch from `fulservice`.
- **PR**: target `fulservice`, body has `Closes GenAI-Labs-Inc/eme-server#<n>`. The project moves it to In review on link and Testing local on merge.
- **Testing EME server / Deployed to PROD**: only a human sets these (and the *Deployed to test on* / *Deployed to PROD on* dates).
- **Blocked**: comment, label `blocked`, Status -> Backlog, Owner type -> Human. Never change Priority or Milestone without a human.
- **CI cost**: run CI once, not on every push. Light checks (build + affected tests) on PRs to `fulservice`; full suite only once on the `fulservice` -> `tested` promotion PR. Any workflow added must use `concurrency` with `cancel-in-progress`, `paths` filters, and no scheduled runs. Don't push repeatedly to a PR just to re-trigger CI.
- Each machine once: `gh auth refresh -s project`.
