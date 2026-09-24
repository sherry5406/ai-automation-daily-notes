# 2026-09-25｜Angular × SignalR：第一步建立 HubConnection

> 今天只學一件事：**Angular 前端怎麼建立一條 SignalR 連線。**
>
> 假設後端 Hub 已經完成，我們今天不碰 C#，只看 Angular 前端。

## 今天學什麼？

昨天先理解 SignalR：

- 一般 HTTP API：前端主動問後端
- SignalR：前端建立長連線後，後端可以主動通知前端

今天把這件事真正寫成 Angular 程式：

```text
Angular
  ↓
HubConnectionBuilder
  ↓
withUrl('/notificationHub')
  ↓
build()
  ↓
connection.start()
  ↓
Connected
```

## 白話原理

可以把 `HubConnection` 想成「前端和後端之間的一條電話線」。

```text
Angular App
    │
    │ 建立連線
    ▼
SignalR Hub
    │
    │ 保持即時通訊
    ▼
後端
```

`HubConnectionBuilder` 是「組裝電話」的地方。

`withUrl()` 告訴它：「我要打哪一支電話。」

`build()` 把設定好的連線物件建立出來。

`start()` 才是真的撥出去。

Microsoft 官方 JavaScript Client 範例也是使用 `HubConnectionBuilder`、`withUrl()`、`build()`，最後呼叫 `connection.start()` 建立連線。citeturn0search0turn0search2

## 實際情境：企業系統通知

例如登入 EIP 後，後端可能需要即時通知：

> 「你的簽核單已經被主管退回。」

如果使用一般 API，前端可能每隔幾秒問一次：

```text
前端：有新通知嗎？
後端：沒有。

前端：有新通知嗎？
後端：沒有。

前端：有新通知嗎？
後端：有！
```

SignalR 則比較像：

```text
前端：我先跟你保持連線。

後端：有新通知了！
前端：收到。
```

這就是 SignalR 很適合「即時通知」的原因。

## Angular 純前端實作

先安裝 JavaScript Client：

```bash
pnpm add @microsoft/signalr
```

接著建立一個 Service，例如：

```text
src/app/core/signalr/
└── signalr.service.ts
```

最小版本：

```ts
import { Injectable } from '@angular/core';
import { HubConnection, HubConnectionBuilder } from '@microsoft/signalr';

@Injectable({ providedIn: 'root' })
export class SignalrService {
  private readonly connection: HubConnection;

  constructor() {
    this.connection = new HubConnectionBuilder()
      .withUrl('/notificationHub')
      .build();
  }

  async start(): Promise<void> {
    await this.connection.start();
    console.log('SignalR Connected');
  }
}
```

### 先記住 4 個東西

| 程式 | 白話意思 |
|---|---|
| `HubConnectionBuilder` | 開始組裝 SignalR 連線 |
| `withUrl()` | 指定後端 Hub URL |
| `build()` | 建立 HubConnection |
| `start()` | 真正開始連線 |

今天先不要急著加重連、事件、Session Activity。

先把「**連得上**」這件事情搞懂。

## Angular Service 為什麼比較適合？

不要每個 Component 都自己建立一條 SignalR：

```text
❌ HeaderComponent → 一條連線
❌ HomeComponent   → 一條連線
❌ NoticeComponent → 一條連線
```

這很容易變成：

```text
同一個頁籤
   ↓
建立很多 SignalR 連線
```

比較好的方向是：

```text
Angular App
     ↓
SignalrService
     ↓
一個 HubConnection
     ↓
各 Component 使用 Service
```

這也會為後面「**單一裝置、單一頁籤**」的設計打基礎。

## 今天先不做自動重連

這點很重要。

今天先不要直接寫：

```ts
.withAutomaticReconnect()
```

因為我們現在是在學「建立連線」。

下一階段才會處理：

```text
Connected
   ↓
網路斷線
   ↓
Reconnecting
   ↓
重新連線成功
   ↓
Connected
```

SignalR JavaScript Client **預設不會自動重連**；如果要自動重連，需要另外設定 `withAutomaticReconnect()`。官方預設重連等待時間為 0、2、10、30 秒，共四次嘗試。citeturn0search0

## 今天練習

請在自己的 Angular 專案完成：

### Step 1

安裝：

```bash
pnpm add @microsoft/signalr
```

### Step 2

建立：

```text
signalr.service.ts
```

### Step 3

完成：

```ts
new HubConnectionBuilder()
  .withUrl('你的後端 Hub URL')
  .build();
```

### Step 4

呼叫：

```ts
await connection.start();
```

### Step 5

開 F12 Console，確認看到：

```text
SignalR Connected
```

## 常見踩雷

### 1. `build()` 不等於已連線

```ts
const connection = new HubConnectionBuilder()
  .withUrl('/notificationHub')
  .build();
```

這只是建立「連線物件」。

真正開始連線是：

```ts
await connection.start();
```

### 2. URL 寫錯

例如後端真正是：

```text
/api/notificationHub
```

前端卻寫：

```text
/notificationHub
```

就會連不上。

### 3. 每個 Component 都 new 一次

這很容易造成同一頁有多條連線。

今天先養成習慣：

> **SignalR 連線集中在 Service 管理。**

### 4. 一開始就把所有功能塞進 Service

不要今天就同時處理：

- 重連
- 通知
- Session Activity
- 閒置登出
- 單頁籤鎖定

先把連線建立好。

## 今天學會什麼？

今天只要記住這一條：

```text
HubConnectionBuilder
        ↓
     withUrl()
        ↓
       build()
        ↓
       start()
        ↓
    Connected
```

如果這條流程懂了，後面的 SignalR 前端程式就會容易很多。

## 下一步

明天進入：

**Angular × SignalR：連線生命週期**

會開始認識：

```text
Disconnected
      ↓
Connecting
      ↓
Connected
      ↓
Reconnecting
      ↓
Connected / Disconnected
```

並開始理解 `onreconnecting`、`onreconnected`、`onclose` 到底是在什麼時候觸發。

## 官方參考

- Microsoft Learn：ASP.NET Core SignalR JavaScript Client
- Microsoft Learn：Get started with ASP.NET Core SignalR

urlSignalR JavaScript Client 官方文件https://learn.microsoft.com/aspnet/core/signalr/javascript-client
