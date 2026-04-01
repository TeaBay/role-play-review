# Role-Play Review (RPR) v3.0

一個 Claude Code skill，透過有角色基礎的專家視角審查任何內容——暴露發現、取捨、分歧，幫助你做出更好的決策。

兩種模式：**lite**（快速單輪）和 **council**（主席主持、有回合上限的深度審議）。

## 工作原理

### Lite 模式（預設）
1. **角色生成** — 按內容生成 3–8 位專家審查員
2. **並行審查** — 每位審查員從自身職能出發，輸出發現與建議
3. **主席綜合** — 整合發現，呈現分歧，給出審查結果
4. **自動修復** — `REQUEST_CHANGES` 結果時執行（遵守 `protected_paths`）

### Council 模式
1. **合約 / MVC** — 使用 `--profile` YAML，或回退至 MVC 預設（主席 + 2 位審查員）
2. **初輪審查** — 所有審查員輸出結構化發現
3. **主席主持審議** — 有回合預算的圓桌；主席執行議程、阻止偏題、追蹤回合數
4. **審查結果** — `APPROVE` / `REQUEST_CHANGES` / `DEFER` / `VETO` / `NO_DECISION`
5. **自動修復** — 僅在 `REQUEST_CHANGES` 時執行；`VETO` 和 `NO_DECISION` 時永遠封鎖

## 特點

- 適用於任何內容類型（代碼、文檔、配置、策略、腳本等）
- 兩種模式：`--mode lite`（快速）和 `--mode council`（契約式審議）
- 透過 `--profile <name>` YAML 自定義審查合約
- 強制回合預算——審議不會無限延伸
- 可見異議——重大分歧會出現在最終報告中
- 自動修復三種模式：`safe`（確認後應用）、`on`（自動）、`off`
- Protected paths（gitignore 格式 glob）——自動修復永不觸碰
- `--ci` 旗標支援 headless 流水線

> **提示：** Lite 模式 token 效率高（3–8 個角色，單輪）。Council 模式回合多時用量較高。建議從聚焦的 scope 開始控制成本。
> 自動修復會直接修改文件——預設 `--auto-fix safe` 會先顯示 diff 再應用。

## 安裝

```bash
# 方式 A：直接複製文件
curl -o ~/.claude/skills/role-play-review.md \
  https://raw.githubusercontent.com/TeaBay/role-play-review/main/SKILL.md

# 方式 B：Clone 後複製
git clone https://github.com/TeaBay/role-play-review.git
cp role-play-review/SKILL.md ~/.claude/skills/role-play-review.md
```

## 使用方法

```
/role-play-review [選項] <範圍>
/rpr [選項] <範圍>
```

### 使用示例

```text
/rpr Review src/auth/
/rpr --mode lite Review docs/api.md
/rpr --mode council --profile crypto-bot Review docs/strategy.md
/rpr --mode council Review docs/design.md
/rpr --ci --output json Review src/
```

### 主要選項

| 選項 | 預設值 | 說明 |
|------|--------|------|
| `--mode lite\|council` | `lite` | 審查模式 |
| `--profile <name>` | — | 載入審查合約 YAML |
| `--auto-fix safe\|on\|off` | `safe` | 自動修復行為 |
| `--max-reviewers N` | `8` | 審查員上限 |
| `--role "<name>"` | — | 只執行單一審查員 |
| `--ci` | `false` | Headless / CI 模式 |
| `--output prose\|json\|markdown` | `prose` | 輸出格式 |

## 輸出示例

```
Chair Outcome: REQUEST_CHANGES

| 審查員        | 狀態 | 分數 | ERROR | WARNING | SUGGESTION |
|---------------|------|------|-------|---------|------------|
| 風險主任      | FAIL | 5    | 2     | 1       | 0          |
| 安全主管      | PASS | 8    | 0     | 2       | 3          |
| API 設計師    | PASS | 9    | 0     | 0       | 2          |

未解決分歧：1 項
  - risk-01：風險主任 vs API 設計師 — 嚴重程度有爭議（user_action_required: true，urgency: high）

chair_justification: "結果為 REQUEST_CHANGES，因為 risk-01 依風險主任職能判斷為阻斷項。api-03 降優先級——風格偏好，不構成阻斷。"

AUTO_FIX：已應用 3 項，已阻擋 0 項，已延遲 1 項（protected path）
```

## 授權

[MIT](LICENSE)
