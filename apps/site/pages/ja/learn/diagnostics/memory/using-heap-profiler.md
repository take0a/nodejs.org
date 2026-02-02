---
title: Using Heap Profiler
layout: learn
---

# ヒーププロファイラーの使用

ヒーププロファイラーは V8 上で動作し、時間の経過に伴うメモリ割り当てをキャプチャします。このドキュメントでは、以下の手法を用いたメモリプロファイリングについて説明します。

1. 割り当てタイムライン
2. サンプリングヒーププロファイラー

[ヒープスナップショットの使用][] ガイドで説明したヒープダンプとは異なり、リアルタイムプロファイリングの目的は、一定期間にわたるメモリ割り当てを把握することです。

## ヒーププロファイラー - 割り当てタイムライン

ヒーププロファイラーはサンプリングヒーププロファイラーに似ていますが、すべての割り当てをトレースします。サンプリングヒーププロファイラーよりもオーバーヘッドが大きいため、本番環境での使用は推奨されません。

> プロファイラーをプログラムから起動および停止するには、[@mmarchini/observe][] を使用します。

### 方法

アプリケーションの起動:

```console
node --inspect index.js
```

> スクリプトには `--inspect-brk` の方が適しています。

Chrome で開発ツールインスタンスに接続し、以下の操作を行います。

- `Memory` タブを選択します。
- `Allocation instrumentation timeline` を選択します。
- プロファイリングを開始します。

![heap profiler tutorial step 1][heap profiler tutorial 1]

ヒーププロファイリングの実行後、メモリの問題を特定するためにサンプルを実行することを強くお勧めします。例えば、Web アプリケーションのヒーププロファイリングを行う場合、`Apache Benchmark` を使用して負荷を生成できます。

```console
$ ab -n 1000 -c 5 http://localhost:3000
```

ロードが完了したら、停止ボタンを押します。

![heap profiler tutorial step 2][heap profiler tutorial 2]

最後に、スナップショットデータを確認します。

![heap profiler tutorial step 3][heap profiler tutorial 3]

メモリに関する用語の詳細については、[役立つリンク](#useful-links)セクションをご覧ください。

## サンプリングヒーププロファイラー

サンプリングヒーププロファイラーは、メモリ割り当てパターンと予約領域を経時的に追跡します。サンプリングベースであるため、オーバーヘッドは本番システムでも使用できるほど低くなっています。

> モジュール [`heap-profiler`][] を使用して、ヒーププロファイラーをプログラムから起動および停止できます。

### 方法

アプリケーションの起動:

```console
$ node --inspect index.js
```

> スクリプトには `--inspect-brk` の方が適しています。

dev-tools インスタンスに接続し、以下の操作を行います。

1. 「Memory」タブを選択します。
2. 「Allocation Sampling」を選択します。
3. プロファイリングを開始します。

![heap profiler tutorial 4][heap profiler tutorial 4]

負荷をかけ、プロファイラーを停止します。スタックトレースに基づいて割り当ての概要が生成されます。ヒープ割り当てが多い関数に注目してみましょう。以下の例をご覧ください。

![heap profiler tutorial 5][heap profiler tutorial 5]

## 便利なリンク

- https://developer.chrome.com/docs/devtools/memory-problems/memory-101/
- https://developer.chrome.com/docs/devtools/memory-problems/allocation-profiler/

[Using Heap Snapshot]: /learn/diagnostics/memory/using-heap-snapshot/
[@mmarchini/observe]: https://www.npmjs.com/package/@mmarchini/observe
[`heap-profiler`]: https://www.npmjs.com/package/heap-profile
[heap profiler tutorial 1]: /static/images/docs/guides/diagnostics/heap-profiler-tutorial-1.png
[heap profiler tutorial 2]: /static/images/docs/guides/diagnostics/heap-profiler-tutorial-2.png
[heap profiler tutorial 3]: /static/images/docs/guides/diagnostics/heap-profiler-tutorial-3.png
[heap profiler tutorial 4]: /static/images/docs/guides/diagnostics/heap-profiler-tutorial-4.png
[heap profiler tutorial 5]: /static/images/docs/guides/diagnostics/heap-profiler-tutorial-5.png
