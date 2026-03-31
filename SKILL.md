---
name: role-play-review
description: "Role-based review council with a chaired legislative-style roundtable. Reviewers speak from their own job function, can raise concerns, disagree with reasons, propose alternatives, and the chair synthesizes consensus, new approaches, and user-decision items."
license: MIT
metadata:
  author: TeaBay
  version: "2.0.0"
  compatibility: "Claude Code"
---

# Role-Play Review (RPR) v2

> Invoke: `/role-play-review` or `/rpr`
> Purpose: Run a role-based review council, then hold a chaired roundtable before sending results to the user.

## Quick Start

RPR v2 works like an internal company review meeting:

1. Build a council of role-specific reviewers
2. Each reviewer submits findings from their own professional perspective
3. A chaired roundtable lets members raise concerns, quote each other, disagree, support conditionally, or propose a new approach
4. The chair synthesizes a final report
5. Optional auto-fix applies only consensus-backed or chair-approved changes

**Default stance:** bounded discussion, explicit disagreement, no recursive debate engine.

**Controls**
- `FINISH` or `ACCEPT` — stop discussion and produce the final report now
- `STOP` — abort
- `ADJUST <instruction>` — change scope, roles, focus, or auto-fix mode before the next phase

**Examples**
```text
/rpr Review src/auth for security and maintainability
/rpr --auto-fix safe Review this trading bot design
/rpr --role "Risk Officer" Review this strategy memo
/rpr --output json --ci Review docs/strategy.md
```

---

## Moderator Protocol (Internal)

You are the **Chair**. You do not act as a domain reviewer. Your job is to:

1. form the right council
2. collect role-based findings
3. run a bounded legislative-style roundtable
4. synthesize agreements, disagreements, and alternatives
5. present a reliable final report

Use the **same language as the user** for prose output. Keep schema keys and enum values in English.

### Core Principles

- **Role fidelity first**: each reviewer must speak from their own functional lens
- **Disagreement is allowed**: "I disagree because ..." is desirable when grounded in role-specific reasoning
- **Bounded deliberation**: the same disputed issue gets at most **2 focused rebuttal turns**
- **Chair synthesis over voting**: this is not a parliament that passes motions by simple majority; the chair summarizes the state of the discussion
- **Minority views preserved**: unresolved but reasonable objections remain visible in the final report
- **No recursive orchestration maze**: no hidden multi-round state machine, no checkpoint protocol, no recursive debate loops

---

## Parameters

```text
MAX_REVIEWERS = 8                 // override: --max-reviewers N
MAX_DISCUSSION_ITEMS = 8          // override: --max-discussion-items N
MAX_REBUTTAL_TURNS = 2            // override: --max-rebuttal-turns N
AUTO_FIX = safe | off | on        // override: --auto-fix <mode>, default safe
OUTPUT_FORMAT = prose             // override: --output prose|json|markdown|csv
ROLE_FILTER = null                // override: --role "<name>"
CI_MODE = false                   // override: --ci
VERIFY_AFTER_FIX = false          // override: --verify-after-fix
```

### CI Mode

When `--ci` is set:
- skip roster confirmation
- treat `AUTO_FIX=safe` as `AUTO_FIX=off` unless the invocation explicitly sets `--auto-fix on`
- if `--output json` is set, return **one final JSON object only**
- never start open-ended discussion; keep only the highest-priority agenda items

---

## Part 1: Build the Council

### 1. Receive Scope

User provides the content or files to review. If the scope is missing, ask once before proceeding.

If the scope is code, design, strategy, or product content, read the relevant files and infer the review surface.

### 2. Generate Role Roster

Create a council with **3-8 reviewers** depending on scope size and risk.

Each role must include:
- `name`
- `job_function`
- `review_lens` (3-5 bullets)
- `target_files_or_topics`
- `what_this_role_cares_about`

For domain-heavy topics, prefer realistic company roles. Example for a crypto trading bot:
- `Quant Lead`
- `Risk Officer`
- `Execution Engineer`
- `Security Lead`
- `Compliance Counsel`
- `Product / Operations Lead`

Do **not** create whimsical personas. Roles should feel like people in the same company meeting.

### 3. Present the Roster

Before review starts, show:
- council roster
- each role's lens
- auto-fix mode
- whether roundtable is enabled

If the user says to proceed immediately, continue.

If `ROLE_FILTER` is set, skip roster generation and run with exactly one reviewer. In that case, skip the roundtable and produce a direct report.

---

## Part 2: First-Pass Role Reviews

Spawn all reviewers in parallel.

Each reviewer must:
- review only from their assigned professional lens
- explain findings in role language
- say **why** the issue matters
- suggest a solution, mitigation, or decision path
- avoid pretending to represent other roles

### Reviewer Output Schema

Return strict JSON:

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

### Reviewer Rules

- `score` is the role's overall confidence in the current design, `0-10`
- `severity` must be one of `ERROR`, `WARNING`, `SUGGESTION`
- every finding must explain both the **issue** and **why it matters**
- every finding should include a recommendation unless truly unknown
- reviewers may be strict, but must remain grounded

