---
title: How to use Streams
layout: learn
authors: mcollina, ceres6, simoneb, codyzu
---

# ストリームの使い方

Node.js アプリケーションで大量のデータを扱うことは、諸刃の剣です。膨大なデータ量に対応できることは非常に便利ですが、パフォーマンスのボトルネックやメモリ枯渇につながる可能性もあります。従来、開発者はこの課題に対処するために、データセット全体を一度にメモリに読み込むという手法を採用していました。このアプローチは、小規模なデータセットでは直感的に操作できますが、大規模なデータ（ファイル、ネットワークリクエストなど）では非効率的で、多くのリソースを消費します。

ここで Node.js ストリームの出番です。ストリームは根本的に異なるアプローチを提供し、データを段階的に処理し、メモリ使用量を最適化します。扱いやすいチャンク単位でデータを処理することで、ストリームは、最も困難なデータセットにも効率的に対応できるスケーラブルなアプリケーションを構築できます。よく言われるように、「ストリームは時間軸上の配列である」のです。

このガイドでは、ストリームの概念、歴史、API の概要、そして使用方法と操作方法に関する推奨事項について説明します。

## Node.js ストリームとは？

Node.js ストリームは、アプリケーション内のデータフローを管理するための強力な抽象化を提供します。パフォーマンスを犠牲にすることなく、ファイルの読み取りや書き込み、ネットワークリクエストなど、大規模なデータセットの処理に優れています。

このアプローチは、データセット全体を一度にメモリにロードするのとは異なります。ストリームはデータをチャンク単位で処理するため、メモリ使用量を大幅に削減します。Node.js のすべてのストリームは [`EventEmitter`][] クラスを継承しており、データ処理のさまざまな段階でイベントを発行できます。これらのストリームは読み取り可能、書き込み可能、​​またはその両方にすることができ、さまざまなデータ処理シナリオに柔軟に対応できます。

### イベント駆動型アーキテクチャ

Node.js はイベント駆動型アーキテクチャを採用しており、リアルタイム I/O に最適です。これは、入力が利用可能になるとすぐに消費し、アプリケーションが出力を生成するとすぐに送信することを意味します。ストリームはこのアプローチとシームレスに統合され、継続的なデータ処理を可能にします。

ストリームは、重要な段階でイベントを発行することでこれを実現します。これらのイベントには、受信データのシグナル ([`data`][] イベント) とストリームの完了シグナル ([`end`][] イベント) が含まれます。開発者はこれらのイベントをリッスンし、それに応じてカスタムロジックを実行できます。このイベント駆動型の性質により、ストリームは外部ソースからのデータ処理において非常に効率的です。

## ストリームを使用する理由

ストリームは、他のデータ処理方法に比べて3つの重要な利点があります。

- **メモリ効率**: ストリームは、データセット全体をメモリにロードするのではなく、データをチャンク単位で消費・処理することで、データを段階的に処理します。これは、大規模なデータセットを扱う際に大きなメリットとなり、メモリ使用量を大幅に削減し、メモリ関連のパフォーマンス問題を防ぎます。
- **応答時間の向上**: ストリームは即時のデータ処理を可能にします。データのチャンクが到着すると、ペイロード全体またはデータセット全体の受信を待たずに処理できます。これにより、レイテンシが削減され、アプリケーション全体の応答性が向上します。
- **リアルタイム処理のためのスケーラビリティ**: データをチャンク単位で処理することで、Node.js ストリームは限られたリソースで大量のデータを効率的に処理できます。このスケーラビリティにより、ストリームは大量のデータをリアルタイムで処理するアプリケーションに最適です。

これらの利点により、ストリームは、特に大規模なデータセットやリアルタイムのデータ処理を行う場合に、高性能でスケーラブルな Node.js アプリケーションを構築するための強力なツールになります。

### パフォーマンスに関する注意

アプリケーションで既にすべてのデータがメモリ上に用意されている場合、ストリームを使用すると不要なオーバーヘッドや複雑さが生じ、アプリケーションの速度が低下する可能性があります。

## ストリームの歴史

このセクションでは、Node.js におけるストリームの歴史について説明します。Node.js バージョン 0.11.5 (2013) より前のバージョンで記述されたコードベースで作業していない限り、古いバージョンのストリーム API に遭遇することはほとんどありませんが、これらの用語は現在も使用されている可能性があります。

