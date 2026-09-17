# AI 自動化教學 04｜Requirement Gate FAIL 後，怎麼安全做到 Auto Fix

## 今天要完成什麼？

昨天已經把：

```text
Requirement Skill
      ↓
    Hook
      ↓
Requirement Gate
      ↓
 PASS / FAIL / BLOCKED
```

接起來。

今天往前一步：**當 Gate 是 FAIL 時，讓 Claude Code 有規則地修正，再重新跑 Gate。**

目標不是讓 AI「看到錯誤就亂改」，而是建立第一個可控閉環：

```text
Skill
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

這會成為後面 YAML Workflow 的第一個真正 Node chain。

> Claude Code 的 Hooks、Skills 與設定方式會隨版本演進。本文涉及 Claude Code 行為時，以目前官方文件為準；Auto Fix 的 Gate 設計則是我們自己的專案流程。

---

## 1. 先把 PASS / FAIL / BLOCKED 定義清楚

Auto Fix 最重要的不是「會修」，而是**知道什麼情況可以修**。

建議先定義：

| 狀態 | 意義 | Auto Fix |
|---|---|---|
| PASS | 條件全部符合 | 不需要修 |
| FAIL | 可以明確指出缺失或格式問題 | 可以嘗試修正 |
| BLOCKED | 缺少必要資訊、權限或前置條件 | 不自動修 |

例如：

```text
Requirement:
「使用者登入後可以看到課程列表」
```

如果 Gate 發現：

```text
FAIL
缺少 acceptance criteria
```

這種問題可以交給 AI 補成結構化內容。

但如果：

```text
BLOCKED
找不到 requirement.md
```

就不應該讓 AI 自己猜需求。

**第一條規則：BLOCKED 不進 Auto Fix。**

---

## 2. Auto Fix 不應該直接改程式碼

這一點很重要。

今天的 Auto Fix 先處理「文件與結構化 Requirement」，不要一開始就讓 AI 改 Angular source code。

推薦：

```text
Requirement Gate
      ↓
   FAIL
      ↓
讀取 Gate report
      ↓
Claude Code Auto Fix
      ↓
只修改 requirement.md
      ↓
重新執行 Gate
```

先把邊界畫小，成功率會高很多。

之後再把相同模式延伸到：

```text
Planning
Implementation
Test
E2E
```

---

## 3. 讓 Gate 輸出機器可讀的結果

如果 Gate 只印：

```text
Requirement 不完整
```

AI 很難穩定處理。

建議增加一份 report：

```text
.gates/
└── requirement-result.json
```

例如：

```json
{
  "gate": "requirement",
  "status": "FAIL",
  "errors": [
    {
      "code": "REQ-001",
      "message": "缺少 acceptance criteria",
      "target": "docs/requirement.md"
    }
  ]
}
```

這樣 Auto Fix 就有明確 Input：

```text
Requirement Gate
      ↓
requirement-result.json
      ↓
Auto Fix Input
```

這就是 Skill Input / Output 思維真正開始發揮作用的地方。

---

## 4. Auto Fix 的 Input / Output

把 Auto Fix 本身也當成一個 Skill。

### Input

```text
Requirement 文件
+
Gate Result
+
允許修改的檔案範圍
```

### Output

```text
修改後 Requirement
+
Fix Summary
+
Ready for Re-Gate
```

可以定義成：

```text
AutoFixInput
├── requirementPath
├── gateResultPath
└── allowedFiles

AutoFixOutput
├── changedFiles
├── fixes
└── nextAction
```

其中 `allowedFiles` 很重要。

例如：

```json
{
  "allowedFiles": [
    "docs/requirement.md"
  ]
}
```

代表這一次 Auto Fix **只能改 Requirement 文件**。

---

## 5. 第一版 Auto Fix Prompt

不要一開始寫超大型 Prompt。

先把規則講清楚：

```text
你現在執行 Requirement Auto Fix。

Input:
1. 讀取 docs/requirement.md
2. 讀取 .gates/requirement-result.json

規則：
- 只修正 Gate report 指出的問題
- 只能修改 docs/requirement.md
- 不得修改 src/
- 不得修改 package.json
- 不得自行增加產品需求
- 不確定的內容不要猜，回傳 BLOCKED

完成後：
1. 說明修改了什麼
2. 不要宣稱 Gate 已通過
3. 下一步必須重新執行 Requirement Gate
```

最後一句尤其重要：

**Auto Fix 不負責宣布 PASS。**

只有 Gate 可以宣布 PASS。

---

## 6. 真正的閉環

現在流程變成：

```text
             ┌──────────────┐
             │ Requirement  │
             │    Skill     │
             └──────┬───────┘
                    ↓
             ┌──────────────┐
             │ Requirement  │
             │     Gate     │
             └──────┬───────┘
                    ↓
                 PASS?
                ↙     ↘
             PASS      FAIL
              ↓          ↓
          Planning    Auto Fix
                         ↓
                    Requirement
                         Gate
                         ↓
                      PASS?
