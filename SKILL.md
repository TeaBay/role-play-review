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
/rpr quick Review docs/api.md
/rpr with security-review profile Review src/
/rpr --ci -j Review src/
```

Natural language and flags are interchangeable — `/rpr --profile security-review Review src/auth/`, `/rpr with security-review Review src/auth/`, and `/rpr security review src/auth/` all do the same thing.

---

## Parameters

Flags are one way to invoke RPR. Natural language is equally valid — the LLM interprets intent, not an argument parser. Use whichever is clearer for your case.

```text
MODE = lite | council          // default council
PROFILE = null                 // review contract
MAX_REVIEWERS = 8              // cap reviewers
AUTO_FIX = safe | off | on    // default safe
OUTPUT_FORMAT = prose          // prose|json|markdown
ROLE_FILTER = null             // single reviewer, no roundtable
CI_MODE = false                // headless pipeline mode
```

Shorthands: `--quick` = lite mode · `--fix`/`--no-fix` = auto-fix on/off · `-j`/`-md` = json/markdown output · `-n N` = max reviewers.

Profile name resolution: `--profile security-review` → looks for `profiles/security-review.yaml` in the skill directory.

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

1. `--mode lite` or `--quick` or natural language requesting quick/fast/lite → lite mode
2. `--mode council` or explicit council request → council mode
3. profile present, no mode flag → use `profile.default_mode`
4. otherwise → council mode

### Council Without Profile — MVC Gate

Council mode without a profile falls back to MVC defaults (Chair + 2 Reviewers, 6 turns, no auto-fix).

- If the user asked for the review in one shot (did not ask to choose a profile): **auto-accept the defaults silently and proceed** — do not interrupt with a confirmation step.
- Otherwise confirm via the AskUserQuestion tool (options: "Use defaults" / "Pick a profile", listing available `profiles/*.yaml` names in the description). Never ask the user to "press Enter" or type y/N — that interaction does not exist in this harness.
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
  "findings": [
    {
      "id": "risk-01",
      "severity": "ERROR",
      "issue": "...",
      "why_it_matters": "...",
      "recommendation": "..."
    }
  ]
}
```

First-pass reviews are generated independently — reviewers have not seen each other's findings yet, so there is no `responses` field here. Disagreement between reviewers happens in the roundtable/council stage (Part 3), not in the first pass.

---

## Part 3A: Lite Mode

Each reviewer speaks once. No conflict-round loop. Chair synthesizes.

If reviewers materially disagree → record disagreement, hand decision to user. Do not attempt roundtable resolution. Recommend council mode if deeper deliberation is needed.

---

## Part 3B: Council Mode

**Agenda**: built from high-severity findings, `discussion_priorities`, `conflict_pairs`, and conflicting recommendations. Deprioritized items must be logged with a reason visible to the user.

**Allowed council actions**: support · disagree with reasons · conditional support · propose alternative · escalate risk · mark as user-decision

**Turn budget** (mandatory, not advisory):
- Track turns internally against `speaking_turns_per_role` and `max_total_turns`
- Role exhausts its turns → it does not speak again on that agenda
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
0. `safe` (the default) = run fix only on `REQUEST_CHANGES`, always honoring `protected_paths` and `require_human_confirm`; every other outcome behaves as "off"
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
