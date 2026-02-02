---
title: Flame Graphs
layout: learn
---

# Flame Graphs

## Flame Graphs の便利な使い方

Flame Graphs は、関数内で消費された CPU 時間を視覚化する方法です。同期処理に時間がかかりすぎている箇所を特定するのに役立ちます。

## Flame Graphs の作成方法

Node.js で Flame Graphs を作成するのは難しいと聞いたことがあるかもしれませんが、それはもう間違いです。
Flame Graphs に Solaris VM は不要になりました！

Flame Graphs は、Node.js 固有のツールではない `perf` 出力から生成されます。CPU 消費時間を視覚化する最も強力な方法ですが、Node.js 8 以降では JavaScript コードの最適化方法に問題が発生する可能性があります。以下の [perf 出力の問題](#perf-output-issues) セクションを参照してください。

### 事前にパッケージ化されたツールを使用する

ローカルで Flame Graph を生成する単一のステップが必要な場合は、[0x](https://www.npmjs.com/package/0x) をお試しください。

本番環境のデプロイメントを診断するには、[0x production servers][] の注意事項をお読みください。

### システム perf ツールを使ってフレームグラフを作成する

このガイドの目的は、フレームグラフを作成する手順を示し、各手順をユーザーが適切に管理できるようにすることです。

各手順をより深く理解したい場合は、後続のセクションで詳細を説明しています。

それでは、作業に取り掛かりましょう。

1. `perf` をインストールします（まだインストールされていない場合は、通常 linux-tools-common パッケージから入手できます）。
2. `perf` を実行してみてください。カーネルモジュールが不足しているというエラーメッセージが表示される場合がありますので、それらもインストールしてください。
3. perf を有効にして node を実行します（Node.js のバージョン固有のヒントについては、[perf 出力の問題](#perf-output-issues) を参照してください）。

```bash
perf record -e cycles:u -g -- node --perf-basic-prof --interpreted-frames-native-stack app.js
```

4. パッケージが不足しているため perf を実行できないという警告でない限り、警告は無視してください。カーネルモジュールのサンプルにアクセスできないという警告が表示される場合もありますが、これはそもそも必要ないものです。
5. `perf script > perfs.out` を実行して、後ほど可視化するデータファイルを生成します。グラフを見やすくするために、[クリーンアップを適用](#filtering-out-nodejs-internal-functions)すると便利です。
6. Brendan Gregg の FlameGraph ツールをクローンします: https://github.com/brendangregg/FlameGraph
7. `cat perfs.out | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl --colors=js > profile.svg` を実行します。

お気に入りのブラウザで FlameGraph ファイルを開き、描画の様子を確認してください。色分けされているので、最初に最も彩度の高いオレンジ色のバーに注目できます。これらはCPU負荷の高い関数を表している可能性が高いです。

ちなみに、フレームグラフの要素をクリックすると、クリックしたセクションが拡大表示されます。

### `perf` を使って実行中のプロセスをサンプリングする

これは、中断したくない実行中のプロセスからフレームグラフデータを記録するのに最適です。再現が難しい問題のある本番環境のプロセスを想像してみてください。

```bash
perf record -F99 -p `pgrep -n node` -g -- sleep 3
```

ちょっと待ってください、`sleep 3` って何のためにあるのでしょうか？これは perf を実行し続けるためのものです。`-p` オプションで別の pid を指定しているにもかかわらず、コマンドはプロセス上で実行され、そのプロセスで終了する必要があります。
perf は、実際にそのコマンドをプロファイリングしているかどうかに関わらず、渡されたコマンドの実行中は実行されます。`sleep 3` は、perf が 3 秒間実行されることを保証します。

`-F` (プロファイリング頻度) が 99 に設定されているのはなぜでしょうか？これは妥当なデフォルト値です。必要に応じて調整できます。
`-F99` は perf に 1 秒あたり 99 回のサンプル取得を指示します。精度を高めるには、値を大きくしてください。値が小さいほど出力が少なくなり、結果の精度も低くなります。必要な精度は、CPU を集中的に使用する関数が実際にどれだけの時間実行されるかによって異なります。顕著な速度低下の原因を探しているのであれば、1 秒あたり 99 フレームで十分でしょう。

3 秒間のパフォーマンス記録を取得したら、上記の最後の 2 つの手順を実行してフレーム グラフの生成に進みます。

### Node.js 内部関数の除外

通常は、呼び出しのパフォーマンスだけを確認したいので、Node.js と V8 の内部関数を除外するとグラフが読みやすくなります。perf ファイルを以下のコマンドでクリーンアップできます。

```bash
sed -i -r \
  -e "/( __libc_start| LazyCompile | v8::internal::| Builtin:| Stub:| LoadIC:|\[unknown\]| LoadPolymorphicIC:)/d" \
  -e 's/ LazyCompile:[*~]?/ /' \
  perfs.out
```

フレームグラフを読んでいて、最も時間を消費している主要な関数に何かが欠けているような違和感を感じた場合は、フィルターなしでフレームグラフを生成してみてください。Node.js 自体に稀な問題が発生している可能性があります。

### Node.js のプロファイリングオプション

`--perf-basic-prof-only-functions` と `--perf-basic-prof` は、JavaScript コードのデバッグに役立つ 2 つのオプションです。その他のオプションは Node.js 自体のプロファイリングに使用されますが、このガイドの範囲外です。

`--perf-basic-prof-only-functions` は出力が少なくなるため、オーバーヘッドが最も少ないオプションです。

### なぜこれらのオプションが必要なのでしょうか？

これらのオプションがなくてもフレームグラフは生成されますが、ほとんどのバーには `v8::Function::Call` というラベルが付きます。

## `perf` 出力の問題

### Node.js 8.x V8 パイプラインの変更

Node.js 8.x 以降では、V8 エンジンの JavaScript コンパイルパイプラインに新しい最適化が導入されており、関数名や参照が perf でアクセスできなくなる場合があります (Turbofan と呼ばれます)。

その結果、Flame Graph で関数名が正しく表示されない可能性があります。

関数名が期待される場所に `ByteCodeHandler:` と表示されることがあります。

[0x](https://www.npmjs.com/package/0x) には、この問題に対する緩和策が組み込まれています。

詳細については、以下を参照してください。

- https://github.com/nodejs/benchmarking/issues/168
- https://github.com/nodejs/diagnostics/issues/148#issuecomment-369348961

### Node.js 10+

Node.js 10.x では、`--interpreted-frames-native-stack` フラグを使用することで Turbofan の問題を解決しています。

`node --interpreted-frames-native-stack --perf-basic-prof-only-functions` を実行すると、V8 が JavaScript をコンパイルする際にどのパイプラインを使用したかに関係なく、Flame Graph に関数名が表示されます。

### Flame Graph でラベルが壊れている

ラベルが以下のように表示される場合

```
node`_ZN2v88internal11interpreter17BytecodeGenerator15VisitStatementsEPNS0_8ZoneListIPNS0_9StatementEEE
```

これは、使用している Linux の perf が demangle サポート付きでコンパイルされていないことを意味します。例として、https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1396654 を参照してください。

## 例

[Flame Graph 演習](https://github.com/naugtur/node-example-flamegraph) を使って、Flame Graph のキャプチャを練習してみましょう。

[0x production servers]: https://github.com/davidmarkclements/0x/blob/master/docs/production-servers.md
