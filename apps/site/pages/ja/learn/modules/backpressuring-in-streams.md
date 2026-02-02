---
title: Backpressuring in Streams
layout: learn
---

# ストリームにおけるバックプレッシャ

データ処理中に発生する一般的な問題は [`backpressure`][] と呼ばれ、データ転送中にバッファの背後にデータが蓄積される現象を指します。転送の受信側で複雑な処理が行われている場合、または何らかの理由で処理速度が遅い場合、受信側のデータが詰まりのように蓄積される傾向があります。

この問題を解決するには、あるソースから別のソースへのスムーズなデータフローを確保するための委任システムが必要です。様々なコミュニティがそれぞれのプログラムでこの問題を独自に解決しており、Unix パイプや TCP ソケットはその好例であり、しばしば _フロー制御_ と呼ばれます。Node.js では、ストリームが採用されている解決策です。

このガイドの目的は、バックプレッシャとは何か、そしてストリームが Node.js のソースコードでどのようにこの問題に対処するかを詳しく説明することです。ガイドの後半では、ストリームを実装する際にアプリケーションのコードを安全かつ最適化するための推奨ベストプラクティスを紹介します。

このガイドでは、Node.jsにおける[`backpressure`][]、[`Buffer`][]、[`EventEmitters`][]の一般的な定義と、[`Stream`][]の使用経験についてある程度の知識があることを前提としています。これらのドキュメントをまだ読んでいない場合は、まずAPIドキュメントを参照することをお勧めします。このガイドを読み進める中で、理解を深めるのに役立ちます。

## データ処理の問題

コンピュータシステムでは、データはパイプ、ソケット、シグナルを介してプロセス間で転送されます。Node.jsには、[`Stream`][]と呼ばれる同様のメカニズムがあります。Streamは素晴らしい！Node.jsで非常に多くの機能を提供し、内部コードベースのほぼすべての部分でこのモジュールが使用されています。開発者の皆さんにも、Streamをぜひ活用してください！

```cjs
const readline = require('node:readline');

// process.stdin and process.stdout are both instances of Streams.
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

rl.question('Why should you use streams? ', answer => {
  console.log(`Maybe it's ${answer}, maybe it's because they are awesome! :)`);

  rl.close();
});
```

```mjs
import readline from 'node:readline';

// process.stdin and process.stdout are both instances of Streams.
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

rl.question('Why should you use streams? ', answer => {
  console.log(`Maybe it's ${answer}, maybe it's because they are awesome! :)`);

  rl.close();
});
```

ストリームを通じて実装されたバックプレッシャー機構が優れた最適化である理由を示す良い例は、Node.js の [`Stream`][] 実装の内部システムツールを比較することで確認できます。

あるシナリオでは、大きなファイル（約 9 GB）を、使い慣れた [`zip(1)`][] ツールを使用して圧縮します。

```
zip The.Matrix.1080p.mkv
```

完了するまでに数分かかる場合がありますが、別のシェルで、別の圧縮ツール [`gzip(1)`][] をラップする Node.js のモジュール [`zlib`][] を取得するスクリプトを実行する場合があります。

```cjs
const fs = require('node:fs');
const gzip = require('node:zlib').createGzip();

const inp = fs.createReadStream('The.Matrix.1080p.mkv');
const out = fs.createWriteStream('The.Matrix.1080p.mkv.gz');

inp.pipe(gzip).pipe(out);
```

```mjs
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

const gzip = createGzip();

const inp = createReadStream('The.Matrix.1080p.mkv');
const out = createWriteStream('The.Matrix.1080p.mkv.gz');

inp.pipe(gzip).pipe(out);
```

結果をテストするには、各圧縮ファイルを開いてみてください。[`zip(1)`][] ツールで圧縮されたファイルはファイルが破損していることを通知しますが、[`Stream`][] で圧縮されたファイルはエラーなく解凍されます。

> この例では、`.pipe()` を使用してデータソースを一方から他方へ取得しています。ただし、適切なエラーハンドラがアタッチされていないことに注意してください。データチャンクを適切に受信できなかった場合、`Readable` ソースまたは `gzip` ストリームは破棄されません。[`pump`][] は、パイプライン内のストリームの 1 つが失敗または終了した場合に、すべてのストリームを適切に破棄するユーティリティツールであり、この場合は必須です。

