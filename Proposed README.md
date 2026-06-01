# Proposed Instruction Changes — Drop-Folder Format

This folder is the write-back path for projects that need to propose a change to `Shared\Instructions.db`. Drop a markdown file here using the template below. The Instructions project picks it up on its next session, logs it in `Instructions_tracker.db`, and surfaces it for approval.

**Important:** other projects only **write** to this folder. They do not edit `Instructions.db` directly. Only the Instructions project writes to the rules DB.

## Filename convention

`YYYYMMDD_<projectcode>_<short-slug>.md`

Examples:

- `20260531_JWH_promote-surface-errors.md`
- `20260601_GARDEN_add-pruning-vocab.md`
- `20260605_DDRC_reactivate-cdc-rule.md`

The `<projectcode>` matches a `code` in the `projects` table of `Instructions.db`. The `<short-slug>` is lowercase, hyphen-separated, descriptive.

## Template

```markdown
# Proposed Instruction Change

## Source project
<JWH | GARDEN | DDRC | INSTRUCTIONS | other>

## Type
<Add new | Edit existing | Retire | Reactivate | ScopeChange | ConditionChange | Reorder>

## Affected rule
<If editing/retiring/reactivating/scope-changing/condition-changing/reordering:
 the rule's current ID and title from Instructions.db. Omit for Add new.>

## Current instruction
<Canonical approved text of the affected rule, or "(new)" for additions.>

## Proposed instruction

- **Title:** <title>
- **Tier:** <Governance | Output discipline | Work conventions | Edge cases & vocabulary>
- **Category:** <Governance | File output | Workflow | Code Files | Standards | Power Query | Documentation | Edge cases | Vocabulary>
- **Scope:** <global | conditional>
- **Conditions** (if conditional): <power_query | gardening_domain | cdc_suppression_data | new condition>
- **active_when:** <runtime trigger, or omit if always-on>
- **Body:**

<full instruction text>

## Rationale
<Why this change matters; what situation motivated it. Include the project context if relevant.>

## Apply when
<The trigger that makes this rule fire — same as active_when, or "always" if always-on within scope.>
```

## Lifecycle of a proposal

1. Project drops the file in this folder.
2. Instructions project, on its next session, picks up files in `New` state and logs each as a row in `proposal_queue` and a corresponding item in `items`. Status: `New` → `Under Review`.
3. User reviews in the tracker interface. Decision: `Approved`, `Rejected`, or `Withdrawn`.
4. If approved:
   - `Instructions.db` is snapshotted to `Shared\db_snapshots\Instructions_pre_<slug>_<timestamp>.db`.
   - The change is applied. `rule_change_log` records the before/after.
   - Status: `Approved` → `Applied`. File is moved to `Applied\`.
5. If rejected or withdrawn:
   - File is moved to `Rejected\` (with rejection rationale in `notes`).
6. Other projects pick up the change on their next chat start (read-on-load).

## What to do if you want to propose a brand-new condition

Adding a new row to the `conditions` table is itself a change — propose it the same way. Set `Type` to `ConditionChange` (or `Add new` for the condition itself), and in `Proposed instruction` list the proposed `code`, `name`, `description`, and which projects should be tagged with the new condition. The Instructions project will route this through the same approval flow.

## What to do if you want to change which conditions are tagged on your project

Propose it the same way. `Type` = `ConditionChange`, source project = the project whose tags are changing, and in `Proposed instruction` list the conditions to add or remove. The Instructions project updates `project_conditions` after approval.
