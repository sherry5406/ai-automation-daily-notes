# Angular × SignalR｜有效 Activity：什麼操作才算使用者還在使用系統？｜2026-10-03

> 今天只學一件事：**不要把「SignalR 有連線」當成「使用者正在使用系統」。**

## 今天學什麼

昨天學了 Leader Failover，今天開始進入企業系統很常見的 **Session Activity**。

我們先只回答一個問題：

> 使用者到底做了什麼，才算「還活躍」？

今天先不做完整的閒置登出，只建立 Activity 的觀念。

## 白話原理

想像公司 EIP：

使用者登入後一直開著頁面，但人其實已經離開電腦。

這時：

```text
SignalR Connected
        ≠
使用者正在操作
```

SignalR 只代表：

> 「前端和 Hub 目前有一條即時連線。」

它不能代表使用者真的還在看頁面。

所以企業系統通常會另外記錄：

```text
LastActivityAt
```

意思就是：

> 「使用者最後一次被判定為有效操作的時間。」

## 實際情境

例如使用者正在操作案件系統：

```text
09:00 登入
09:03 點擊案件
09:05 編輯資料
09:08 儲存
```

可以把最後一次有效操作更新成：

```text
LastActivityAt = 09:08
```

接著使用者 20 分鐘都沒有操作：

```text
09:28
↓
沒有新的有效 Activity
↓
可能進入閒置警告
```

注意：**今天只學 Activity，不決定幾分鐘後登出。**

## 哪些行為可以算 Activity？

企業系統通常可以挑選真正代表「使用者正在使用系統」的操作，例如：

```text
滑鼠點擊
鍵盤輸入
表單操作
路由切換
儲存資料
查詢資料
```

但不要看到任何瀏覽器事件都回報。

例如：

```text
mousemove
scroll
resize
```

如果每次都送 SignalR，可能變成：

```text
使用者移動滑鼠
↓
mousemove
↓
SignalR
↓
mousemove
↓
SignalR
↓
大量事件
```

這是不好的設計。

## Angular 純前端實作重點

可以先建立一個 Activity Service，讓整個 Angular App 統一管理最後一次 Activity。

```ts
@Injectable({ providedIn: 'root' })
export class ActivityService {
  readonly lastActivityAt = signal<Date | null>(null);

  markActivity(): void {
    this.lastActivityAt.set(new Date());
  }
}
```

Component 或全域事件監聽器發現有效操作時：

```ts
this.activity.markActivity();
```

今天先做到這裡。

不要急著加入 Timer、Logout 或 SignalR。

## SignalR 在這裡扮演什麼角色？

之後完整架構可能會是：

```text
使用者操作
    ↓
ActivityService
    ↓
判斷是否需要回報
    ↓
SignalR
    ↓
Session Activity
    ↓
後端記錄 LastActivityAt
```

但有一個重要原則：

> **前端可以回報 Activity，但「Session 是否真的有效」最後還是由後端決定。**

前端不能自己宣稱：

```text
「我有操作，所以 Session 一定還有效。」
```

## 今天練習

想像你正在做企業 EIP。

請把下面行為分成「可能算 Activity」和「不適合每次都回報」：

```text
點擊查詢
輸入文字
儲存表單
切換路由
mousemove
scroll
resize
```

一個合理的第一版可以是：

```text
✅ 點擊查詢
✅ 輸入文字
✅ 儲存表單
✅ 切換路由

⚠️ mousemove
⚠️ scroll
⚠️ resize
```

後面如果真的需要監聽這些事件，再使用 debounce / throttle 降低回報頻率。

## 常見踩雷

### 1. 把 SignalR Connected 當成 Activity

```text
Connected
↓
「使用者還活躍」
```

❌ 不一定。

使用者可能開著瀏覽器去吃午餐。

### 2. 所有瀏覽器事件都打 SignalR

```text
mousemove → SignalR
scroll → SignalR
resize → SignalR
```

❌ 很容易造成大量不必要的事件。

### 3. 前端自己決定 Session 已經失效

Session 的最終有效性應該由後端認證／Session 規則決定。

前端主要負責：

```text
收集有效 Activity
↓
回報
↓
顯示 UI
↓
收到 SessionExpired 後處理登出
```

## 今天只要記住一句話

> **SignalR 是即時連線；Activity 是使用者真的有操作；Session 是否有效則由後端決定。**

三件事情不要混在一起。

## 下一步

下一篇：

**Angular × SignalR｜Session LastActivityAt：多久回報一次才合理？**

會開始討論：

```text
每次操作都送？
      ↓
還是 30 秒一次？
      ↓
Debounce？Throttle？
      ↓
怎麼避免 SignalR 被打爆？
```
