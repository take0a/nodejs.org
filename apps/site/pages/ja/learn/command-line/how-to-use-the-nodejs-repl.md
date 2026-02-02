---
title: How to use the Node.js REPL
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, vaishnav-mk, AugustinMauyroy
---

# Node.js REPLの使い方

## Node.js REPLとは？

Node.jsには、JavaScriptコードを対話的に実行できるREPL（Read-Eval-Print Loop）環境が組み込まれています。REPLはターミナルからアクセスでき、小さなコードをテストするのに最適です。

## Node.js REPL の使い方

`node` コマンドは、Node.js スクリプトを実行するために使用します。

```bash
node script.js
```

実行するスクリプトや引数を指定せずに `node` コマンドを実行すると、REPL セッションが開始されます。

```bash
node
```

> **注:** `REPL` は Read Evaluate Print Loop の略で、ユーザー入力として単一の式を受け取り、実行後に結果をコンソールに返すプログラミング言語環境（基本的にはコンソールウィンドウ）です。REPL セッションは、単純な JavaScript コードを素早くテストするための便利な方法を提供します。

ターミナルで試してみると、次のような結果になります。

```bash
❯ node
>
```

コマンドはアイドル モードのまま、何かが入力されるまで待機します。

> **ヒント:** ターミナルの開き方がわからない場合は、「お使いのオペレーティングシステムでターミナルを開く方法」をGoogleで検索してください。

REPLは、より正確にはJavaScriptコードの入力を待機しています。

まずはシンプルに、次のように入力してみましょう。

```console
> console.log('test')
test
undefined
>
```

最初の値「test」は、コンソールに出力するように指示した出力です。次に「undefined」が返されます。これは「console.log()」を実行した際の戻り値です。
Node はこのコード行を読み取り、評価し、結果を出力した後、次のコード行を待機する状態に戻ります。Node は、セッションが終了するまで、REPL で実行されるすべてのコードに対して、この 3 つのステップをループします。これが REPL の名前の由来です。

Node は、JavaScript コードの任意の行の結果を自動的に出力します。出力を指示する必要はありません。例えば、次の行を入力して Enter キーを押します。

```console
> 5 === '5'
false
>
```

上記の2行の出力の違いに注目してください。Node.jsのREPLは`console.log()`の実行後に`undefined`を出力しましたが、一方では`5 === '5'`の結果のみを出力しています。前者はJavaScriptの単なる文であり、後者は式であることを覚えておいてください。

場合によっては、テストしたいコードが複数行に渡ることがあります。例えば、乱数を生成する関数を定義したい場合、REPLセッションで次の行を入力してEnterキーを押します。

```console
function generateRandom() {
...
```

Node REPLは、コードの記述がまだ完了していないことを判断して、複数行モードに移行し、さらにコードを入力できるようにします。関数定義を終えてEnterキーを押してください。

```console
function generateRandom() {
...return Math.random()
}
undefined
```

### 特殊変数 `_`

コードの後に​​ `_` と入力すると、直前の操作の結果が表示されます。

### 上矢印キー

上矢印キーを押すと、現在の REPL セッション、さらには以前の REPL セッションで実行されたコード行の履歴にアクセスできます。

### ドットコマンド

REPLには、ドット「.」で始まる特別なコマンドがいくつかあります。

- `.help`: ドットコマンドのヘルプを表示します。
- `.editor`: エディターモードを有効にし、複数行のJavaScriptコードを簡単に記述できます。このモードに入ったら、Ctrl + D を押すと記述したコードが実行されます。
- `.break`: 複数行の式を入力中に .break コマンドを入力すると、それ以降の入力が中断されます。Ctrl + C を押すのと同じです。
- `.clear`: REPLコンテキストを空のオブジェクトにリセットし、現在入力中の複数行の式をクリアします。
- `.load`: 現在の作業ディレクトリを基準としたJavaScriptファイルを読み込みます。
- `.save`: REPLセッションで入力した内容をすべてファイルに保存します（ファイル名を指定してください）。
- `.exit`: REPLを終了します（Ctrl + Cを2回押すのと同じ動作です）。

REPLは、`.editor`を起動しなくても、複数行の文を入力していることを認識します。

例えば、次のような反復処理を入力し始めたとします。

```console
[1, 2, 3].forEach(num => {
```

そして `enter` を押すと、REPL は 3 つのドットで始まる新しい行に移動し、そのブロックで作業を続行できることを示します。

```console
... console.log(num)
... })
```

行末に `.break` と入力すると、複数行モードが停止し、ステートメントは実行されません。

### JavaScript ファイルから REPL を実行する

`repl` を使って、JavaScript ファイルに REPL をインポートできます。

```cjs
const repl = require('node:repl');
```

```mjs
import repl from 'node:repl';
```

repl変数を使うと、様々な操作を実行できます。
REPLコマンドプロンプトを起動するには、次の行を入力します。

```js
repl.start();
```

コマンドラインでファイルを実行します。

```bash
node repl.js
```

```console
> const n = 10
```

REPLの起動時に表示する文字列を渡すことができます。デフォルトは「>」（末尾にスペースあり）ですが、カスタムプロンプトを定義することもできます。

```js
// a Unix style prompt
const local = repl.start('$ ');
```

REPLを終了するときにメッセージを表示できます

```js
local.on('exit', () => {
  console.log('exiting repl');
  process.exit();
});
```

REPL モジュールの詳細については、[repl ドキュメント](https://nodejs.org/api/repl.html) を参照してください。
