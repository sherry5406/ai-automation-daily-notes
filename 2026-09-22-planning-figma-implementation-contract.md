# AI 自動化教學 08｜Planning → Figma → Implementation Contract

> 目標：把 Planning 的輸出正式交給 Figma 與 Implementation，而不是讓下一個 Skill 重新猜需求。

## 今天要完成什麼？

上一集已經完成 Planning 的：

```text
Planning Skill
    ↓
Planning Gate
    ↓
FAIL
    ↓
Auto Fix
    ↓
Re-Gate
    ↓
PASS
```

今天先不做 YAML Workflow，也不急著進 E2E。

我們要處理下一個很容易失控的問題：

> **Planning PASS 之後，Figma 與 Implementation 到底拿到什麼？**

如果沒有明確的 Input / Output Contract，很容易變成：

```text
Planning PASS
     ↓
Figma Skill 自己猜
     ↓
Implementation Skill 再猜一次
     ↓
最後實作和 Requirement 不一致
```

今天要把它變成：

```text
Requirement
    ↓
Planning
    ↓
Planning Gate
    ↓ PASS
Figma Input Contract
    ↓
Figma Output Contract
    ↓
Implementation Input Contract
    ↓
Implementation
```

---

## 1. 核心概念：每一個 Skill 都要有「交接格式」

前面我們已經建立：

```text
Requirement Skill
    ↓
Requirement Output
    ↓
Requirement Gate
```

以及：

```text
Planning Skill
    ↓
Planning Output
    ↓
Planning Gate
```

今天再往前一步：

```text
Planning Output
       ↓
   Figma Input
       ↓
   Figma Output
       ↓
Implementation Input
```

這裡的重點不是 JSON 一定要長什麼樣子，而是：

> **下一個 Skill 不應該靠猜測上一個 Skill 的意思。**

---

## 2. 先定義 Planning Output

假設 Angular 學習平台要做：

> 影片只能跳轉到已觀看範圍。

Planning Skill 可以輸出：

```json
{
  "feature": "video-watch-progress",
  "goal": "限制使用者只能跳轉到已觀看範圍",
  "frontend": [
    "VideoPlayerComponent",
    "WatchProgressService",
    "seek validation"
  ],
  "backend": [
    "watched-ranges API",
    "progress persistence"
  ],
  "ui": [
    "video player",
    "progress indicator",
    "blocked seek feedback"
  ],
  "acceptance": [
    "已觀看區間可以回看",
    "未觀看區間不可跳轉",
    "重新進入課程後保留觀看進度"
  ]
}
```

這裡多了一個重要欄位：

```text
ui
```

因為接下來 Figma 不是重新理解整個 Requirement，而是根據已確認的 Planning，整理 UI 所需要的資訊。

---

## 3. Figma Skill 的 Input Contract

第一版不要把整份專案資訊全部塞給 Figma Skill。

只提供它真正需要的資訊：

```json
{
  "feature": "video-watch-progress",
  "goal": "限制使用者只能跳轉到已觀看範圍",
  "ui": [
    "video player",
    "progress indicator",
    "blocked seek feedback"
  ],
  "acceptance": [
    "已觀看區間可以回看",
    "未觀看區間不可跳轉"
  ]
}
```

### Figma Skill 的責任

Figma Skill 不應該重新決定：

- API 怎麼設計
- Angular Service 怎麼寫
- Backend 怎麼實作
- Requirement 是否成立

它只負責：

> **把已確認的產品需求與 Planning，轉成 UI / UX 設計需求。**

這就是 Skill Boundary。

---

## 4. Figma Output Contract

假設 Figma / UI Planning 最後產生：

```json
{
  "screens": [
    {
      "id": "course-video",
      "purpose": "觀看課程影片",
      "components": [
        "VideoPlayer",
        "ProgressBar",
        "SeekBlockedMessage"
      ],
      "states": [
        "loading",
        "playing",
        "paused",
        "seek-blocked",
        "completed"
      ],
      "responsive": [
        "desktop",
        "tablet",
        "mobile"
      ]
    }
  ]
}
```

