---
title: Anatomy of an HTTP Transaction
layout: learn
---

# HTTPトランザクションの仕組み

このガイドの目的は、Node.jsにおけるHTTP処理のプロセスをしっかりと理解してもらうことです。言語やプログラミング環境に関わらず、HTTPリクエストの仕組みを大まかに理解していることを前提としています。また、Node.jsの[`EventEmitters`][]と[`Streams`][]についてもある程度の知識があることを前提としています。
これらの機能についてよく知らない場合は、それぞれのAPIドキュメントをざっと読んでみることをお勧めします。

## サーバーの作成

Node.js の Web サーバーアプリケーションでは、いずれ Web サーバーオブジェクトを作成する必要があります。これは [`createServer`][] を使って行います。

```cjs
const http = require('node:http');

const server = http.createServer((request, response) => {
  // magic happens here!
});
```

```mjs
import http from 'node:http';

const server = http.createServer((request, response) => {
  // magic happens here!
});
```

[`createServer`][] に渡される関数は、そのサーバーに対して行われるHTTPリクエストごとに1回呼び出されるため、リクエストハンドラーと呼ばれます。実際、[`createServer`][] によって返される [`Server`][] オブジェクトは [`EventEmitter`][] であり、ここで使用しているのは、`server` オブジェクトを作成し、その後リスナーを追加するための簡略化です。

```js
const server = http.createServer();
server.on('request', (request, response) => {
  // the same kind of magic happens here!
});
```

HTTPリクエストがサーバーに到達すると、Node.jsはトランザクションを処理するための便利なオブジェクト（`request`と`response`）を使ってリクエストハンドラー関数を呼び出します。これらについては後ほど説明します。

実際にリクエストを処理するには、`server`オブジェクトの[`listen`][]メソッドを呼び出す必要があります。ほとんどの場合、`listen`に渡すのはサーバーがリッスンするポート番号だけです。他にもいくつかのオプションがありますので、[APIリファレンス][]を参照してください。

## メソッド、URL、ヘッダー

リクエストを処理する際、まず最初にすべきことは、メソッドとURLを確認して適切なアクションを実行することです。Node.jsでは、`request`オブジェクトに便利なプロパティを設定することで、この処理を比較的簡単に行うことができます。

```js
const { method, url } = request;
```

> `request` オブジェクトは [`IncomingMessage`][] のインスタンスです。

ここでの `method` は常に通常の HTTP メソッド/動詞です。`url` はサーバー、プロトコル、ポート番号を除いた完全な URL です。一般的な URL では、これは 3 番目のスラッシュ以降のすべてを指します。

ヘッダーもそれほど遠くありません。`request` 内の `headers` という独自のオブジェクトに含まれています。

```js
const { headers } = request;
const userAgent = headers['user-agent'];
```

ここで重要なのは、クライアントが実際にどのように送信したかに関わらず、すべてのヘッダーは小文字のみで表現されるということです。これにより、どのような目的であれヘッダーの解析作業が簡素化されます。

ヘッダーが重複している場合、ヘッダーに応じて、値は上書きされるか、カンマ区切りの文字列として結合されます。場合によっては、この方法が問題となる可能性があるため、[`rawHeaders`][] も利用可能です。

## リクエストボディ

`POST` または `PUT` リクエストを受信する際、リクエストボディはアプリケーションにとって重要になる場合があります。ボディデータの取得は、リクエストヘッダーへのアクセスよりも少し複雑です。ハンドラーに渡される `request` オブジェクトは [`ReadableStream`][] インターフェースを実装しています。このストリームは、他のストリームと同様に、リッスンしたり、他の場所にパイプしたりできます。ストリームの `'data'` イベントと `'end'` イベントをリッスンすることで、ストリームから直接データを取得できます。

各 `'data'` イベントで出力されるチャンクは [`Buffer`][] です。文字列データになることが分かっている場合は、データを配列にまとめて `'end'` で連結し、文字列化するのが最善策です。

```js
let body = [];
request
  .on('data', chunk => {
    body.push(chunk);
  })
  .on('end', () => {
    body = Buffer.concat(body).toString();
    // at this point, `body` has the entire request body stored in it as a string
  });
```

> これは少し面倒に思えるかもしれませんし、多くの場合、実際面倒です。幸いなことに、[`npm`][] には [`concat-stream`][] や [`body`][] といったモジュールがあり、これらのロジックの一部を隠蔽することができます。この方法を試す前に、何が起こっているのかをよく理解しておくことが重要です。だからこそ、あなたはここにいるのです！

## エラーについて

`request` オブジェクトは [`ReadableStream`][] であるため、[`EventEmitter`][] でもあり、エラー発生時には EventEmitter として動作します。

