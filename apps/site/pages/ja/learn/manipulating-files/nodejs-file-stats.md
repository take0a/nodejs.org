---
title: Node.js file stats
layout: learn
authors: flaviocopes, ZYSzys, MylesBorins, fhemberger, LaRuaNa, ahmadawais, clean99, ovflowd, vaishnav-mk
---

# Node.js のファイル情報

すべてのファイルには、Node.js を使って調べることができる詳細情報が付属しています。特に、[`fs` モジュール](https://nodejs.org/api/fs.html) が提供する `stat()` メソッドを使用すると便利です。

ファイルパスを渡してこのメ​​ソッドを呼び出すと、Node.js はファイルの詳細を取得すると、渡されたコールバック関数を呼び出します。この関数には、エラーメッセージとファイル情報の 2 つのパラメータが渡されます。

```cjs
const fs = require('node:fs');

fs.stat('/Users/joe/test.txt', (err, stats) => {
  if (err) {
    console.error(err);
  }
  // we have access to the file stats in `stats`
});
```

```mjs
import fs from 'node:fs';

fs.stat('/Users/joe/test.txt', (err, stats) => {
  if (err) {
    console.error(err);
  }
  // we have access to the file stats in `stats`
});
```

Node.js には、ファイルの情報が準備されるまでスレッドをブロックする sync メソッドも用意されています。

```cjs
const fs = require('node:fs');

try {
  const stats = fs.statSync('/Users/joe/test.txt');
} catch (err) {
  console.error(err);
}
```

```mjs
import fs from 'node:fs';

try {
  const stats = fs.statSync('/Users/joe/test.txt');
} catch (err) {
  console.error(err);
}
```

ファイル情報は stats 変数に含まれています。この stats 変数を使ってどのような情報を抽出できるでしょうか？

**多くの情報を取得できます。例えば、以下の情報です。**

- ファイルがディレクトリかファイルかを確認するには、`stats.isFile()` と `stats.isDirectory()` を使用します。
- ファイルがシンボリックリンクかを確認するには、`stats.isSymbolicLink()` を使用します。
- ファイルサイズ（バイト単位）を確認するには、`stats.size` を使用します。

他にも高度なメソッドはありますが、日常のプログラミングで主に使うのはこのメソッドです。

```cjs
const fs = require('node:fs');

fs.stat('/Users/joe/test.txt', (err, stats) => {
  if (err) {
    console.error(err);
    return;
  }

  stats.isFile(); // true
  stats.isDirectory(); // false
  stats.isSymbolicLink(); // false
  console.log(stats.size); // 1024000 //= 1MB
});
```

```mjs
import fs from 'node:fs';

fs.stat('/Users/joe/test.txt', (err, stats) => {
  if (err) {
    console.error(err);
    return;
  }

  stats.isFile(); // true
  stats.isDirectory(); // false
  stats.isSymbolicLink(); // false
  console.log(stats.size); // 1024000 //= 1MB
});
```

必要に応じて、`fs/promises` モジュールが提供する promise ベースの `fsPromises.stat()` メソッドを使用することもできます。

```cjs
const fs = require('node:fs/promises');

async function example() {
  try {
    const stats = await fs.stat('/Users/joe/test.txt');
    stats.isFile(); // true
    stats.isDirectory(); // false
    stats.isSymbolicLink(); // false
    console.log(stats.size); // 1024000 //= 1MB
  } catch (err) {
    console.log(err);
  }
}
example();
```

```mjs
import fs from 'node:fs/promises';

try {
  const stats = await fs.stat('/Users/joe/test.txt');
  stats.isFile(); // true
  stats.isDirectory(); // false
  stats.isSymbolicLink(); // false
  console.log(stats.size); // 1024000 //= 1MB
} catch (err) {
  console.log(err);
}
```

`fs` モジュールの詳細については、[公式ドキュメント](https://nodejs.org/api/fs.html) を参照してください。
