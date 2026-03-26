---
name: role-play-review
description: "Generic recursive role-play review framework. Dynamically generates expert reviewer roles, runs them in parallel, holds a roundtable debate, then auto-fixes issues in a loop. Use for code reviews, game scripts, API design, or any content needing multiple expert perspectives."
license: MIT
metadata:
  author: TeaBay
  version: "1.4.0"
  compatibility: "Claude Code"
---

# Role-Play Review (RPR)

> Invoke: `/role-play-review` or `/rpr`
> Purpose: Multi-role deep review of any content, achieving consensus through roundtable discussion.

## Quick Start

RPR generates expert reviewer roles for your content, runs them in parallel, holds a roundtable debate, then auto-fixes issues — repeating up to 5 rounds until all reviewers pass (score ≥ 8/10).

**Best for:** Code reviews, game scripts, technical docs, API design — any content that benefits from multiple expert perspectives.

**Cost:** ~0.1–1M tokens (3-5 roles), ~1–3M (6-12 roles), ~3–6M (13-20 roles, multi-round). Roughly US$0.01–0.30 per run on Sonnet. Diff injection from Round 2 reduces re-reading cost. Start with a focused scope.

**Usage:**
```
/rpr
```
Then describe what to review. Examples:

- `Review src/auth/ for security and correctness`
- `Review the dialogue in scenes/act1.lua for character voice consistency`
- `Review my API spec (openapi.yaml) for RESTfulness, error handling, and versioning`

**Auto-fix is ON by default** — RPR will edit your files directly. Use `ADJUST AUTO_FIX=safe` during review to require confirmation, or `ADJUST AUTO_FIX=off` to disable.

**Controls during review:**
- `ACCEPT` — stop early, accept current state
- `STOP` — abort the review
- `ADJUST <instruction>` — change focus, add/remove roles, or change AUTO_FIX mode (takes effect next round)

When ADJUST removes files from scope, existing findings on those files are marked SCOPE_REMOVED and listed in a Moderator message before the next round begins.

---

## Moderator Protocol (Internal — not user-facing)

You are the **Moderator**. You do not review content — you only orchestrate and judge.
Use the **same language as the user's message** for all prose output — propagate this language to all sub-agent prompts. Exception: status tokens (PASS, FAIL, CARRY, TIMEOUT, PARSE_ERROR, ABSENT, ERROR, WARNING, SUGGESTION, DEFERRED, MANUAL_REQUIRED, etc.), JSON field names, and schema-defined string enum values MUST remain in English regardless of output language.

## Parameters

```
MAX_ROUNDS = 5              // override: --max-rounds N
PASS_THRESHOLD = 8          // override: --threshold N
AUTO_FIX = on | safe | off  // override: --auto-fix <mode>, default on
MAX_DEPTH = 2               // fixed
MAX_TOTAL_SUB_AGENTS = 30   // fixed
MAX_SAFE_RETRY = 2          // fixed
MAX_CONFLICT_ROUNDS = 3     // fixed
```

**Invocation override**: Append `KEY=VALUE` or `--flag` after scope. Example: `/rpr --max-rounds 3 --threshold 7 --auto-fix safe Review src/auth/`. Unrecognized keys are ignored with a warning.

**Moderator state** (persist across rounds, initialized before Round 1):
- `round_counter = 0` — increments at Phase A start, before spawning
- `round_history = {}` — `{reviewer: [{round, phase_a_score, revised_score, status}]}` for trend/regression tracking
- `sub_agent_counter = 0` — counts sub-agents spawned by reviewers (depth-2+); top-level reviewers (depth-1) excluded
- `open_conflicts = []` — findings with overlapping fix suggestions
- `open_deferred = []` — findings with vague suggestions
- `fix_log = []` — all fix attempts with status: APPLIED / FIX_FAILED / CONFLICT / DEFERRED / MANUAL_REQUIRED
- `modified_files = []` — file paths modified in current round's Phase E. Lifecycle: reset → [] at Phase A start → populated during Phase E → consumed by CARRY_FORWARD at Phase E end. If Phase E is skipped (D7/D8), remains [].
- `consecutive_zero_applied = 0` — tracks consecutive Phase E rounds with 0 applied fixes; reset on any round with applied > 0

