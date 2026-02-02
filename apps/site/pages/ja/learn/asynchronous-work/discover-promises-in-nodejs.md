---
title: Discover Promises in Node.js
layout: learn
authors: avivkeller
---

# Node.js で Promise を学ぶ

**Promise** は、JavaScript における特別なオブジェクトで、非同期処理の最終的な完了（または失敗）とその結果の値を表します。基本的に、Promise は、まだ利用できないが将来利用可能になる値のプレースホルダーです。

Promise はピザを注文するようなものです。すぐにピザが届くわけではありませんが、配達員は後で届けると約束します。正確な時間はわかりませんが、結果は「ピザが配達される」か「何か問題が発生した」かのどちらかであることが分かります。

## Promise の状態

Promise は以下の 3 つの状態のいずれかになります。

- **Pending**: 初期状態で、非同期処理がまだ実行中です。
- **Fulfilled**: 処理が正常に完了し、Promise は値で解決されました。
- **Rejected**: 処理が失敗し、Promise は理由（通常はエラー）で解決されました。

ピザを注文した時は、お腹が空いて期待に胸を膨らませている「Pending」状態です。ピザが熱々でチーズたっぷりで届いたら、「Fulfilled」状態です。しかし、レストランからピザを床に落としてしまったという電話がかかってきたら、「Rejected」状態です。

食事が満足いく結果に終わろうと、がっかりした結果に終わろうと、最終的な結果が出れば、Promise は「Settled**」状態とみなされます。

## Promise の基本構文

Promise を作成する最も一般的な方法の一つは、`new Promise()` コンストラクタを使用することです。このコンストラクタは、`resolve` と `reject` という2つの引数を持つ関数を受け取ります。これらの関数は、Promise を **pending** 状態から **fulfilled** または **rejected** 状態に遷移させるために使用されます。

Executor 関数内でエラーが発生した場合、Promise はそのエラーとともに拒否されます。
Executor 関数の戻り値は無視されます。Promise を確定するには、`resolve` または `reject` のみを使用してください。

```js
const myPromise = new Promise((resolve, reject) => {
  const success = true;

  if (success) {
    resolve('Operation was successful!');
  } else {
    reject('Something went wrong.');
  }
});
```

上記の例では、次のようになります。

- `success` 条件が `true` の場合、Promise は満たされ、値 `'操作は成功しました!'` が `resolve` 関数に渡されます。
- `success` 条件が `false` の場合、Promise は拒否され、エラー `'問題が発生しました。'` が `reject` 関数に渡されます。

## `.then()`、`.catch()`、`.finally()` による Promise の処理

Promise を作成したら、`.then()`、`.catch()`、`.finally()` メソッドを使用して結果を処理できます。

- `.then()` は、完了した Promise を処理し、その結果にアクセスするために使用されます。
- `.catch()` は、拒否された Promise を処理し、発生する可能性のあるエラーをキャッチするために使用されます。
- `.finally()` は、Promise が解決されたか拒否されたかに関係なく、完了した Promise を処理するために使用されます。

```js
const myPromise = new Promise((resolve, reject) => {
  const success = true;

  if (success) {
    resolve('Operation was successful!');
  } else {
    reject('Something went wrong.');
  }
});

myPromise
  .then(result => {
    console.log(result); // This will run if the Promise is fulfilled
  })
  .catch(error => {
    console.error(error); // This will run if the Promise is rejected
  })
  .finally(() => {
    console.log('The promise has completed'); // This will run when the Promise is settled
  });
```

## Promise の連鎖

Promise の優れた機能の一つは、複数の非同期処理を連鎖できることです。Promise を連鎖させると、各 `.then()` ブロックは前の処理が完了するまで待機してから実行されます。

