---
title: How to publish a Node-API package
layout: learn
---

# パッケージのNode-API版を非Node-API版と並行して公開する方法

以下の手順は、パッケージ「iotivity-node」を使用して説明しています。

- まず、非Node-API版を公開します。
  - `package.json` のバージョンを更新します。`iotivity-node` の場合、バージョンは
  `1.2.0-2` になります。
  - リリースチェックリストを確認します（テスト/デモ/ドキュメントが問題ないことを確認）。
  - `npm publish`
- 次に、Node-API バージョンを公開します。
  - `package.json` のバージョンを更新します。`iotivity-node` の場合、
  バージョンは `1.2.0-3` になります。バージョン管理については、
  [semver.org](https://semver.org/#spec-item-9) に記載されているプレリリース版のバージョンスキームに従うことをお勧めします（例：`1.2.0-napi`）。
  - リリースチェックリストを確認します（テスト、デモ、ドキュメントに問題がないことを確認します）。
  - `npm publish --tag n-api`

この例では、リリースに `n-api` というタグを付けることで、
バージョン 1.2.0-3 は Node-API 非対応の公開バージョン（1.2.0-2）よりも新しいバージョンですが、
`npm install iotivity-node` を実行して `iotivity-node` をインストールしようとしても、インストールされません。これにより、デフォルトで非Node-APIバージョンがインストールされます。Node-APIバージョンを取得するには、ユーザーは「npm install iotivity-node@n-api」を実行する必要があります。npmでのタグの使用に関する詳細は、["dist-tagsの使用"][] を参照してください。

## パッケージの Node-API バージョンへの依存関係を導入する方法

`iotivity-node` の Node-API バージョンを依存関係として追加するには、`package.json` は次のようになります。

```json
"dependencies": {
  "iotivity-node": "n-api"
}
```

> ["dist-tagsの使用"][] で説明したように、通常のバージョンとは異なり、タグ付きバージョンは `package.json` 内で `"^2.0.0"` のようなバージョン範囲で指定できません。これは、タグが1つのバージョンのみを参照するためです。そのため、パッケージメンテナーが同じタグを使用してパッケージの後のバージョンをタグ付けする場合、`npm update` は後のバージョンを受け取ります。これは最新の公開バージョン以外の許容バージョンである必要があります。`package.json` 依存関係は、次のように正確なバージョンを参照する必要があります。

```json
"dependencies": {
  "iotivity-node": "1.2.0-3"
}
```

["dist-tagsの使用"]: https://docs.npmjs.com/getting-started/using-tags