**Session checkpoint**: After each round summary table, Moderator outputs `<!-- RPR_STATE: {compact JSON of all state vars} -->`. If session is interrupted, user pastes the checkpoint to resume. Context degradation rules (below) apply only to reviewer/roundtable prompt content, NOT to Moderator's own state tracking.

---

## Part 1: Role Discovery

### 1. Receive Scope → Read Files → Analyze Domain

User provides review scope. If scope or target files are missing, ask user before proceeding.
Read involved files, identify independently reviewable dimensions (one role per dimension, never merge).
If a file is binary (non-text, non-image, non-PDF): skip it, notify user, and exclude from review targets.

### 2. Generate Role Roster

For each role define:
- Role name (distinctive personality, not generic title) + one-line expertise
- Review focus (3-5 items), scoring dimensions (2–4 short snake_case labels, e.g. api_correctness, error_handling)
- Background context files + review target files
- Recursive spawning conditions (e.g., 'only if findings > 50 and depth < MAX_DEPTH'; default 'never'. Injected into {predefined_conditions} in the reviewer prompt.)

Role count baseline (adjust by complexity and risk): empty/trivial file (0 lines or <10 non-blank lines) → warn user, minimum 1 reviewer; 1-199 lines: 3-5; 200-499 lines: 5-8; 500-2000 lines: 8-12; 2000-4999 lines: 12-15; 5000+ lines or multi-file monorepo: 15-20. Max 20.

### 3. Generate Severity Calibration

Produce domain-specific severity calibration covering at least one example per severity level, as a bullet list: &lt;issue type&gt; → ERROR|WARNING|SUGGESTION. Examples: style preference → SUGGESTION; undocumented breaking change → WARNING; data corruption or injection vector → ERROR.

### 4. Present Roster for User Confirmation

Display: role list, severity calibration, auto-fix mode options, estimated time.
User may add/remove roles, adjust focus, choose auto-fix mode. Proceed to Part 2 after confirmation.
If user explicitly requests to skip roster review (e.g., "just start", "go", "begin", "proceed", "yes") or responds with only role adjustments implying readiness, apply adjustments and proceed directly.

---

## Part 2: Review Loop

One round = Phase A → B → C → D → E. Auto-loop until all PASS or MAX_ROUNDS. Phase B/D may terminate the loop (final report or escalation); Phase C is skipped when Phase B terminates (terminal paths: B-priority-1 escalation or B-priority-3 all-pass); if user chooses "retry this round" after B-priority-1 escalation, a new Phase A is spawned without incrementing round_counter; Phase E is entered only via Phase D route D6; all other D-routes skip it.

**Auto-continue**: After summary table, if not all PASS and under MAX_ROUNDS, spawn next round immediately (no pause). If approaching output limits, Moderator may continue in the next turn. Moderator MUST append the line "(auto-continuing next round) | User may type ACCEPT / STOP / ADJUST to intervene" after every round summary table so control options are always visible in-band.
User may send ACCEPT/STOP/ADJUST between rounds; Moderator never solicits. Exceptions requiring user input (Moderator MUST pause): REGRESSION (Phase B), AUTO_FIX=safe confirmation (Phase E), 0 successful reviewers (Phase B), MAX_ROUNDS reached (Phase D), stagnation early stop (Phase D), only MANUAL_REQUIRED blockers (Phase D), consecutive ABSENT for same reviewer (Phase C).

**ADJUST handler** (when user sends ADJUST mid-run): (1) Acknowledge and describe the change being applied. (2) If files are removed from scope, mark all existing findings scoped to those files as SCOPE_REMOVED and list them. (3) If roles are added/removed, update the reviewer roster for the next round. (4) If AUTO_FIX mode changes, apply immediately. All changes take effect at the start of the next Phase A (current round completes normally).

### Phase A: Parallel Review

Spawn all reviewers in parallel (multiple Agent tool calls in a single response). If reviewer count > 8, spawn in batches of up to 8; wait for each batch to return before spawning the next. If an entire batch fails (all reviewers return infrastructure error, not PARSE_ERROR), retry the batch once before per-reviewer error recovery. This batch-level retry does NOT count against each reviewer's 1-retry-per-round individual quota.

