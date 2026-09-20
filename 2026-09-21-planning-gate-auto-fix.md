# AI 自動化教學 07｜Planning Gate → Auto Fix → Re-Gate

> 目標：把 Requirement 階段已經驗證過的「Skill → Gate → Auto Fix → Gate → PASS」模式，完整複製到 Planning 階段。

## 今天要完成什麼？

上一集已經把 Planning Gate 做成可執行的 Gate Script。

今天不新增 YAML Workflow，也先不進 Figma，而是完成第二個真正可重複的自動化閉環：

```text
Planning Skill
      ↓
Planning Gate
      ↓
  PASS / FAIL / BLOCKED
       ↓
      FAIL
       ↓
   Auto Fix
       ↓
 Planning Gate
       ↓
     PASS
```

完成後，你的架構會從：

```text
Requirement
  ↓
Gate
  ↓
Auto Fix
  ↓
Gate
  ↓
PASS
```

擴充成：

```text
Requirement
  ↓
Requirement Gate
  ↓
Planning
  ↓
Planning Gate
  ↓
Auto Fix
  ↓
Planning Gate
  ↓
PASS
```

---

## 1. 今天的核心概念：Gate 與 Auto Fix 必須分離

最重要的一條規則：

> **Auto Fix 可以修改，但不能宣布 PASS。**

責任應該保持清楚：

| 元件 | 責任 |
|---|---|
| Planning Skill | 產生 Planning Output |
| Planning Gate | 判斷 Output 是否符合規則 |
| Auto Fix | 根據 FAIL 原因修正內容 |
| Re-Gate | 再次客觀驗證 |
| Human Gate | 處理 AI 無法可靠判斷的決策 |

不要做成：

```text
Planning Skill → 「我覺得沒問題」→ PASS
```

而是：

```text
Planning Skill
     ↓
Planning Gate
     ↓
FAIL
     ↓
Auto Fix
     ↓
Planning Gate
     ↓
PASS
```

這個差異就是「AI 幫忙寫東西」與「AI 開發自動化」的差別。

---

## 2. Planning Gate 應該檢查什麼？

第一版先只檢查可以明確驗證的規則，不要一開始就把所有事情交給 AI。

例如 Planning Output 要求：

```json
{
  "feature": "學習平台影片播放",
  "goal": "限制使用者只能跳轉到已觀看範圍",
  "frontend": [
    "播放進度管理",
    "seek 權限控制"
  ],
  "backend": [
    "回傳已觀看區間"
  ],
  "acceptance": [
    "未觀看區間不可跳轉",
    "已觀看區間可以回看"
  ]
}
```

Gate 第一版可以檢查：

1. `feature` 是否存在
2. `goal` 是否存在
3. `frontend` 是否為陣列且至少一項
4. `backend` 是否為陣列且至少一項
5. `acceptance` 是否為陣列且至少一項
6. 是否明確區分 Frontend / Backend

這些是 deterministic rules，適合 Script。

---

## 3. Gate Result 統一格式

延續前面的設計，讓 Planning Gate 輸出機器可讀 JSON：

```json
{
  "status": "FAIL",
  "gate": "planning",
  "errors": [
    {
      "code": "MISSING_ACCEPTANCE",
      "message": "acceptance 至少需要一項"
    }
  ],
  "warnings": [],
  "retryable": true
}
```

如果是資訊不足而不是內容格式錯誤：

```json
{
  "status": "BLOCKED",
  "gate": "planning",
  "errors": [
    {
      "code": "MISSING_REQUIREMENT",
      "message": "缺少已確認的 Requirement"
    }
  ],
  "warnings": [],
  "retryable": false
}
```

如果全部通過：

```json
{
  "status": "PASS",
  "gate": "planning",
  "errors": [],
  "warnings": [],
  "retryable": false
}
```

### 為什麼一定要有 `retryable`？

因為 Auto Fix 不應該看到所有 FAIL 都直接修。

