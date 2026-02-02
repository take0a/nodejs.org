---
title: Mocking in tests
layout: learn
authors: JakobJingleheimer
---

# テストにおけるモック

モックとは、模造品、つまり操り人形を作成する手段です。これは通常、「when 'a', do 'b'」という操り人形的な方法で行われます。その考え方は、動く部分の数を制限し、「重要でない」ものを制御することです。「モック」と「スタブ」は、技術的には異なる種類の「テストダブル」です。興味深いことに、スタブは何もしない（no-op）ものの、呼び出しを追跡する代替手段です。モックは、偽の実装（「when 'a', do 'b'」）を持つスタブです。このドキュメントでは、この違いは重要ではなく、スタブをモックと呼びます。

テストは決定論的である必要があります。つまり、任意の順序で、任意の回数実行でき、常に同じ結果を生成する必要があります。適切な設定とモックによってこれが可能になります。

Node.js は、さまざまなコードをモックする多くの方法を提供しています。

この記事では、次の種類のテストについて説明します。

| type             | description                               | example                                                                                        | mock candidates                          |
| :--------------- | :---------------------------------------- | :--------------------------------------------------------------------------------------------- | :--------------------------------------- |
| ユニット | 分離できる最小のコード | `const sum = (a, b) => a + b` | 独自のコード、外部コード、外部システム |
| コンポーネント | ユニット + 依存関係 | `const arithmetic = (op = sum, a, b) => ops[op](a, b)` | 外部コード、外部システム |
| 統合 | コンポーネントの統合 | - | 外部コード、外部システム |
| エンドツーエンド (e2e) | アプリ + 外部データストア、配信など | 偽のユーザー (例: Playwright エージェント) が、実際の外部システムに接続されたアプリを文字通り使用しています。 | なし (モックしない) |

モックするべき場合とすべきでない場合については様々な考え方がありますが、大まかな概要を以下に示します。

## モックするべき場合とすべきでない場合

モックの対象となる主なコードは 3 つあります。

- 独自のコード
- 外部コード
- 外部システム

### 独自のコード

これはプロジェクトが管理するコードです。

```mjs displayName="your-project/main.mjs"
import foo from './foo.mjs';

export function main() {
  const f = foo();
}
```

ここで、`foo` は `main` の「独自のコード」依存関係です。

#### Why

`main` の真の単体テストを行うには、`foo` をモック化する必要があります。`main` が動作することをテストしているのであって、`main` + `foo` が動作することをテストしているのではありません（これは別のテストです）。

#### Why not

`foo` をモック化することは、特に `foo` がシンプルで、十分にテストされており、めったに更新されない場合、メリットよりも手間がかかる可能性があります。

`foo` をモック化しない方が、より信頼性が高く、`foo` のカバレッジが向上するため、より良い場合があります（`main` のテストで `foo` も検証されるため）。ただし、これはノイズを生み出す可能性があります。`foo` が失敗すると、他の多くのテストも失敗するため、問題の追跡がより面倒になります。問題の最終的な原因となっている項目のテストが 1 つだけ失敗している場合、それは非常に簡単に特定できます。一方、100 件のテストが失敗すると、真の問題を見つけるのは干し草の山から針を探すようなものになります。

### 外部コード

これはプロジェクトで制御できない部分です。

```mjs displayName="your-project/main.mjs"
import bar from 'bar';

export function main() {
  const f = bar();
}
```

ここで、`bar` は外部パッケージ（例えば npm の依存関係）です。

ユニットテストでは、これは必ずモック化すべきであることは議論の余地がありません。コンポーネントテストや統合テストでは、これが何であるかによってモック化するかどうかが変わります。

#### Why

プロジェクトでメンテナンスされていないコードが動作することを確認することは、ユニットテストの目的ではありません（そのようなコードには独自のテストが必要です）。

#### Why not

モック化が現実的でない場合もあります。例えば、React や Angular のような大規模なフレームワークをモック化することはほとんどありません（薬が病状より悪くなるからです）。

### 外部システム

これには、データベース、環境（Web アプリの場合は Chromium や Firefox、Node アプリの場合は OS など）、ファイルシステム、メモリストアなどが挙げられます。

理想的には、これらをモック化する必要はありません。各ケースごとに何らかの方法で独立したコピーを作成する（コストや実行時間の増加などにより、通常は非現実的です）以外に、次善策はモックを使用することです。モックを使用しないと、テストは互いに妨害し合います。

```mjs displayName="storage.mjs"
import { db } from 'db';

export function read(key, all = false) {
  validate(key, val);

  if (all) {
    return db.getAll(key);
  }

  return db.getOne(key);
}

export function save(key, val) {
  validate(key, val);

  return db.upsert(key, val);
}
```

