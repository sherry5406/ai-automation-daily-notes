# AI 自動化教學 13｜YAML Workflow 05：第一個完整可執行 Workflow

日期：2026-09-27

## 今天要完成什麼

前 4 篇已經建立 Node、Dependency、`on_pass`、`on_fail`、`human_gate`、Parallel。

今天不新增很多語法，而是把這些能力第一次組成一條完整流程：

```text
Requirement
   ↓
Requirement Gate
   ↓
PASS?
 ↙     ↘
FAIL   PASS
 ↓       ↓
Auto Fix Planning
 ↓       ↓
Re-Gate  Planning Gate
           ↓
     Implementation
           ↓
   ┌───────┼────────┐
  Lint  Typecheck  Build
   └───────┼────────┘
           ↓
       Test Gate
           ↓
          E2E
           ↓
      Human Gate
           ↓
           PR
```

今天的目標是讓 Workflow Engine 已經有一個可以「照表執行」的最小版本。

---

## 一、先固定角色

到目前為止，角色分工如下：

```text
Skill     → 產生結果
Gate      → 驗證結果
Auto Fix  → 修正 FAIL
Workflow  → 決定下一步怎麼走
Human     → 處理 BLOCKED / 最終決策
```

這個分工很重要，因為 Workflow 不應該自己重新判斷程式碼是否正確，而是消費前面 Node 的結果。

---

## 二、今天的最小 Workflow Contract

下面 YAML 是**本教學自行設計的 Automation Contract**，不是 Claude Code 官方 YAML 語法。

```yaml
workflow:
  name: angular-feature-delivery
  version: 1

  nodes:
    - id: requirement
      type: skill
      run: requirement-skill
      output: requirement.json
      on_pass: requirement-gate

    - id: requirement-gate
      type: gate
      run: pnpm gate:requirement
      on_pass: planning
      on_fail: requirement-auto-fix
      on_blocked: human-gate

    - id: requirement-auto-fix
      type: agent
      run: auto-fix-requirement
      on_pass: requirement-gate

    - id: planning
      type: skill
      run: planning-skill
      input: requirement.json
      output: planning.json
      on_pass: planning-gate

    - id: planning-gate
      type: gate
      run: pnpm gate:planning
      on_pass: implementation
      on_fail: planning-auto-fix
      on_blocked: human-gate

    - id: planning-auto-fix
      type: agent
      run: auto-fix-planning
      on_pass: planning-gate

    - id: implementation
      type: skill
      run: implementation-skill
      input: planning.json
      output: implementation.json
      on_pass: lint

    - id: lint
      type: gate
      run: pnpm lint
      parallel_group: implementation-tests
      on_pass: test-gate
      on_fail: auto-fix-code

    - id: typecheck
      type: gate
      run: pnpm typecheck
      parallel_group: implementation-tests
      on_pass: test-gate
      on_fail: auto-fix-code

    - id: build
      type: gate
      run: pnpm build
      parallel_group: implementation-tests
      on_pass: test-gate
      on_fail: auto-fix-code

    - id: test-gate
      type: gate
      depends_on:
        - lint
        - typecheck
        - build
      run: test-summary
      on_pass: e2e
      on_fail: auto-fix-code
      on_blocked: human-gate

    - id: auto-fix-code
      type: agent
      run: auto-fix-code
      on_pass: lint

    - id: e2e
      type: gate
      run: pnpm e2e
      on_pass: human-gate
      on_fail: auto-fix-code
      on_blocked: human-gate

    - id: human-gate
      type: human_gate
      on_approve: pr
      on_reject: stop

    - id: pr
      type: action
      run: create-pr

    - id: stop
      type: terminal
```

---

## 三、這條 Workflow 實際怎麼走？

### Case A：一路 PASS

```text
Requirement
 ↓
Requirement Gate PASS
 ↓
Planning
 ↓
Planning Gate PASS
 ↓
Implementation
 ↓
Lint ─────┐
Typecheck ├→ Test Gate PASS
Build ────┘
 ↓
E2E PASS
 ↓
Human Gate
 ↓
APPROVED
 ↓
PR
```

這就是第一條完整 Happy Path。

---

## 四、FAIL：一定回到 Gate

假設 Typecheck FAIL：

```text
Typecheck FAIL
      ↓
  Auto Fix Code
      ↓
     Lint
      ↓
  Typecheck
      ↓
    Build
      ↓
  Test Gate
```

這裡有一個規則不能破：

> Auto Fix 完成，不代表 PASS。

Auto Fix 只能把流程送回 Gate。

```text
FAIL
 ↓
Auto Fix
 ↓
Re-Gate
 ↓
PASS / FAIL / BLOCKED
```

---

## 五、BLOCKED：不要讓 AI 猜

例如：

- 缺少必要環境變數
- 測試環境不存在
- Figma 設計資訊不足
- Requirement 無法判定

流程：

```text
Gate
 ↓
BLOCKED
 ↓
Human Gate
```

Human Gate 再決定：

```text
APPROVED → 繼續
REJECTED → STOP
```

第一版不要讓 Auto Fix 處理 BLOCKED，避免 AI 為了讓流程繼續而自行補猜資訊。