```text
FAIL + retryable=true
        ↓
     Auto Fix

BLOCKED
        ↓
   不自動猜測
        ↓
 Human / Requirement
```

---

## 4. Auto Fix 的責任

Auto Fix 收到的 Input 不應該只是「請把 Planning 修好」。

應該把 Gate 的失敗原因直接傳進去：

```text
Planning Output
+
Planning Gate Result
+
允許修改範圍
```

例如：

```json
{
  "task": "Fix planning gate failure",
  "gate": "planning",
  "errors": [
    {
      "code": "MISSING_ACCEPTANCE",
      "message": "acceptance 至少需要一項"
    }
  ],
  "allowed_files": [
    "docs/planning/current.json"
  ],
  "max_retries": 2
}
```

這樣 Auto Fix 才是「針對錯誤修正」，而不是讓 Agent 自由發揮。

---

## 5. 第一版 Auto Fix 流程

可以先用最簡單的流程：

```text
run planning-gate
       ↓
status == PASS?
   ↙          ↘
 YES           NO
  ↓             ↓
Continue     retryable?
                ↙  ↘
              NO   YES
              ↓      ↓
           BLOCKED Auto Fix
                      ↓
                run planning-gate
                      ↓
                    PASS?
```

再加上 Retry 上限：

```text
MAX_RETRY = 2
```

完整狀態：

```text
FAIL
 ↓
Auto Fix #1
 ↓
Gate
 ↓
FAIL
 ↓
Auto Fix #2
 ↓
Gate
 ↓
FAIL
 ↓
BLOCKED
```

不要讓 Agent 無限修改。

---

## 6. 可以直接使用的 Shell 骨架

例如建立：

```text
scripts/automation/planning-loop.sh
```

概念骨架：

```bash
#!/usr/bin/env bash
set -euo pipefail

MAX_RETRY=2

for ((attempt=0; attempt<=MAX_RETRY; attempt++)); do
  result=$(node scripts/gates/planning-gate.mjs)
  status=$(node -e "console.log(JSON.parse(process.argv[1]).status)" "$result")

  if [ "$status" = "PASS" ]; then
    echo "$result"
    exit 0
  fi

  if [ "$status" = "BLOCKED" ]; then
    echo "$result"
    exit 2
  fi

  if [ "$attempt" -eq "$MAX_RETRY" ]; then
    echo "$result"
    exit 1
  fi

  node scripts/automation/planning-auto-fix.mjs "$result"
done
```

注意：這只是 orchestration 骨架。

真正的 Gate 邏輯仍然放在：

```text
scripts/gates/planning-gate.mjs
```

Auto Fix 邏輯則放在：

```text
scripts/automation/planning-auto-fix.mjs
```

這樣未來換成 Claude Code、其他 Agent 或 CI，都不需要重新設計 Gate。

---

## 7. Angular 專案裡怎麼使用？

假設你現在做的是 Angular 學習平台：

```text
Requirement
「影片只能跳轉到已觀看範圍」
```

Planning Skill 產生：

```text
Frontend
├─ WatchProgressService
├─ VideoPlayerComponent
└─ seek validation

Backend
├─ watched ranges API
└─ progress persistence

Acceptance
├─ 已觀看範圍可回看
├─ 未觀看範圍不可跳轉
└─ 重新進入課程後保留進度
```

Planning Gate 驗證結構。

如果漏掉 Acceptance：

```text
Planning Gate
     ↓
FAIL
     ↓
MISSING_ACCEPTANCE
     ↓
Auto Fix
     ↓
補上 Acceptance
     ↓
Planning Gate
     ↓
PASS
```

到這一步，才允許進入 Figma / Implementation。

---

## 8. 今天先不要做的事情

這很重要。

今天不要急著加入：

- YAML Workflow Engine
- Parallel Execution
- 完整 AI Semantic Gate
- E2E Agent
- PR 自動建立

因為目前真正重要的是把第二個閉環做穩：

```text
Skill
 ↓
Gate
 ↓
Auto Fix
 ↓
Gate
 ↓
PASS
```

