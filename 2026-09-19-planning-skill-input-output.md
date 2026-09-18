# AI 自動化教學 05｜把 Planning 做成第二個 Skill：Input / Output 怎麼設計

## 今天要完成什麼？

前幾篇已經建立：

```text
Requirement Skill
      ↓
Requirement Gate
      ↓
PASS / FAIL / BLOCKED
      ↓
FAIL → Auto Fix → Re-Gate
```

今天往下一個 Node 前進：**Planning Skill**。

目標不是先做一份很複雜的 Planning，而是先把「Requirement PASS 之後，要交給 Planning 什麼、Planning 要產出什麼」定義清楚。

今天完成後，流程會變成：

```text
Requirement Skill
      ↓
Requirement Gate
      ↓
PASS
      ↓
Planning Skill
      ↓
Planning Output
```

這一步很重要，因為後面的 Planning Gate、Figma、Implementation 都會依賴這個 Output。

> Claude Code 的 Skills 是以 `SKILL.md` 為核心，可放在個人 `~/.claude/skills/` 或專案 `.claude/skills/`。本系列實作型 Skill 建議先放專案內，讓流程與團隊一起版本控制。Claude Code 官方文件目前也說明 project skill 位於 `.claude/skills/<skill-name>/SKILL.md`，而 personal skill 位於 `~/.claude/skills/<skill-name>/SKILL.md`。

---

## 1. 先不要急著寫 Planning Skill

先想清楚一件事：

**Planning Skill 的責任是什麼？**

它不是：

```text
Requirement Skill
= 分析需求
Planning Skill
= 再分析一次需求
```

而是：

```text
Requirement
  ↓
已確認「要做什麼」
  ↓
Planning
  ↓
定義「準備怎麼做」
```

所以可以先定義：

| Skill | 責任 |
|---|---|
| Requirement Skill | 把需求整理成可驗收的規格 |
| Requirement Gate | 驗證需求是否完整 |
| Planning Skill | 把已確認需求轉成實作計畫 |
| Planning Gate | 驗證實作計畫是否可執行 |

這樣每個 Node 才不會互相重疊。

---

## 2. Planning Skill 的 Input

第一版不要讓 Planning Skill 直接讀整個世界。

先限制 Input：

```text
PlanningInput
├── requirement.md
├── requirement-result.json
└── project-context
```

其中：

### requirement.md

已經通過 Requirement Gate 的需求。

### requirement-result.json

例如：

```json
{
  "gate": "requirement",
  "status": "PASS",
  "version": 1
}
```

### project-context

例如：

```text
Angular 21
Nx monorepo
pnpm
existing shared components
```

重點是：

> Planning 只能根據已確認的 Requirement 與專案上下文做計畫，不應該自行增加產品需求。

---

## 3. Planning Skill 的 Output

如果 Output 只寫：

```text
完成登入頁
```

後面根本沒有辦法做 Gate。

第一版建議至少包含：

```text
PlanningOutput
├── goal
├── affectedAreas
├── implementationSteps
├── filesToCreate
├── filesToModify
├── dependencies
├── testPlan
└── openQuestions
```

例如：

```json
{
  "goal": "新增課程列表頁",
  "affectedAreas": [
    "apps/web/course",
    "libs/shared/ui"
  ],
  "implementationSteps": [
    "建立 CourseListComponent",
    "建立 Course API service",
    "串接既有 Card component",
    "加入 loading / empty / error state"
  ],
  "filesToCreate": [
    "course-list.component.ts",
    "course.service.ts"
  ],
  "filesToModify": [
    "app.routes.ts"
  ],
  "dependencies": [],
  "testPlan": [
    "component test",
    "API error state test",
    "E2E course list flow"
  ],
  "openQuestions": []
}
```

這樣 Planning Output 才能成為下一個 Gate 的 Input。

---

## 4. 建立第一個 Planning Skill

在 Angular 專案：

```text
.claude/
└── skills/
    └── planning/
        └── SKILL.md
```

`SKILL.md` 第一版可以很簡單：

```markdown
---
name: planning
description: 將已通過 Requirement Gate 的需求轉成可執行的 Angular 實作計畫。當需求已確認，需要規劃檔案、元件、API、測試與實作步驟時使用。
---

# Planning Skill

## Input

讀取：

- 已通過 Gate 的 requirement.md
- Requirement Gate result
- 專案技術上下文

## Rules

1. 不新增產品需求。
2. 不直接修改 Angular source code。
3. 優先重用既有 component、service、utility。
4. 明確列出會新增與修改的檔案。
5. 明確列出測試策略。
6. 不確定的地方放到 openQuestions。
7. 不得自行宣稱 Planning Gate PASS。

## Output

產生：

- goal
- affectedAreas
- implementationSteps
- filesToCreate
- filesToModify
- dependencies
- testPlan
- openQuestions

## Success Criteria

Planning 必須能讓下一個 Implementation Skill 根據 Output 開始工作。
```

這裡先故意不放大量 Angular 細節。

