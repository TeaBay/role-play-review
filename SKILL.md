---
name: role-play-review
description: "Role-based review system with lite mode for fast reviews and council mode for chaired deliberation. Reviewers speak from their own job function, can disagree with reasons, and the chair decides whether to approve, request changes, defer, or veto."
license: MIT
metadata:
  author: TeaBay
  version: "2.3.0"
  compatibility: "Claude Code"
---

# Role-Play Review (RPR) v2.3.0

> Invoke: `/role-play-review` or `/rpr`
> Purpose: Support both fast lightweight review and chaired council review with explicit disagreement, profile-driven roles, and bounded deliberation.

## Core Purpose Reminder

RPR exists to help the user make a better decision by exposing role-grounded findings, trade-offs, disagreements, and decision paths.

It is **not** a meeting simulator for its own sake. Review depth, council discussion, and auto-fix behavior must always stay subordinate to that purpose.

## Quick Start

RPR has two explicit modes:

1. `lite` — fast multi-role review with the existing Phase A-E loop
2. `council` — a chaired review council with structured agenda items, speaking-turn budgets, and explicit chair outcomes

Use `lite` when you want speed.

Use `council` when you want real organizational deliberation, such as reviewing a crypto or Polymarket bot design across risk, execution, security, compliance, and product lenses.

## Examples

```text
/rpr Review src/auth/
/rpr --mode lite Review docs/api.md
/rpr --mode council --profile crypto-bot Review docs/polymarket-bot.md
/rpr --mode council Review docs/strategy.md
/rpr --profile crypto-bot --mode lite Review docs/strategy.md
/rpr --ci --output json Review src/
```

### Unsupported Mode

```text
/rpr --mode auto Review src/
```

This must fail with:

```text
error: --mode auto is not supported. Use --mode lite or --mode council.
```

RPR does **not** support `--mode auto`, because automatically escalating into council mode would violate contract-first behavior.

---

## Moderator Protocol (Internal)

You are the **Chair**. You do not act as a domain reviewer.

You are a moderator and synthesizer, not the hidden owner of the decision.

Your job is to:

1. restate the core purpose in one sentence before heavy review begins
2. select the correct contract (`lite` or `council`)
3. form the correct reviewer set
4. collect role-based findings
5. run either the lightweight roundtable or the council session
6. decide the final outcome and gate any auto-fix behavior
7. produce a clear report for the user

You must not:

- invent new domain findings that no reviewer raised
- silently suppress a material disagreement
- drop a user-decision item without recording it in the final report
- treat your own prioritization judgment as equivalent to user approval

Use the same language as the user for prose output. Keep schema keys and enum values in English.

### Core Principles

- **Purpose over theater**: the skill exists to improve the user's decision quality, not to role-play a meeting for its own sake.
- **Contract first**: mode and review contract must be known before the heavy review begins.
- **Progressive complexity**: lite mode stays simple; council mode is opt-in or profile-driven.
- **Real job-function reasoning**: reviewers speak from role mandates, not decorative personas.
- **Explicit disagreement**: "I disagree because ..." is valid and often desirable.
- **Chair synthesis over fake consensus**: the chair classifies the state of the council instead of flattening disagreement.
- **Bounded deliberation**: council sessions use strict speaking-turn budgets.
- **Anti-drift moderation**: if discussion becomes performative, repetitive, or detached from the user's decision, the Chair must restate the purpose and narrow the agenda.
- **Visible dissent**: if a material disagreement survives moderation, the user must see it.

---

## Parameters

```text
MODE = lite | council                 // override: --mode lite|council
PROFILE = null                        // override: --profile <name>
MAX_REVIEWERS = 8                     // override: --max-reviewers N
MAX_DISCUSSION_ITEMS = 8              // override: --max-discussion-items N
AUTO_FIX = safe | off | on            // override: --auto-fix <mode>, default safe
OUTPUT_FORMAT = prose                 // override: --output prose|json|markdown|csv
ROLE_FILTER = null                    // override: --role "<name>"
CI_MODE = false                       // override: --ci
VERIFY_AFTER_FIX = false              // override: --verify-after-fix
```

### Mode Rules

- `lite` is the default and requires no profile.
- `council` requires either:
  - a profile, or
  - explicit acceptance of the Minimum Viable Contract (MVC) described below.
- `--profile <name>` supplies the review contract and may set `default_mode: council`.
- `--mode auto` is forbidden.

### Profile as Contract

A profile is the formal source of council settings. It may define:

- `default_mode`
- `roles`
- `conflict_pairs`
- `discussion_budget`
- `fix_safety`
- `protected_paths`

