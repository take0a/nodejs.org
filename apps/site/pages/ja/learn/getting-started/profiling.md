---
title: Profiling Node.js Applications
layout: learn
---

# Node.js アプリケーションのプロファイリング

Node.js アプリケーションのプロファイリングでは、アプリケーション実行中の CPU、メモリ、その他のランタイムメトリックを分析してパフォーマンスを測定します。
これは、アプリケーションの効率、応答性、スケーラビリティに影響を与える可能性のあるボトルネック、CPU 使用率の高さ、メモリリーク、または低速な関数呼び出しを特定するのに役立ちます。

Node.js アプリケーションのプロファイリングには多くのサードパーティ製ツールがありますが、多くの場合、最も簡単な方法は Node.js の組み込みプロファイラーを使用することです。
組み込みプロファイラーは、プログラム実行中に一定の間隔でスタックをサンプリングする [V8 内部のプロファイラー][] を使用します。これらのサンプリング結果と、JIT コンパイルなどの重要な最適化イベントを、一連のティックとして記録します。

```
code-creation,LazyCompile,0,0x2d5000a337a0,396,"bp native array.js:1153:16",0x289f644df68,~
code-creation,LazyCompile,0,0x2d5000a33940,716,"hasOwnProperty native v8natives.js:198:30",0x289f64438d0,~
code-creation,LazyCompile,0,0x2d5000a33c20,284,"ToName native runtime.js:549:16",0x289f643bb28,~
code-creation,Stub,2,0x2d5000a33d40,182,"DoubleToIStub"
code-creation,Stub,2,0x2d5000a33e00,507,"NumberToStringStub"
```

以前は、ティックを解釈するには V8 のソースコードが必要でした。
幸いなことに、Node.js 4.4.0 以降では、V8 をソースから別途ビルドすることなく、この情報を容易に利用できるツールが導入されました。
組み込みのプロファイラーがアプリケーションのパフォーマンスに関する洞察をどのように提供するかを見てみましょう。

ティックプロファイラーの使い方を説明するために、シンプルな Express アプリケーションを操作します。このアプリケーションには 2 つのハンドラーがあり、1 つはシステムに新しいユーザーを追加するためのものです。

```js
app.get('/newUser', (req, res) => {
  let username = req.query.username || '';
  const password = req.query.password || '';

  username = username.replace(/[^a-zA-Z0-9]/g, '');

  if (!username || !password || users[username]) {
    return res.sendStatus(400);
  }

  const salt = crypto.randomBytes(128).toString('base64');
  const hash = crypto.pbkdf2Sync(password, salt, 10000, 512, 'sha512');

  users[username] = { salt, hash };

  res.sendStatus(200);
});
```

もう 1 つはユーザー認証の試行を検証するためのものです。

```js
app.get('/auth', (req, res) => {
  let username = req.query.username || '';
  const password = req.query.password || '';

  username = username.replace(/[^a-zA-Z0-9]/g, '');

  if (!username || !password || !users[username]) {
    return res.sendStatus(400);
  }

  const { salt, hash } = users[username];
  const encryptHash = crypto.pbkdf2Sync(password, salt, 10000, 512, 'sha512');

  if (crypto.timingSafeEqual(hash, encryptHash)) {
    res.sendStatus(200);
  } else {
    res.sendStatus(401);
  }
});
```

_これらのハンドラは、Node.js アプリケーションでユーザーを認証するための推奨ハンドラではなく、あくまでも説明目的で使用されていることにご注意ください。一般的に、独自の暗号化認証メカニズムを設計しようとすべきではありません。既存の実績のある認証ソリューションを使用する方がはるかに優れています。_

さて、アプリケーションをデプロイし、ユーザーからリクエストのレイテンシが高いという苦情が寄せられていると仮定します。組み込みのプロファイラーを使えば、簡単にアプリを実行できます。

```
NODE_ENV=production node --prof app.js
```

そして、`ab` (ApacheBench) を使用してサーバーに負荷をかけます。

```
curl -X GET "http://localhost:8080/newUser?username=matt&password=password"
ab -k -c 20 -n 250 "http://localhost:8080/auth?username=matt&password=password"
```

