# 2026-10-10｜Angular × SignalR：集中處理 HTTP 401 與 SessionExpired

## 今天學什麼

今天只學一件事：HTTP API 回傳登入失效，或 SignalR 收到 `SessionExpired` 時，讓兩種入口共用同一套登出流程。

## 白話原理

HTTP 和 SignalR 都可能發現登入狀態失效。如果各自執行登出，可能重複停止連線、清除資料、顯示訊息或導頁。

```text
HTTP 401 ─────────┐
                  ├─→ SessionExpiryService
SignalR SessionExpired ┘        ↓
                          防止重複處理
                                ↓
                         停止 SignalR
                                ↓
                         清除登入狀態
                                ↓
                         導回登入頁
```

## 實際情境

使用者編輯資料時，API 回傳 401；同一時間也收到 SignalR 的 SessionExpired。兩個入口都通知共用處理器，而不是各自實作登出。

## Angular 純前端實作重點

概念範例：

```ts
@Injectable({ providedIn: 'root' })
export class SessionExpiryService {
  private handling = false;

  async handleExpired(): Promise<void> {
    if (this.handling) return;
    this.handling = true;

    try {
      await this.signalR.stop();
    } catch {
      // 連線可能早已中斷，仍繼續清理
    }

    this.auth.clearLocalSession();
    await this.router.navigate(['/login'], {
      queryParams: { reason: 'session-expired' },
      replaceUrl: true,
    });
  }

  resetForNewSession(): void {
    this.handling = false;
  }
}
```

上例中的 `signalR.stop()`、`clearLocalSession()` 是專案整合點，請接到既有 Service。重新登入成功後，記得重設防重複旗標。

HTTP Interceptor 只應對符合後端契約的 Session 過期回應呼叫此方法；不要把所有 401 都一律當成 SessionExpired，因為登入帳密錯誤等情況也可能回傳 401。

SignalR Service 可以透過 Observable／Subject 發出 SessionExpired 通知，由共用處理器接收，避免 SignalR Service 與登出 Service 互相注入而形成循環依賴。

## 今天練習

模擬 HTTP 401 與 SignalR SessionExpired 幾乎同時發生，確認登出清理與導頁只執行一次。

## 常見踩雷

- 所有 401 都直接登出，忽略錯誤碼與 API 契約。
- HTTP 與 SignalR 各寫一套登出流程。
- 新登入後沒有重設防重複旗標。
- 只清除前端資料，就以為伺服器端 Session 已撤銷。

前端清理不能取代後端 Session 驗證或 Token 撤銷機制。

## 下一步

下一篇：登入後啟動 SignalR，登出或 Session 失效時停止連線。
