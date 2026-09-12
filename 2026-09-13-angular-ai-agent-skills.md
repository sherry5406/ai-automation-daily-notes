# Angular × AI 實用教學 02｜Angular 官方 Agent Skills 怎麼裝？

> 給前端工程師的簡單版：Angular + Claude Code

## 先講結論

如果你使用 Claude Code 開發 Angular，Angular 官方提供的 Agent Skills 可以讓 AI 更了解 Angular 官方推薦的開發方式。

建議採用「專案內設定」，不要把 Angular 專案規則全部做成全域設定。

---

## 1. 安裝 Angular 官方 Agent Skills

先進入 Angular 專案根目錄：

```bash
cd 你的-angular-project
```

然後執行：

```bash
npx skills add https://github.com/angular/skills -a claude-code
```

這樣 Skill 會以 Claude Code 的方式加入目前開發環境。

---

## 2. 為 Claude Code 建立 Angular 專案設定

在同一個 Angular 專案執行：

```bash
ng generate ai-config --tool=claude-code
```

這一步可以幫專案建立 AI 使用的設定，例如 `CLAUDE.md`。

---

## 3. Angular 版本要不要注意？

要。

例如公司可能同時有：

```text
專案 A → Angular 20
專案 B → Angular 21
```

所以不要只告訴 AI「這是 Angular 專案」，最好讓專案自己記錄版本。

---

## 4. 建議加入 Angular Project Rules

可以在 `CLAUDE.md` 放這些基本規則：

```md
# Angular Project Rules

- Angular version: 20
- TypeScript version: xxx
- 使用 Signals
- 優先使用 Angular 官方推薦寫法
- 不要自行升級 Angular
```

### 白話解釋

- **Angular version**：告訴 AI 現在是哪一版 Angular。
- **TypeScript version**：避免 AI 使用不符合目前 TS 版本的語法或功能。
- **使用 Signals**：告訴 AI 專案的狀態管理方向。
- **官方推薦寫法**：優先採用 Angular 官方目前建議的 API 與模式。
- **不要自行升級 Angular**：AI 不可以看到新版本就直接幫你升級。

> `TypeScript version: xxx` 只是範例。實際版本請以專案 `package.json` 為準。

---

## 5. Skill、MCP、Project Rules 差在哪？

記三句話就好：

```text
Skill       → 教 AI 怎麼做
MCP         → 給 AI 工具可以操作
Project Rule → 告訴 AI 這個專案的規矩
```

所以你的 Angular 專案可以變成：

```text
Claude Code
   │
   ├── Angular Official Skills
   │
   ├── Angular MCP
   │
   ├── CLAUDE.md
   │
   └── EIP Rules / Skills
          ↓
      Angular 20 專案
```

---

## 6. 為什麼適合 EIP？

因為你可以把「Angular 官方規則」和「公司 EIP 規則」分開。

```text
Angular Official Skills
        ↓
Angular 官方做法
        +
EIP Rules
        ↓
公司自己的共用元件 / SCSS / 架構規範
```

這比把所有規則混在一起更容易維護。

---

## 今天學會什麼？

只要記住這個流程：

```text
Angular 專案
   ↓
npx skills add
   ↓
Angular Official Skills
   ↓
ng generate ai-config
   ↓
CLAUDE.md
   ↓
補上 Angular 20 / EIP 專案規則
```

下一步再接 **Claude Code Hooks**，讓「修改程式 → 自動檢查」開始跑起來。