次の ab の出力が得られます。

```
Concurrency Level:      20
Time taken for tests:   46.932 seconds
Complete requests:      250
Failed requests:        0
Keep-Alive requests:    250
Total transferred:      50250 bytes
HTML transferred:       500 bytes
Requests per second:    5.33 [#/sec] (mean)
Time per request:       3754.556 [ms] (mean)
Time per request:       187.728 [ms] (mean, across all concurrent requests)
Transfer rate:          1.05 [Kbytes/sec] received

...

Percentage of the requests served within a certain time (ms)
  50%   3755
  66%   3804
  75%   3818
  80%   3825
  90%   3845
  95%   3858
  98%   3874
  99%   3875
 100%   4225 (longest request)
```

この出力から、1秒あたり約5件のリクエストしか処理できておらず、平均的なリクエストの往復処理時間はわずか4秒弱であることがわかります。実際の例では、ユーザーリクエストに対応するために多くの関数で多くの処理を実行する可能性がありますが、このシンプルな例でさえ、正規表現のコンパイル、ランダムソルトの生成、ユーザーパスワードからの一意のハッシュの生成、あるいはExpressフレームワーク自体の処理などで時間が浪費される可能性があります。

`--prof` オプションを使用してアプリケーションを実行したため、アプリケーションのローカル実行と同じディレクトリに tick ファイルが生成されました。このファイルは `isolate-0xnnnnnnnnnnnn-v8.log` (`n` は数字) という形式です。

このファイルを理解するには、Node.js バイナリにバンドルされている tick プロセッサを使用する必要があります。このプロセッサを実行するには、`--prof-process` フラグを使用します。

```
node --prof-process isolate-0xnnnnnnnnnnnn-v8.log > processed.txt
```

お気に入りのテキストエディタでprocessed.txtを開くと、いくつかの異なる種類の情報が表示されます。ファイルは複数のセクションに分かれており、さらに言語ごとに分かれています。まず、summary セクションを見てみましょう。

```
 [Summary]:
   ticks  total  nonlib   name
     79    0.2%    0.2%  JavaScript
  36703   97.2%   99.2%  C++
      7    0.0%    0.0%  GC
    767    2.0%          Shared libraries
    215    0.6%          Unaccounted
```

これは、収集されたサンプルの97%がC++コードで発生していることを示しています。処理された出力の他のセクションを見る際には、JavaScriptではなくC++で行われている作業に最も注意を払う必要があることがわかります。これを念頭に置き、次に、どのC++関数が最もCPU時間を消費しているかに関する情報を含む\[C++\]セクションを見つけ、次の点を確認します。

```
 [C++]:
   ticks  total  nonlib   name
  19557   51.8%   52.9%  node::crypto::PBKDF2(v8::FunctionCallbackInfo<v8::Value> const&)
   4510   11.9%   12.2%  _sha1_block_data_order
   3165    8.4%    8.6%  _malloc_zone_malloc
```

上位3つのエントリが、プログラムのCPU時間の72.1%を占めていることがわかります。この出力から、CPU時間の少なくとも51.8%がPBKDF2という関数によって消費されていることがすぐにわかります。この関数は、ユーザーのパスワードからハッシュを生成する関数です。しかし、下位2つのエントリがアプリケーションにどのように影響するかはすぐには分かりません（仮にそうであったとしても、例としてそうではないと仮定します）。これらの関数間の関係をより深く理解するために、次に各関数の主な呼び出し元に関する情報を提供する\[Bottom up (heavy) profile\]セクションを見てみましょう。このセクションを調べると、次のことがわかります。

```
   ticks parent  name
  19557   51.8%  node::crypto::PBKDF2(v8::FunctionCallbackInfo<v8::Value> const&)
  19557  100.0%    v8::internal::Builtins::~Builtins()
  19557  100.0%      LazyCompile: ~pbkdf2 crypto.js:557:16

   4510   11.9%  _sha1_block_data_order
   4510  100.0%    LazyCompile: *pbkdf2 crypto.js:557:16
   4510  100.0%      LazyCompile: *exports.pbkdf2Sync crypto.js:552:30

   3165    8.4%  _malloc_zone_malloc
   3161   99.9%    LazyCompile: *pbkdf2 crypto.js:557:16
   3161  100.0%      LazyCompile: *exports.pbkdf2Sync crypto.js:552:30
```