### ストリーム 0

ストリームの最初のバージョンは、Node.js と同時にリリースされました。Stream クラスはまだ存在しませんでしたが、さまざまなモジュールがこの概念を採用し、`read`/`write` 関数を実装していました。`util.pump()` 関数は、ストリーム間のデータフローを制御するために使用されていました。

### Streams 1 (クラシック)

2011年の Node.js v0.4.0 のリリースで、Stream クラスと `pipe()` メソッドが導入されました。

### Streams 2

2012年、Node.js v0.10.0 のリリースとともに、Streams 2 が発表されました。このアップデートでは、Readable、Writable、Duplex、Transform などの新しいストリームサブクラスが追加されました。さらに、`readable` イベントも追加されました。後方互換性を維持するため、`data` イベントリスナーを追加するか、`pause()` メソッドまたは `resume()` メソッドを呼び出すことで、ストリームを旧モードに切り替えることができました。

### Streams 3

2013年、Node.js v0.11.5 で Streams 3 がリリースされました。これは、ストリームに `data` イベントハンドラーと `readable` イベントハンドラーの両方が存在するという問題に対処するためのものです。これにより、「現在の」モードと「古い」モードを選択する必要がなくなりました。Streams 3 は、Node.js におけるストリームの最新バージョンです。

## ストリームの種類

### Readable

[`Readable`][] は、データソースを順次読み取るために使用するクラスです。Node.js API における `Readable` ストリームの代表的な例としては、ファイル読み取り時の [`fs.ReadStream` ][]、HTTP リクエスト読み取り時の [`http.IncomingMessage` ][]、標準入力読み取り時の [`process.stdin` ][] などがあります。

#### 主要なメソッドとイベント

Readable ストリームは、データ処理を細かく制御できるいくつかのコアメソッドとイベントによって動作します。

- **[`on('data')`][]**: このイベントは、ストリームからデータが利用可能になるたびにトリガーされます。ストリームは処理可能な限り高速にデータをプッシュするため、非常に高速であり、高スループットのシナリオに適しています。
- **[`on('end')`][]**: ストリームから読み取るデータがなくなったときに発行されます。これは、データ配信の完了を示します。このイベントは、ストリームからすべてのデータが消費された場合にのみ発生します。
- **[`on('readable')`][]**: このイベントは、ストリームから読み取り可能なデータが存在する場合、またはストリームの終端に達した場合にトリガーされます。これにより、必要に応じてより制御されたデータ読み取りが可能になります。
- **[`on('close')`][]**: このイベントは、ストリームとその基盤となるリソースが閉じられたときに発行され、これ以上イベントが発行されないことを示します。
- **[`on('error')`][]**: このイベントは、処理中にエラーが発生したことを通知するために、いつでも発行される可能性があります。このイベントのハンドラーを使用することで、キャッチされない例外を回避できます。

これらのイベントの使用方法のデモについては、次のセクションで説明します。

#### 基本的な読み取り可能ストリーム

動的にデータを生成するシンプルな読み取り可能ストリームの実装例を次に示します。

```js
class MyStream extends Readable {
  #count = 0;
  _read(size) {
    this.push(':-)');
    if (++this.#count === 5) {
      this.push(null);
    }
  }
}

const stream = new MyStream();

stream.on('data', chunk => {
  console.log(chunk.toString());
});
```

このコードでは、`MyStream` クラスは Readable を拡張し、[`_read()`][] メソッドをオーバーライドして文字列 ":-)" を内部バッファにプッシュします。文字列を 5 回プッシュした後、`null` をプッシュすることでストリームの終了を通知します。[`on('data')`][] イベントハンドラは、受信した各チャンクをコンソールに出力します。

#### readable イベントによる高度な制御

データフローをさらに細かく制御するには、readable イベントを使用します。このイベントはより複雑ですが、ストリームからデータを読み取るタイミングを明示的に制御できるため、特定のアプリケーションではパフォーマンスが向上します。

```js
const stream = new MyStream({
  highWaterMark: 1,
});

stream.on('readable', () => {
  console.count('>> readable event');
  let chunk;
  while ((chunk = stream.read()) !== null) {
    console.log(chunk.toString()); // Process the chunk
  }
});
stream.on('end', () => console.log('>> end event'));
```

