---
title: Running TypeScript with a runner
layout: learn
authors: AugustinMauroy
---

# ランナーを使った TypeScript の実行

TypeScript の組み込みサポートよりも高度な処理が必要な場合（または Node.js v22.7.0 より前のバージョンを使用している場合）、ランナーを使用する（複雑な処理の大部分を自動で処理します）か、[トランスパイル](./transpile) を使ってすべてを自分で処理するかの 2 つの選択肢があります。

## `ts-node` を使った TypeScript コードの実行

[ts-node](https://typestrong.org/ts-node/) は、Node.js 用の TypeScript 実行環境です。TypeScript コードをコンパイルせずに Node.js 内で直接実行できます。デフォルトでは、`transpileOnly` が有効になっていない限り、`ts-node` が型チェックを実行します。`ts-node` は実行時に型エラーを検出できますが、コードをリリースする前に `tsc` を使って型チェックすることをお勧めします。

`ts-node` を使用するには、まずインストールする必要があります。

```bash
npm i -D ts-node
```

次に、次のように TypeScript コードを実行できます。

```bash
npx ts-node example.ts
```

## `tsx` を使った TypeScript コードの実行

[tsx](https://tsx.is/) は、Node.js 用のもう 1 つの TypeScript 実行環境です。TypeScript コードをコンパイルせずに Node.js 内で直接実行できます。ただし、コードの型チェックは行われません。そのため、コードをリリースする前に、まず `tsc` で型チェックを行い、その後 `tsx` で実行することをお勧めします。

`tsx` を使用するには、まずインストールする必要があります。

```bash
npm i -D tsx
```

次に、次のように TypeScript コードを実行できます。

```bash
npx tsx example.ts
```

### `node` 経由で `tsx` を登録する

`node` 経由で `tsx` を使用する場合は、`--import` 経由で `tsx` を登録できます。

```bash
node --import=tsx example.ts
```
