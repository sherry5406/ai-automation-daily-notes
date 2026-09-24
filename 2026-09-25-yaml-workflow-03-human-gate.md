# AI 自動化教學 11｜YAML Workflow 03：加入 human_gate，處理 BLOCKED 與需要人工確認的節點

> 目標：延續昨天的 `Node + Dependency + on_pass / on_fail`，今天只加入 `human_gate`，讓 Workflow 能安全處理 `BLOCKED` 與「AI 不應自行決定」的情境。

## 今天要完成什麼

昨天我們已經可以描述：

```text
Gate
├── PASS → 下一個 Node
└── FAIL → Auto Fix → Re-Gate
```

今天只增加一個能力：

```text
BLOCKED
   ↓
human_gate
   ↓
人工決定
   ├── APPROVE → 繼續
   └── REJECT  → 停止
```

完成後，我們的 Workflow 第一次具備「AI 自動化，但保留人工決策邊界」的能力。

---

## 1. 為什麼需要 human_gate？

`FAIL` 和 `BLOCKED` 不應該被視為同一種問題。

### FAIL

通常代表：

```text
有輸入
有規則
有結果
但沒有通過
```

例如：

```text
Requirement Gate
→ FAIL
→ Acceptance Criteria 不完整
```

這種情況可以交給 Auto Fix。

### BLOCKED

通常代表：

```text
缺少必要資訊
缺少權限
需要人工決策
或自動化系統無法安全判斷
```

例如：

```text
Figma URL 未提供
API 規格存在兩個互斥版本
刪除既有功能需要產品確認
```

這種情況不應該讓 AI 自己猜。

因此我們加入：

```text
BLOCKED
   ↓
human_gate
```

---

## 2. human_gate 的責任是什麼？

先把責任切乾淨：

```text
Skill
 ↓
產生結果

Gate
 ↓
驗證結果

Auto Fix
 ↓
修正 FAIL

Human Gate
 ↓
處理需要人工決策的情況

Workflow
 ↓
決定下一步
```

注意：

> `human_gate` 不是讓人幫 AI 做所有事情。

它只應該放在：

```text
AI 無法安全自動決定
```

的節點。

---

## 3. 先定義最小 human_gate Contract

今天先採用最簡單的格式：

```yaml
- id: requirement-human-gate
  type: human_gate
  depends_on:
    - requirement-gate
  reason: requirement-blocked
  on_approve:
    - planning
  on_reject:
    - stop
```

這是本教學自己的 Workflow Contract，**不是 Claude Code 官方 YAML 語法**。

Claude Code 官方提供 Skills、Hooks、Subagents、MCP 等能力；我們是在這些能力之上建立自己的 Workflow 層。官方文件也將 Hook 定位為生命週期事件上的自動化機制，而 Skill 則是可重複使用的知識與工作流程。

官方文件：
https://code.claude.com/docs/en/features-overview

---

## 4. `on_approve` / `on_reject` 是什麼？

昨天我們處理的是 Gate：

```yaml
on_pass:
  - planning

on_fail:
  - requirement-auto-fix
```

今天 Human Gate 不使用 `on_pass`，而是明確表達人工決策：

```yaml
on_approve:
  - planning

on_reject:
  - stop
```

白話就是：

```text
人工批准
  ↓
繼續 Workflow

人工拒絕
  ↓
停止 Workflow
```

這樣讀 YAML 時，不需要猜「PASS 是 AI 判定還是人判定」。

---

## 5. 把 BLOCKED 接進昨天的 Workflow

昨天的核心流程：

```text
Requirement
   ↓
Requirement Gate
   │
   ├── PASS → Planning
   │
   └── FAIL → Auto Fix → Re-Gate
```

今天增加 BLOCKED：

```text
Requirement
   ↓
Requirement Gate
   │
   ├── PASS → Planning
   │
   ├── FAIL → Auto Fix → Re-Gate
   │
   └── BLOCKED → Human Gate
```

Workflow 可以寫成：

```yaml
version: 3

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
    on_pass:
      - planning
    on_fail:
      - requirement-auto-fix
    on_blocked:
      - requirement-human-gate

  - id: requirement-auto-fix
    type: auto_fix
    skill: requirement-fix
    depends_on:
      - requirement-gate
    on_pass:
      - requirement-re-gate

  - id: requirement-re-gate
    type: gate
    gate: requirement
    depends_on:
      - requirement-auto-fix
    on_pass:
      - planning
    on_blocked:
      - requirement-human-gate

  - id: requirement-human-gate
    type: human_gate
    depends_on:
      - requirement-gate
    reason: requirement-blocked
    on_approve:
      - planning
    on_reject:
      - stop

  - id: planning
    type: skill
    skill: planning
```

