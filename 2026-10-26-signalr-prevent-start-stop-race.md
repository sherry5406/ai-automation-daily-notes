# Angular × SignalR｜避免 start() 與 stop() 互相競爭

> 日期：2026-10-26  
> 範圍：Angular 前端；假設後端 SignalR Hub 已完成。

## 今日主題

用同一條「生命週期佇列」依序執行 SignalR 的 `start()` 與 `stop()`，避免登入啟動尚未完成，登出就同時要求停止，造成連線狀態混亂。

## 白話解釋

昨天學到的共用 Promise，可以讓同一時間發生的多個 `start()` 共用同一個啟動結果。

但還有另一種情況：

1. 使用者登入，程式呼叫 `start()`，連線正在建立。
2. 使用者很快登出，程式又呼叫 `stop()`。
3. 如果兩個非同步操作互相重疊，程式就可能遇到難以預期的連線狀態。

可以把 `start()` 和 `stop()` 想成同一扇門的「開門」與「關門」。不要讓兩個動作同時操作；先讓前一個動作結束，再執行下一個。

今天用 Promise 佇列把生命週期操作排隊。這是前端用來管理操作順序的方式，不是 SignalR 內建的功能。

## 實務情境

使用者登入成功後，Angular 的 AuthService 啟動 SignalR；但使用者可能立刻按登出、切換帳號，或因登入狀態失效而觸發停止連線。

我們希望：

- 登入啟動先執行完，再處理後續的停止要求。
- 同時送進來的多個啟動要求，不會重複真正呼叫 `connection.start()`。
- 前一個操作失敗時，佇列仍能繼續處理下一個要求。
- 所有元件都透過同一個共用 Service 管理連線，不直接操作 HubConnection。

## Angular 實作

### 1. 在共用 Service 建立生命週期佇列

以下為精簡範例，請把 Hub URL 換成專案實際使用的路徑。

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

  // 所有 start / stop 都透過這條佇列依序執行。
  private lifecycleQueue: Promise<void> = Promise.resolve();

  private enqueueLifecycle(
    action: () => Promise<void>,
  ): Promise<void> {
    const operation = this.lifecycleQueue.then(action, action);

    // 保留 operation 的錯誤給原本的呼叫者，
    // 但讓佇列本身恢復成可繼續執行的狀態。
    this.lifecycleQueue = operation.catch(() => undefined);

    return operation;
  }

  start(): Promise<void> {
    return this.enqueueLifecycle(async () => {
      // 已連線或正在其他連線流程中，就不要再呼叫 start()。
      if (this.connection.state !== HubConnectionState.Disconnected) {
        return;
      }

      await this.connection.start();
    });
  }

  stop(): Promise<void> {
    return this.enqueueLifecycle(async () => {
      if (this.connection.state === HubConnectionState.Disconnected) {
        return;
      }

      await this.connection.stop();
    });
  }
}
```

### 2. 登入與登出都呼叫同一個 Service

```ts
// 登入成功
await this.signalRService.start();

// 登出時
await this.signalRService.stop();
```

實際專案要依登入流程決定錯誤處理，例如登入後啟動失敗時記錄錯誤或提供重試。不要因為 SignalR 連線失敗，就擅自認定使用者已登出；登入與 Session 的有效性仍由應用程式和後端規則決定。

## 這段程式如何運作？

假設啟動尚未完成時就收到停止要求：

```text
start() 進入佇列
   ↓
等待 connection.start() 完成
   ↓
stop() 才開始執行
   ↓
connection.stop()
```

如果有兩次 `start()` 幾乎同時進來：

- 第一次真正呼叫 `connection.start()`。
- 第二次等第一次結束後才檢查連線狀態。
- 如果當時已連線，第二次就直接結束，不會再呼叫一次 `start()`。

佇列中的每個操作都會依序執行；即使某次操作失敗，後續操作仍可繼續排隊。不過，失敗仍會回傳給原本呼叫它的程式，方便做錯誤處理。

## 練習

1. 在登入流程呼叫 `start()` 後，立刻呼叫 `stop()`，觀察兩個操作是否依序完成。
2. 同時呼叫兩次 `start()`，確認實際的 `connection.start()` 不會被重複執行。
3. 模擬初次連線失敗，確認下一個排隊的操作仍能執行。
4. 搜尋專案中所有直接使用 `connection.start()` 或 `connection.stop()` 的地方，將生命週期管理集中到共用 Service。

## 常見陷阱

- **只用狀態判斷，卻沒有安排操作順序：** 狀態檢查和非同步操作之間仍可能出現競爭；佇列負責讓操作依序執行。
- **錯誤後讓佇列永遠維持 rejected 狀態：** 若沒有在佇列鏈上恢復，後續操作可能都無法執行。範例用 `operation.catch(() => undefined)` 恢復佇列，但仍把原始錯誤交給呼叫者。
- **元件直接操作 HubConnection：** 只要有地方繞過共用 Service，佇列就無法協調那個操作。
- **把 `start()` 當成 Session 驗證：** SignalR 連上不代表使用者 Session 一定有效；仍要遵守後端的驗證與過期事件。
- **以為佇列能取代所有重連處理：** `withAutomaticReconnect()` 和初次連線失敗的重試策略仍需分別設計。

## 下一步

下一步可以把登入、登出、切換帳號與 SessionExpired 都整合到同一個連線生命週期入口，並測試「舊帳號的連線尚未停止，新帳號已開始登入」的情境。