**Spawning limits**: Recursive depth max MAX_DEPTH (depth 1 = top-level reviewers; depth >= MAX_DEPTH cannot spawn further). Global sub-agent cap uses MAX_TOTAL_SUB_AGENTS. At the start of each Phase A, divide remaining sub-agent budget evenly (floor) across all reviewers being spawned this round (not per-batch; CARRY reviewers excluded). Each reviewer's allocation is fixed for the round. If floor = 0 but remaining > 0, assign 1 slot each to the first `remaining` reviewers (by descending complexity), 0 to the rest.

**Context degradation** (Moderator MUST apply when building reviewer prompts for Round 2+):
- Round 1: no previous summary; reviewers read full target files
- Round 2+: Moderator injects Phase E diff into prompt; reviewers verify changed regions without full file re-read (see Diff injection below)
- Round 2: ERROR full (id+severity+location+issue+suggestion) + WARNING slim (id+severity+location+issue, no fix_old/fix_new) + SUGGESTION count only
- Round 3+: ERROR slim (id+severity+location one-liner) + unresolved WARNING slim (id+severity+location) + SUGGESTION omitted
- All rounds: FIXED findings as single count summary line
After each round, Moderator discards full output for PASS/CARRY reviewers; retains scores and trend only.

**Diff injection** (Round 2+): When Phase E applied edits, Moderator captures the unified diff of all modified files. If the diff exceeds 5000 lines, inject a file-level summary instead (modified files + line count changes); reviewers re-read affected regions as needed. In the Round 2+ reviewer prompt, inject this diff (or summary) after the "Previous round summary" block. Reviewers focus verification on changed regions and re-read full files only when surrounding context is needed. If Phase E was skipped (D7/D8) or applied 0 fixes, omit the diff block (no changes to verify).

**Reviewer prompt template** (Moderator constructs and injects — sub-agents only see this prompt). Moderator internal state variables (round_counter, round_history, fix_log, modified_files, open_conflicts, open_deferred, sub_agent_counter) must NEVER be included verbatim in reviewer or roundtable prompts.

```
You are "{role_name}", {expertise_description}.
All output in {output_language}. [Moderator: set this to the user's language]

Scope: {scope}
Review focus: {focus_list}
Severity Calibration: {calibration_guide_or_reference} [Moderator: Round 1 inject full calibration; Round 2+ inject 'Apply your Round 1 severity calibration unchanged.']
Scoring dimensions: {dimension_list}. score = min across all dimensions (0-10 integer). Do not inflate scores.

Context files (must read, not review targets): {context_path_list}
Review target files: {target_path_list}

[Moderator: omit this block for Round 1. Include for Round 2+:]
Previous round summary (JSON array): [{"id": "<finding_id>", "status": "FIXED|OPEN|DEFERRED|CONFLICT|MANUAL_REQUIRED", "note": "<one-line summary>"}]. Include ERROR + WARNING entries per degradation schedule above; SUGGESTION count only. Priority: verify FIXED items first by re-examining relevant file locations.

[Moderator: omit for Round 1 or when no Phase E changes. Include for Round 2+ when Phase E applied fixes:]
Changes applied since your last review:
{phase_e_diff_unified}
Focus verification on changed regions. Re-read full target files only when surrounding context is needed to assess a fix.

Recursive Spawning: conditions {predefined_conditions}, current depth {depth}/{max_depth}, depth >= {max_depth} spawning forbidden. Sub-agent budget remaining: {remaining}/{MAX_TOTAL_SUB_AGENTS}. If remaining = 0, do NOT spawn any sub-agents. When spawning sub-agents, you MUST include "All output in {output_language}" in their prompt.

## Output (strict JSON)
Return ONLY a JSON object — no text before or after. Do NOT use markdown code fences. Output raw JSON only, starting with { and ending with }.
Your output MUST conform to this structure:
{"reviewer": "role_name", "score": 8, "scores_breakdown": {"dimension_name": 8}, "findings": [{"id": "file:severity:slug:line", "severity": "ERROR", "location": "file_path:line", "issue": "description", "suggestion": "proposed fix", "fix_old": "exact original text", "fix_new": "exact replacement text"}], "sub_agent_findings": [], "summary": "one paragraph summary"}
Provide fix_old + fix_new for ERROR and WARNING findings when the fix is clear. fix_old must contain enough surrounding context to be unique in the target file. Each finding must cite a specific location.
```

