# AI 自動化教學 02｜Claude Code Hook 與第一個 Gate

## 今天要完成什麼？

昨天建立 Requirement Skill，今天先建立第一個可獨立驗證的 Requirement Gate，為後續 Hook、Auto Fix 與 YAML Workflow 打基礎。

核心分工：

- Skill：負責做事
- Gate：負責驗收
- Hook：負責在事件發生時觸發檢查

## 1. 建立最小 Requirement Gate

目錄：

```text
scripts/
└── gates/
    └── requirement-gate.sh
```

```bash
#!/usr/bin/env bash
set -e

FILE="docs/requirement.md"

if [ ! -f "$FILE" ]; then
  echo "BLOCKED: $FILE 不存在"
  exit 2
fi

if [ ! -s "$FILE" ]; then
  echo "FAIL: $FILE 是空的"
  exit 1
fi

echo "PASS: Requirement Gate"
exit 0
```

結果契約先固定：

```text
0 → PASS
1 → FAIL
2 → BLOCKED
```

Windows / PowerShell 專案可以改寫成 `.ps1`，重點是 exit code 要穩定。

## 2. 為什麼 Gate 要獨立？

不要一開始就把「是否通過」全部交給 AI 判斷。

程式可以客觀檢查的事情，例如檔案存在、內容是否為空，就先用 Deterministic Gate。

之後再加入 AI Semantic Gate：

```text
Deterministic Gate
      ↓
AI Semantic Gate
      ↓
PASS
```

## 3. Hook 的角色

專案級 Claude Code 設定可放在 `.claude/settings.json`。

但 Hook 的事件與設定欄位可能隨 Claude Code 版本更新，因此實際接線時應先查當下最新官方文件，不要直接複製舊範例。

今天先做 Gate，再接 Hook：

```text
Gate 可以獨立執行
        ↓
再接 Hook
```

這樣 Hook 出問題時，比較容易定位。

## 4. 驗證 Gate

Requirement 不存在：

```text
BLOCKED
exit code = 2
```

Requirement 是空檔案：

```text
FAIL
exit code = 1
```

Requirement 有內容：

```text
PASS
exit code = 0
```

## 5. 下一步

接下來把流程串成：

```text
Requirement Skill
      ↓
更新 requirement.md
      ↓
Hook
      ↓
Requirement Gate
      ↓
   PASS?
   ↙   ↘
 FAIL   PASS
  ↓       ↓
Auto Fix  Planning
```

這就是第一個自動化閉環：

**Skill → Gate → Auto Fix → Gate → PASS**

## 常見問題

### FAIL 與 BLOCKED 的差別？

`FAIL` 表示已經可以檢查，但沒有通過。

`BLOCKED` 表示缺少必要前置條件，無法繼續判斷。

例如：Requirement 存在但內容不符合規則是 FAIL；Requirement 根本不存在是 BLOCKED。

## 今天學會什麼？

先把 Gate 做成可以獨立驗證的工具，再用 Hook 自動觸發它。這個順序會讓後面的 Auto Fix 與 YAML Workflow 更容易組合。

## 下一步

下一篇正式把 Claude Code Hook 接到 Requirement Gate，再處理 Auto Fix。
