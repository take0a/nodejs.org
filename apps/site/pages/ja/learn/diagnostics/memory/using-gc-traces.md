---
title: Tracing garbage collection
layout: learn
---

# ガベージコレクションのトレース

このガイドでは、ガベージコレクションのトレースの基礎について説明します。

このガイドを読み終える頃には、以下のことができるようになります。

- Node.js アプリケーションでトレースを有効にする
- トレースを解釈する
- Node.js アプリケーションにおける潜在的なメモリ問題を特定する

ガベージコレクターの仕組みについては学ぶべきことがたくさんありますが、特に覚えておくべきことは、GC が実行されている間はコードは実行されていないということです。

ガベージコレクションの実行頻度と実行時間、そしてその結果はどうなるのかを知りたいと思うかもしれません。

## セットアップ

このガイドでは、以下のスクリプトを使用します。

```mjs
// script.mjs

import os from 'node:os';

let len = 1_000_000;
const entries = new Set();

function addEntry() {
  const entry = {
    timestamp: Date.now(),
    memory: os.freemem(),
    totalMemory: os.totalmem(),
    uptime: os.uptime(),
  };

  entries.add(entry);
}

function summary() {
  console.log(`Total: ${entries.size} entries`);
}

// execution
(() => {
  while (len > 0) {
    addEntry();
    process.stdout.write(`~~> ${len} entries to record\r`);
    len--;
  }

  summary();
})();
```

> ここでリークが明らかであっても、実際のアプリケーションではリークの原因を特定するのは困難です。

## ガベージコレクションのトレース付きで実行

`--trace-gc` フラグを使用すると、プロセスのコンソール出力でガベージコレクションのトレースを表示できます。

```console
$ node --trace-gc script.mjs
```

> 注: この [exercise][] のソースコードは Node.js Diagnostics リポジトリにあります。

次のような出力が得られるはずです。

```bash
[39067:0x158008000]     2297 ms: Scavenge 117.5 (135.8) -> 102.2 (135.8) MB, 0.8 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
[39067:0x158008000]     2375 ms: Scavenge 120.0 (138.3) -> 104.7 (138.3) MB, 0.9 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
[39067:0x158008000]     2453 ms: Scavenge 122.4 (140.8) -> 107.1 (140.8) MB, 0.7 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
[39067:0x158008000]     2531 ms: Scavenge 124.9 (143.3) -> 109.6 (143.3) MB, 0.7 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
[39067:0x158008000]     2610 ms: Scavenge 127.1 (145.5) -> 111.8 (145.5) MB, 0.7 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
[39067:0x158008000]     2688 ms: Scavenge 129.6 (148.0) -> 114.2 (148.0) MB, 0.8 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
[39067:0x158008000]     2766 ms: Scavenge 132.0 (150.5) -> 116.7 (150.5) MB, 1.1 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
Total: 1000000 entries
```

読みにくいですか？ここでいくつかの概念を復習し、`--trace-gc` フラグの出力について説明しましょう。

### `--trace-gc` でトレースを調べる

`--trace-gc` フラグ（または `--trace_gc` フラグ、どちらでも構いません）は、すべてのガベージコレクションイベントをコンソールに出力します。
各行の構成は次のように記述できます。

```bash
[13973:0x110008000]       44 ms: Scavenge 2.4 (3.2) -> 2.0 (4.2) MB, 0.5 / 0.0 ms  (average mu = 1.000, current mu = 1.000) allocation failure
```

| トークン値 | 解釈 |
| ----------------------------------------------------- | ---------------------------------------- |
| 13973 | 実行中のプロセスの PID |
| 0x110008000 | 分離 (JS ヒープ インスタンス) |
| 44 ミリ秒 | プロセス開始からの時間 (ミリ秒) |
| スカベンジ | GC のタイプ / フェーズ |
| 2.4 | GC 前のヒープ使用量 (MB) |
| (3.2) | GC 前のヒープ使用量 (MB) |
| 2.0 | GC 後のヒープ使用量 (MB) |
| (4.2) | GC 後のヒープ使用量 (MB) |
| 0.5 / 0.0 ミリ秒 (平均 mu = 1.000、現在の mu = 1.000) | GC にかかった時間 (ミリ秒) |
| 割り当て失敗 | GC の理由 |

