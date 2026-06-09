# Instructions DB — Read-on-Load Protocol

## What this is

The central rule set for all Claude-driven projects.

**Source:** `<your-instructions-folder>\Shared\Instructions.db` (READ ONLY from other projects)

**Read interface:** the view `v_instructions_for_project` in that DB. The canonical read query is catalogued inside the Instructions project's tracker as `select_active_instructions_for_project`.

## On every chat start (and after compaction)

1. Open `Shared\Instructions.db` (read-only).
2. Find your project in the `projects` table by `code`. If missing, ask the user to register the project and assign conditions (the "On first activation in a project, ask which conditions apply" rule).
3. Query `v_instructions_for_project` filtered by your project code:

   ```sql
   SELECT id, title, instruction, active_when, scope,
          tier, tier_order, category, category_order, display_order
   FROM v_instructions_for_project
   WHERE project_code = :project_code
   ORDER BY tier_order, category_order, display_order;
   ```

4. You may ask questions if it's not clear how the instructions apply, or if the user would like to propose an amendment. If a change is to be requested, follow **Proposing a change** below. Until a proposed change is accepted, follow the rules in the Instructions DB.
5. Copy the result into memory as your active rule set, ordered by `(tier_order, category_order, display_order)`.
6. Record the `Instructions.db` file hash and the row count in memory so the user can see which version of the rule set governed the conversation.
7. If a rule has `active_when`, treat it as a runtime trigger — load it but only fire when the trigger fires.

## Memory layout per project

One memory file of type `reference` named `instructions_active_<projectcode>.md` (e.g., `instructions_active_jwh.md`) holding the resolved rule set + load timestamp + `Instructions.db` SHA256 hash. Updated only by the read-on-load step. This is the cache the fallback relies on.

## Memory fallback

If `Instructions.db` is unreachable (file locked, sync issue, network drive offline, etc.):

1. Fall back to the rule set already in memory from the previous load.
2. Warn the user that you are on a cached rule set, show the cached hash and load timestamp.
3. Resume normal work.
4. Retry `Instructions.db` at the start of each new response until reachable. On reconnect, re-load and report the new hash.

## Proposing a change (no copy/paste from this DB)

Write a markdown file in `Shared\Proposed\` following the format in `Shared\Proposed\README.md`. The four sections are:

1. **Current instruction** — canonical approved text, or "(new)" for additions
2. **Proposed instruction** — exact replacement text, including title, category, tier, scope, conditions if any, and `active_when` if any
3. **Rationale** — why this change matters
4. **Apply when** — what trigger makes this rule fire (or "always")

The Instructions project picks it up, logs it as an item in `Instructions_tracker.db`, surfaces it for the user's approval, and applies the approved change.

Until a proposed change is accepted, follow the rules in the Instructions DB.

## Hard rules

- Never edit `Instructions.db` from inside another project. Reads only.
- Never invent rules. If a behavior is unclear, ask the user.
- After every chat compaction, re-load from `Instructions.db` (step 1 above).

## Where things live

| What | Where |
|---|---|
| The rules DB | `Shared\Instructions.db` |
| This protocol | `Shared\startup.md` |
| Proposal drop folder | `Shared\Proposed\` |
| Accepted proposals (archive) | `Shared\Proposed\Applied\` |
| Rejected proposals (archive) | `Shared\Proposed\Rejected\` |
| Pre-write DB snapshots | `Shared\db_snapshots\` |

All paths are relative to `<your-instructions-folder>\` (the `instructions_folder` parameter in each project tracker).
