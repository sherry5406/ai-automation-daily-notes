# 2026-10-11｜Angular × SignalR：登入後啟動、登出時停止連線

## 今天學什麼

今天只學一件事：Angular 應該在什麼時機啟動 SignalR，以及登出時如何停止連線。

## 白話原理

登入成功後，前端才啟動需要登入的 SignalR 連線；使用者登出或 Session 失效時，則停止連線並清理前端狀態。

```text
登入成功
   ↓
建立／取得 HubConnection
   ↓
先註冊事件
   ↓
start()
   ↓
接收即時事件
   ↓
登出／SessionExpired
   ↓
stop()
   ↓
清除登入狀態並導回登入頁
```

## 實際情境

員工管理系統登入後要接收通知與 SessionExpired。使用者登出時，應停止即時連線並執行專案既有的登出流程。

## Angular 純前端實作重點

概念範例：

```ts
async startAfterLogin(): Promise<void> {
  if (this.connection.state === HubConnectionState.Connected) {
    return;
  }

  await this.connection.start();
}

async stopOnLogout(): Promise<void> {
  if (this.connection.state === HubConnectionState.Disconnected) {
    return;
  }

  await this.connection.stop();
}
```

這是簡化範例。只檢查 Connected 還不足以避免兩個呼叫同時通過狀態檢查；後續可用共用 Promise 防止重複啟動。應先註冊 `connection.on(...)` 事件，再呼叫 `start()`。

停止前端連線不等於伺服器端 Session 已撤銷；仍須遵循既有的登出 API 與後端驗證規則。

## 今天練習

1. 登入 API 成功後，啟動 SignalR。
2. 主動登出時，停止 SignalR、清理登入狀態並導回登入頁。
3. 收到 SessionExpired 時，走共用 SessionExpiryService。
4. 確認沒有每個 Component 都自行建立 HubConnection。

## 常見踩雷

- 每個頁面各建一條連線。
- 以為自動重連會處理初次 start() 失敗。
- 登出只導頁，沒有停止連線。
- 只停止 SignalR 就認為伺服器端已登出。
- 多個地方同時呼叫 start()，造成重複啟動競態。

## 下一步

下一篇：使用共用 Promise 管理啟動流程，避免重複呼叫 start()。