### Phase B: Judgment

Error recovery (apply first):
- Agent returns error or no response → mark TIMEOUT; empty string response → mark TIMEOUT (logged as EMPTY_RESPONSE); non-JSON or schema-invalid JSON → extraction pipeline: (1) strip leading/trailing non-JSON text, attempt parse of outermost `{...}`, (2) try markdown code fence extraction, (3) if both fail → retry once with reinforced prefix 'CRITICAL: Output raw JSON only — no explanation, no code fences.', (4) if retry fails → mark PARSE_ERROR
- TIMEOUT / PARSE_ERROR excluded from scoring judgment but count as failed for the 80% success threshold (pre-check below). Retries of sub-agents count against sub_agent_counter (top-level reviewer retries do not).
- Each reviewer gets at most 1 total retry per round (whether from parse failure or success threshold retry — not both).

Pre-check:
- Success < 80% (strictly less than; exactly 80% proceeds as PARTIAL_REVIEW without retry) → automatically retry failed reviewers that have NOT yet consumed their 1-retry-per-round quota. Retry-exhausted reviewers remain failed. After retries, continue with available results (mark PARTIAL_REVIEW).

By priority (evaluated after pre-check):
1. 0 successful reviewers → escalate to user (options: retry this round or abort)
2. Regression check (round 2+): any re-spawned reviewer's score drops ≥2 OR drops below PASS_THRESHOLD from above → REGRESSION, escalate (revert last round's Phase E edits if any + re-run / accept + continue / abort). Drop of 1 → MINOR_REGRESSION (logged, no pause).
   - Baseline: previous Phase C revised_score, or Phase A score if no Phase C ran
   - Regression exemption table (first match wins):
     1. CARRY this round — not re-spawned, no current score → EXEMPT
     2. TIMEOUT/PARSE_ERROR this round — no valid current score → EXEMPT
     3. No valid prior score (TIMEOUT/PARSE_ERROR/ABSENT in prior round) → treat as first-time scoring, EXEMPT
     4. Re-spawned from CARRY — held CARRY in immediately preceding round, re-spawned due to modified files → baseline = most recent pre-CARRY valid score from round_history
     5. CHALLENGE_REENTRY — passed reviewer pulled back by CHALLENGE → Phase C score drop is normal; treat as first-time in subsequent Phase A, EXEMPT
3. All scores >= PASS_THRESHOLD AND round is NOT PARTIAL_REVIEW → final report. If PARTIAL_REVIEW with all available scores >= PASS_THRESHOLD → log outcome, continue to next round (absent reviewers cannot be assumed passing).
4. SUGGESTION-only exit: all reviewers have 0 ERROR + 0 WARNING across all findings (only SUGGESTION or no findings) → final report regardless of scores. Log SUGGESTION_ONLY_PASS. Rationale: SUGGESTIONs cannot trigger auto-fix; looping would produce identical results.
5. Any score < PASS_THRESHOLD → Phase C

### Phase C: Roundtable Response

**Roundtable skip**: If all failing reviewers' findings target entirely non-overlapping files (no two failing reviewers share a target file in their findings' location fields), skip Phase C — proceed to Phase D with Phase A scores as final scores for this round. Log ROUNDTABLE_SKIPPED in round summary. Rationale: roundtable debate requires shared concerns; non-overlapping reviewers would only AGREE.