[`pump`][] は Node.js 8.x 以前でのみ必要です。Node.js 10.x 以降では、[`pump`][] の代わりに [`pipeline`][] が導入されています。
これは、ストリーム間のパイプ処理を行い、エラーを転送し、適切にクリーンアップして、パイプラインが完了したらコールバックを返すモジュールメソッドです。

パイプラインの使用例を以下に示します。

```cjs
const fs = require('node:fs');
const { pipeline } = require('node:stream');
const zlib = require('node:zlib');

// Use the pipeline API to easily pipe a series of streams
// together and get notified when the pipeline is fully done.
// A pipeline to gzip a potentially huge video file efficiently:

pipeline(
  fs.createReadStream('The.Matrix.1080p.mkv'),
  zlib.createGzip(),
  fs.createWriteStream('The.Matrix.1080p.mkv.gz'),
  err => {
    if (err) {
      console.error('Pipeline failed', err);
    } else {
      console.log('Pipeline succeeded');
    }
  }
);
```

```mjs
import fs from 'node:fs';
import { pipeline } from 'node:stream';
import zlib from 'node:zlib';

// Use the pipeline API to easily pipe a series of streams
// together and get notified when the pipeline is fully done.
// A pipeline to gzip a potentially huge video file efficiently:

pipeline(
  fs.createReadStream('The.Matrix.1080p.mkv'),
  zlib.createGzip(),
  fs.createWriteStream('The.Matrix.1080p.mkv.gz'),
  err => {
    if (err) {
      console.error('Pipeline failed', err);
    } else {
      console.log('Pipeline succeeded');
    }
  }
);
```

[`stream/promises`][] モジュールを使用して、`async` / `await` でパイプラインを使用することもできます。

```cjs
const fs = require('node:fs');
const { pipeline } = require('node:stream/promises');
const zlib = require('node:zlib');

async function run() {
  try {
    await pipeline(
      fs.createReadStream('The.Matrix.1080p.mkv'),
      zlib.createGzip(),
      fs.createWriteStream('The.Matrix.1080p.mkv.gz')
    );
    console.log('Pipeline succeeded');
  } catch (err) {
    console.error('Pipeline failed', err);
  }
}
```

```mjs
import fs from 'node:fs';
import { pipeline } from 'node:stream/promises';
import zlib from 'node:zlib';

async function run() {
  try {
    await pipeline(
      fs.createReadStream('The.Matrix.1080p.mkv'),
      zlib.createGzip(),
      fs.createWriteStream('The.Matrix.1080p.mkv.gz')
    );
    console.log('Pipeline succeeded');
  } catch (err) {
    console.error('Pipeline failed', err);
  }
}
```

## データが多すぎる、速すぎる

[`Readable`][] ストリームが [`Writable`][] にデータを渡す速度が速すぎる場合があります。これは、コンシューマーが処理できる量をはるかに超えています。

そうなると、コンシューマーは後で処理するためにすべてのデータチャンクをキューに入れ始めます。書き込みキューはどんどん長くなり、処理全体が完了するまで、より多くのデータをメモリに保持する必要があります。

ディスクへの書き込みはディスクからの読み取りよりもはるかに遅いため、ファイルを圧縮してハードディスクに書き込もうとすると、書き込みディスクが読み取り速度に追いつかず、バックプレッシャーが発生します。

```js
// Secretly the stream is saying: "whoa, whoa! hang on, this is way too much!"
// Data will begin to build up on the read side of the data buffer as
// `write` tries to keep up with the incoming data flow.
inp.pipe(gzip).pipe(outputFile);
```

これがバックプレッシャー機構が重要な理由です。バックプレッシャー機構がなければ、プロセスはシステムのメモリを大量に消費し、他のプロセスの速度を低下させ、完了するまでシステムの大部分を占有することになります。

その結果、いくつかの問題が発生します。

- 他のすべての実行中のプロセスの速度低下
- ガベージコレクタの過負荷
- メモリ枯渇

以下の例では、`.write()` 関数の [return value][] を削除し、`true` に変更します。これにより、Node.js コアにおけるバックプレッシャーのサポートが実質的に無効になります。「変更された」バイナリとは、`return ret;` 行を削除し、代わりに `return true;` 行を追加して `node` バイナリを実行することを意味します。