這個 Output 很重要，因為 Implementation 不應該只收到一句：

```text
「請照 Figma 做出來」
```

而應該拿到結構化的 UI Contract。

---

## 5. Implementation Input Contract

Implementation Skill 可以收到三份已確認資訊：

```text
Requirement Output
       +
Planning Output
       +
Figma Output
       ↓
Implementation Input
```

例如：

```json
{
  "requirement": {
    "feature": "video-watch-progress",
    "goal": "限制使用者只能跳轉到已觀看範圍"
  },
  "planning": {
    "frontend": [
      "VideoPlayerComponent",
      "WatchProgressService"
    ],
    "backend": [
      "watched-ranges API"
    ]
  },
  "design": {
    "screen": "course-video",
    "components": [
      "VideoPlayer",
      "ProgressBar",
      "SeekBlockedMessage"
    ],
    "states": [
      "loading",
      "playing",
      "paused",
      "seek-blocked",
      "completed"
    ]
  }
}
```

Implementation Skill 現在才開始寫 Angular。

---

## 6. Angular Implementation 怎麼落地？

假設專案是 Angular：

```text
src/app/features/course-video/
├── components/
│   ├── video-player/
│   ├── progress-bar/
│   └── seek-blocked-message/
├── services/
│   └── watch-progress.service.ts
└── models/
    └── watch-progress.model.ts
```

Implementation Skill 的任務是依據 Contract 實作，而不是重新發明架構。

例如：

```ts
export interface WatchRange {
  start: number;
  end: number;
}
```

以及：

```ts
export interface WatchProgress {
  videoId: string;
  watchedRanges: WatchRange[];
}
```

這些細節應該由 Planning / Architecture 決策逐步產生，而不是讓 UI Skill 隨意決定。

---

## 7. 最重要的規則：Implementation 不能修改上游 Contract

這是今天最值得記住的一條規則。

如果 Implementation 發現：

```text
Planning 說要 A
但實作發現應該是 B
```

不要直接偷偷改成 B。

應該回到上游：

```text
Implementation
      ↓
發現 Contract 不足
      ↓
BLOCKED / CHANGE REQUEST
      ↓
Planning
      ↓
Planning Gate
      ↓
重新進入 Implementation
```

這樣才能避免：

```text
Requirement A
     ↓
Planning B
     ↓
Figma C
     ↓
Implementation D
```

最後每一層都「看起來合理」，但彼此完全不一致。

---

## 8. 建議建立 Artifact Chain

從今天開始，可以把每一階段的結果視為 Artifact：

```text
artifacts/
├── requirement.json
├── planning.json
├── design.json
└── implementation.json
```

例如：

```text
Requirement Artifact
        ↓
Planning Artifact
        ↓
Design Artifact
        ↓
Implementation Artifact
```

每一個 Artifact 都可以被下一個 Gate 驗證。

這會讓後面的 YAML Workflow 非常容易抽象。

---

## 9. 下一步 YAML Workflow 會長什麼樣？

現在已經有足夠的抽象，可以先想像成：

```yaml
nodes:
  - id: requirement
    skill: requirement
    gate: requirement-gate

  - id: planning
    skill: planning
    gate: planning-gate
    depends_on:
      - requirement

  - id: design
    skill: figma
    depends_on:
      - planning

  - id: implementation
    skill: implementation
    depends_on:
      - design
```

注意：

> **今天先不要真的建立 Workflow Engine。**

現在只是把 Node 所需要的 Input / Output Contract 建立好。

下一階段再把這些 Node 放進 YAML。

---

## 10. Gate 應該放在哪裡？

推薦：

```text
Requirement
   ↓
Requirement Gate
   ↓ PASS
Planning
   ↓
Planning Gate
   ↓ PASS
Figma
   ↓
Design Gate
   ↓ PASS
Implementation
   ↓
Test Gate
```