這裡有一個很重要的細節：

```yaml
on_blocked:
  - requirement-human-gate
```

也是我們自己定義的 Workflow Contract。

Claude Code 並沒有一個叫做 `on_blocked` 的官方 YAML Workflow 欄位。

這個欄位只是讓我們自己的 Runner 能清楚表達：

> Gate 回傳 BLOCKED 時，要把控制權交給 Human Gate。

---

## 6. Runner 現在多一個分支

昨天的 Runner 可以理解成：

```js
function nextNodes(node, result) {
  switch (result.status) {
    case 'PASS':
      return node.on_pass ?? [];

    case 'FAIL':
      return node.on_fail ?? [];

    case 'BLOCKED':
      return [];

    default:
      throw new Error(`Unknown gate status: ${result.status}`);
  }
}
```

今天改成：

```js
function nextNodes(node, result) {
  switch (result.status) {
    case 'PASS':
      return node.on_pass ?? [];

    case 'FAIL':
      return node.on_fail ?? [];

    case 'BLOCKED':
      return node.on_blocked ?? [];

    default:
      throw new Error(`Unknown gate status: ${result.status}`);
  }
}
```

這樣：

```js
nextNodes(requirementGate, { status: 'BLOCKED' });
```

就會得到：

```text
requirement-human-gate
```

---

## 7. Human Gate 的結果也要標準化

不要讓 Runner 直接解析：

```text
好的
可以
繼續吧
OK
```

這樣很容易出現歧義。

我們先定義機器可讀結果：

### APPROVED

```json
{
  "status": "APPROVED",
  "node": "requirement-human-gate",
  "reason": "Product owner confirmed the requirement exception"
}
```

### REJECTED

```json
{
  "status": "REJECTED",
  "node": "requirement-human-gate",
  "reason": "Requirement needs clarification"
}
```

因此 Workflow Runner 只需要處理：

```text
APPROVED
   ↓
on_approve

REJECTED
   ↓
on_reject
```

---

## 8. 為什麼不能讓 Human Gate 直接修改 Requirement？

第一版先不要這麼做。

Human Gate 的責任是：

```text
決策
```

不是：

```text
編輯需求
```

例如：

```text
Requirement BLOCKED
       ↓
Human Gate
       ↓
APPROVED
       ↓
Planning
```

如果人工真的要修改 Requirement，應該產生一個新的 Artifact Version：

```text
requirement-v1
      ↓
BLOCKED
      ↓
Human Decision
      ↓
requirement-v2
      ↓
Requirement Gate
```

這樣才能保留：

```text
誰決定
何時決定
決定什麼
```

之後要做 Audit Trail 時會非常有用。

---

## 9. Angular 實際情境

假設今天 Angular 專案收到需求：

> 新增「價格」頁籤。

Requirement Skill 產生：

```json
{
  "feature": "price-tab",
  "acceptanceCriteria": [
    "使用者可以切換價格頁籤"
  ],
  "design": {
    "figmaUrl": null
  }
}
```

Requirement Gate 發現：

```text
Figma URL 是後續 Implementation 必要輸入
```

因此不是一般 FAIL，而是：

```json
{
  "status": "BLOCKED",
  "gate": "requirement",
  "errors": [
    "Figma reference is required before implementation"
  ]
}
```

Workflow：

```text
Requirement Gate
      ↓
   BLOCKED
      ↓
Human Gate
```

人工確認：

> 「這次先不需要 Figma，可以依既有元件規範實作。」

Runner 收到：

```json
{
  "status": "APPROVED"
}
```

然後：

```text
Human Gate
     ↓
APPROVED
     ↓
Planning
```

這就是 Human Gate 最實際的價值：

> AI 不需要猜，人在真正需要決策的地方接手一次即可。

---

## 10. Human Gate 不等於每一步都問人

錯誤設計：

```text
Requirement
 ↓
Human
 ↓
Planning
 ↓
Human
 ↓
Figma
 ↓
Human
 ↓
Implementation
 ↓
Human
```

這樣自動化幾乎沒有價值。

比較好的設計：

