# AI 自動化教學 14｜YAML Workflow 06：Claude Code Hook → Gate Script

日期：2026-09-28

## 今天要完成什麼

上一篇已經有完整 Workflow，但目前還有一個問題：

> Workflow 知道要跑 Gate，Claude Code 本身卻可能在修改程式後忘記執行 Gate。

今天把兩邊接起來：

```text
Claude Code
   ↓
Tool / Edit
   ↓
PostToolUse Hook
   ↓
Gate Script
   ↓
PASS / FAIL / BLOCKED
   ↓
Workflow
```

今天先做最小版本：**修改 Angular TypeScript 檔案後，自動觸發 typecheck Gate。**

---

## 一、先釐清 Skill、Hook、Gate、Workflow

```text
Skill     → 告訴 Claude「怎麼做」
Hook      → 保證「某個事件發生時要執行」
Gate      → 判定「結果是否合格」
Workflow  → 決定「下一步往哪裡走」
```

Claude Code 官方目前把 Hooks 定義為生命週期事件觸發的自動化；Hooks 可以執行 command、HTTP、prompt 或 subagent。官方也特別指出：如果某件事每次都必須發生，應優先用 Hook，而不是只靠 Skill 或提示詞。citeturn1search1

專案級設定仍放在 `.claude/settings.json`，而 Skill 放在 `.claude/skills/<name>/SKILL.md`。citeturn1search0

---

## 二、今天只做一個 Hook

不要一次監控所有檔案。

我們先針對 Angular 的 `.ts` 檔案：

```text
Edit *.ts
   ↓
PostToolUse
   ↓
pnpm typecheck
```

這個設計有一個很重要的目的：

> **讓 Gate 從「Claude 記得要跑」變成「Claude 修改後系統自動跑」。**

---

## 三、Angular 專案目錄

```text
.claude/
├── settings.json
└── skills/
    └── implementation/
        └── SKILL.md

tools/
└── gates/
    └── typecheck-gate.mjs

src/
└── app/
    └── app.component.ts
```

---

## 四、先建立 Gate Script

`tools/gates/typecheck-gate.mjs`

```js
import { spawnSync } from 'node:child_process';

const result = spawnSync('pnpm', ['typecheck'], {
  stdio: 'inherit',
  shell: process.platform === 'win32'
});

if (result.error) {
  console.error(JSON.stringify({
    status: 'BLOCKED',
    summary: result.error.message
  }));
  process.exit(2);
}

if (result.status === 0) {
  console.log(JSON.stringify({
    status: 'PASS',
    gate: 'typecheck'
  }));
  process.exit(0);
}

console.log(JSON.stringify({
  status: 'FAIL',
  gate: 'typecheck'
}));
process.exit(1);
```

`package.json`：

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "gate:typecheck": "node tools/gates/typecheck-gate.mjs"
  }
}
```

這裡仍然遵守之前的 Contract：

```text
0 → PASS
1 → FAIL
2 → BLOCKED
```

---

## 五、加入 Claude Code Hook

`.claude/settings.json` 可以設定 Hooks。今天只示範概念上最小的 `PostToolUse`：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node tools/gates/typecheck-gate.mjs"
          }
        ]
      }
    ]
  }
}
```

官方文件目前將 `PostToolUse` 定位為工具成功執行後的生命週期事件；Hook 可以用 matcher 限定要在哪些工具事件上觸發。citeturn1search1

**注意：**實際部署前應依你目前安裝的 Claude Code 版本，使用官方 Hooks reference 驗證 matcher 與 handler 欄位；不要把本教學自訂 Workflow YAML 欄位和 Claude Code `settings.json` 語法混在一起。官方文件是最終依據。citeturn1search0

---

## 六、Hook 跑完之後，Workflow 怎麼接？

Hook 不應該自己負責整個 Workflow。

它只負責：

```text
事件發生
 ↓
執行 Gate
 ↓
產生 Result
```

Workflow 再消費 Result：

```text
PASS
 ↓
on_pass
 ↓
下一個 Node
```

```text
FAIL
 ↓
on_fail
 ↓
Auto Fix
 ↓
Re-Gate
```

```text
BLOCKED
 ↓
on_blocked
 ↓
Human Gate
```

