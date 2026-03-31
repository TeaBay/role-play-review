---
name: role-play-review
description: "Role-based review system with lite mode for fast reviews and council mode for chaired deliberation. Reviewers speak from their own job function, can disagree with reasons, and the chair decides whether to approve, request changes, defer, or veto."
license: MIT
metadata:
  author: TeaBay
  version: "2.1.0"
  compatibility: "Claude Code"
---

# Role-Play Review (RPR) v2.1

> Invoke: `/role-play-review` or `/rpr`
> Purpose: Support both fast lightweight review and chaired council review with explicit disagreement, profile-driven roles, and bounded deliberation.

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

Your job is to:

1. select the correct contract (`lite` or `council`)
2. form the correct reviewer set
3. collect role-based findings
4. run either the lightweight roundtable or the council session
5. decide the final outcome and gate any auto-fix behavior
6. produce a clear report for the user

Use the same language as the user for prose output. Keep schema keys and enum values in English.

### Core Principles

- **Contract first**: mode and review contract must be known before the heavy review begins.
- **Progressive complexity**: lite mode stays simple; council mode is opt-in or profile-driven.
- **Real job-function reasoning**: reviewers speak from role mandates, not decorative personas.
- **Explicit disagreement**: "I disagree because ..." is valid and often desirable.
- **Chair synthesis over fake consensus**: the chair classifies the state of the council instead of flattening disagreement.
- **Bounded deliberation**: council sessions use strict speaking-turn budgets.

---

## Parameters

```text
MODE = lite | council                 // override: --mode lite|council
PROFILE = null                        // override: --profile <name>
MAX_REVIEWERS = 8                     // override: --max-reviewers N
MAX_DISCUSSION_ITEMS = 8              // override: --max-discussion-items N
MAX_CONFLICT_ROUNDS = 3               // lite mode only
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
    mandate: "主持議程，作出最終結局決定，有 VETO 權"
    is_required: true
    voting_weight: null
  - id: reviewer_a
    mandate: "提出技術正確性問題"
    is_required: true
    voting_weight: 1
  - id: reviewer_b
    mandate: "提出風險及潛在副作用問題"
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

Interactive behavior:

```text
⚠️  Council mode is enabled without a profile.
    Using Minimum Viable Contract (MVC v1.0):
    - roles: Chair + 2 Reviewers
    - speaking_turns_per_role: 2
    - max_total_turns: 6
    - auto_fix_enabled: false
Continue? [y/N]
```

- `y` => continue
- `N` or empty => exit code 1 with guidance to use `--profile`
- `--ci` => auto-accept MVC, emit warning: `Using MVC defaults for council mode`

### 5. Generate Roles

In lite mode, generate 3-8 practical reviewers from the scope.

In council mode, use profile roles when available. Without a profile, use the MVC roles.

If `ROLE_FILTER` is set, run exactly one reviewer and skip roundtable/council behavior.

---

## Part 2: First-Pass Role Reviews

Spawn all reviewers in parallel.

Each reviewer must:

- review only from its assigned role lens
- explain the issue and why it matters
- propose a recommendation or decision path
- avoid pretending to speak for other roles

### Reviewer Output Schema

```json
{
  "reviewer": "Risk Officer",
  "job_function": "Portfolio and operational risk",
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
  ]
}
```

---

## Part 3A: Lite Mode Review Loop

Lite mode preserves the existing high-level structure:

- Phase A: reviewer pass
- Phase B: judgment
- Phase C: lightweight roundtable
- Phase D: re-judgment
- Phase E: optional auto-fix

### Lite-Mode Rule

`MAX_CONFLICT_ROUNDS = 3` remains active **only in lite mode**.

Lite mode keeps the existing roundtable model and does **not** use council speaking-turn budgets.

---

## Part 3B: Council Mode Session

Council mode replaces the old Phase C roundtable behavior.

### Agenda Construction

The Chair builds agenda items from:

- high-severity findings
- findings flagged in `discussion_priorities`
- profile-defined `conflict_pairs`
- materially conflicting recommendations

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
- once a role exhausts its turns, the Chair must cut off further interventions from that role
- once the session hits `max_total_turns`, the Chair must classify the issue and stop further debate

### Council Summary Fields

Council output should track:

- `speaking_turns_used`
- `speaking_turns_budget`
- `total_turns_exhausted`
- `mvc_used`

---

## Legacy Phase C Integration / Replacement Rules

| Mechanism | Lite Mode | Council Mode | Notes |
|---|---|---|---|
| `MAX_CONFLICT_ROUNDS = 3` | keep | disable | replaced by council discussion budget |
| old roundtable debate (`AGREE/DISAGREE/COMPROMISE/CHALLENGE`) | keep | replace with `COUNCIL_SESSION` | council has chair-led agenda |
| Phase D consensus check | keep | disable | chair outcome is the only final council judgment |
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

### AUTO_FIX Safety Rules

1. `VETO` always blocks auto-fix.
2. `protected_paths` are never modified by auto-fix.
3. If `require_human_confirm: true`, interactive mode must show the diff and wait for confirmation before applying changes.
4. In `--ci`, human confirmation is skipped, but `VETO` and `protected_paths` still apply.
5. All auto-fix outcomes must be recorded in `fix_log`.

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
      "enum": ["APPROVE", "REQUEST_CHANGES", "DEFER", "VETO", null]
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

---

## Final Report Expectations

Final reports should clearly separate:

- findings
- agreements
- disagreements
- chair outcome
- whether user intervention is still required
- whether auto-fix ran, was blocked, or was deferred

In short: **lite mode stays fast, council mode becomes contractual.**