---

## 六、Angular 專案怎麼落地？

先在 `package.json` 建立 deterministic checks：

```json
{
  "scripts": {
    "lint": "ng lint",
    "typecheck": "tsc --noEmit",
    "build": "ng build",
    "e2e": "playwright test",
    "gate:requirement": "node tools/gates/requirement-gate.mjs",
    "gate:planning": "node tools/gates/planning-gate.mjs"
  }
}
```

建議目錄：

```text
.claude/
├── skills/
│   ├── requirement/
│   │   └── SKILL.md
│   ├── planning/
│   │   └── SKILL.md
│   └── implementation/
│       └── SKILL.md
│
├── settings.json
│
.tools/
└── gates/
    ├── requirement-gate.mjs
    ├── planning-gate.mjs
    └── test-summary.mjs

.workflow/
└── angular-feature.yml
```

Claude Code 目前的官方 Skills 做法是以 `SKILL.md` 定義 Skill；專案級 Skill 放在 `.claude/skills/<skill-name>/SKILL.md`，而且 Skill 可以直接用 `/skill-name` 呼叫。citeturn1view0

---

## 七、Workflow Engine 第一版只需要做什麼？

不要一開始就做完整平台。

第一版只需要四件事：

```text
1. 讀 YAML
2. 找到目前 Node
3. 執行 Node
4. 根據結果跳到下一個 Node
```

概念上：

```text
load workflow
    ↓
run node
    ↓
read result
    ↓
if PASS → on_pass
if FAIL → on_fail
if BLOCKED → on_blocked
    ↓
run next node
```

Parallel 只多一個步驟：

```text
找到同一 parallel_group
        ↓
同時執行
        ↓
等待全部完成
        ↓
Join Point
        ↓
Test Gate
```

---

## 八、今天的驗證方式

不要直接拿大型專案測。

先準備一個假的 Workflow Result。

### PASS

```json
{
  "status": "PASS"
}
```

確認：

```text
PASS → 下一個 Node
```

### FAIL

```json
{
  "status": "FAIL",
  "summary": "TypeScript compilation failed"
}
```

確認：

```text
FAIL → Auto Fix → Re-Gate
```

### BLOCKED

```json
{
  "status": "BLOCKED",
  "summary": "Missing environment configuration"
}
```

確認：

```text
BLOCKED → Human Gate
```

### Parallel

確認：

```text
Lint
Typecheck
Build
```

三個都完成之前：

```text
Test Gate = NOT READY
```

---

## 九、常見問題

### Q1：今天是不是已經完成完整 AI Automation？

還沒有。

現在完成的是第一個可執行的 **Workflow Contract**。

真正執行還需要一個 Workflow Runner / Engine。

### Q2：為什麼不直接用 GitHub Actions？

GitHub Actions 很適合執行 CI/CD，但今天的 Workflow 還包含：

- Skill
- Auto Fix Agent
- AI Semantic Gate
- Human Gate

所以我們現在先建立自己的 orchestration layer。

未來可以讓它呼叫 GitHub Actions，而不是取代 GitHub Actions。

### Q3：Workflow Engine 是不是一定要自己寫？

不一定。

今天先自己定義 Contract，是為了先理解 orchestration 的核心概念。

之後再評估要不要交給現有 workflow engine、CI 平台或 Agent SDK 執行。

### Q4：Claude Code 本身會直接執行這個 YAML 嗎？

今天這份 YAML **不代表 Claude Code 原生支援這個格式**。

它是我們自己的 Automation Contract。

Claude Code 官方目前提供 Skills、Hooks 等能力；Skills 可以透過 `SKILL.md` 建立，Hooks 則是事件觸發的自動化機制。citeturn1view0turn1view1

---

## 十、今天完成後，多了什麼能力？

前面是零散能力：

```text
Node
Dependency
Branch
Human Gate
Parallel
```

今天第一次串成：

```text
Requirement
 ↓
Gate
 ↓
Auto Fix / Planning
 ↓
Gate
 ↓
Implementation
 ↓
Parallel Tests
 ↓
E2E
 ↓
Human Gate
 ↓
PR
```

也就是從：

> 「我們有很多自動化零件」

進入：

> **「這些零件可以按照規則串成一條開發流程。」**

---

## 十一、今天學會什麼？

只記住這 5 件事：

1. Workflow 負責 orchestration。
2. Skill 負責產生結果。
3. Gate 負責驗證結果。
4. Auto Fix 修完一定 Re-Gate。
5. BLOCKED 不猜，交給 Human Gate。

---

## 下一步

下一篇進入：

**YAML Workflow 06｜把 Claude Code Hook 接到 Gate**

我們會把現在的：

```text
Workflow
 ↓
Gate
```

再接回 Claude Code：

```text
Claude Code
 ↓
Skill / Agent
 ↓
Hook
 ↓
Gate Script
 ↓
PASS / FAIL / BLOCKED
 ↓
Workflow
```

這一步完成後，Workflow 就不只是「一份 YAML 設計」，而是開始與實際 Claude Code 執行生命週期接起來。