```

而 `BLOCKED` 則直接停止：

```text
Gate
 ↓
BLOCKED
 ↓
停止
 ↓
Human / 補充資訊
```

這樣才不會形成 AI 無限自動修改。

---

## 7. 必須加上 Retry 上限

Auto Fix 一定要有上限。

例如：

```text
MAX_AUTO_FIX = 2
```

流程：

```text
Gate FAIL
 ↓
Auto Fix #1
 ↓
Gate FAIL
 ↓
Auto Fix #2
 ↓
Gate FAIL
 ↓
BLOCKED / Human Gate
```

不要做成：

```text
FAIL → Fix → FAIL → Fix → FAIL → Fix → ...
```

這是自動化流程很常見的風險。

---

## 8. Gate Script 的概念

可以先讓 Gate Script 接受固定的結果：

```bash
#!/usr/bin/env bash

RESULT_FILE=".gates/requirement-result.json"

if [ ! -f "$RESULT_FILE" ]; then
  echo "BLOCKED: requirement gate result not found"
  exit 2
fi

STATUS=$(jq -r '.status' "$RESULT_FILE")

case "$STATUS" in
  PASS)
    echo "PASS"
    exit 0
    ;;
  FAIL)
    echo "FAIL"
    exit 1
    ;;
  BLOCKED)
    echo "BLOCKED"
    exit 2
    ;;
  *)
    echo "BLOCKED: unknown gate status"
    exit 2
    ;;
esac
```

重點不是這支 Script 有多複雜，而是它建立了穩定的契約：

```text
exit 0 → PASS
exit 1 → FAIL
exit 2 → BLOCKED
```

後面的 Hook、YAML Workflow、CI 都可以重複利用這個契約。

---

## 9. Auto Fix 的安全邊界

第一版建議固定四條：

### ① 限制修改範圍

```text
allowedFiles
```

### ② 不允許自己新增需求

```text
No requirement invention
```

### ③ 不允許自己宣布 PASS

```text
Auto Fix ≠ Gate
```

### ④ 限制重試次數

```text
MAX_AUTO_FIX = 2
```

這四條會讓後面的自動化流程安全很多。

---

## 10. 如何驗證成功？

準備三個測試案例。

### Case A：第一次 Gate 就 PASS

```text
Skill
 ↓
Gate
 ↓
PASS
```

結果：

```text
不執行 Auto Fix
直接進 Planning
```

### Case B：FAIL → Auto Fix → PASS

```text
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

結果：

```text
成功形成第一個自動修復閉環
```

### Case C：FAIL → Auto Fix → FAIL → FAIL

```text
Gate
 ↓
FAIL
 ↓
Auto Fix #1
 ↓
FAIL
 ↓
Auto Fix #2
 ↓
FAIL
 ↓
BLOCKED
```

結果：

```text
停止自動化
等待人工處理
```

---

## 常見問題

### Q1：為什麼不讓 AI 一直修到 PASS？

因為 AI 可能為了讓 Gate 通過而改變原本的需求。

我們真正要的是：

```text
需求正確
 ↓
Gate PASS
```

而不是：

```text
Gate PASS
```

兩者不完全相同。

### Q2：Auto Fix 可以直接改 Angular 程式嗎？

可以，但**現在先不要**。

先把 Requirement Auto Fix 做穩，再把相同模式延伸到程式碼：

```text
Implementation Gate
 ↓
FAIL
 ↓
Auto Fix Code
 ↓
Test Gate
```

那時候還需要加入 lint、typecheck、unit test、E2E 等更嚴格的 Gate。

### Q3：Hook 是不是負責 Auto Fix？

不是。

依然維持責任分離：

```text
Skill = 做事情
Hook = 觸發
Gate = 驗收
Auto Fix = 修正
```

---

## 今天學會什麼？

今天完成了整套自動化最重要的第一個閉環：

```text
Skill
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

而且我們建立了四個重要原則：

1. **BLOCKED 不自動修。**
2. **Auto Fix 不負責宣布 PASS。**
3. **限制 Auto Fix 可以修改的檔案。**
4. **Auto Fix 必須有 Retry 上限。**

這四個原則之後可以直接延伸到 Angular Implementation。

---

## 下一步

下一篇開始把這個模式帶進 Angular 專案：

```text
Requirement
 ↓
Planning Skill
 ↓
Planning Gate
 ↓
Auto Fix
 ↓
PASS
```

接著再逐步加入你前面規劃的：

```text
Figma
 ↓
Implementation
 ↓
lint
 ↓
typecheck
 ↓
build
 ↓
E2E
```

最後才把這些 Node 用 YAML Workflow 串成完整的 AI Development Automation。

---

## 官方參考

- Claude Code Hooks：官方 Hooks reference，確認目前 Hook event、設定範圍與 command hook 行為。
- Claude Code Skills：官方 Skills 文件，確認 Skill 的建立與使用方式。

> 本篇的 `PASS / FAIL / BLOCKED`、Gate Script 與 Auto Fix 流程是專案自訂的自動化設計，不是 Claude Code 官方固定規格。
