---
title: Node.js File Paths
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, amiller-gh, ahmadawais
---

# Node.js のファイルパス

システム内のすべてのファイルにはパスがあります。Linux や macOS ではパスは `/users/joe/file.txt` のようになりますが、Windows では `C:\users\joe\file.txt` のような構造になります。

アプリケーションでパスを使用する際は、この違いを考慮する必要があるため、注意が必要です。

`const path = require('node:path');` を使用してこのモジュールをファイルにインクルードすると、そのメソッドを使用できるようになります。

## パスから情報を取得する

パスを指定すると、以下のメソッドを使って情報を抽出できます。

- `dirname`: ファイルの親フォルダを取得します。
- `basename`: ファイル名部分を取得します。
- `extname`: ファイル拡張子を取得します。

### 例

```cjs
const path = require('node:path');

const notes = '/users/joe/notes.txt';

path.dirname(notes); // /users/joe
path.basename(notes); // notes.txt
path.extname(notes); // .txt
```

```mjs
import path from 'node:path';

const notes = '/users/joe/notes.txt';

path.dirname(notes); // /users/joe
path.basename(notes); // notes.txt
path.extname(notes); // .txt
```

`basename` に 2 番目の引数を指定すると、拡張子のないファイル名を取得できます。

```js
path.basename(notes, path.extname(notes)); // notes
```

## パスの操作

`path.join()` を使用すると、パスの2つ以上の部分を結合できます。

```js
const name = 'joe';
path.join('/', 'users', name, 'notes.txt'); // '/users/joe/notes.txt'
```

`path.resolve()` を使用して相対パスから絶対パスへの変換結果を取得できます。

```js
path.resolve('joe.txt'); // '/Users/joe/joe.txt' if run from my home folder
```

この場合、Node.jsは単に`/joe.txt`を現在の作業ディレクトリに追加します。2番目のパラメータでフォルダを指定した場合、`resolve`は最初のフォルダを2番目のフォルダのベースとして使用します。

```js
path.resolve('tmp', 'joe.txt'); // '/Users/joe/tmp/joe.txt' if run from my home folder
```

最初のパラメータがスラッシュで始まる場合、それは絶対パスであることを意味します。

```js
path.resolve('/etc', 'joe.txt'); // '/etc/joe.txt'
```

`path.normalize()` は、`.` や `..` などの相対指定子や二重スラッシュが含まれている場合に、実際のパスを計算しようとするもう 1 つの便利な関数です。

```js
path.normalize('/users/joe/..//test.txt'); // '/users/test.txt'
```

**resolve も normalize もパスが存在するかどうかを確認しません**。取得した情報に基づいてパスを計算するだけです。