このセクションの解析には、上記の生のティックカウントよりも少し手間がかかります。
上記の各「コールスタック」において、親列のパーセンテージは、現在の行の関数によって前の行の関数が呼び出されたサンプルの割合を示しています。例えば、上記の中央の「コールスタック」の \_sha1_block_data_order では、`_sha1_block_data_order` がサンプルの 11.9% で発生していることがわかります。これは上記の生のカウントから分かっていました。しかし、ここでは、`_sha1_block_data_order` が常に Node.js 暗号モジュール内の pbkdf2 関数によって呼び出されていたこともわかります。
同様に、`_malloc_zone_malloc` もほぼ例外なく同じ pbkdf2 関数によって呼び出されていたことがわかります。したがって、このビューの情報を使用すると、ユーザーのパスワードからのハッシュ計算は、上記の 51.8% だけでなく、上位 3 つの最も多くサンプリングされた関数の CPU 時間すべてを占めていることがわかります。これは、`_sha1_block_data_order` と `_malloc_zone_malloc` の呼び出しが pbkdf2 関数によって行われたためです。

この時点で、パスワードベースのハッシュ生成が最適化の対象となるべきであることは明らかです。幸いなことに、[非同期プログラミングの利点][] を十分に理解しており、ユーザーのパスワードからハッシュを生成する作業が同期的に行われ、イベントループが固定されていることを理解しています。これにより、ハッシュ計算中に他の受信リクエストを処理することができなくなります。

この問題を解決するには、上記のハンドラーに小さな変更を加えて、pbkdf2 関数の非同期バージョンを使用します。

```js
app.get('/auth', (req, res) => {
  let username = req.query.username || '';
  const password = req.query.password || '';

  username = username.replace(/[^a-zA-Z0-9]/g, '');

  if (!username || !password || !users[username]) {
    return res.sendStatus(400);
  }

  crypto.pbkdf2(
    password,
    users[username].salt,
    10000,
    512,
    'sha512',
    (err, hash) => {
      if (users[username].hash.toString() === hash.toString()) {
        res.sendStatus(200);
      } else {
        res.sendStatus(401);
      }
    }
  );
});
```

上記の ab ベンチマークをアプリの非同期バージョンで新たに実行すると、次の結果が得られます。

```
Concurrency Level:      20
Time taken for tests:   12.846 seconds
Complete requests:      250
Failed requests:        0
Keep-Alive requests:    250
Total transferred:      50250 bytes
HTML transferred:       500 bytes
Requests per second:    19.46 [#/sec] (mean)
Time per request:       1027.689 [ms] (mean)
Time per request:       51.384 [ms] (mean, across all concurrent requests)
Transfer rate:          3.82 [Kbytes/sec] received

...

Percentage of the requests served within a certain time (ms)
  50%   1018
  66%   1035
  75%   1041
  80%   1043
  90%   1049
  95%   1063
  98%   1070
  99%   1071
 100%   1079 (longest request)
```

やった！アプリは現在、1秒あたり約20件のリクエストを処理できるようになっています。これは、同期ハッシュ生成を使用していた場合と比べて約4倍の速度です。さらに、平均レイテンシは以前の4秒から1秒強に短縮されています。

この（確かに不自然な）例のパフォーマンス調査を通して、V8 tickプロセッサがNode.jsアプリケーションのパフォーマンスをより深く理解する上でどのように役立つかご理解いただけたかと思います。

[フレームグラフの作成方法][診断フレームグラフ]も参考になるかもしれません。

[V8 内部のプロファイラー]: https://v8.dev/docs/profile
[非同期プログラミングの利点]: https://nodesource.com/blog/why-asynchronous
[診断フレームグラフ]: /learn/diagnostics/flame-graphs
