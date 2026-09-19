# AI 自動化教學 06｜把 Planning Gate 做成真正可執行的 Gate Script

## 今天要完成什麼？

上一篇已經把 Planning Skill 的 Input / Output Contract 定義好了：

```text
Requirement PASS
      ↓
Planning Skill
      ↓
Planning Output
```

今天不要急著進 Figma 或 Implementation，而是把 Planning Output 真的交給一個可以執行的 Gate Script。

今天完成後會得到第二個可驗證的閉環：

```text
Planning Skill
      ↓
Planning Output
      ↓
Planning Gate
      ↓
PASS / FAIL / BLOCKED
```

下一階段才接：

```text
FAIL → Auto Fix → Re-Gate → PASS
```

---

## 1. 為什麼 Gate 要獨立成 Script？

如果只靠 Claude 自己說：

```text
「Planning 看起來沒問題，可以進 Implementation。」
```

這其實不是 Gate。

因為真正的 Gate 應該是：

```text
Input
  ↓
規則檢查
  ↓
機器可讀結果
  ↓
PASS / FAIL / BLOCKED
```

這樣 Hook、Auto Fix、YAML Workflow、CI 才能接得起來。

因此今天開始建立：

```text
scripts/
└── gates/
    └── planning-gate.mjs
```

---

## 2. 先定義 Planning Gate Contract

今天沿用上一篇的 Planning Output：

```json
{
  "goal": "新增課程列表頁",
  "affectedAreas": ["apps/web/course"],
  "implementationSteps": ["建立 Component", "串接 API"],
  "filesToCreate": ["course-list.component.ts"],
  "filesToModify": ["app.routes.ts"],
  "dependencies": [],
  "testPlan": ["component test", "E2E"],
  "openQuestions": []
}
```

Gate 第一版只驗證結構與基本一致性，不負責判斷「這是不是最佳架構」。

### PASS 條件

- `goal` 必須存在且非空字串
- `affectedAreas` 必須是陣列
- `implementationSteps` 至少一項
- `filesToCreate` 與 `filesToModify` 至少一項
- `testPlan` 至少一項
- `openQuestions` 必須存在且為陣列
- 不允許同一檔案同時出現在 create 與 modify

### BLOCKED 條件

如果輸入本身不是合法 JSON、檔案不存在，這不是「Planning 品質不好」，而是 Gate 沒有足夠資料判斷。

因此：

```text
缺資料 / 無法解析
      ↓
BLOCKED
```

而不是：

```text
缺資料
 ↓
FAIL
 ↓
Auto Fix
```

這延續前面建立的原則：**BLOCKED 不自動猜。**

---

## 3. Gate Script 第一版

建立：

```text
scripts/gates/planning-gate.mjs
```

內容：

```js
import { readFile } from 'node:fs/promises';

const inputPath = process.argv[2] ?? '.ai/planning-output.json';

const result = {
  gate: 'planning',
  status: 'PASS',
  version: 1,
  errors: [],
  warnings: [],
};

const fail = (message) => {
  result.status = 'FAIL';
  result.errors.push(message);
};

try {
  const raw = await readFile(inputPath, 'utf8');
  const planning = JSON.parse(raw);

  if (typeof planning.goal !== 'string' || !planning.goal.trim()) {
    fail('goal 必須是非空字串');
  }

  if (!Array.isArray(planning.affectedAreas)) {
    fail('affectedAreas 必須是陣列');
  }

  if (
    !Array.isArray(planning.implementationSteps) ||
    planning.implementationSteps.length === 0
  ) {
    fail('implementationSteps 至少需要一個步驟');
  }

  if (
    (!Array.isArray(planning.filesToCreate) || planning.filesToCreate.length === 0) &&
    (!Array.isArray(planning.filesToModify) || planning.filesToModify.length === 0)
  ) {
    fail('filesToCreate 與 filesToModify 至少需要一項');
  }

  if (!Array.isArray(planning.testPlan) || planning.testPlan.length === 0) {
    fail('testPlan 至少需要一項');
  }

  if (!Array.isArray(planning.openQuestions)) {
    fail('openQuestions 必須是陣列');
  }

  const creates = new Set(planning.filesToCreate ?? []);
  const modifies = new Set(planning.filesToModify ?? []);

  for (const file of creates) {
    if (modifies.has(file)) {
      fail(`檔案不能同時出現在 filesToCreate 與 filesToModify：${file}`);
    }
  }
} catch (error) {
  result.status = 'BLOCKED';
  result.errors.push(`無法讀取或解析 Planning Output：${error.message}`);
}

console.log(JSON.stringify(result, null, 2));
process.exitCode = result.status === 'PASS' ? 0 : 1;
```

這裡有一個重要設計：

**stdout 永遠輸出 JSON。**

這樣未來不論是 Hook、另一個 Agent、YAML Runner 或 CI，都可以直接讀結果。

---

## 4. package.json 加入指令

```json
{
  "scripts": {
    "gate:planning": "node scripts/gates/planning-gate.mjs"
  }
}
```

之後可以：

```bash
pnpm gate:planning .ai/planning-output.json
```

如果你的專案使用 npm，也可以：

```bash
npm run gate:planning -- .ai/planning-output.json
```

---

## 5. 建立 PASS 測試資料

建立：

