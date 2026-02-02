---
title: How to read environment variables from Node.js
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, manishprivet, nikhilbhatt, ycmjason
---

# Node.js から環境変数を読み取る方法

Node.js の `process` コアモジュールは、プロセス開始時に設定されたすべての環境変数を保持する `env` プロパティを提供します。

以下のコードは `app.js` を実行し、`USER_ID` と `USER_KEY` を設定します。

```bash
USER_ID=239482 USER_KEY=foobar node app.js
```

これにより、ユーザーの `USER_ID` が **239482** として、`USER_KEY` が **foobar** として渡されます。これはテストには適していますが、本番環境では、変数をエクスポートするための bash スクリプトを設定する必要があるでしょう。

> 注: `process` は Node.js のグローバルオブジェクトであるため、インポートする必要はありません。

上記のコードで設定した `USER_ID` および `USER_KEY` 環境変数にアクセスする例を以下に示します。

```js
console.log(process.env.USER_ID); // "239482"
console.log(process.env.USER_KEY); // "foobar"
```

同様の方法で、設定したカスタム環境変数にもアクセスできます。

Node.js 20 では、**実験的**な [.env ファイルのサポート](https://nodejs.org/docs/v24.5.0/api/environment_variables.html#env-files) が導入されました。

Node.js アプリケーションの実行時に、`--env-file` フラグを使用して環境ファイルを指定できるようになりました。以下に、`.env` ファイルの例と、`process.env` を使用してその変数にアクセスする方法を示します。

```bash
# .env file
PORT=3000
```

jsファイル内は

```js
console.log(process.env.PORT); // "3000"
```

`.env` ファイルで設定された環境変数を使用して `app.js` ファイルを実行します。

```bash
node --env-file=.env app.js
```

このコマンドは、`.env` ファイルからすべての環境変数を読み込み、`process.env` でアプリケーションが使用できるようにします。

また、`--env-file` 引数を複数渡すこともできます。後続のファイルは、前のファイルで定義された既存の変数を上書きします。

```bash
node --env-file=.env --env-file=.development.env app.js
```

> 注意: 環境とファイルで同じ変数が定義されている場合、環境の値が優先されます。

`.env` ファイルから値を読み込む必要がある場合は、`--env-file-if-exists` フラグを使用することで、ファイルが存在しない場合にエラーが発生するのを回避できます。

```bash
node --env-file-if-exists=.env app.js
```

## `process.loadEnvFile(path)` を使用したプログラムによる `.env` ファイルの読み込み

Node.js には、コードから直接 `.env` ファイルを読み込むための組み込み API が用意されています: [`process.loadEnvFile(path)`](https://nodejs.org/api/process.html#processloadenvfilepath)。

このメソッドは、`--env-file` フラグと同様に、`.env` ファイルから `process.env` に変数を読み込みますが、プログラムから呼び出すこともできます。

このメソッドは初期化後に呼び出されるため、起動関連の環境変数 (`NODE_OPTIONS` など) の設定はプロセスに影響を与えません (ただし、これらの変数には引き続き `process.env` 経由でアクセスできます)。

### 例

```txt
// .env file
PORT=1234
```

```js
const { loadEnvFile } = require('node:process');

// Loads environment variables from the default .env file
loadEnvFile();

console.log(process.env.PORT); // Logs '1234'
```

カスタム パスを指定することもできます。

```js
const { loadEnvFile } = require('node:process');
loadEnvFile('./config/.env');
```
