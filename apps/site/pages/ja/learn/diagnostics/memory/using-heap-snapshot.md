---
title: Using Heap Snapshot
layout: learn
---

# ヒープスナップショットの使用

実行中のアプリケーションからヒープスナップショットを取得し、[Chrome デベロッパーツール][] に読み込むことで、特定の変数を調べたり、リテーナーのサイズを確認したりできます。
複数のスナップショットを比較して、経時的な変化を確認することもできます。

## 警告

スナップショットを作成すると、メインスレッドの他のすべての作業が停止します。
ヒープの内容によっては、1分以上かかる場合もあります。
スナップショットはメモリ内に作成されるため、ヒープサイズが2倍になり、メモリ全体がいっぱいになってアプリがクラッシュする可能性があります。

本番環境でヒープスナップショットを取得する場合は、スナップショットを取得するプロセスがクラッシュしてもアプリケーションの可用性に影響がないことを確認してください。

## 方法

### ヒープスナップショットの取得

ヒープスナップショットを取得する方法は複数あります。

1. インスペクタ経由
2. 外部シグナルとコマンドラインフラグ経由
3. プロセス内での `writeHeapSnapshot` 呼び出し経由
4. インスペクタプロトコル経由

#### 1. インスペクターでメモリプロファイリングを使用する

> アクティブにメンテナンスされているすべてのバージョンの Node.js で動作します

`--inspect` フラグを付けて Node.js を実行し、インスペクターを開きます。
![open inspector][open inspector image]

ヒープスナップショットを取得する最も簡単な方法は、ローカルで実行されているプロセスにインスペクターを接続することです。次に、「メモリ」タブに移動して、ヒープスナップショットを取得します。

![take a heap snapshot][take a heap snapshot image]

#### 2. `--heapsnapshot-signal` フラグを使用する

> v12.0.0 以降で動作します

シグナルに反応してヒープスナップショットを作成するコマンドラインフラグを指定してノードを起動できます。

```
$ node --heapsnapshot-signal=SIGUSR2 index.js
```

```
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
node         1  5.5  6.1 787252 247004 ?       Ssl  16:43   0:02 node --heapsnapshot-signal=SIGUSR2 index.js
$ kill -USR2 1
$ ls
Heap.20190718.133405.15554.0.001.heapsnapshot
```

詳細については、[heapsnapshot-signal flag][] の最新のドキュメントを参照してください。

#### 3. `writeHeapSnapshot` 関数を使用する

> v11.13.0 以降で動作します
> [heapdump パッケージ][] を使用することで、それ以前のバージョンでも動作します

サーバー上で実行されているアプリケーションなど、動作中のプロセスのスナップショットが必要な場合は、以下の方法で取得できます。

```js
require('v8').writeHeapSnapshot();
```

ファイル名のオプションについては、[`writeHeapSnapshot` ドキュメント][] を参照してください。

プロセスを停止せずに呼び出す方法が必要なため、HTTP ハンドラー内、またはオペレーティングシステムからのシグナルへの反応として呼び出すことをお勧めします。スナップショットをトリガーする HTTP エンドポイントを公開しないように注意してください。
他のユーザーがアクセスできないようにする必要があります。

Node.js v11.13.0 より前のバージョンでは、[heapdump パッケージ][] を使用できます。

#### 4. インスペクタープロトコルを使用してヒープスナップショットをトリガーする

インスペクタープロトコルを使用すると、プロセス外部からヒープスナップショットをトリガーできます。

APIを使用するために、Chromiumから実際のインスペクターを実行する必要はありません。

以下は、`websocat`と`jq`を使用したbashでのスナップショットトリガーの例です。

```bash
#!/bin/bash
set -e

kill -USR1 "$1"
rm -f fifo out
mkfifo ./fifo
websocat -B 10000000000 "$(curl -s http://localhost:9229/json | jq -r '.[0].webSocketDebuggerUrl')" < ./fifo > ./out &
exec 3>./fifo
echo '{"method": "HeapProfiler.enable", "id": 1}' > ./fifo
echo '{"method": "HeapProfiler.takeHeapSnapshot", "id": 2}' > ./fifo
while jq -e "[.id != 2, .result != {}] | all" < <(tail -n 1 ./out); do
  sleep 1s
  echo "Capturing Heap Snapshot..."
done

echo -n "" > ./out.heapsnapshot
while read -r line; do
  f="$(echo "$line" | jq -r '.params.chunk')"
  echo -n "$f" >> out.heapsnapshot
  i=$((i+1))
done < <(cat out | tail -n +2 | head -n -1)

exec 3>&-
```

以下は、インスペクタ プロトコルで使用できるメモリ プロファイリング ツールの一覧です (網羅的ではありません)。

- [OpenProfiling for Node.js][openprofiling]

## ヒープスナップショットでメモリリークを見つける方法

2つのスナップショットを比較することで、メモリリークを検出できます。スナップショットの差分に不要な情報が含まれていないことを確認することが重要です。
以下の手順で、スナップショット間のクリーンな差分が生成されます。

1. プロセスがすべてのソースを読み込み、ブートストラップを完了するまで待ちます。最大でも数秒かかります。
2. メモリリークが発生していると思われる機能の使用を開始します。リークの原因ではない初期割り当てが行われている可能性があります。
3. ヒープスナップショットを1つ取得します。
4. しばらくその機能を使用し続けます。できればその間、他の処理を実行しないでください。
5. もう一度ヒープスナップショットを取得します。2つのスナップショットの差分には、リークの原因となっていた内容がほぼ含まれているはずです。
6. Chromium/Chrome 開発者ツールを開き、 _Memory_ タブに移動します。
7. 古いスナップショットファイルを最初に読み込み、次に新しいスナップショットファイルを読み込みます。 ![Load button in tools][load button image]
8. 新しいスナップショットを選択し、上部のドロップダウンでモードを「_Summary_」から「_Comparison_」に切り替えます。![Comparison dropdown][comparison image]
9. 大きな正の差分を探し、下部のパネルでその原因となった参照を調べます。

[このヒープスナップショット演習][heapsnapshot exercise]で、ヒープスナップショットの取得とメモリリークの検出を練習できます。

[open inspector image]: /static/images/docs/guides/diagnostics/tools.png
[take a heap snapshot image]: /static/images/docs/guides/diagnostics/snapshot.png
[heapsnapshot-signal flag]: https://nodejs.org/api/cli.html#--heapsnapshot-signalsignal
[heapdump package]: https://www.npmjs.com/package/heapdump
[`writeHeapSnapshot` docs]: https://nodejs.org/api/v8.html#v8writeheapsnapshotfilenameoptions
[openprofiling]: https://github.com/vmarchaud/openprofiling-node
[load button image]: /static/images/docs/guides/diagnostics/load-snapshot.png
[comparison image]: /static/images/docs/guides/diagnostics/compare.png
[heapsnapshot exercise]: https://github.com/naugtur/node-example-heapdump
[Chrome Developer Tools]: https://developer.chrome.com/docs/devtools/
