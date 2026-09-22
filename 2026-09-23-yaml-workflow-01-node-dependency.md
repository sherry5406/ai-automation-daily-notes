# AI 自動化教學 09｜YAML Workflow 01：先把 Skill / Gate 抽象成 Node

> 目標：先做出最小、真的能執行的 YAML Workflow，不一次把 `on_fail`、`human_gate`、Parallel 全部塞進來。

## 今天要完成什麼

前面已經有：

```text
Skill → Gate → Auto Fix → Gate → PASS
```

今天開始把這套流程「資料化」：用 YAML 描述每一個工作節點，以及節點之間的 Dependency。

今天只完成兩個核心欄位：

```yaml
nodes:
  - id: requirement
    type: skill

  - id: requirement-gate
    type: gate
    depends_on:
      - requirement
```

今天先不要做真正的 Workflow Engine。先把 **Workflow Contract** 定義清楚，讓下一篇可以在這個格式上加入 `on_pass` / `on_fail`。

---

## 1. 為什麼需要 YAML Workflow？

如果沒有 Workflow，流程很容易散落在 Prompt、Skill、Hook 和 Script 裡：

```text
Claude 要自己記得：
先 Requirement
再 Gate
再 Planning
再 Gate...
```

問題是：流程越長，越容易漏步驟。

YAML 的角色不是「取代 Claude Code」，而是把流程規則變成一份可以被程式讀取的資料：

```text
Skill / Gate = 做什麼
YAML Workflow = 先做誰、後做誰
Workflow Runner = 依規則執行
```

---

## 2. Node 是什麼？

把每一個可以獨立執行或驗證的工作，視為一個 Node。

例如：

```text
Requirement Skill
Requirement Gate
Planning Skill
Planning Gate
```

可以先抽象成：

```yaml
nodes:
  - id: requirement
    type: skill

  - id: requirement-gate
    type: gate
    depends_on:
      - requirement

  - id: planning
    type: skill
    depends_on:
      - requirement-gate

  - id: planning-gate
    type: gate
    depends_on:
      - planning
```

這裡最重要的是 `id` 與 `depends_on`。

### id

每個 Node 必須有唯一 ID。

### depends_on

表示：

> 「這個 Node 必須等哪些 Node 完成後才能開始？」

因此：

```yaml
planning:
  depends_on:
    - requirement-gate
```

代表 Requirement Gate 沒完成，Planning 就不能開始。

---

## 3. 第一版 Workflow Contract

建立：

```text
.automation/
└── workflow.yml
```

內容：

```yaml
version: 1

workflow:
  id: angular-feature
  name: Angular Feature Development

nodes:
  - id: requirement
    type: skill
    skill: requirement

  - id: requirement-gate
    type: gate
    gate: requirement
    depends_on:
      - requirement

  - id: planning
    type: skill
    skill: planning
    depends_on:
      - requirement-gate

  - id: planning-gate
    type: gate
    gate: planning
    depends_on:
      - planning
```

注意：這是**我們自己的 Workflow Contract**，不是 Claude Code 官方 YAML 語法。

Claude Code 官方目前提供 Skills、Hooks、MCP、CLI 等能力；YAML Workflow Engine 是我們在這些能力之上自行建立的自動化層，因此這裡先從最小資料格式開始。

---

## 4. 為什麼 `type` 與 `skill` / `gate` 要分開？

不要直接寫：

```yaml
- id: requirement
  type: requirement
```

比較好的方式是：

```yaml
- id: requirement
  type: skill
  skill: requirement
```

以及：

```yaml
- id: requirement-gate
  type: gate
  gate: requirement
```

因為未來可能會有：

```yaml
skill
 gate
 human_gate
 script
 agent
```

`type` 是 Node 類型，後面的欄位才指定真正執行什麼。

這樣 Workflow Runner 才能寫成：

```text
if type == skill
    執行 Skill

if type == gate
    執行 Gate

if type == human_gate
    等待人工確認
```

---

## 5. 最小 Workflow Runner

今天先不要寫複雜 Engine。

先建立：

```text
scripts/
└── workflow-runner.mjs
```

第一版只做三件事：

1. 讀 YAML
2. 找到沒有 Dependency 的 Node
3. 依 Dependency 順序輸出執行順序

例如輸出：

```text
[WORKFLOW] angular-feature
[RUN] requirement
[RUN] requirement-gate
[RUN] planning
[RUN] planning-gate
```

這一步甚至可以先不真正執行 Claude Code。

目的只有一個：

> 先確認 YAML 描述的流程是正確的。

---

## 6. Angular 專案中的實際對應

假設 Angular 專案：

```text
my-angular-app/
├── src/
├── angular.json
├── package.json
├── CLAUDE.md
├── .claude/
│   ├── skills/
│   │   ├── requirement/
│   │   └── planning/
│   └── settings.json
├── scripts/
│   ├── requirement-gate.mjs
│   ├── planning-gate.mjs
│   └── workflow-runner.mjs
└── .automation/
    └── workflow.yml
```

整個關係變成：