```text
.ai/planning-output.json
```

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
    "加入 loading、empty、error state"
  ],
  "filesToCreate": [
    "apps/web/course/course-list.component.ts",
    "apps/web/course/course.service.ts"
  ],
  "filesToModify": [
    "apps/web/app.routes.ts"
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

執行：

```bash
pnpm gate:planning .ai/planning-output.json
```

應該得到：

```json
{
  "gate": "planning",
  "status": "PASS",
  "version": 1,
  "errors": [],
  "warnings": []
}
```

而且 process exit code 是 `0`。

---

## 6. 再測一個 FAIL

把 `testPlan` 改成空陣列：

```json
"testPlan": []
```

再次執行：

```bash
pnpm gate:planning .ai/planning-output.json
```

應該得到類似：

```json
{
  "gate": "planning",
  "status": "FAIL",
  "version": 1,
  "errors": [
    "testPlan 至少需要一項"
  ],
  "warnings": []
}
```

這次 exit code 應該是 `1`。

因此可以讓外部流程判斷：

```text
exit 0 → PASS
exit 1 → FAIL
```

---

## 7. 再測一個 BLOCKED

例如指定不存在的檔案：

```bash
pnpm gate:planning .ai/not-found.json
```

會得到：

```json
{
  "gate": "planning",
  "status": "BLOCKED",
  "version": 1,
  "errors": [
    "無法讀取或解析 Planning Output：..."
  ],
  "warnings": []
}
```

這時候不要 Auto Fix。

因為 AI 根本沒有足夠資訊知道真正的 Planning Output 是什麼。

正確流程：

```text
BLOCKED
  ↓
補齊 Input
  ↓
重新執行 Gate
```

---

## 8. 為什麼今天還不接 Hook？

因為現在要先把三個東西分開驗證：

```text
Planning Skill
= 產生 Planning

Planning Gate
= 驗證 Planning

Hook
= 決定什麼時候自動觸發 Gate
```

如果一開始就全部綁在一起：

```text
Skill + Hook + Gate + Auto Fix
```

出了問題會很難知道到底是哪一層有問題。

現在先做到：

```text
Planning Skill
      ↓
Planning Output
      ↓
Planning Gate
      ↓
PASS / FAIL / BLOCKED
```

下一篇再接：

```text
Planning Gate FAIL
      ↓
Auto Fix
      ↓
Planning Gate
```

---

## 9. 這個 Gate 未來怎麼接 YAML Workflow？

現在的 JSON 已經可以成為 Workflow Node 的標準結果：

```yaml
nodes:
  planning_gate:
    command: pnpm gate:planning .ai/planning-output.json
    on_pass: implementation
    on_fail: planning_auto_fix
    on_blocked: human_gate
```

注意：今天這段 YAML **先不要真的建立 Runner**。

我們只是先確定 Gate 的輸出格式可以支援未來 Workflow。

真正的 YAML Workflow 等前面的 Skill / Gate / Auto Fix 閉環穩定後再做。

---

## 10. Angular 專案中的完整位置

目前可以整理成：

```text
Angular Nx Project
│
├── .claude/
│   └── skills/
│       ├── requirement/
│       │   └── SKILL.md
│       └── planning/
│           └── SKILL.md
│
├── .ai/
│   └── planning-output.json
│
├── scripts/
│   └── gates/
│       └── planning-gate.mjs
│
├── package.json
└── CLAUDE.md
```

這裡開始可以清楚看到：

```text
.claude/  → AI 行為與 Skills
.ai/      → AI 流程產物
scripts/  → 可執行驗收規則
```

這個分層會讓後面接 Hook、YAML Workflow、CI 比較乾淨。

---

## 如何驗證今天成功？

完成後至少跑三個 Case：

### Case A

```text
完整 Planning
↓
PASS
```

### Case B

```text
缺少 testPlan
↓
FAIL
```

### Case C

```text
Planning Output 不存在
↓
BLOCKED
```

三種結果都能由機器讀取，就代表今天的 Gate Contract 成立。

---

## 常見問題

### Q1：為什麼不直接用 AI 判斷 Planning 好不好？

可以，但那會是後面的 **AI Semantic Gate**。

今天先做 deterministic gate：

```text
欄位有沒有
格式對不對
基本規則有沒有違反
```

這些規則應該優先用程式檢查，穩定又容易重現。

### Q2：AI Semantic Gate 什麼時候做？

後面再做。

完整流程會變成：

```text
Deterministic Gate
       ↓
PASS
       ↓
AI Semantic Gate
       ↓
PASS / FAIL / BLOCKED
```

### Q3：Gate FAIL 就一定 Auto Fix 嗎？

不一定。

只有「可安全自動修正」的問題才進 Auto Fix。

例如：

```text
缺 testPlan
→ 可以要求 AI 補齊
```

但：

```text
需求本身不明確
→ BLOCKED
→ 不應該猜
```

---

## 今天學會什麼？

今天真正建立的是：

```text
Skill Output
     ↓
Deterministic Gate
     ↓
Machine-readable Result
     ↓
PASS / FAIL / BLOCKED
```

這一步完成後，Planning 不再只是「AI 寫了一份計畫」，而是開始成為一個**可以被自動化流程驗收的 Node**。

---

## 下一步

下一篇進入核心閉環：

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

會開始處理一個很重要的問題：

> **Auto Fix 到底允許 AI 改什麼？又怎麼避免 AI 為了讓 Gate PASS 而亂改 Requirement？**

等這個閉環穩定後，再往 Figma → Implementation 前進。

---

## 官方參考

- Claude Code Hooks：https://code.claude.com/docs/en/hooks
- Claude Code Skills：https://code.claude.com/docs/en/skills

Claude Code 官方目前的 Hook 文件說明 Hooks 可以在生命週期特定事件自動執行，並可使用 command、HTTP、MCP tool、prompt 或 agent handler；Hooks 的 scope 也可以放在專案 `.claude/settings.json`，因此今天的 Gate Script 可以作為後續 Hook 的實際執行目標。citeturn1view0

本文中的 `planning-gate.mjs`、PASS / FAIL / BLOCKED Contract，以及未來 YAML Node 設計，屬於本系列建立的 AI Development Automation 架構，不是 Claude Code 官方內建的 Planning Gate。
