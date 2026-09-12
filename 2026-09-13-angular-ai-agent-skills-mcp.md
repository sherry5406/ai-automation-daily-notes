# 2026-09-13｜Angular × AI：讓 AI 更懂你的 Angular 專案

## 今天學什麼？

如果你有用 Claude Code、Cursor 或其他 AI Coding 工具，可以把 Angular 官方提供的 **Agent Skills** 和 **Angular CLI MCP** 接起來。

白話講：

> 不要只叫 AI「幫我寫 Angular」，而是讓 AI 先知道 Angular 現在推薦怎麼寫，再讓它使用 Angular CLI 的工具做檢查。

Angular 官方目前提供 `angular-developer`、`angular-new-app` 等 Agent Skills；Angular CLI 也提供 MCP Server，可以讓 AI 做 workspace 分析、build、test 等工作。citeturn1search0turn1search1

## 1. Agent Skill 是什麼？

可以把 Skill 想成「給 AI 的 Angular 使用說明書」。

例如 `angular-developer` 會提供 Angular 架構、Signals、routing、forms、SSR、testing 等開發指引。citeturn1search0

這比單純跟 AI 說：

```text
幫我寫一個 Angular Component
```

更好，因為 AI 有一套針對 Angular 的規則可以參考。

## 2. Angular MCP 是什麼？

Skill 比較像「告訴 AI 怎麼做」。

MCP 比較像「給 AI 工具可以真的去做」。

Angular CLI MCP 可以提供像這些工具：

```text
list_projects
get_best_practices
run_target
search_documentation
```

例如 AI 可以分析 workspace，然後直接執行 build、test、lint 或 E2E target。citeturn1search1

## 3. 最簡單的設定

如果你的 AI 工具支援 Angular CLI MCP，可以使用：

```json
{
  "mcpServers": {
    "angular-cli": {
      "command": "npx",
      "args": ["-y", "@angular/cli", "mcp"]
    }
  }
}
```

Angular 官方文件目前提供 Cursor、VS Code、Gemini CLI 等環境的設定方式。citeturn1search1

## 4. Angular CLI 還可以產生 AI 設定

現在 Angular CLI 有 `ai-config`：

```bash
ng generate ai-config --tool=claude-code
```

它可以產生 AI 使用的設定，例如 `CLAUDE.md`、`AGENTS.md` 以及 MCP 設定，讓 AI 更容易按照 Angular 專案規則工作。citeturn1search5

## 5. 對你的 EIP 專案有什麼用？

你目前想做的是：

```text
Claude Code
  ↓
Skills
  ↓
Hooks
  ↓
Gate
  ↓
Test / E2E
```

Angular 官方 Skill + MCP 可以放在中間：

```text
需求
 ↓
Claude Code
 ↓
Angular Skill
 ↓
實作
 ↓
Angular CLI MCP
 ↓
Build / Test / E2E
 ↓
Gate
```

這樣 AI 不只是「寫程式」，還可以自己驗證 Angular 專案。

## 今天直接記住 3 件事

1. **Skill = 告訴 AI 怎麼寫 Angular。**
2. **MCP = 給 AI 工具去操作 Angular 專案。**
3. **Gate = 最後判斷這次修改能不能過關。**

### 小提醒

Angular 官方目前也在持續加強 AI 開發體驗；官方 roadmap 把 AI experience 列為目前三大目標之一。citeturn1search3

所以如果你正在做「Angular × AI 自動化」，這一塊值得納入你的工具箱。

**今天的實用結論：先讓 AI 懂 Angular，再讓 AI 有 Angular 工具，最後才讓 Gate 自動驗收。**
