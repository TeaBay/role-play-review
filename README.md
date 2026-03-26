# Role-Play Review (RPR)

A Claude Code skill that reviews any content through dynamically generated expert roles — parallel review, roundtable consensus, and auto-fix in a recursive loop.

## How It Works

1. **Role Discovery** — Analyzes your content, generates a roster of expert reviewers (3-20 depending on complexity), each with a distinct personality, focus area, and scoring dimensions
2. **Parallel Review** — All reviewers examine the content simultaneously, producing structured findings with severity ratings and fix suggestions
3. **Roundtable** — Failing reviewers debate findings (AGREE / DISAGREE / COMPROMISE / CHALLENGE), revise scores
4. **Auto-fix** — Applies fixes from ERROR and WARNING findings directly to your files
5. **Loop** — Repeats up to 5 rounds until all reviewers pass (score >= 9/10)

## Features

- Generic — works on any content type (code, writing, game scripts, configs, etc.)
- Recursive sub-agent spawning (depth-limited, globally capped at 30)
- Conflict detection for overlapping fixes
- Regression detection across rounds
- Three auto-fix modes: `on` (automatic), `safe` (confirm before applying), `off`
- Context degradation to manage token usage in later rounds

## Install

Copy the skill file to your Claude Code skills directory:

```bash
# Option A: Copy the file directly
curl -o ~/.claude/skills/role-play-review.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/role-play-review/main/SKILL.md

# Option B: Clone and copy
git clone https://github.com/YOUR_USERNAME/role-play-review.git
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
| Reviewer          | Score | ERROR | WARNING | SUGGESTION | Status |
|-------------------|-------|-------|---------|------------|--------|
| 嚴格架構師        | 9     | 0     | 1       | 2          | PASS   |
| 安全偵探          | 9     | 0     | 0       | 3          | PASS   |
| 效能狂人          | 10    | 0     | 0       | 1          | PASS   |
Trend: 嚴格架構師 R1:6 → R2:8 → R3:9 (↑)
FIXES: 12 APPLIED, 0 FIX_FAILED, 0 CONFLICT, 1 DEFERRED
```

## License

[MIT](LICENSE)
