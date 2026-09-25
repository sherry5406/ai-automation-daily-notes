# AI 自動化教學 12｜YAML Workflow 04：Parallel Execution

日期：2026-09-26

## 今天要完成什麼

昨天我們加入了 `human_gate`，今天再增加一個能力：**Parallel Execution（平行執行）**。

目標不是一次把 Workflow 做得很複雜，而是先解決一個 Angular 前端開發最常見的問題：

> 一個實作完成後，Lint、Typecheck、Build 彼此沒有依賴，為什麼要一個跑完才跑下一個？

今天把它變成：

```text
Implementation
      ↓
   Test Gate
   ↙      ↘
Lint   Typecheck
   ↘      ↙
   Test Summary
      ↓
   PASS / FAIL
```

---

## 一、核心概念：什麼是 Parallel Execution？

平常的 Workflow 是串行：

```text
Lint
 ↓
Typecheck
 ↓
Build
```

如果三個 Node 互相沒有依賴，其實可以平行：

```text
        ┌→ Lint ─────┐
        │             │
Implementation → Typecheck → Test Gate
        │             │
        └→ Build ────┘
```

這裡有一個非常重要的判斷：

> **只有沒有互相 Dependency 的工作，才適合 Parallel。**

例如：

```text
Lint       ─┐
Typecheck   ├→ Test Gate
Build      ─┘
```

可以平行。

但：

```text
Build
 ↓
E2E
```

如果 E2E 必須使用 Build 產物，就不能平行。

---

## 二、今天的 Workflow Contract

再次提醒：下面 YAML 是**本教學自行設計的 Automation Contract**，不是 Claude Code 官方 YAML 語法。

Claude Code 官方目前把 Skills、Hooks、Subagents、MCP 等視為不同的擴充能力；Hooks 在生命週期事件觸發，Skills 則是可重複使用的知識與工作流程。citeturn1search1

今天只增加一個欄位：

```yaml
parallel: true
```

完整最小範例：

```yaml
workflow:
  name: implementation-test

  nodes:
    - id: implementation
      type: skill
      run: implementation-skill
      output: implementation.json
      on_pass: test-lint

    - id: test-lint
      type: gate
      run: pnpm lint
      parallel_group: test-checks
      on_pass: test-summary
      on_fail: auto-fix

    - id: test-typecheck
      type: gate
      run: pnpm typecheck
      parallel_group: test-checks
      on_pass: test-summary
      on_fail: auto-fix

    - id: test-build
      type: gate
      run: pnpm build
      parallel_group: test-checks
      on_pass: test-summary
      on_fail: auto-fix

    - id: test-summary
      type: gate
      depends_on:
        - test-lint
        - test-typecheck
        - test-build
      run: test-summary
      on_pass: e2e
      on_fail: human_gate
```

這裡先故意不加入複雜的 retry、timeout、resource limit 等功能。

---

## 三、為什麼要有 `parallel_group`？

最簡單的方式不是讓 Workflow Engine 猜：

> 「看起來沒有 dependency，所以我幫你平行。」

而是由 Workflow 明確宣告：

```yaml
parallel_group: test-checks
```

例如：

```yaml
- id: test-lint
  parallel_group: test-checks

- id: test-typecheck
  parallel_group: test-checks

- id: test-build
  parallel_group: test-checks
```

這樣的好處是：

1. YAML 一眼看得懂。
2. Workflow Engine 不需要自行推測。
3. 未來可以限制哪些 Node 可以平行。
4. Debug 時容易知道哪些工作屬於同一批。

---

## 四、Parallel 的真正關鍵：Join Point

平行最重要的不是「同時跑」，而是：

> **什麼時候可以繼續？**

所以需要一個 Join Point：

```text
Lint ─────────┐
              │
Typecheck ────┼→ Test Summary
              │
Build ────────┘
```

我們用：

```yaml
depends_on:
  - test-lint
  - test-typecheck
  - test-build
```

代表：

> 三個 Gate 都完成後，才允許 `test-summary` 執行。

這就是 Parallel Workflow 裡非常重要的 **Fan-out → Fan-in**。

```text
Fan-out
   ↓
┌──────┬──────────┬──────┐
Lint Typecheck   Build
└──────┴──────────┴──────┘
   ↓
Fan-in
   ↓
Test Summary
```

---

## 五、PASS / FAIL 怎麼處理？

今天先沿用前面已經建立好的規則。

### 全部 PASS

```text
Lint PASS
Typecheck PASS
Build PASS
      ↓
Test Summary PASS
      ↓
E2E
```

### 任一失敗

```text
Lint PASS
Typecheck FAIL
Build PASS
      ↓
Test Summary FAIL
      ↓
Auto Fix / Human Gate
```

注意：

> **不是三個裡面兩個 PASS 就算 PASS。**

第一版規則採最簡單的：

```text
全部 PASS → PASS
任何 FAIL → FAIL
無法判定 / 缺少必要資訊 → BLOCKED
```

---

## 六、Angular 專案實際範例

假設你的 Angular 專案使用 pnpm，可以在 `package.json` 定義：

```json
{
  "scripts": {
    "lint": "ng lint",
    "typecheck": "tsc --noEmit",
    "build": "ng build"
  }
}
```

Workflow 就可以直接呼叫：

```yaml
- id: lint
  type: gate
  run: pnpm lint
  parallel_group: implementation-tests

- id: typecheck
  type: gate
  run: pnpm typecheck
  parallel_group: implementation-tests

- id: build
  type: gate
  run: pnpm build
  parallel_group: implementation-tests
```

