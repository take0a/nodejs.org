---
title: An introduction to the npm package manager
layout: learn
authors: flaviocopes, MylesBorins, LaRuaNa, jgb-solutions, amiller-gh, ahmadawais
---

# npm パッケージマネージャーの紹介

## npm 入門

`npm` は Node.js の標準パッケージマネージャーです。

2022年9月時点で、npm レジストリには210万以上のパッケージが登録されていると報告されており、世界最大の単一言語コードリポジトリとなっています。ほぼあらゆる言語のパッケージが見つかるはずです。

npm は Node.js パッケージの依存関係をダウンロードおよび管理するための手段として始まりましたが、現在ではフロントエンド JavaScript でも利用されるツールとなっています。

> [**Yarn**](https://yarnpkg.com) と [**pnpm**](https://pnpm.io) は npm cli の代替ツールです。ぜひご確認ください。

## パッケージ

`npm` は、プロジェクトの依存関係のインストール、更新、ダウンロード管理を行います。依存関係とは、Node.js アプリケーションの動作に必要な、ライブラリやパッケージなどの事前に構築されたコードです。

### すべての依存関係のインストール

プロジェクトに`package.json`ファイルがある場合は、

```bash
npm install
```

を実行することで、プロジェクトに必要なものがすべて `node_modules` フォルダにインストールされ、まだ存在しない場合は作成されます。

### 単一パッケージのインストール

特定のパッケージをインストールするには、次のコマンドを実行します。

```bash
npm install <package-name>
```

さらに、npm 5以降では、このコマンドは`<package-name>`を`package.json`ファイルの_dependencies_に追加します。バージョン5より前では、`--save`フラグを追加する必要がありました。

このコマンドには、他にもフラグが追加されていることがよくあります。

- `--save-dev` はインストールを行い、`package.json` ファイル _devDependencies_ にエントリを追加します。
- `--no-save` はインストールを行いますが、`package.json` ファイル _dependencies_ にエントリを追加しません。
- `--save-optional` はインストールを行い、`package.json` ファイル _optionalDependencies_ にエントリを追加します。
- `--no-optional` はオプションの依存関係がインストールされないようにします。

フラグの短縮形も使用できます:

- \-S: `--save`
- \-D: `--save-dev`
- \-O: `--save-optional`

_devDependencies_ と _dependencies_ の違いは、前者はテストライブラリなどの開発ツールを含むのに対し、後者は本番環境でアプリにバンドルされている点です。

_optionalDependencies_ の違いは、依存関係のビルドが失敗してもインストールは失敗しないという点です。ただし、依存関係の不足に対処するのはプログラム側の役割です。[optional dependencies](https://docs.npmjs.com/cli/configuring-npm/package-json#optionaldependencies) の詳細については、こちらをご覧ください。

### パッケージのアップデート

アップデートも簡単に実行できます。

```bash
npm update
```

`npm` は、バージョン制約を満たす新しいバージョンがないか、すべてのパッケージをチェックします。

更新するパッケージを 1 つだけ指定することもできます。

```bash
npm update <package-name>
```

## バージョン管理

通常のダウンロードに加えて、`npm` は **バージョン管理** も管理します。そのため、パッケージの特定のバージョンを指定したり、必要なバージョンよりも高いバージョンまたは低いバージョンを要求したりできます。

多くの場合、ライブラリは別のライブラリのメジャーリリースとのみ互換性があります。

あるいは、ライブラリの最新リリースにまだ修正されていないバグがあり、それが問題を引き起こしている場合もあります。

ライブラリのバージョンを明示的に指定することで、全員が同じバージョンのパッケージを使用するようになり、`package.json` ファイルが更新されるまで、チーム全体で同じバージョンを実行できるようになります。

これらのすべてのケースでバージョン管理は非常に役立ち、`npm` はセマンティックバージョニング (semver) 標準に準拠しています。

特定のバージョンのパッケージをインストールするには、以下のコマンドを実行します。

```bash
npm install <package-name>@<version>
```

## タスクの実行

package.jsonファイルは、以下のコマンドで実行できるコマンドラインタスクを指定するためのフォーマットをサポートしています。

```bash
npm run <task-name>
```

例えば:

```json
{
  "scripts": {
    "start-dev": "node lib/server-development",
    "start": "node lib/server-production"
  }
}
```

Webpack を実行するためにこの機能を使用するのは非常に一般的です:

```json
{
  "scripts": {
    "watch": "webpack --watch --progress --colors --config webpack.conf.js",
    "dev": "webpack --progress --colors --config webpack.conf.js",
    "prod": "NODE_ENV=production webpack -p --config webpack.conf.js"
  }
}
```

忘れたり、間違えたりしやすい長いコマンドを入力する代わりに、以下のように実行できます。

```console
$ npm run watch
$ npm run dev
$ npm run prod
```
