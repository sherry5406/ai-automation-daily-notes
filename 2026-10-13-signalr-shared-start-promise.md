# Angular × SignalR：使用共用 Promise 避免重複呼叫 `start()`

- 日期：2026-10-13
- 今日主題：用共用 Promise 確保同一時間只有一次初始連線
- 前提：後端 Hub 已完成；以下只處理 Angular 前端。

## 1. 白話解釋

假設登入完成後，頁面元件、通知服務都想啟動 SignalR。如果它們各自呼叫 `connection.start()`，短時間內就可能重複啟動，造成連線狀態錯誤或難以追查的例外。

**共用 Promise** 的想法很簡單：第一次呼叫時真的啟動連線，並把這次工作的 Promise 存起來；其他同時進來的呼叫，直接共用同一個 Promise，不再呼叫第二次 `start()`。

可以把它想成大家共用同一張「連線處理中的號碼牌」，而不是每個人都重新開一條連線。

## 2. 實務情境

使用者登入後，可能同時發生：

- AppComponent 想啟動即時通知。
- NotificationService 想確保連線可用。
- 路由切換後的頁面也呼叫了初始化方法。

這些呼叫可能幾乎同時發生。把啟動邏輯集中在一個 SignalR Service，並共用啟動中的 Promise，就能避免多個呼叫一起執行 `start()`。

## 3. Angular 實作

先安裝套件（若專案尚未安裝）：

```bash
npm install @microsoft/signalr
```

在單一服務中建立 HubConnection，並讓所有呼叫端只使用 `ensureStarted()`。以下是精簡示範：

```ts
import { Injectable } from '@angular/core';
import {
  HubConnection,
  HubConnectionBuilder,
  HubConnectionState,
} from '@microsoft/signalr';

@Injectable({ providedIn: 'root' })
export class SignalRService {
  private readonly connection: HubConnection =
    new HubConnectionBuilder()
      .withUrl('/hubs/notifications')
      .withAutomaticReconnect()
      .build();

  private startPromise: Promise<void> | null = null;

  ensureStarted(): Promise<void> {
    if (this.connection.state === HubConnectionState.Connected) {
      return Promise.resolve();
    }

    // 同一時間已有啟動工作：共用它，不再呼叫 start()
    if (this.startPromise) {
      return this.startPromise;
    }

    // Connecting / Reconnecting 時不要再呼叫 start()。
    // 正常架構下，所有初始啟動都必須經過本方法。
    if (this.connection.state !== HubConnectionState.Disconnected) {
      return Promise.resolve();
    }

    this.startPromise = this.connection.start().finally(() => {
      // 成功或失敗都清掉；失敗後才能再次嘗試
      this.startPromise = null;
    });

    return this.startPromise;
  }
}
```

登入成功後呼叫：

```ts
this.signalRService.ensureStarted().catch(error => {
  console.error('SignalR 初始連線失敗', error);
  // 可顯示連線提示，或安排由使用者觸發的重試
});
```

注意：範例的 `withUrl('/hubs/notifications')` 要換成專案實際的 Hub URL，並依專案的驗證方式設定 token 或 Cookie。

## 4. 程式碼重點

- `startPromise === null`：目前沒有初始啟動工作。
- `this.startPromise = this.connection.start()...`：把正在進行的工作保存起來。
- 第二個呼叫若在第一次完成前進來，會直接拿到同一個 Promise。
- `finally()` 不論成功或失敗都會清除 Promise；初始連線失敗時，後續才有機會重新嘗試。
- 已經是 `Connected` 就直接返回，不必再啟動。

## 5. 練習

在兩個不同的地方幾乎同時呼叫：

```ts
this.signalRService.ensureStarted();
this.signalRService.ensureStarted();
```

請觀察並回答：

1. 在第一次 `start()` 尚未完成時，第二次呼叫會不會再執行一次 `start()`？
2. 如果初始連線失敗，`startPromise` 會不會一直卡住？
3. 如果連線已經是 `Connected`，還會不會呼叫 `start()`？

預期答案：不會重複啟動；失敗後 Promise 會被清除；已連線時直接返回。

## 6. 常見陷阱

1. **不要在多個元件各自建立 HubConnection。** 共用 Promise 只能協調同一個服務實例；每個元件各建一條連線仍然是多條連線。
2. **不要讓其他程式碼直接呼叫 `connection.start()`。** 所有初始啟動應集中經過 `ensureStarted()`，否則無法保證共用 Promise 有效。
3. **`withAutomaticReconnect()` 不會自動處理第一次 `start()` 失敗。** 初始連線失敗仍需由應用程式決定何時重試。
4. **不要在 `Reconnecting` 時再次呼叫 `start()`。** 自動重連流程應由 SignalR 管理；可以透過 `onreconnected` 更新 UI 狀態。
5. **這個 Promise 只防止同一個 Service 實例中的重複初始啟動。** 它不會解決多個瀏覽器分頁之間的連線競爭；那是另一個跨分頁協調問題。

## 7. 下一步

下一篇可以把這個模式延伸成完整的連線管理：統一處理初始連線失敗、重試時機，以及 `Connected / Reconnecting / Disconnected` 狀態，避免元件各自判斷連線狀態。
