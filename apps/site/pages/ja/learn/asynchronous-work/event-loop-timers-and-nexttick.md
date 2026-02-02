---
title: The Node.js Event Loop
layout: learn
---

# Node.js イベントループ

## イベントループとは？

イベントループは、Node.js がデフォルトで単一の JavaScript スレッドを使用するにもかかわらず、可能な限りシステムカーネルに操作をオフロードすることで、ノンブロッキング I/O 操作を実行できるようにする仕組みです。

最近のカーネルのほとんどはマルチスレッドであるため、バックグラウンドで複数の操作を実行できます。これらの操作の 1 つが完了すると、カーネルは Node.js に適切なコールバックを **poll** キューに追加し、最終的に実行するように指示します。この点については、このトピックの後半で詳しく説明します。

## イベントループの説明

Node.js が起動すると、イベントループが初期化され、指定された入力スクリプトが処理されます（または [REPL][] にドロップされますが、このドキュメントでは説明されていません）。このスクリプトでは、非同期 API 呼び出し、タイマーのスケジュール設定、`process.nextTick()` の呼び出しなどが行われる場合があります。その後、イベントループの処理が開始されます。

次の図は、イベントループの処理順序を簡略化して示しています。

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

> 各ボックスは、イベントループの「フェーズ」と呼ばれます。

各フェーズには、実行するコールバックの FIFO キューがあります。各フェーズはそれぞれ独自の特殊性を持っていますが、一般的には、イベントループが特定のフェーズに入ると、そのフェーズに固有の操作を実行し、その後、キューが空になるかコールバックの実行回数の上限に達するまで、そのフェーズのキュー内のコールバックを実行します。キューが空になるかコールバックの実行回数の上限に達すると、イベントループは次のフェーズに移行し、これを繰り返します。

