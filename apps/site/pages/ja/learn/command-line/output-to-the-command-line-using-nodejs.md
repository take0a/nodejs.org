---
title: Output to the command line using Node.js
layout: learn
authors: flaviocopes, potch, MylesBorins, fhemberger, LaRuaNa, amiller-gh, ahmadawais, AugustinMauroy
---

# Node.js を使用したコマンドラインへの出力

### console モジュールを使用した基本的な出力

Node.js は [`console` モジュール](https://nodejs.org/docs/latest-v22.x/api/console.html) を提供しており、コマンドラインを操作するための非常に便利な方法を数多く提供しています。

これは基本的に、ブラウザにある `console` オブジェクトと同じです。

最も基本的かつ最もよく使われるメソッドは `console.log()` で、渡された文字列をコンソールに出力します。

オブジェクトを渡した場合は、文字列としてレンダリングされます。

`console.log` には複数の変数を渡すことができます。例:

```js
const x = 'x';
const y = 'y';

console.log(x, y);
```

Node.js は両方を出力します。

変数と書式指定子を渡すことで、きれいなフレーズをフォーマットすることもできます。

例えば:

```js
console.log('My %s has %d ears', 'cat', 2);
```

- `%s` は変数を文字列としてフォーマットします
- `%d` は変数を数値としてフォーマットします
- `%i` は変数を整数部分のみとしてフォーマットします
- `%o` は変数をオブジェクトとしてフォーマットします

例:

```js
console.log('%o', Number);
```

### コンソールをクリアします

`console.clear()` はコンソールをクリアします（動作は使用するコンソールによって異なる場合があります）

### 要素のカウント

`console.count()` は便利なメソッドです。

次のコードをご覧ください。

```js
const x = 1;
const y = 2;
const z = 3;

console.count(
  'The value of x is ' + x + ' and has been checked .. how many times?'
);

console.count(
  'The value of x is ' + x + ' and has been checked .. how many times?'
);

console.count(
  'The value of y is ' + y + ' and has been checked .. how many times?'
);
```

`console.count()` は文字列が印刷された回数をカウントし、その横にカウント結果を表示します。

リンゴとオレンジを数えることもできます。

```js
const oranges = ['orange', 'orange'];
const apples = ['just one apple'];

oranges.forEach(fruit => {
  console.count(fruit);
});
apples.forEach(fruit => {
  console.count(fruit);
});
```

### カウントのリセット

console.countReset() メソッドは、console.count() で使用されるカウンターをリセットします。

リンゴとオレンジの例を使ってこれを説明します。

```js
const oranges = ['orange', 'orange'];
const apples = ['just one apple'];

oranges.forEach(fruit => {
  console.count(fruit);
});
apples.forEach(fruit => {
  console.count(fruit);
});

console.countReset('orange');

oranges.forEach(fruit => {
  console.count(fruit);
});
```

`console.countReset('orange')` の呼び出しによって値カウンターがゼロにリセットされることに注意してください。

### スタックトレースを出力する

関数の呼び出しスタックトレースを出力すると便利な場合があります。たとえば、「コードのその部分にどうやって到達したのか？」という疑問に答える場合などです。

`console.trace()` を使用すると、これを実行できます。

```js
const function2 = () => console.trace();
const function1 = () => function2();
function1();
```

スタックトレースが出力されます。Node.js REPLでこれを実行した場合の出力は次のとおりです。

```bash
Trace
    at function2 (repl:1:33)
    at function1 (repl:1:25)
    at repl:1:1
    at ContextifyScript.Script.runInThisContext (vm.js:44:33)
    at REPLServer.defaultEval (repl.js:239:29)
    at bound (domain.js:301:14)
    at REPLServer.runBound [as eval] (domain.js:314:12)
    at REPLServer.onLine (repl.js:440:10)
    at emitOne (events.js:120:20)
    at REPLServer.emit (events.js:210:7)
```

### 所要時間を計算する

`time()` と `timeEnd()` を使うと、関数の実行にかかる時間を簡単に計算できます。

```js
const doSomething = () => console.log('test');
const measureDoingSomething = () => {
  console.time('doSomething()');
  // do something, and measure the time it takes
  doSomething();
  console.timeEnd('doSomething()');
};
measureDoingSomething();
```

### stdout と stderr

前述の通り、console.log はコンソールにメッセージを出力するのに最適です。これは標準出力、または `stdout` と呼ばれます。

`console.error` は `stderr` ストリームに出力します。

これはコンソールに表示されますが、通常の出力とは別に処理できます。

### 出力に色を付ける

> **注意**
> このリソースの部分はバージョン 22.11 に基づいて設計されており、`styleText` は「開発中」と記載されています。

多くの場合、ターミナルに適切な出力を得るために、特定のテキストを貼り付けたくなるでしょう。

`node:util` モジュールには `styleText` 関数が用意されています。使い方を確認しましょう。

まず、`node:util` モジュールから `styleText` 関数をインポートする必要があります。

```mjs
import { styleText } from 'node:util';
```

```cjs
const { styleText } = require('node:util');
```

Then, you can use it to style your text:

```js
console.log(
  styleText(['red'], 'This is red text ') +
    styleText(['green', 'bold'], 'and this is green bold text ') +
    'this is normal text'
);
```

最初の引数はスタイルの配列、2番目の引数はスタイルを適用したいテキストです。[ドキュメント](https://nodejs.org/docs/latest-v22.x/api/util.html#utilstyletextformat-text-options)をご覧ください。
