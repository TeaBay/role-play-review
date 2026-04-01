# Role-Play Review (RPR) v3.0

A Claude Code skill that reviews any content through role-grounded expert perspectives — exposing findings, trade-offs, and disagreements to help you make better decisions.

Two modes: **lite** (fast single-pass) and **council** (chaired deliberation with turn budgets).

## How It Works

### Lite Mode (default)
1. **Role Generation** — Generates 3–8 expert reviewers matched to your content
2. **Parallel Review** — Each reviewer examines the content from their role mandate, producing findings with severity and recommendations
3. **Chair Synthesis** — Chair consolidates findings, surfaces disagreements, and issues an outcome
4. **Auto-fix** — Applies fixes for `REQUEST_CHANGES` outcome (respects `protected_paths`)

### Council Mode
1. **Profile / MVC Contract** — Uses a `--profile` YAML or falls back to MVC defaults (Chair + 2 reviewers)
2. **First-Pass Reviews** — All reviewers produce structured findings
3. **Chaired Deliberation** — Turn-budgeted roundtable; Chair enforces agenda, rejects drift, and tracks turns
4. **Outcome** — `APPROVE` / `REQUEST_CHANGES` / `DEFER` / `VETO` / `NO_DECISION`
5. **Auto-fix** — Runs only on `REQUEST_CHANGES`; blocked on `VETO` and `NO_DECISION`

## Features

- Works on any content type (code, docs, configs, strategies, scripts, etc.)
- Two modes: `--mode lite` (fast) and `--mode council` (contractual deliberation)
- Custom review contracts via `--profile <name>` YAML
- Mandatory turn budgets — deliberation cannot run forever
- Visible dissent — material disagreements surface in the final report
- Auto-fix with `safe` (confirm), `on` (automatic), `off` modes
- Protected paths (gitignore-style globs) — never touched by auto-fix
- `--ci` flag for headless pipelines (auto-accepts prompts, emits JSON-friendly output)

> **Note:** Lite mode is token-efficient (3–8 roles, single pass). Council mode with many turns uses more tokens. Start focused to manage cost.
> Auto-fix modifies files directly — default `--auto-fix safe` shows a diff before applying.

## Install

```bash
# Option A: Copy the file directly
curl -o ~/.claude/skills/role-play-review.md \
  https://raw.githubusercontent.com/TeaBay/role-play-review/main/SKILL.md

# Option B: Clone and copy
git clone https://github.com/TeaBay/role-play-review.git
cp role-play-review/SKILL.md ~/.claude/skills/role-play-review.md
```

## Usage

```
/role-play-review [options] <scope>
/rpr [options] <scope>
```

### Examples

```text
/rpr Review src/auth/
/rpr --mode lite Review docs/api.md
/rpr --mode council --profile crypto-bot Review docs/strategy.md
/rpr --mode council Review docs/design.md
/rpr --ci --output json Review src/
```

### Key Options

| Option | Default | Description |
|--------|---------|-------------|
| `--mode lite\|council` | `lite` | Review mode |
| `--profile <name>` | — | Load a review contract YAML |
| `--auto-fix safe\|on\|off` | `safe` | Auto-fix behaviour |
| `--max-reviewers N` | `8` | Cap number of reviewers |
| `--role "<name>"` | — | Run a single reviewer only |
| `--ci` | `false` | Headless / CI mode |
| `--output prose\|json\|markdown` | `prose` | Output format |

## Example Output

```
Chair Outcome: REQUEST_CHANGES

| Reviewer         | Status | Score | ERROR | WARNING | SUGGESTION |
|------------------|--------|-------|-------|---------|------------|
| Risk Officer     | FAIL   | 5     | 2     | 1       | 0          |
| Security Lead    | PASS   | 8     | 0     | 2       | 3          |
| API Designer     | PASS   | 9     | 0     | 0       | 2          |

Unresolved disagreements: 1
  - risk-01: Risk Officer vs API Designer — severity disputed (user_action_required: true, urgency: high)

chair_justification: "Outcome is REQUEST_CHANGES because risk-01 is a blocker per Risk Officer mandate. API Designer finding api-03 deprioritized — style preference, not blocking."

AUTO_FIX: 3 applied, 0 blocked, 1 deferred (protected path)
```

## License

[MIT](LICENSE)
