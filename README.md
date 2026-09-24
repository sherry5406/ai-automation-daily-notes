# AI 自動化教學 Daily Notes

以 Angular 前端開發流程為實際案例，逐步建立：

**Claude Code + Skills + Hooks + Gate Scripts + YAML Workflow → AI Development Automation**

另外加入一條實務學習支線：

**Angular × SignalR → 純前端串接 → 連線生命週期 → 即時事件 → 單一裝置單一頁籤 → Session Activity / 閒置登出**

## 教學路線

```text
/grill-me-doc
    ↓
Requirement Skill
    ↓
Hook
    ↓
Requirement Gate
    ↓
PASS?
 ↙      ↘
FAIL    PASS
 ↓        ↓
Auto Fix  Planning
           ↓
      Planning Gate
           ↓
          Figma
           ↓
     Implementation
           ↓
        Test Gate
           ↓
           E2E
           ↓
      AI Acceptance
           ↓
       Human Gate
           ↓
           PR
```

## 每日文章

| 日期 | 主題 |
|---|---|
| 2026-09-11 | [AI 自動化教學 01](./2026-09-11-claude-code-ai-automation-01.md) |
| 2026-09-12 | [Angular linkedSignal 實用技巧](./2026-09-12-angular-linkedSignal-實用技巧.md) |
| 2026-09-13 | [Angular AI Agent Skills + MCP](./2026-09-13-angular-ai-agent-skills-mcp.md) |
| 2026-09-13 | [Angular AI Agent Skills](./2026-09-13-angular-ai-agent-skills.md) |
| 2026-09-15 | [Claude Code Requirement Skill](./2026-09-15-claude-code-requirement-skill.md) |
| 2026-09-16 | [Claude Code Hooks + First Gate](./2026-09-16-claude-code-hooks-first-gate.md) |
| 2026-09-17 | [Claude Code Hook + Requirement Gate](./2026-09-17-claude-code-hook-requirement-gate.md) |
| 2026-09-18 | [Requirement Gate FAIL → Auto Fix](./2026-09-18-requirement-gate-auto-fix.md) |
| 2026-09-19 | [Planning Skill Input / Output](./2026-09-19-planning-skill-input-output.md) |
| 2026-09-20 | [Planning Gate Script](./2026-09-20-planning-gate-script.md) |
| 2026-09-21 | [Planning Gate → Auto Fix → Re-Gate](./2026-09-21-planning-gate-auto-fix.md) |
| 2026-09-22 | [Planning → Figma → Implementation Contract](./2026-09-22-planning-figma-implementation-contract.md) |
| 2026-09-23 | [YAML Workflow 01：Node + Dependency](./2026-09-23-yaml-workflow-01-node-dependency.md) |
| 2026-09-24 | [YAML Workflow 02：on_pass / on_fail](./2026-09-24-yaml-workflow-02-on-pass-on-fail.md) |
| 2026-09-25 | [YAML Workflow 03：human_gate](./2026-09-25-yaml-workflow-03-human-gate.md) |
| 2026-09-24 | [Angular × SignalR｜原理](./2026-09-24-signalr-原理.md) |
| 2026-09-25 | [Angular × SignalR｜第一步建立 HubConnection](./2026-09-25-signalr-hubconnection.md) |

## Angular × SignalR 學習路線

```text
SignalR 原理
    ↓
Angular HubConnection
    ↓
連線生命週期
    ↓
接收後端事件
    ↓
斷線與自動重連
    ↓
單一裝置／單一頁籤
    ↓
有效 Activity
    ↓
Session LastActivityAt
    ↓
閒置登出／SessionExpired
```

## 核心迭代

目前先完成最小閉環：

```text
Skill → Gate → Auto Fix → Gate → PASS
```

再逐步加入：

```text
Planning → Figma → Implementation → Test → E2E → AI Acceptance → Human Gate → PR
```

並將已驗證的流程逐步抽象成：

```text
YAML Workflow
  ↓
Node
  ↓
Dependency
  ↓
on_pass / on_fail
  ↓
on_blocked / human_gate
  ↓
Parallel
```

## 原則

- 每篇只聚焦一個小主題，依前後順序累積。
- Angular × SignalR 支線採純前端角度教學，假設後端 Hub 已經完成。
- 優先提供可以直接在 Angular 專案操作的範例。
- Claude Code Hooks / Skills / CLI 等會隨版本更新的內容，以最新官方文件為準。
- Gate 負責驗收，Auto Fix 負責修正，不讓 Auto Fix 自己宣布 PASS。
- BLOCKED 不進自動修復，避免 AI 在缺少必要資訊時自行猜測。
- BLOCKED 可交給 Human Gate，由人工做明確的 APPROVED / REJECTED 決策。
- Human Gate 負責決策，不在第一版直接修改 Requirement 或程式碼。
- Auto Fix 必須有修改範圍與 Retry 上限。
- Skill 建議依責任拆分，並明確定義 Input / Output Contract。
- Deterministic Gate 優先處理可機器驗證的規則，AI Semantic Gate 再處理需要語意判斷的品質問題。
- YAML Workflow 是本教學建立的 Automation Contract，不宣稱是 Claude Code 官方 YAML 語法。
- `on_pass / on_fail / on_blocked` 負責描述結果分支；Auto Fix 完成後仍必須 Re-Gate，不能自行宣布原 Gate PASS。
- SignalR 的 Session 最終有效性以後端規則為準；前端負責連線、事件處理、有效 Activity 回報與 UI 狀態。