ここでは、readableイベントを使用して、必要に応じて手動でストリームからデータを取得します。readableイベントハンドラ内のループは、バッファが一時的に空になったか、ストリームが終了したことを示す「null」を返すまで、ストリームバッファからデータを読み取り続けます。「highWaterMark」を1に設定するとバッファサイズが小さくなり、readableイベントがより頻繁にトリガーされ、データフローをより細かく制御できるようになります。

前のコードでは、次のような出力が得られます。

```bash
>> readable event: 1
:-):-)
:-)
:-)
:-)
>> readable event: 2
>> readable event: 3
>> readable event: 4
>> end event
```

理解を深めてみましょう。`on('readable')` イベントをアタッチすると、最初に `read()` が呼び出されます。これは、`readable` イベントの発行をトリガーする可能性があるためです。このイベントの発行後、`while` ループの最初の反復処理で `read` を呼び出します。そのため、最初の 2 つのスマイリーが 1 行に表示されます。その後、`null` がプッシュされるまで `read` を呼び出し続けます。`read` の呼び出しごとに新しい `readable` イベントの発行がプログラムされますが、「flow」モード（つまり `readable` イベントを使用）であるため、イベントの発行は `nextTick` にスケジュールされます。そのため、ループの同期コードが終了した時点で、すべてのイベントが最後に表示されます。

注: `NODE_DEBUG=stream` を指定してコードを実行すると、各 `push` の後に `emitReadable` がトリガーされていることを確認できます。

各スマイリーの前に呼び出される読み取り可能なイベントを確認したい場合は、次のように `push` を `setImmediate` または `process.nextTick` にラップできます。

```js
class MyStream extends Readable {
  #count = 0;
  _read(size) {
    setImmediate(() => {
      this.push(':-)');
      if (++this.#count === 5) {
        return this.push(null);
      }
    });
  }
}
```

And we’ll get:

```bash
>> readable event: 1
:-)
>> readable event: 2
:-)
>> readable event: 3
:-)
>> readable event: 4
:-)
>> readable event: 5
:-)
>> readable event: 6
>> end event
```

### Writable

[`Writable`][] ストリームは、ファイルの作成、データのアップロード、またはデータの連続出力を伴うあらゆるタスクに役立ちます。読み取り可能ストリームがデータのソースを提供するのに対し、Node.js の書き込み可能ストリームはデータの宛先として機能します。Node.js API における書き込み可能ストリームの代表的な例としては、[`fs.WriteStream` ][]、[`process.stdout` ][]、[`process.stderr` ][] などがあります。

#### 書き込み可能ストリームの主要メソッドとイベント

- **[`.write()`][]**: このメソッドは、ストリームにデータチャンクを書き込むために使用されます。このメソッドは、定義された制限 (highWaterMark) までデータをバッファリングし、さらにデータをすぐに書き込むことができるかどうかを示すブール値を返します。
- **[`.end()`][]**: このメソッドは、データ書き込みプロセスの終了を通知します。ストリームに書き込み操作を完了し、必要に応じてクリーンアップを実行するように通知します。

#### 書き込み可能なストリームの作成

以下は、すべての入力データを大文字に変換してから標準出力に書き込む書き込み可能なストリームを作成する例です。

```cjs
const { once } = require('node:events');
const { Writable } = require('node:stream');

class MyStream extends Writable {
  constructor() {
    super({ highWaterMark: 10 /* 10 bytes */ });
  }
  _write(data, encode, cb) {
    process.stdout.write(data.toString().toUpperCase() + '\n', cb);
  }
}

async function main() {
  const stream = new MyStream();

  for (let i = 0; i < 10; i++) {
    const waitDrain = !stream.write('hello');

    if (waitDrain) {
      console.log('>> wait drain');
      await once(stream, 'drain');
    }
  }

  stream.end('world');
}

// Call the async function
main().catch(console.error);
```

