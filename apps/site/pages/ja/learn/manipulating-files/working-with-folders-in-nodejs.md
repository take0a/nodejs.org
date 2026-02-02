---
title: Working with folders in Node.js
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, liangpeili, LaRuaNa, ahmadawais, clean99
---

# Node.js でのフォルダ操作

Node.js の `fs` コアモジュールには、フォルダ操作に使用できる便利なメソッドが多数用意されています。

## フォルダの存在を確認する

`fs.access()`（およびそれに対応する promise ベースの `fsPromises.access()`）を使用して、フォルダが存在し、Node.js がその権限でアクセスできるかどうかを確認します。

## 新しいフォルダを作成する

`fs.mkdir()`、`fs.mkdirSync()`、または `fsPromises.mkdir()` を使用して、新しいフォルダを作成します。

```cjs
const fs = require('node:fs');

const folderName = '/Users/joe/test';

try {
  if (!fs.existsSync(folderName)) {
    fs.mkdirSync(folderName);
  }
} catch (err) {
  console.error(err);
}
```

```mjs
import fs from 'node:fs';

const folderName = '/Users/joe/test';

try {
  if (!fs.existsSync(folderName)) {
    fs.mkdirSync(folderName);
  }
} catch (err) {
  console.error(err);
}
```

## ディレクトリの内容を読み取ります

ディレクトリの内容を読み取るには、`fs.readdir()`、`fs.readdirSync()`、または `fsPromises.readdir()` を使用します。

このコードは、フォルダの内容（ファイルとサブフォルダの両方）を読み取り、それらの相対パスを返します。

```cjs
const fs = require('node:fs');

const folderPath = '/Users/joe';

fs.readdirSync(folderPath);
```

```mjs
import fs from 'node:fs';

const folderPath = '/Users/joe';

fs.readdirSync(folderPath);
```

完全なパスを取得できます:

```js
fs.readdirSync(folderPath).map(fileName => {
  return path.join(folderPath, fileName);
});
```

結果をフィルタリングして、ファイルのみを返し、フォルダーを除外することもできます。

```cjs
const fs = require('node:fs');

const isFile = fileName => {
  return fs.lstatSync(fileName).isFile();
};

fs.readdirSync(folderPath)
  .map(fileName => {
    return path.join(folderPath, fileName);
  })
  .filter(isFile);
```

```mjs
import fs from 'node:fs';

const isFile = fileName => {
  return fs.lstatSync(fileName).isFile();
};

fs.readdirSync(folderPath)
  .map(fileName => {
    return path.join(folderPath, fileName);
  })
  .filter(isFile);
```

## フォルダ名を変更する

フォルダ名を変更するには、`fs.rename()`、`fs.renameSync()`、または`fsPromises.rename()`を使用します。最初の引数は現在のパス、2番目の引数は新しいパスです。

```cjs
const fs = require('node:fs');

fs.rename('/Users/joe', '/Users/roger', err => {
  if (err) {
    console.error(err);
  }
  // done
});
```

```mjs
import fs from 'node:fs';

fs.rename('/Users/joe', '/Users/roger', err => {
  if (err) {
    console.error(err);
  }
  // done
});
```

`fs.renameSync()` は同期バージョンです。

```cjs
const fs = require('node:fs');

try {
  fs.renameSync('/Users/joe', '/Users/roger');
} catch (err) {
  console.error(err);
}
```

```mjs
import fs from 'node:fs';

try {
  fs.renameSync('/Users/joe', '/Users/roger');
} catch (err) {
  console.error(err);
}
```

`fsPromises.rename()` は promise ベースのバージョンです。

```cjs
const fs = require('node:fs/promises');

async function example() {
  try {
    await fs.rename('/Users/joe', '/Users/roger');
  } catch (err) {
    console.log(err);
  }
}
example();
```

```mjs
import fs from 'node:fs/promises';

try {
  await fs.rename('/Users/joe', '/Users/roger');
} catch (err) {
  console.log(err);
}
```

## フォルダを削除する

フォルダを削除するには、`fs.rmdir()`、`fs.rmdirSync()`、または `fsPromises.rmdir()` を使用します。

```cjs
const fs = require('node:fs');

fs.rmdir(dir, err => {
  if (err) {
    throw err;
  }

  console.log(`${dir} is deleted!`);
});
```

```mjs
import fs from 'node:fs';

fs.rmdir(dir, err => {
  if (err) {
    throw err;
  }

  console.log(`${dir} is deleted!`);
});
```

内容のあるフォルダを削除するには、`fs.rm()` に `{ recursive: true }` オプションを指定して、内容を再帰的に削除します。

`{ recursive: true, force: true }` を指定すると、フォルダが存在しない場合に例外が無視されます。

```cjs
const fs = require('node:fs');

fs.rm(dir, { recursive: true, force: true }, err => {
  if (err) {
    throw err;
  }

  console.log(`${dir} is deleted!`);
});
```

```mjs
import fs from 'node:fs';

fs.rm(dir, { recursive: true, force: true }, err => {
  if (err) {
    throw err;
  }

  console.log(`${dir} is deleted!`);
});
```
