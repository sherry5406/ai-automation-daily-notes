# Angular × SignalR｜Leader 失效後怎麼接班？｜2026-10-02

> 今天只學一個小主題：**Leader 頁籤消失後，Follower 怎麼接手 SignalR？**

## 今天學什麼

昨天我們知道多個頁籤可以透過 Leader / Follower 的概念，讓主要頁籤負責 SignalR。

今天只處理一件事：

**Leader 不見了，誰接手？**

先不要做完整分散式選舉演算法，只建立「偵測失效 → 重新競選 → 新 Leader 建立連線」的基本觀念。

## 白話原理

把多個頁籤想成一個辦公室：

```text
Tab A → Leader → 負責 SignalR
Tab B → Follower
Tab C → Follower
```

如果 Tab A 被關掉：

```text
Tab A ❌ 消失
   ↓
B / C 發現 Leader 不見了
   ↓
重新選 Leader
   ↓
Tab B 成為 Leader
   ↓
Tab B 建立 SignalR Connection
```

重點是：**Follower 平常不要因為自己不是 Leader 就一直建立 SignalR Connection。**

## 實際情境

企業 EIP 裡，使用者開了三個頁籤：

```text
案件列表   → Tab A
案件明細   → Tab B
通知中心   → Tab C
```

如果由 Tab A 負責即時通知：

```text
Backend SignalR
      ↓
   Tab A
      ↓
BroadcastChannel
   ↙       ↘
Tab B      Tab C
```

此時使用者把「案件列表」頁籤關掉：

```text
Tab A ❌
```

如果沒有接班機制，SignalR 就可能跟著消失，其他頁籤也收不到即時事件。

因此需要：

```text
Leader 消失
   ↓
偵測 timeout / heartbeat 中斷
   ↓
重新競選
   ↓
新 Leader
   ↓
建立 SignalR
```

## Angular 純前端實作重點

今天先把責任拆開：

```text
TabCoordinatorService
    ↓
負責「我是 Leader 還是 Follower？」

SignalrService
    ↓
只有 Leader 負責 HubConnection
```

不要把 Leader Election 和 SignalR Connection 全部塞進同一個 Service。

### Heartbeat 的概念

Leader 可以定期發送心跳：

```ts
setInterval(() => {
  channel.postMessage({
    type: 'leader-heartbeat',
    tabId: this.tabId,
  });
}, 2000);
```

Follower 收到心跳後記住最後一次時間：

```ts
lastHeartbeatAt = Date.now();
```

如果超過某個時間都沒有收到：

```ts
if (Date.now() - this.lastHeartbeatAt > 5000) {
  // Leader 可能已經消失
}
```

> 這只是教學用的簡化概念。實務上 timeout 要考慮瀏覽器背景頁籤節流、裝置效能與多頁籤同時搶 Leader 的競態問題。

## 簡短程式範例

### Leader 發心跳

```ts
private startHeartbeat(): void {
  this.heartbeatTimer = setInterval(() => {
    this.channel.postMessage({
      type: 'leader-heartbeat',
      tabId: this.tabId,
    });
  }, 2000);
}
```

### Follower 偵測 Leader

```ts
private checkLeader(): void {
  const expired = Date.now() - this.lastHeartbeatAt > 5000;

  if (expired) {
    this.tryBecomeLeader();
  }
}
```

### 成為 Leader 後才建立 SignalR

```ts
if (this.isLeader()) {
  await this.signalr.connect();
}
```

這裡最重要的不是程式碼，而是這個規則：

```text
Follower
   ↓
不建立主要 SignalR

Leader
   ↓
建立主要 SignalR
```

## 今天練習

想像你有：

```text
Tab A = Leader
Tab B = Follower
Tab C = Follower
```

請自己回答：

1. Tab A 關閉後，誰可以接班？
2. 新 Leader 成為 Tab B 後，誰負責建立 SignalR？
3. Tab C 需要自己建立另一條 SignalR 嗎？

答案先記住：

```text
Leader 改變
   ↓
新 Leader 建立 SignalR
   ↓
其他 Tab 繼續當 Follower
```

## 常見踩雷

### 1. 每個 Tab 發現 Leader 不見就一起搶

可能變成：

```text
Tab B → 我要當 Leader
Tab C → 我要當 Leader
```

結果兩個都建立 SignalR。

所以 Leader Election 必須處理競態。

### 2. 用固定 timeout 就認定 Leader 死掉

瀏覽器背景頁籤可能被節流，heartbeat 不一定能準時執行。

因此：

```text
沒有 heartbeat
≠
100% 確定 Leader 已關閉
```

### 3. 新 Leader 建立 Connection，但舊 Leader 其實還活著

這會產生兩條 Connection。

因此實務上需要更嚴謹的協調機制，避免「雙 Leader」。

### 4. 把 Leader 當成 Session Owner

Leader 只是：

> 「哪一個瀏覽器頁籤負責 SignalR。」

不是：

> 「哪一個頁籤代表這個使用者登入狀態。」

Session 最終仍由後端認證與 Session 規則決定。

## 今天只要記住

```text
Leader 消失
    ↓
偵測失效
    ↓
重新競選
    ↓
新 Leader
    ↓
建立 SignalR
```

最重要的一句話：

> **SignalR Connection 的生命週期，應該跟 Leader 身分綁定，而不是每一個 Tab 各自建立。**

## 下一步

下一篇進入：

**Angular × SignalR｜有效 Activity：什麼操作才算使用者還在使用系統？**

會開始把 SignalR 和企業系統很常見的：

```text
mousemove
click
keydown
route change
API activity
```

串起來，但會特別區分：

**「使用者真的有操作」和「SignalR 還連著」不是同一件事。**
