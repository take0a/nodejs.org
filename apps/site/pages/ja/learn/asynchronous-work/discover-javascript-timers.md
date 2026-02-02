---
title: Discover JavaScript Timers
layout: learn
authors: flaviocopes, MylesBorins, LaRuaNa, amiller-gh, ahmadawais, ovflowd
---

# JavaScript タイマーについて学ぶ

## `setTimeout()`

JavaScript コードを書いているときに、関数の実行を遅らせたい場合があります。

これが `setTimeout` の役割です。後で実行するコールバック関数と、その関数の実行をどれくらい遅らせたいかを表す値（ミリ秒単位）を指定します。

```js
setTimeout(() => {
  // runs after 2 seconds
}, 2000);

setTimeout(() => {
  // runs after 50 milliseconds
}, 50);
```

この構文は新しい関数を定義します。この中で任意の関数を呼び出すことも、既存の関数名とパラメータのセットを渡すこともできます。

```js
const myFunction = (firstParam, secondParam) => {
  // do something
};

// runs after 2 seconds
setTimeout(myFunction, 2000, firstParam, secondParam);
```

`setTimeout` は、Node.js では [`Timeout`](https://nodejs.org/api/timers.html#class-timeout) インスタンスを返しますが、ブラウザでは数値のタイマー ID を返します。このオブジェクトまたは ID は、スケジュールされた関数の実行をキャンセルするために使用できます。

```js
const timeout = setTimeout(() => {
  // should run after 2 seconds
}, 2000);

// I changed my mind
clearTimeout(timeout);
```

### 遅延ゼロ

タイムアウト遅延を `0` に指定すると、コールバック関数は可能な限り早く実行されますが、現在の関数の実行後に以下の処理が行われます。

```js
setTimeout(() => {
  console.log('after ');
}, 0);

console.log(' before ');
```

このコードは次のように出力されます

```bash
before
after
```

これは、スケジューラで関数をキューイングすることで、高負荷タスクによるCPUのブロックを回避し、高負荷計算の実行中に他の関数を実行できるようにする場合に特に便利です。

> 一部のブラウザ（IEとEdge）には、これと全く同じ機能を実行する `setImmediate()` メソッドが実装されていますが、これは標準ではなく、[他のブラウザでは利用できません](https://caniuse.com/#feat=setimmediate)。ただし、Node.jsでは標準関数です。

## `setInterval()`

`setInterval` は `setTimeout` に似た関数ですが、コールバック関数を1回実行するのではなく、指定した時間間隔（ミリ秒単位）で永続的に実行するという点が異なります。

```js
setInterval(() => {
  // runs every 2 seconds
}, 2000);
```

上記の関数は、`clearInterval` を使用して停止するように指示し、`setInterval` が返した間隔 ID を渡さない限り、2 秒ごとに実行されます。

```js
const timeout = setInterval(() => {
  // runs every 2 seconds
}, 2000);

clearInterval(timeout);
```

setIntervalコールバック関数内で`clearInterval`を呼び出すのは一般的です。これにより、再実行または停止を自動判断できます。例えば、次のコードはApp.somethingIWaitの値が`arrived`でない限り、何かを実行します。

```js
const interval = setInterval(() => {
  if (App.somethingIWait === 'arrived') {
    clearInterval(interval);
  }
  // otherwise do things
}, 100);
```

## 再帰的な setTimeout

`setInterval` は、関数の実行終了時刻を考慮せずに、n ミリ秒ごとに関数を開始します。

関数の実行時間が常に同じであれば、問題ありません。

![setInterval は正常に動作しています](/static/images/learn/javascript-timers/setinterval-ok.png)

ネットワーク状況によっては、関数の実行時間が異なる場合があります。例えば、次のようになります。

![setInterval の実行時間が変動します](/static/images/learn/javascript-timers/setinterval-varying-duration.png)

また、長時間実行される関数が次の関数と重複する場合もあります。

![setInterval が重複しています](/static/images/learn/javascript-timers/setinterval-overlapping.png)

これを回避するには、コールバック関数の終了時に再帰的な setTimeout が呼び出されるようにスケジュールを設定します。

```js
const myFunction = () => {
  // do something

  setTimeout(myFunction, 1000);
};

setTimeout(myFunction, 1000);
```

このシナリオを実現するには、次のコードを使用します。

![再帰的な setTimeout](/static/images/learn/javascript-timers/recursive-settimeout.png)

`setTimeout` と `setInterval` は、Node.js の [Timers モジュール](https://nodejs.org/api/timers.html) を通じて利用できます。

Node.js には `setImmediate()` も用意されています。これは `setTimeout(() => {}, 0)` と同等で、主に Node.js イベントループで使用されます。
