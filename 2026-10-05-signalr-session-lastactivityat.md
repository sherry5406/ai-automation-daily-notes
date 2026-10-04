# Angular × SignalR｜Session LastActivityAt｜2026-10-05

> 今天只學一件事：**怎麼用 LastActivityAt 判斷「使用者最後一次有效操作」？**

## 今天學什麼

昨天我們學的是 Activity 回報頻率：

```text
使用者操作很多次
      ↓
前端不要每次都送 SignalR
      ↓
控制回報頻率
```

今天再往前一步，學一個企業系統很常見的欄位：

```text
LastActivityAt
```

白話就是：

> **這個使用者最後一次被系統認定為「有在使用」的時間。**

今天先不要做完整閒置登出，只把「最後活動時間」這個概念弄懂。

---

## 白話原理

假設使用者 09:00 登入：

```text
09:00 登入
09:10 查詢案件
09:18 編輯資料
09:25 儲存
```

那麼最後一次有效操作是：

```text
LastActivityAt = 09:25
```

之後如果使用者一直沒有操作，前端可以知道：

```text
現在時間 - LastActivityAt
```

例如現在 09:40：

```text
09:40 - 09:25 = 15 分鐘
```

這就是後面「閒置多久」判斷的基礎。

---

## 實際情境：企業 EIP

假設公司規定：

> 使用者閒置 30 分鐘後，需要提醒使用者；超過規則後 Session 失效。

流程可以先想成：

```text
使用者操作
   ↓
前端 Activity
   ↓
回報後端
   ↓
後端更新 LastActivityAt
   ↓
繼續使用
```

如果很久沒有有效操作：

```text
LastActivityAt
      ↓
計算 Idle Time
      ↓
接近閒置門檻
      ↓
顯示提醒
```

今天先停在這裡。

**真正的 Session 是否有效，最後仍然由後端決定。**

---

## Angular 純前端實作重點

可以先在 Angular Service 裡保存前端目前認知的最後活動時間：

```ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ActivityService {
  readonly lastActivityAt = signal<number>(Date.now());

  markActivity(): void {
    this.lastActivityAt.set(Date.now());
  }
}
```

例如使用者完成一次查詢：

```ts
search(): void {
  this.activityService.markActivity();

  // 呼叫查詢 API
}
```

這裡的重點不是 `Date.now()` 本身，而是：

> **把「有效操作發生了」集中成一個可以管理的 Activity。**

---

## SignalR 在這裡負責什麼？

不要把 SignalR 想成「計時器」。

比較合理的分工是：

```text
Angular
  ↓
偵測有效操作
  ↓
ActivityService
  ↓
控制回報頻率
  ↓
SignalR / API
  ↓
後端 Session 規則
```

SignalR 是即時通道，不是 Session 規則本身。

所以不要在前端自己決定：

```ts
if (idleMinutes > 30) {
  // 一定已經 Logout
}
```

比較安全的想法是：

```text
前端：我好像已經閒置很久
        ↓
後端：Session 是否真的失效？
        ↓
後端給最後結果
```

---

## 一個很重要的觀念：前端時間 ≠ 後端權威時間

這是今天最值得記住的地方。

前端可以有：

```ts
lastActivityAt = Date.now();
```

但這個時間只是：

> **瀏覽器認為最後一次操作的時間。**

真正的 Session 有效時間，企業系統通常應由後端掌握。

例如：

```text
Browser
LastActivityAt = 10:00

Server
LastActivityAt = 09:59:58
```

有些微差異很正常。

不要讓前端的時間直接凌駕後端 Session 規則。

---

## 今天練習

想像你正在做企業 EIP：

使用者完成以下操作：

```text
09:00 登入
09:05 點選「案件查詢」
09:08 滑鼠移動
09:12 點擊「查詢」
09:20 閱讀頁面
09:30 點擊「儲存」
```

請回答：

### 哪些比較適合算「有效 Activity」？

```text
A. 登入
B. 滑鼠移動
C. 點擊查詢
D. 閱讀頁面
E. 儲存
```

建議答案：

```text
A / C / E
```

至於滑鼠移動，要看產品需求；通常不建議每一次滑鼠事件都當成一次需要回報後端的 Activity。

---

## 常見踩雷

### 1. 每次 mousemove 都更新後端

```ts
window.addEventListener('mousemove', () => {
  connection.invoke('Activity');
});
```

這很容易產生大量訊息。

應該先在前端節流，再回報。

---

### 2. 前端自己決定 Session 已經失效

```ts
if (idleMinutes >= 30) {
  logout();
}
```

這種寫法容易把「前端 UI 閒置」和「後端 Session 有效性」混在一起。

後面會學比較完整的：

```text
Activity
   ↓
LastActivityAt
   ↓
Idle Time
   ↓
Warning
   ↓
SessionExpired
```

---

### 3. 把 SignalR Connected 當成 Activity

```text
SignalR Connected
      ≠
使用者正在操作
```

SignalR 可能一直保持 Connected，但使用者已經把瀏覽器放著半小時沒有操作。

---

## 今天只要記住

```text
LastActivityAt
      ↓
「最後一次有效操作」
      ↓
用來計算 Idle Time
      ↓
後面才能做閒置提醒
```

再記住一句：

> **前端可以偵測 Activity，但 Session 是否有效，最終以後端規則為準。**

---

## 下一步

下一篇進入：

**Angular × SignalR｜Idle Time：怎麼知道使用者閒置多久？**

會開始把：

```text
LastActivityAt
      ↓
現在時間 - LastActivityAt
      ↓
Idle Time
      ↓
Warning
```

真正串起來。