也就是：

> **每個重要 Artifact 產生後，都有一個品質邊界。**

不是只有最後 Build 才檢查。

---

## 11. Design Gate 第一版可以檢查什麼？

先不要做 AI Semantic Gate。

第一版可以 deterministic：

```text
Design Gate
├── screen 是否存在
├── component 是否存在
├── state 是否至少一個
├── responsive 是否定義
└── requirement 的主要 UI 是否有對應
```

例如：

```json
{
  "status": "FAIL",
  "gate": "design",
  "errors": [
    {
      "code": "MISSING_STATE",
      "message": "course-video 缺少 seek-blocked state"
    }
  ],
  "retryable": true
}
```

後面才可以增加 AI Semantic Gate：

```text
「這個 Design 是否真的覆蓋 Requirement？」
```

---

## 12. Angular Skill / MCP 在這一段的角色

這裡再次把三者分清楚：

```text
Angular Skill
    ↓
告訴 Agent Angular 最佳實務

Angular MCP
    ↓
提供 Angular workspace / CLI / docs 等工具能力

Automation Gate
    ↓
驗證這一步是否完成
```

所以不要把 Angular MCP 當成流程控制器。

例如：

```text
Implementation Skill
        ↓
Angular Skill
        ↓
Angular MCP
        ↓
修改 Angular workspace
        ↓
Test Gate
```

這樣 Domain Capability 與 Automation Orchestration 就分離了。

---

## 13. 如何驗證今天成功？

可以做一個最小 Demo。

### Step 1：建立 Planning Artifact

```json
{
  "feature": "course-video",
  "goal": "觀看課程影片",
  "ui": ["video-player", "progress-bar"],
  "acceptance": ["已觀看區間可回看"]
}
```

### Step 2：產生 Design Artifact

```json
{
  "screens": [
    {
      "id": "course-video",
      "components": ["VideoPlayer", "ProgressBar"],
      "states": ["loading", "playing", "paused"]
    }
  ]
}
```

### Step 3：Design Gate

```text
PASS
```

### Step 4：Implementation 收到三份 Artifact

```text
Requirement
+ Planning
+ Design
```

然後才開始 Angular 實作。

---

## 14. 常見問題

### Q1：為什麼不能直接把 Requirement 丟給 Implementation？

可以，但會失去中間的設計與驗證邊界。

我們現在做的是可重複的 AI Development Automation，因此需要讓每一階段的責任清楚。

### Q2：Figma 一定要變成 JSON 嗎？

不一定。

重點是要有一個明確的 Design Contract。JSON 只是最容易被 Script / Gate / Workflow 消費的形式之一。

### Q3：Implementation 發現設計不合理怎麼辦？

不要直接修改上游 Artifact。

回報：

```text
CHANGE_REQUEST
```

再讓 Planning / Design 重新產生並通過 Gate。

### Q4：是不是每個 Skill 都需要 Gate？

不一定。

但只要 Output 會成為下一階段的重要輸入，就值得有 Gate。

---

## 今天學會什麼？

今天完成的是：

```text
Planning
   ↓
Figma / Design
   ↓
Implementation
```

之間的資料交接設計。

最重要的概念是：

> **Skill 不是一段 Prompt，而是一個有 Input / Output Contract 的工作單位。**

因此現在整個流程開始變成：

```text
Requirement Artifact
       ↓
Requirement Gate
       ↓
Planning Artifact
       ↓
Planning Gate
       ↓
Design Artifact
       ↓
Design Gate
       ↓
Implementation
       ↓
Test Gate
```

這就是之後 YAML Workflow 能成立的基礎。

---

## 下一步

下一篇進入：

**YAML Workflow 01｜把 Skill / Gate 抽象成 Node**

會開始真正定義：

```yaml
node
skill
input
output
depends_on
on_pass
on_fail
```

先建立最小 Workflow，不一次加入 Parallel、Human Gate、完整 AI Acceptance。

等最小版本跑通後，再逐步增加能力。