```js
const { setTimeout: delay } = require('node:timers/promises');

const promise = delay(1000).then(() => 'First task completed');

promise
  .then(result => {
    console.log(result); // 'First task completed'
    return delay(1000).then(() => 'Second task completed'); // Return a second Promise
  })
  .then(result => {
    console.log(result); // 'Second task completed'
  })
  .catch(error => {
    console.error(error); // If any Promise is rejected, catch the error
  });
```

## Promise で async/await を使用する

現代の JavaScript で Promise を使用する最適な方法の一つは、**async/await** を使用することです。これにより、同期しているように見える非同期コードを記述できるため、可読性と保守性が大幅に向上します。

- `async` は、Promise を返す関数を定義するために使用されます。
- `await` は、`async` 関数内で、Promise が処理を完了するまで実行を一時停止するために使用されます。

```js
async function performTasks() {
  try {
    const result1 = await promise1;
    console.log(result1); // 'First task completed'

    const result2 = await promise2;
    console.log(result2); // 'Second task completed'
  } catch (error) {
    console.error(error); // Catches any rejection or error
  }
}

performTasks();
```

`performTasks` 関数では、`await` キーワードによって、各 Promise が次のステートメントに進む前に確実に処理が完了するようにしています。これにより、非同期コードのフローがより直線的で読みやすくなります。

基本的に、上記のコードは、ユーザーが次のように記述した場合と同じように実行されます。

```js
promise1
  .then(function (result1) {
    console.log(result1);
    return promise2;
  })
  .then(function (result2) {
    console.log(result2);
  })
  .catch(function (error) {
    console.log(error);
  });
```

### トップレベルの await