Only spawn reviewers that need to participate: score < PASS_THRESHOLD (failing pool), or (round 2+ only) whose findings were the target of a CHALLENGE (the reviewer named in responding_to) in the immediately preceding round's Phase C AND whose score was >= PASS_THRESHOLD (these are designated CHALLENGE_REENTRY). Failing reviewers named in a CHALLENGE remain in the failing pool and are NOT labeled CHALLENGE_REENTRY.
Passed reviewers carry previous score (CARRY). A PASSED reviewer CHALLENGEd back into roundtable: if its revised_score drops below PASS_THRESHOLD, this is not REGRESSION (normal roundtable outcome), but the reviewer re-enters the failing pool.
Phase C failure (agent error, empty response, or invalid JSON) → carry Phase A score forward as revised_score, mark ABSENT. Phase C retry: shares the 1-retry-per-round limit with Phase B — if a reviewer already used its retry in Phase B this round, Phase C gets no further retry for that reviewer. ABSENT reviewers are NOT exempt from the all-pass threshold check. If the same reviewer is ABSENT for 2 consecutive rounds in which it was spawned (CARRY rounds do not count toward or break the ABSENT streak), escalate to user. ABSENT counter resets when the reviewer successfully responds in any subsequent round.

**Roundtable prompt template** (Moderator MUST inline the full Roundtable Output Schema — sub-agents cannot reference SKILL.md):

```
You are "{role_name}", joining the roundtable.
All output in {output_language}. [Moderator: set this to the user's language]

Scope: {scope} | Focus: {focus_list}
Scoring dimensions: {dimension_list}. revised_score = min across all dimensions (0-10 integer). Do not inflate scores.
Apply your Phase A severity calibration unchanged. Do NOT spawn sub-agents in the roundtable phase.

Your most recent findings (slim JSON — fix_old/fix_new stripped to reduce token cost): {reviewer_own_previous_findings}
Other reviewers' findings summary (max 20 entries; prioritize ERROR > WARNING > SUGGESTION; if truncated append "(+N more omitted)"): {each compressed as reviewer | severity | location | issue}
Unresolved CONFLICTs: {list}

Respond in your professional role: AGREE / DISAGREE / COMPROMISE / CHALLENGE.
CHALLENGE: populate challenge_detail with the challenged issue; if raising a genuinely new finding not in your existing set, include it in the new_findings[] array. Concede gracefully when persuaded.

## Output (strict JSON)
Return ONLY a JSON object — no text before or after. Do NOT use markdown code fences. Output raw JSON only, starting with { and ending with }.
Your output MUST conform to this structure:
{"reviewer": "role_name", "revised_score": 8, "revised_scores_breakdown": {"dimension_name": 8}, "discussion_points": [{"responding_to": "role_name", "position": "AGREE", "argument": "reasoning", "challenge_detail": "", "new_findings": []}], "revised_findings": [{"id": "finding_id", "severity": "ERROR", "location": "file_path:line", "issue": "description", "suggestion": "fix", "fix_old": "old", "fix_new": "new", "change_reason": "reason", "changed": true}], "sub_agent_findings": [], "summary": "one paragraph summary"}
revised_findings must include ONLY findings where changed=true. The Moderator merges these with unchanged findings from the previous round.
```

### Phase D: Re-judgment

By priority:
1. All revised_score >= PASS_THRESHOLD → final report
2. SUGGESTION-only exit (same criteria as B4): all reviewers have 0 ERROR + 0 WARNING across all findings → final report regardless of scores. Log SUGGESTION_ONLY_PASS.
3. Reached MAX_ROUNDS → escalate to user
4. Same failing reviewers with identical finding sets (matched by file+severity+slug components of id, to account for line-number shifts) and same severities for 2 immediately consecutive rounds in which the reviewer was re-spawned (CARRY and CARRY_UNRESOLVED rounds are skipped — they do not count toward or reset this counter) → early stop, escalate. Score-only stagnation (different findings, same score) ≠ early stop.
5. Only blockers are all MANUAL_REQUIRED → escalate to user
6. Failures + AUTO_FIX on/safe + has ERROR/WARNING → Phase E
7. Failures + AUTO_FIX on/safe + no ERROR/WARNING findings (including zero findings) → Phase E skipped; apply CARRY_FORWARD rules with modified_files=[] (no files changed). Note LOW_SCORE_NO_FIXABLE in round summary.
8. Failures + AUTO_FIX=off → Phase E skipped; apply CARRY_FORWARD rules with modified_files=[] (no files changed).

Phase D has no user-confirmation gates (unlike Phase E AUTO_FIX=safe). Escalation paths (D2-D4) terminate the loop rather than pausing for mid-phase approval.

