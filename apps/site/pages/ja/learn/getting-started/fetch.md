---
title: Node.js Fetch
layout: learn
authors: benhalverson, LankyMoose
---

# Node.js で Undici の Fetch API を使用する

## はじめに

[Undici](https://undici.nodejs.org) は、Node.js の fetch API を強化する HTTP クライアントライブラリです。Undici はゼロから開発されており、Node.js の組み込み HTTP クライアントに依存しません。高パフォーマンスアプリケーションに最適な機能を多数備えています。

Undici の仕様準拠については、[Undici ドキュメント](https://undici.nodejs.org/#/?id=specification-compliance-1) をご覧ください。

## Basic GET Usage

```js
async function main() {
  // Like the browser fetch API, the default method is GET
  const response = await fetch('https://jsonplaceholder.typicode.com/posts');
  const data = await response.json();
  console.log(data);
  // returns something like:
  //   {
  //   userId: 1,
  //   id: 1,
  //   title: 'sunt aut facere repellat provident occaecati excepturi optio reprehenderit',
  //   body: 'quia et suscipit\n' +
  //     'suscipit recusandae consequuntur expedita et cum\n' +
  //     'reprehenderit molestiae ut ut quas totam\n' +
  //     'nostrum rerum est autem sunt rem eveniet architecto'
  // }
}

main().catch(console.error);
```

## Basic POST Usage

```js
// Data sent from the client to the server
const body = {
  title: 'foo',
  body: 'bar',
  userId: 1,
};

async function main() {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    headers: {
      'User-Agent': 'undici-stream-example',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  const data = await response.json();
  console.log(data);
  // returns something like:
  // { title: 'foo', body: 'bar', userId: 1, id: 101 }
}

main().catch(console.error);
```

## Undici による Fetch API のカスタマイズ

Undici では、`fetch` 関数にオプションを指定することで、Fetch API をカスタマイズできます。例えば、カスタムヘッダー、リクエストメソッド、リクエストボディを設定できます。Undici で Fetch API をカスタマイズする方法の例を以下に示します。

[fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) 関数は、フェッチする URL とオプションオブジェクトの 2 つの引数を取ります。オプションオブジェクトは、リクエストをカスタマイズするために使用できる [Request](https://undici.nodejs.org/#/docs/api/Dispatcher?id=parameter-requestoptions) オブジェクトです。この関数は、[Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises) を返し、これは [Response](https://undici.nodejs.org/#/docs/api/Dispatcher?id=parameter-responsedata) オブジェクトに解決されます。

以下の例では、JSON ペイロードを含む POST リクエストを Ollama API に送信しています。Ollama は、ローカルマシンで LLM (Large Language Models) を実行できる CLI ツールです。[こちら](https://ollama.com/download) からダウンロードできます。

```bash
ollama run mistral
```

これにより、`mistral` モデルがダウンロードされ、ローカルマシン上で実行されます。

プールを使用すると、同じサーバーへの接続を再利用できるため、パフォーマンスが向上します。Undici でプールを使用する例を以下に示します。

```js
import { Pool } from 'undici';

const ollamaPool = new Pool('http://localhost:11434', {
  connections: 10,
});

/**
 * Stream the completion of a prompt using the Ollama API.
 * @param {string} prompt - The prompt to complete.
 * @link https://github.com/ollama/ollama/blob/main/docs/api.md
 **/
async function streamOllamaCompletion(prompt) {
  const { statusCode, body } = await ollamaPool.request({
    path: '/api/generate',
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ prompt, model: 'mistral' }),
  });

  // You can read about HTTP status codes here: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
  // 200 means the request was successful.
  if (statusCode !== 200) {
    // consuming the response body is mandatory: https://undici.nodejs.org/#/?id=garbage-collection
    await body.dump();
    throw new Error(`Ollama request failed with status ${statusCode}`);
  }

  let partial = '';

  const decoder = new TextDecoder();
  for await (const chunk of body) {
    partial += decoder.decode(chunk, { stream: true });
    console.log(partial);
  }

  console.log('Streaming complete.');
}

try {
  await streamOllamaCompletion('What is recursion?');
} catch (error) {
  console.error('Error calling Ollama:', error);
} finally {
  console.log('Closing Ollama pool.');
  ollamaPool.close();
}
```

## Undici を使用したレスポンスのストリーミング

[Streams](https://nodejs.org/docs/v22.14.0/api/stream.html#stream) は、Node.js の機能で、データのチャンクを読み書きできます。

```js
import { Writable } from 'node:stream';

import { stream } from 'undici';

async function fetchGitHubRepos() {
  const url = 'https://api.github.com/users/nodejs/repos';

  await stream(
    url,
    {
      method: 'GET',
      headers: {
        'User-Agent': 'undici-stream-example',
        Accept: 'application/json',
      },
    },
    res => {
      let buffer = '';

      return new Writable({
        write(chunk, encoding, callback) {
          buffer += chunk.toString();
          callback();
        },
        final(callback) {
          try {
            const json = JSON.parse(buffer);
            console.log(
              'Repository Names:',
              json.map(repo => repo.name)
            );
          } catch (error) {
            console.error('Error parsing JSON:', error);
          }
          console.log('Stream processing completed.');
          console.log(`Response status: ${res.statusCode}`);
          callback();
        },
      });
    }
  );
}

fetchGitHubRepos().catch(console.error);
```