```mjs
import { once } from 'node:events';
import { Writable } from 'node:stream';

class MyStream extends Writable {
  constructor() {
    super({ highWaterMark: 10 /* 10 bytes */ });
  }
  _write(data, encode, cb) {
    process.stdout.write(data.toString().toUpperCase() + '\n', cb);
  }
}
const stream = new MyStream();

for (let i = 0; i < 10; i++) {
  const waitDrain = !stream.write('hello');

  if (waitDrain) {
    console.log('>> wait drain');
    await once(stream, 'drain');
  }
}

stream.end('world');
```

このコードでは、`MyStream` はバッファ容量 ([`highWaterMark`][]) が 10 バイトのカスタム [`Writable`][] ストリームです。[`_write`][] メソッドをオーバーライドして、データを書き出す前に大文字に変換します。

ループはストリームに hello を 10 回書き込もうとします。バッファがいっぱいになった場合 (`waitDrain` が `true` になった場合)、ストリームのバッファがオーバーフローしないように、[`drain`][] イベントを待機してから処理を続行します。

出力は次のようになります。

```bash
HELLO
>> wait drain
HELLO
HELLO
>> wait drain
HELLO
HELLO
>> wait drain
HELLO
HELLO
>> wait drain
HELLO
HELLO
>> wait drain
HELLO
WORLD
```

### デュプレックス

[`Duplex`][] ストリームは、読み取り可能インターフェースと書き込み可能インターフェースの両方を実装しています。

#### デュプレックスストリームの主なメソッドとイベント

デュプレックスストリームは、「読み取り可能および書き込み可能ストリーム」で説明されているすべてのメソッドとイベントを実装しています。

デュプレックスストリームの良い例として、`net` モジュールの `Socket` クラスが挙げられます。

```cjs
const net = require('node:net');

// Create a TCP server
const server = net.createServer(socket => {
  socket.write('Hello from server!\n');

  socket.on('data', data => {
    console.log(`Client says: ${data.toString()}`);
  });

  // Handle client disconnection
  socket.on('end', () => {
    console.log('Client disconnected');
  });
});

// Start the server on port 8080
server.listen(8080, () => {
  console.log('Server listening on port 8080');
});
```

```mjs
import net from 'node:net';

// Create a TCP server
const server = net.createServer(socket => {
  socket.write('Hello from server!\n');

  socket.on('data', data => {
    console.log(`Client says: ${data.toString()}`);
  });

  // Handle client disconnection
  socket.on('end', () => {
    console.log('Client disconnected');
  });
});

// Start the server on port 8080
server.listen(8080, () => {
  console.log('Server listening on port 8080');
});
```

前のコードは、ポート 8080 で TCP ソケットを開き、接続中のクライアントに `Hello from server!` を送信し、受信したデータをログに記録します。

```cjs
const net = require('node:net');

// Connect to the server at localhost:8080
const client = net.createConnection({ port: 8080 }, () => {
  client.write('Hello from client!\n');
});

client.on('data', data => {
  console.log(`Server says: ${data.toString()}`);
});

// Handle the server closing the connection
client.on('end', () => {
  console.log('Disconnected from server');
});
```

```mjs
import net from 'node:net';

// Connect to the server at localhost:8080
const client = net.createConnection({ port: 8080 }, () => {
  client.write('Hello from client!\n');
});

client.on('data', data => {
  console.log(`Server says: ${data.toString()}`);
});

// Handle the server closing the connection
client.on('end', () => {
  console.log('Disconnected from server');
});
```

上記のコードは、TCP ソケットに接続し、`Hello from client` メッセージを送信し、受信したデータをログに記録します。

### 変換

[`Transform`][] ストリームは双方向ストリームであり、出力は入力に基づいて計算されます。名前が示すように、通常は読み取り可能なストリームと書き込み可能なストリームの間で使用され、通過するデータを変換します。

#### 変換ストリームの主なメソッドとイベント

双方向ストリームのすべてのメソッドとイベントに加えて、以下のメソッドとイベントがあります。

- **[`_transform`][]**: この関数は、読み取り可能な部分と書き込み可能な部分の間のデータフローを処理するために内部的に呼び出されます。アプリケーションコードからこれを呼び出してはなりません。

#### 変換ストリームの作成

新しい変換ストリームを作成するには、`Transform` コンストラクターに `options` オブジェクトを渡します。このオブジェクトには、`push` メソッドを使用して入力データから出力データを計算する `transform` 関数が含まれています。

