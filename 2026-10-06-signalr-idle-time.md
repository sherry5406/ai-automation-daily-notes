# 2026-10-06｜Angular × SignalR：Idle Time

## 今天學什麼

今天只學一件事：如何用最後一次有效操作時間，計算使用者已經閒置多久。

## 白話原理

`LastActivityAt` 是系統最後一次認定使用者有有效操作的時間。

```text
Idle Time = 現在時間 - LastActivityAt
```

例如最後操作是 10:00，現在是 10:20，Idle Time 就是 20 分鐘。

## 實際情境

企業系統可能設定閒置 25 分鐘顯示提醒、30 分鐘進入登出流程：

```text
最後操作 10:00
    ↓
10:20：閒置 20 分鐘
    ↓
10:25：顯示提醒
    ↓
10:30：依後端 Session 規則確認是否失效
```

## Angular 純前端實作重點

先把最後操作時間存成 Signal：

```ts
readonly lastActivityAt = signal(Date.now());

markActivity(): void {
  this.lastActivityAt.set(Date.now());
}

getIdleTime(): number {
  return Date.now() - this.lastActivityAt();
}
```

有效操作發生時呼叫 `markActivity()`。不要把 SignalR 仍然 Connected 當成使用者仍在操作。

## 今天練習

若 `LastActivityAt = 09:00`，現在是 09:23，閒置時間就是 23 分鐘。若提醒門檻是 20 分鐘，應顯示提醒；若登出門檻是 30 分鐘，還沒到前端預定的提醒／登出門檻。

## 常見踩雷

- 把 SignalR Connected 當成 Activity。
- 每次 mousemove 都直接呼叫後端。
- 讓前端計時器決定後端 Session 是否有效。

前端負責偵測與呈現，Session 最終有效性以後端規則為準。

## 下一步

下一篇：Idle Warning——閒置時先提醒使用者，而不是直接登出。
