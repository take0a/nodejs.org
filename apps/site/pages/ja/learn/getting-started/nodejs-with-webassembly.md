---
title: Node.js with WebAssembly
layout: learn
authors: lancemccluskey, ovflowd
---

# Node.js と WebAssembly

**[WebAssembly](https://webassembly.org)** は、C/C++、Rust、AssemblyScript など、様々な言語からコンパイル可能な、高性能なアセンブリ言語です。現在、Chrome、Firefox、Safari、Edge、Node.js でサポートされています。

WebAssembly 仕様では、2 つのファイル形式が規定されています。1 つは WebAssembly モジュールと呼ばれるバイナリ形式で、拡張子は `.wasm` です。もう 1 つは WebAssembly テキスト形式で、拡張子は `.wat` です。

## 主要な概念

- モジュール - コンパイル済みの WebAssembly バイナリ（`.wasm` ファイル）。
- メモリ - サイズ変更可能な ArrayBuffer。
- テーブル - メモリに格納されていない参照の、サイズ変更可能な型付き配列。
- インスタンス - メモリ、テーブル、および変数を含むモジュールのインスタンス化。

WebAssembly を使用するには、`.wasm` バイナリファイルと、WebAssembly と通信するための一連の API が必要です。Node.js は、グローバル `WebAssembly` オブジェクトを通じて必要な API を提供します。

```js
console.log(WebAssembly);
/*
Object [WebAssembly] {
  compile: [Function: compile],
  validate: [Function: validate],
  instantiate: [Function: instantiate]
}
*/
```

## WebAssembly モジュールの生成

WebAssembly バイナリファイルを生成するには、以下の複数の方法があります。

- WebAssembly (`.wat`) を手動で記述し、[wabt](https://github.com/webassembly/wabt) などのツールを使用してバイナリ形式に変換する
- C/C++ アプリケーションで [emscripten](https://emscripten.org/) を使用する
- Rust アプリケーションで [wasm-pack](https://rustwasm.github.io/wasm-pack/book/) を使用する
- TypeScript のような操作性を好む場合は [AssemblyScript](https://www.assemblyscript.org/) を使用する

> これらのツールの中には、バイナリファイルだけでなく、ブラウザで実行するための JavaScript の「グルー」コードと対応する HTML ファイルも生成するものがあります。

## 使い方

WebAssembly モジュールを作成したら、Node.js の `WebAssembly` オブジェクトを使ってインスタンス化できます。

```js
// Assume add.wasm file exists that contains a single function adding 2 provided arguments
const fs = require('node:fs');

// Use the readFileSync function to read the contents of the "add.wasm" file
const wasmBuffer = fs.readFileSync('/path/to/add.wasm');

// Use the WebAssembly.instantiate method to instantiate the WebAssembly module
WebAssembly.instantiate(wasmBuffer).then(wasmModule => {
  // Exported function lives under instance.exports object
  const { add } = wasmModule.instance.exports;
  const sum = add(5, 6);
  console.log(sum); // Outputs: 11
});
```

```mjs
// Assume add.wasm file exists that contains a single function adding 2 provided arguments
import fs from 'node:fs/promises';

// Use readFile to read contents of the "add.wasm" file
const wasmBuffer = await fs.readFile('/path/to/add.wasm');

// Use the WebAssembly.instantiate method to instantiate the WebAssembly module
const wasmModule = await WebAssembly.instantiate(wasmBuffer);

// Exported function lives under instance.exports object
const { add } = wasmModule.instance.exports;

const sum = add(5, 6);

console.log(sum); // Outputs: 11
```

## OS とのやり取り

WebAssembly モジュールは、単体では OS 機能に直接アクセスできません。この機能にアクセスするには、サードパーティ製ツール [Wasmtime](https://docs.wasmtime.dev/) を使用します。`Wasmtime` は [WASI](https://wasi.dev/) API を利用して OS 機能にアクセスします。

## リソース

- [WebAssembly の一般情報](https://webassembly.org/)
- [MDN ドキュメント](https://developer.mozilla.org/en-US/docs/WebAssembly)
- [WebAssembly を手書きする](https://webassembly.github.io/spec/core/text/index.html)