ここでは、以下の2つのイベントにのみ焦点を当てます。

- Scavenge
- Mark-sweep

ヒープは_スペース_に分割されています。これらのスペースの中には、「新しい」スペースと「古い」スペースがあります。

> 👉 実際のヒープ構造は少し異なりますが、この記事ではよりシンプルなバージョンを前提とします。詳細については、Orinocoに関するこちらの[Peter Marshallの講演][]をご覧ください。

### Scavenge

Scavengeとは、新しいスペースにガベージコレクションを実行するアルゴリズムの名前です。新しいスペースは、オブジェクトが作成される場所です。
新しいスペースは、ガベージコレクションのために小さく高速になるように設計されています。

Scavenge のシナリオを想像してみましょう。

- `A`、`B`、`C`、`D` をアロケートしました。
  ```bash
  | A | B | C | D | <unallocated> |
  ```
- `E` をアロケートしたい
- 十分なスペースがないため、メモリが枯渇している
- その後、（ガベージ）コレクションが起動する
- 不要なオブジェクトは回収される
- 生きているオブジェクトはそのまま残る
- `B` と `D` が不要なオブジェクトだと仮定
  ```bash
  | A | C | <unallocated> |
  ```
- これで `E` をアロケートすることができます
  ```bash
  | A | C | E | <unallocated> |
  ```

v8 は、古い領域への 2 回の Scavenge 操作の後にガベージ コレクションされるのではなく、オブジェクトを昇格します。

> 👉 Full [Scavenge scenario][]

### マークスイープ

マークスイープは、古い空間からオブジェクトを収集するために使用されます。古い空間とは、新しい空間で生き残ったオブジェクトが生息する場所です。

このアルゴリズムは2つのフェーズで構成されます。

- **マーク**: まだ生きているオブジェクトを黒、それ以外を白としてマークします。
- **スイープ**: 白いオブジェクトをスキャンし、それらをフリースペースに変換します。

> 👉 実際、マークとスイープのステップはもう少し複雑です。
> 詳細については、こちらの[document][]をご覧ください。

<img src="https://upload.wikimedia.org/wikipedia/commons/4/4a/Animation_of_the_Naive_Mark_and_Sweep_Garbage_Collector_Algorithm.gif" alt="mark and sweep algorithm"/>

## `--trace-gc` の動作

### メモリリーク

さて、先ほどのターミナルウィンドウに戻ると、
コンソールに多数の `Mark-sweep` イベントが表示されます。
また、イベント後に収集されたメモリ量はごくわずかであることもわかります。

これでガベージコレクションのエキスパートになりました！ 何が推測できるでしょうか？

おそらくメモリリークが発生しているでしょう！ しかし、それをどのように確認できるでしょうか？
(注意: この例ではかなり明白ですが、実際のアプリケーションではどうでしょうか？)

しかし、そのコンテキストをどのように特定できるでしょうか？

### 不適切な割り当てのコンテキストを取得する方法

1. 古い領域が継続的に増加していることが観測されたとします。
2. ヒープの合計が上限に近づくように、[`--max-old-space-size`][] を減らします。
3. メモリ不足に達するまでプログラムを実行します。
4. 生成されたログに、失敗したコンテキストが表示されます。
5. メモリ不足に達した場合は、ヒープサイズを約10%ずつ増やし、これを数回繰り返します。同じパターンが見られる場合、メモリリークが発生していることを示しています。
6. メモリ不足が発生していない場合は、ヒープサイズをその値に固定します。ヒープを圧縮すると、メモリフットプリントと計算レイテンシが削減されます。

例えば、次のコマンドで `script.mjs` を実行してみてください。

