# Claude Code AI 自動化實戰 01｜從 Skill 到自動化 Gate

> AI 自動化教學｜Angular / EIP Frontend

## 🎯 今天的目標

不要只是讓 AI「幫我寫程式」，而是讓 AI 開始具備：

- 自己檢查
- 自己發現問題
- 自己修正
- 修正後再次驗證
- 只有真正需要人決策的地方才找人

核心概念：

```text
Skill → Gate → Auto Fix → Gate → PASS
```

---

## 🧩 整體架構

目前先不導入 YAML Workflow，先把最核心的自動化迴圈跑通：

```text
Claude Code
   │
   ├── mattpocock/skills
   │       ├── grill-with-docs
   │       ├── to-spec
   │       ├── implement
   │       ├── tdd
   │       └── code-review
   │
   ├── EIP Custom Skills / Rules
   │
   ├── Hooks
   │
   ├── Gate Scripts
   │
   ├── AI Gate
   │
   ├── Figma MCP
   │
   ├── API Mock
   │
   └── E2E
```

這幾個元件的角色不一樣：

| 元件 | 主要角色 |
|---|---|
| Skill | 告訴 AI「怎麼做事」 |
| Rule | 告訴 AI「專案規則」 |
| Hook | 在特定事件自動觸發 |
| Gate Script | 用程式檢查客觀條件 |
| AI Gate | 判斷需求、架構、規格是否符合 |
| Figma MCP | 取得設計資訊 |
| API Mock | 在後端完成前支援前端開發 |
| E2E | 驗證真正的使用者流程 |

---

## ① 需求確認：先讓 AI 找出需求缺口

第一個階段不要直接叫 AI 寫程式。

先使用 `grill-with-docs` 類型的 Skill，把需求文件讀完並提出問題。

例如需求：

```text
新增「公告列表」頁面
```

看起來很簡單，但實際上 AI 應該先確認：

- 路由是什麼？
- API 是什麼？
- Request / Response 格式？
- Loading 怎麼顯示？
- Empty 怎麼顯示？
- Error 怎麼顯示？
- 分頁規則？
- 是否使用 EIP 共用 Table？
- 是否有既有 SCSS？
- Figma 設計在哪？
- Acceptance Criteria 是什麼？

最後產生 Requirement Checklist。

---

## ② AI Gate：需求真的完整嗎？

AI Gate 不是另一個「幫忙寫程式」的 AI。

它比較像工程上的守門員。

結果可以定義成三種：

```text
PASS
FAIL
BLOCKED
```

### PASS

代表目前條件足夠，可以進下一階段。

### FAIL

代表可以明確修正，例如：

```text
FAIL
- 缺少 API Response 定義
- 缺少 Empty State
- 缺少 Error State
```

AI 可以修正文件或補充規格，再重新執行 Gate。

### BLOCKED

代表需要人做決策，例如：

```text
BLOCKED
- Figma 與需求文件對 Table 行為描述不同
- 無法判斷應使用哪個 EIP 共用元件
```

這種問題才真正需要工程師介入。

---

## ③ Skill 與 EIP 規則要分開

這是整套架構很重要的一點。

### mattpocock/skills

比較像「工程方法論」。

例如：

```text
grill-with-docs
→ 先問清楚需求

to-spec
→ 把需求整理成技術規格

implement
→ 依規格實作

tdd
→ 先建立測試再實作

code-review
→ 檢查程式品質
```

### EIP Custom Skills / Rules

則是「我們公司／專案自己的規則」。

例如：

```text
EIP Angular Architecture
EIP Shared Component
EIP SCSS Convention
EIP API Convention
EIP Routing Convention
```

所以可以理解成：

```text
Matt Skills = 怎麼做

EIP Rules = 我們這個專案要怎麼做
```

兩者搭配才適合大型企業前端。

---

## ④ 不要一開始就做 Workflow Engine

現在先不要急著建立：

```text
YAML Workflow
Workflow Engine
Agent Orchestrator
```

因為如果底層的 Skill、Gate、Hook 都還沒有穩定，Workflow 只是在把不穩定的流程自動化。

第一階段真正應該證明的是：

```text
Skill
 ↓
Gate
 ↓
Fail?
 ↓
AI Auto Fix
 ↓
Gate
 ↓
PASS
```

只要這個 Loop 可以穩定跑起來，再往後面擴充才有價值。

---

## ⑤ 最終想達成的前端流程

完整版本會逐步變成：

```text
需求規格
   ↓
Grill
   ↓
AI Gate
   ↓
To Spec
   ↓
人工架構確認
   ↓
Figma MCP ───── API Mock
   ↓                 ↓
UI 實作          API 實作
   ↓                 ↓
EIP UI Gate      EIP Code Gate
   └────────┬────────┘
            ↓
       自動化測試
            ↓
          E2E
            ↓
         AI Gate
            ↓
      Auto Fix / PASS
            ↓
       Final Acceptance
            ↓
            PR
```

---

## 🧪 今天先實作哪一段？

第一個實驗建議只做：

```text
Claude Code
    ↓
Matt Pocock Skill
    ↓
Requirement Checklist
    ↓
Gate Script
    ↓
PASS / FAIL
```

成功後再加入：

```text
Hook
→ 自動觸發 Gate
```

再加入：

```text
FAIL
→ Claude 自動修正
→ 再跑 Gate
```

最後才串：

```text
Figma MCP
API Mock
E2E
```

---

## 📌 今天的重點

> **Skill 負責做事，Gate 負責決定能不能繼續，Hook 負責自動觸發。**

真正成熟的 AI Coding Workflow，不是讓 AI 一次寫完全部程式，而是讓 AI 在每個重要節點都能「自己驗證」。

下一篇將進入實作：**Claude Code + mattpocock/skills 安裝與 `/setup-matt-pocock-skills`，並開始建立第一個 EIP 專案 Skill。**
