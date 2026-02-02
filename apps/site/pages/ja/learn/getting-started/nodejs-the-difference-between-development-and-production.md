---
title: Node.js, the difference between development and production
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, RenanTKN, mcollina
---

# Node.js：開発環境と本番環境の違い

**Node.js には開発環境と本番環境の違いはありません**。つまり、Node.js を本番環境設定で動作させるために特別な設定は必要ありません。
ただし、npm レジストリ内のいくつかのライブラリは `NODE_ENV` 変数を認識し、デフォルトで `development` 設定を使用します。
Node.js は常に `NODE_ENV=production` に設定して実行してください。

アプリケーションを設定する一般的な方法は、[twelve factor methodology](https://12factor.net/) を使用することです。

## なぜ NODE_ENV はアンチパターンとみなされるのでしょうか？

環境とは、エンジニアがソフトウェア製品を構築、テスト、デプロイ、管理できるデジタルプラットフォームまたはシステムです。通常、アプリケーションが実行される環境には4つの段階（種類）があります。

- 開発
- テスト
- ステージング
- 本番環境

`NODE_ENV` の根本的な問題は、開発者が最適化とソフトウェアの動作を、ソフトウェアが実行される環境と混同していることに起因します。その結果、次のようなコードが生成されます。

```js
if (process.env.NODE_ENV === 'development') {
  // ...
}

if (process.env.NODE_ENV === 'production') {
  // ...
}

if (['production', 'staging'].includes(process.env.NODE_ENV)) {
  // ...
}
```

これは一見無害に見えるかもしれませんが、本番環境とステージング環境が異なるため、信頼性の高いテストが不可能になります。例えば、`NODE_ENV` を `development` に設定するとテストが成功し、製品の機能も成功する可能性がありますが、`NODE_ENV` を `production` に設定すると失敗する可能性があります。
したがって、`NODE_ENV` を `production` 以外の値に設定することは、_アンチパターン_ と見なされます。