這樣可以避免 Hook 變成另一個 Workflow Engine。

---

## 七、最重要的閉環終於接起來

現在我們原本的：

```text
Skill
 ↓
Gate
 ↓
Auto Fix
 ↓
Re-Gate
 ↓
PASS
```

前面再加上 Hook：

```text
Skill / Agent
      ↓
   修改程式
      ↓
      Hook
      ↓
     Gate
      ↓
 ┌────┼─────┐
PASS FAIL BLOCKED
       ↓      ↓
    Auto Fix Human Gate
       ↓
    Re-Gate
       ↓
      PASS
```

這才開始接近真正的 AI Development Automation。

---

## 八、如何驗證成功

### Case 1：修改正確 TypeScript

Claude 修改：

```text
src/app/app.component.ts
```

Hook 自動觸發：

```text
PostToolUse
 ↓
typecheck-gate
 ↓
PASS
```

### Case 2：故意製造 TypeScript 錯誤

例如：

```ts
const count: number = 'hello';
```

結果：

```text
Edit
 ↓
Hook
 ↓
Typecheck
 ↓
FAIL
```

此時不能直接往下一個 Workflow Node。

應該：

```text
FAIL
 ↓
Auto Fix
 ↓
Re-Gate
```

### Case 3：環境本身無法執行

例如：

- pnpm 不存在
- Node 環境錯誤
- 必要設定缺失

結果：

```text
BLOCKED
 ↓
Human Gate
```

---

## 九、常見問題

### Q1：為什麼不用 Skill 提醒 Claude「改完一定跑 typecheck」？

因為提醒不是保證。

Skill 是 Claude 可以理解與遵循的工作流程；Hook 是事件發生時自動觸發的機制。官方也明確建議：需要每次都發生的 guardrail，應使用 Hook。citeturn1search1

### Q2：Hook 可以直接 Auto Fix 嗎？

第一版不建議。

先讓 Hook：

```text
修改 → Gate → Result
```

再由 Workflow 決定：

```text
FAIL → Auto Fix → Re-Gate
```

責任比較清楚，也比較容易除錯。

### Q3：Hook 和 Workflow 是不是同一件事？

不是。

```text
Hook = Event Trigger
Workflow = Orchestration
```

Hook 告訴系統「現在該檢查了」。

Workflow 決定「檢查結果出來後下一步怎麼走」。

### Q4：所有 Edit 都跑完整測試會不會太慢？

會。

所以實務上不要一開始就：

```text
每次 Edit
 ↓
lint + typecheck + build + E2E
```

可以逐步演進：

```text
Edit .ts
 ↓
typecheck
```

```text
完成一個 Feature
 ↓
lint + typecheck + build
```

```text
準備 PR
 ↓
E2E + AI Acceptance
```

---

## 十、今天完成後，多了什麼能力？

昨天：

```text
Workflow
 ↓
Gate
```

今天：

```text
Claude Code
 ↓
Tool Event
 ↓
Hook
 ↓
Gate
 ↓
Workflow
```

最大的改變是：

> **Gate 不再只存在於 Workflow 裡，而是可以被 Claude Code 的生命週期事件自動觸發。**

也就是從「流程有規定」進一步變成「系統會自動執行」。

---

## 十一、今天學會什麼

只記住 5 件事：

1. Skill 負責教 Claude 怎麼做。
2. Hook 負責在指定生命週期事件自動觸發。
3. Gate 負責判定 PASS / FAIL / BLOCKED。
4. Workflow 負責處理 Gate 結果與下一步。
5. Auto Fix 完成後仍然必須 Re-Gate。

---

## 下一步

下一篇進入：

**YAML Workflow 07｜Hook 結果如何回傳給 Workflow Engine**

會把今天的：

```text
Hook → Gate → Result
```

再進一步標準化成：

```json
{
  "node": "typecheck",
  "status": "FAIL",
  "exitCode": 1,
  "artifacts": [],
  "summary": "TypeScript compilation failed"
}
```

然後讓 Workflow Engine 可以真正依這個 Result 決定：

```text
PASS → on_pass
FAIL → on_fail
BLOCKED → on_blocked
```

這一步完成後，我們的 Hook、Gate、Workflow 才真正開始使用同一份 Machine-readable Contract。