```text
Requirement
 ↓
Gate
 ↓
PASS
 ↓
Planning
 ↓
Gate
 ↓
PASS
 ↓
Implementation
 ↓
Test
 ↓
E2E
 ↓
AI Acceptance
 ↓
Human Gate
 ↓
PR
```

人工只出現在：

```text
需要產品決策
需要風險確認
需要最終驗收
```

的地方。

---

## 11. 今天的 Workflow 結構

現在可以整理成：

```text
                  ┌── PASS ──→ Planning
                  │
Requirement → Gate
                  │
                  ├── FAIL ──→ Auto Fix → Re-Gate
                  │
                  └── BLOCKED → Human Gate
                                      │
                              ┌───────┴───────┐
                              ↓               ↓
                          APPROVED         REJECTED
                              ↓               ↓
                          Planning          STOP
```

這已經開始接近真正的 AI Development Automation。

---

## 12. 如何驗證成功

今天至少驗證 5 個案例。

### Test 1：Gate PASS

```json
{
  "status": "PASS"
}
```

預期：

```text
→ planning
```

### Test 2：Gate FAIL

```json
{
  "status": "FAIL"
}
```

預期：

```text
→ requirement-auto-fix
```

### Test 3：Gate BLOCKED

```json
{
  "status": "BLOCKED"
}
```

預期：

```text
→ requirement-human-gate
```

### Test 4：Human APPROVED

```json
{
  "status": "APPROVED"
}
```

預期：

```text
→ planning
```

### Test 5：Human REJECTED

```json
{
  "status": "REJECTED"
}
```

預期：

```text
→ stop
```

---

## 13. 常見問題

### Q1：BLOCKED 為什麼不直接算 FAIL？

因為兩者處理方式不同：

```text
FAIL
 ↓
可以修正
 ↓
Auto Fix
```

而：

```text
BLOCKED
 ↓
缺少資訊／需要決策
 ↓
Human Gate
```

如果全部都算 FAIL，AI 很容易開始猜答案。

---

### Q2：Human Gate 可以讓 AI 自己 approve 嗎？

不建議。

如果這個節點叫 `human_gate`，它的目的就是建立真正的人工作業邊界。

否則只是換了一個名稱，並沒有增加安全性。

---

### Q3：Human Gate 可以修改程式碼嗎？

第一版不要。

先讓 Human Gate 只產生：

```text
APPROVED
REJECTED
```

真正的修改交給下一個 Skill 或明確的人工修改流程。

---

### Q4：`human_gate` 是 Claude Code 官方功能嗎？

不是我們今天 YAML 裡的這個欄位。

`human_gate` 是本教學自己設計的 Workflow Node。

Claude Code 本身有自己的權限、Hook、AskUserQuestion、Plan 等機制；不要把本教學的 YAML Contract 誤認成 Claude Code 原生 Workflow DSL。官方文件目前仍將 Hooks、Skills、Subagents 等視為不同層級的擴充能力。

官方文件：
https://code.claude.com/docs/en/features-overview

---

## 今天學會什麼

今天只新增一個能力：

```text
BLOCKED → Human Decision
```

但這讓我們的 Workflow 有了非常重要的邊界：

```text
AI 可以自動做的
        ↓
自動做

AI 不應自行決定的
        ↓
Human Gate
```

目前整套 Workflow 已經可以表達：

```text
Skill
 ↓
Gate
 ├── PASS → 下一步
 ├── FAIL → Auto Fix → Re-Gate
 └── BLOCKED → Human Gate
                    ├── APPROVED → 下一步
                    └── REJECTED → STOP
```

---

## 下一步

下一篇再處理 **YAML Workflow 04：Parallel Execution**。

屆時會把可以同時執行的工作拆開，例如：

```text
Implementation
      ↓
 ┌────┼────┐
 ↓    ↓    ↓
Lint Type E2E
 └────┼────┘
      ↓
   Test Gate
```

但仍然維持今天建立的規則：

```text
Gate
 ↓
PASS / FAIL / BLOCKED
 ↓
必要時 Human Gate
```

先把單一流程的控制邊界做好，再增加 Parallel，避免 Workflow 一開始就變得過度複雜。

---

## 本篇完成後，整個自動化多了什麼能力？

昨天：

```text
PASS → 下一步
FAIL → Auto Fix → Re-Gate
```

今天：

```text
PASS → 下一步
FAIL → Auto Fix → Re-Gate
BLOCKED → Human Gate
```

也就是：

> **AI 不只是知道什麼時候該繼續，也知道什麼時候必須停下來交給人決定。**
