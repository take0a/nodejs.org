---
title: Run Node.js scripts from the command line
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, akazyti, AugustinMauroy
---

# コマンドラインから Node.js スクリプトを実行する

Node.js プログラムを実行する一般的な方法は、Node.js をインストールしたらグローバルに利用可能な `node` コマンドを実行し、実行するファイル名を渡すことです。

メインの Node.js アプリケーションファイルが `app.js` の場合、次のように入力して呼び出すことができます。

```bash
node app.js
```

上記では、シェルにスクリプトを `node` で実行するように明示的に指示しています。この情報は、JavaScript ファイルに ["shebang"](<https://en.wikipedia.org/wiki/Shebang_(Unix)>) 行を埋め込むことでも実現できます。"shebang" はファイルの先頭行で、スクリプトの実行にどのインタープリタを使用するかを OS に指示します。以下は JavaScript の先頭行です。

```js
#!/usr/bin/node
```

上記では、インタープリタの絶対パスを明示的に指定しています。すべてのOSのbinフォルダに`node`があるわけではありませんが、`env`は必ず存在します。`env`を実行するようにOSに指示するには、nodeをパラメータとして指定します。

```js
#!/usr/bin/env node

// your javascript code
```

shebang を使用するには、ファイルに実行権限が必要です。`app.js` に実行権限を付与するには、次のコマンドを実行します。

```bash
chmod u+x app.js
```

コマンドを実行するときは、`app.js` ファイルが含まれているディレクトリと同じディレクトリにいることを確認してください。

## `node` にファイルパスではなく文字列を引数として渡します。

文字列を引数として実行するには、`-e` または `--eval "script"` を使用します。引数に続く引数を JavaScript として評価します。REPL で定義済みのモジュールもスクリプト内で使用できます。

Windows では、cmd.exe は引用符として二重の `"` しか認識しないため、一重引用符は正しく動作しません。Powershell または Git bash では、`'` と `"` の両方が使用できます。

```bash
node -e "console.log(123)"
```

## アプリケーションを自動的に再起動する

Node.js V16 以降では、ファイルが変更されたときにアプリケーションを自動的に再起動する組み込みオプションがあります。これは開発用途に便利です。
この機能を使用するには、Node.js に `--watch` フラグを渡す必要があります。

```bash
node --watch app.js
```

そのため、ファイルを変更すると、アプリケーションは自動的に再起動します。
[`--watch` フラグのドキュメント](https://nodejs.org/docs/latest-v22.x/api/cli.html#--watch)をお読みください。

## Node.js でタスクを実行する

Node.js には、`package.json` ファイルで定義された特定のコマンドを実行できる組み込みのタスクランナーが用意されています。これは、テストの実行、プロジェクトのビルド、コードの lint チェックといった反復的なタスクの自動化に特に役立ちます。

### `--run` フラグの使用

[`--run`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--run) フラグを使用すると、`p​​ackage.json` ファイルの `scripts` セクションから指定したコマンドを実行できます。例えば、次のような `package.json` があるとします。

```json
{
  "type": "module",
  "scripts": {
    "start": "node app.js",
    "dev": "node --run start -- --watch",
    "test": "node --test"
  }
}
```

`--run` フラグを使用して `test` スクリプトを実行できます。

```bash
node --run test
```

### コマンドへの引数の渡し方

`package.json` ファイルの `scripts` オブジェクトにある `dev` キーについて説明します。

`-- --another-argument` 構文は、コマンドに引数を渡すために使用されます。この場合、`--watch` 引数が `dev` スクリプトに渡されます。

```bash
node --run dev
```

### 環境変数

`--run` フラグは、スクリプトに役立つ特定の環境変数を設定します。

- `NODE_RUN_SCRIPT_NAME`: 実行するスクリプトの名前。
- `NODE_RUN_PACKAGE_JSON_PATH`: 処理対象の `package.json` ファイルへのパス。

### 意図的な制限事項

Node.js タスクランナーは、`npm run` や `yarn run` などの他のタスクランナーと比較して、意図的に機能が制限されています。パフォーマンスとシンプルさを重視しており、`pre` スクリプトや `post` スクリプトの実行といった機能は省略されています。そのため、単純なタスクには適していますが、すべてのユースケースに対応できるとは限りません。
