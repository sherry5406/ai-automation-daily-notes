# Angular × SignalR｜Activity 回報頻率｜2026-10-04

> 今天只學一件事：**使用者有操作，不代表每一次操作都要送一次 SignalR。**

## 今天學什麼

昨天知道了什麼叫「有效 Activity」。

今天進一步學：

> **Activity 要多久回報一次後端？**

最重要的觀念是：

```text
使用者一直操作
    ↓
不要每個事件都送 SignalR
    ↓
debounce / throttle
    ↓
控制回報頻率
```

## 白話原理

假設使用者一直移動滑鼠：

```text
mousemove
mousemove
mousemove
mousemove
mousemove
mousemove
...
```

如果每次都：

```ts
connection.invoke('Activity');
```

很容易變成大量即時訊息。

企業系統通常更在意的是：

> 「這個人最近還有在使用系統嗎？」

而不是：

> 「他這 1 秒鐘動了幾次滑鼠？」

所以前端可以把大量操作整理成較低頻率的 Activity 回報。

## 實際情境

例如企業 EIP 設定：

```text
使用者最後一次有效操作：10:00:00

10:00:01  點擊
10:00:02  輸入
10:00:03  切換欄位
10:00:04  點擊
10:00:05  儲存
```

不需要送 5 次：

```text
Activity
Activity
Activity
Activity
Activity
```

可以整理成：

```text
使用者持續有操作
       ↓
前端更新 lastActivityAt
       ↓
每隔一段時間回報一次
       ↓
後端更新 Session Activity
```

## Angular 純前端實作重點

今天先用最簡單的概念，不急著加入 RxJS 複雜操作。

```ts
private lastActivityAt = Date.now();

recordActivity(): void {
  this.lastActivityAt = Date.now();
}
```

然後另外用計時器定期判斷：

```ts
setInterval(() => {
  const now = Date.now();

  if (now - this.lastActivityAt < 60_000) {
    this.reportActivity();
  }
}, 30_000);
```

意思是：

```text
每 30 秒檢查一次
        ↓
最近 60 秒有操作？
        ↓
       Yes
        ↓
回報 Activity
```

> 這只是教學範例。實際專案應依後端 Session 規則決定回報間隔，不要直接照抄數字。

## debounce 和 throttle，今天先懂差別

### debounce

> 「等使用者停止操作一段時間，再做一次。」

例如搜尋框：

```text
輸入 A
輸入 AB
輸入 ABC
輸入 ABCD
     ↓
停止輸入
     ↓
才 Call API
```

### throttle

> 「不管事件多密集，固定時間最多做一次。」

例如：

```text
mousemove 很頻繁
      ↓
最多每 30 秒回報一次 Activity
```

**Activity 通常比較適合用 throttle / 固定間隔控制。**

## 今天練習

想想這些操作：

```text
滑鼠移動
鍵盤輸入
點擊按鈕
切換路由
儲存資料
```

請分類：

```text
高頻事件
→ 不應每次都送 SignalR

低頻／明確操作
→ 可以直接更新 Activity
```

今天只需要理解「控制頻率」，先不要實作完整 ActivityService。

## 常見踩雷

### 1. 每個 mousemove 都 invoke SignalR

```ts
window.addEventListener('mousemove', () => {
  connection.invoke('Activity');
});
```

不建議。

### 2. debounce / throttle 的時間亂設定

時間應該配合後端 Session timeout 設計，而不是前端自己猜一個數字。

### 3. 把 SignalR 連線當成 Activity

```text
Connected
≠
使用者正在操作
```

SignalR 可以連著，但使用者可能已經離開電腦很久。

### 4. 前端自己決定 Session 是否有效

前端可以負責回報 Activity，但：

> **Session 最終是否有效，仍以後端規則為準。**

## 今天只要記住

```text
Activity Event 很多
        ↓
不要全部送 SignalR
        ↓
前端先控制頻率
        ↓
定期回報 Activity
        ↓
後端判斷 Session
```

## 下一步

下一篇：

**Angular × SignalR｜閒置計時器：怎麼知道使用者多久沒操作？**

會開始把：

```text
lastActivityAt
      ↓
idle time
      ↓
warning
      ↓
SessionExpired
```

串成真正的「閒置登出」流程。
