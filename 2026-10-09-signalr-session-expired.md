# 2026-10-09｜Angular × SignalR：Session Expired 與前端登出流程

## 今天學什麼

今天只學一件事：後端通知 Session 已失效時，Angular 怎麼結束目前登入狀態？

## 白話原理

前端閒置倒數到 00:00，不代表後端 Session 一定過期。瀏覽器可能休眠或節流，因此 Session 有效性以後端為準。前端負責接收既有 SignalR 事件、清理 UI 登入狀態並導回登入頁。

## 實際情境

```text
收到後端 SessionExpired 事件
       ↓
停止 SignalR Connection
       ↓
清除前端登入狀態與敏感快取
       ↓
導回登入頁
```

如果倒數到 00:00 卻沒有收到事件，應使用專案既有的 Session 驗證或 API 確認，不要只憑前端計時就判定 Session 已過期。

## Angular 純前端實作重點

- 在共用 Service 註冊事件，不要每個頁面各自處理。
- 登出清理流程加上只執行一次的保護。
- 若專案已有 AuthService、HTTP Interceptor 或共用登出流程，請整合到既有機制。
- `SessionExpired` 是範例事件名稱，請改成後端實際定義的名稱與 payload。

## 簡短程式範例

```ts
private expiredHandled = false;

registerSessionEvents(): void {
  this.connection.on('SessionExpired', () => {
    void this.handleSessionExpired();
  });
}

private async handleSessionExpired(): Promise<void> {
  if (this.expiredHandled) return;
  this.expiredHandled = true;

  try {
    await this.connection.stop();
  } catch {
    // 連線可能早已中斷，仍要繼續清理
  }

  this.clearAuthState();
  await this.router.navigate(['/login'], {
    queryParams: { reason: 'session-expired' }
  });
}
```

## 今天練習

1. 收到後端 SessionExpired：停止連線、清理狀態、導回登入頁。
2. 倒數到 00:00 但沒有事件：呼叫既有驗證流程，再依 API 結果處理。

## 常見踩雷

- 只靠前端倒數判定 Session 過期。
- 每個 Component 都註冊一次事件，導致重複登出。
- 沒有防重複處理，讓 SignalR 事件與 API 401 同時觸發兩次清理。
- 只導頁卻不清理前端登入狀態。

## 下一步

Angular 如何集中處理 HTTP 401 與 SignalR SessionExpired，避免多個入口重複執行登出流程。