---

## Part 3: Legislative-Style Roundtable

This is the heart of RPR v2.

The roundtable exists to create **common sense before user-facing output**.

It is not a free-for-all debate. It is a chaired council meeting with structure.

### 1. Build the Agenda

The Chair builds an agenda from:
- high-severity findings
- findings flagged in `discussion_priorities`
- conflicting recommendations
- disagreements that matter to final user advice

Cap the agenda at `MAX_DISCUSSION_ITEMS`.

Each agenda item should name:
- the finding or topic under discussion
- the roles most affected
- the reason it needs discussion

### 2. Raise-Hand Mechanic

For each agenda item, reviewers may submit a structured intervention:

```json
{
  "reviewer": "Execution Engineer",
  "agenda_id": "A2",
  "action": "raise_hand",
  "stance": "disagree",
  "targets": ["risk-max-drawdown-01", "Risk Officer"],
  "quote_or_reference": "Add a hard portfolio loss cap and kill-switch trigger.",
  "reason": "I disagree because a global kill-switch without venue-aware throttling can worsen slippage during fragmented liquidity events.",
  "alternative": "Use venue-level throttles first, then escalate to a global halt.",
  "decision_signal": "new_approach"
}
```

### Allowed `stance` Values

- `support`
- `disagree`
- `conditional`
- `alternative`
- `risk_escalation`
- `user_decision`

### 3. Discussion Rules

- reviewers may quote another reviewer's wording or finding ID
- reviewers must say **why** they agree or disagree
- reviewers may propose a better alternative than the original suggestion
- reviewers must stay within their role lens
- each agenda item gets at most `MAX_REBUTTAL_TURNS` focused rebuttal turns
- if an issue is clearly a business trade-off, the Chair should stop debate and mark it `USER_DECISION_REQUIRED`

### 4. What Reviewers Are Allowed to Do

Allowed:
- "I disagree because ..."
- "I agree, but only if ..."
- "This is overstated from a security perspective because ..."
- "This conflicts with execution reality because ..."
- "A better approach is ..."

Not allowed:
- endless free-form back-and-forth
- spawning new sub-agents inside the roundtable
- rewriting the entire review scope mid-discussion
- pretending consensus exists when it does not

### 5. Chair Synthesis Per Agenda Item

After discussion, the Chair must classify each agenda item as exactly one of:

- `CONSENSUS`
- `NEW_APPROACH`
- `UNRESOLVED_DISAGREEMENT`
- `USER_DECISION_REQUIRED`

For each item, the Chair records:
- short summary
- who agreed
- who disagreed
- strongest reasoning on each side
- final recommended path

---

## Part 4: Optional Fix Phase

This phase is optional and should happen **after** the roundtable, never before.

### Fix Eligibility

A change may be applied automatically only if it is one of:
- backed by `CONSENSUS`
- a `NEW_APPROACH` explicitly endorsed by the Chair as stronger than the original

Do **not** auto-apply:
- `UNRESOLVED_DISAGREEMENT`
- `USER_DECISION_REQUIRED`
- vague suggestions with no concrete implementation path

### Preferred Patch Format

When a concrete code or text change is needed, prefer structured patch/diff style guidance over fragile free-text replacement.

### Auto-Fix Modes

- `off`: no edits
- `safe`: show proposed changes and wait for confirmation
- `on`: apply eligible changes directly

If `VERIFY_AFTER_FIX=true`, re-run only the reviewers whose role lens directly touches the changed files or changed topic.

---

## Part 5: Final Report

The final report should feel like the output of a serious internal review meeting.

### Required Sections

1. `Council Summary`
2. `Consensus Items`
3. `Newly Synthesized Approaches`
4. `Unresolved Disagreements`
5. `User-Decision Items`
6. `Applied Changes` (if any)
7. `Minority Views`
8. `Recommended Next Step`

### Final Report Rules

- preserve role-specific reasoning where it matters
- do not flatten all disagreement into fake consensus
- if a new approach emerged, explain how it differs from the original
- if the user must decide, frame the trade-off clearly
- if output is JSON, include the same buckets as structured arrays

### JSON Final Output Shape

When `--output json` is requested, return a single JSON object:

```json
{
  "rpr_version": "2.0.0",
  "scope": "string",
  "council": [],
  "consensus_items": [],
  "new_approaches": [],
  "unresolved_disagreements": [],
  "user_decision_items": [],
  "applied_changes": [],
  "minority_views": [],
  "recommended_next_step": "string"
}
```

---

## Operating Notes

- Prefer one strong council meeting over a fragile many-round protocol
- Prefer explicit disagreement over silent averaging
- Prefer chair synthesis over pseudo-mathematical state tracking
- Prefer bounded discussion over endless deliberation
- Prefer exposing trade-offs to the user over forcing fake closure

In short: **RPR v2 is a chaired review council, not a recursive debate machine.**
