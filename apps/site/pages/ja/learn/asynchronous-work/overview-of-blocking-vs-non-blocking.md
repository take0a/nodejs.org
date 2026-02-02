---
title: Overview of Blocking vs Non-Blocking
layout: learn
authors: ovflowd, HassanBahati
---

# ブロッキングとノンブロッキングの概要

この概要では、Node.jsにおける**ブロッキング**呼び出しと**ノンブロッキング**呼び出しの違いについて説明します。イベントループとlibuvについても言及しますが、これらのトピックに関する事前の知識は必要ありません。読者はJavaScript言語とNode.jsの[コールバックパターン](/learn/asynchronous-work/javascript-asynchronous-programming-and-callbacks)の基礎知識があることを前提としています。

> 「I/O」とは、主に[libuv](https://libuv.org/)によってサポートされているシステムのディスクおよびネットワークとのやり取りを指します。

## ブロッキング

**ブロッキング** とは、Node.js プロセスにおける追加の JavaScript の実行が、JavaScript 以外の操作が完了するまで待機しなければならない状態です。これは、**ブロッキング** 操作の実行中はイベントループが JavaScript の実行を継続できないために発生します。

Node.js では、I/O などの JavaScript 以外の操作を待機するのではなく、CPU を集中的に使用することでパフォーマンスが低下する JavaScript は、通常、**ブロッキング** とは呼ばれません。Node.js 標準ライブラリの libuv を使用する同期メソッドは、最も一般的に使用される**ブロッキング** 操作です。ネイティブモジュールにも**ブロッキング** メソッドが存在する場合があります。

Node.js 標準ライブラリのすべての I/O メソッドは、**非ブロッキング** な非同期バージョンを提供しており、コールバック関数を受け入れます。一部のメソッドには、名前が `Sync` で終わる**ブロッキング** メソッドもあります。

## コードの比較

**ブロッキング**メソッドは**同期**的に実行され、**非ブロッキング**メソッドは**非同期**的に実行されます。

ファイルシステムモジュールを例に挙げると、これは**同期**的なファイル読み取りです。

```js
const fs = require('node:fs');

const data = fs.readFileSync('/file.md'); // blocks here until file is read
```

以下は同等の **非同期** の例です。

```js
const fs = require('node:fs');

fs.readFile('/file.md', (err, data) => {
  if (err) {
    throw err;
  }
});
```

最初の例は2番目の例よりもシンプルに見えますが、2行目ではファイル全体が読み込まれるまで追加のJavaScriptの実行が**ブロック**されるという欠点があります。同期バージョンでは、エラーが発生した場合、それをキャッチしないとプロセスがクラッシュすることに注意してください。非同期バージョンでは、エラーをスローするかどうかは作成者が決定します。

例を少し拡張してみましょう。

```js
const fs = require('node:fs');

const data = fs.readFileSync('/file.md'); // blocks here until file is read
console.log(data);
moreWork(); // will run after console.log
```

以下は、同様ですが同等ではない非同期の例です。

```js
const fs = require('node:fs');

fs.readFile('/file.md', (err, data) => {
  if (err) {
    throw err;
  }

  console.log(data);
});
moreWork(); // will run before console.log
```

上記の最初の例では、`console.log` が `moreWork()` の前に呼び出されます。2番目の例では、`fs.readFile()` が**非ブロッキング** であるため、JavaScript の実行は継続され、`moreWork()` が最初に呼び出されます。ファイルの読み取りが完了するのを待たずに `moreWork()` を実行できる機能は、スループットを向上させるための重要な設計上の選択です。

## 同時実行性とスループット

Node.js における JavaScript の実行はシングルスレッドです。そのため、同時実行性とは、他の処理を完了した後に JavaScript コールバック関数を実行できるイベントループの能力を指します。同時実行が想定されるコードは、I/O などの JavaScript 以外の操作が行われている間もイベントループの実行を継続できるようにする必要があります。

例として、Web サーバーへの各リクエストの完了に 50 ミリ秒かかり、そのうち 45 ミリ秒が非同期で実行可能なデータベース I/O である場合を考えてみましょう。**ノンブロッキング** 非同期操作を選択すると、リクエストごとに 45 ミリ秒が解放され、他のリクエストを処理できるようになります。これは、**ブロッキング** メソッドではなく **ノンブロッキング** メソッドを使用するだけで、処理能力が大幅に向上することを意味します。

イベントループは、同時実行処理のために追加のスレッドが作成される可能性のある他の多くの言語のモデルとは異なります。

## ブロッキングコードとノンブロッキングコードを混在させる危険性

I/Oを扱う際に避けるべきパターンがいくつかあります。例を見てみましょう。

```js
const fs = require('node:fs');

fs.readFile('/file.md', (err, data) => {
  if (err) {
    throw err;
  }

  console.log(data);
});
fs.unlinkSync('/file.md');
```

上記の例では、`fs.unlinkSync()` が `fs.readFile()` の前に実行される可能性が高く、`file.md` が実際に読み込まれる前に削除されてしまいます。これを完全に **ノンブロッキング** で、正しい順序で実行されることが保証されるより良い方法は、次のとおりです。

```js
const fs = require('node:fs');

fs.readFile('/file.md', (readFileErr, data) => {
  if (readFileErr) {
    throw readFileErr;
  }

  console.log(data);

  fs.unlink('/file.md', unlinkErr => {
    if (unlinkErr) {
      throw unlinkErr;
    }
  });
});
```

上記は、`fs.readFile()` のコールバック内に `fs.unlink()` への **非ブロッキング** 呼び出しを配置することで、正しい操作順序を保証します。

## 追加リソース

- [libuv](https://libuv.org/)