```cjs
const { Transform } = require('node:stream');

const upper = new Transform({
  transform(data, enc, cb) {
    this.push(data.toString().toUpperCase());
    cb();
  },
});
```

```mjs
import { Transform } from 'node:stream';

const upper = new Transform({
  transform(data, enc, cb) {
    this.push(data.toString().toUpperCase());
    cb();
  },
});
```

このストリームは、あらゆる入力を受け取り、それを大文字で出力します。

## ストリームの操作方法

ストリームを操作する場合、通常はソースからデータを読み取り、出力先にデータを書き込むことになりますが、その際にデータの変換が必要になる場合もあります。以下のセクションでは、そのためのさまざまな方法について説明します。

### `.pipe()`

[`.pipe()`][] メソッドは、読み取り可能なストリームを書き込み可能（または変換可能）なストリームに連結します。これは目的を達成する簡単な方法のように見えますが、すべてのエラー処理をプログラマに委譲するため、正しく実行するのが困難です。

次の例は、パイプが現在のファイルを大文字でコンソールに出力しようとしている様子を示しています。

```cjs
const fs = require('node:fs');
const { Transform } = require('node:stream');

let errorCount = 0;
const upper = new Transform({
  transform(data, enc, cb) {
    if (errorCount === 10) {
      return cb(new Error('BOOM!'));
    }
    errorCount++;
    this.push(data.toString().toUpperCase());
    cb();
  },
});

const readStream = fs.createReadStream(__filename, { highWaterMark: 1 });
const writeStream = process.stdout;

readStream.pipe(upper).pipe(writeStream);

readStream.on('close', () => {
  console.log('Readable stream closed');
});

upper.on('close', () => {
  console.log('Transform stream closed');
});

upper.on('error', err => {
  console.error('\nError in transform stream:', err.message);
});

writeStream.on('close', () => {
  console.log('Writable stream closed');
});
```

```mjs
import fs from 'node:fs';
import { Transform } from 'node:stream';

let errorCount = 0;
const upper = new Transform({
  transform(data, enc, cb) {
    if (errorCount === 10) {
      return cb(new Error('BOOM!'));
    }
    errorCount++;
    this.push(data.toString().toUpperCase());
    cb();
  },
});

const readStream = fs.createReadStream(import.meta.filename, {
  highWaterMark: 1,
});
const writeStream = process.stdout;

readStream.pipe(upper).pipe(writeStream);

readStream.on('close', () => {
  console.log('Readable stream closed');
});

upper.on('close', () => {
  console.log('Transform stream closed');
});

upper.on('error', err => {
  console.error('\nError in transform stream:', err.message);
});

writeStream.on('close', () => {
  console.log('Writable stream closed');
});
```

10文字を書き込んだ後、`upper` はコールバックでエラーを返し、ストリームを終了します。しかし、他のストリームには通知されないため、メモリリークが発生します。出力は次のようになります。

```bash
CONST FS =
Error in transform stream: BOOM!
Transform stream closed
```

### `pipeline()`

`.pipe()` メソッドの落とし穴や低レベルの複雑さを回避するには、ほとんどの場合、[`pipeline()`][] メソッドの使用をお勧めします。このメソッドは、エラー処理とクリーンアップ処理を自動的に実行することで、ストリームをパイプするより安全で堅牢な方法です。

次の例は、`pipeline()` を使用することで、前の例の落とし穴を回避する方法を示しています。

```cjs
const fs = require('node:fs');
const { Transform, pipeline } = require('node:stream');

let errorCount = 0;
const upper = new Transform({
  transform(data, enc, cb) {
    if (errorCount === 10) {
      return cb(new Error('BOOM!'));
    }
    errorCount++;
    this.push(data.toString().toUpperCase());
    cb();
  },
});

const readStream = fs.createReadStream(__filename, { highWaterMark: 1 });
const writeStream = process.stdout;

readStream.on('close', () => {
  console.log('Readable stream closed');
});

upper.on('close', () => {
  console.log('\nTransform stream closed');
});

writeStream.on('close', () => {
  console.log('Writable stream closed');
});

pipeline(readStream, upper, writeStream, err => {
  if (err) {
    return console.error('Pipeline error:', err.message);
  }
  console.log('Pipeline succeeded');
});
```

