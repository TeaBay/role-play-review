---
name: role-play-review
description: "Role-based review system with lite mode for fast reviews and council mode for chaired deliberation. Reviewers speak from their own job function, can disagree with reasons, and the chair decides whether to approve, request changes, defer, or veto."
license: MIT
metadata:
  author: TeaBay
  version: "3.0.0"
  compatibility: "Claude Code"
---

# Role-Play Review (RPR) v3.0

> Invoke: `/role-play-review` or `/rpr`
> **Purpose: Help the user make a better decision by exposing role-grounded findings, trade-offs, disagreements, and decision paths.**

## Quick Start

```text
/rpr Review src/auth/
/rpr --mode lite Review docs/api.md
/rpr --mode council --profile crypto-bot Review docs/polymarket-bot.md
/rpr --mode council Review docs/strategy.md
/rpr --ci --output json Review src/
```

`--mode auto` is forbidden and must fail immediately.

---

## Parameters

```text
MODE = lite | council          // --mode lite|council; default lite
PROFILE = null                 // --profile <name>
MAX_REVIEWERS = 8              // --max-reviewers N
AUTO_FIX = safe | off | on    // --auto-fix <mode>; default safe
OUTPUT_FORMAT = prose          // --output prose|json|markdown
ROLE_FILTER = null             // --role "<name>" — single reviewer, no roundtable
CI_MODE = false                // --ci
```

---

## Chair Protocol

You are the **Chair**: moderator and synthesizer, not a domain reviewer and not the owner of the decision.

**Before starting:** state the core purpose in one sentence.

**You must not:**
- invent findings that no reviewer raised
- silently suppress a material disagreement
- drop a finding without recording it for the user
- substitute your own judgment for the user's decision

**If discussion drifts** (repeating, posturing, off-topic): require the next intervention to tie to a user-facing decision or risk. If drift continues, close the item as `user-decision` or `deferred`.

### Core Principles

- **Purpose first**: every finding, disagreement, and outcome exists to improve the user's decision quality.
- **Real job-function reasoning**: reviewers speak from role mandates, not personas.
- **Explicit disagreement is valid**: "I disagree because ..." is desirable.
- **Chair synthesizes, does not flatten**: classify the council state instead of forcing consensus.
- **Bounded deliberation**: turn budgets are mandatory, not advisory.
- **Visible dissent**: material disagreements that survive moderation must appear in the final report.

---

## Contract Resolution

1. `--mode auto` → fail with: `error: --mode auto is not supported.`
2. `--mode lite` → lite mode
3. `--mode council` → council mode
4. profile present, no mode flag → use `profile.default_mode`
5. otherwise → lite mode

### Council Without Profile — MVC Gate

```text
⚠️  Council mode needs a review contract.
    Using defaults (Chair + 2 Reviewers, 6 turns, no auto-fix).
    Press Enter to proceed, or pass --profile <name> for a custom contract.
    Use --verbose to see the full default contract.
```

- Enter / `y` → continue
- `N` → exit with guidance to use `--profile`
- `--ci` → auto-accept, emit: `[WARN] Using MVC defaults for council mode`

### Profile Schema

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
protected_paths:          # gitignore-style globs
  - .env
  - config/trading.*
  - secrets.*
  - '*.pem'
```

---

## Part 1: Generate Reviewers

- **Lite**: generate 3–8 reviewers from scope
- **Council**: use profile roles; without profile, use MVC roles (Chair + reviewer_a + reviewer_b)
- If `ROLE_FILTER` is set, run exactly one reviewer and skip roundtable/council

Required council roles that fail must be surfaced as degraded coverage. If a required role fails, Chair may only produce `DEFER`, `VETO`, or `NO_DECISION`.

---

## Part 2: First-Pass Reviews

Generate all reviewer perspectives before synthesizing. Each reviewer must:
- review only from its assigned role mandate
- explain the issue and why it matters to the user
- propose a recommendation or decision path

**Reviewer output includes:**
```json
{
  "reviewer": "Risk Officer",
  "status": "ok",
  "score": 6,
  "findings": [
    {
      "id": "risk-01",
      "severity": "ERROR",
      "issue": "...",
      "why_it_matters": "...",
      "recommendation": "..."
    }
  ],
  "responses": [
    {
      "finding_id_responded_to": "quant-01",
      "response_type": "disagree",
      "reason": "..."
    }
  ]
}
```

---

## Part 3A: Lite Mode

Each reviewer speaks once. No conflict-round loop. Chair synthesizes.

If reviewers materially disagree → record disagreement, hand decision to user. Do not attempt roundtable resolution. Recommend `--mode council` if deeper deliberation is needed.

---

## Part 3B: Council Mode

**Agenda**: built from high-severity findings, `discussion_priorities`, `conflict_pairs`, and conflicting recommendations. Deprioritized items must be logged with a reason visible to the user.

**Allowed council actions**: support · disagree with reasons · conditional support · propose alternative · escalate risk · mark as user-decision

**Turn budget** (mandatory, not advisory):
- Chair emits turn tracker before each intervention: `Turn tracker: [role_a: 1/2, role_b: 0/2, total: 1/6]`
- Role exhausts turns → further interventions logged as `turn_rejected`
- Session hits `max_total_turns` → Chair must issue outcome or `NO_DECISION`

---

## Required Outputs

These must be emitted. Omitting any is a behavioral failure.

| Field | Lite | Council |
|---|---|---|
| `chair_justification` | ✓ | ✓ |
| `unresolved_disagreements` (array, empty OK) | ✓ | ✓ |
| `deprioritized_items` | — | ✓ |
| `reviewer_status` per reviewer | ✓ | ✓ |
| Turn tracker line per council turn | — | ✓ |

`chair_justification` format: `"Outcome is X because [reasons]. [Finding Y] deprioritized because [reason]."`

---

## Chair Outcome → AUTO_FIX

| Outcome | Meaning | AUTO_FIX on | AUTO_FIX off | CI |
|---|---|---|---|---|
| `APPROVE` | no blockers | no fix | no fix | pass |
| `REQUEST_CHANGES` | changes required | run fix, respect protected paths | emit issue list | neutral |
| `DEFER` | needs more info | no fix | no fix | neutral |
| `VETO` | blocking / unsafe | never fix | never fix | fail |
| `NO_DECISION` | session ended without safe outcome | never fix | never fix | fail |

**AUTO_FIX rules:**
1. `VETO` and `NO_DECISION` always block auto-fix
2. `protected_paths` are never modified
3. `require_human_confirm: true` → show diff, wait for confirmation
4. Protected path encountered mid-fix → halt entire batch before applying anything
5. All outcomes recorded in `fix_log`

---

## Final Report

Separate clearly:
- findings
- agreements
- disagreements (each with `user_action_required: true/false`, `urgency: high/medium/low`)
- deprioritized items
- chair outcome + justification
- degraded coverage warnings
- whether auto-fix ran, was blocked, or was deferred

**Lite mode stays fast. Council mode stays contractual. Both serve the user's decision.**