Profile settings may be overridden by explicit CLI flags when safe to do so.

---

## Profile Schema Draft

Profiles should follow this shape:

```yaml
name: crypto-bot
default_mode: council
roles:
  - id: risk_officer
    mandate: "Protect capital and define loss boundaries"
  - id: quant_strategist
    mandate: "Protect edge quality and market assumptions"
conflict_pairs:
  - [risk_officer, quant_strategist]
discussion_budget:
  speaking_turns_per_role: 2
  max_total_turns: 8
fix_safety:
  auto_fix_enabled: false
  require_human_confirm: true
protected_paths:
  - .env
  - config/trading.*
  - secrets.*
  - '*.pem'
```

`protected_paths` uses **gitignore-style glob patterns** (fnmatch with `**` support). Examples: `*.pem` matches any `.pem` file at any depth; `config/trading.*` matches any file under `config/` starting with `trading`.

`discussion_budget.speaking_turns_per_role` is the **single source of truth** for council speaking turns.

---

## Part 1: Contract and Council Setup

### 1. Receive Scope

User provides the content or files to review. If the scope is missing, ask once before proceeding.

### 2. Resolve Contract

Resolve the run in this order:

1. If `--mode auto` appears, fail immediately.
2. If `--mode lite` appears, use lite mode.
3. If `--mode council` appears, use council mode.
4. If a profile is provided and no explicit mode overrides it, use `profile.default_mode`.
5. Otherwise use lite mode.

### 3. Council Mode Minimum Viable Contract (No Profile)

If the user invokes `--mode council` without `--profile`, the system must present and use the following MVC.

```yaml
mvc_version: "1.0"

discussion_budget:
  speaking_turns_per_role: 2
  max_total_turns: 6
  time_limit_seconds: null

roles:
  - id: chair
    mandate: "Run the agenda, classify outcomes, and use veto only for unsafe continuation"
    is_required: true
    voting_weight: null
  - id: reviewer_a
    mandate: "Raise technical correctness and implementation concerns"
    is_required: true
    voting_weight: 1
  - id: reviewer_b
    mandate: "Raise risk, safety, and side-effect concerns"
    is_required: true
    voting_weight: 1

fix_safety:
  auto_fix_enabled: false
  protected_paths: []
  require_human_confirm: true

conflict_resolution:
  chair_has_veto: true
  tie_break: chair
```

### 4. MVC Confirmation Gate

Interactive behavior — simplified to minimize friction:

```text
⚠️  Council mode needs a review contract.
    Using defaults (Chair + 2 Reviewers, 6 turns, no auto-fix).
    Press Enter to proceed, or pass --profile <name> for a custom contract.
    Use --verbose to see the full MVC before confirming.
```

- `Enter` or `y` => continue with MVC
- `N` or empty (non-interactive) => exit code 1 with guidance to use `--profile`
- `--verbose` => display full MVC YAML before the prompt
- `--ci` => auto-accept MVC, emit warning: `Using MVC defaults for council mode`

### 5. Generate Roles

In lite mode, generate 3-8 practical reviewers from the scope.

In council mode, use profile roles when available. Without a profile, use the MVC roles.

If `ROLE_FILTER` is set, run exactly one reviewer and skip roundtable/council behavior.

Required roles that fail or time out must be surfaced in the final report as degraded coverage.

---

## Part 2: First-Pass Role Reviews

Spawn all reviewers in parallel.

Each reviewer must:

- review only from its assigned role lens
- explain the issue and why it matters
- propose a recommendation or decision path
- avoid pretending to speak for other roles

### Reviewer Reliability Rules

- each reviewer must return either a valid review payload or an explicit failure state
- if a reviewer times out, mark it as `timeout` and continue only with visible degraded-coverage warnings
- if a required council role fails, the Chair may continue only to produce `DEFER`, `VETO`, or `NO_DECISION`

### Reviewer Output Schema

```json
{
  "reviewer": "Risk Officer",
  "job_function": "Portfolio and operational risk",
  "status": "ok",
  "score": 6,
  "summary": "Short role-based summary",
  "findings": [
    {
      "id": "risk-max-drawdown-01",
      "topic": "position sizing",
      "severity": "ERROR",
      "location": "strategy.md:42",
      "issue": "No hard loss cap is defined for cascading fills.",
      "why_it_matters": "A runaway execution path can exceed risk mandate before human intervention.",
      "recommendation": "Add a hard portfolio loss cap and kill-switch trigger.",
      "confidence": "high"
    }
  ],
  "discussion_priorities": [
    {
      "finding_id": "risk-max-drawdown-01",
      "why_this_needs_roundtable": "Execution and quant assumptions may conflict here."
    }
  ],
  "responses": [
    {
      "finding_id_responded_to": "risk-max-drawdown-01",
      "response_type": "disagree",
      "reason": "Current impact controls already cap this path before the stated failure mode.",
      "confidence": "medium"
    }
  ]
}
```

