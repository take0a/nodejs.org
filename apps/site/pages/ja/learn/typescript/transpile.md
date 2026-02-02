---
title: Running TypeScript code using transpilation
layout: learn
authors: AugustinMauroy
---

# トランスパイルを使用した TypeScript コードの実行

トランスパイルとは、ソースコードをある言語から別の言語に変換するプロセスです。TypeScript の場合、これは TypeScript コードを JavaScript コードに変換するプロセスです。ブラウザや Node.js は TypeScript コードを直接実行できないため、この処理は必須です。

## TypeScript を JavaScript にコンパイルする

TypeScript コードを実行する最も一般的な方法は、まず JavaScript にコンパイルすることです。これは、TypeScript コンパイラ `tsc` を使って行うことができます。

**ステップ 1:** TypeScript コードをファイル（例: `example.ts`）に記述します。

<!--
  Maintainers note: this code is duplicated in the previous article, please keep them in sync
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

**ステップ 2:** パッケージマネージャーを使用して TypeScript をローカルにインストールします。

この例では npm を使用します。詳細については、[npm パッケージマネージャーの紹介](/learn/getting-started/an-introduction-to-the-npm-package-manager) をご覧ください。

```bash displayName="Install TypeScript locally"
npm i -D typescript # -D is a shorthand for --save-dev
```

**ステップ 3:** `tsc` コマンドを使用して TypeScript コードを JavaScript にコンパイルします。

```bash
npx tsc example.ts
```

> **注:** `npx` は、Node.js パッケージをグローバルにインストールせずに実行できるツールです。

`tsc` は TypeScript コンパイラで、TypeScript コードを JavaScript にコンパイルします。
このコマンドを実行すると、Node.js で実行できる `example.js` という新しいファイルが生成されます。
TypeScript コードのコンパイルと実行方法がわかったので、TypeScript のバグ防止機能を実際に確認してみましょう。

**ステップ 4:** Node.js を使用して JavaScript コードを実行します。

```bash
node example.js
```

ターミナルに TypeScript コードの出力が表示されます。

## 型エラーがある場合

TypeScript コードに型エラーがある場合、TypeScript コンパイラはそれを検出し、コードの実行をブロックします。例えば、`justine` の `age` プロパティを文字列に変更すると、TypeScript はエラーをスローします。

意図的に型エラーを発生させるために、コードを次のように変更します。

```ts
// @errors: 2322 2554
type User = {
  name: string;
  age: number;
};

function isAdult(user: User): boolean {
  return user.age >= 18;
}

const justine: User = {
  name: 'Justine',
  age: 'Secret!',
};

const isJustineAnAdult: string = isAdult(justine, "I shouldn't be here!");
```

ご覧のとおり、TypeScriptはバグが発生する前に発見するのに非常に役立ちます。これが、TypeScriptが開発者の間で非常に人気がある理由の一つです。
