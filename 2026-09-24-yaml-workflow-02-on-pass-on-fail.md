# AI 自動化教學 10｜YAML Workflow 02：加入 on_pass / on_fail，把 Gate 閉環搬進 Workflow

> 目標：在昨天的 `Node + Dependency` 之上，只增加 `on_pass` / `on_fail`，把已經驗證過的 `Skill → Gate → Auto Fix → Re-Gate → PASS` 變成可以描述的 Workflow。

## 今天要完成什麼

昨天我們先完成：

```text
Node
  ↓
Dependency
  ↓
Execution Order
```

今天不重新介紹 YAML，也不急著加入 `human_gate`、Parallel、Retry。

只做一件事：

```text
Gate
 ↓
PASS → 下一個 Node
FAIL → Auto Fix
          ↓
       Re-Gate
```

也就是把目前已經做好的核心閉環，正式放進 Workflow Contract。

---

## 1. `on_pass` / `on_fail` 是什麼？

可以把它想成「Gate 結果的交通號誌」。

### `on_pass`

Gate 通過後，要去哪裡？

```yaml
on_pass:
  - planning
```

意思是：

> Requirement Gate PASS 後，允許進入 Planning。

### `on_fail`

Gate 失敗後，要做什麼？

```yaml
on_fail:
  - requirement-auto-fix
```

意思是：

> Requirement Gate FAIL 後，交給 Auto Fix，而不是直接往下一階段走。

因此流程開始從「只有依賴」變成「有結果分支」。

---

## 2. 先把昨天的 Workflow 擴充

昨天：

```yaml
- id: requirement-gate
  type: gate
  gate: requirement
  depends_on:
    - requirement
```

今天改成：

```yaml
- id: requirement-gate
  type: gate
  gate: requirement
  depends_on:
    - requirement
  on_pass:
    - planning
  on_fail:
    - requirement-auto-fix
```

這裡有一個很重要的觀念：

`depends_on` 與 `on_pass / on_fail` 負責的是不同事情。

```text
depends_on
= 我什麼時候可以開始？

on_pass / on_fail
= 我完成後，依結果要去哪裡？
```

不要把兩者混成一個欄位。

---

## 3. 建立今天的 Workflow Contract

建立或修改：

```text
.automation/
└── workflow.yml
```

第一版：

```yaml
version: 2

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
```

這裡先注意一件事：

這仍然是**本教學自行設計的 Workflow Contract**，不是 Claude Code 官方 YAML 語法。

Claude Code 提供 Skills、Hooks、MCP、CLI 等能力；我們是在這些能力上面建立自己的 Workflow 層。Agent Skills 本身也被 Anthropic 定位成可組合、可重用的程序性能力。citeturn0news18

---

## 4. 現在 Workflow 已經可以表達什麼？

把 YAML 翻成白話：

```text
Requirement Skill
      ↓
Requirement Gate
      │
      ├── PASS → Planning
      │
      └── FAIL → Auto Fix
                    ↓
                 Re-Gate
                    ↓
                  PASS
                    ↓
                 Planning
```

這就是我們前面花幾篇文章建立的核心閉環。

最大的改變是：

> 「下一步做什麼」不再只存在 Claude 的 Prompt 裡，而是開始變成 Workflow Data。

---

## 5. 為什麼 Auto Fix 不直接寫 `on_pass: planning`？

這是一個很重要的安全邊界。

不要設計成：

```yaml
- id: requirement-auto-fix
  type: auto_fix
  on_pass:
    - planning
```

因為這很容易讓人誤解成：

```text
Auto Fix 成功執行
    ↓
直接 PASS
    ↓
Planning
```

但我們前面已經定義：

> **Auto Fix 只能修正，不能宣布 Gate PASS。**

所以應該是：

```text
Auto Fix
   ↓
Re-Gate
   ↓
PASS?
```

因此 YAML 要寫成：

```yaml
- id: requirement-auto-fix
  type: auto_fix
  depends_on:
    - requirement-gate
  on_pass:
    - requirement-re-gate
```

這裡的 `on_pass` 只代表：

> Auto Fix 工作本身完成後，去執行 Re-Gate。

它**不是** Requirement 已經通過。

