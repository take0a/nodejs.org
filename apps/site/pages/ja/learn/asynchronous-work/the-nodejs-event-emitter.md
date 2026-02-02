---
title: The Node.js Event emitter
layout: learn
authors: flaviocopes, MylesBorins, fhemberger, LaRuaNa, ahmadawais, ovflowd
---

# Node.js イベントエミッター

ブラウザでJavaScriptを扱ったことがある方なら、マウスのクリック、キーボードのボタンの押下、マウスの動きへの反応など、ユーザーインタラクションの多くがイベントによって処理されていることをご存知でしょう。

バックエンド側では、Node.jsは[`events`モジュール](https://nodejs.org/api/events.html)を使用して同様のシステムを構築するオプションを提供しています。

このモジュールは、イベント処理に使用する`EventEmitter`クラスを提供しています。

これを初期化するには、以下を使用します。

```cjs
const EventEmitter = require('node:events');

const eventEmitter = new EventEmitter();
```

```mjs
import EventEmitter from 'node:events';

const eventEmitter = new EventEmitter();
```

このオブジェクトは、`on` メソッドや `emit` メソッドなど、多くのメソッドを公開しています。

- `emit` はイベントをトリガーするために使用されます。
- `on` は、イベントがトリガーされたときに実行されるコールバック関数を追加するために使用されます。

例えば、`start` イベントを作成し、サンプルとして、コンソールにログを出力するだけでイベントに反応してみましょう。

```js
eventEmitter.on('start', () => {
  console.log('started');
});
```

実行すると

```js
eventEmitter.emit('start');
```

イベントハンドラー関数がトリガーされ、コンソールログが出力されます。

`emit()` に追加の引数として渡すことで、イベントハンドラーに引数を渡すことができます。

```js
eventEmitter.on('start', number => {
  console.log(`started ${number}`);
});

eventEmitter.emit('start', 23);
```

複数の引数:

```js
eventEmitter.on('start', (start, end) => {
  console.log(`started from ${start} to ${end}`);
});

eventEmitter.emit('start', 1, 100);
```

EventEmitter オブジェクトは、イベントを操作するための他のメソッドもいくつか公開しています。

- `once()`: 1回限りのリスナーを追加します。
- `removeListener()` / `off()`: イベントからイベントリスナーを削除します。
- `removeAllListeners()`: イベントのすべてのリスナーを削除します。

これらのメソッドの詳細については、[公式ドキュメント](https://nodejs.org/api/events.html) をご覧ください。
