# AI 自動化教學 03｜把 Claude Code Hook 接到 Requirement Gate

## 今天要完成什麼？

昨天我們先把 Requirement Gate 做成可以獨立執行的工具，今天把它接進 Claude Code Hook。

目標不是一次完成所有自動化，而是先做到：

```text
Claude Code 事件
      ↓
    Hook
      ↓
Requirement Gate
      ↓
 PASS / FAIL / BLOCKED
```

完成後，AI 不需要靠「自己記得跑 Gate」，而是由 Hook 在指定事件發生時自動觸發檢查。

> 注意：Claude Code 的 Hook 設定與事件名稱可能隨版本更新。實際使用前，請以當下 Claude Code 官方文件為準；本文重點放在架構與接法。

## 1. 先確認昨天的 Gate

昨天建立的是：

```text
scripts/
└── gates/
    └── requirement-gate.sh
```

它有固定的 exit code：

```text
0 → PASS
1 → FAIL
2 → BLOCKED
```

這個「穩定的輸出契約」很重要，因為 Hook 不應該自己理解 Requirement，而只需要知道 Gate 的結果。

## 2. Hook 負責什麼？

可以把三者想成：

```text
Skill
→ 「我要完成 Requirement」

Gate
→ 「Requirement 合不合格？」

Hook
→ 「什麼時候自動跑 Gate？」
```

因此不要把 Gate 的判斷邏輯全部寫進 Hook。

推薦：

```text
Hook
  ↓
呼叫 Gate Script
  ↓
讀取 exit code
  ↓
決定是否繼續
```

## 3. 專案設定放在哪？

專案級 Claude Code 設定放在：

```text
.claude/
└── settings.json
```

建議把與目前 Angular 專案有關的 Hook 放在專案設定，而不是把 EIP 專案的流程規則全部塞進全域設定。

例如：

```text
my-angular-project/
├── .claude/
│   └── settings.json
├── scripts/
│   └── gates/
│       └── requirement-gate.sh
├── docs/
│   └── requirement.md
├── src/
└── package.json
```

## 4. Hook 的最小設計

實際 Hook JSON 請以目前 Claude Code 官方文件公布的事件與欄位為準。

設計概念可以先抽象成：

```json
{
  "hooks": {
    "<指定事件>": [
      {
        "command": "./scripts/gates/requirement-gate.sh"
      }
    ]
  }
}
```

這裡故意使用 `<指定事件>`，不要直接複製網路上的舊設定，因為 Hook API 可能改變。

真正要做的事情只有一個：

```text
事件發生
   ↓
執行 requirement-gate.sh
```

## 5. 為什麼今天不直接做 Auto Fix？

因為我們要先確認：

```text
Hook
 ↓
Gate
```

本身可靠。

如果一開始就做：

```text
Hook
 ↓
Gate
 ↓
AI Auto Fix
 ↓
Gate
 ↓
Hook
 ↓
...
```

一旦失敗，很難知道問題到底出在 Hook、Gate 還是 Auto Fix。

所以先建立最小閉環：

```text
Hook → Gate
```

下一階段再加入：

```text
FAIL
 ↓
Auto Fix
 ↓
Gate
 ↓
PASS
```

## 6. 驗證方式

準備三種情境。

### 情境 A：Requirement 不存在

```text
BLOCKED
exit code = 2
```

代表前置條件不足，不應直接進入下一階段。

### 情境 B：Requirement 存在但不符合規則

```text
FAIL
exit code = 1
```

這時可以交給後面的 Auto Fix。

### 情境 C：Requirement 通過

```text
PASS
exit code = 0
```

才允許進入 Planning。

## 7. 今天完成後，架構多了什麼能力？

昨天：

```text
Requirement Skill
      ↓
Requirement Gate
```

今天：

```text
Requirement Skill
      ↓
    Hook
      ↓
Requirement Gate
      ↓
 PASS / FAIL / BLOCKED
```

最大的改變是：

**Gate 不再依賴 AI 自己記得執行。**

Hook 開始成為「自動化流程的觸發器」。

## 常見問題

### Hook 可以取代 Gate 嗎？

不建議。

Hook 是觸發機制，Gate 是驗收機制。兩個責任分開，之後才能把同一個 Gate 用在：

```text
Hook
CI
Local Script
YAML Workflow
```

### Gate 為什麼要有 exit code？

因為不同自動化工具都可以用 exit code 判斷結果，不需要理解人類語言。

```text
0 = PASS
1 = FAIL
2 = BLOCKED
```

這就是之後串接 YAML Workflow 的基礎。

## 今天學會什麼？

今天只完成一件重要的事：

**把「何時檢查」與「檢查什麼」分開。**

```text
Hook = 何時跑
Gate = 怎麼驗收
Skill = 怎麼完成工作
```

## 下一步

下一篇進入真正的核心閉環：

```text
Skill
 ↓
Hook
 ↓
Gate
 ↓
FAIL
 ↓
Auto Fix
 ↓
Gate
 ↓
PASS
```

我們會開始設計第一個可重複執行的 **Auto Fix**，並處理「什麼情況 AI 可以自動修、什麼情況必須 BLOCKED」的界線。