真正的 PASS 必須由：

```yaml
- id: requirement-re-gate
  type: gate
  gate: requirement
```

重新判定。

---

## 6. Gate 結果要有明確狀態

今天開始，Gate 建議固定輸出：

```json
{
  "status": "PASS",
  "gate": "requirement",
  "errors": [],
  "warnings": []
}
```

FAIL：

```json
{
  "status": "FAIL",
  "gate": "requirement",
  "errors": [
    "Requirement missing acceptance criteria"
  ],
  "warnings": []
}
```

BLOCKED：

```json
{
  "status": "BLOCKED",
  "gate": "requirement",
  "errors": [
    "Required requirement document was not found"
  ],
  "warnings": []
}
```

Workflow Runner 不應該只看 exit code 或文字訊息，而應該把 Gate 結果轉成標準狀態。

---

## 7. BLOCKED 為什麼不能直接走 Auto Fix？

例如：

```text
Requirement Gate
      ↓
   BLOCKED
```

原因可能是：

```text
需求文件不存在
Figma URL 沒提供
必要輸入缺失
權限不足
```

這不是「程式碼寫錯」。

如果讓 Auto Fix 自己猜：

```text
BLOCKED
   ↓
Auto Fix
   ↓
AI 自己補資料
```

就很容易產生假的需求或假的設計。

因此今天的規則固定：

```text
PASS    → on_pass
FAIL    → on_fail
BLOCKED → STOP / Human Gate
```

先不要把 BLOCKED 當成 FAIL。

---

## 8. 修改 Workflow Runner 的責任

昨天 Runner 只需要處理：

```text
Dependency
```

今天開始 Runner 多一個責任：

```text
讀取 Node 結果
      ↓
判斷 status
      ↓
選擇 on_pass / on_fail
```

概念上的執行邏輯：

```text
run(node)
   ↓
result.status
   │
   ├── PASS
   │      ↓
   │   on_pass
   │
   ├── FAIL
   │      ↓
   │   on_fail
   │
   └── BLOCKED
          ↓
       STOP
```

注意：這裡仍然先做「流程控制」，不要急著讓 Runner 自己呼叫 Claude API。

先把 Workflow Engine 的規則驗證好。

---

## 9. 最小 Runner 邏輯

例如可以先用 JavaScript 寫一個非常簡單的 dispatch：

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

這段程式的重點不是複雜度，而是把規則固定下來。

例如：

```js
nextNodes(requirementGate, { status: 'PASS' });
```

結果：

```text
planning
```

如果：

```js
nextNodes(requirementGate, { status: 'FAIL' });
```

結果：

```text
requirement-auto-fix
```

如果：

```js
nextNodes(requirementGate, { status: 'BLOCKED' });
```

結果：

```text
[]
```

這就是我們要的最小 Workflow Decision Engine。

---

## 10. 三種情境實際跑一次

### 情境 A：第一次就 PASS

```text
Requirement Skill
      ↓
Requirement Gate
      ↓
PASS
      ↓
Planning
```

不需要 Auto Fix。

---

### 情境 B：第一次 FAIL

```text
Requirement Skill
      ↓
Requirement Gate
      ↓
FAIL
      ↓
Auto Fix
      ↓
Re-Gate
      ↓
PASS
      ↓
Planning
```

這就是目前核心閉環。

---

### 情境 C：BLOCKED

```text
Requirement Skill
      ↓
Requirement Gate
      ↓
BLOCKED
      ↓
STOP
```

這時候不能讓 AI 自己猜答案。

後面加入 `human_gate` 後，再把：

```text
BLOCKED
   ↓
Human Gate
```

接起來。

---

## 11. Angular 專案中的完整位置

目前可以維持：

```text
my-angular-app/
├── src/
├── angular.json
├── package.json
├── CLAUDE.md
│
├── .claude/
│   ├── skills/
│   │   ├── requirement/
│   │   ├── planning/
│   │   └── requirement-fix/
│   └── settings.json
│
├── scripts/
│   ├── requirement-gate.mjs
│   ├── planning-gate.mjs
│   └── workflow-runner.mjs
│
└── .automation/
    └── workflow.yml
```

責任分工：