```bash
node --trace-gc --max-old-space-size=50 script.mjs
```

OOM が発生するはずです:

```bash
[...]
<--- Last few GCs --->
[40928:0x148008000]      509 ms: Mark-sweep 46.8 (65.8) -> 40.6 (77.3) MB, 6.4 / 0.0 ms  (+ 1.4 ms in 11 steps since start of marking, biggest step 0.2 ms, walltime since start of marking 24 ms) (average mu = 0.977, current mu = 0.977) finalize incrementa[40928:0x148008000]      768 ms: Mark-sweep 56.3 (77.3) -> 47.1 (83.0) MB, 35.9 / 0.0 ms  (average mu = 0.927, current mu = 0.861) allocation failure scavenge might not succeed
<--- JS stacktrace --->
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory [...]
```

では、100MB で試してみましょう:

```bash
node --trace-gc --max-old-space-size=100 script.mjs
```

同様の現象が発生するはずですが、唯一の違いは、最後の GC トレースにはより大きなヒープ サイズが含まれるという点です。

```bash
<--- Last few GCs --->
[40977:0x128008000]     2066 ms: Mark-sweep (reduce) 99.6 (102.5) -> 99.6 (102.5) MB, 46.7 / 0.0 ms  (+ 0.0 ms in 0 steps since start of marking, biggest step 0.0 ms, walltime since start of marking 47 ms) (average mu = 0.154, current mu = 0.155) allocati[40977:0x128008000]     2123 ms: Mark-sweep (reduce) 99.6 (102.5) -> 99.6 (102.5) MB, 47.7 / 0.0 ms  (+ 0.0 ms in 0 steps since start of marking, biggest step 0.0 ms, walltime since start of marking 48 ms) (average mu = 0.165, current mu = 0.175) allocati
```

> 注: 実際のアプリケーションでは、コード内でリークしたオブジェクトを見つけるのは面倒な場合があります。ヒープスナップショットが役立つ場合があります。[guide dedicated to heap snapshot][] をご覧ください。

### 遅延

ガベージコレクションが多すぎるか、オーバーヘッドを引き起こしているかをどのように判断しますか？

1. トレースデータ、特に連続するガベージコレクション間の時間を確認します。
2. トレースデータ、特にGCに費やされた時間を確認します。
3. 2回のGC間の時間がGCに費やされた時間よりも短い場合、アプリケーションのメモリ不足が深刻です。
4. 2回のGC間の時間とGCに費やされた時間が非常に長い場合、アプリケーションはより小さなヒープを使用できる可能性があります。
5. 2回のGC間の時間がGCに費やされた時間よりもはるかに長い場合、アプリケーションは比較的健全です。

## リークを修正する

では、リークを修正しましょう。エントリの保存にオブジェクトを使用する代わりに、ファイルを使用することもできます。

スクリプトを少し変更してみましょう。

```mjs
// script-fix.mjs
import fs from 'node:fs/promises';
import os from 'node:os';

let len = 1_000_000;
const fileName = `entries-${Date.now()}`;

async function addEntry() {
  const entry = {
    timestamp: Date.now(),
    memory: os.freemem(),
    totalMemory: os.totalmem(),
    uptime: os.uptime(),
  };
  await fs.appendFile(fileName, JSON.stringify(entry) + '\n');
}

async function summary() {
  const stats = await fs.lstat(fileName);
  console.log(`File size ${stats.size} bytes`);
}

// execution
(async () => {
  await fs.writeFile(fileName, '----START---\n');
  while (len > 0) {
    await addEntry();
    process.stdout.write(`~~> ${len} entries to record\r`);
    len--;
  }

  await summary();
})();
```

`Set` を使用してデータを保存するのは決して悪い習慣ではありません。プログラムのメモリ使用量に注意するだけで十分です。

> 注: この [exercise][] のソースコードは、Node.js Diagnostics リポジトリにあります。

それでは、このスクリプトを実行してみましょう。

```
node --trace-gc script-fix.mjs
```