### Phase E: Auto-fix

Collect unresolved ERROR + WARNING findings, sort by severity (ERROR before WARNING); tiebreak by file path then line number.

**Conflict detection (up-front, before any edits):** Two findings conflict if they target the same file and their location line ranges overlap and both have fix_old + fix_new → mark CONFLICT, skip both. Conflicts are transitive: if A conflicts with B and B conflicts with C, skip all three as a group. If a finding's location is whole-file (file path only, no line range), it conflicts with all other findings in the same file that have fix_old + fix_new. Ranges evaluated on original (pre-edit) line numbers only — do not re-check after edits.

After CONFLICTs are identified and excluded, apply remaining findings in sorted order:
- Has `fix_old` + `fix_new` → Edit directly
- If fix_old is not found in the target file, mark FIX_FAILED. If fix_old matches multiple locations: use the finding's location line range to pick the match whose range overlaps the stated location; if no unique match can be determined, mark DEFERRED.
- Vague suggestion (no fix_old/fix_new) → mark DEFERRED
- Edit fails → mark FIX_FAILED

**Counter tracking:** All per-finding outcome classification (APPLIED, FIX_FAILED, CONFLICT, DEFERRED, rejection-skip) and counter updates complete before the 0-applied check below.
- Same finding FIX_FAILED, DEFERRED, or CONFLICT for MAX_CONFLICT_ROUNDS consecutive rounds → MANUAL_REQUIRED (never retried; still scored). CONFLICT groups promoted together.
  Matching rules:
  1. Primary: match by id (fallback: file+severity+slug when line numbers shift); uncertain match = new finding (counter resets)
  2. Secondary: same file+line range with identical fix_old → inherit counter regardless of id
  3. Inheritance: when both primary and secondary match with differing prior counters, inherit the higher value
  4. Reset: counter resets when finding is APPLIED or disappears from all reviewers' findings

**AUTO_FIX=safe:** present all proposed fixes as a batch for user confirmation before applying (exception to auto-continue). User may reject individual fixes: rejected fixes are skipped this round and remain open (not counted as FIX_FAILED). If user rejects all fixes for MAX_SAFE_RETRY consecutive rounds, escalate: offer (a) switch to AUTO_FIX=off, (b) manually review specific fixes, or (c) abort. Otherwise treat as 0 applied (see below).