## ガベージコレクションへの過剰な負荷

簡単なベンチマークを見てみましょう。上記と同じ例を用いて、両方のバイナリの中央値を取得するために、いくつかのタイムトライアルを実行しました。

```
   trial (#)  | `node` binary (ms) | modified `node` binary (ms)
=================================================================
      1       |      56924         |           55011
      2       |      52686         |           55869
      3       |      59479         |           54043
      4       |      54473         |           55229
      5       |      52933         |           59723
=================================================================
average time: |      55299         |           55975
```

どちらも実行に約1分かかるので、大きな違いはありませんが、私たちの推測が正しいかどうかを確認するために、詳しく見てみましょう。Linuxツール[`dtrace`][]を使用して、V8ガベージコレクターで何が起こっているかを評価します。

GC（ガベージコレクター）の測定時間は、ガベージコレクターが実行する単一のスイープの完全なサイクルの間隔を示しています。

```
approx. time (ms) | GC (ms) | modified GC (ms)
=================================================
          0       |    0    |      0
          1       |    0    |      0
         40       |    0    |      2
        170       |    3    |      1
        300       |    3    |      1

         *             *           *
         *             *           *
         *             *           *

      39000       |    6    |     26
      42000       |    6    |     21
      47000       |    5    |     32
      50000       |    8    |     28
      54000       |    6    |     35
```

2つのプロセスは開始時点では同じで、GCも同じ速度で動作しているように見えますが、バックプレッシャーシステムが適切に機能していれば、数秒後にはデータ転送が終了するまでGCの負荷が4～8ミリ秒の一定間隔に分散されることがわかります。

しかし、バックプレッシャーシステムが機能していない場合、V8ガベージコレクションは遅延し始めます。GCと呼ばれる通常のバイナリは1分間に約**75**回実行されるのに対し、修正されたバイナリは**36**回しか実行されません。

これは、メモリ使用量の増加によってゆっくりと徐々に蓄積される負債です。バックプレッシャーシステムが機能していない状態でデータが転送されると、チャンク転送ごとにメモリ使用量が増加します。

割り当てられるメモリ量が多いほど、GCが1回のスイープで処理しなければならないメモリ量も増えます。スイープが大きいほど、GCが解放できる領域を決定する必要があり、より広いメモリ空間でデタッチされたポインタをスキャンするために、より多くの計算能力が消費されます。

## メモリ消費量

各バイナリのメモリ消費量を調べるために、各プロセスごとに `/usr/bin/time -lp sudo ./node ./backpressure-example/zlib.js` を実行してメモリ消費量を計測しました。

通常のバイナリの出力は次のとおりです。

```
Respecting the return value of .write()
=============================================
real        58.88
user        56.79
sys          8.79
  87810048  maximum resident set size
         0  average shared memory size
         0  average unshared data size
         0  average unshared stack size
     19427  page reclaims
      3134  page faults
         0  swaps
         5  block input operations
       194  block output operations
         0  messages sent
         0  messages received
         1  signals received
        12  voluntary context switches
    666037  involuntary context switches
```

仮想メモリが占​​有する最大バイトサイズは約87.81MBです。

[`.write()`][]関数の[戻り値][]を変更すると、次のようになります。

```
Without respecting the return value of .write():
==================================================
real        54.48
user        53.15
sys          7.43
1524965376  maximum resident set size
         0  average shared memory size
         0  average unshared data size
         0  average unshared stack size
    373617  page reclaims
      3139  page faults
         0  swaps
        18  block input operations
       199  block output operations
         0  messages sent
         0  messages received
         1  signals received
        25  voluntary context switches
    629566  involuntary context switches
```

仮想メモリが占​​有する最大バイトサイズは約1.52GBです。

バックプレッシャーを委譲するストリームがない場合、割り当てられるメモリ空間は桁違いに大きくなります。同じプロセスでも、この差は歴然としています。

この実験は、Node.jsのバックプレッシャー機構がコンピューティングシステムにとっていかに最適化され、費用対効果が高いかを示しています。それでは、その仕組みを詳しく見ていきましょう。

## バックプレッシャーはどのようにこれらの問題を解決するのでしょうか？