これらの操作のいずれかがさらに操作をスケジュールする可能性があり、**poll** フェーズで処理される新しいイベントはカーネルによってキューに登録されるため、ポーリングイベントが処理されている間にもポーリングイベントがキューに登録されることがあります。その結果、実行時間の長いコールバックは、ポーリングフェーズをタイマーのしきい値よりもはるかに長く実行させる可能性があります。詳細については、[**timers**](#timers) および [**poll**](#poll) セクションを参照してください。

> WindowsとUnix/Linuxの実装には若干の差異がありますが、今回のデモでは重要ではありません。最も重要な部分はここにあります。実際には7つか8つのステップがありますが、ここで重要なのは、Node.jsが実際に使用する上記のステップです。

## フェーズの概要

- **timers**: このフェーズでは、`setTimeout()` と `setInterval()` によってスケジュールされたコールバックを実行します。
- **pending callbacks**: 次のループ反復まで延期された I/O コールバックを実行します。
- **idle, prepare**: 内部でのみ使用されます。
- **poll**: 新しい I/O イベントを取得し、I/O 関連のコールバックを実行します (close コールバック、タイマーによってスケジュールされたコールバック、および `setImmediate()` を除くほぼすべてのコールバック)。Node は適切な場合にここでブロックします。
- **check**: `setImmediate()` コールバックがここで呼び出されます。
- **close callbacks**: 一部のクローズコールバック (例: `socket.on('close', ...)`)。

イベントループの各実行の間に、Node.js は非同期 I/O またはタイマーを待機しているかどうかを確認し、待機していない場合は正常にシャットダウンします。

libuv 1.45.0 (Node.js 20) 以降、イベントループの動作が変更され、以前のバージョンではタイマーが **poll** フェーズの前後両方で実行されていましたが、現在は **poll** フェーズの後にのみ実行されるようになりました。この変更は、`setImmediate()` コールバックのタイミングや、特定のシナリオにおけるタイマーとの相互作用に影響を与える可能性があります。

## フェーズの詳細

### timers

タイマーは、ユーザーが実行を希望する正確な時間ではなく、指定されたコールバックが実行される可能性のある **しきい値** を指定します。タイマーコールバックは、指定された時間が経過した後、スケジュール可能な限り早く実行されます。ただし、オペレーティングシステムのスケジュール設定や他のコールバックの実行によって遅延される可能性があります。

> 技術的には、[**poll** フェーズ](#poll) がタイマーの実行タイミングを制御します。

例えば、100 ミリ秒のしきい値後にタイムアウトを実行するようにスケジュール設定し、スクリプトがファイルの非同期読み取りを開始して 95 ミリ秒かかるとします。

```js
const fs = require('node:fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);

// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

イベントループが **poll** フェーズに入ると、キューは空です（`fs.readFile()` はまだ完了していません）。そのため、最も早いタイマーのしきい値に達するまで、残りミリ秒数だけ待機します。95 ミリ秒が経過するまで待機している間に、`fs.readFile()` はファイルの読み取りを終了し、完了までに 10 ミリ秒かかるコールバックが **poll** キューに追加され、実行されます。コールバックが終了すると、キューにコールバックがもう存在しないため、イベントループは最も早いタイマーのしきい値に達したことを確認し、**timers** フェーズに戻ってタイマーのコールバックを実行します。この例では、タイマーがスケジュールされてからコールバックが実行されるまでの合計遅延は 105 ミリ秒であることがわかります。

> **poll** フェーズでイベント ループが枯渇するのを防ぐために、[libuv][] (Node.js イベント ループとプラットフォームのすべての非同期動作を実装する C ライブラリ) にも、イベントのポーリングを停止するまでのハード最大値 (システム依存) があります。

### pending callbacks

このフェーズでは、TCPエラーの種類など、いくつかのシステム操作に対するコールバックを実行します。例えば、TCPソケットが接続時に`ECONNREFUSED`を受け取った場合、一部の\*nixシステムではエラーの報告を待機する必要があります。これは **pending callbacks**フェーズで実行されるようにキューに登録されます。

### poll

**poll** フェーズには主に 2 つの機能があります。

1. I/O をブロックしてポーリングする時間を計算する。
2. **poll** キュー内のイベントを処理する。

イベントループが **poll** フェーズに入り、タイマーがスケジュールされていない場合、次の 2 つのうちのいずれかが発生します。

- _**poll** キューが **空でない場合**_、イベントループはコールバックのキューを同期的に繰り返し実行し、キューが空になるかシステム依存のハードリミットに達するまで実行を続けます。

- _**poll** キューが **空の場合**_、さらに次の 2 つのうちのいずれかが発生します。
  - `setImmediate()` によってスクリプトがスケジュールされている場合、イベントループは **poll** フェーズを終了し、**check** フェーズに進んでスケジュールされたスクリプトを実行します。

  - スクリプトが `setImmediate()` によってスケジュールされていない場合、イベントループはコールバックがキューに追加されるまで待機し、追加されるとすぐに実行します。

**poll** キューが空になると、イベントループは _時間しきい値に達した_ タイマーをチェックします。1つ以上のタイマーが準備完了状態の場合、イベントループは **timers** フェーズに戻り、それらのタイマーのコールバックを実行します。

### check

このフェーズでは、イベントループは **poll** フェーズの完了直後にコールバックを実行できます。**poll** フェーズがアイドル状態になり、スクリプトが `setImmediate()` でキューイングされている場合、イベントループは待機せずに **check** フェーズに進むことがあります。

`setImmediate()` は、実際にはイベントループの別のフェーズで実行される特別なタイマーです。これは、**poll** フェーズの完了後に実行されるコールバックをスケジュールする libuv API を使用します。

通常、コードが実行されると、イベントループは最終的に **poll** フェーズに到達し、そこで接続やリクエストなどの受信を待機します。ただし、`setImmediate()` でコールバックがスケジュールされており、**poll** フェーズがアイドル状態になった場合、イベントループは終了し、**poll** イベントを待機せずに **check** フェーズに進みます。

### close callbacks

ソケットまたはハンドルが突然閉じられた場合（例：`socket.destroy()`）、このフェーズで `'close'` イベントが発行されます。それ以外の場合は、`process.nextTick()` を介して発行されます。

## `setImmediate()` vs `setTimeout()`

`setImmediate()` と `setTimeout()` は似ていますが、呼び出されるタイミングによって動作が異なります。

- `setImmediate()` は、現在の **poll** フェーズが完了するとスクリプトを実行するように設計されています。
- `setTimeout()` は、ミリ秒単位の最小しきい値が経過した後にスクリプトを実行するようにスケジュールします。

タイマーの実行順序は、呼び出されるコンテキストによって異なります。両方がメインモジュール内から呼び出された場合、タイミングはプロセスのパフォーマンスによって制限されます（マシン上で実行されている他のアプリケーションの影響を受ける可能性があります）。

例えば、次のスクリプトを I/O サイクル内（つまりメインモジュール）以外で実行した場合、2 つのタイマーの実行順序はプロセスのパフォーマンスによって制限されるため、非決定的になります。

```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```bash
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

ただし、I/O サイクル内で 2 つの呼び出しを移動すると、即時コールバックが常に最初に実行されます。

```js
// timeout_vs_immediate.js
const fs = require('node:fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```bash
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

`setTimeout()` ではなく `setImmediate()` を使用する主な利点は、I/O サイクル内でスケジュールされている場合、存在するタイマーの数に関係なく、`setImmediate()` が常にタイマーの前に実行されることです。

## `process.nextTick()`

### `process.nextTick()` を理解する

`process.nextTick()` は非同期 API の一部であるにもかかわらず、図には表示されていないことにお気づきかもしれません。これは、`process.nextTick()` が厳密にはイベントループの一部ではないためです。代わりに、`nextTickQueue` は、イベントループの現在のフェーズに関係なく、現在の操作が完了した後に処理されます。ここで、_操作_ は、基盤となる C/C++ ハンドラーからの遷移として定義され、実行する必要がある JavaScript を処理します。

図をもう一度見てみると、特定のフェーズで `process.nextTick()` を呼び出すたびに、イベントループが続行される前に `process.nextTick()` に渡されるすべてのコールバックが解決されます。これは、**`process.nextTick()` を再帰的に呼び出すことで I/O を「枯渇」させ**、イベントループが **ポーリング** フェーズに到達できないため、問題が発生する可能性があります。

### なぜそんなことが許されるのでしょうか？

なぜこのようなものがNode.jsに含まれているのでしょうか？その理由の一つは、APIは必ずしも非同期である必要がない場合でも常に非同期であるべきという設計思想にあります。例えば、次のコードスニペットをご覧ください。

```js
function apiCall(arg, callback) {
  if (typeof arg !== 'string') {
    return process.nextTick(
      callback,
      new TypeError('argument should be string')
    );
  }
}
```

このスニペットは引数チェックを行い、正しくない場合はエラーをコールバックに渡します。API は最近更新され、`process.nextTick()` に引数を渡せるようになりました。これにより、コールバック後に渡された引数をコールバックの引数として渡すことができるため、関数をネストする必要がなくなりました。

ここでは、ユーザーにエラーを返しますが、これはユーザーコードの残りの実行を許可した _後_ にのみ行われます。`process.nextTick()` を使用することで、`apiCall()` は常にユーザーコードの残りの実行を許可した _後_ 、かつイベントループの実行が許可される _前_ にコールバックを実行することが保証されます。これを実現するために、JS のコールスタックは展開され、指定されたコールバックがすぐに実行されるため、`process.nextTick()` を再帰呼び出ししても、v8 の `RangeError: Maximum call stack size exceeded` が発生することはありません。

この考え方は、潜在的に問題を引き起こす可能性のある状況につながる可能性があります。
次のスニペットを例に挙げましょう。

```js
let bar = null;

// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall(callback) {
  callback();
}

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {
  // since someAsyncApiCall hasn't completed, bar hasn't been assigned any value
  console.log('bar', bar); // null
});

bar = 1;
```

ユーザーは `someAsyncApiCall()` を非同期シグネチャを持つように定義していますが、実際には同期的に動作します。`someAsyncApiCall()` が呼び出されると、`someAsyncApiCall()` に提供されたコールバックがイベントループの同じフェーズで呼び出されます。これは、`someAsyncApiCall()` 自体が実際には非同期処理を実行していないためです。その結果、コールバックは `bar` を参照しようとしますが、スクリプトが完了まで実行されていないため、その変数がスコープ内に存在しない可能性があります。

コールバックを `process.nextTick()` に配置することで、スクリプトは完了まで実行することができ、コールバックが呼び出される前にすべての変数、関数などを初期化できます。また、イベントループの継続を禁止するという利点もあります。イベントループの継続が許可される前にエラーをユーザーに警告することは有用です。`process.nextTick()` を使用した前述の例を以下に示します。

```js
let bar = null;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

もう一つの現実世界の例を挙げます。

```js
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```

ポートのみが渡された場合、そのポートは即座にバインドされます。そのため、`'listening'` コールバックは即座に呼び出される可能性があります。問題は、その時点で `.on('listening')` コールバックが設定されていないことです。

この問題を回避するために、`'listening'` イベントは `nextTick()` のキューに登録され、スクリプトが完了するまで実行されます。これにより、ユーザーは任意のイベントハンドラーを設定できます。

## `process.nextTick()` vs `setImmediate()`

ユーザーの観点からは似たような呼び出しが2つありますが、名前が紛らわしいです。

- `process.nextTick()` は同じフェーズで即座に実行されます。
- `setImmediate()` はイベントループの次の反復処理、つまり「tick」で実行されます。

本質的には、これらの名前は入れ替えるべきです。`process.nextTick()` は `setImmediate()` よりも即座に実行されますが、これは過去の遺物であり、今後変更される可能性は低いでしょう。この変更を行うと、npm 上のパッケージの大部分が動作しなくなります。毎日新しいモジュールが追加されているため、待つ時間が長ければ長いほど、動作しなくなる可能性が高くなります。混乱を招くかもしれませんが、名前自体は変更されません。

> 開発者の皆様には、理解しやすいため、常に `setImmediate()` を使用することをお勧めします。

## なぜ `process.nextTick()` を使うのでしょうか？

主な理由は2つあります。

1. ユーザーがエラー処理を行えるようにするため、不要なリソースをクリーンアップするため、あるいはイベントループが継続する前にリクエストを再試行できるようにするため。

2. コールスタックが展開された後、イベントループが継続する前にコールバックを実行できるようにする必要がある場合もあります。

一例として、ユーザーの期待に応えることが挙げられます。簡単な例を以下に示します。

```js
const server = net.createServer();
server.on('connection', conn => {});

server.listen(8080);
server.on('listening', () => {});
```

例えば、`listen()` はイベントループの先頭で実行されますが、リスニングコールバックは `setImmediate()` 内に配置されているとします。ホスト名が渡されない限り、ポートへのバインドは即座に行われます。イベントループが続行するには、**poll** フェーズに到達する必要があります。つまり、リスニングイベントの前に接続が受信され、接続イベントが発行される可能性がゼロではないということです。

別の例として、`EventEmitter` を拡張し、コンストラクタ内からイベントを発行する方法があります。

```js
const EventEmitter = require('node:events');

class MyEmitter extends EventEmitter {
  constructor() {
    super();
    this.emit('event');
  }
}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

コンストラクターからすぐにイベントを発行することはできません。スクリプトは、ユーザーがそのイベントにコールバックを割り当てる時点まで処理が進んでいないためです。そのため、コンストラクター内で `process.nextTick()` を使用して、コンストラクターの終了後にイベントを発行するコールバックを設定することで、期待どおりの結果を得ることができます。

```js
const EventEmitter = require('node:events');

class MyEmitter extends EventEmitter {
  constructor() {
    super();

    // use nextTick to emit the event once a handler is assigned
    process.nextTick(() => {
      this.emit('event');
    });
  }
}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

[libuv]: https://libuv.org/
[REPL]: https://nodejs.org/api/repl.html#repl_repl
