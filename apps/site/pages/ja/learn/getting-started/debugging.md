---
title: Debugging Node.js
layout: learn
---

# Node.js のデバッグ

このガイドは、Node.js アプリとスクリプトのデバッグを始めるのに役立ちます。

## インスペクターを有効にする

`--inspect` スイッチを指定して起動すると、Node.js プロセスはデバッグクライアントをリッスンします。デフォルトでは、ホストとポート番号 127.0.0.1:9229 をリッスンします。
各プロセスには一意の [UUID][] も割り当てられます。

インスペクタークライアントは、接続するためにホストアドレス、ポート番号、および UUID を認識し、指定する必要があります。
完全な URL は `ws://127.0.0.1:9229/0f2c936f-b1cd-4ac9-aab3-f63b0f33d55e` のようになります。

Node.js は `SIGUSR1` シグナルを受信すると、デバッグメッセージのリッスンも開始します。(`SIGUSR1` は Windows では使用できません。) Node.js 7 以前では、これによりレガシーのデバッガー API が有効化されます。Node.js 8 以降では、インスペクター API が有効化されます。

## セキュリティへの影響

デバッガはNode.js実行環境への完全なアクセス権を持つため、このポートに接続できる悪意のある攻撃者は、Node.jsプロセスに代わって任意のコードを実行できる可能性があります。パブリックネットワークおよびプライベートネットワーク上でデバッガポートを公開することによるセキュリティへの影響を理解することが重要です。

### デバッグポートを公開することは安全ではありません。

デバッガーがパブリックIPアドレス、または0.0.0.0にバインドされている場合、そのIPアドレスにアクセスできるすべてのクライアントがデバッガーに制限なく接続し、任意のコードを実行できるようになります。

デフォルトでは、`node --inspect` は127.0.0.1にバインドされます。デバッガーへの外部接続を許可する場合は、パブリックIPアドレスまたは0.0.0.0などを明示的に指定する必要があります。そうしないと、重大なセキュリティ上の脅威にさらされる可能性があります。セキュリティ上のリスクを防ぐため、適切なファイアウォールとアクセス制御を導入することをお勧めします。

