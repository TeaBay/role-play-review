# Role-Play Review (RPR)

一個 Claude Code skill，透過動態生成的專家角色審查任何內容——並行審查、圓桌辯論、自動修復，循環執行直到所有人通過。

## 工作原理

1. **角色發現** — 分析你的內容，生成專家審查員名單（視複雜度 3-20 個），每人有獨特個性、專注範圍和評分維度
2. **並行審查** — 所有審查員同時審查內容，輸出帶有嚴重性評級和修復建議的結構化發現
3. **圓桌討論** — 未通過的審查員辯論各自的發現（AGREE / DISAGREE / COMPROMISE / CHALLENGE），修訂分數
4. **自動修復** — 直接將 ERROR 和 WARNING 發現的修復應用到你的文件
5. **循環** — 最多重複 5 輪，直到所有審查員通過（分數 ≥ 8/10）

## 特點

- 通用——適用於任何內容類型（代碼、寫作、遊戲腳本、配置文件等）
- 遞歸 sub-agent 生成（深度限制，全局上限 30 個）
- 自動偵測用戶語言，所有輸出跟隨對話語言
- 衝突檢測（重疊修復）
- 跨輪次回歸檢測
- 三種自動修復模式：`on`（自動）、`safe`（應用前確認）、`off`
- Context 降級管理及圓桌差異輸出，控制 token 用量

## 安裝

將 skill 文件複製到你的 Claude Code skills 目錄：

```bash
# 方式 A：直接複製文件
curl -o ~/.claude/skills/role-play-review.md \
  https://raw.githubusercontent.com/TeaBay/role-play-review/main/SKILL.md

# 方式 B：Clone 後複製
git clone https://github.com/TeaBay/role-play-review.git
cp role-play-review/SKILL.md ~/.claude/skills/role-play-review.md
```

## 使用方法

在 Claude Code 中輸入：

```
/role-play-review
```

或

```
/rpr
```

然後描述要審查的內容。RPR 會生成合適的專家角色，執行審查循環，並生成最終報告。

**審查中途可輸入：** `ACCEPT` 提早接受、`STOP` 中止、`ADJUST` 調整方向。

## 使用示例

- `Review src/auth/ for security and correctness`（代碼安全審查）
- `Review the dialogue in scenes/act1.lua for character voice consistency`（遊戲腳本審查）
- `Review my API spec (openapi.yaml) for RESTfulness, error handling, and versioning`（API 設計審查）

## 輸出示例

```
ROUND 3/5:
| 審查員            | 分數 | ERROR | WARNING | SUGGESTION | 狀態 |
|-------------------|------|-------|---------|------------|------|
| 嚴格架構師        | 9    | 0     | 1       | 2          | PASS |
| 安全偵探          | 9    | 0     | 0       | 3          | PASS |
| 效能狂人          | 10   | 0     | 0       | 1          | PASS |
Trend: 嚴格架構師 R1:6 → R2:8 → R3:9 (↑)
FIXES: 12 APPLIED, 0 FIX_FAILED, 0 CONFLICT, 1 DEFERRED
```

## Token 成本

視乎 scope 和審查員數量，約 **0.5–4M tokens**。建議從聚焦的 scope 開始。

## 授權

[MIT](LICENSE)