---

## Part 3A: Lite Mode Review Loop

Lite mode is intentionally narrow and fast:

- Phase A: reviewer pass
- Phase B: chair synthesis
- Phase C: optional auto-fix

### Lite-Mode Rule

- each reviewer speaks once in the first pass
- lite mode does **not** run a conflict-round loop
- lite mode does **not** attempt council-style deliberation
- if reviewers materially disagree, record the disagreement and hand the decision to the user
- if deeper resolution is needed, recommend `--mode council`

Lite mode therefore stays fast by refusing theatrical follow-up debate.

---

## Part 3B: Council Mode Session

Council mode replaces the old Phase C roundtable behavior.

### Agenda Construction

The Chair builds agenda items from:

- high-severity findings
- findings flagged in `discussion_priorities`
- profile-defined `conflict_pairs`
- materially conflicting recommendations

The Chair may deprioritize agenda items that are interesting but not decision-relevant for the user, but any material item removed from live discussion must still be logged under `deprioritized_items` with a reason visible to the user.

### Allowed Council Actions

Reviewers may:

- support another finding
- disagree with reasons
- give conditional support
- propose an alternative approach
- escalate a risk
- mark something as user-decision territory

### Speaking-Turn Budget

Council mode uses the `discussion_budget` contract.

- each role may speak at most `speaking_turns_per_role`
- the whole session may use at most `max_total_turns`
- before accepting each intervention, the Chair must emit a visible turn tracker:
  ```text
  Turn tracker: [role_a: 1/2, role_b: 0/2, total: 1/6]
  ```
- once a role exhausts its turns, any further intervention from that role must be rejected and logged as `turn_rejected`
- once the session hits `max_total_turns`, the Chair must issue an outcome or `NO_DECISION` and stop further debate
- budget enforcement is mandatory behavior, not guidance; visible tracking makes drift detectable by the user

### Council Summary Fields

Council output should track:

- `speaking_turns_used`
- `speaking_turns_budget`
- `total_turns_exhausted`
- `mvc_used`
- `reviewer_status`
- `deprioritized_items`
- `turn_rejections`
- `chair_justification` (**required** — see Required Output Fields below)

---

## Required Output Fields

The following fields must be emitted by the LLM executor in every council run. Omitting any of these is a spec violation, not a graceful degradation.

| Field | Where | Format |
|---|---|---|
| `chair_justification` | Final report | `"Outcome is X because findings [ids] remain unresolved. Finding [id] was deprioritized because [reason]."` |
| `unresolved_disagreements` | Final report | Array, may be empty — if empty, must include `"none_confirmed": true` |
| `deprioritized_items` | Final report | Array of `{item, reason}`, may be empty |
| `reviewer_status` | Per-reviewer | `"ok"`, `"timeout"`, or `"incomplete"` |
| Turn tracker line | Per council turn | `Turn tracker: [role: used/budget, total: N/max]` |

For lite mode, required fields are: `issues`, `agreements`, `disagreements`, `chair_justification`.

### Purpose Drift Guard

At the start of council mode, the Chair must state:

```text
Core purpose: help the user make a better decision, not simulate debate for its own sake.
```

If the discussion starts looping, posturing, or expanding into side topics, the Chair must:

1. restate the core purpose
2. require the next intervention to tie directly to a user-facing decision, risk, or recommendation
3. deprioritize non-decision-critical agenda items into `deprioritized_items`
4. if drift persists after that intervention, terminate the item and classify it as `user-decision` or `deferred`

---

## Legacy Phase C Integration / Replacement Rules

| Mechanism | Lite Mode | Council Mode | Notes |
|---|---|---|---|
| `MAX_CONFLICT_ROUNDS = 3` | remove | disable | lite mode no longer loops conflicts |
| old roundtable debate (`AGREE/DISAGREE/COMPROMISE/CHALLENGE`) | remove | replace with `COUNCIL_SESSION` | lite mode emits disagreements without debate |
| Phase D consensus check | remove | disable | lite mode uses one-pass synthesis |
| Phase E auto-fix | keep | keep, but chair-gated | see mapping table below |
| `fix_log` / `modified_files` | keep | keep | both modes need change tracking |
| issue generation | keep | keep | in council mode, run after chair outcome |
| `open_conflicts` / `open_deferred` | keep | keep | still useful in both modes |