あるプロセスから別のプロセスにデータを転送する関数はいくつかあります。Node.js には [`.pipe()`][] という組み込み関数があります。他にも [パッケージ][] を利用できます。しかし、最終的には、このプロセスの基本レベルでは、データの _ソース_ と _コンシューマー_ という 2 つの別々のコンポーネントが存在します。

ソースから [`.pipe()`][] が呼び出されると、転送するデータがあることがコンシューマーに通知されます。pipe 関数は、イベントトリガーに対して適切なバックプレッシャークロージャを設定するのに役立ちます。

Node.js では、ソースは [`Readable`][] ストリーム、コンシューマーは [`Writable`][] ストリームです（どちらも [`Duplex`][] または [`Transform`][] ストリームと置き換えることができますが、このガイドでは扱いません）。

バックプレッシャーがトリガーされるタイミングは、[`Writable`][] の [`.write()`][] 関数の戻り値に正確に限定できます。もちろん、この戻り値はいくつかの条件によって決まります。

データバッファが [`highWaterMark`][] を超えている場合、または書き込みキューが現在ビジー状態の場合、[`.write()`][] は `false` を返します。

`false` 値が返されると、バックプレッシャーシステムが起動します。バックプレッシャーシステムは、受信中の [`Readable`][] ストリームへのデータ送信を一時停止し、コンシューマー側の準備が整うまで待機します。データバッファが空になると、[`'drain'`][] イベントが発行され、受信データフローが再開されます。

キューの処理が完了すると、バックプレッシャーによってデータの送信が再開されます。
使用されていたメモリ領域は解放され、次のデータバッチの準備が整います。

これにより、[`.pipe()`][] 関数は、常に一定量のメモリを使用できるようになります。メモリリークや無限バッファリングは発生せず、ガベージコレクターはメモリ内の1つの領域のみを処理することになります。

では、バックプレッシャーがそれほど重要なのに、なぜ（おそらく）聞いたことがないのでしょうか？
答えは簡単です。Node.js はこれらすべてを自動的に処理してくれるからです。

これは素晴らしいですね！しかし、カスタムストリームの実装方法を理解しようとすると、あまり良いことではありません。

> ほとんどのマシンでは、バッファがいっぱいになったかどうかを判断するバイトサイズがあります（これはマシンによって異なります）。Node.js では [`highWaterMark`][] を独自に設定できますが、一般的にはデフォルトは 16KB（16384、または objectMode ストリームの場合は 16）に設定されています。この値を上げたい場合は、設定しても構いませんが、注意が必要です。

## `.pipe()` のライフサイクル

バックプレッシャーをより深く理解するために、[`Readable`][] ストリームが [`Writable`][] ストリームに [pipe][] されるまでのライフサイクルのフローチャートを以下に示します。

```
                                                     +===================+
                         x-->  Piping functions   +-->   src.pipe(dest)  |
                         x     are set up during     |===================|
                         x     the .pipe method.     |  Event callbacks  |
  +===============+      x                           |-------------------|
  |   Your Data   |      x     They exist outside    | .on('close', cb)  |
  +=======+=======+      x     the data flow, but    | .on('data', cb)   |
          |              x     importantly attach    | .on('drain', cb)  |
          |              x     events, and their     | .on('unpipe', cb) |
+---------v---------+    x     respective callbacks. | .on('error', cb)  |
|  Readable Stream  +----+                           | .on('finish', cb) |
+-^-------^-------^-+    |                           | .on('end', cb)    |
  ^       |       ^      |                           +-------------------+
  |       |       |      |
  |       ^       |      |
  ^       ^       ^      |    +-------------------+         +=================+
  ^       |       ^      +---->  Writable Stream  +--------->  .write(chunk)  |
  |       |       |           +-------------------+         +=======+=========+
  |       |       |                                                 |
  |       ^       |                              +------------------v---------+
  ^       |       +-> if (!chunk)                |    Is this chunk too big?  |
  ^       |       |     emit .end();             |    Is the queue busy?      |
  |       |       +-> else                       +-------+----------------+---+
  |       ^       |     emit .write();                   |                |
  |       ^       ^                                   +--v---+        +---v---+
  |       |       ^-----------------------------------<  No  |        |  Yes  |
  ^       |                                           +------+        +---v---+
  ^       |                                                               |
  |       ^               emit .pause();          +=================+     |
  |       ^---------------^-----------------------+  return false;  <-----+---+
  |                                               +=================+         |
  |                                                                           |
  ^            when queue is empty     +============+                         |
  ^------------^-----------------------<  Buffering |                         |
               |                       |============|                         |
               +> emit .drain();       |  ^Buffer^  |                         |
               +> emit .resume();      +------------+                         |
                                       |  ^Buffer^  |                         |
                                       +------------+   add chunk to queue    |
                                       |            <---^---------------------<
                                       +============+
```

