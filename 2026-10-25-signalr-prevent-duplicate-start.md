# Angular × SignalR｜使用共用 Promise 避免重複呼叫 start()

> 日期：2026-10-25  
> 範圍：Angular 前端；假設後端 SignalR Hub 已完成。

## 今日主題

當登入流程、頁面初始化或多個元件幾乎同時呼叫 `start()` 時，如何確保同一條 SignalR 連線只啟動一次。

## 白話解釋

`connection.start()` 是非同步操作。它開始連線後，不會立刻完成，而是要等到 Promise resolve。

假設登入流程和通知元件同時呼叫 `start()`：

1. 第一次呼叫開始連線，但還在等待。
2. 第二次呼叫進來時，連線可能仍是 `Connecting`。
3. 如果程式沒有集中管理，就可能再次呼叫 `start()`，造成重複啟動或狀態錯誤。

解法是把「正在啟動的 Promise」存起來。其他呼叫者不用再啟動一次，而是共用同一個 Promise，等待同一個連線結果。

## 實務情境

使用者登入時，`AuthService` 會啟動 SignalR；同時，導覽列的通知元件也會確認 SignalR 是否已啟動。兩邊可能在很短的時間內呼叫服務。

這種情況下，SignalR 連線應由一個共用的 Angular singleton service 管理，而不是讓每個元件各自建立或啟動連線。

## Angular 實作

### 1. 在共用 Service 管理 HubConnection

以下是精簡範例。請將 Hub URL 換成專案實際使用的路徑。

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

  // 保存目前正在執行的 start()，讓同時進來的呼叫共用它。
  private startPromise: Promise<void> | null = null;

  start(): Promise<void> {
    // 若已有啟動流程，直接回傳同一個 Promise。
    if (this.startPromise) {
      return this.startPromise;
    }

    // 已連線、正在重連，或正處於其他非 Disconnected 狀態時，
    // 不要再呼叫 start()。
    if (this.connection.state !== HubConnectionState.Disconnected) {
      return Promise.resolve();
    }

    this.startPromise = this.connection
      .start()
      .catch((error) => {
        console.error('SignalR 初次連線失敗', error);
        throw error; // 讓呼叫端知道啟動失敗
      })
      .finally(() => {
        // 成功或失敗都清除 Promise。
        // 成功後由 connection.state 判斷已連線；
        // 失敗後下次呼叫才有機會重新嘗試。
        this.startPromise = null;
      });

    return this.startPromise;
  }

  stop(): Promise<void> {
    return this.connection.stop();
  }
}
```

### 2. 呼叫端等待同一個啟動結果

```ts
async onLoginSuccess(): Promise<void> {
  try {
    await this.signalRService.start();
    console.log('SignalR 啟動流程完成');
  } catch {
    // 依專案需求顯示錯誤或安排稍後重試。
    // 不要假設初次連線失敗時 withAutomaticReconnect() 一定會重試。
  }
}
```

## 這段程式的關鍵

- **先檢查 `startPromise`：** 如果已經有人開始啟動，後來的呼叫者就共用它。
- **再檢查連線狀態：** 不是 `Disconnected` 就不再呼叫 `start()`。
- **用 `finally()` 清除 Promise：** 不會把已完成或失敗的 Promise 永遠留著。
- **失敗時重新拋出錯誤：** 呼叫端可以用 `try/catch` 做適當處理。

例如，同時有 3 個地方呼叫 `start()`，而第一次啟動仍在等待時，三個呼叫會共用同一個 Promise，而不是執行 3 次 `connection.start()`。

## 練習

1. 在專案中搜尋所有 `connection.start()`，確認是否只有共用的 SignalR Service 會直接呼叫它。
2. 在登入流程和另一個初始化流程中，同時呼叫 `signalRService.start()`。
3. 暫時觀察啟動結果，確認同一時間只執行一次真正的 `connection.start()`。
4. 測試初次連線失敗後再次呼叫 `start()`，確認失敗的 Promise 已被清除，後續呼叫可以重新嘗試。

## 常見陷阱

- **每個元件都自己建立 HubConnection：** 共用 Promise 只能協調同一個 Service 實例，不能合併不同的連線物件。
- **只檢查 `connection.state`：** 只看狀態不一定能讓呼叫者共用同一個啟動結果；保存 Promise 才能處理「啟動中」的併發呼叫。
- **吞掉錯誤：** 若 catch 後只記錄錯誤、不重新拋出，呼叫端可能誤以為啟動成功。
- **以為自動重連會處理所有失敗：** `withAutomaticReconnect()` 主要處理連線建立後的意外中斷，不會自動重試第一次 `start()` 失敗。
- **把這個方法當成跨分頁鎖：** Service singleton 只在同一個 Angular 執行環境內共用；不同瀏覽器分頁仍需要另外的跨分頁協調機制。

## 下一步

下一篇可進一步處理「登入切換或登出時，如何避免尚未完成的 start() 與 stop() 互相競爭」，讓 SignalR 連線生命週期更穩定。