```mjs
import fs from 'node:fs';
import { Transform, pipeline } from 'node:stream';

let errorCount = 0;
const upper = new Transform({
  transform(data, enc, cb) {
    if (errorCount === 10) {
      return cb(new Error('BOOM!'));
    }
    errorCount++;
    this.push(data.toString().toUpperCase());
    cb();
  },
});

const readStream = fs.createReadStream(import.meta.filename, {
  highWaterMark: 1,
});
const writeStream = process.stdout;

readStream.on('close', () => {
  console.log('Readable stream closed');
});

upper.on('close', () => {
  console.log('\nTransform stream closed');
});

writeStream.on('close', () => {
  console.log('Writable stream closed');
});

pipeline(readStream, upper, writeStream, err => {
  if (err) {
    return console.error('Pipeline error:', err.message);
  }
  console.log('Pipeline succeeded');
});
```

この場合、すべてのストリームは次の出力で閉じられます。

```bash
CONST FS =
Transform stream closed
Writable stream closed
Pipeline error: BOOM!
Readable stream closed
```

[`pipeline()`][] メソッドには [`async pipeline()`][] バージョンもあり、こちらはコールバックを受け取らず、パイプラインが失敗した場合に拒否される Promise を返します。

### 非同期イテレータ

非同期イテレータは、Streams API とのインターフェースの標準的な方法として推奨されています。Web と Node.js の両方におけるすべてのストリームプリミティブと比較して、非同期イテレータは理解しやすく使いやすく、バグの減少とコードの保守性向上に貢献します。Node.js の最新バージョンでは、非同期イテレータはストリームを操作するためのよりエレガントで読みやすい方法として登場しました。イベントを基盤として構築された非同期イテレータは、ストリームの利用を簡素化する高レベルの抽象化を提供します。

Node.js では、すべての読み取り可能なストリームは非同期イテレータです。つまり、`for await...of` 構文を使用して、ストリームのデータが利用可能になるとループ処理を行い、各データを非同期コードの効率性と簡潔性で処理できます。

#### ストリームで非同期イテレータを使用するメリット

ストリームで非同期イテレータを使用すると、非同期データフローの処理がいくつかの点で簡素化されます。

- **可読性の向上**: コード構造がよりクリーンで読みやすくなります。特に複数の非同期データソースを処理する場合に有効です。
- **エラー処理**: 非同期イテレータでは、通常の非同期関数と同様に、try/catch ブロックを使用して簡単にエラーを処理できます。
- **フロー制御**: コンシューマーが次のデータを待機することでフローを制御するため、非同期イテレータは本質的にバックプレッシャーを管理し、メモリの使用と処理をより効率的に行うことができます。

非同期イテレータは、特に非同期データソースを処理する場合や、よりシーケンシャルなループベースのデータ処理を好む場合に、読み取り可能なストリームを操作するための、より現代的で読みやすい方法を提供します。

読み取り可能なストリームで非同期イテレータを使用する例を以下に示します。

```cjs
const fs = require('node:fs');
const { pipeline } = require('node:stream/promises');

async function main() {
  await pipeline(
    fs.createReadStream(__filename),
    async function* (source) {
      for await (let chunk of source) {
        yield chunk.toString().toUpperCase();
      }
    },
    process.stdout
  );
}

main().catch(console.error);
```

```mjs
import fs from 'fs';
import { pipeline } from 'stream/promises';

await pipeline(
  fs.createReadStream(import.meta.filename),
  async function* (source) {
    for await (const chunk of source) {
      yield chunk.toString().toUpperCase();
    }
  },
  process.stdout
);
```

このコードは、新しい変換ストリームを定義することなく、前の例と同じ結果を実現します。簡潔にするために、前の例のエラーは削除されています。パイプラインの非同期バージョンが使用されており、発生する可能性のあるエラーを処理するために `try...catch` ブロックで囲む必要があります。

### オブジェクトモード

デフォルトでは、ストリームは文字列、[`Buffer`][]、[`TypedArray`][]、または [`DataView`][] を処理できます。これらとは異なる任意の値（オブジェクトなど）がストリームにプッシュされると、`TypeError` がスローされます。ただし、`objectMode` オプションを `true` に設定することで、オブジェクトを処理できるようになります。これにより、ストリームは、ストリームの終了を示すために使用される `null` を除く、任意の JavaScript 値を処理できるようになります。つまり、読み取り可能なストリームでは任意の値を `push` および `read` でき、書き込み可能なストリームでは任意の値を `write` できます。