> 複数のストリームを連結してデータを操作するパイプラインを設定する場合、おそらく [`Transform`][] ストリームを実装することになるでしょう。

この場合、[`Readable`][] ストリームからの出力は [`Transform`][] に入り、[`Writable`][] にパイプされます。

```js
Readable.pipe(Transformable).pipe(Writable);
```

バックプレッシャーは自動的に適用されますが、[`Transform`][] ストリームの受信および送信の `highWaterMark` は両方とも操作可能であり、バックプレッシャーシステムに影響を与えることに注意してください。

## バックプレッシャーのガイドライン

[Node.js v0.10][] 以降、[`Stream`][] クラスは、[`.read()`][] または [`.write()`][] のアンダースコアバージョン ([`._read()`][] および [`._write()`][]) を使用して、これらの関数の動作を変更できるようになりました。

[読み取り可能ストリームの実装][] および [書き込み可能ストリームの実装][] に関するガイドラインが文書化されています。これらのガイドラインは既にお読みいただいているものと仮定し、次のセクションではより詳細に説明します。

## カスタムストリームを実装する際の遵守すべきルール

ストリームの黄金律は、**常にバックプレッシャーを尊重する** ことです。ベストプラクティスとは、矛盾のないプラクティスです。内部的なバックプレッシャーサポートと競合する動作を避けるように注意していれば、確実にベストプラクティスに従っていると言えるでしょう。

一般的には、

1. 指示がない限り、`.push()` は使用しないでください。
2. false が返された後は `.write()` を呼び出さず、代わりに 'drain' を待機します。
3. ストリームは、Node.js のバージョンや使用するライブラリによって異なります。注意してテストを行ってください。

> 3. ブラウザストリームを構築するための非常に便利なパッケージは [`readable-stream`][] です。Rodd Vagg 氏が [素晴らしいブログ記事][] でこのライブラリの有用性について解説しています。簡単に言うと、このライブラリは [`Readable`][] ストリームに対して一種の自動化されたグレースフルデグラデーション機能を提供し、古いバージョンのブラウザと Node.js をサポートします。

## Readable Streams 固有のルール

これまで、[`.write()`][] がバックプレッシャーにどのような影響を与えるかを見てきましたが、特に [`Writable`][] ストリームに重点を置いてきました。Node.js の機能により、データは技術的には [`Readable`][] から [`Writable`][] へと下流に流れます。
しかし、データ、物質、エネルギーのあらゆる伝送において見られるように、送信元は送信先と同様に重要であり、[`Readable`][] ストリームはバックプレッシャーの処理に不可欠です。

これらのプロセスは互いに連携して効率的に通信を行います。[`Readable`][] が [`Writable`][] ストリームからのデータ送信停止要求を無視した場合、[`.write()`][] の戻り値が正しくない場合と同様に問題が発生する可能性があります。

したがって、[`.write()`][] の戻り値だけでなく、[`._read()`][] メソッドで使用される [`.push()`][] の戻り値も考慮する必要があります。[`.push()`][] が `false` 値を返した場合、ストリームはソースからの読み取りを停止します。それ以外の場合は、読み取りを中断することなく続行します。

[`.push()`][] の使用における不適切な例を以下に示します。

```js
// This is problematic as it completely ignores the return value from the push
// which may be a signal for backpressure from the destination stream!
class MyReadable extends Readable {
  _read(size) {
    let chunk;
    while (null !== (chunk = getNextChunk())) {
      this.push(chunk);
    }
  }
}
```

以下は、`Readable` ストリームが `this.push()` の戻り値をチェックしてバックプレッシャーを尊重する良い例です。