リモートデバッガークライアントの接続を安全に許可する方法については、「[リモートデバッグシナリオの有効化](#enabling-remote-debugging-scenarios)」セクションを参照してください。

### ローカルアプリケーションはインスペクタにフルアクセスできます。

インスペクタのポートを127.0.0.1（デフォルト）にバインドした場合でも、ローカルマシンで実行されているアプリケーションは無制限にアクセスできます。これは、ローカルデバッガが簡単にアタッチできるようにするための設計です。

### ブラウザ、WebSocket、同一オリジンポリシー

Webブラウザで開いたウェブサイトは、ブラウザのセキュリティモデルに基づいてWebSocketおよびHTTPリクエストを送信できます。一意のデバッガセッションIDを取得するには、最初のHTTP接続が必要です。同一オリジンポリシーは、ウェブサイトがこのHTTP接続を確立できないようにします。[DNSリバインディング攻撃](https://en.wikipedia.org/wiki/DNS_rebinding)に対するセキュリティを強化するため、Node.jsは
接続の「Host」ヘッダーにIPアドレスまたは「localhost」が正確に指定されていることを確認します。

これらのセキュリティポリシーは、ホスト名を指定してリモートデバッグサーバーに接続することを禁止します。この制限を回避するには、IPアドレスを指定するか、以下で説明するSSHトンネルを使用します。

## Inspector クライアント

`node inspect myscript.js` で、最小限の CLI デバッガーが利用できます。
いくつかの商用ツールやオープンソースツールも Node.js Inspector に接続できます。

### Chrome DevTools 55+、Microsoft Edge

#### オプション 1: 組み込みの DevTools UI を使用する

- ブラウザで `chrome://inspect` (Microsoft Edge の場合は `edge://inspect`) を開きます。
- [構成] ボタンをクリックし、ターゲットホストとポートがリストされていることを確認します。
- Node.js アプリケーションが [リモートターゲット] リストに表示されます。

#### オプション 2: 手動で接続する

- `http://localhost:<inspect-port>/json/list` にアクセスします。`devtoolsFrontendUrl` を含む JSON オブジェクトが返されます。
- レスポンスから `devtoolsFrontendUrl` の値をコピーし、ブラウザーのアドレスバーに貼り付けます。

詳細については、[Chrome DevTools フロントエンド](https://github.com/ChromeDevTools/devtools-frontend) および [Microsoft Edge DevTools ガイド](https://learn.microsoft.com/microsoft-edge/devtools-guide-chromium/) をご覧ください。

### Visual Studio Code 1.10 以降

- デバッグパネルで設定アイコンをクリックし、`.vscode/launch.json` を開きます。初期設定では「Node.js」を選択してください。

詳しくは https://code.visualstudio.com/docs/nodejs/nodejs-debugging をご覧ください。

### Visual Studio 2017 以降

- メニューから「デバッグ > デバッグの開始」を選択するか、F5 キーを押します。
- [詳細な手順](https://github.com/Microsoft/nodejstools/wiki/Debugging)

### JetBrains WebStorm およびその他の JetBrains IDE

- 新しい Node.js デバッグ構成を作成し、「デバッグ」をクリックします。Node.js 7 以降では、デフォルトで `--inspect` が使用されます。無効にするには、IDE レジストリで `js.debugger.node.use.inspect` のチェックを外してください。WebStorm およびその他の JetBrains IDE での Node.js の実行とデバッグの詳細については、[WebStorm オンラインヘルプ](https://www.jetbrains.com/help/webstorm/running-and-debugging-node-js.html) をご覧ください。

### chrome-remote-interface

- [Inspector Protocol][] エンドポイントへの接続を容易にするライブラリ。

詳細については、https://github.com/cyrus-and/chrome-remote-interface をご覧ください。

### Eclipse Wild Web Developer 拡張機能を搭載した Eclipse IDE

- .js ファイルから、「デバッグ... > Node プログラム」を選択するか、
- 実行中の Node.js アプリケーションにデバッガーをアタッチするためのデバッグ構成を作成します（`--inspect` で既に起動済み）。

詳細については、https://eclipseide.org/ をご覧ください。

## コマンドラインオプション

以下の表は、さまざまなランタイムフラグがデバッグに与える影響を示しています。

| フラグ | 意味 |
| ------ | ------- |
| --inspect | インスペクタ エージェントを有効にします。デフォルトのアドレスとポート (127.0.0.1:9229) でリッスンします。 |
| --inspect=\[host:port] | インスペクタ エージェントを有効にします。アドレスまたはホスト名 `host` (デフォルト: 127.0.0.1) にバインドします。ポート `port` (デフォルト: 9229) でリッスンします。 |
| --inspect-brk | インスペクタ エージェントを有効にします。デフォルトのアドレスとポート (127.0.0.1:9229) でリッスンします。ユーザー コードの開始前にブレークします。 |
| --inspect-brk=\[host:port] | インスペクタ エージェントを有効にします。アドレスまたはホスト名 `host` (デフォルト: 127.0.0.1) にバインドします。ポート `port` (デフォルト: 9229) でリッスンします。ユーザー コードの開始前にブレークします。 |
| --inspect-wait |インスペクタ エージェントを有効にします。デフォルトのアドレスとポート (127.0.0.1:9229) を listen します。デバッガがアタッチされるまで待機します。 |
| --inspect-wait=\[host:port] | インスペクタ エージェントを有効にします。アドレスまたはホスト名 `host` (デフォルト: 127.0.0.1) にバインドします。ポート `port` (デフォルト: 9229) を listen します。デバッガがアタッチされるまで待機します。 |
| --disable-sigusr1 | プロセスに SIGUSR1 シグナルを送信してデバッグ セッションを開始する機能を無効にします。 |
| node inspect script.js | --inspect フラグでユーザのスクリプトを実行する子プロセスを生成し、メイン プロセスを使用して CLI デバッガを実行します。 |
| node inspect --port=xxxx script.js | --inspect フラグでユーザのスクリプトを実行する子プロセスを生成し、メイン プロセスを使用して CLI デバッガを実行します。ポート `port` (デフォルト: 9229) を listen します。 |

## リモートデバッグの有効化

デバッガーをパブリックIPアドレスでリッスンさせないことを推奨します。リモートデバッグ接続を許可する必要がある場合は、代わりにSSHトンネルを使用することをお勧めします。以下の例は説明のみを目的としています。
続行する前に、特権サービスへのリモートアクセスを許可することによるセキュリティリスクを理解してください。

デバッグを可能にするリモートマシン（remote.example.com）でNode.jsを実行しているとします。そのマシンでは、インスペクターがlocalhost（デフォルト）のみをリッスンするようにNodeプロセスを開始する必要があります。

```bash
node --inspect server.js
```

これで、デバッグ クライアント接続を開始するローカル マシンで、SSH トンネルを設定できます。

```bash
ssh -L 9221:localhost:9229 user@remote.example.com
```

これにより、SSHトンネルセッションが開始され、ローカルマシンのポート9221への接続がremote.example.comのポート9229に転送されます。これで、Chrome DevToolsやVisual Studio Codeなどのデバッガーをlocalhost:9221にアタッチできるようになりました。これにより、Node.jsアプリケーションがローカルで実行されているかのようにデバッグできるようになります。

## レガシーデバッガー

**レガシーデバッガーはNode.js 7.7.0以降で非推奨となりました。代わりに `--inspect` と Inspector を使用してください。**

バージョン7以前で **--debug** または **--debug-brk** スイッチを指定して起動した場合、Node.jsは廃止されたV8デバッグプロトコルで定義されたデバッグコマンドをTCPポート（デフォルトでは `5858`）でリッスンします。このプロトコルをサポートするデバッガークライアントであれば、実行中のプロセスに接続してデバッグできます。以下に、よく知られているデバッガークライアントをいくつか示します。

V8デバッグプロトコルは、メンテナンスもドキュメント化も終了しました。

### Built-in Debugger

Start `node debug script_name.js` to start your script under the builtin command-line debugger. Your script starts in another Node.js process started with the `--debug-brk` option, and the initial Node.js process runs the `_debugger.js` script and connects to your target. See [docs](https://nodejs.org/dist/latest/docs/api/debugger.html) for more information.

### node-inspector

Chrome DevTools で Node.js アプリをデバッグするには、Chromium で使用される [Inspector Protocol][] を Node.js で使用される V8 デバッガープロトコルに変換する中間プロセスを使用します。詳細については、https://github.com/node-inspector/node-inspector をご覧ください。

[Inspector Protocol]: https://chromedevtools.github.io/debugger-protocol-viewer/v8/
[UUID]: https://tools.ietf.org/html/rfc4122
