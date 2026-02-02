---
title: Collecting code coverage in Node.js
layout: learn
authors: avivkeller
---

# Node.js でのコードカバレッジの収集

Node.js は、テストランナーを通じてコードカバレッジの組み込みサポートを提供しており、[`--experimental-test-coverage`](https://nodejs.org/api/cli.html#--experimental-test-coverage) フラグを使用して有効にできます。

`run()` API を使用する場合は、`coverage` オプションを `true` に設定する必要があります。`run()` API の詳細については、[`node:test` ドキュメント](https://nodejs.org/docs/latest/api/test.html#runoptions) を参照してください。

## コードカバレッジとは？

コードカバレッジは、テストランナーがテスト中にプログラムのソースコードがどの程度実行されたかを測定する指標です。コードベースのどの部分がテストされ、どの部分がテストされていないかを明らかにし、テストスイートのギャップを特定するのに役立ちます。これにより、ソフトウェアのテストをより包括的に実施でき、未検出のバグのリスクを最小限に抑えることができます。コードカバレッジは通常パーセンテージで表され、高いコードカバレッジ率ほど、より徹底したテストカバレッジを示します。コードカバレッジの詳細な説明については、Wikipediaの[「コードカバレッジ」の記事](https://en.wikipedia.org/wiki/Code_coverage)をご覧ください。

## 基本的なカバレッジレポート

Node.js におけるコードカバレッジの仕組みを、簡単な例を使って見ていきましょう。

> **注:** この例とこのファイル内の他のすべての例は、CommonJS を使用して記述されています。この概念に馴染みがない場合は、[CommonJS モジュール](https://nodejs.org/docs/latest/api/modules.html) のドキュメントをご覧ください。

```cjs displayName="main.js"
function add(a, b) {
  return a + b;
}

function isEven(num) {
  return num % 2 === 0;
}

function multiply(a, b) {
  return a * b;
}

module.exports = { add, isEven, multiply };
```

```cjs displayName="main.test.js"
const { test } = require('node:test');

const { add, isEven } = require('./main.js');

test('add() should add two numbers', t => {
  t.assert.strictEqual(add(1, 2), 3);
});

test('isEven() should report whether a number is even', t => {
  t.assert.ok(isEven(0));
});
```

このモジュールには、`add`、`isEven`、`multiply` という3つの関数があります。

テストファイルでは、`add()` 関数と `isEven()` 関数をテストしています。`multiply()` 関数はどのテストにも含まれていないことに注意してください。

テスト実行中にコードカバレッジを収集するには、次のスニペットを参照してください。

```bash displayName="CLI"
node --experimental-test-coverage --test main.test.js
```

```js displayName="run()"
run({ files: ['main.test.js'], coverage: true });
```

テストを実行すると、次のようなレポートが表示されます。

```text displayName="Coverage Report"
✔ add() should add two numbers (1.505987ms)
✔ isEven() should report whether a number is even (0.175859ms)
ℹ tests 2
ℹ suites 0
ℹ pass 2
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 59.480373
ℹ start of coverage report
ℹ -------------------------------------------------------------
ℹ file         | line % | branch % | funcs % | uncovered lines
ℹ -------------------------------------------------------------
ℹ main.js      |  76.92 |   100.00 |   66.67 | 9-11
ℹ main.test.js | 100.00 |   100.00 |  100.00 |
ℹ -------------------------------------------------------------
ℹ all files    |  86.96 |   100.00 |   80.00 |
ℹ -------------------------------------------------------------
ℹ end of coverage report
```

カバレッジレポートは、テストでカバーされたコードの割合の内訳を示します。

- **行カバレッジ**: テスト中に実行された行の割合。
- **分岐カバレッジ**: テストされたコード分岐（if-else 文など）の割合。
- **関数カバレッジ**: テスト中に呼び出された関数の割合。

この例では、次のようになります。

- `main.js` は、`multiply()` 関数がテストされていないため、行カバレッジが 76.92%、関数カバレッジが 66.67% です。カバーされていない行 (9～11) はこの関数に対応しています。
- `main.test.js` は、すべてのメトリックで 100% のカバレッジを示しており、テスト自体が完全に実行されたことを示しています。

## 含める、除外する

アプリケーションを開発しているときに、特定のファイルやコード行を除外する必要がある状況に遭遇することがあります。

Node.js は、コメントを使用して特定のコードセクションを無視したり、CLI を使用してパターン全体を除外したりするなど、これを処理するためのメカニズムを提供しています。

### コメントの使用

```cjs displayName="main.js"
function add(a, b) {
  return a + b;
}

function isEven(num) {
  return num % 2 === 0;
}

/* node:coverage ignore next 3 */
function multiply(a, b) {
  return a * b;
}

module.exports = { add, isEven, multiply };
```

```text displayName="Coverage Report"
✔ add() should add two numbers (1.430634ms)
✔ isEven() should report whether a number is even (0.202118ms)
ℹ tests 2
ℹ suites 0
ℹ pass 2
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 60.507104
ℹ start of coverage report
ℹ -------------------------------------------------------------
ℹ file         | line % | branch % | funcs % | uncovered lines
ℹ -------------------------------------------------------------
ℹ main.js      | 100.00 |   100.00 |  100.00 |
ℹ main.test.js | 100.00 |   100.00 |  100.00 |
ℹ -------------------------------------------------------------
ℹ all files    | 100.00 |   100.00 |  100.00 |
ℹ -------------------------------------------------------------
ℹ end of coverage report
```

この修正された `main.js` ファイルでカバレッジをレポートすると、すべてのメトリックで 100% のカバレッジが表示されます。これは、カバーされていない行 (9～11) が無視されているためです。

コメントを使用してコードの一部を無視する方法は複数あります。

```cjs displayName="ignore next"
function add(a, b) {
  return a + b;
}

function isEven(num) {
  return num % 2 === 0;
}

/* node:coverage ignore next 3 */
function multiply(a, b) {
  return a * b;
}

module.exports = { add, isEven, multiply };
```

```cjs displayName="ignore next"
function add(a, b) {
  return a + b;
}

function isEven(num) {
  return num % 2 === 0;
}

/* node:coverage ignore next */
function multiply(a, b) {
  /* node:coverage ignore next */
  return a * b;
  /* node:coverage ignore next */
}

module.exports = { add, isEven, multiply };
```

```cjs displayName="disable"
function add(a, b) {
  return a + b;
}

function isEven(num) {
  return num % 2 === 0;
}

/* node:coverage disable */
function multiply(a, b) {
  return a * b;
}
/* node:coverage enable */

module.exports = { add, isEven, multiply };
```

これらの異なる方法はいずれも、すべてのメトリクスで 100% のコードカバレッジを持つ同じレポートを生成します。

### CLI の使用

Node.js には、カバレッジレポートに特定のファイルを含めるか除外するかを管理するための 2 つの CLI 引数が用意されています。

[`--test-coverage-include`](https://nodejs.org/api/cli.html#--test-coverage-include) フラグ（`run()` API の `coverageIncludeGlobs`）は、指定された glob パターンに一致するファイルのみにカバレッジを制限します。デフォルトでは `/node_modules/` ディレクトリ内のファイルは除外されますが、このフラグを使用することで明示的に含めることができます。

[`--test-coverage-exclude`](https://nodejs.org/api/cli.html#--test-coverage-exclude) フラグ（`run()` API の `coverageExcludeGlobs`）は、指定された glob パターンに一致するファイルをカバレッジレポートから除外します。

これらのフラグは複数回使用できます。両方を一緒に使用する場合、ファイルは包含ルールに準拠すると同時に除外ルールも回避する必要があります。

```text displayName="Directory Structure"
.
├── main.test.js
├── src
│   ├── age.js
│   └── name.js
```

```text displayName="Coverage Report"
ℹ start of coverage report
ℹ -------------------------------------------------------------
ℹ file         | line % | branch % | funcs % | uncovered lines
ℹ -------------------------------------------------------------
ℹ main.test.js | 100.00 |   100.00 |  100.00 |
ℹ src/age.js   |  45.45 |   100.00 |    0.00 | 3-5 7-9
ℹ src/name.js  | 100.00 |   100.00 |  100.00 |
ℹ -------------------------------------------------------------
ℹ all files    |  88.68 |   100.00 |   75.00 |
ℹ -------------------------------------------------------------
ℹ end of coverage report
```

上記のレポートでは `src/age.js` のカバレッジは最適とは言えませんが、`--test-coverage-exclude` フラグ (`run()` API の `coverageExcludeGlobs`) を使用すると、レポートから完全に除外できます。

```bash displayName="CLI"
node --experimental-test-coverage --test-coverage-exclude=src/age.js --test main.test.js
```

```js displayName="run()"
run({
  files: ['main.test.js'],
  coverage: true,
  coverageExclude: ['src/age.js'],
});
```

```text displayName="New coverage report"
ℹ start of coverage report
ℹ -------------------------------------------------------------
ℹ file         | line % | branch % | funcs % | uncovered lines
ℹ -------------------------------------------------------------
ℹ main.test.js | 100.00 |   100.00 |  100.00 |
ℹ src/name.js  | 100.00 |   100.00 |  100.00 |
ℹ -------------------------------------------------------------
ℹ all files    | 100.00 |   100.00 |  100.00 |
ℹ -------------------------------------------------------------
ℹ end of coverage report
```

このカバレッジレポートにはテストファイルも含まれていますが、`src/` ディレクトリ内の JavaScript ファイルのみを対象としています。この場合、`--test-coverage-include` フラグ（`run()` API では `coverageIncludeGlobs`）を使用できます。

```bash displayName="CLI"
node --experimental-test-coverage --test-coverage-include=src/*.js --test main.test.js
```

```js displayName="run()"
run({ files: ['main.test.js'], coverage: true, coverageInclude: ['src/*.js'] });
```

```text displayName="New coverage report"
ℹ start of coverage report
ℹ ------------------------------------------------------------
ℹ file        | line % | branch % | funcs % | uncovered lines
ℹ ------------------------------------------------------------
ℹ src/age.js  |  45.45 |   100.00 |    0.00 | 3-5 7-9
ℹ src/name.js | 100.00 |   100.00 |  100.00 |
ℹ ------------------------------------------------------------
ℹ all files   |  72.73 |   100.00 |   66.67 |
ℹ ------------------------------------------------------------
ℹ end of coverage report
```

## しきい値

デフォルトでは、すべてのテストに合格すると、Node.js は実行が成功したことを示すコード `0` で終了します。ただし、カバレッジが失敗した場合にコード `1` で終了するようにカバレッジレポートを設定できます。

Node.js は現在、サポートされている 3 つのカバレッジすべてに対してしきい値をサポートしています。

- [`--test-coverage-lines`](https://nodejs.org/api/cli.html#--test-coverage-linesthreshold) (`run()` API の `lineCoverage`) は行カバレッジを示します。
- [`--test-coverage-branches`](https://nodejs.org/api/cli.html#--test-coverage-branchesthreshold) (`run()` API の `branchCoverage`) は分岐カバレッジを示します。
- 関数カバレッジには [`--test-coverage-functions`](https://nodejs.org/api/cli.html#--test-coverage-functionsthreshold) (`run()` API の `functionCoverage`) を使用します。

前の例で行カバレッジ >= 90% を要求したい場合は、`--test-coverage-lines=90` フラグ (`run()` API の `lineCoverage: 90`) を使用できます。

```bash displayName="CLI"
node --experimental-test-coverage --test-coverage-lines=90 --test main.test.js
```

```js displayName="run()"
run({ files: ['main.test.js'], coverage: true, lineCoverage: 90 });
```

```text displayName="Coverage Report"
ℹ start of coverage report
ℹ -------------------------------------------------------------
ℹ file         | line % | branch % | funcs % | uncovered lines
ℹ -------------------------------------------------------------
ℹ main.test.js | 100.00 |   100.00 |  100.00 |
ℹ src/age.js   |  45.45 |   100.00 |    0.00 | 3-5 7-9
ℹ src/name.js  | 100.00 |   100.00 |  100.00 |
ℹ -------------------------------------------------------------
ℹ all files    |  88.68 |   100.00 |   75.00 |
ℹ -------------------------------------------------------------
ℹ end of coverage report
ℹ Error: 88.68% line coverage does not meet threshold of 90%.
```