```text
Requirement Skill
    ↓
產生 requirement artifact

Requirement Gate
    ↓
驗證 artifact

Requirement Auto Fix
    ↓
修正 artifact

Requirement Re-Gate
    ↓
重新驗證

Workflow
    ↓
決定下一步
```

這個分層非常重要。

---

## 12. 今天如何驗證成功

今天至少做 4 個測試。

### Test 1：PASS 分支

Gate 回傳：

```json
{
  "status": "PASS"
}
```

Runner 應該：

```text
→ planning
```

---

### Test 2：FAIL 分支

Gate 回傳：

```json
{
  "status": "FAIL"
}
```

Runner 應該：

```text
→ requirement-auto-fix
```

---

### Test 3：Auto Fix 完成後不能直接 Planning

確認：

```text
Auto Fix
   ↓
Re-Gate
   ↓
PASS
   ↓
Planning
```

而不是：

```text
Auto Fix
   ↓
Planning
```

---

### Test 4：BLOCKED 必須停止

Gate 回傳：

```json
{
  "status": "BLOCKED"
}
```

Runner 應該：

```text
[BLOCKED] requirement-gate
[STOP]
```

不能執行 Auto Fix。

---

## 13. 常見問題

### Q1：`on_pass` 跟 `depends_on` 有什麼差？

一句話：

```text
depends_on = 開始前的條件
on_pass     = 完成後的成功路徑
on_fail     = 完成後的失敗路徑
```

---

### Q2：為什麼不讓 FAIL 直接回到原本的 Gate？

因為 Gate 只是驗收。

如果 FAIL 後直接再跑一次 Gate：

```text
Gate FAIL
 ↓
Gate
 ↓
Gate
```

問題沒有被修正。

應該是：

```text
Gate FAIL
 ↓
Auto Fix
 ↓
Re-Gate
```

---

### Q3：Auto Fix 失敗怎麼辦？

今天先保持簡單：

```text
Auto Fix
 ↓
FAIL / BLOCKED
 ↓
STOP
```

Retry、最大次數、Human Gate 會在後續文章再加入。

---

### Q4：這是不是已經是完整 Workflow Engine？

還不是。

今天只有：

```text
Node
Dependency
on_pass
on_fail
status
```

還缺：

```text
human_gate
Parallel
Retry
Artifact
Execution log
```

我們會一個一個加，避免一次複雜化。

---

## 今天學會什麼

今天真正完成的不是「多寫兩個 YAML 欄位」，而是把 Workflow 從：

```text
A → B → C
```

提升成：

```text
A
↓
Gate
├── PASS → B
└── FAIL → Fix → Re-Gate → PASS → B
```

因此目前架構已經變成：

```text
                 Workflow
                    │
             ┌──────┴──────┐
             ↓             ↓
           Skill          Gate
                           │
                    ┌──────┼──────┐
                    ↓      ↓      ↓
                  PASS    FAIL  BLOCKED
                    ↓      ↓      ↓
                  Next   AutoFix  STOP
                           ↓
                         Re-Gate
```

這才開始接近真正的 AI Development Automation。

---

## 下一步

下一篇不急著做 Parallel。

先處理一個更實際的問題：

**YAML Workflow 03｜Retry 與 Auto Fix 次數上限**

我們會解決：

```text
Gate FAIL
 ↓
Auto Fix
 ↓
Re-Gate FAIL
 ↓
Auto Fix
 ↓
Re-Gate FAIL
 ↓
？？？
```

避免 AI 一直修、一直跑、一直消耗資源。

下一篇會加入最小版本：

```yaml
retry:
  max_attempts: 2
```

並定義：

```text
PASS    → 下一步
FAIL    → Retry / Auto Fix
BLOCKED → STOP
超過上限 → Human Gate / STOP
```

等這個閉環穩定後，再加入 `human_gate` 與 Parallel execution。

---

## 官方參考

- Anthropic Agent Skills：
  https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- Claude Code 官方文件：
  https://code.claude.com/docs/en/overview

> 注意：本文的 `workflow.yml`、`on_pass`、`on_fail`、`status` 等格式，是本教學建立的 Automation Contract，不是 Claude Code 官方 YAML Workflow 語法。citeturn0news18
