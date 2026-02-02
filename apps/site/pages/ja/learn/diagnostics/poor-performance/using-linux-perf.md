---
title: Using Linux Perf
layout: learn
---

# Linux Perf の使用

[Linux Perf](https://perf.wiki.kernel.org/index.php/Main_Page) は、JavaScript、ネイティブ、OS レベルのフレームを用いた低レベルの CPU プロファイリングを提供します。

**重要**: このチュートリアルは Linux でのみ利用可能です。

## 使用方法

Linux Perf は通常、`linux-tools-common` パッケージから入手できます。`--perf-basic-prof` または `--perf-basic-prof-only-functions` のいずれかを使用することで、_perf_events_ をサポートする Node.js アプリケーションを起動できます。

`--perf-basic-prof` は常にファイル (/tmp/perf-PID.map) に書き込むため、ディスクサイズが無限に大きくなる可能性があります。
それが懸念される場合は、モジュール [linux-perf](https://www.npmjs.com/package/linux-perf)
または `--perf-basic-prof-only-functions` を使用してください。

両者の主な違いは、`--perf-basic-prof-only-functions` の方が出力が少ないことです。これは、本番環境でのプロファイリングに適したオプションです。

```console
# Launch the application an get the PID
$ node --perf-basic-prof-only-functions index.js &
[1] 3870
```

次に、希望する頻度に基づいてイベントを記録します。

```console
$ sudo perf record -F 99 -p 3870 -g
```

このフェーズでは、信頼性の高い分析のためにより多くのレコードを生成するために、アプリケーションで負荷テストを実行することをお勧めします。ジョブが完了したら、コマンドにSIGINT (Ctrl-C) を送信して perf プロセスを終了してください。

`perf` は、`/tmp` フォルダ内に、通常 `/tmp/perf-PID.map` (上記の例では `/tmp/perf-3870.map`) というファイルを生成します。このファイルには、呼び出された各関数のトレースが含まれます。

これらの結果を特定のファイルに集約するには、次のコマンドを実行します。

```console
$ sudo perf script > perfs.out
```

```console
$ cat ./perfs.out
node 3870 25147.878454:          1 cycles:
        ffffffffb5878b06 native_write_msr+0x6 ([kernel.kallsyms])
        ffffffffb580d9d5 intel_tfa_pmu_enable_all+0x35 ([kernel.kallsyms])
        ffffffffb5807ac8 x86_pmu_enable+0x118 ([kernel.kallsyms])
        ffffffffb5a0a93d perf_pmu_enable.part.0+0xd ([kernel.kallsyms])
        ffffffffb5a10c06 __perf_event_task_sched_in+0x186 ([kernel.kallsyms])
        ffffffffb58d3e1d finish_task_switch+0xfd ([kernel.kallsyms])
        ffffffffb62d46fb __sched_text_start+0x2eb ([kernel.kallsyms])
        ffffffffb62d4b92 schedule+0x42 ([kernel.kallsyms])
        ffffffffb62d87a9 schedule_hrtimeout_range_clock+0xf9 ([kernel.kallsyms])
        ffffffffb62d87d3 schedule_hrtimeout_range+0x13 ([kernel.kallsyms])
        ffffffffb5b35980 ep_poll+0x400 ([kernel.kallsyms])
        ffffffffb5b35a88 do_epoll_wait+0xb8 ([kernel.kallsyms])
        ffffffffb5b35abe __x64_sys_epoll_wait+0x1e ([kernel.kallsyms])
        ffffffffb58044c7 do_syscall_64+0x57 ([kernel.kallsyms])
        ffffffffb640008c entry_SYSCALL_64_after_hwframe+0x44 ([kernel.kallsyms])
....
```

生の出力は理解しにくい場合があるため、通常は生のファイルを使用してフレームグラフを生成し、より視覚的にわかりやすくします。

![Node.js フレームグラフの例](https://user-images.githubusercontent.com/26234614/129488674-8fc80fd5-549e-4a80-8ce2-2ba6be20f8e8.png)

この結果からフレームグラフを生成するには、[こちらのチュートリアル](/learn/diagnostics/flame-graphs#create-a-flame-graph-with-system-perf-tools)の手順6以降に従ってください。

`perf` 出力は Node.js 専用のツールではないため、Node.js における JavaScript コードの最適化方法に問題が発生する可能性があります。詳細については、[perf 出力の問題](/learn/diagnostics/flame-graphs#perf-output-issues) を参照してください。

## Useful Links

- /learn/diagnostics/flame-graphs
- https://www.brendangregg.com/blog/2014-09-17/node-flame-graphs-on-linux.html
- https://perf.wiki.kernel.org/index.php/Main_Page
- https://blog.rafaelgss.com.br/node-cpu-profiler