**This round applied fixes = 0** → increment consecutive_zero_applied counter (Moderator state, initialized to 0, reset on any round with applied > 0). If consecutive_zero_applied >= 2 → escalate to user (options: switch AUTO_FIX mode, provide manual fixes, or abort). Otherwise → skip CARRY_FORWARD, carry passed reviewers (status: CARRY), re-spawn all failing reviewers in next Phase A (user's AUTO_FIX setting unchanged for future rounds). Note: unlike D7/D8 which produce CARRY_UNRESOLVED for failing reviewers, this path forces re-spawn to break a stuck fix loop.

After Phase E, Moderator outputs a modified_files list.

**CARRY_FORWARD**: Compare each reviewer's target files against actually modified files:
1. Overlap exists → re-spawn
2. No overlap + score >= PASS_THRESHOLD → carry previous score (CARRY)
3. No overlap + score < PASS_THRESHOLD → carry previous score (CARRY_UNRESOLVED; findings remain open, highlighted in final report)
4. When in doubt (e.g., reviewer's target file list is empty, or modified_files scope cannot be determined) → re-spawn

Return to Phase A.

---

## Output Schemas

### Reviewer Output Schema

```json
{
  "reviewer": "role_name",
  "score": 8,
  "scores_breakdown": {"<dimension_name (must match scoring dimensions exactly)>": 8},
  "findings": [{
    "id": "<unique, format: file:severity:short_slug[:line] — line optional/informational; cross-round matching uses file+severity+slug>",
    "severity": "ERROR|WARNING|SUGGESTION",
    "location": "<file_path:line[-end_line]> or <file_path> for whole-file issues",
    "issue": "description",
    "suggestion": "proposed fix",
    "fix_old": "<optional — exact original text for auto-fix>",
    "fix_new": "<optional — exact replacement text for auto-fix>"
  }],
  "sub_agent_findings": [],
  "summary": "one paragraph summary"
}
```

score, scores_breakdown values: integer 0-10 (JSON number, not string). score = min across all dimensions.
sub_agent_findings: array of objects with same schema as findings. Always output []; depth >= MAX_DEPTH MUST output []. Moderator flattens all nested findings into the reviewer's findings list.

### Roundtable Output Schema

```json
{
  "reviewer": "role_name",
  "revised_score": 8,
  "revised_scores_breakdown": {"<dimension_name (must match scoring dimensions exactly)>": 8},
  "discussion_points": [{
    "responding_to": "<role_name>",
    "position": "AGREE|DISAGREE|COMPROMISE|CHALLENGE",
    "argument": "reasoning",
    "challenge_detail": "<required if position=CHALLENGE, otherwise omit>",
    "new_findings": []
  }],
  "revised_findings": [{
    "id": "<same as original finding — immutable; match by file+severity+slug>",
    "severity": "ERROR|WARNING|SUGGESTION",
    "location": "<file_path:line[-end_line]> or <file_path> for whole-file issues",
    "issue": "description",
    "suggestion": "proposed fix",
    "fix_old": "<optional>",
    "fix_new": "<optional>",
    "change_reason": "why this finding changed",
    "changed": true
  }],
  "sub_agent_findings": [],
  "summary": "one paragraph summary"
}
```

revised_score, revised_scores_breakdown: integer 0-10 (JSON number).
revised_findings: include ONLY findings where changed=true. Set changed=true on every entry (all included entries have changed, by definition). Omit unchanged findings — Moderator carries them forward automatically.
new_findings: array of finding objects (same schema as Reviewer Output Schema findings[]); include ONLY when position=CHALLENGE raises a genuinely new issue not in the reviewer's existing set; otherwise output []. Moderator assigns a formal id if none given, then merges new_findings into the challenger's active finding set after Phase C.
sub_agent_findings: output [] in roundtable phase (no recursive spawning).

---

## Per-Round Output

```
ROUND {N}/{MAX_ROUNDS}:
| Reviewer | Score | ERROR* | WARNING* | SUGGESTION* | Status |
|----------|-------|--------|----------|-------------|--------|
(* open/unresolved count for this reviewer as of this round)
Status: PASS | FAIL | CARRY | CARRY_UNRESOLVED | TIMEOUT | PARSE_ERROR | ABSENT
[Round flags (if any): PARTIAL_REVIEW | REGRESSION | MINOR_REGRESSION | LOW_SCORE_NO_FIXABLE | ROUNDTABLE_SKIPPED | SUGGESTION_ONLY_PASS]
Trend: {reviewer} R1:X → R2:Y (↑/↓/=)
FIXES: {N} APPLIED, {N} FIX_FAILED, {N} CONFLICT, {N} DEFERRED
(auto-continuing next round) | User may type ACCEPT / STOP / ADJUST to intervene
```

---

## Final Report

Output TL;DR first, then the full report.

```
## TL;DR
Conclusion: {all passed / needs escalation}
Scope: {scope} | Reviewers: {N} | Rounds: {N} | Auto-fix: {mode}
Top ERRORs (if any): 1. ... 2. ... 3. ...
Unresolved: {N} ERROR, {N} WARNING, {N} SUGGESTION

## Full Report
| Reviewer | Final Score | Dimension Breakdown | Trend |
Unresolved Issues (by severity)
CARRY_UNRESOLVED Findings (never re-examined due to no file overlap — requires manual attention)
Unresolved Challenges (raised via CHALLENGE, not yet resolved)
Auto-fix Log (APPLIED / FIX_FAILED / CONFLICT / DEFERRED / MANUAL_REQUIRED)
Severity Calibration Compliance
Moderator Recommendations
```

---

## Limitations

- Phase C is parallel-independent, not real-time interactive discussion
- Sub-agent global cap relies on prompt compliance — Moderator audits via sub_agent_findings
- Moderator context management: round 3+, keep full output only for failing reviewers; passed reviewers → scores-only summary
- Auto-fix precision depends on fix_old/fix_new quality from reviewers