`request` ストリームでエラーが発生すると、ストリームに `'error'` イベントが発行されます。**このイベントのリスナーがない場合、エラーが _throw_ され、Node.js プログラムがクラッシュする可能性があります。** したがって、ログに記録してそのまま処理を続行する場合でも、リクエストストリームに `'error'` リスナーを追加する必要があります。(ただし、何らかの HTTP エラーレスポンスを送信するのが最善策です。これについては後ほど詳しく説明します。)

```js
request.on('error', err => {
  // This prints the error message and stack trace to `stderr`.
  console.error(err.stack);
});
```

他の抽象化やツールなど、[これらのエラーを処理する][]方法は他にもありますが、エラーは発生する可能性があり、実際に発生するので、対処する必要があることを常に認識しておく必要があります。

## これまでの内容

ここまでで、サーバーの作成と、リクエストからメソッド、URL、ヘッダー、本文を取得する方法について説明しました。これらをすべてまとめると、次のようになります。

```cjs
const http = require('node:http');

http
  .createServer((request, response) => {
    const { headers, method, url } = request;
    let body = [];
    request
      .on('error', err => {
        console.error(err);
      })
      .on('data', chunk => {
        body.push(chunk);
      })
      .on('end', () => {
        body = Buffer.concat(body).toString();
        // At this point, we have the headers, method, url and body, and can now
        // do whatever we need to in order to respond to this request.
      });
  })
  .listen(8080); // Activates this server, listening on port 8080.
```

```mjs
import http from 'node:http';

http
  .createServer((request, response) => {
    const { headers, method, url } = request;
    let body = [];
    request
      .on('error', err => {
        console.error(err);
      })
      .on('data', chunk => {
        body.push(chunk);
      })
      .on('end', () => {
        body = Buffer.concat(body).toString();
        // At this point, we have the headers, method, url and body, and can now
        // do whatever we need to in order to respond to this request.
      });
  })
  .listen(8080); // Activates this server, listening on port 8080.
```

この例を実行すると、リクエストを_受信_することはできますが、_応答_することはできません。実際、この例をWebブラウザで実行すると、クライアントに何も返されないため、リクエストはタイムアウトします。

ここまで、`response`オブジェクトについては全く触れていませんでした。これは[`ServerResponse`][]のインスタンスであり、[`WritableStream`][]です。このオブジェクトには、クライアントにデータを返送するための便利なメソッドが多数含まれています。これについては次に説明します。

## HTTPステータスコード

設定しない場合、レスポンスのHTTPステータスコードは常に200になります。もちろん、すべてのHTTPレスポンスでこの値が必要なわけではなく、場合によっては異なるステータスコードを送信したい場合もあるでしょう。そのためには、`statusCode`プロパティを設定します。

```js
response.statusCode = 404; // Tell the client that the resource wasn't found.
```

他にもショートカットがいくつかありますが、後ほど説明します。

## レスポンスヘッダーの設定

ヘッダーは、[`setHeader`][] という便利なメソッドを使って設定されます。

```js
response.setHeader('Content-Type', 'application/json');
response.setHeader('X-Powered-By', 'bacon');
```

レスポンスにヘッダーを設定する際、ヘッダー名の大文字と小文字は区別されません。
ヘッダーを繰り返し設定した場合、最後に設定した値が送信されます。

## ヘッダーデータの明示的な送信

これまで説明したヘッダーとステータスコードの設定方法は、「暗黙的なヘッダー」の使用を前提としています。つまり、ボディデータの送信を開始する前に、Node.jsが適切なタイミングでヘッダーを送信してくれることを期待しているということです。

必要であれば、レスポンスストリームにヘッダーを明示的に書き込むこともできます。
これを行うには、[`writeHead`][] というメソッドを使用します。このメソッドは、ステータスコードとヘッダーをストリームに書き込みます。

```js
response.writeHead(200, {
  'Content-Type': 'application/json',
  'X-Powered-By': 'bacon',
});
```

ヘッダーを（暗黙的または明示的に）設定したら、応答データの送信を開始する準備が整います。

## レスポンス本文の送信

`response` オブジェクトは [`WritableStream`][] であるため、レスポンス本文をクライアントに書き出すには、通常のストリームメソッドを使用するだけです。

```js
response.write('<html>');
response.write('<body>');
response.write('<h1>Hello, World!</h1>');
response.write('</body>');
response.write('</html>');
response.end();
```

ストリームの `end` 関数は、ストリームの最後のデータビットとして送信するオプションのデータも取得できるため、上記の例を次のように簡略化できます。

```js
response.end('<html><body><h1>Hello, World!</h1></body></html>');
```

> ボディにデータを書き込む前に、ステータスとヘッダーを設定することが重要です。HTTPレスポンスではヘッダーがボディよりも先に送信されるため、これは理にかなっています。

