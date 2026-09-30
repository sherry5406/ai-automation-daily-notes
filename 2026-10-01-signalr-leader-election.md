# Angular × SignalR｜Leader Election：多頁籤到底誰負責連 SignalR？｜2026-10-01

> 今天只學一件事：**同一個使用者開很多頁籤時，怎麼選出一個頁籤當 Leader，負責 SignalR？**

## 今天學什麼

昨天我們知道「單一裝置、單一頁籤」的目標，不一定是禁止使用者開很多頁籤，而是避免每個頁籤都建立一條主要 SignalR Connection。

今天只先理解一個概念：**Leader Election（選出一個 Leader）**。

先不要急著做完整的分散式鎖，也不處理所有瀏覽器例外。

今天只建立這個架構觀念：

```text
Tab A ──┐
Tab B ──┼→ Tab Coordination → 選出 Leader
Tab C ──┘                         ↓
                               SignalR
                                  ↓
                         收到即時事件
                                  ↓
                         同步給其他 Tab
```

## 白話原理

把瀏覽器的多個頁籤想成同一個辦公室的 3 個人：

```text
小明（Tab A）
小華（Tab B）
小美（Tab C）
```

如果 3 個人都各自打電話給同一個 SignalR Hub：

```text
Tab A → SignalR Connection 1
Tab B → SignalR Connection 2
Tab C → SignalR Connection 3
```

就可能造成不必要的連線與事件處理。

另一種設計是先選一個人當「代表」：

```text
Tab A → Leader → SignalR
Tab B → Follower
Tab C → Follower
```

Leader 收到通知後，再透過瀏覽器頁籤之間的通訊機制，把資料同步給其他頁籤。

今天先記住：

> **Leader 的責任是「代表這個瀏覽器分頁群組處理 SignalR」。**

## 實際情境：企業 EIP 通知

假設使用者登入企業系統後開了 3 個頁籤：

```text
案件列表
案件明細
Dashboard
```

我們希望不要變成：

```text
3 個頁籤
↓
3 條 SignalR Connection
↓
同一個通知收到 3 次
```

而是：

```text
3 個頁籤
   ↓
選出 1 個 Leader
   ↓
Leader 建立 SignalR Connection
   ↓
收到「案件已更新」
   ↓
通知其他頁籤
```

這樣可以把「即時連線」集中管理。

## Angular 純前端實作重點

今天先不要把 Leader Election 寫成很複雜的 Service。

先把責任拆開：

```text
SignalrService
    ↓
只負責 SignalR Connection

TabCoordinatorService
    ↓
只負責「我是 Leader 還是 Follower」
```

這個拆法很重要。

不要讓 `SignalrService` 同時負責：

- 判斷目前是不是 Leader
- 搶 Leader
- 處理其他 Tab
- 建立 SignalR
- 收通知
- Session Activity

否則最後會變成一個很難維護的巨大 Service。

## 簡短程式範例

先假設我們有一個很簡化的狀態：

```ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class TabCoordinatorService {
  readonly isLeader = signal(false);

  becomeLeader(): void {
    this.isLeader.set(true);
  }

  becomeFollower(): void {
    this.isLeader.set(false);
  }
}
```

SignalR Service 可以根據這個狀態決定要不要建立 Connection：

```ts
if (this.tabCoordinator.isLeader()) {
  await this.connectSignalR();
}
```

這只是今天的「概念版」，**還不是完整的 Leader Election**。

真正的問題是：

> 「到底誰先成為 Leader？」

這就是下一層要處理的事情。

## Tab 之間怎麼傳訊息？

瀏覽器本身有可以讓不同頁籤溝通的機制，例如 `BroadcastChannel`。

概念上可以想成：

```ts
const channel = new BroadcastChannel('app-tab');

channel.postMessage({
  type: 'NotificationReceived',
  data: message,
});
```

其他頁籤可以接收：

```ts
channel.onmessage = event => {
  console.log(event.data);
};
```

今天先不用把它做完整。

只要理解：

```text
SignalR
  ↓
Leader Tab
  ↓
BroadcastChannel
  ↓
Follower Tabs
```

## 今天練習

想像你開了：

```text
Tab A
Tab B
Tab C
```

請回答：

1. 哪一個 Tab 建立 SignalR Connection？
2. Tab B 收到通知時，應該自己再建立 SignalR 嗎？
3. Leader 關掉後，剩下的 Tab 怎麼辦？

今天只要先想通第 1 題：

> **應該只有 Leader 建立主要 SignalR Connection。**

## 常見踩雷

### 1. 每個 Tab 都建立 Connection

最直覺，但可能造成：

```text
Tab A → Connection
Tab B → Connection
Tab C → Connection
```

如果後端會對每個 Connection 發送同一個事件，前端就可能重複處理。

### 2. 看到 `localStorage` 就直接當成完整 Leader Election

例如：

```ts
localStorage.setItem('leader', 'true');
```

這只能當成很粗略的示意。

真正實務上還要考慮：

- 兩個 Tab 同時啟動
- Leader 關閉
- Leader 瀏覽器 Crash
- Leader 沒有正常清理狀態
- Follower 怎麼接班
- 多個 Tab 同時認為自己是 Leader

所以今天不要把 `localStorage` 這一行當成完整解法。

### 3. 把 Leader 概念和登入 Session 混在一起

Leader 只是「誰代表這組 Tab 管理 SignalR」。

它不是：

```text
Leader = 登入者
```

也不是：

```text
Follower = 沒登入
```

Session 是否有效仍然由後端認證與 Session 規則決定。

## 今天只要記住

```text
多個 Tab
   ↓
選一個 Leader
   ↓
Leader 建立 SignalR
   ↓
Leader 收到即時事件
   ↓
BroadcastChannel
   ↓
其他 Tab 更新自己的 UI
```

而且：

> **Leader Election 是「誰負責連線」的問題，不是「誰有權限」的問題。**

## 下一步

下一篇進入：

**Angular × SignalR｜Leader 掉線了怎麼辦？**

會開始理解：

```text
Leader
  ↓
關閉 / Crash / 網路中斷
  ↓
Follower 發現 Leader 不在
  ↓
重新選出 Leader
  ↓
新的 Leader 建立 SignalR
```

這才是真正的 **Leader Election / Tab Takeover**。

## 官方參考

- MDN：Broadcast Channel API
- Microsoft Learn：ASP.NET Core SignalR JavaScript Client
- Angular 官方：Signals
