---
title: Node.js WebSocket
layout: learn
authors: callezenwaka
---

# Node.js のネイティブ WebSocket クライアント

## はじめに

[Node.js v21](https://github.com/nodejs/node/blob/47a59bde2aadb3ad1b377c0ef12df7abc28840e9/doc/changelogs/CHANGELOG_V21.md#L1329-L1345) 以降、[WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) は [Undici](https://undici.nodejs.org) ライブラリを使用して拡張され、組み込みの WebSocket クライアントが導入されました。これにより、Node.js アプリケーションのリアルタイム通信が簡素化されます。[Node.js v22.4.0](https://github.com/nodejs/node/releases/tag/v22.4.0) リリースでは、WebSocket API が安定版としてマークされ、本番環境での使用が可能になりました。

## WebSocketとは

[WebSocket](https://en.wikipedia.org/wiki/WebSocket)は、単一のTCP接続で同時双方向通信を可能にする標準化された通信プロトコルです。全二重（双方向）通信が可能で、HTTPとは異なります。WebSocketは、プロトコル移行にHTTP Upgradeヘッダーを使用することでHTTPとの互換性を実現しています。サーバーは初期リクエストなしでクライアントにコンテンツをプッシュし、継続的なメッセージ交換のために接続を維持できるため、HTTPポーリングなどの代替手段よりもオーバーヘッドを抑えながら、リアルタイムのデータ転送に最適です。WebSocket通信は通常、TCPポート443（セキュリティ保護済み）または80（セキュリティ保護なし）を介して行われるため、Web以外の接続におけるファイアウォールの制限を回避できます。このプロトコルは、暗号化されていない接続と暗号化された接続にそれぞれ独自のURIスキーム（ws://とwss://）を定義しており、すべての主要ブラウザでサポートされています。

## ネイティブ WebSocket クライアント

Node.js は、クライアント接続に [ws](https://www.npmjs.com/package/ws) や [socket.io](https://www.npmjs.com/package/socket.io) などの外部ライブラリに依存することなく、WebSocket クライアントとして動作できるようになりました。これにより、Node.js アプリケーションは送信 WebSocket 接続を直接開始および管理できるようになり、リアルタイムデータフィードへの接続や他の WebSocket サーバーとのやり取りといったタスクを効率化できます。ユーザーは、標準の [new WebSocket()](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket/WebSocket) コンストラクターを使用して、WebSocket クライアント接続を作成できるようになりました。

上記の内容を基に、基本的なユースケースを示す新しい WebSocket クライアント機能のより実用的な例を追加してみましょう。

### 基本的な接続とメッセージの処理

```javascript
// Creates a new WebSocket connection to the specified URL.
const socket = new WebSocket('ws://localhost:8080');

// Executes when the connection is successfully established.
socket.addEventListener('open', event => {
  console.log('WebSocket connection established!');
  // Sends a message to the WebSocket server.
  socket.send('Hello Server!');
});

// Listen for messages and executes when a message is received from the server.
socket.addEventListener('message', event => {
  console.log('Message from server: ', event.data);
});

// Executes when the connection is closed, providing the close code and reason.
socket.addEventListener('close', event => {
  console.log('WebSocket connection closed:', event.code, event.reason);
});

// Executes if an error occurs during the WebSocket communication.
socket.addEventListener('error', error => {
  console.error('WebSocket error:', error);
});
```

### JSONデータの送受信

```javascript
const socket = new WebSocket('ws://localhost:8080');

socket.addEventListener('open', () => {
  const data = { type: 'message', content: 'Hello from Node.js!' };
  socket.send(JSON.stringify(data));
});

socket.addEventListener('message', event => {
  try {
    const receivedData = JSON.parse(event.data);
    console.log('Received JSON:', receivedData);
  } catch (error) {
    console.error('Error parsing JSON:', error);
    console.log('Received data was:', event.data);
  }
});
```

上記の `javascript` コードは、WebSocket アプリケーションで一般的に使用される [JSON](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Scripting/JSON) データの送受信を示しています。送信前に [JSON.stringify()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify) を使用して JavaScript オブジェクトを JSON 文字列に変換します。また、受信した文字列を [JSON.parse()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse) を使用して JavaScript オブジェクトに戻します。最後に、JSON 解析のエラー処理も含まれています。

これにより、依存関係の管理が軽減され、互換性が向上します。開発者は追加の WebSocket クライアントライブラリのインストールとメンテナンスを回避できます。組み込み実装は最新の Web 標準に準拠しており、相互運用性が向上します。この機能強化は、WebSocket 通信のクライアント側に重点を置いており、Node.js が WebSocket クライアントとして機能できるようになります。

## 理解しておくべき重要な点

Node.js v22 は、ネイティブ WebSocket サーバーの実装を**提供していません**。Web ブラウザやその他のクライアントからの受信接続を受け入れる WebSocket サーバーを作成するには、[ws](https://www.npmjs.com/package/ws) や [socket.io](https://www.npmjs.com/package/socket.io) などのライブラリを使用する必要があります。つまり、Node.js は WebSocket サーバーに簡単に **接続** できるようになりましたが、WebSocket サーバーとして **動作** するには依然として外部ツールが必要です。

## まとめ

Node.js v22では、アプリケーションがWebSocketサーバーと「クライアント」としてシームレスにやり取りできるようになりましたが、Node.js内でのWebSocketサーバーの作成は依然として既存のライブラリに依存しています。開発者がNode.jsプロジェクトでリアルタイム通信を実装する際には、この違いを理解することが重要です。