## エラーについてもう一つ注意点

`response` ストリームは `'error'` イベントを発行する可能性があり、いずれはこれにも対処する必要が出てきます。`request` ストリームのエラーに関するアドバイスはすべて、ここでも適用されます。

## すべてをまとめる

HTTPレスポンスの作成方法を学んだので、次はすべてをまとめてみましょう。
先ほどの例を基に、ユーザーから送信されたすべてのデータを返すサーバーを作成します。`JSON.stringify` を使って、データをJSON形式に変換します。

```cjs
const http = require('node:http');

http
  .createServer((request, response) => {
    const { headers, method, url } = request;
    let body = [];
    request
      .on('error', err => {
        console.error(err);
      })
      .on('data', chunk => {
        body.push(chunk);
      })
      .on('end', () => {
        body = Buffer.concat(body).toString();
        // BEGINNING OF NEW STUFF

        response.on('error', err => {
          console.error(err);
        });

        response.statusCode = 200;
        response.setHeader('Content-Type', 'application/json');
        // Note: the 2 lines above could be replaced with this next one:
        // response.writeHead(200, {'Content-Type': 'application/json'})

        const responseBody = { headers, method, url, body };

        response.write(JSON.stringify(responseBody));
        response.end();
        // Note: the 2 lines above could be replaced with this next one:
        // response.end(JSON.stringify(responseBody))

        // END OF NEW STUFF
      });
  })
  .listen(8080);
```

```mjs
import http from 'node:http';

http
  .createServer((request, response) => {
    const { headers, method, url } = request;
    let body = [];
    request
      .on('error', err => {
        console.error(err);
      })
      .on('data', chunk => {
        body.push(chunk);
      })
      .on('end', () => {
        body = Buffer.concat(body).toString();
        // BEGINNING OF NEW STUFF

        response.on('error', err => {
          console.error(err);
        });

        response.statusCode = 200;
        response.setHeader('Content-Type', 'application/json');
        // Note: the 2 lines above could be replaced with this next one:
        // response.writeHead(200, {'Content-Type': 'application/json'})

        const responseBody = { headers, method, url, body };

        response.write(JSON.stringify(responseBody));
        response.end();
        // Note: the 2 lines above could be replaced with this next one:
        // response.end(JSON.stringify(responseBody))

        // END OF NEW STUFF
      });
  })
  .listen(8080);
```

## エコーサーバーの例

前の例を簡略化して、リクエストで受信したデータをそのままレスポンスで返すシンプルなエコーサーバーを作成しましょう。必要なのは、リクエストストリームからデータを取得し、そのデータをレスポンスストリームに書き込むだけです。これは、前の例と同じです。

```cjs
const http = require('node:http');

http
  .createServer((request, response) => {
    let body = [];
    request
      .on('data', chunk => {
        body.push(chunk);
      })
      .on('end', () => {
        body = Buffer.concat(body).toString();
        response.end(body);
      });
  })
  .listen(8080);
```

```mjs
import http from 'node:http';

http
  .createServer((request, response) => {
    let body = [];
    request
      .on('data', chunk => {
        body.push(chunk);
      })
      .on('end', () => {
        body = Buffer.concat(body).toString();
        response.end(body);
      });
  })
  .listen(8080);
```

では、これを微調整してみましょう。以下の条件に該当する場合にのみ echo を送信します。

- リクエストメソッドが POST の場合。
- URL が `/echo` の場合。

それ以外の場合は、単に 404 を返します。

```cjs
const http = require('node:http');

http
  .createServer((request, response) => {
    if (request.method === 'POST' && request.url === '/echo') {
      let body = [];
      request
        .on('data', chunk => {
          body.push(chunk);
        })
        .on('end', () => {
          body = Buffer.concat(body).toString();
          response.end(body);
        });
    } else {
      response.statusCode = 404;
      response.end();
    }
  })
  .listen(8080);
```

```mjs
import http from 'node:http';

http
  .createServer((request, response) => {
    if (request.method === 'POST' && request.url === '/echo') {
      let body = [];
      request
        .on('data', chunk => {
          body.push(chunk);
        })
        .on('end', () => {
          body = Buffer.concat(body).toString();
          response.end(body);
        });
    } else {
      response.statusCode = 404;
      response.end();
    }
  })
  .listen(8080);
```

> このようにURLをチェックすることで、ある種の「ルーティング」を行っています。
> ルーティングには、`switch` 文のように単純なものから、[`express`][] のようなフレームワーク全体を使うほど複雑なものまで、様々な種類があります。ルーティングだけを行うものをお探しの場合は、[`router`][] をお試しください。