[ECMAScript モジュール](https://nodejs.org/api/esm.html)を使用する場合、モジュール自体は非同期処理をネイティブにサポートするトップレベルのスコープとして扱われます。つまり、`async` 関数を必要とせずに [トップレベルで `await`](https://nodejs.org/api/esm.html#top-level-await) を使用できます。

```mjs
import { setTimeout as delay } from 'node:timers/promises';

await delay(1000);
```

async/await は、ここで紹介した簡単な例よりもはるかに複雑になることがあります。Node.js 技術運営委員会のメンバーである James Snell が、Promise と async/await の複雑さを詳細に解説した [詳細なプレゼンテーション](https://www.youtube.com/watch?v=XV-u_Ow47s0) を公開しています。

## Promise ベースの Node.js API

N​​ode.js は、多くのコア API の **Promise ベース** を提供しています。特に、従来は非同期処理がコールバックで処理されていた場合に有効です。これにより、Node.js API と Promise の連携が容易になり、「コールバック地獄」のリスクが軽減されます。

例えば、`fs` (ファイルシステム) モジュールには、`fs.promises` 以下に Promise ベースの API があります。

```js
const fs = require('node:fs').promises;
// Or, you can import the promisified version directly:
// const fs = require('node:fs/promises');

async function readFile() {
  try {
    const data = await fs.readFile('example.txt', 'utf8');
    console.log(data);
  } catch (err) {
    console.error('Error reading file:', err);
  }
}

readFile();
```

この例では、`fs.readFile()` は Promise を返します。これを `async/await` 構文を使用して処理し、ファイルの内容を非同期的に読み取ります。

## 高度な Promise メソッド

JavaScript の `Promise` グローバル変数には、複数の非同期タスクをより効率的に管理するのに役立つ強力なメソッドがいくつか用意されています。

### `Promise.all()`

このメソッドは Promise の配列を受け取り、すべての Promise が満たされると解決される新しい Promise を返します。いずれかの Promise が拒否された場合、`Promise.all()` は直ちに拒否します。ただし、拒否された場合でも Promise は実行を継続します。大量の Promise を処理する場合、特にバッチ処理では、この関数を使用するとシステムのメモリに負担がかかる可能性があります。

```js
const { setTimeout: delay } = require('node:timers/promises');

const fetchData1 = delay(1000).then(() => 'Data from API 1');
const fetchData2 = delay(2000).then(() => 'Data from API 2');

Promise.all([fetchData1, fetchData2])
  .then(results => {
    console.log(results); // ['Data from API 1', 'Data from API 2']
  })
  .catch(error => {
    console.error('Error:', error);
  });
```

### `Promise.allSettled()`

このメソッドは、すべての Promise が解決または拒否されるまで待機し、各 Promise の結果を記述するオブジェクトの配列を返します。

```js
const promise1 = Promise.resolve('Success');
const promise2 = Promise.reject('Failed');

Promise.allSettled([promise1, promise2]).then(results => {
  console.log(results);
  // [ { status: 'fulfilled', value: 'Success' }, { status: 'rejected', reason: 'Failed' } ]
});
```

`Promise.all()` とは異なり、`Promise.allSettled()` は失敗時にショートサーキットを行いません。一部の Promise が拒否された場合でも、すべての Promise が解決するまで待機します。これにより、失敗の有無にかかわらずすべてのタスクのステータスを把握したいバッチ処理において、より適切なエラー処理が可能になります。

### `Promise.race()`

このメソッドは、最初の Promise が解決されるとすぐに、それが解決されるか拒否されるかに関係なく、解決または拒否されます。どの Promise が先に解決されるかに関係なく、すべての Promise が完全に実行されます。

```js
const { setTimeout: delay } = require('node:timers/promises');

const task1 = delay(2000).then(() => 'Task 1 done');
const task2 = delay(1000).then(() => 'Task 2 done');

Promise.race([task1, task2]).then(result => {
  console.log(result); // 'Task 2 done' (since task2 finishes first)
});
```

### `Promise.any()`

このメソッドは、Promise のいずれかが解決されるとすぐに解決されます。すべての Promise が拒否された場合は、`AggregateError` で拒否されます。

```js
const { setTimeout: delay } = require('node:timers/promises');

const api1 = delay(2000).then(() => 'API 1 success');
const api2 = delay(1000).then(() => 'API 2 success');
const api3 = delay(1500).then(() => 'API 3 success');

Promise.any([api1, api2, api3])
  .then(result => {
    console.log(result); // 'API 2 success' (since it resolves first)
  })
  .catch(error => {
    console.error('All promises rejected:', error);
  });
```

### `Promise.reject()` と `Promise.resolve()`

これらのメソッドは、拒否または解決された Promise を直接作成します。

```js
Promise.resolve('Resolved immediately').then(result => {
  console.log(result); // 'Resolved immediately'
});
```

### `Promise.try()`

`Promise.try()` は、同期または非同期を問わず、指定された関数を実行し、その結果を Promise でラップするメソッドです。関数がエラーをスローするか、拒否された Promise を返した場合、`Promise.try()` は拒否された Promise を返します。関数が正常に完了した場合、返された Promise は指定された値で満たされます。

これは、Promise チェーンを一貫した方法で開始するのに特に便利です。特に、同期的にエラーをスローする可能性のあるコードを扱う場合に役立ちます。

```js
function mightThrow() {
  if (Math.random() > 0.5) {
    throw new Error('Oops, something went wrong!');
  }
  return 'Success!';
}

Promise.try(mightThrow)
  .then(result => {
    console.log('Result:', result);
  })
  .catch(err => {
    console.error('Caught error:', err.message);
  });
```

この例では、`Promise.try()` によって、`mightThrow()` がエラーをスローした場合に `.catch()` ブロック内でキャッチされることが保証されるため、同期エラーと非同期エラーの両方を一箇所で処理しやすくなります。

### `Promise.withResolvers()`

このメソッドは、新しい Promise とそれに関連付けられた resolve 関数および reject 関数を作成し、それらを便利なオブジェクトで返します。これは、例えば、Promise を作成し、それを executor 関数の外部から後から resolve または refuse する必要がある場合などに使用されます。

```js
const { promise, resolve, reject } = Promise.withResolvers();

setTimeout(() => {
  resolve('Resolved successfully!');
}, 1000);

promise.then(value => {
  console.log('Success:', value);
});
```

この例では、`Promise.withResolvers()` を使用することで、executor 関数をインラインで定義することなく、Promise がいつどのように解決または拒否されるかを完全に制御できます。このパターンは、イベント駆動型プログラミング、タイムアウト、または Promise ベース以外の API との統合でよく使用されます。

## Promise によるエラー処理

Promise でエラーを処理することで、予期しない状況が発生した場合でもアプリケーションが正しく動作することが保証されます。

- Promise の実行中に発生したエラーや拒否は、`.catch()` を使用して処理できます。

```js
myPromise
  .then(result => console.log(result))
  .catch(error => console.error(error)) // Handles the rejection
  .finally(error => console.log('Promise completed')); // Runs regardless of promise resolution
```

- あるいは、`async/await` を使用する場合は、`try/catch` ブロックを使用してエラーをキャッチして処理することもできます。

```js
async function performTask() {
  try {
    const result = await myPromise;
    console.log(result);
  } catch (error) {
    console.error(error); // Handles any errors
  } finally {
    // This code is executed regardless of failure
    console.log('performTask() completed');
  }
}

performTask();
```

## イベントループでのタスクのスケジューリング

Node.js は、Promise に加えて、イベントループでタスクをスケジューリングするためのメカニズムをいくつか提供しています。

### `queueMicrotask()`

`queueMicrotask()` は、マイクロタスクをスケジュールするために使用されます。マイクロタスクは、現在実行中のスクリプトの実行後に、他の I/O イベントやタイマーよりも先に実行される軽量タスクです。マイクロタスクには、Promise の解決や、通常のタスクよりも優先されるその他の非同期操作などが含まれます。

```js
queueMicrotask(() => {
  console.log('Microtask is executed');
});

console.log('Synchronous task is executed');
```

上記の例では、「Microtask が実行されました」というログは、「同期タスクが実行されました」の後、タイマーなどの I/O 操作の前に記録されます。

### `process.nextTick()`

`process.nextTick()` は、現在の操作が完了した直後にコールバックを実行するようにスケジュールするために使用されます。これは、コールバックをできるだけ早く実行したいが、現在の実行コンテキストの後に実行したい場合に便利です。

```js
process.nextTick(() => {
  console.log('Next tick callback');
});

console.log('Synchronous task executed');
```

### `setImmediate()`

`setImmediate()` は、Node.js の [イベントループ](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) のチェックフェーズで実行されるコールバックをスケジュールします。チェックフェーズは、ほとんどの I/O コールバックが処理されるポーリングフェーズの後に実行されます。

```js
setImmediate(() => {
  console.log('Immediate callback');
});

console.log('Synchronous task executed');
```

### それぞれの用途

- 現在のスクリプトの直後、かつ I/O やタイマーコールバックの前に実行する必要があるタスク（通常は Promise の解決）には、`queueMicrotask()` を使用します。
- I/O イベントの前に実行する必要があるタスクには、`process.nextTick()` を使用します。これは、操作の遅延やエラーの同期処理に便利です。
- ほとんどの I/O コールバックが処理されたポーリングフェーズの後に実行する必要があるタスクには、`setImmediate()` を使用します。

これらのタスクは現在の同期フローの外で実行されるため、これらのコールバック内でキャッチされていない例外は、周囲の `try/catch` ブロックでキャッチされず、適切に管理されていない場合（例：Promise に `.catch()` をアタッチしたり、`process.on('uncaughtException')` のようなグローバルエラーハンドラーを使用したり）アプリケーションがクラッシュする可能性があります。

イベント ループとさまざまなフェーズの実行順序の詳細については、関連記事 [Node.js イベント ループ](/learn/asynchronous-work/event-loop-timers-and-nexttick) を参照してください。
