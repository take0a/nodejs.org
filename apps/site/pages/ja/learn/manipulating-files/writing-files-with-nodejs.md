---
title: Writing files with Node.js
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, clean99, ovflowd, vaishnav-mk
---

# Node.js でファイルを書き込む

## ファイルを書き込む

Node.js でファイルに書き込む最も簡単な方法は、`fs.writeFile()` API を使用することです。

```js
const fs = require('node:fs');

const content = 'Some content!';

fs.writeFile('/Users/joe/test.txt', content, err => {
  if (err) {
    console.error(err);
  } else {
    // file written successfully
  }
});
```

### ファイルの同期書き込み

代わりに、同期バージョンの `fs.writeFileSync()` を使用することもできます。

```js
const fs = require('node:fs');

const content = 'Some content!';

try {
  fs.writeFileSync('/Users/joe/test.txt', content);
  // file written successfully
} catch (err) {
  console.error(err);
}
```

`fs/promises` モジュールが提供する promise ベースの `fsPromises.writeFile()` メソッドを使用することもできます。

```js
const fs = require('node:fs/promises');

async function example() {
  try {
    const content = 'Some content!';
    await fs.writeFile('/Users/joe/test.txt', content);
  } catch (err) {
    console.log(err);
  }
}

example();
```

デフォルトでは、この API は **ファイルの内容を置き換えます**（ファイルが既に存在する場合）。

**フラグを指定することでデフォルトを変更できます。**

```js
fs.writeFile('/Users/joe/test.txt', content, { flag: 'a+' }, err => {});
```

#### おそらく使用するフラグは以下のとおりです。

| フラグ | 説明 | ファイルが存在しない場合は作成されます |
| ---- | -------------------------------------------------------------------------------------------------------------------------------- | :-----------------------------------: |
| `r+` | このフラグはファイルを **読み取り** と **書き込み** 用に開きます | ❌ |
| `w+` | このフラグはファイルを **読み取り** と **書き込み** 用に開き、ストリームをファイルの **先頭** に配置します | ✅ |
| `a` | このフラグはファイルを **書き込み** 用に開き、ストリームをファイルの **末尾** に配置します | ✅ |
| `a+` | このフラグはファイルを **読み取り** と **書き込み** 用に開き、ストリームをファイルの **末尾** に配置します | ✅ |

- フラグの詳細については、[fs ドキュメント](https://nodejs.org/api/fs.html#file-system-flags) を参照してください。

## ファイルへのコンテンツの追加

ファイルへのコンテンツの追加は、ファイルを新しいコンテンツで上書きするのではなく、追加したい場合に便利です。

### 例

ファイルの末尾にコンテンツを追加する便利なメソッドは、`fs.appendFile()` (および対応する `fs.appendFileSync()`) です。

```js
const fs = require('node:fs');

const content = 'Some content!';

fs.appendFile('file.log', content, err => {
  if (err) {
    console.error(err);
  } else {
    // done!
  }
});
```

#### Promise の例

`fsPromises.appendFile()` の例を以下に示します。

```js
const fs = require('node:fs/promises');

async function example() {
  try {
    const content = 'Some content!';
    await fs.appendFile('/Users/joe/test.txt', content);
  } catch (err) {
    console.log(err);
  }
}

example();
```