```text
                    workflow.yml
                         │
          ┌──────────────┴──────────────┐
          ↓                             ↓
   Requirement Skill              Requirement Gate
   .claude/skills/...             scripts/...gate.mjs
          │                             │
          └──────────────┬──────────────┘
                         ↓
                   Planning Skill
                         │
                         ↓
                   Planning Gate
```

這時 YAML 是「流程圖的可執行資料」，而不是把所有商業規則都塞進 YAML。

---

## 7. 如何驗證成功

### 驗證 1：Dependency 是否正確

故意把：

```yaml
planning:
  depends_on:
    - requirement-gate
```

改成不存在的：

```yaml
planning:
  depends_on:
    - xxx
```

Runner 應該直接報錯，而不是猜測。

### 驗證 2：是否遵守順序

預期：

```text
requirement
→ requirement-gate
→ planning
→ planning-gate
```

不能變成：

```text
planning
→ requirement
```

### 驗證 3：Cycle Detection

例如：

```yaml
requirement:
  depends_on:
    - planning

planning:
  depends_on:
    - requirement
```

這形成循環：

```text
requirement → planning → requirement
```

Runner 必須直接 `BLOCKED`，不能無限執行。

---

## 8. 今天先不要加入 on_pass / on_fail

這是今天刻意保留的部分。

如果現在一次加入：

```yaml
on_pass:
on_fail:
human_gate:
parallel:
retry:
auto_fix:
```

Workflow Contract 很快會變得難以理解。

我們現在先把：

```text
Node
  ↓
Dependency
  ↓
Execution Order
```

做穩。

下一篇再加入：

```yaml
on_pass:
  - planning

on_fail:
  - auto-fix
```

如此就可以把目前已經完成的：

```text
Skill → Gate → Auto Fix → Gate → PASS
```

真正搬進 Workflow。

---

## 9. 與 Angular MCP 的關係

Angular 官方目前提供 Angular CLI MCP Server，可以讓 AI Agent 操作 Angular CLI workspace，例如 workspace 分析、build、test、lint 等；官方也建議把 Angular Agent Skills 與 Angular CLI MCP 結合：Skills 提供開發規則，MCP 提供操作工具。citehttps://angular.dev/ai/mcp

因此我們未來可以讓 Node 指向 Angular 能力，例如：

```yaml
- id: angular-test
  type: tool
  tool: angular-cli.run_target
  target: test
  depends_on:
    - implementation
```

但今天先不要實作這一層。

目前分層保持：

```text
Angular Skill
    = Angular 怎麼寫

Angular MCP
    = Angular 可以做什麼

Gate
    = 結果是否合格

Workflow
    = 什麼順序做
```

Angular 官方 Agent Skills 目前包含 `angular-developer` 等 Skills；Angular CLI MCP 則可透過 `npx @angular/cli mcp` 啟動。citehttps://angular.dev/ai/agent-skills

---

## 10. 常見問題

### Q1：YAML 是 Claude Code 官方功能嗎？

不是。

本教學的 YAML 是我們自己建立的 Workflow Contract，用來描述 AI Development Automation 流程。

### Q2：為什麼不直接用 GitHub Actions？

GitHub Actions 很適合 CI/CD，但我們現在處理的是更上層的 Agent Development Workflow，例如：

```text
Requirement
→ Planning
→ Figma
→ Implementation
```

兩者未來可以整合，而不是互相取代。

### Q3：為什麼不讓 Claude 自己決定下一步？

因為「可預期」是自動化的核心。

Agent 可以負責執行與判斷，但重要流程最好由 Workflow 明確定義。

### Q4：今天的 Runner 已經能自動跑 Skill 嗎？

還沒有。

今天先驗證 Workflow Contract 與 Dependency。

下一階段才讓 Runner 真正呼叫 Skill、Gate、Auto Fix。

---

## 今天學會什麼

今天完成了 AI Development Automation 的第一個 Workflow 抽象層：

```text
Skill / Gate
     ↓
   Node
     ↓
Dependency
     ↓
Execution Order
```

最重要的不是 YAML 本身，而是把「流程規則」從 Prompt 裡抽離出來。

現在整個架構變成：

```text
Claude Code
   │
   ├── Skills      → 工作能力
   ├── Hooks       → 觸發機制
   ├── Gate        → 驗收
   ├── Auto Fix    → 修正
   └── MCP         → 工具能力

Workflow YAML      → 流程與依賴
```

---

## 下一步

下一篇：

**YAML Workflow 02｜加入 on_pass / on_fail：把 Skill → Gate → Auto Fix → Re-Gate 串起來**

目標是正式把目前已完成的核心閉環搬進 Workflow：

```text
Requirement Skill
      ↓
Requirement Gate
      ↓
    PASS?
    ↙   ↘
 FAIL   PASS
  ↓       ↓
Auto Fix Planning
  ↓
Re-Gate
  ↓
PASS
```

再下一階段才加入 `human_gate`、Parallel execution 與完整的 Workflow Runner。

---

## 官方文件

- Anthropic Claude Code CLI Reference：
  https://docs.anthropic.com/en/docs/claude-code/cli-usage
- Angular Agent Skills：
  https://angular.dev/ai/agent-skills
- Angular CLI MCP Server：
  https://angular.dev/ai/mcp
