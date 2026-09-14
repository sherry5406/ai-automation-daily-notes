# AI 自動化教學 01｜先把 Requirement 做成第一個 Skill

> 今天開始正式進入「一步一步做出 AI Development Automation」的主線。
>
> 今天只做一件事：把「需求整理」從一段 Prompt，拆成一個可以重複使用的 Claude Code Skill。

## 今天要完成什麼？

今天完成後，你會有：

```text
.claude/
└── skills/
    └── requirement/
        └── SKILL.md
```

之後可以直接在 Claude Code 執行：

```text
/requirement
```

讓 Claude 按照固定流程整理需求，而不是每次重新貼一大段 Prompt。

---

## 1. 先理解：什麼時候該做 Skill？

如果你常常對 Claude Code 說：

```text
請先讀需求
→ 找出不清楚的地方
→ 整理功能範圍
→ 列出驗收條件
→ 不要急著寫 Code
```

這其實已經是一個「固定流程」。

Claude Code 官方目前的 Skills 設計，就是把這類可重複的指令、Checklist、多步驟流程放進 `SKILL.md`。Skill 可以由 Claude 在符合情境時自動使用，也可以直接用 `/skill-name` 呼叫。citeturn1view0

而且 Skill 的內容只有在使用時才載入，不需要把所有流程都塞進 `CLAUDE.md`。citeturn1view0

所以第一個原則很簡單：

> **CLAUDE.md 放「這個專案是什麼、有哪些規則」；Skill 放「要怎麼完成一件事」。**

---

## 2. Skill 的基本結構

建立：

```text
.claude/
└── skills/
    └── requirement/
        └── SKILL.md
```

`SKILL.md` 最基本由兩部分組成：

1. YAML frontmatter：告訴 Claude 什麼時候使用
2. Markdown body：真正的執行流程

這是目前 Claude Code 官方文件的 Skill 結構。citeturn1view0

---

## 3. 第一個 Requirement Skill

建立 `.claude/skills/requirement/SKILL.md`：

```md
---
name: requirement
description: 整理前端需求，產生可實作與可驗收的 Requirement。當使用者提供需求、規格或功能描述，且需要釐清範圍、找出缺漏或整理驗收條件時使用。
---

# Requirement Skill

## 目標

把原始需求整理成「可以交給 Planning 與 Implementation 使用」的 Requirement。

## 執行流程

1. 讀取使用者提供的需求與相關專案文件。
2. 整理 User Story / 功能目標。
3. 列出 In Scope。
4. 列出 Out of Scope。
5. 找出缺少或模糊的資訊。
6. 整理 Acceptance Criteria。
7. 不要自行猜測商業規則。
8. 若關鍵資訊不足，標記為 BLOCKED，並列出需要人工確認的項目。

## Output

最後輸出：

### Requirement
- 功能目標
- 使用者流程
- In Scope
- Out of Scope
- Acceptance Criteria

### Open Questions
- 尚未確認的問題

### Status
- PASS：需求足夠明確，可以進入 Planning
- BLOCKED：仍有關鍵資訊需要人工確認
```

---

## 4. 為什麼現在先不要做 Hook？

我們目前刻意不急著加入 Hook。

因為現在要先驗證：

```text
Skill
 ↓
真的能穩定產生一致的 Output
```

等 Requirement Skill 穩定後，下一階段才加入：

```text
Skill
 ↓
Hook
 ↓
Requirement Gate
```

這樣比較容易知道問題到底出在 Skill，還是出在自動化流程。

---

## 5. Input / Output 怎麼設計？

這是之後所有 Skill 都要遵守的核心概念。

### Input

Requirement Skill 的 Input 可以是：

```text
使用者需求
+ PRD / requirement.md
+ 現有專案資訊
+ API 規格
+ UI / Figma 資訊（如果已存在）
```

### Output

不要只輸出一段漂亮的文字，而要輸出下一個 Node 可以使用的資料：

```text
Requirement
├── Goal
├── Scope
├── User Flow
├── Acceptance Criteria
├── Open Questions
└── Status
```

這個設計非常重要。

因為我們最後想做到的是：

```text
Requirement Skill
       ↓
Requirement Gate
       ↓
Planning Skill
```

如果 Requirement 沒有固定 Output，下一個階段就很難自動化。

---

## 6. 在 Claude Code 裡測試

進入 Angular 專案後啟動 Claude Code：

```bash
claude
```

先確認 Skill 有被載入：

```text
/skills
```

Claude Code 官方文件目前也提供 `/skills` 來查看可用 Skills。citeturn1view0

然後直接測試：

```text
/requirement

我要新增一個會員列表頁。
可以搜尋會員姓名，點擊會員後可以看到詳細資料。
```