實際執行時，可以由 Workflow Engine 啟動三個獨立 process。

概念上等同：

```bash
pnpm lint &
pnpm typecheck &
pnpm build &
wait
```

但真正的 Workflow Engine 不應只靠 shell `&` 解決，因為還需要記錄：

- 每個 Node 的開始時間
- 每個 Node 的結束時間
- exit code
- stdout / stderr
- PASS / FAIL / BLOCKED
- 哪一個 Node 失敗

---

## 七、建議的 Result Contract

每個 Gate 最後都輸出統一格式：

```json
{
  "node": "test-typecheck",
  "status": "FAIL",
  "exitCode": 1,
  "startedAt": "2026-09-26T07:00:00+08:00",
  "finishedAt": "2026-09-26T07:00:08+08:00",
  "summary": "TypeScript compilation failed"
}
```

這樣 `test-summary` 不需要重新執行 Typecheck。

它只需要讀取：

```text
Lint Result
Typecheck Result
Build Result
```

然後做 deterministic aggregation。

---

## 八、不要讓 AI 決定是否 PASS

這裡再次延續核心原則：

```text
Gate
 ↓
產生客觀結果
 ↓
Workflow Aggregator
 ↓
PASS / FAIL / BLOCKED
```

不要寫成：

```text
Claude 看完三個結果
 ↓
Claude 覺得應該 PASS
```

因為 `lint`、`typecheck`、`build` 都有明確 exit code。

這類規則應該優先使用 deterministic Gate。

AI Semantic Gate 留給真正需要語意判斷的事情，例如：

- Requirement 是否真的被實作
- UI 是否符合設計意圖
- E2E 是否涵蓋主要使用情境

---

## 九、如何驗證成功

先做最小測試，不要直接接整個 Angular 專案。

### 測試 1：全部 PASS

```text
Lint       → PASS
Typecheck  → PASS
Build      → PASS
```

預期：

```text
Test Summary → PASS
```

### 測試 2：其中一個 FAIL

```text
Lint       → PASS
Typecheck  → FAIL
Build      → PASS
```

預期：

```text
Test Summary → FAIL
```

### 測試 3：其中一個 BLOCKED

例如 Build 所需的環境變數不存在：

```text
Lint       → PASS
Typecheck  → PASS
Build      → BLOCKED
```

預期：

```text
Test Summary → BLOCKED
      ↓
Human Gate
```

### 測試 4：確認 Join Point

```text
Lint 完成
Typecheck 完成
Build 尚未完成
```

此時：

```text
Test Summary
```

**不能開始。**

---

## 十、常見問題

### Q1：Parallel 是不是一定比較快？

不一定。

如果三個工作會搶同一個 CPU、記憶體或磁碟 I/O，平行反而可能變慢。

所以第一版只處理「彼此獨立，而且值得平行」的 Node。

### Q2：`lint` 失敗了，還要跑 typecheck / build 嗎？

第一版建議照跑。

原因很簡單：一次拿到完整結果，Auto Fix 才知道目前到底有幾個問題。

如果未來專案很大，再增加：

```yaml
fail_fast: true
```

但今天先不要加入。

### Q3：Parallel 跟 Agent Teams 一樣嗎？

不一樣。

Parallel Workflow 是：

> **Workflow 層級的 Node 平行執行。**

Agent Teams 是 Claude Code 的另一種協作能力。

今天我們只處理 Workflow Engine 的執行順序，不把兩者混在一起。

### Q4：`parallel_group` 是 Claude Code 官方功能嗎？

不是。

它是這套教學自行定義的 Workflow Contract。

Claude Code 官方文件目前把 Hooks、Skills、Subagents、MCP 等能力分開定義；Workflow YAML 則是我們自己在專案中建立的 orchestration layer。citeturn1search1

---

## 十一、今天完成後，多了什麼能力？

昨天：

```text
Node
 ↓
Dependency
 ↓
on_pass / on_fail
 ↓
human_gate
```

今天：

```text
              ┌→ Lint ─────┐
Implementation ├→ Typecheck ├→ Test Summary
              └→ Build ────┘
```

也就是 Workflow 開始具備：

```text
Sequential
    +
Branching
    +
Human Gate
    +
Parallel Execution
```

而且我們仍然維持核心閉環：

```text
Skill
 ↓
Gate
 ↓
FAIL
 ↓
Auto Fix
 ↓
Re-Gate
 ↓
PASS
```

---

## 十二、今天學會什麼？

今天只記住 4 件事：

1. 沒有 Dependency 的 Node 才適合 Parallel。
2. Parallel 一定要有 Join Point。
3. 多個 Gate 結果應該由 deterministic rule 聚合。
4. Auto Fix 完成後仍然必須 Re-Gate。

最重要的是：

> **Workflow 負責決定「怎麼走」，Gate 負責決定「是否合格」。**

---

## 下一步

下一篇進入：

**YAML Workflow 05｜把 Skill + Gate + Auto Fix 組成第一個完整可執行 Workflow**

我們會把目前已完成的：

```text
Node
Dependency
on_pass
on_fail
human_gate
Parallel
```

真正組合成：

```text
Requirement
   ↓
Requirement Gate
   ↓
PASS?
 ↙     ↘
FAIL   PASS
 ↓       ↓
Auto Fix Planning
 ↓       ↓
Re-Gate Planning Gate
           ↓
        Parallel
      Lint / Typecheck / Build
           ↓
        Test Gate
```

到那時候，才開始接近真正的 **AI Development Automation Workflow**。
