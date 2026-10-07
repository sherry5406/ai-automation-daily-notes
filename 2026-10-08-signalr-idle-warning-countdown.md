# 2026-10-08｜Angular × SignalR：Idle Warning 倒數計時

## 今天學什麼

今天只學一件事：

> 閒置提醒出現後，怎麼做「還有幾秒會登出」的倒數？

今天先不處理真正的登出，只處理前端 UI 倒數。

## 白話原理

假設：

- 閒置 25 分鐘：顯示提醒
- 閒置 30 分鐘：Session 到期

當提醒出現時，可以直接算：

```
剩餘時間 = Logout 門檻 - Idle Time
```

例如：

```
Idle Time = 27:30
Logout 門檻 = 30:00

剩餘 = 02:30
```

重點是：**不要每秒累加自己的計數器當成真正的 Session 狀態。**

比較穩定的做法是每次更新時都重新用時間戳計算：

```
remaining = logoutAt - Date.now()
```

## 實際情境

企業系統常見流程：

```
使用者停止操作
      ↓
Idle Warning
      ↓
還有 05:00
      ↓
還有 04:59
      ↓
使用者重新操作
      ↓
Warning 關閉
      ↓
重新計算 Activity
```

如果完全沒有操作：

```
倒數到 00:00
      ↓
前端進入 Session Expired 流程
      ↓
由後端確認 Session 狀態
```

## Angular 純前端實作重點

先準備兩個時間：

```ts
const WARNING_TIME = 25 * 60 * 1000;
const LOGOUT_TIME = 30 * 60 * 1000;
```

顯示剩餘秒數：

```ts
readonly remainingSeconds = signal(0);

updateCountdown(): void {
  const idleTime = Date.now() - this.lastActivityAt();
  const remaining = Math.max(LOGOUT_TIME - idleTime, 0);

  this.remainingSeconds.set(Math.ceil(remaining / 1000));
}
```

畫面可以：

```html
@if (showIdleWarning()) {
  <p>您已閒置一段時間</p>
  <p>將在 {{ remainingSeconds() }} 秒後登出</p>
}
```

今天先記住一個原則：

> **倒數是 UI 顯示，不是 Session 真相。**

## 今天練習

假設登出門檻是 30 分鐘：

1. Idle 25 分鐘 → 剩餘多久？
2. Idle 27 分 30 秒 → 剩餘多久？
3. Idle 29 分 59 秒 → 顯示什麼？
4. 使用者在倒數期間操作 → 應該怎麼辦？

答案：

```
25:00 → 05:00
27:30 → 02:30
29:59 → 00:01

使用者操作
  ↓
markActivity()
  ↓
關閉 Warning
  ↓
重新計算 Idle Time
```

## 常見踩雷

### 1. 只用 setInterval 累減數字

例如：

```ts
seconds--;
```

如果瀏覽器背景分頁被節流，計時可能不準。

### 2. 把倒數歸零當成後端 Session 一定過期

前端計時器只是 UI。

### 3. 使用者操作後忘記清除 Warning

重新 Activity 後，提醒應該消失，倒數也應重新計算。

### 4. 建立很多個 Timer

Activity Service 應集中管理計時器，避免重複訂閱。

## 下一步

下一篇學：

> **Session Expired：閒置時間到了，Angular 要怎麼處理登出？**

會開始把：

```
Idle Warning
   ↓
Countdown
   ↓
Session Expired
   ↓
Logout / 導向登入頁
```

串成完整流程。

