---
name: role-play-review
description: "Role-based review system with lite mode for fast reviews and council mode for chaired deliberation. Reviewers speak from their own job function, can disagree with reasons, and the chair decides whether to approve, request changes, defer, or veto. Supports OpenClaw multi-model dispatch for genuine perspective diversity."
license: MIT
metadata:
  author: TeaBay
  version: "4.0.0"
  compatibility: "Claude Code + OpenClaw"
---

# Role-Play Review (RPR) v4.0

> Invoke: `/role-play-review` or `/rpr`
> **Purpose: Help the user make a better decision by exposing role-grounded findings, trade-offs, disagreements, and decision paths.**

## Quick Start

```text
/rpr Review src/auth/
/rpr --mode lite Review docs/api.md
/rpr --mode council --profile crypto-bot Review docs/polymarket-bot.md
/rpr --engine openclaw --mode council Review docs/strategy.md
/rpr --ci --output json Review src/
```

`--mode auto` is forbidden and must fail immediately.

---

## Parameters

```text
MODE = lite | council          // --mode lite|council; default lite
ENGINE = native | openclaw     // --engine native|openclaw; default native
PROFILE = null                 // --profile <name>
MAX_REVIEWERS = 8              // --max-reviewers N
AUTO_FIX = safe | off | on    // --auto-fix <mode>; default safe
OUTPUT_FORMAT = prose          // --output prose|json|markdown
ROLE_FILTER = null             // --role "<name>" — single reviewer, no roundtable
CI_MODE = false                // --ci
```

---

## Engine Selection

### Native Engine (default)

All reviewers are simulated by the current LLM session. Fast, zero-setup, but all perspectives share the same model's biases.

### OpenClaw Engine

Each reviewer is dispatched to a **separate model** via `openclaw agent --local`. This produces genuinely diverse perspectives because different model families have different reasoning patterns, biases, and blind spots.

**Prerequisites:** OpenClaw CLI installed and configured with at least one model provider.

**How it works:**
1. Chair (current session) generates the reviewer roster and crafts a role-specific prompt for each reviewer
2. Each reviewer prompt is dispatched via: `openclaw agent --local --agent <agent_id> --message "<prompt>" --json`
3. Chair collects all responses, parses findings, and proceeds to synthesis

**Model assignment priority:**
1. Profile `roles[].model` field (explicit per-role)
2. Profile `openclaw.default_model` field
3. OpenClaw's default model (from `openclaw models list`)

**Dispatch rules:**
- Each reviewer call MUST include the full review context (file contents, role mandate, output schema) — OpenClaw agents are stateless per call
- Set `--thinking medium` for complex reviews, `--thinking off` for simple scope
- Parse the `payloads[0].text` field from the JSON response
- If a dispatch fails (timeout, model error), mark that reviewer as `degraded` and continue with remaining reviewers
- Chair MUST note which model produced each reviewer's findings in the final report

**Fallback:** If OpenClaw is unavailable or all dispatches fail, fall back to native engine with a warning:
```text
[WARN] OpenClaw dispatch failed — falling back to native engine (single-model review)
```

---

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
openclaw:                           # OpenClaw engine settings (optional)
  default_model: github-copilot/gpt-5.1
  default_agent: admin_bot          # agent id for dispatch
  thinking: medium                  # off|minimal|low|medium|high|xhigh
roles:
  - id: risk_officer
    mandate: "Protect capital and define loss boundaries"
    model: xai/grok-4               # override model for this role
  - id: quant_strategist
    mandate: "Protect edge quality and market assumptions"
    model: github-copilot/claude-sonnet-4.6
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

### Native Engine Dispatch

Generate all reviewer outputs within the current session sequentially.

### OpenClaw Engine Dispatch

For each reviewer, construct a self-contained prompt and dispatch via bash:

```bash
openclaw agent --local \
  --agent "${AGENT_ID}" \
  --message "${REVIEWER_PROMPT}" \
  --thinking "${THINKING_LEVEL}" \
  --json
```

**Reviewer prompt template** (sent to each OpenClaw agent):

```text
You are a ${ROLE_NAME} reviewing the following content.
Your mandate: ${MANDATE}

## Content to Review
${CONTENT}

## Instructions
1. Review ONLY from your role mandate perspective
2. Output valid JSON matching this schema exactly:
${OUTPUT_SCHEMA}
3. Be specific — cite file paths, line numbers, or sections
4. Score 1-10 where 10 = no issues from your role's perspective
```

**Parallel dispatch:** When MAX_REVIEWERS > 1, dispatch all reviewers in parallel (multiple bash calls in one turn). Collect all results before proceeding to synthesis.

**Model attribution:** Tag each reviewer's findings with the model that produced them:
```json
{
  "reviewer": "Risk Officer",
  "engine": "openclaw",
  "model": "xai/grok-4",
  ...
}
```

**Reviewer output includes:**
```json
{
  "reviewer": "Risk Officer",
  "engine": "native | openclaw",
  "model": "current-session | xai/grok-4",
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

### Council Mode with OpenClaw Engine

When `ENGINE = openclaw`, council deliberation rounds are dispatched to OpenClaw agents:

1. **First pass**: all reviewers dispatched in parallel (see Part 2)
2. **Deliberation rounds**: Chair constructs a round prompt containing the agenda + all prior findings + specific questions, then dispatches each reviewer for their turn
3. **Round prompt template**:
```text
You are ${ROLE_NAME}. Your mandate: ${MANDATE}

## Council Agenda
${AGENDA_ITEMS}

## Prior Findings from All Reviewers
${ALL_FINDINGS_JSON}

## Your Task This Round
Respond to the findings above. You may:
- support (agree with another reviewer's finding)
- disagree (with reasons)
- conditional support (agree if conditions met)
- propose alternative
- escalate risk
- mark as user-decision

Output valid JSON: {"responses": [{"finding_id": "...", "action": "...", "reason": "..."}]}
```

4. Chair collects all round responses, updates the turn tracker, and decides whether another round is needed or an outcome can be issued
5. **Model diversity in deliberation**: each reviewer retains its assigned model across all rounds — the same model that produced the first-pass findings also participates in deliberation

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

**OpenClaw engine additions:**
- model attribution table (which model reviewed which role)
- any degraded reviewers (dispatch failures) and fallback actions taken
- total token usage per reviewer (from OpenClaw meta)

Example model attribution:
```text
| Reviewer         | Engine   | Model                              | Status |
|------------------|----------|------------------------------------|--------|
| Risk Officer     | openclaw | xai/grok-4                         | ok     |
| Security Lead    | openclaw | github-copilot/claude-sonnet-4.6   | ok     |
| API Designer     | openclaw | github-copilot/gpt-5.1             | ok     |
```

**Lite mode stays fast. Council mode stays contractual. Both serve the user's decision.**
