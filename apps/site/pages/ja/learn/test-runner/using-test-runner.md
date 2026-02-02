---
title: Using Node.js's test runner
layout: learn
authors: JakobJingleheimer
---

# Node.js のテストランナーの使用

Node.js には、柔軟で堅牢な組み込みテストランナーが搭載されています。このガイドでは、その設定方法と使い方を説明します。

```text displayName="Architecture overview"
example/
  ├ …
  ├ src/
    ├ app/…
    └ sw/…
  └ test/
    ├ globals/
      ├ …
      ├ IndexedDb.js
      └ ServiceWorkerGlobalScope.js
    ├ setup.mjs
    ├ setup.units.mjs
    └ setup.ui.mjs
```

```bash displayName="Install dependencies"
npm init -y
npm install --save-dev concurrently
```

```json displayName="package.json"
{
  "name": "example",
  "scripts": {
    "test": "concurrently --kill-others-on-fail --prefix none npm:test:*",
    "test:sw": "node --import ./test/setup.sw.mjs --test './src/sw/**/*.spec.*'",
    "test:units": "node --import ./test/setup.units.mjs --test './src/app/**/*.spec.*'",
    "test:ui": "node --import ./test/setup.ui.mjs --test './src/app/**/*.test.*'"
  }
}
```

> **注**: globs は Node.js バージョン 21 以降で使用でき、globs 自体を引用符で囲む必要があります (引用符で囲まないと、最初は動作しているように見えても実際には動作しないなど、想定とは異なる動作をする可能性があります)。

常に必要な項目がいくつかあるため、以下のような基本設定ファイルに記述しておきます。このファイルは、他のよりカスタマイズされた設定によってインポートされます。

## 一般的な設定

<details>
<summary>`test/setup.mjs`</summary>

```js
import { register } from 'node:module';

register('some-typescript-loader');
// TypeScript is supported hereafter
// BUT other test/setup.*.mjs files still must be plain JavaScript!
```

</details>

