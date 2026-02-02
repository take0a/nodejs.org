---
title: Running TypeScript Natively
layout: learn
authors: AugustinMauroy, khaosdoctor, jakebailey, robpalme
---

# TypeScript をネイティブに実行

Node.js で TypeScript として有効なコードを直接記述できます。事前にトランスパイルする必要はありません。

v22.18.0 以降を使用しており、ソースコードに [消去可能な TypeScript 構文](https://devblogs.microsoft.com/typescript/announceing-typescript-5-8-beta/#the---erasablesyntaxonly-option) のみが含まれている場合は、フラグを指定せずに TypeScript コードを実行できます。

```bash
node example.ts
```

v22.18.0 未満のバージョンを使用している場合は、`--experimental-strip-types` フラグを使用して、TypeScript ファイルを Node.js で直接実行できます。

```bash
node --experimental-strip-types example.ts
```

これで完了です！TypeScript コードを事前にトランスパイルすることなく Node.js で直接実行できるようになり、TypeScript を使用して型関連のエラーをキャッチできるようになりました。

必要に応じて、[`--no-experimental-strip-types`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--no-experimental-strip-types) フラグで無効化できます。

```bash
node --no-experimental-strip-types example.ts
```

v22.7.0 では、`enum` や `namespace` などの変換を必要とする TypeScript 専用の構文を有効にするフラグ [`--experimental-transform-types`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--experimental-transform-types) が追加されました。`--experimental-transform-types` を有効にすると、自動的に `--experimental-strip-types` も有効になるため、同じコマンドで両方のフラグを使用する必要はありません。

```bash
node --experimental-transform-types another-example.ts
```

このフラグはオプトインであり、コードで必要な場合にのみ使用する必要があります。

## 制約事項

Node.js における TypeScript のサポートには、留意すべき制約事項がいくつかあります。

詳細については、[API ドキュメント](https://nodejs.org/docs/latest-v22.x/api/typescript.html#typescript-features) をご覧ください。

### 設定

Node.js TypeScript ローダー ([Amaro](https://github.com/nodejs/amaro)) は、TypeScript コードの実行に `tsconfig.json` を必要としません。また、使用しません。

Node.js の動作を反映するようにエディターと `tsc` を設定することをお勧めします。そのためには、[こちら](https://nodejs.org/api/typescript.html#type-stripping) に記載されている `compilerOptions` を使用して `tsconfig.json` を作成し、TypeScript バージョン **5.7 以上** を使用することをお勧めします。
