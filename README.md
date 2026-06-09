# Centralized AI Instructions DB

*The multi-project evolution of [ai-assistant-rules](https://github.com/janinewhite/ai-assistant-rules) — one shared `Instructions.db` that every project reads from, with conditional applicability, a proposal-folder workflow, and an audit log. Same philosophy as v1: local SQLite, queryable, version-controlled, no cloud.*

This README is a snapshot of how I currently work with an AI assistant across multiple projects. It's the next layer on top of the v1 pattern — kept simple where it can be, infrastructure where it has to be.

## Why this exists

In v1, every project carried its own copy of the `instructions` table. That worked when I had one or two projects. By the time I had five, three problems had become routine:

1. **Editing a rule meant editing it in N places.** Even with copy-paste, it's easy to miss one.
2. **Wording drifted across copies.** Same titled rule, slightly different text in each project. The differences were rarely intentional — they were artifacts of which project I last refined the rule in.
3. **New rules failed to propagate.** Something I learned in one project would sit unused in the others until I happened to notice and copy it across.

The fix: one DB, many projects, with a clean way for each project to ask "which rules apply to *me*?" and a clean way to propose changes without letting any project edit the canonical source.

## How it works

**One central rules DB.** `Shared\Instructions.db` holds all the rules. It also holds the list of projects, the conditions a project can be tagged with, and the tag assignments.

**Two applicability mechanisms.**

- **Global** rules apply to every active project. No tag, no junction row — just `scope = 'global'`.
- **Conditional** rules apply only to projects tagged with a matching condition. The condition names the *reason* the rule applies ("Power Query project," "Gardening domain"), not the project itself. When a project changes its nature, condition assignments change with it and the right rules turn on or off automatically.

**A read interface (the view).** `v_instructions_for_project` joins the rules to the projects through the condition junctions and returns, for any given `project_code`, the rules that apply to it in reading order. Other projects open `Instructions.db` read-only, run a single SELECT against the view, and get their active ruleset.

**Tier ordering.** Rules belong to one of four tiers and load in that order:

| Tier | What belongs |
|---|---|
| 1. Governance | Rules about the conversation, the rule system, and data integrity. Read first. |
| 2. Output discipline | What every deliverable looks like. |
| 3. Work conventions | Domain-specific: code, queries, formatting. |
| 4. Edge cases & vocabulary | Narrow triggers and shared language. |

`display_order` controls the fine-grained sequence and is globally unique across active rules (enforced by the partial unique index above) — a single linear sequence across every tier and category, not per-category numbering. The tier and category sort orders still drive the reading order at the group level; the global `display_order` falls out from those groups in (tier, category) order. The point of the tier hierarchy: the AI encounters foundational rules ("re-read instructions at chat start," "surface errors," "don't change instructions without approval") before domain-specific ones, which catches mistakes earlier.

**A write-back path (the Proposed folder).** Other projects don't edit `Instructions.db` directly. To propose a change, a project drops a markdown file in `Shared\Proposed\` following a four-section template:

1. **Current instruction** — the canonical approved text, or "(new)" for additions.
2. **Proposed instruction** — the exact replacement text, including title, scope, conditions, and `active_when`.
3. **Rationale** — why the change matters.
4. **Apply when** — the trigger (mirrors `active_when`), or "always" if always-on within scope.

The Instructions project picks up the file on its next session, logs it as a row in the tracker's `proposal_queue`, surfaces it for review, and — after approval — applies the change. Processed files move to `Applied\` or `Rejected\` so the queue stays clean.

**Snapshots and audit log.** Before any change to `Instructions.db`, the DB is copied to `Shared\db_snapshots\Instructions_pre_<slug>_<timestamp>.db`. Every applied change adds a row to `rule_change_log` in a separate tracker DB with: the change type (Add / Edit / Retire / Reactivate / ScopeChange / ConditionChange / Reorder), before/after text, before/after metadata as JSON, the proposal filename, the snapshot path, the post-write SHA256 of `Instructions.db`, and any approval notes. The audit trail is complete and never overwritten.

**HTML interfaces, browser-only.**

- A **read-only viewer** for other projects to browse the rules that apply to them, filterable by project / tier / category / text.
- An **editing surface** for the central DB itself: rule CRUD, category / condition / project management, project-condition tagging.
- A **project tracker** for managing the proposal queue, the audit log, parameters, scripts, and the everyday work on the Instructions project itself.

All three run entirely in the browser via the File System Access API. Nothing is sent to a server. The editor's Save writes a snapshot first, then overwrites the live file — in one click, no manual file moves. Both the editor and viewer remember the last-used folder.

## The schema

```sql
CREATE TABLE projects (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  code         TEXT NOT NULL UNIQUE,
  name         TEXT NOT NULL,
  active       TEXT NOT NULL DEFAULT 'Yes' CHECK(active IN ('Yes','No')),
  date_added   TEXT NOT NULL,
  date_retired TEXT
);

CREATE TABLE conditions (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  code        TEXT NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  description TEXT
);

CREATE TABLE project_conditions (
  project_id   INTEGER NOT NULL REFERENCES projects(id),
  condition_id INTEGER NOT NULL REFERENCES conditions(id),
  PRIMARY KEY (project_id, condition_id)
);

CREATE TABLE tiers (
  id          INTEGER PRIMARY KEY,
  name        TEXT NOT NULL UNIQUE,
  sort_order  INTEGER NOT NULL UNIQUE,
  description TEXT
);

CREATE TABLE categories (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT NOT NULL UNIQUE,
  sort_order  INTEGER NOT NULL
);

CREATE TABLE instructions (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  title           TEXT NOT NULL,
  instruction     TEXT NOT NULL,
  tier_id         INTEGER NOT NULL REFERENCES tiers(id),
  category_id     INTEGER NOT NULL REFERENCES categories(id),
  display_order   INTEGER,
  scope           TEXT NOT NULL CHECK(scope IN ('global','conditional')),
  active          TEXT NOT NULL DEFAULT 'Yes' CHECK(active IN ('Yes','No')),
  active_when     TEXT,
  date_added      TEXT NOT NULL,
  date_retired    TEXT,
  notes           TEXT,
  migration_notes TEXT,
  CHECK ((active = 'Yes' AND display_order IS NOT NULL)
      OR (active = 'No'  AND display_order IS NULL))
);

CREATE UNIQUE INDEX ix_instructions_active_display_order
  ON instructions(display_order) WHERE active='Yes';

CREATE TABLE instruction_conditions (
  instruction_id INTEGER NOT NULL REFERENCES instructions(id),
  condition_id   INTEGER NOT NULL REFERENCES conditions(id),
  PRIMARY KEY (instruction_id, condition_id)
);

CREATE VIEW v_instructions_for_project AS
SELECT
  p.code   AS project_code,
  i.id, i.title, i.instruction, i.active_when, i.scope,
  t.name   AS tier,     t.sort_order AS tier_order,
  c.name   AS category, c.sort_order AS category_order,
  i.display_order
FROM instructions i
JOIN tiers t      ON t.id = i.tier_id
JOIN categories c ON c.id = i.category_id
CROSS JOIN projects p
WHERE i.active = 'Yes' AND p.active = 'Yes'
  AND ( i.scope = 'global'
     OR EXISTS (SELECT 1 FROM instruction_conditions ic
                JOIN project_conditions pc ON pc.condition_id = ic.condition_id
                WHERE ic.instruction_id = i.id AND pc.project_id = p.id))
ORDER BY p.code, t.sort_order, c.sort_order, i.display_order;
```

A separate tracker DB (`Instructions_tracker.db`) carries the `items`, `notes`, `parameters`, `settings`, `data_items`, `scripts`, `query_versions`, `calendar`, `layout_options`, plus the two Instructions-specific tables: `rule_change_log` (every applied change to `Instructions.db`) and `proposal_queue` (the queue of files in `Shared\Proposed\`).

## Rule lifecycle policies

Three policies govern how rows in `instructions` are created, retired, and ordered. They started as conventions and were promoted to schema constraints once the convention began to leak.

**ID stability.** `instructions.id` is `INTEGER PRIMARY KEY AUTOINCREMENT`. Once an ID is assigned, it is permanent — the rule keeps its original ID for the life of the table, the ID is never reused for another rule, and the ID does not change when the rule is edited, reordered, deactivated, or reactivated. Rule numbers therefore do not correspond to display order; a rule added later can occupy any display position. Stable IDs let `rule_change_log.instruction_id`, items-table notes, memory cache entries, and external references keep working across reorders.

**Deactivation, not deletion.** Rules are never removed from the table. When a rule is no longer in force, set `active='No'` and populate `date_retired`. The row stays. `v_instructions_for_project` filters on `active='Yes'`, so retired rules are invisible to running sessions but remain in the table for historical lookup and possible reactivation. Reactivating sets `active='Yes'`, clears `date_retired`, and assigns a fresh `display_order`. The ID stays the same.

**Globally-unique display_order.** `display_order` is unique across all active rules (enforced by the partial unique index `WHERE active='Yes'`). The order is global, not per-category. Retired rules have `display_order=NULL` (enforced by the table-level CHECK), so the slot is free for an active rule to occupy. Display orders may be reassigned freely among active rules to reorder the rule set; reassignment does not change IDs.

These policies make the rule set linearly orderable and let any external reference to a rule by ID stay valid indefinitely — the meta-layer that keeps the system from rotting as it grows.

## Rule writing standards

Two conventions for writing rules, on top of the lifecycle policies:

**Self-contained text.** Each rule should state its directive and reasoning in full within its own body. A reader who has only one rule's text should understand what to do and why. Rules can reference each other by concept (e.g., "the surface-errors rule," "the proposal-and-approval flow") but not by number — IDs are stable but display positions are not, and "see Rule #5" rots the first time something gets renumbered.

**Surface conflicts during proposal.** When proposing a new rule or editing an existing one, evaluate whether the proposed wording conflicts with any other active rule (contradiction, ambiguity about which fires, overlapping scope without clear precedence) and surface any conflict in the proposal's rationale before approval. Resolve by amending the new rule, editing the older rule(s), or explicitly defining precedence.

## Continuous improvement

Every approved decision in the system — architecture, schema, vendor or tool selection, coding pattern, workflow convention, and the rules themselves — stays subject to revision. Approval is the gate that lets a change be implemented; it is not a commitment that the choice will never change. New information, errors surfaced under the surface-errors rule, shifting requirements, or a better understanding of the problem are all legitimate reasons to revisit a prior decision. Re-evaluation runs through the same approval gates as the original — the proposal protocol for the rules DB, the project-tracker workflow for project work, the snapshot-and-verify pattern for file changes. Continuous improvement does not bypass approval; it works through it.


## Folder structure

```
Instructions/
├── Shared/                            ← projects' read surface (read-only for them)
│   ├── Instructions.db                ← the unified rules
│   ├── startup.md                     ← read-on-load protocol for other projects
│   ├── Instructions_interface.html    ← read-only viewer
│   ├── Proposed/                      ← projects' write surface
│   │   ├── README.md                  ← format spec
│   │   ├── Applied/                   ← archive of accepted proposals
│   │   ├── Rejected/                  ← archive of rejected proposals
│   │   └── *.md                       ← open proposals
│   └── db_snapshots/                  ← pre-write snapshots
├── Instructions_tracker.db            ← project tracker for THIS project
├── Instructions_tracker.html          ← tracker UI
├── Instructions_editing_interface.html ← admin editor for Instructions.db
└── Queries/                           ← maintenance scripts
```

Only the Instructions project writes to `Instructions.db`. Other projects read and propose.

## What another project does on load

From `Shared\startup.md`:

1. Open `Instructions.db` read-only.
2. Find your project in `projects`. If you're not there, ask the user to register and tag the project with the conditions that apply.
3. Run `SELECT * FROM v_instructions_for_project WHERE project_code = :code ORDER BY tier_order, category_order, display_order`.
4. (If anything's unclear or you want to propose an amendment, ask now — until a proposed change is accepted, the rules in the DB govern.)
5. Copy the result into memory as your active rule set.
6. Record the `Instructions.db` file hash and the row count so the user can see which version of the rule set governed the conversation.
7. Treat `active_when` as a runtime trigger — load the rule but only fire it when the trigger fires.

If the DB is unreachable, fall back to the cached rule set in memory from the previous load, warn the user, and retry on next response.

## Adapting this for your own work

If you've been running v1 (one `instructions` table per project) and are ready to consolidate:

1. **Decide what's truly global vs. truly conditional.** Most rules are global in spirit but were never copied across projects. A handful are domain-specific (DAX rules only apply to Power BI projects; gardening vocabulary only to gardening). Under v2 you don't need a "project-pinned" category — express it as a condition whose tag is on one project today. The condition name is what makes it self-documenting.

2. **Pick your initial conditions.** Mine are `power_query`, `gardening_domain`, and `cdc_suppression_data` (the last for a project that retired but might come back). Yours will be different. The test for whether something deserves to be a condition: "if I started a new project tomorrow, would I want the rules tagged with this to come along automatically because of the new project's *nature*?"

3. **Pick your tiers.** I use four: Governance, Output discipline, Work conventions, Edge cases & vocabulary. Other splits are fine. The point is to load the foundational ones first.

4. **Migrate your rules.** Most rules from v1 land directly in v2 with minor edits. A few will get rewritten or merged — e.g., my "DB is the primary source of information" got reworded to clarify the distinction between the Instructions DB, the project tracker DB, and any project-specific DB.

5. **Build the proposal workflow once.** A markdown template in `Shared\Proposed\README.md`, a routine of picking up files at session start, a `rule_change_log` in your tracker. This is the piece that prevents the new central DB from becoming a free-for-all.

6. **Keep the snapshot-before-write habit.** Every change to `Instructions.db` produces a versioned snapshot in `db_snapshots\`. The editor in this repo does this automatically; if you do edits any other way, do it manually.

The transition from v1 to v2 took me about a weekend. The biggest task was deciding which rules became global, which became conditional, and what the conditions should be — once that was sorted, the migration itself was a script.

## Why local-only

A common question is "wouldn't this be easier as a SaaS?" Yes — and that's exactly why it isn't.

I built this around a family-scale workflow — including a Family Health Tracker for me and my disabled husband — where "no third-party enterprise sees our data" is a hard requirement, not a preference. The AI assistant patterns that work for that workflow are the same ones that work for any data-engineering job: queryable, version-controlled, auditable, and entirely on disks I control. Local SQLite plus a browser-only HTML page hits every requirement without dragging in a hosted service or an account on someone else's platform.

The patterns scale up or down. The value is highest where the data is personal.

## What's coming

The Family Health Tracker is the concrete demonstration of what this infrastructure enables when family scale meets accessibility and privacy. When it's shareable, it'll go in its own repo. If you're caregiving for someone with complex needs and the off-the-shelf options send your data somewhere you don't want it to go, watch this space.

## License

MIT (same as v1). Use it, fork it, take what's useful, leave the rest.

---

*See also: [ai-assistant-rules](https://github.com/janinewhite/ai-assistant-rules) — v1, the simpler per-project pattern. Start there if you want the foundation; come here when one rule table per project starts feeling like duplication.*