次に、各セットアップごとに専用の `setup` ファイルを作成します（各セットアップにベースの `setup.mjs` ファイルがインポートされていることを確認してください）。セットアップを分離する理由はいくつかありますが、最も明白な理由は [YAGNI](https://en.wikipedia.org/wiki/You_aren't_gonna_need_it) とパフォーマンスです。セットアップするものの多くは環境固有のモック/スタブであり、これらは非常にコストがかかり、テスト実行の速度を低下させる可能性があります。不要な場合は、これらのコスト（CI に支払う実際の費用、テストの完了を待つ時間など）を回避する必要があります。

以下の各例は実際のプロジェクトから抜粋したものです。必ずしもお客様のプロジェクトに適切/適用できるとは限りませんが、いずれも広く適用可能な一般的な概念を示しています。

## テストケースの動的生成

テストケースを動的に生成したい場合があります。例えば、複数のファイルにわたって同じテストを実行したい場合などです。これは少し複雑ですが、可能です。`test` と `testContext.test` を使用する必要があります（`describe` は使用できません）。

### 簡単な例

```js displayName="23.8.0 and later"
import assert from 'node:assert/strict';
import { test } from 'node:test';

import { detectOsInUserAgent } from '…';

const userAgents = [
  {
    ua: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.3',
    os: 'WIN',
  },
  // …
];

test('Detect OS via user-agent', { concurrency: true }, t => {
  for (const { os, ua } of userAgents) {
    t.test(ua, () => assert.equal(detectOsInUserAgent(ua), os));
  }
});
```

```js displayName="prior to 23.8.0"
import assert from 'node:assert/strict';
import { test } from 'node:test';

import { detectOsInUserAgent } from '…';

const userAgents = [
  { ua: '…', os: 'WIN' },
  // …
];

test('Detect OS via user-agent', { concurrency: true }, async t => {
  const cases = userAgents.map(({ os, ua }) => {
    t.test(ua, () => assert.equal(detectOsInUserAgent(ua), os));
  });

  await Promise.allSettled(cases);
});
```

### 高度な例

```js displayName="23.8.0 and later"
import assert from 'node:assert/strict';
import { test } from 'node:test';

import { getWorkspacePJSONs } from './getWorkspacePJSONs.mjs';

const requiredKeywords = ['node.js', 'sliced bread'];

test('Check package.jsons', { concurrency: true }, async t => {
  const pjsons = await getWorkspacePJSONs();

  for (const pjson of pjsons) {
    // ⚠️ `t.test`, NOT `test`
    t.test(`Ensure fields are properly set: ${pjson.name}`, () => {
      assert.partialDeepStrictEqual(pjson.keywords, requiredKeywords);
    });
  }
});
```

```js displayName="prior to 23.8.0"
import assert from 'node:assert/strict';
import { test } from 'node:test';

import { getWorkspacePJSONs } from './getWorkspacePJSONs.mjs';

const requiredKeywords = ['node.js', 'sliced bread'];

test('Check package.jsons', { concurrency: true }, async t => {
  const pjsons = await getWorkspacePJSONs();

  const cases = pjsons.map(pjson =>
    // ⚠️ `t.test`, NOT `test`
    t.test(`Ensure fields are properly set: ${pjson.name}`, () => {
      assert.partialDeepStrictEqual(pjson.keywords, requiredKeywords);
    })
  );

  // Allow the cases to run concurrently.
  await Promise.allSettled(cases);
});
```

```js displayName="./getWorkspacePJSONs.mjs"
import { globSync } from 'node:fs';
import { fileURLToPath } from 'node:url';

// 注: これは、fs.globSync ではなく fs.glob を活用した非同期ジェネレーターとして
// 実装する方が適切ですが、ジェネレーター、特に非同期ジェネレーターはあまり理解されていないため、
// 理解しやすいようにこの簡略化された例が提供されています。

/**
 * Get all the package.json files, by default 1-level deep within ./workspaces/
 */
export function getWorkspacePJSONs(path = './workspaces/*/package.json') {
  return Promise.all(
    globSync(
      // ⚠️ import.meta.resolve のようなファイル URL 文字列を渡すと、glob* は何も言わずに失敗する。
      fileURLToPath(import.meta.resolve(path))
    ).map(path => import(path, { with: { type: 'json' } }))
  );
}
```

> **注**: バージョン 23.8.0 より前では、`testContext.test` が自動的に待機されないため、設定が大きく異なります。

## ServiceWorker テスト

[`ServiceWorkerGlobalScope`](https://developer.mozilla.org/docs/Web/API/ServiceWorkerGlobalScope) には、他の環境に存在しない非常に特殊な API が含まれており、その一部の API は一見他の API と似ていますが (例 `fetch`)、動作が拡張されています。これらの API を無関係なテストに含めないようにする必要があります。

<details>
<summary>`test/setup.sw.mjs`</summary>

```js
import { beforeEach } from 'node:test';

import { ServiceWorkerGlobalScope } from './globals/ServiceWorkerGlobalScope.js';

import './setup.mjs'; // 💡

beforeEach(globalSWBeforeEach);
function globalSWBeforeEach() {
  globalThis.self = new ServiceWorkerGlobalScope();
}
```

</details>

```js
import assert from 'node:assert/strict';
import { describe, mock, it } from 'node:test';

import { onActivate } from './onActivate.js';

describe('ServiceWorker::onActivate()', () => {
  const globalSelf = globalThis.self;
  const claim = mock.fn(async function mock__claim() {});
  const matchAll = mock.fn(async function mock__matchAll() {});

  class ActivateEvent extends Event {
    constructor(...args) {
      super('activate', ...args);
    }
  }

  before(() => {
    globalThis.self = {
      clients: { claim, matchAll },
    };
  });
  after(() => {
    global.self = globalSelf;
  });

  it('should claim all clients', async () => {
    await onActivate(new ActivateEvent());

    assert.equal(claim.mock.callCount(), 1);
    assert.equal(matchAll.mock.callCount(), 1);
  });
});
```

## スナップショットテスト

これはJestによって普及し、現在では多くのライブラリがこの機能を搭載しており、Node.jsもv22.3.0以降で実装されています。コンポーネントのレンダリング出力や[Infrastructure as Code](https://en.wikipedia.org/wiki/Infrastructure_as_code)設定の検証など、様々なユースケースがあります。ユースケースに関わらず、概念は同じです。

[`--experimental-test-snapshots`]() でこの機能を有効にする以外に、特に必要な設定はありません。ただし、オプションの設定を確認するには、既存のテスト設定ファイルに次のようなコードを追加するとよいでしょう。

<details>
<summary>`test/setup.ui.mjs`</summary>

デフォルトでは、node は構文の強調表示検出と互換性のないファイル名 `.js.snapshot` を生成します。生成されたファイルは実際には CJS ファイルなので、より適切なファイル名は `.snapshot.cjs` (または以下のように簡潔に `.snap.cjs`) で終わります。この方がESMプロジェクトでも適切に処理されます。

```js
import { basename, dirname, extname, join } from 'node:path';
import { snapshot } from 'node:test';

snapshot.setResolveSnapshotPath(generateSnapshotPath);
/**
 * @param {string} testFilePath '/tmp/foo.test.js'
 * @returns {string} '/tmp/foo.test.snap.cjs'
 */
function generateSnapshotPath(testFilePath) {
  const ext = extname(testFilePath);
  const filename = basename(testFilePath, ext);
  const base = dirname(testFilePath);

  return join(base, `${filename}.snap.cjs`);
}
```

</details>

以下の例は、UI コンポーネントの [テスト ライブラリ](https://testing-library.com/) を使用したスナップショット テストを示しています。`assert.snapshot` にアクセスする 2 つの異なる方法に注意してください。):

```js
import { describe, it } from 'node:test';

import { prettyDOM } from '@testing-library/dom';
import { render } from '@testing-library/react'; // Any framework (ex svelte)

import { SomeComponent } from './SomeComponent.jsx';

describe('<SomeComponent>', () => {
  // 「fat-arrow」構文を好む人にとっては、一貫性を保つために次の構文の方が良いでしょう。
  it('should render defaults when no props are provided', t => {
    const component = render(<SomeComponent />).container.firstChild;

    t.assert.snapshot(prettyDOM(component));
  });

  it('should consume `foo` when provided', function () {
    const component = render(<SomeComponent foo="bar" />).container.firstChild;

    this.assert.snapshot(prettyDOM(component));
    // `this` works only when `function` is used (not "fat arrow").
  });
});
```

> ⚠️ `assert.snapshot` はテストのコンテキスト（`t` または `this`）から取得され、**`node:assert` からは取得されません**。これは、テストのコンテキストが `node:assert` ではアクセスできないスコープにアクセスできるため必要です（`assert.snapshot` を使用するたびに、`snapshot(this, value)` のように手動でスコープを指定する必要があり、非常に面倒です）。

## ユニットテスト

ユニットテストは最も単純なテストであり、通常は特別な設定は必要ありません。テストの大部分はユニットテストになる可能性が高いため、この設定は最小限に抑えることが重要です。設定パフォーマンスがわずかに低下すると、それが拡大し、連鎖的に影響を及ぼします。
<details>
<summary>`test/setup.units.mjs`</summary>

```js
import { register } from 'node:module';

import './setup.mjs'; // 💡

register('some-plaintext-loader');
// plain-text files like graphql can now be imported:
// import GET_ME from 'get-me.gql'; GET_ME = '
```

</details>

```js
import assert from 'node:assert/strict';
import { describe, it } from 'node:test';

import { Cat } from './Cat.js';
import { Fish } from './Fish.js';
import { Plastic } from './Plastic.js';

describe('Cat', () => {
  it('should eat fish', () => {
    const cat = new Cat();
    const fish = new Fish();

    assert.doesNotThrow(() => cat.eat(fish));
  });

  it('should NOT eat plastic', () => {
    const cat = new Cat();
    const plastic = new Plastic();

    assert.throws(() => cat.eat(plastic));
  });
});
```

## ユーザーインターフェーステスト

UI テストには通常、DOM と、場合によってはブラウザ固有の API（以下で使用されている [`IndexedDb`](https://developer.mozilla.org/docs/Web/API/IndexedDB_API) など）が必要です。これらの API の設定は非常に複雑で、コストがかかる傾向があります。

<details>
<summary>`test/setup.ui.mjs`</summary>

`IndexedDb` のような API を使用しているものの、それが非常に独立している場合、以下のようなグローバルモックはおそらく適切ではありません。代わりに、この `beforeEach` を `IndexedDb` がアクセスされる特定のテストに移動することをお勧めします。`IndexedDb`（または他のモジュール）にアクセスするモジュール自体が広くアクセスされている場合は、そのモジュールをモック化するか（おそらくこちらの方がより良い選択肢です）、またはこのモジュールをそのままにしておくことをお勧めします。

```js
import { register } from 'node:module';

// ⚠️ JSDomのインスタンスが1つだけインスタンス化されていることを確認してください。
// 複数インスタンス化されている場合は、🤬
import jsdom from 'global-jsdom';

import './setup.units.mjs'; // 💡

import { IndexedDb } from './globals/IndexedDb.js';

register('some-css-modules-loader');

jsdom(undefined, {
  url: 'https://test.example.com', // ⚠️ これを明記しないと、多くの 🤬
});

// グローバルを装飾する方法の例。
// JSDOM の `history` はナビゲーションを処理しません。次のコードでほとんどのケースを処理します。
const pushState = globalThis.history.pushState.bind(globalThis.history);
globalThis.history.pushState = function mock_pushState(data, unused, url) {
  pushState(data, unused, url);
  globalThis.location.assign(url);
};

beforeEach(globalUIBeforeEach);
function globalUIBeforeEach() {
  globalThis.indexedDb = new IndexedDb();
}
```

</details>

UIテストには2つの異なるレベルがあります。ユニットテスト（外部と依存関係をモック化したもの）と、エンドツーエンドテスト（IndexedDbなどの外部のみをモック化し、残りのチェーンは実際のもの）です。一般的に前者はより純粋な選択肢であり、後者は[Playwright](https://playwright.dev/)や[Puppeteer](https://pptr.dev/)などのツールによるエンドツーエンドの完全自動化ユーザビリティテストに委ねられるのが一般的です。以下は前者の例です。

```js
import { before, describe, mock, it } from 'node:test';

import { screen } from '@testing-library/dom';
import { render } from '@testing-library/react'; // Any framework (ex svelte)

// ⚠️ SomeOtherComponent は静的インポートではないことに注意してください。
// これは、自身のインポートのモックを容易にするために必要です。

describe('<SomeOtherComponent>', () => {
  let SomeOtherComponent;
  let calcSomeValue;

  before(async () => {
    // ⚠️ 順序が重要です。モックは、そのコンシューマーがインポートされる前にセットアップする必要があります。

    // `--experimental-test-module-mocks` を設定する必要があります。
    calcSomeValue = mock.module('./calcSomeValue.js', {
      calcSomeValue: mock.fn(),
    });

    ({ SomeOtherComponent } = await import('./SomeOtherComponent.jsx'));
  });

  describe('when calcSomeValue fails', () => {
    // これをスナップショットで処理することは、脆弱であるため望ましくありません。
    // エラー メッセージに重要でない更新が加えられると、スナップショットテストが
    // 誤って失敗します (そして、実際の価値がないのにスナップショットを更新する必要があります)。

    it('should fail gracefully by displaying a pretty error', () => {
      calcSomeValue.mockImplementation(function mock__calcSomeValue() {
        return null;
      });

      render(<SomeOtherComponent />);

      const errorMessage = screen.queryByText('unable');

      assert.ok(errorMessage);
    });
  });
});
```
