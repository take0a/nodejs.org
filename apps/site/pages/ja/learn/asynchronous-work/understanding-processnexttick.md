---
title: Understanding process.nextTick()
layout: learn
authors: flaviocopes, MylesBorins, LaRuaNa, ahmadawais, ovflowd, marksist300
---

# process.nextTick() を理解する

Node.js のイベントループを理解する上で、重要な部分の一つが `process.nextTick()` です。
ランタイムが JavaScript にイベントをコールバックするたびに、これを「ティック」と呼びます。

`process.nextTick()` に関数を渡すと、現在の操作が完了した直後に、イベントループの次のフェーズに進む前に、この関数を呼び出すようにエンジンに指示します。

```js
process.nextTick(() => {
  // do something
});
```

イベントループは現在の関数コードを処理中です。この処理が終了すると、JS エンジンはその処理中に `nextTick` 呼び出しに渡されたすべての関数を実行します。

これは、JS エンジンに関数を非同期的に（現在の関数の後に）処理するよう指示する方法ですが、キューに入れるのではなく、できるだけ早く処理するように指示できます。

`setTimeout(() => {}, 0)` を呼び出すと、関数は次のティックの終了時に実行されます。これは、呼び出しを優先して次のティックの開始直前に実行する `nextTick()` を使用する場合よりも大幅に遅くなります。

次のイベントループの反復処理でそのコードが既に実行されていることを確認したい場合は、`nextTick()` を使用します。

実行順序とイベントループの仕組みの詳細については、[関連記事](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) をご覧ください。