```cjs
const { Readable } = require('node:stream');

const readable = Readable({
  objectMode: true,
  read() {
    this.push({ hello: 'world' });
    this.push(null);
  },
});
```

```mjs
import { Readable } from 'node:stream';

const readable = Readable({
  objectMode: true,
  read() {
    this.push({ hello: 'world' });
    this.push(null);
  },
});
```

オブジェクトモードで作業する場合、`highWaterMark` オプションはバイト数ではなくオブジェクト数を指すことに注意することが重要です。

### バックプレッシャー

ストリームを使用する場合、プロデューサーがコンシューマーに過負荷をかけないようにすることが重要です。そのため、Node.js API のすべてのストリームでバックプレッシャーメカニズムが使用されており、実装者はその動作を維持する責任があります。

データバッファが [`highWaterMark`][] を超えた場合、または書き込みキューが現在ビジー状態の場合、[`.write()`][] は `false` を返します。

`false` 値が返されると、バックプレッシャーシステムが起動します。バックプレッシャーシステムは、受信する [`Readable`][] ストリームへのデータ送信を一時停止し、コンシューマーが再び準備ができるまで待機します。データバッファが空になると、[`'drain'`][] イベントが発行され、受信データフローが再開されます。

バックプレッシャーをより深く理解するには、[`バックプレッシャーガイド`][] をご覧ください。

## ストリームと Web ストリーム

ストリームの概念は Node.js に限ったものではありません。実際、Node.js には [`Web Streams`][] と呼ばれるストリーム概念の別の実装があり、これは [`WHATWG Streams Standard`][] を実装しています。これらの背後にある概念は似ていますが、API が異なり、直接的な互換性がないことに留意することが重要です。

[`Web Streams`][] は [`ReadableStream`][]、[`WritableStream`][]、[`TransformStream`][] クラスを実装しており、これらは Node.js の [`Readable`][]、[`Writable`][]、[`Transform`][] ストリームと相同です。

### ストリームと Web ストリームの相互運用性

Node.js は、Web ストリームと Node.js ストリーム間の変換を行うユーティリティ関数を提供しています。これらの関数は、各ストリームクラスの `toWeb` メソッドと `fromWeb` メソッドとして実装されています。

[`Duplex`][] クラスの次の例は、読み取り可能ストリームと書き込み可能ストリームの両方を Web ストリームに変換して操作する方法を示しています。

```cjs
const { Duplex } = require('node:stream');

const duplex = Duplex({
  read() {
    this.push('world');
    this.push(null);
  },
  write(chunk, encoding, callback) {
    console.log('writable', chunk);
    callback();
  },
});

const { readable, writable } = Duplex.toWeb(duplex);
writable.getWriter().write('hello');

readable
  .getReader()
  .read()
  .then(result => {
    console.log('readable', result.value);
  });
```

```mjs
import { Duplex } from 'node:stream';

const duplex = Duplex({
  read() {
    this.push('world');
    this.push(null);
  },
  write(chunk, encoding, callback) {
    console.log('writable', chunk);
    callback();
  },
});

const { readable, writable } = Duplex.toWeb(duplex);
writable.getWriter().write('hello');

readable
  .getReader()
  .read()
  .then(result => {
    console.log('readable', result.value);
  });
```

ヘルパー関数は、Node.jsモジュールからWebストリームを返す必要がある場合、あるいはその逆の場合に役立ちます。ストリームを定期的に使用する場合は、非同期イテレータを使用することで、Node.jsとWebストリームの両方とシームレスに連携できます。

```cjs
const { pipeline } = require('node:stream/promises');

async function main() {
  const { body } = await fetch('https://nodejs.org/api/stream.html');

  await pipeline(
    body,
    new TextDecoderStream(),
    async function* (source) {
      for await (const chunk of source) {
        yield chunk.toString().toUpperCase();
      }
    },
    process.stdout
  );
}

main().catch(console.error);
```