只要 Requirement 與 Planning 都可以使用相同模式，後面才值得抽象成 Workflow Node。

---

## 9. Angular Skill + MCP 放在哪裡？

這也是今天可以開始確立的架構決策。

Angular 官方目前提供官方 Agent Skills，例如 `angular-developer`，用來提供 Angular 開發指導；Angular CLI 也提供 MCP Server，可以讓 Agent 操作 workspace、執行 build/test、搜尋 Angular 官方文件等。citehttps://angular.dev/ai/agent-skills citehttps://angular.dev/ai/mcp

因此後續可以把它們視為「Domain Capability」，而不是 Gate 本身：

```text
Claude Code
   │
   ├─ Angular Skills
   │      ↓
   │   Angular 開發規則
   │
   ├─ Angular MCP
   │      ↓
   │   Workspace / Build / Test / Docs
   │
   └─ Automation Skills
          ↓
       Requirement
       Planning
       Implementation
       E2E
```

**Skill 負責「怎麼做」，MCP 負責「可以操作什麼」，Gate 負責「有沒有做好」。**

Angular 官方也提供 `ng generate ai-config`，可以產生像 `CLAUDE.md`、`AGENTS.md` 與 Angular MCP 設定等 AI configuration。這表示後面進入 Angular 專案整合時，可以優先採用 Angular CLI 官方產生方式，而不是自己複製一套可能過期的設定。citehttps://angular.dev/cli/generate/ai-config

---

## 10. 如何驗證今天成功？

刻意做一個失敗案例：

```json
{
  "feature": "影片播放",
  "goal": "限制 seek"
}
```

執行：

```bash
pnpm planning:gate
```

預期：

```text
FAIL
MISSING_FRONTEND
MISSING_BACKEND
MISSING_ACCEPTANCE
```

接著執行 Auto Fix：

```bash
pnpm planning:auto-fix
```

再執行：

```bash
pnpm planning:gate
```

預期：

```text
PASS
```

最後再故意移除 Requirement，確認：

```text
BLOCKED
```

而不是讓 AI 自己猜一個 Requirement。

---

## 11. 常見問題

### Q1：為什麼不直接讓 Claude 自己檢查？

因為「自己產生、自己判斷、自己宣布通過」容易形成沒有真正驗收的閉環。

Gate 應該成為獨立的品質邊界。

### Q2：Auto Fix 修錯了怎麼辦？

靠三件事情控制：

```text
Allowed Files
+
Retry Limit
+
Re-Gate
```

### Q3：BLOCKED 為什麼不 Auto Fix？

因為 BLOCKED 通常代表「資訊不足」。

例如：

```text
缺少 API Contract
缺少 Requirement
缺少設計稿
```

這時候 AI 不應該自行猜測。

### Q4：Angular MCP 是不是 Gate？

不是。

MCP 是工具能力；Gate 是驗收機制。

例如 Angular MCP 可以執行 build/test，但最後是否 PASS，應該由 Gate 定義明確規則。

---

## 今天學會什麼？

今天完成的是第二個可重複的 AI Automation Loop：

```text
Planning Skill
    ↓
Planning Gate
    ↓
FAIL
    ↓
Auto Fix
    ↓
Re-Gate
    ↓
PASS
```

同時確立三個重要邊界：

1. **Skill 產生結果。**
2. **Gate 驗證結果。**
3. **Auto Fix 只能修正，不能宣布 PASS。**

以及：

> **Skill 是能力、MCP 是工具、Gate 是品質邊界。**

---

## 下一步

下一篇進入：

**Planning → Figma → Implementation 的銜接設計**

重點不是馬上開始切版，而是先定義：

```text
Planning PASS
      ↓
Figma Input Contract
      ↓
Implementation Input Contract
```

讓下一個 Skill 可以可靠接收上一個 Skill 的 Output。

等這個 Contract 穩定後，再開始設計 YAML Workflow 的 Node / Dependency / on_pass / on_fail。