```js
class MyReadable extends Readable {
  _read(size) {
    let chunk;
    let canPushMore = true;
    while (canPushMore && null !== (chunk = getNextChunk())) {
      canPushMore = this.push(chunk);
    }
  }
}
```

さらに、カスタムストリームの外部から見ると、バックプレッシャーを無視することには落とし穴があります。この良い例の反例では、アプリケーションのコードは、データが利用可能になった場合（[`'data'` イベント][] によって通知される）、データを強制的に通過させます。

```js
// これにより、Node.js が設定したバックプレッシャーメカニズムが無視され、
// 宛先ストリームの準備ができているかどうかに関係なく、無条件にデータがプッシュされます。
readable.on('data', data => writable.write(data));
```

以下は、Readable ストリームで [`.push()`][] を使用する例です。

```cjs
const { Readable } = require('node:stream');

// Create a custom Readable stream
const myReadableStream = new Readable({
  objectMode: true,
  read(size) {
    // Push some data onto the stream
    this.push({ message: 'Hello, world!' });
    this.push(null); // Mark the end of the stream
  },
});

// Consume the stream
myReadableStream.on('data', chunk => {
  console.log(chunk);
});

// Output:
// { message: 'Hello, world!' }
```

```mjs
import { Readable } from 'node:stream';

// Create a custom Readable stream
const myReadableStream = new Readable({
  objectMode: true,
  read(size) {
    // Push some data onto the stream
    this.push({ message: 'Hello, world!' });
    this.push(null); // Mark the end of the stream
  },
});

// Consume the stream
myReadableStream.on('data', chunk => {
  console.log(chunk);
});

// Output:
// { message: 'Hello, world!' }
```

この例では、[`.push()`][] を使用して単一のオブジェクトをストリームにプッシュするカスタム Readable ストリームを作成します。[`._read()`][] メソッドは、ストリームがデータを処理する準備ができたときに呼び出されます。この例では、すぐにデータをストリームにプッシュし、null をプッシュしてストリームの終了をマークします。

次に、'data' イベントをリッスンし、ストリームにプッシュされる各データチャンクをログに記録することで、ストリームを処理します。この例では、ストリームにプッシュされるデータチャンクは 1 つだけなので、表示されるログメッセージは 1 つだけです。

## 書き込み可能ストリームに固有のルール

[`.write()`][] は、いくつかの条件に応じて true または false を返す可能性があることを思い出してください。幸いなことに、独自の [`Writable`][] ストリームを構築する場合、[`stream state machine`][] がコールバックを処理し、バックプレッシャーを処理するタイミングを決定し、データフローを最適化します。

ただし、[`Writable`][] を直接使用する場合は、[`.write()`][] の戻り値を尊重し、以下の条件に細心の注意を払う必要があります。

- 書き込みキューがビジー状態の場合、[`.write()`][] は false を返します。
- データチャンクが大きすぎる場合、[`.write()`][] は false を返します（上限は変数 [`highWaterMark`][] で示されます）。

```js
// この書き込み可能オブジェクトは、JavaScript コールバックの非同期性のため無効です。
// 最後のコールバックの前に各コールバックの return 文がないと、
// 複数のコールバックが呼び出される可能性が高くなります。
class MyWritable extends Writable {
  _write(chunk, encoding, callback) {
    if (chunk.toString().indexOf('a') >= 0) {
      callback();
    } else if (chunk.toString().indexOf('b') >= 0) {
      callback();
    }
    callback();
  }
}

// The proper way to write this would be:
if (chunk.contains('a')) {
  return callback();
}

if (chunk.contains('b')) {
  return callback();
}
callback();
```

[`._writev()`][] を実装する際には、いくつか注意すべき点があります。
この関数は [`.cork()`][] と連動していますが、次のような書き方をするとよくある間違いがあります。

```js
// ここで .uncork() を 2 回使用すると、C++ レイヤーで 2 回の呼び出しが行われ、
// cork/uncork テクニックが役に立たなくなります。
ws.cork();
ws.write('hello ');
ws.write('world ');
ws.uncork();

ws.cork();
ws.write('from ');
ws.write('Matteo');
ws.uncork();

// これを記述する正しい方法は、次のイベント ループで実行される 
// process.nextTick() を使用することです。
ws.cork();
ws.write('hello ');
ws.write('world ');
process.nextTick(doUncork, ws);

ws.cork();
ws.write('from ');
ws.write('Matteo');
process.nextTick(doUncork, ws);

// As a global function.
function doUncork(stream) {
  stream.uncork();
}
```