因為 Skill 的第一個責任是**定義流程與契約**，而不是變成一本 Angular 教科書。

---

## 5. 為什麼 Planning 也要有 Output Contract？

因為下一步會是：

```text
Planning Skill
      ↓
Planning Output
      ↓
Planning Gate
```

如果 Planning 每次輸出格式都不同：

```text
今天是 Markdown
明天是 JSON
後天漏掉 testPlan
```

Gate 就很難穩定。

所以現在開始建立一個重要觀念：

> **每個 Skill 都應該有明確 Input / Output Contract。**

這就是之後 YAML Workflow 可以串起來的基礎。

---

## 6. Planning Gate 可以檢查什麼？

今天先不完整實作 Gate，只先定義驗收規則。

例如：

```text
Planning Gate

[ ] goal 存在
[ ] implementationSteps 不為空
[ ] filesToCreate / filesToModify 有明確內容
[ ] testPlan 不為空
[ ] openQuestions 已列出未知事項
[ ] 沒有偷偷新增 Requirement
```

因此流程開始變得很清楚：

```text
Requirement Gate
       ↓
     PASS
       ↓
Planning Skill
       ↓
Planning Gate
       ↓
PASS / FAIL / BLOCKED
```

---

## 7. 最重要的責任分離

現在可以把每個角色固定下來：

```text
Skill
= 產生結果

Gate
= 驗收結果

Hook
= 在適當事件觸發自動化

Auto Fix
= 修正 Gate 指出的問題
```

所以不要做成：

```text
Planning Skill
  ↓
自己判斷「我寫得很好」
  ↓
直接 Implementation
```

而是：

```text
Planning Skill
      ↓
Planning Output
      ↓
Planning Gate
      ↓
PASS?
  ↙     ↘
FAIL    PASS
 ↓        ↓
Auto Fix  Implementation
```

這樣才能持續擴充。

---

## 8. 如何驗證成功？

今天先做三個測試案例。

### Case A：完整 Planning

```text
Requirement PASS
 ↓
Planning
 ↓
所有必要欄位都有
 ↓
Planning Gate PASS
```

### Case B：缺少 testPlan

```text
Planning
 ↓
缺少 testPlan
 ↓
Planning Gate FAIL
```

之後才進 Auto Fix。

### Case C：需求不明確

例如 Requirement 本身寫：

```text
「課程頁做得更好」
```

Planning 不應該自行猜：

```text
加入搜尋
加入排序
加入收藏
```

應該：

```text
Planning
 ↓
openQuestions
 ↓
BLOCKED / Human clarification
```

這會延續前一篇建立的安全原則。

---

## 9. 今天完成後，系統多了什麼能力？

原本：

```text
Requirement
 ↓
Gate
 ↓
PASS
```

現在：

```text
Requirement
 ↓
Requirement Gate
 ↓
PASS
 ↓
Planning Skill
 ↓
Planning Output
 ↓
Planning Gate
```

也就是從：

> 「AI 知道需求」

進一步變成：

> 「AI 能把需求轉成可驗收的實作計畫」。

---

## 常見問題

### Q1：Planning Skill 要不要放全域？

本系列目前建議放**專案內**：

```text
.claude/skills/planning/SKILL.md
```

因為你的 Planning 通常會受到：

```text
Angular 版本
Nx workspace
專案架構
Coding rules
EIP 規範
```

影響。

共通的 Angular 知識可以另外透過 Angular Skill 提供，不要把所有專案規則塞進同一個 Skill。

### Q2：那 Angular Skill 與 Planning Skill 會不會重複？

不應該。

可以想成：

```text
Angular Skill
= Angular 專業能力

Planning Skill
= 我們公司的開發流程能力
```

兩者可以一起使用。

### Q3：為什麼不直接讓 Planning 修改程式？

因為 Planning 與 Implementation 是不同階段。

```text
Planning
= 決定怎麼做

Implementation
= 實際做
```

拆開之後，Planning 可以先被人工或 Gate 驗收，再允許 AI 寫 code。

---

## 今天學會什麼？

今天不是在追求「做出一個很聰明的 Planning Agent」。

而是先建立一個更重要的基礎：

```text
Input Contract
      ↓
Skill
      ↓
Output Contract
      ↓
Gate
```

這會讓後面的：

```text
Figma
Implementation
Test
E2E
AI Acceptance
```

都可以用同一種方式設計。

---

## 下一步

下一篇正式把：

```text
Planning Output
      ↓
Planning Gate
      ↓
PASS / FAIL / BLOCKED
```

做成可以執行的 Gate Script。

完成後就會形成第二個完整閉環：

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

再下一階段才進入 Figma → Implementation。

---

## 官方參考

- Claude Code Skills：`https://code.claude.com/docs/en/skills`
- Claude Code Hooks：`https://code.claude.com/docs/en/hooks`

本文涉及 Claude Code Skills 的儲存位置與 `SKILL.md` 結構，依目前官方文件整理；Planning Input / Output Contract、Gate 與 Auto Fix 則是本系列自訂的 AI Development Automation 架構。