素晴らしい！では、これをもっとシンプルにしてみましょう。`request` オブジェクトは [`ReadableStream`][] で、`response` オブジェクトは [`WritableStream`][] であることを覚えておいてください。
つまり、[`pipe`][] を使って一方から他方へデータを渡すことができるということです。まさにこれがエコーサーバーに必要な機能です！

```cjs
const http = require('node:http');

http
  .createServer((request, response) => {
    if (request.method === 'POST' && request.url === '/echo') {
      request.pipe(response);
    } else {
      response.statusCode = 404;
      response.end();
    }
  })
  .listen(8080);
```

```mjs
import http from 'node:http';

http
  .createServer((request, response) => {
    if (request.method === 'POST' && request.url === '/echo') {
      request.pipe(response);
    } else {
      response.statusCode = 404;
      response.end();
    }
  })
  .listen(8080);
```

ストリーム、最高！

でも、まだ終わりではありません。このガイドで何度も触れたように、エラーは発生する可能性があり、実際に発生するので、対処する必要があります。

リクエストストリームのエラーを処理するには、エラーを `stderr` に出力し、`Bad Request` を示す 400 ステータスコードを送信します。ただし、実際のアプリケーションでは、エラーを検査して、正しいステータスコードとメッセージを特定する必要があります。エラーについては、通常どおり [`Error` ドキュメント][] を参照してください。

レスポンスでは、エラーを `stderr` に出力します。

```cjs
const http = require('node:http');

http
  .createServer((request, response) => {
    request.on('error', err => {
      console.error(err);
      response.statusCode = 400;
      response.end();
    });
    response.on('error', err => {
      console.error(err);
    });
    if (request.method === 'POST' && request.url === '/echo') {
      request.pipe(response);
    } else {
      response.statusCode = 404;
      response.end();
    }
  })
  .listen(8080);
```

```mjs
import http from 'node:http';

http
  .createServer((request, response) => {
    request.on('error', err => {
      console.error(err);
      response.statusCode = 400;
      response.end();
    });
    response.on('error', err => {
      console.error(err);
    });
    if (request.method === 'POST' && request.url === '/echo') {
      request.pipe(response);
    } else {
      response.statusCode = 404;
      response.end();
    }
  })
  .listen(8080);
```

これで、HTTP リクエスト処理の基本のほとんどを学習しました。ここまでで、以下のことができるようになっているはずです。

- リクエストハンドラ関数を使用して HTTP サーバーをインスタンス化し、ポートを listen する。
- `request` オブジェクトからヘッダー、URL、メソッド、および本文データを取得する。
- `request` オブジェクト内の URL やその他のデータに基づいてルーティングを決定する。
- `response` オブジェクトを介して、ヘッダー、HTTP ステータスコード、および本文データを送信する。
- `request` オブジェクトから `response` オブジェクトにデータをパイプする。
- `request` ストリームと `response` ストリームの両方でストリームエラーを処理する。

これらの基本から、多くの一般的なユースケースに対応する Node.js HTTP サーバーを構築できます。これらの API は他にも多くの機能を提供しているので、[`EventEmitters`][]、[`Streams`][]、[`HTTP`][] の API ドキュメントを必ずお読みください。

[`EventEmitters`]: https://nodejs.org/api/events.html
[`Streams`]: https://nodejs.org/api/stream.html
[`createServer`]: https://nodejs.org/api/http.html#http_http_createserver_requestlistener
[`Server`]: https://nodejs.org/api/http.html#http_class_http_server
[`listen`]: https://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback
[API reference]: https://nodejs.org/api/http.html
[`IncomingMessage`]: https://nodejs.org/api/http.html#http_class_http_incomingmessage
[`ReadableStream`]: https://nodejs.org/api/stream.html#stream_class_stream_readable
[`rawHeaders`]: https://nodejs.org/api/http.html#http_message_rawheaders
[`Buffer`]: https://nodejs.org/api/buffer.html
[`concat-stream`]: https://www.npmjs.com/package/concat-stream
[`body`]: https://www.npmjs.com/package/body
[`npm`]: https://www.npmjs.com
[`EventEmitter`]: https://nodejs.org/api/events.html#events_class_eventemitter
[handling these errors]: https://nodejs.org/api/errors.html
[`ServerResponse`]: https://nodejs.org/api/http.html#http_class_http_serverresponse
[`setHeader`]: https://nodejs.org/api/http.html#http_response_setheader_name_value
[`WritableStream`]: https://nodejs.org/api/stream.html#stream_class_stream_writable
[`writeHead`]: https://nodejs.org/api/http.html#http_response_writehead_statuscode_statusmessage_headers
[`express`]: https://www.npmjs.com/package/express
[`router`]: https://www.npmjs.com/package/router
[`pipe`]: https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options
[`Error` documentation]: https://nodejs.org/api/errors.html
[`HTTP`]: https://nodejs.org/api/http.html
