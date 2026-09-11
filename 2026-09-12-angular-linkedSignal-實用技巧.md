# 2026-09-12｜Angular `linkedSignal`：會跟著來源變，但又可以讓使用者自己改

## 一句話

`linkedSignal` 適合「預設值跟著另一個狀態變，但使用者又可以自己選」的情境。

Angular 官方文件指出，`linkedSignal` 是可寫入的 Signal，而且它的初始／重設值會跟著來源的 reactive computation 更新。citeturn0search1turn0search10

## 最常見的例子

例如下拉選單：

```ts
shippingOptions = signal([
  '宅配',
  '超商取貨',
  '郵寄',
]);

selectedOption = linkedSignal(() => this.shippingOptions()[0]);
```

一開始：

```text
selectedOption = 宅配
```

使用者可以自己選：

```ts
this.selectedOption.set('超商取貨');
```

如果之後整個選項清單換掉：

```ts
this.shippingOptions.set([
  '門市取貨',
  '郵局',
]);
```

`selectedOption` 會重新依照新的來源取得有效值。

## 為什麼不用 `effect()`？

很多人第一個想法會是：

```ts
shippingOptions = signal(...);
selectedOption = signal(...);

effect(() => {
  this.selectedOption.set(this.shippingOptions()[0]);
});
```

但 Angular 官方建議：`effect()` 應該是最後才考慮的 API。單純處理「一個狀態跟著另一個狀態變」時，優先考慮 `computed()` 或 `linkedSignal()`。citeturn0search0

白話：

```text
只是狀態之間有關係
→ computed / linkedSignal

真的要跟外部世界同步
→ effect
```

## 前端實務可以用在哪？

很適合：

- 下拉選單的預設選項
- 篩選條件
- 分頁選擇
- 商品規格選擇
- API 回來後的預設值
- Parent Input 改變後，仍允許使用者修改的值

## 今天學會什麼？

不要看到兩個 Signal 有關係就先寫 `effect()`。

先問自己：

> 「這個值是不是跟另一個狀態有關，而且又允許使用者自己改？」

如果是，可以優先看看 `linkedSignal`。

參考：Angular 官方 Signals / linkedSignal 文件。citeturn0search10turn0search0
