# Angular × SignalR｜收到事件後，要 `update()` 還是重新 Call API？｜2026-09-28

> 今天只學一件事：**SignalR 收到資料後，不是每次都要重新 Call API。要看後端送來的是「完整資料」還是「只是通知有變化」。**

## 今天學什麼

昨天我們學會：

```text
SignalR
   ↓
connection.on('ReceiveNotification', ...)
   ↓
Angular Signal
   ↓
畫面更新
```

今天多走一步：

> **收到 SignalR 事件後，什麼時候直接 `update()`？什麼時候再 Call API？**

先記一個最簡單的規則：

```text
完整資料       → 可以直接 update()
只有 ID / 變更通知 → 再 Call API
```

---

## 白話原理

把 SignalR 想成公司同事打電話告訴你：

### 情況 A：同事把完整資料告訴你

> 「案件 A-001 審核完成，狀態 Approved，金額 500 萬，更新時間 10:30。」

你已經知道完整資訊了。

所以前端可以直接更新 Signal。

### 情況 B：同事只告訴你「案件變了」

> 「嘿，案件 A-001 有變化。」

你不知道到底變成什麼。

這時候就去 API 拿最新資料。

```text
SignalR
   ↓
「A-001 變了」
   ↓
GET /api/cases/A-001
   ↓
拿最新完整資料
   ↓
更新 Signal
```

---

## 實際情境 1：通知 → 直接 `update()`

昨天的 Notification 範例就是很好的情境。

後端傳來完整資料：

```ts
{
  id: 'N001',
  title: '案件審核完成',
  message: '案件 A-2026-001 已完成審核'
}
```

前端：

```ts
this.connection.on(
  'ReceiveNotification',
  (message: NotificationMessage) => {
    this.notifications.update(list => [
      message,
      ...list,
    ]);
  },
);
```

假設原本有 5 筆：

```text
A
B
C
D
E
```

收到新的 `F`：

```ts
[
  message,
  ...list,
]
```

結果：

```text
F  ← 新通知
A
B
C
D
E
```

所以總共 **6 筆**。

`...list` 是把原本 5 筆展開；`message` 放在最前面。

這種情況不需要為了顯示這一筆通知，再多打一支 API。

---

## 實際情境 2：案件更新 → Call API

假設後端只傳：

```ts
{
  caseId: 'A-2026-001'
}
```

這代表：

> 「這個案件有變化。」

但前端不知道：

```text
狀態變什麼？
價格變什麼？
誰修改？
什麼時間修改？
其他欄位有沒有變？
```

所以比較安全的方式是：

```ts
this.connection.on('CaseUpdated', async (caseId: string) => {
  const latestCase = await this.caseApi.getById(caseId);

  this.cases.update(list =>
    list.map(item =>
      item.id === latestCase.id
        ? latestCase
        : item,
    ),
  );
});
```

流程：

```text
SignalR
   ↓
CaseUpdated
   ↓
caseId
   ↓
GET API
   ↓
最新完整案件
   ↓
Signal.update()
   ↓
UI
```

---

## 實際情境 3：SignalR 直接傳完整案件 → 不用 Call API

如果後端直接傳完整資料：

```ts
{
  id: 'A-2026-001',
  status: 'Approved',
  price: 5000000,
  updatedAt: '2026-09-28T07:00:00'
}
```

前端就可以直接更新：

```ts
this.connection.on('CaseUpdated', (updatedCase) => {
  this.cases.update(list =>
    list.map(item =>
      item.id === updatedCase.id
        ? updatedCase
        : item,
    ),
  );
});
```

不需要：

```text
SignalR → API → API → API
```

多打一層沒有必要的 API。

---

## Angular `update()` 到底在做什麼？

例如：

```ts
this.notifications.update(list => [
  message,
  ...list,
]);
```

可以白話理解成：

```text
拿目前的 list
      ↓
加入新的 message
      ↓
產生新的陣列
      ↓
把 Signal 更新成新的陣列
```

如果：

```text
list = [A, B, C, D, E]
message = F
```

結果：

```text
[F, A, B, C, D, E]
```

不是：

```text
[F, [A, B, C, D, E]]
```

因為 `...list` 會把陣列內容展開。

---

## 為什麼不要直接 `push()`？

不要把 Signal 裡面的陣列直接改掉：

```ts
list.push(message);
```

今天先記住比較簡單的寫法：

```ts
this.notifications.update(list => [
  message,
  ...list,
]);
```

也就是：

> **拿舊資料 → 產生新資料 → 更新 Signal。**

這樣資料流會比較清楚，也比較符合 Angular Signal 的使用方式。

---

## 一張圖記住今天的觀念

```text
                 SignalR
                    ↓
               收到事件
                    ↓
          ┌─────────┴─────────┐
          ↓                   ↓
      完整資料             只有 ID / 通知
          ↓                   ↓
    直接 update()          Call API
          ↓                   ↓
          └─────────┬─────────┘
                    ↓
              Angular Signal
                    ↓
                   UI
```

---

## 今天的判斷口訣

### ① 完整資料

```text
SignalR 已經把我要的資料給我
↓
直接 update()
```

### ② 只有 ID

```text
SignalR 只告訴我「誰變了」
↓
Call API
↓
拿最新資料
↓
update()
```

### ③ 整批最新資料

如果後端一次把整批最新資料送過來：

```ts
this.notifications.set(latestList);
```

這時比較適合 `set()`，因為你不是「在舊資料上加一筆」，而是「整批替換成最新資料」。

---

## 今天練習

想像你正在做企業 EIP：

### A. 通知鈴鐺

後端傳：

```ts
{
  id: 'N001',
  title: '審核完成',
  message: '案件 A-001 已完成'
}
```

問自己：

> 要 `update()` 還是 Call API？

答案：**直接 `update()`。**

### B. 案件狀態

後端只傳：

```ts
'A-001'
```

問自己：

> 要 `update()` 還是 Call API？

答案：**Call API 拿最新案件，再 update。**

---

## 常見踩雷

### 1. SignalR 收到任何事件都 Call API

不一定需要。

如果事件本身已經帶完整資料，再 Call API 只是多一次網路請求。

### 2. SignalR 收到任何資料都直接覆蓋

也不一定對。

如果只有 `caseId`，你不能把 `caseId` 當成完整案件塞進列表。

### 3. 把 SignalR 當成資料庫

SignalR 比較像：

> 「通知管道」

API 比較像：

> 「取得最新資料的管道」

所以兩者通常是合作，而不是互相取代。

---

## 今天只要記住一句話

> **SignalR 告訴你「發生什麼事」；API 負責讓你取得「最新完整資料」。但如果 SignalR 已經給你完整資料，就可以直接更新 Signal。**

---

## 下一步

下一篇進入：

**Angular × SignalR｜斷線與自動重連**

會把前面學過的：

```text
Disconnected
Connecting
Connected
Reconnecting
```

真正串起來，理解 `withAutomaticReconnect()` 在實務上怎麼處理網路短暫中斷。
