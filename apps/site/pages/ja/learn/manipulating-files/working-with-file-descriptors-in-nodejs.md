---
title: Working with file descriptors in Node.js
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, clean99, vaishnav-mk
---

# Node.js でのファイル記述子の操作

ファイルシステム内のファイルを操作するには、まずファイル記述子を取得する必要があります。

ファイル記述子は、開いているファイルへの参照であり、`fs` モジュールが提供する `open()` メソッドを使用してファイルを開いたときに返される数値 (fd) です。この数値 (`fd`) は、オペレーティングシステム内で開いているファイルを一意に識別します。

```cjs
const fs = require('node:fs');

fs.open('/Users/joe/test.txt', 'r', (err, fd) => {
  // fd is our file descriptor
});
```

```mjs
import fs from 'node:fs';

fs.open('/Users/joe/test.txt', 'r', (err, fd) => {
  // fd is our file descriptor
});
```

`fs.open()` 呼び出しの 2 番目のパラメータとして `r` を使用していることに注意してください。

このフラグは、ファイルを読み取り用に開くことを意味します。

**よく使用するその他のフラグは次のとおりです。**

| フラグ | 説明 | ファイルが存在しない場合は作成されます |
| ---- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| r+ | このフラグは、ファイルを読み書き用に開きます | ❌ |
| w+ | このフラグは、ファイルを読み書き用に開き、ストリームをファイルの先頭に配置します | ✅ |
| a | このフラグは、ファイルを書き込み用に開き、ストリームをファイルの末尾に配置します | ✅ |
| a+ | このフラグは、ファイルを読み書き用に開き、ストリームをファイルの末尾に配置します | ✅ |

コールバックでファイル記述子を渡す代わりに、ファイル記述子を返す `fs.openSync` メソッドを使用してファイルを開くこともできます。

```cjs
const fs = require('node:fs');

try {
  const fd = fs.openSync('/Users/joe/test.txt', 'r');
} catch (err) {
  console.error(err);
}
```

```mjs
import fs from 'node:fs';

try {
  const fd = fs.openSync('/Users/joe/test.txt', 'r');
} catch (err) {
  console.error(err);
}
```

ファイル記述子を取得したら、任意の方法で、`fs.close()` の呼び出しやファイルシステムを操作するその他の多くの操作など、ファイル記述子を必要とするすべての操作を実行できます。

`fs/promises` モジュールが提供する promise ベースの `fsPromises.open` メソッドを使用してファイルを開くこともできます。

`fs/promises` モジュールは、Node.js v14 以降でのみ利用可能です。v14 より前、v10 より後では、代わりに `require('fs').promises` を使用できます。v10 より前、v8 より後では、`util.promisify` を使用して `fs` メソッドを promise ベースのメソッドに変換できます。

```cjs
const fs = require('node:fs/promises');
// Or const fs = require('fs').promises before v14.
async function example() {
  let filehandle;
  try {
    filehandle = await fs.open('/Users/joe/test.txt', 'r');
    console.log(filehandle.fd);
    console.log(await filehandle.readFile({ encoding: 'utf8' }));
  } finally {
    if (filehandle) {
      await filehandle.close();
    }
  }
}
example();
```

```mjs
import fs from 'node:fs/promises';
// Or const fs = require('fs').promises before v14.
let filehandle;
try {
  filehandle = await fs.open('/Users/joe/test.txt', 'r');
  console.log(filehandle.fd);
  console.log(await filehandle.readFile({ encoding: 'utf8' }));
} finally {
  if (filehandle) {
    await filehandle.close();
  }
}
```

以下は `util.promisify` の例です。

```cjs
const fs = require('node:fs');
const util = require('node:util');

async function example() {
  const open = util.promisify(fs.open);
  const fd = await open('/Users/joe/test.txt', 'r');
}
example();
```

```mjs
import fs from 'node:fs';
import util from 'node:util';

async function example() {
  const open = util.promisify(fs.open);
  const fd = await open('/Users/joe/test.txt', 'r');
}
example();
```

`fs/promises` モジュールの詳細については、[fs/promises API](https://nodejs.org/api/fs.html#promise-example) を参照してください。