次の2点に注意してください。

- マークスイープイベントの発生頻度が低い
- メモリ使用量が最初のスクリプトでは130MBを超えていたのに対し、新しいバージョンでは25MBを超えていない。

新しいバージョンは最初のバージョンよりもメモリへの負荷が低いため、これは非常に理にかなっています。

**まとめ**: このスクリプトの改善点についてどう思いますか？
新しいバージョンのスクリプトは遅いことに気付いたかもしれません。
`Set` を再び使用し、メモリが特定のサイズに達した場合にのみその内容をファイルに書き込むようにしたらどうでしょうか？

> [`getheapstatistics`][] APIが役立つかもしれません。

## ボーナス: プログラムでガベージコレクションをトレースする

### `v8` モジュールの使用

プロセスの全期間にわたるトレースを取得したくない場合があります。
その場合は、プロセス内からフラグを設定します。
`v8` モジュールは、フラグをオンザフライで設定するためのAPIを公開しています。

```js
import v8 from 'v8';

// enabling trace-gc
v8.setFlagsFromString('--trace-gc');

// disabling trace-gc
v8.setFlagsFromString('--notrace-gc');
```

### パフォーマンスフックの使用

Node.js では、[performance hooks][] を使用してガベージコレクションをトレースできます。

```cjs
const { PerformanceObserver } = require('node:perf_hooks');

// Create a performance observer
const obs = new PerformanceObserver(list => {
  const entry = list.getEntries()[0];
  /*
  The entry is an instance of PerformanceEntry containing
  metrics of a single garbage collection event.
  For example:
  PerformanceEntry {
    name: 'gc',
    entryType: 'gc',
    startTime: 2820.567669,
    duration: 1.315709,
    kind: 1
  }
  */
});

// Subscribe to notifications of GCs
obs.observe({ entryTypes: ['gc'] });

// Stop subscription
obs.disconnect();
```

### パフォーマンスフックを使ったトレースの調査

[PerformanceObserver][] のコールバックから [PerformanceEntry][] として GC 統計を取得できます。

例:

```json
{
  "name": "gc",
  "entryType": "gc",
  "startTime": 2820.567669,
  "duration": 1.315709,
  "kind": 1
}
```

| Property  | Interpretation                                                       |
| --------- | ------------------------------------------------------------------------------------------------ |
| name | パフォーマンス エントリの名前。 |
| entryType | パフォーマンス エントリのタイプ。 |
| startTime | 高解像度のミリ秒単位のタイムスタンプは、パフォーマンス エントリの開始時刻を示します。 |
| duration | このエントリの経過時間の合計 (ミリ秒単位)。 |
| kind | 発生したガベージ コレクション操作のタイプ。 |
| flags | GC に関する追加情報。 |

詳細については、[パフォーマンス フックに関するドキュメント][performance hooks]を参照してください。

[PerformanceEntry]: https://nodejs.org/api/perf_hooks.html#perf_hooks_class_performanceentry
[PerformanceObserver]: https://nodejs.org/api/perf_hooks.html#perf_hooks_class_performanceobserver
[`--max-old-space-size`]: https://nodejs.org/api/cli.html#--max-old-space-sizesize-in-megabytes
[performance hooks]: https://nodejs.org/api/perf_hooks.html
[exercise]: https://github.com/nodejs/diagnostics/tree/main/documentation/memory/step3/exercise
[guide dedicated to heap snapshot]: /learn/diagnostics/memory/using-heap-snapshot#how-to-find-a-memory-leak-with-heap-snapshots
[document]: https://github.com/thlorenz/v8-perf/blob/master/gc.md#marking-state
[Scavenge scenario]: https://github.com/thlorenz/v8-perf/blob/master/gc.md#sample-scavenge-scenario
[talk of Peter Marshall]: https://v8.dev/blog/trash-talk
[`getheapstatistics`]: https://nodejs.org/dist/latest-v16.x/docs/api/v8.html#v8getheapstatistics