```mjs
import { pipeline } from 'node:stream/promises';

const { body } = await fetch('https://nodejs.org/api/stream.html');

await pipeline(
  body,
  new TextDecoderStream(),
  async function* (source) {
    for await (const chunk of source) {
      yield chunk.toString().toUpperCase();
    }
  },
  process.stdout
);
```

フェッチ本体は `ReadableStream<Uint8Array>` であるため、チャンクを文字列として扱うには [`TextDecoderStream`][] が必要であることに注意してください。

この作業は、[Matteo Collina][] が [Platformatic's Blog][] で公開したコンテンツに基づいています。

[`Stream`]: https://nodejs.org/api/stream.html
[`Buffer`]: https://nodejs.org/api/buffer.html
[`TypedArray`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
[`DataView`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView
[`TextDecoderStream`]: https://developer.mozilla.org/en-US/docs/Web/API/TextDecoderStream
[`EventEmitter`]: https://nodejs.org/api/events.html#class-eventemitter
[`Writable`]: https://nodejs.org/api/stream.html#stream_writable_streams
[`Readable`]: https://nodejs.org/api/stream.html#stream_readable_streams
[`Duplex`]: https://nodejs.org/api/stream.html#stream_duplex_and_transform_streams
[`Transform`]: https://nodejs.org/api/stream.html#stream_duplex_and_transform_streams
[`drain`]: https://nodejs.org/api/stream.html#stream_event_drain
[`on('data')`]: https://nodejs.org/api/stream.html#stream_event_data
[`data`]: https://nodejs.org/api/stream.html#stream_event_data
[`on('end')`]: https://nodejs.org/api/stream.html#event-end
[`end`]: https://nodejs.org/api/stream.html#event-end
[`on('readable')`]: https://nodejs.org/api/stream.html#event-readable
[`on('close')`]: https://nodejs.org/api/stream.html#event-close_1
[`on('error')`]: https://nodejs.org/api/stream.html#event-error_1
[`.read()`]: https://nodejs.org/docs/latest/api/stream.html#stream_readable_read_size
[`_read()`]: https://nodejs.org/api/stream.html#readable_readsize
[`.write()`]: https://nodejs.org/api/stream.html#stream_writable_write_chunk_encoding_callback
[`_write`]: https://nodejs.org/api/stream.html#writable_writechunk-encoding-callback
[`.end()`]: https://nodejs.org/api/stream.html#writableendchunk-encoding-callback
[`'drain'`]: https://nodejs.org/api/stream.html#stream_event_drain
[`_transform`]: https://nodejs.org/api/stream.html#transform_transformchunk-encoding-callback
[`Readable.from()`]: https://nodejs.org/api/stream.html#streamreadablefromiterable-options
[`highWaterMark`]: https://nodejs.org/api/stream.html#stream_buffering
[`.pipe()`]: https://nodejs.org/docs/latest/api/stream.html#stream_readable_pipe_destination_options
[`pipeline()`]: https://nodejs.org/api/stream.html#stream_stream_pipeline_streams_callback
[`async pipeline()`]: https://nodejs.org/api/stream.html#streampipelinesource-transforms-destination-options
[`Web Streams`]: https://nodejs.org/api/webstreams.html
[`ReadableStream`]: https://nodejs.org/api/webstreams.html#class-readablestream
[`WritableStream`]: https://nodejs.org/api/webstreams.html#class-writablestream
[`TransformStream`]: https://nodejs.org/api/webstreams.html#class-transformstream
[`WHATWG Streams Standard`]: https://streams.spec.whatwg.org/
[`backpressure guide`]: /learn/modules/backpressuring-in-streams
[`fs.readStream`]: https://nodejs.org/api/fs.html#class-fsreadstream
[`http.IncomingMessage`]: https://nodejs.org/api/http.html#class-httpincomingmessage
[`process.stdin`]: https://nodejs.org/api/process.html#processstdin
[`fs.WriteStream`]: https://nodejs.org/api/fs.html#class-fswritestream
[`process.stdout`]: https://nodejs.org/api/process.html#processstdout
[`process.stderr`]: https://nodejs.org/api/process.html#processstderr
[Matteo Collina]: https://github.com/mcollina
[Platformatic's Blog]: https://blog.platformatic.dev/a-guide-to-reading-and-writing-nodejs-streams
