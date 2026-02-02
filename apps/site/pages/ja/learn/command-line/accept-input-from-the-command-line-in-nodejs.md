---
title: Accept input from the command line in Node.js
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais
---

# Node.js でコマンドラインからの入力を受け入れる

Node.js CLI プログラムをインタラクティブにするにはどうすればよいですか？

Node.js バージョン 7 以降では、[`readline` モジュール](https://nodejs.org/docs/latest-v22.x/api/readline.html) が提供されており、まさにこれを実行します。つまり、`process.stdin` ストリームなどの読み取り可能なストリームから入力を 1 行ずつ取得します。Node.js プログラムの実行中は、このストリームはターミナル入力となります。

```cjs
const readline = require('node:readline');

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

rl.question(`What's your name?`, name => {
  console.log(`Hi ${name}!`);
  rl.close();
});
```

```mjs
import readline from 'node:readline';

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

rl.question(`What's your name?`, name => {
  console.log(`Hi ${name}!`);
  rl.close();
});
```

このコードはユーザーに名前を尋ね、テキストが入力されてEnterキーが押されると、挨拶を送信します。

`question()`メソッドは最初のパラメータ（質問）を表示し、ユーザーの入力を待ちます。Enterキーが押されると、コールバック関数を呼び出します。

このコールバック関数では、readlineインターフェースを閉じます。

`readline`には他にもいくつかのメソッドが用意されています。上記のパッケージドキュメントでご確認ください。

パスワードを要求する場合は、パスワードをエコーバックするのではなく、代わりに`*`記号を表示するのが最適です。
