---
title: Understanding setImmediate()
layout: learn
authors: flaviocopes, MylesBorins, LaRuaNa, ahmadawais, clean99, ovflowd
---

# setImmediate() を理解する

コードを非同期で、かつできるだけ早く実行したい場合、Node.js が提供する `setImmediate()` 関数を使用するという選択肢があります。

```js
setImmediate(() => {
  // run something
});
```

setImmediate() 引数として渡される関数は、イベントループの次の反復処理で実行されるコールバックです。

`setImmediate()` は、`setTimeout(() => {}, 0)`（0ms のタイムアウトを渡す）や `process.nextTick()`、`Promise.then()` とどう違うのでしょうか？

`process.nextTick()` に渡される関数は、イベントループの現在の反復処理において、現在の処理が終了した後に実行されます。つまり、常に `setTimeout` と `setImmediate` の前に実行されます。

0ms の遅延を持つ `setTimeout()` コールバックは `setImmediate()` と非常によく似ています。実行順序は様々な要因によって異なりますが、どちらもイベントループの次の反復処理で実行されます。

`process.nextTick` コールバックが `process.nextTick queue` に追加されます。 `Promise.then()` コールバックが `promises microtask queue` に追加されました。`setTimeout` および `setImmediate` コールバックが `macrotask queue` に追加されました。

イベントループは、まず `process.nextTick queue` 内のタスクを実行し、次に `promises microtask queue` を実行し、最後に `macrotask queue` を実行します。

`setImmediate()`、`process.nextTick()`、`Promise.then()` の順序を示す例を以下に示します。

```js
const baz = () => console.log('baz');
const foo = () => console.log('foo');
const zoo = () => console.log('zoo');

const start = () => {
  console.log('start');
  setImmediate(baz);
  new Promise((resolve, reject) => {
    resolve('bar');
  }).then(resolve => {
    console.log(resolve);
    process.nextTick(zoo);
  });
  process.nextTick(foo);
};

start();

// start foo bar zoo baz
```

このコードはまず `start()` を呼び出し、次に `process.nextTick queue` 内の `foo()` を呼び出します。その後、`promises microtask queue` を処理し、`bar` を出力すると同時に `process.nextTick queue` に `zoo()` を追加します。そして、追加されたばかりの `zoo()` を呼び出します。最後に、`macrotask queue` 内の `baz()` が呼び出されます。

上記の原則は CommonJS の場合にも当てはまりますが、ES モジュール（例: `mjs` ファイル）では実行順序が異なることに注意してください。

```js
// start bar foo zoo baz
```

これは、ロードされるESモジュールが非同期操作としてラップされているため、スクリプト全体が実際には既に「promises microtask queue」に格納されているからです。そのため、promiseが即座に解決されると、そのコールバックが「microtask」キューに追加されます。Node.jsは他のキューに移動するまでキューをクリアしようとするため、最初に「bar」が出力されます。
