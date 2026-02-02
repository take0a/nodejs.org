---
title: Introduction to Node.js
layout: learn
authors: flaviocopes, potch, MylesBorins, RomainLanz, virkt25, Trott, onel0p3z, ollelauribostrom, MarkPieszak, fhemberger, LaRuaNa, FrozenPandaz, mcollina, amiller-gh, ahmadawais, saqibameen, dangen-effy, aymen94, benhalverson
---

# Node.js入門

Node.js はオープンソースでクロスプラットフォームな JavaScript ランタイム環境です。ほぼあらゆるプロジェクトで人気のツールです。

Node.js は、Google Chrome の中核である V8 JavaScript エンジンをブラウザの外部で実行します。これにより、Node.js は非常に高いパフォーマンスを実現します。

Node.js アプリは単一プロセスで実行され、リクエストごとに新しいスレッドを作成する必要はありません。Node.js は標準ライブラリに、JavaScript コードのブロッキングを防ぐための非同期 I/O プリミティブを提供しています。さらに、Node.js のライブラリは一般的に非ブロッキングパラダイムを使用して記述されています。そのため、Node.js ではブロッキング動作は例外であり、一般的ではありません。

Node.js がネットワークからの読み取り、データベースやファイルシステムへのアクセスなどの I/O 操作を実行する際、スレッドをブロックして CPU サイクルを無駄に待機させるのではなく、レスポンスが返ってくると操作を再開します。

これにより、Node.js は単一のサーバーで数千の同時接続を処理できます。スレッドの同時実行性を管理する負担は、バグの大きな原因となる可能性があります。

Node.js には独自の利点があります。ブラウザ用の JavaScript を記述する何百万人ものフロントエンド開発者が、全く異なる言語を習得することなく、クライアントサイドのコードに加えてサーバーサイドのコードも記述できるようになったのです。

Node.js では、すべてのユーザーがブラウザを更新するのを待つ必要がないため、新しい ECMAScript 標準を問題なく使用できます。Node.js のバージョンを変更することで、使用する ECMAScript のバージョンを決定できます。また、フラグを指定して Node.js を実行することで、特定の試験的な機能を有効にすることもできます。

## Node.js アプリケーションの例

Node.js の Hello World の最も一般的な例は、Web サーバーです。

```cjs
const { createServer } = require('node:http');

const hostname = '127.0.0.1';
const port = 3000;

const server = createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello World');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

```mjs
import { createServer } from 'node:http';

const hostname = '127.0.0.1';
const port = 3000;

const server = createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello World');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

このスニペットを実行するには、`server.js` ファイルとして保存し、ターミナルで `node server.js` を実行してください。
mjs バージョンのコードを使用する場合は、`server.mjs` ファイルとして保存し、ターミナルで `node server.mjs` を実行してください。

このコードは、まず Node.js の [`http` モジュール](https://nodejs.org/api/http.html) をインクルードします。

Node.js には、ネットワークのファーストクラスサポートを含む、優れた [標準ライブラリ](https://nodejs.org/api/) があります。

`http` の `createServer()` メソッドは、新しい HTTP サーバーを作成して返します。

サーバーは指定されたポートとホスト名でリッスンするように設定されています。サーバーの準備が完了すると、コールバック関数が呼び出され、この場合はサーバーが稼働中であることを通知します。

新しいリクエストを受信するたびに、[`request` イベント](https://nodejs.org/api/http.html#http_event_request) が呼び出され、リクエスト（[`http.IncomingMessage`](https://nodejs.org/api/http.html#http_class_http_incomingmessage) オブジェクト）とレスポンス（[`http.ServerResponse`](https://nodejs.org/api/http.html#http_class_http_serverresponse) オブジェクト）の2つのオブジェクトが提供されます。

これら2つのオブジェクトは、HTTP 呼び出しを処理するために不可欠です。

最初のオブジェクトはリクエストの詳細を提供します。このシンプルな例では使用されていませんが、リクエストヘッダーとリクエストデータにアクセスできます。

2番目は呼び出し元にデータを返すために使用されます。

この場合、次のようになります。

```js
res.statusCode = 200;
```

`statusCode` プロパティを `200` に設定し、レスポンスが成功したことを示します。

`Content-Type` ヘッダーを設定します。

```js
res.setHeader('Content-Type', 'text/plain');
```

そして、コンテンツを `end()` の引数として追加して、レスポンスを閉じます。

```js
res.end('Hello World\n');
```

まだ行っていない場合は、[ダウンロード](https://nodejs.org/en/download) Node.js してください。