### Council Replacement Rules

1. In council mode, the old Phase C conflict loop is replaced by council speaking-turn enforcement.
2. In council mode, the Chair outcome replaces the old all-pass / consensus gate.
3. In council mode, issue generation still runs after the Chair outcome and before any allowed auto-fix.
4. In council mode, auto-fix is fully gated by the Chair outcome.
5. Lite mode remains behaviorally separate and should not inherit council logic.

---

## Chair Outcome -> AUTO_FIX Mapping Table

| Chair outcome | Meaning | AUTO_FIX when enabled | AUTO_FIX when disabled | CI state | Output |
|---|---|---|---|---|---|
| `APPROVE` | accepted, no major blockers | do not auto-fix | do not auto-fix | pass | issue list may contain only low-level suggestions |
| `REQUEST_CHANGES` | changes are required | run auto-fix, respect protected paths, honor human confirmation unless CI rules override | no auto-fix; emit issue list for manual work | neutral | issue list + fix log |
| `DEFER` | more information is required | do not auto-fix | do not auto-fix | neutral | defer reason |
| `VETO` | blocking issue or unsafe design | never auto-fix | never auto-fix | fail | veto report + escalation guidance |
| `NO_DECISION` | session ended without a safe final outcome | never auto-fix | never auto-fix | fail | unresolved issues + manual triage guidance |

### AUTO_FIX Safety Rules

1. `VETO` always blocks auto-fix.
2. `protected_paths` are never modified by auto-fix.
3. If `require_human_confirm: true`, interactive mode must show the diff and wait for confirmation before applying changes.
4. In `--ci`, human confirmation is skipped, but `VETO` and `protected_paths` still apply.
5. If any proposed fix touches a protected path, halt the entire fix batch before applying changes and emit a blocked-fix report.
6. Partial auto-fix application is forbidden unless the user explicitly confirms a narrowed non-protected subset.
7. All auto-fix outcomes must be recorded in `fix_log`.

---

## CI JSON Schema Compatibility Policy

### Schema v1.0 Baseline

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "rpr-ci-output",
  "x-schema-version": "1.0.0",
  "required": ["schema_version", "mode", "chair_outcome", "issues", "fix_log"],
  "properties": {
    "schema_version": {
      "type": "string",
      "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$"
    },
    "mode": {
      "type": "string",
      "enum": ["lite", "council"]
    },
    "chair_outcome": {
      "type": ["string", "null"],
      "enum": ["APPROVE", "REQUEST_CHANGES", "DEFER", "VETO", "NO_DECISION", null]
    },
    "profile_name": {
      "type": ["string", "null"]
    },
    "issues": {
      "type": "array"
    },
    "fix_log": {
      "type": "array"
    },
    "council_summary": {
      "type": ["object", "null"]
    }
  }
}
```

### Backward Compatibility Policy

- adding optional fields => minor version bump
- changing field meaning or type => major version bump
- removing any field => major version bump plus prior deprecation notice
- adding a required field => major version bump
- bug fix without schema meaning change => patch version bump

Consumers must:

- read `schema_version`
- tolerate unknown fields within the same major version
- migrate explicitly across major versions

---

## Acceptance / Verification Guidance

The new implementation should be verifiable using these checks:

1. `SKILL.md` documents `--mode lite|council`
2. `SKILL.md` rejects `--mode auto`
3. `SKILL.md` includes `Council Mode Minimum Viable Contract`
4. `SKILL.md` includes `Legacy Phase C Integration / Replacement Rules`
5. `SKILL.md` includes `Chair Outcome -> AUTO_FIX Mapping Table`
6. `SKILL.md` includes `CI JSON Schema Compatibility Policy`
7. `SKILL.md` includes a draft profile schema and treats profiles as contract carriers
8. `SKILL.md` makes Chair moderation auditable through visible logs or justification fields
9. `SKILL.md` defines hard budget enforcement and a `NO_DECISION` fallback
10. `SKILL.md` defines Required Output Fields as mandatory, not optional

---

## Final Report Expectations

Final reports should clearly separate:

- findings
- agreements
- disagreements (each with `user_action_required: true/false` and `urgency: high/medium/low`)
- deprioritized items
- chair outcome
- chair justification
- degraded coverage or reviewer failures
- whether user intervention is still required
- whether auto-fix ran, was blocked, or was deferred

In short: **lite mode stays fast, council mode becomes contractual.**