```mjs displayName="storage.test.mjs"
import assert from 'node:assert/strict';
import { describe, it } from 'node:test';

import { db } from 'db';

import { save } from './storage.mjs';

describe('storage', { concurrency: true }, () => {
  it('should retrieve the requested item', async () => {
    await db.upsert('good', 'item'); // give it something to read
    await db.upsert('erroneous', 'item'); // give it a chance to fail

    const results = await read('a', true);

    assert.equal(results.length, 1); // ensure read did not retrieve erroneous item

    assert.deepEqual(results[0], { key: 'good', val: 'item' });
  });

  it('should save the new item', async () => {
    const id = await save('good', 'item');

    assert.ok(id);

    const items = await db.getAll();

    assert.equal(items.length, 1); // ensure save did not create duplicates

    assert.deepEqual(items[0], { key: 'good', val: 'item' });
  });
});
```

上記では、最初のケースと2番目のケース（`it()` 文）は同時に実行され、同じストアを変更するため（競合状態）、互いに悪影響を及ぼす可能性があります。`save()` の挿入により、本来は有効な `read()` のテストが、見つかった項目に対するアサーションに失敗する可能性があります（`read()` も `save()` に対して同様の動作をする可能性があります）。

## モック対象

### モジュール + ユニット

これは、Node.js テストランナーの [`mock`](https://nodejs.org/api/test.html#class-mocktracker) を利用しています。

```mjs
import assert from 'node:assert/strict';
import { before, describe, it, mock } from 'node:test';

describe('foo', { concurrency: true }, () => {
  const barMock = mock.fn();
  let foo;

  before(async () => {
    const barNamedExports = await import('./bar.mjs')
      // discard the original default export
      .then(({ default: _, ...rest }) => rest);

    // It's usually not necessary to manually call restore() after each
    // nor reset() after all (node does this automatically).
    mock.module('./bar.mjs', {
      defaultExport: barMock,
      // Keep the other exports that you don't want to mock.
      namedExports: barNamedExports,
    });

    // This MUST be a dynamic import because that is the only way to ensure the
    // import starts after the mock has been set up.
    ({ foo } = await import('./foo.mjs'));
  });

  it('should do the thing', () => {
    barMock.mock.mockImplementationOnce(function bar_mock() {
      /* … */
    });

    assert.equal(foo(), 42);
  });
});
```

### APIs

あまり知られていない事実ですが、`fetch` をモックする組み込みの方法があります。[`undici`](https://github.com/nodejs/undici) は `fetch` の Node.js 実装です。`node` に同梱されていますが、現時点では `node` 自体には公開されていないため、インストールする必要があります (例: `npm install undici`)。

```mjs displayName="endpoints.spec.mjs"
import assert from 'node:assert/strict';
import { beforeEach, describe, it } from 'node:test';

import { MockAgent, setGlobalDispatcher } from 'undici';

import endpoints from './endpoints.mjs';

describe('endpoints', { concurrency: true }, () => {
  let agent;
  beforeEach(() => {
    agent = new MockAgent();
    setGlobalDispatcher(agent);
  });

  it('should retrieve data', async () => {
    const endpoint = 'foo';
    const code = 200;
    const data = {
      key: 'good',
      val: 'item',
    };

    agent
      .get('https://example.com')
      .intercept({
        path: endpoint,
        method: 'GET',
      })
      .reply(code, data);

    assert.deepEqual(await endpoints.get(endpoint), {
      code,
      data,
    });
  });

  it('should save data', async () => {
    const endpoint = 'foo/1';
    const code = 201;
    const data = {
      key: 'good',
      val: 'item',
    };

    agent
      .get('https://example.com')
      .intercept({
        path: endpoint,
        method: 'PUT',
      })
      .reply(code, data);

    assert.deepEqual(await endpoints.save(endpoint), {
      code,
      data,
    });
  });
});
```

### 時間

ドクター・ストレンジのように、あなたも時間を制御できます。通常は、不自然なテスト実行の遅延を避けるため、便宜上これを行います（`setTimeout()` がトリガーされるまで3分も待ちたいですか？）。また、時間移動をしたい場合もあるでしょう。これは、Node.js テストランナーの [`mock.timers`](https://nodejs.org/api/test.html#class-mocktimers) を活用します。

ここでタイムゾーン（タイムスタンプの `Z`）を使用していることに注意してください。一貫したタイムゾーンを指定しないと、予期しない結果につながる可能性があります。

```mjs displayName="master-time.spec.mjs"
import assert from 'node:assert/strict';
import { describe, it, mock } from 'node:test';

import ago from './ago.mjs';

describe('whatever', { concurrency: true }, () => {
  it('should choose "minutes" when that\'s the closet unit', () => {
    mock.timers.enable({ now: new Date('2000-01-01T00:02:02Z') });

    const t = ago('1999-12-01T23:59:59Z');

    assert.equal(t, '2 minutes ago');
  });
});
```

これは、[スナップショット テスト](https://nodejs.org/api/test.html#snapshot-testing) など、静的フィクスチャ (リポジトリにチェックインされている) と比較する場合に特に便利です。