理想結果不是直接開始寫 Angular，而是先產生：

```text
Requirement

Goal
建立會員列表與詳細資料流程。

In Scope
- 會員列表
- 姓名搜尋
- 會員詳細資料

Out of Scope
- 會員新增
- 會員刪除

Acceptance Criteria
- 可以輸入姓名搜尋
- 可以點擊會員
- 可以進入會員詳細資料

Open Questions
- 搜尋 API 是什麼？
- 詳細資料 API 是什麼？

Status
BLOCKED
```

這就是我們要的第一個「可被 Gate 檢查的輸出」。

---

## 7. 今天先建立最小閉環

今天的自動化還很小：

```text
User Requirement
       ↓
Requirement Skill
       ↓
Structured Requirement
```

下一步才會變成：

```text
User Requirement
       ↓
Requirement Skill
       ↓
Requirement Gate
       ↓
PASS / BLOCKED
```

再下一步：

```text
FAIL
 ↓
Auto Fix
 ↓
Requirement Gate
 ↓
PASS
```

這就是我們要慢慢做出來的核心：

```text
Skill → Gate → Auto Fix → Gate → PASS
```

---

## 8. 專案 Skill 還是 Global Skill？

這裡先建立一個重要觀念。

Claude Code 目前支援多個 Skill 載入層級：

```text
~/.claude/skills/          → 個人，全專案
.claude/skills/            → 專案
<subdir>/.claude/skills/   → 子目錄 / Monorepo
```

官方文件指出，放在 `~/.claude/skills/` 的 Skill 會套用到這台機器上的所有專案；放在專案 `.claude/skills/` 則只在該 Repository 的 Session 載入。citeturn1view0

對我們現在這套 EIP / Angular 自動化架構，我建議：

```text
通用能力
~/.claude/skills/

專案流程
.claude/skills/
```

例如：

```text
~/.claude/skills/
└── code-review/

EIP.Web/.claude/skills/
├── requirement/
├── planning/
├── implementation/
└── e2e/
```

因為不同 Angular 專案可能有不同版本、規範與 Gate，不應該全部塞進 Global。

---

## 9. 如何驗證成功？

完成後檢查：

```text
.claude/skills/requirement/SKILL.md
```

並確認：

```text
/skills
```

可以看到 `requirement`。

再執行：

```text
/requirement
```

確認 Claude 是否會先整理需求，而不是直接開始寫 Code。

### 成功標準

```text
✅ Skill 可以被找到
✅ /requirement 可以手動執行
✅ 有固定 Input
✅ 有固定 Output
✅ 有 PASS / BLOCKED
✅ 不會直接進入 Implementation
```

---

## 常見問題

### Q1：是不是每個 Skill 都要放很多 Prompt？

不是。

官方目前也建議 Skill body 保持精簡，因為 Skill 載入後內容會留在 context 中。citeturn1view0

所以不要把整本公司規範都塞進一個 Skill。

---

### Q2：Requirement Skill 可以直接叫 Planning Skill 嗎？

現在先不要。

先讓：

```text
Requirement Skill
```

穩定後，再設計：

```text
Requirement
 ↓
Planning
```

這樣才能清楚測試每個 Node 的責任。

---

### Q3：Skill 跟 CLAUDE.md 最大差別？

最簡單記法：

```text
CLAUDE.md
= 專案規則 / 長期背景

Skill
= 可重複執行的工作流程
```

這個界線會是我們後面設計整套 AI Automation 的基礎。

---

## 今天學會什麼？

今天只需要記住 4 件事：

1. **Skill 是可重複使用的工作流程。**
2. **SKILL.md = Frontmatter + Instructions。**
3. **Skill 一定要有清楚的 Input / Output。**
4. **現在先完成 Skill，下一步才加入 Gate。**

最重要的是這一條：

```text
不要先想怎麼把整套 AI Automation 做完。
先讓一個 Skill 做對。
```

---

## 下一步

下一篇開始做 **Requirement Gate**。

我們會把今天的 Output 接到第一個 Gate：

```text
Requirement Skill
       ↓
Requirement Gate
       ↓
 ┌─────┴─────┐
 PASS       BLOCKED
  ↓            ↓
Planning     Human
```

到那時候，AI 才不只是「會做事」，而是開始具備：

> **做完之後，必須先通過檢查，才能進下一步。**

---

## 官方來源

- Claude Code Skills 官方文件：
  https://code.claude.com/docs/en/skills
- Angular Agent Skills：
  https://angular.dev/ai/agent-skills
- Angular CLI MCP：
  https://angular.dev/ai/mcp

Claude Code Skills 的目錄位置、`SKILL.md` 格式、Skill 的自動載入與手動 `/skill-name` 呼叫方式，以目前官方文件為準。citeturn1view0
