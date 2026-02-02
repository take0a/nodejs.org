---
title: Introduction to TypeScript
layout: learn
authors: sbielenica, ovflowd, vaishnav-mk, AugustinMauroy
---

# TypeScript 入門

## TypeScript とは

**[TypeScript](https://www.typescriptlang.org)** は、Microsoft が保守・開発しているオープンソース言語です。

基本的に、TypeScript は JavaScript に構文を追加することで、エディターとの緊密な統合をサポートします。エディター内や CI/CD パイプラインでエラーを早期に検出し、より保守性の高いコードを作成できます。

TypeScript のその他のメリットについては後ほど詳しく説明しますので、まずはいくつか例を見てみましょう。

## 最初の TypeScript コード

このコードスニペットを見て、一緒に解説しましょう。

<!--
  Maintainers note: this code is duplicated in the next article, please keep them in sync
-->

```ts
type User = {
  name: string;
  age: number;
};

function isAdult(user: User): boolean {
  return user.age >= 18;
}

const justine = {
  name: 'Justine',
  age: 23,
} satisfies User;

const isJustineAnAdult = isAdult(justine);
```

最初の部分（キーワード `type` を使用）は、ユーザーを表すカスタムオブジェクト型の宣言を担当します。その後、この新しく作成された型を利用して、`User` 型の引数を1つ受け取り、`boolean` を返す関数 `isAdult` を作成します。その後、先ほど定義した関数を呼び出すために使用できるサンプルデータである `justine` を作成します。最後に、`justine` が成人かどうかの情報を格納する新しい変数を作成します。

この例について、他に知っておくべきことがあります。まず、宣言された型に準拠していない場合、TypeScript は誤りを通知し、誤用を防止します。次に、すべてを明示的に型指定する必要はありません。TypeScript が型を推論します。例えば、変数 `isJustineAnAdult` は、明示的に型指定していなくても `boolean` 型になります。また、`justine` は、`User` 型として宣言していなくても、関数の有効な引数になります。

## TypeScript は何で構成されていますか？

TypeScript は、コード自体と型定義という2つの主要なコンポーネントで構成されています。

### TypeScript コード

コード部分は、型アノテーション用の TypeScript 固有の構文が追加された通常の JavaScript です。TypeScript コードをコンパイルすると、TypeScript 固有の部分はすべて削除され、あらゆる環境で実行できるクリーンな JavaScript になります。例:

```ts displayName="example.ts"
function greet(name: string) {
  console.log(`Hello, ${name}!`);
}
```

### 型定義

型定義は、既存の JavaScript コードの形状を記述します。通常、`.d.ts` ファイルに保存され、実際の実装は含まれず、型のみを記述します。これらの定義は、JavaScript との相互運用性にとって不可欠です。コードは通常、TypeScript として配布されるのではなく、サイドカー型定義ファイルを含む JavaScript にトランスパイルされます。

例えば、Node.js を TypeScript と併用する場合、Node.js API の型定義が必要になります。これは `@types/node` から入手できます。以下のコマンドでインストールしてください。

```bash
npm add --save-dev @types/node
```

これらの型定義により、TypeScript は Node.js API を理解し、`fs.readFile` や `http.createServer` などの関数を使用する際に適切な型チェックと自動補完を提供できるようになります。例えば、次のようになります。

```ts
// @errors: 2345
import { resolve } from 'node:path';

resolve(123, 456);
```

多くの人気JavaScriptライブラリは、DefinitelyTypedコミュニティによってメンテナンスされている`@types`名前空間で型定義を利用できます。これにより、既存のJavaScriptライブラリをTypeScriptプロジェクトにシームレスに統合できます。

### 変換機能

TypeScript には、特に React などのフレームワークで使用される JSX 向けの強力な変換機能も備わっています。TypeScript コンパイラは、Babel と同様に、JSX 構文を通常の JavaScript に変換できます。この記事ではこれらの変換機能については取り上げませんが、TypeScript は型チェックツールであるだけでなく、最新の JavaScript 構文をさまざまな環境に対応可能なバージョンに変換するビルドツールでもあることを覚えておくとよいでしょう。

## TypeScript コードの実行方法

さて、TypeScript コードができました。では、どのように実行すればいいのでしょうか？
TypeScript コードを実行する方法はいくつかありますが、次の記事ですべて取り上げます。