[`.cork()`][] は何度でも呼び出すことができますが、再びストリームを流すには [`.uncork()`][] を同じ回数呼び出すように注意する必要があります。

## まとめ

ストリームは Node.js でよく使われるモジュールです。内部構造にとって重要であり、開発者にとっては Node.js モジュールエコシステム全体への拡張と接続に不可欠です。

これで、バックプレッシャーを考慮した独自の [`Writable`][] および [`Readable`][] ストリームのトラブルシューティングと安全なコーディングが可能になり、同僚や友人と知識を共有できるようになることを願っています。

Node.js でアプリケーションを構築する際に、ストリーミング機能の向上と最大限活用に役立つ他の API 関数については、[`Stream`][] の詳細を必ずお読みください。

[`Stream`]: https://nodejs.org/api/stream.html
[`Buffer`]: https://nodejs.org/api/buffer.html
[`EventEmitters`]: https://nodejs.org/api/events.html
[`Writable`]: https://nodejs.org/api/stream.html#stream_writable_streams
[`Readable`]: https://nodejs.org/api/stream.html#stream_readable_streams
[`Duplex`]: https://nodejs.org/api/stream.html#stream_duplex_and_transform_streams
[`Transform`]: https://nodejs.org/api/stream.html#stream_duplex_and_transform_streams
[`zlib`]: https://nodejs.org/api/zlib.html
[`'drain'`]: https://nodejs.org/api/stream.html#stream_event_drain
[`'data'` event]: https://nodejs.org/api/stream.html#stream_event_data
[`.read()`]: https://nodejs.org/docs/latest/api/stream.html#stream_readable_read_size
[`.write()`]: https://nodejs.org/api/stream.html#stream_writable_write_chunk_encoding_callback
[`._read()`]: https://nodejs.org/docs/latest/api/stream.html#stream_readable_read_size_1
[`._write()`]: https://nodejs.org/docs/latest/api/stream.html#stream_writable_write_chunk_encoding_callback_1
[`._writev()`]: https://nodejs.org/api/stream.html#stream_writable_writev_chunks_callback
[`.cork()`]: https://nodejs.org/api/stream.html#writablecork
[`.uncork()`]: https://nodejs.org/api/stream.html#stream_writable_uncork
[`.push()`]: https://nodejs.org/docs/latest/api/stream.html#stream_readable_push_chunk_encoding
[implementing Writable streams]: https://nodejs.org/docs/latest/api/stream.html#stream_implementing_a_writable_stream
[implementing Readable streams]: https://nodejs.org/docs/latest/api/stream.html#stream_implementing_a_readable_stream
[other packages]: https://github.com/sindresorhus/awesome-nodejs#streams
[`backpressure`]: https://en.wikipedia.org/wiki/Backpressure_routing
[Node.js v0.10]: https://nodejs.org/docs/v0.10.0/
[`highWaterMark`]: https://nodejs.org/api/stream.html#stream_buffering
[return value]: https://github.com/nodejs/node/blob/55c42bc6e5602e5a47fb774009cfe9289cb88e71/lib/_stream_writable.js#L239
[`readable-stream`]: https://github.com/nodejs/readable-stream
[great blog post]: https://r.va.gg/2014/06/why-i-dont-use-nodes-core-stream-module.html
[`dtrace`]: https://dtrace.org/about/
[`zip(1)`]: https://linux.die.net/man/1/zip
[`gzip(1)`]: https://linux.die.net/man/1/gzip
[`stream state machine`]: https://en.wikipedia.org/wiki/Finite-state_machine
[`.pipe()`]: https://nodejs.org/docs/latest/api/stream.html#stream_readable_pipe_destination_options
[piped]: https://nodejs.org/docs/latest/api/stream.html#stream_readable_pipe_destination_options
[`pump`]: https://github.com/mafintosh/pump
[`pipeline`]: https://nodejs.org/api/stream.html#stream_stream_pipeline_streams_callback
[`stream/promises`]: https://nodejs.org/api/stream.html#streampipelinesource-transforms-destination-options
