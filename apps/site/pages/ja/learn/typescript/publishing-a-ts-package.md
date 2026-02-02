---
title: Publishing a TypeScript package
layout: learn
authors: JakobJingleheimer
---

# TypeScript パッケージの公開

この記事では、TypeScript の公開に関する項目について具体的に説明します。公開とは、npm（またはその他のパッケージマネージャー）経由でパッケージとして配布することを意味します。本番環境で実行するためのアプリやサーバー（PWA やエンドポイントサーバーなど）をコンパイルすることではありません。

重要な注意事項：

- [パッケージの公開](../modules/publishing-a-package) のすべてがここに適用されます。
  - `main` などのフィールドは公開されたコンテンツに対して操作されるため、TypeScript ソースコードが JavaScript にトランスパイルされると、JavaScript が公開されたコンテンツとなり、`main` は JavaScript ファイル拡張子を持つ JavaScript ファイルを指します（例：`main.ts` → `"main": "main.js"`）。

  - `scripts.test` などのフィールドはソースコードに対して操作されるため、ソースコードのファイル拡張子を使用します（例：`"test": "node --test './src/**/*.test.ts'`）。

- Node.jsはTypeScriptコードを「[type stripping](https://nodejs.org/api/typescript.html#type-stripping)」と呼ばれるプロセスで実行します。このプロセスでは、Node.js（[Amaro](https://github.com/nodejs/amaro)経由）がTypeScript固有の構文を削除し、Node.jsが既に理解している標準のJavaScriptを残します。この動作は、Node.jsバージョン22.18.0以降、デフォルトで有効になっています。
  - Node.jsは`node_modules`内の型を削除しません。これは、公式TypeScriptコンパイラ（`tsc`）とVS Codeの一部で重大なパフォーマンス問題を引き起こす可能性があるためです。そのため、TypeScriptのメンテナーは、少なくとも現時点では、生のTypeScriptを公開することを推奨していません。

- Node.jsで`enum`などのTypeScript固有の機能を使用するには、引き続きフラグ（[`--experimental-transform-types`](https://nodejs.org/api/typescript.html#typescript-features)）が必要です。いずれにせよ、これらよりも優れた代替手段がしばしば存在します。
  - TypeScript 固有の機能が存在しない（つまりコードを Node.js で実行できない）ようにするには、TypeScript バージョン 5.8 以降で [`erasableSyntaxOnly`](https://devblogs.microsoft.com/typescript/announce-typescript-5-8-beta/#the---erasablesyntaxonly-option) 構成オプションを設定してください。

- GitHub Actions 内の依存関係も含め、依存関係を最新の状態に保つには [dependabot](https://docs.github.com/en/code-security/dependabot) を使用してください。これは非常に簡単な設定で、一度設定すれば後は不要です。

- `.nvmrc` は Node.js のマルチバージョンマネージャーである [`nvm`](https://github.com/nvm-sh/nvm) から取得されます。これにより、プロジェクトで一般的に使用する Node.js のバージョンを指定できます。

リポジトリのディレクトリ概要は次のようになります。

```text displayName="Files co-located"
example-ts-pkg/
├ .github/
│ ├ workflows/
│ │ ├ ci.yml
│ │ └ publish.yml
│ └ dependabot.yml
├ src/
│ ├ foo.fixture.js
│ ├ main.ts
│ ├ main.test.ts
│ ├ some-util.ts
│ └ some-util.test.ts
├ LICENSE
├ package.json
├ README.md
└ tsconfig.json
```

```text displayName="Files co-located but segregated"
example-ts-pkg/
├ .github/
│ ├ workflows/
│ │ ├ ci.yml
│ │ └ publish.yml
│ └ dependabot.yml
├ src/
│ ├ __test__/
│ │ ├ foo.fixture.js
│ │ ├ main.test.ts
│ ├ main.ts
│ └ some-util/
│   ├ __test__
│   │ └ some-util.test.ts
│   └ some-util.ts
├ LICENSE
├ package.json
├ README.md
└ tsconfig.json
```

```text displayName="'src' and 'test' fully segregated"
example-ts-pkg/
├ .github/
│ ├ workflows/
│ │ ├ ci.yml
│ │ └ publish.yml
│ └ dependabot.yml
├ src/
│ ├ main.ts
│ ├ some-util.ts
├ test/
│ ├ foo.fixture.js
│ ├ main.ts
│ └ some-util.ts
├ LICENSE
├ package.json
├ README.md
└ tsconfig.json
```

公開されたパッケージのディレクトリの概要は次のようになります。

```text displayName="Fully flat"
example-ts-pkg/
├ LICENSE
├ main.d.ts
├ main.d.ts.map
├ main.js
├ package.json
├ README.md
├ some-util.d.ts
├ some-util.d.ts.map
└ some-util.js
```

```text displayName="With 'dist'"
example-ts-pkg/
├ dist/
│ ├ main.d.ts
│ ├ main.d.ts.map
│ ├ main.js
│ ├ some-util.d.ts
│ ├ some-util.d.ts.map
│ └ some-util.js
├ LICENSE
├ package.json
└ README.md
```

ディレクトリ構成に関する注意点：テストの配置には、いくつかの一般的な方法があります。最小知識の原則では、テストは実装と隣接して配置することが推奨されています。場合によっては、同じディレクトリ内に配置するか、`__test__` のような引き出し内に配置することもあります（これも実装と隣接しており、「ファイルは同一場所に配置されながらも分離されている」という意味です）。あるいは、`src/` の兄弟である `test/` を作成する方法もあります（「'src' と 'test' は完全に分離されている」という意味です）。これは、ミラー構造または「ジャンク引き出し」のいずれかで実現されます。

## 型の扱い方

### 型をテストのように扱う

型の目的は、実装が動作しないことを警告することです。

```ts
// @errors: 2322
const foo = 'a';
const bar: number = 1 + foo;
```

TypeScript は、上記のコードが意図したとおりに動作しないことを警告しています。これは、ユニットテストがコードが意図したとおりに動作しないことを警告するのと同じです。これらは補完的なものであり、異なることを検証するため、両方を使用する必要があります。

お使いのエディター（例：VS Code）には、TypeScript のサポートが組み込まれている可能性があり、作業中にエラーが表示されます。そうでない場合、またはエラーを見逃した場合は、CI が役立ちます。

以下の [GitHub Action](https://github.com/features/actions) は、`main` ブランチへの PR で型が検査に合格していることを自動的にチェック（および要求）する CI タスクを設定します。

```yaml displayName=".github/workflows/ci.yml"
# yaml-language-server: $schema=https://json.schemastore.org/github-workflow.json

name: Tests

on:
  pull_request:
    branches: ['*']

jobs:
  check-types:
    # Separate these from tests because
    # they are platform and node-version independent
    # and need be run only once.

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'
      - name: npm clean install
        run: npm ci
      # You may want to run a lint check here too
      - run: node --run types:check

  get-matrix:
    # Automatically pick active LTS versions
    runs-on: ubuntu-latest
    outputs:
      latest: ${{ steps.set-matrix.outputs.requireds }}
    steps:
      - uses: ljharb/actions/node/matrix@main
        id: set-matrix
        with:
          versionsAsRoot: true
          type: majors
          preset: '>= 22' # glob is not backported below 22.x

  test:
    needs: [get-matrix]
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false
      matrix:
        node-version: ${{ fromJson(needs.get-matrix.outputs.latest) }}
        os:
          - macos-latest
          - ubuntu-latest
          - windows-latest

    steps:
      - uses: actions/checkout@v4
      - name: Use node ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - name: npm clean install
        run: npm ci
      - run: node --run test
```

```json displayName="package.json"
{
  "name": "example-ts-pkg",
  "scripts": {
    "test": "node --test './src/**/*.test.ts'",
    "types:check": "tsc --noEmit"
  },
  "devDependencies": {
    "typescript": "^5.7.2"
  }
}
```

```json displayName="tsconfig.json (flat output)"
{
  "compilerOptions": {
    "allowArbitraryExtensions": true,
    "declaration": true,
    "declarationMap": true,
    "lib": ["es2023"],
    "module": "NodeNext",
    "outDir": "./",
    "resolveJsonModule": true,
    "rewriteRelativeImportExtensions": true,
    "target": "es2022"
  },
  // These may be different for your repo:
  "include": ["./src"],
  "exclude": ["**/*/*.test.*", "**/*.fixture.*"]
}
```

```json displayName="tsconfig.json ('dist' output)"
{
  "compilerOptions": {
    "allowArbitraryExtensions": true,
    "declaration": true,
    "declarationMap": true,
    "lib": ["es2023"],
    "module": "NodeNext",
    "outDir": "./dist",
    "resolveJsonModule": true,
    "rewriteRelativeImportExtensions": true,
    "target": "es2022"
  },
  // These may be different for your repo:
  "include": ["./src"],
  "exclude": ["**/*/*.test.*", "**/*.fixture.*"]
}
```

テストファイルには異なる `tsconfig.json` が適用されている可能性があることに注意してください（上記のサンプルでは除外されているのはこのためです）。

### 型宣言を生成する

型宣言（`.d.ts` など）は、サイドカーファイルとして型情報を提供します。これにより、実行コードは型を持ちながらも、バニラ JavaScript で記述できます。

型宣言はソースコードに基づいて生成されるため、公開プロセスの一環としてビルドすることができ、リポジトリにチェックインする必要はありません。

次の例では、npm レジストリに公開する直前に型宣言が生成されています。

```yaml displayName=".github/workflows/publish.yml"
# yaml-language-server: $schema=https://json.schemastore.org/github-workflow.json

# This is mostly boilerplate.

name: Publish to npm
on:
  push:
    tags:
      - '**@*'

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci

      # - name: Publish to npm
      #   run: … npm publish …
```

```diff displayName="package.json"
{
  "name": "example-ts-pkg",
  "scripts": {
+   "prepack": "tsc",
    "types:check": "tsc --noEmit"
  }
}
```

```ini displayName=".npmignore"
*.*ts       # foo.cts foo.mts foo.ts
!*.d.*ts
*.fixture.*
```

```ini displayName=".npmignore ('dist' output)"
src
test
```

ユーザーがどのバージョンの Node.js LTS を実行するかわからないため、すべての Node.js LTS バージョンをサポートするようにコンパイルされたパッケージを公開する必要があります。この記事の `tsconfig` は Node.js 18.x 以降をサポートしています。

`npm publish` は [`prepack` を事前に自動的に実行します](https://docs.npmjs.com/cli/using-npm/scripts#npm-publish)。`npm` は `npm pack --dry-run` の前にも `prepack` を自動的に実行します (そのため、実際に公開しなくても、公開されるパッケージの内容を簡単に確認できます)。**注意** : [`node --run` はそれを_実行しません_](../command-line/run-nodejs-scripts-from-the-command-line.md#using-the---run-flag)。この手順では `node --run` を使用できないため、この注意事項はここでは適用されませんが、他の手順では適用されます。

実際に npm に公開する手順については、別の記事で説明します (この記事の範囲外には、長所と短所がいくつかあります)。

#### 詳細

型宣言の生成は決定論的です。つまり、同じ入力から毎回同じ出力が得られます。そのため、型宣言をgitにコミットする必要はありません。

[`npm publish`](https://docs.npmjs.com/cli/commands/npm-publish) は、コマンド実行時に適用可能かつ利用可能なすべてのものを取得します。そのため、直前に型宣言を生成しておけば、それらは利用可能であり、取得されます。

デフォルトでは、`npm publish` は（ほぼ）すべてを取得します（[パッケージに含まれるファイル](https://docs.npmjs.com/cli/commands/npm-publish#files-included-in-package) を参照）。公開パッケージを最小限に保つには（`node_modules` に関する「宇宙で最も重いオブジェクト」ミームを参照）、特定のファイル（テストやテストフィクスチャなど）をパッケージ化から除外する必要があります。これらを [`.npmignore`](https://docs.npmjs.com/cli/using-npm/developers#keeping-files-out-of-your-package) で指定されたオプトアウトリストに追加します。`!*.d.ts` 例外がリストされていることを確認してください。リストされていない場合、生成された型宣言は公開されません。または、[package.json "files"](https://docs.npmjs.com/cli/configuring-npm/package-json#files) を使用してオプトインを作成することもできます（誤ってファイルを省略すると、下流のユーザーに対してパッケージが壊れる可能性があるため、このオプションはあまり安全ではありません）。
