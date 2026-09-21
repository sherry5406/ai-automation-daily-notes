# AI 自動化教學 Daily Notes

以 Angular 前端開發流程為實際案例，逐步建立：

**Claude Code + Skills + Hooks + Gate Scripts + YAML Workflow → AI Development Automation**

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

## 核心迭代

目前先完成最小閉環：

```text
Skill → Gate → Auto Fix → Gate → PASS
```

再逐步加入：

```text
Planning → Figma → Implementation → Test → E2E → AI Acceptance → Human Gate → PR
```

## 原則

- 每篇只聚焦一個小主題，依前後順序累積。
- 優先提供可以直接在 Angular 專案操作的範例。
- Claude Code Hooks / Skills / CLI 等會隨版本更新的內容，以最新官方文件為準。
- Gate 負責驗收，Auto Fix 負責修正，不讓 Auto Fix 自己宣布 PASS。
- BLOCKED 不進自動修復，避免 AI 在缺少必要資訊時自行猜測。
- Auto Fix 必須有修改範圍與 Retry 上限。
- Skill 建議依責任拆分，並明確定義 Input / Output Contract。
- Deterministic Gate 優先處理可機器驗證的規則，AI Semantic Gate 再處理需要語意判斷的品質問題。
