---
title: Reading files with Node.js
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, clean99, benhalverson
---

# Node.js でファイルを読み込む

Node.js でファイルを読み込む最も簡単な方法は、`fs.readFile()` メソッドにファイルパス、エンコーディング、そしてファイルデータ（およびエラー）を引数として呼び出されるコールバック関数を渡すことです。

```cjs
const fs = require('node:fs');

fs.readFile('/Users/joe/test.txt', 'utf8', (err, data) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(data);
});
```

```mjs
import fs from 'node:fs';

fs.readFile('/Users/joe/test.txt', 'utf8', (err, data) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(data);
});
```

あるいは、同期バージョンの `fs.readFileSync()` を使用することもできます。

```cjs
const fs = require('node:fs');

try {
  const data = fs.readFileSync('/Users/joe/test.txt', 'utf8');
  console.log(data);
} catch (err) {
  console.error(err);
}
```

```mjs
import fs from 'node:fs';

try {
  const data = fs.readFileSync('/Users/joe/test.txt', 'utf8');
  console.log(data);
} catch (err) {
  console.error(err);
}
```

`fs/promises` モジュールが提供する promise ベースの `fsPromises.readFile()` メソッドを使用することもできます。

```cjs
const fs = require('node:fs/promises');

async function example() {
  try {
    const data = await fs.readFile('/Users/joe/test.txt', { encoding: 'utf8' });
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
example();
```

```mjs
import fs from 'node:fs/promises';

try {
  const data = await fs.readFile('/Users/joe/test.txt', { encoding: 'utf8' });
  console.log(data);
} catch (err) {
  console.error(err);
}
```

`fs.readFile()`、`fs.readFileSync()`、`fsPromises.readFile()` の 3 つはすべて、データを返す前にファイルの内容全体をメモリ上で読み取ります。

つまり、大きなファイルはメモリ消費量とプログラムの実行速度に大きな影響を与えます。

このような場合、ストリームを使用してファイルの内容を読み取る方が適切です。

```mjs
import fs from 'fs';
import { pipeline } from 'node:stream/promises';
import path from 'path';

const fileUrl = 'https://www.gutenberg.org/files/2701/2701-0.txt';
const outputFilePath = path.join(process.cwd(), 'moby.md');

async function downloadFile(url, outputPath) {
  const response = await fetch(url);

  if (!response.ok || !response.body) {
    // consuming the response body is mandatory: https://undici.nodejs.org/#/?id=garbage-collection
    await response.body?.cancel();
    throw new Error(`Failed to fetch ${url}. Status: ${response.status}`);
  }

  const fileStream = fs.createWriteStream(outputPath);
  console.log(`Downloading file from ${url} to ${outputPath}`);

  await pipeline(response.body, fileStream);
  console.log('File downloaded successfully');
}

async function readFile(filePath) {
  const readStream = fs.createReadStream(filePath, { encoding: 'utf8' });

  try {
    for await (const chunk of readStream) {
      console.log('--- File chunk start ---');
      console.log(chunk);
      console.log('--- File chunk end ---');
    }
    console.log('Finished reading the file.');
  } catch (error) {
    console.error(`Error reading file: ${error.message}`);
  }
}

try {
  await downloadFile(fileUrl, outputFilePath);
  await readFile(outputFilePath);
} catch (error) {
  console.error(`Error: ${error.message}`);
}
```
