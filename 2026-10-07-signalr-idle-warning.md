# 2026-10-07｜Angular × SignalR：Idle Warning

## 今天學什麼

今天只學一件事：使用者閒置達到提醒門檻時，先顯示提醒，不要立刻登出。

## 白話原理

假設閒置 25 分鐘顯示提醒、30 分鐘達到登出門檻：

```text
LastActivityAt
    ↓
計算 Idle Time
    ↓
達到 25 分鐘
    ↓
顯示「即將登出」
    ↓
有效操作 → 更新 Activity、關閉提醒
```

## 實際情境

企業系統可以顯示：「您已閒置一段時間，若沒有操作，系統將在 5 分鐘後結束工作階段。」

若使用者重新操作，前端更新 Activity 並關閉提醒；若沒有操作，後續仍須依後端 Session 規則處理。

## Angular 純前端實作重點

```ts
readonly showIdleWarning = signal(false);

checkIdle(): void {
  const idleTime = Date.now() - this.lastActivityAt();
  const warningTime = 25 * 60 * 1000;

  this.showIdleWarning.set(idleTime >= warningTime);
}

markActivity(): void {
  this.lastActivityAt.set(Date.now());
  this.showIdleWarning.set(false);
}
```

這是概念範例；實務上應集中管理檢查排程，避免每個 Component 各自建立 Timer。

## 今天練習

- 閒置 15 分鐘、提醒門檻 20 分鐘：不提醒。
- 閒置 20 分鐘：顯示提醒。
- 提醒期間有有效操作：更新 Activity、關閉提醒。
- 閒置達到登出門檻：依後端規則確認 Session 狀態。

## 常見踩雷

- 提醒出現後無法取消。
- 每次檢查都建立新的 Timer。
- 把前端倒數當成 Session 最終判斷。

## 下一步

下一篇：Idle Warning 倒數計時，讓使用者知道還剩多少時間。
