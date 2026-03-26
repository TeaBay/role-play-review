# Role-Play Review (RPR)

A Claude Code skill that reviews any content through dynamically generated expert roles — parallel review, roundtable consensus, and auto-fix in a recursive loop.

## How It Works

1. **Role Discovery** — Analyzes your content, generates a roster of expert reviewers (3-20 depending on complexity), each with a distinct personality, focus area, and scoring dimensions
2. **Parallel Review** — All reviewers examine the content simultaneously, producing structured findings with severity ratings and fix suggestions
3. **Roundtable** — Failing reviewers debate findings (AGREE / DISAGREE / COMPROMISE / CHALLENGE), revise scores
4. **Auto-fix** — Applies fixes from ERROR and WARNING findings directly to your files
5. **Loop** — Repeats up to 5 rounds until all reviewers pass (score >= 8/10)

## Features

- Generic — works on any content type (code, writing, game scripts, configs, etc.)
- Recursive sub-agent spawning (depth-limited, globally capped at 30)
- Automatically responds in the user's language
- Conflict detection for overlapping fixes
- Regression detection across rounds
- Three auto-fix modes: `on` (automatic), `safe` (confirm before applying), `off`
- Token-efficient: context degradation from round 2, roundtable transmits only changed findings

> **Note:** RPR can consume 0.1–6M tokens per run (3–5 roles: 0.1–1M, 6–12 roles: 1–3M, 13–20 roles multi-round: 3–6M). Roughly US$0.01–0.30 per run on Sonnet. Start with a focused scope to avoid large bills.
> Auto-fix modifies files directly — use `AUTO_FIX=safe` to review changes before applying.

## Install

Copy the skill file to your Claude Code skills directory:

```bash
# Option A: Copy the file directly
curl -o ~/.claude/skills/role-play-review.md \
  https://raw.githubusercontent.com/TeaBay/role-play-review/main/SKILL.md

# Option B: Clone and copy
git clone https://github.com/TeaBay/role-play-review.git
cp role-play-review/SKILL.md ~/.claude/skills/role-play-review.md
```

## Usage

In Claude Code:

```
/role-play-review
```

or

```
/rpr
```

Then provide the scope of what you want reviewed. RPR will generate the appropriate expert roles, run the review loop, and produce a final report.

## Example Output

```
ROUND 3/5:
| Reviewer              | Score | ERROR | WARNING | SUGGESTION | Status |
|-----------------------|-------|-------|---------|------------|--------|
| The Iron Architect    | 9     | 0     | 1       | 2          | PASS   |
| Security Hawk         | 9     | 0     | 0       | 3          | PASS   |
| Performance Obsessive | 10    | 0     | 0       | 1          | PASS   |
Trend: The Iron Architect R1:6 → R2:8 → R3:9 (↑)
FIXES: 12 APPLIED, 0 FIX_FAILED, 0 CONFLICT, 1 DEFERRED
```

## License

[MIT](LICENSE)
