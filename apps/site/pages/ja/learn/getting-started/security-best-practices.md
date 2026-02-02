---
title: Security Best Practices
layout: learn
authors: RafaelGSS, UlisesGascon, fraxken, facutuesca, mhdawson, arhart, naugtur, anonrig
---

# セキュリティのベストプラクティス

## 目的

このドキュメントは、現在の[脅威モデル][]を拡張し、Node.jsアプリケーションを保護する方法に関する広範なガイドラインを提供することを目的としています。

## ドキュメントの内容

- ベストプラクティス：ベストプラクティスを簡潔にまとめたものです。[この問題][セキュリティガイダンスの問題] または [このガイドライン][Node.js ガイドライン] を参考にしてください。このドキュメントは Node.js に特化したものであることにご注意ください。より広範な内容をお探しの場合は、[OSSF ベストプラクティス][] をご検討ください。
- 攻撃の説明：脅威モデルで言及している攻撃について、分かりやすい英語で説明し、可能な場合はコード例もいくつか示します。
- サードパーティライブラリ：脅威（タイポスクワッティング攻撃、悪意のあるパッケージなど）と、Node モジュールの依存関係に関するベストプラクティスを定義します。

## 脅威リスト

Node.js の [脅威モデル][] は、_Node.js 自体の脆弱性_ と見なされるものとそうでないものを定義しています。以下のトピックの一部は、このモデルによれば Node.js コアの脆弱性とはみなされませんが、それでも Node.js ソフトウェアの構築と運用時に考慮すべき重要なアプリケーションレベルの脅威です。

### HTTPサーバーのサービス拒否（CWE-400）

これは、受信したHTTPリクエストの処理方法が原因で、アプリケーションが本来の目的を果たせなくなる攻撃です。これらのリクエストは、悪意のある攻撃者によって意図的に作成される必要はありません。設定ミスやバグのあるクライアントが、サービス拒否を引き起こすようなリクエストパターンをサーバーに送信することもあります。

HTTPリクエストはNode.js HTTPサーバーによって受信され、登録されたリクエストハンドラーを介してアプリケーションコードに渡されます。サーバーはリクエストボディの内容を解析しません。したがって、リクエストハンドラーに渡された後のリクエストボディの内容によって引き起こされるDoSは、Node.js自体の脆弱性ではありません。正しく処理するのはアプリケーションコードの責任です。

Webサーバーがソケットエラーを適切に処理するようにしてください。例えば、エラーハンドラーなしでサーバーを作成した場合、DoS攻撃に対して脆弱になります。

```cjs
const net = require('node:net');

const server = net.createServer(function (socket) {
  // socket.on('error', console.error) // this prevents the server to crash
  socket.write('Echo server\r\n');
  socket.pipe(socket);
});

server.listen(5000, '0.0.0.0');
```

```mjs
import net from 'node:net';

const server = net.createServer(function (socket) {
  // socket.on('error', console.error) // this prevents the server to crash
  socket.write('Echo server\r\n');
  socket.pipe(socket);
});

server.listen(5000, '0.0.0.0');
```

_不正なリクエスト_が実行されると、サーバーがクラッシュする可能性があります。

リクエストの内容が原因ではないDoS攻撃の例として[Slowloris][]が挙げられます。この攻撃では、HTTPリクエストがゆっくりと断片化され、一度に1つのフラグメントとして送信されます。リクエスト全体が送信されるまで、サーバーは進行中のリクエスト専用のリソースを確保し続けます。このようなリクエストが同時に多数送信されると、同時接続数はすぐに最大値に達し、サービス拒否状態になります。このように、攻撃はリクエストの内容ではなく、サーバーに送信されるリクエストのタイミングとパターンに依存します。

**軽減策**

- リバースプロキシを使用して、リクエストの受信と Node.js アプリケーションへの転送を行います。
リバースプロキシは、キャッシュ、負荷分散、IP ブラックリストなどの機能を提供し、DoS 攻撃が効果を発揮する可能性を低減します。
- サーバーのタイムアウトを適切に設定し、アイドル状態の接続やリクエストの到着が遅い接続を破棄できるようにします。[`http.Server`][] の各種タイムアウト、特に `headersTimeout`、`requestTimeout`、`timeout`、`keepAliveTimeout` を参照してください。
- ホストごとおよび合計で、オープンソケットの数を制限します。[http docs][] の特に `agent.maxSockets`、`agent.maxTotalSockets`、`agent.maxFreeSockets`、`server.maxRequestsPerSocket` を参照してください。

### DNSリバインディング (CWE-346)

これは、[--inspect スイッチ][] を使用してデバッグインスペクタを有効にして実行されている Node.js アプリケーションを標的とする攻撃です。

Webブラウザで開かれたウェブサイトはWebSocketおよびHTTPリクエストを送信できるため、ローカルで実行されているデバッグインスペクタを標的とする可能性があります。
これは通常、最新のブラウザに実装されている[同一オリジンポリシー][]によって阻止されます。このポリシーは、スクリプトが異なるオリジンのリソースにアクセスすることを禁止します（つまり、悪意のあるウェブサイトはローカルIPアドレスから要求されたデータを読み取ることができません）。

しかし、DNSリバインディングを利用することで、攻撃者はリクエストのオリジンを一時的に制御し、ローカルIPアドレスから送信されているように見せかけることができます。
これは、ウェブサイトとそのIPアドレス解決に使用されるDNSサーバーの両方を制御することで実現されます。詳細については、[DNSリバインディング wiki][]を参照してください。

**軽減策**

- _SIGUSR1_ シグナルに `process.on(‘SIGUSR1’, …)` リスナーをアタッチして、Inspector を無効化します。
- 本番環境では Inspector プロトコルを実行しないでください。

### 機密情報の不正なアクセス (CWE-552)

パッケージの公開時に、カレントディレクトリに含まれるすべてのファイルとフォルダが npm レジストリにプッシュされます。

この動作を制御するメカニズムとして、`.npmignore` および `.gitignore` でブロックリストを定義するか、`package.json` で許可リストを定義する方法があります。

**軽減策**

- `npm publish --dry-run` を使用して、公開するすべてのファイルをリストします。パッケージを公開する前に、必ず内容を確認してください。
- `.gitignore` や `.npmignore` などの無視ファイルを作成し、管理することも重要です。

これらのファイルを通じて、公開しないファイル/フォルダを指定できます。
`package.json` の [files プロパティ][] を使用すると、逆の操作（許可リスト）が可能です。
- 万が一、漏洩してしまった場合は必ず[パッケージを非公開に][]してください。

### HTTP リクエスト・スマグリング (CWE-444)

これは、2 つの HTTP サーバー（通常はプロキシと Node.js アプリケーション）を介した攻撃です。クライアントが送信した HTTP リクエストは、まずフロントエンドサーバー（プロキシ）を通過し、その後バックエンドサーバー（アプリケーション）にリダイレクトされます。
フロントエンドとバックエンドが曖昧な HTTP リクエストを異なる方法で解釈する場合、攻撃者はフロントエンドでは認識されないもののバックエンドでは認識される悪意のあるメッセージを送信し、プロキシサーバーをすり抜けて「スマグリング」する可能性があります。

詳細な説明と例については、[CWE-444][] を参照してください。

この攻撃は、Node.js が（任意の）HTTP サーバーとは異なる方法で HTTP リクエストを解釈することに依存しているため、攻撃が成功する原因は Node.js、フロントエンドサーバー、またはその両方の脆弱性である可能性があります。
Node.js によるリクエストの解釈方法が HTTP 仕様（[RFC7230][] を参照）に準拠している場合、Node.js の脆弱性とはみなされません。

**緩和策**

- HTTP サーバーを作成する際に `insecureHTTPParser` オプションを使用しないでください。
- フロントエンドサーバーを設定して、曖昧なリクエストを正規化してください。
- Node.js と選択したフロントエンドサーバーの両方で、新たな HTTP リクエストスマグリングの脆弱性がないか継続的に監視してください。
- エンドツーエンドで HTTP/2 を使用し、可能であれば HTTP ダウングレードを無効にしてください。

### タイミング攻撃による情報漏洩 (CWE-208)

これは、例えばアプリケーションがリクエストに応答するまでの時間を測定することで、攻撃者が潜在的に機密性の高い情報を入手できるようにする攻撃です。この攻撃はNode.jsに限ったものではなく、ほぼすべてのランタイムを標的とする可能性があります。

この攻撃は、アプリケーションがタイミングに敏感な操作（例：分岐）でシークレット情報を使用する場合に発生します。一般的なアプリケーションにおける認証処理を考えてみましょう。ここでは、基本的な認証方法にメールアドレスとパスワードが資格情報として含まれています。
ユーザー情報は、ユーザーが入力した情報（理想的にはDBMS）から取得されます。
ユーザー情報を取得すると、パスワードはデータベースから取得されたユーザー情報と比較されます。組み込みの文字列比較を使用すると、同じ長さの値に対して時間がかかります。
この比較を許容できる時間だけ実行すると、リクエストの応答時間が意図せず長くなります。リクエストの応答時間を比較することで、攻撃者は大量のリクエストからパスワードの長さと値を推測できます。

**緩和策**

- crypto API は、定数時間アルゴリズムを用いて実際の機密値と想定される機密値を比較する関数 `timingSafeEqual` を公開しています。

- パスワードの比較には、ネイティブ crypto モジュールでも利用可能な [scrypt][] を使用できます。

- より一般的には、可変時間演算でシークレットを使用することは避けてください。これには、シークレットに基づく分岐や、攻撃者が同じインフラストラクチャ（同じクラウドマシンなど）上に共存する可能性がある場合にシークレットをメモリのインデックスとして使用することが含まれます。JavaScript で定数時間コードを記述するのは困難です（JIT の影響も一因です）。暗号アプリケーションでは、組み込みの暗号 API を使用するか、WebAssembly（ネイティブに実装されていないアルゴリズムの場合）を使用してください。

### 悪意のあるサードパーティ製モジュール (CWE-1357)

Node.js の [脅威モデル][] によると、悪意のあるサードパーティ製モジュールを必要とするシナリオは、Node.js コアの脆弱性とはみなされません。これは、Node.js が実行を要求されたコード（依存関係を含む）を信頼できるものとして扱うためです。しかし、悪意のある依存関係や侵害された依存関係は、Node.js ユーザーにとって依然として最も重大なアプリケーションレベルのリスクの一つであり、そのように対処する必要があります。

現在、Node.js では、あらゆるパッケージがネットワークアクセスなどの強力なリソースにアクセスできます。
さらに、ファイルシステムにもアクセスできるため、あらゆるデータをどこにでも送信できます。

Node プロセスで実行されるすべてのコードは、`eval()`（または同等の関数）を使用して任意のコードを追加読み込み、実行できます。
ファイルシステムへの書き込みアクセス権を持つすべてのコードは、読み込み済みの新規ファイルまたは既存のファイルに書き込むことで、同様のことを実現する可能性があります。

**例**

- 攻撃者が、よく知られているロギングライブラリのメンテナーアカウントを侵害し、ロガーの初期化時に環境変数（データベースのパスワードやアクセストークンなど）をリモートサーバーに盗み出す新しいマイナーバージョンを配布します。
- よく知られているフレームワークに似た名前を持つ、タイポスクワッティングパッケージがnpmレジストリに公開されます。インストールされると、開発者のマシンから攻撃者が管理するエンドポイントにSSHキーを送信するインストール後スクリプトが実行されます。

依存関係のバージョンを固定し、一般的なワークフローまたはnpmスクリプトを使用して脆弱性の自動チェックを実行してください。
パッケージをインストールする前に、そのパッケージがメンテナンスされており、期待されるすべてのコンテンツが含まれていることを確認してください。
GitHubのソースコードは公開されているものと必ずしも同じではないため、_node_modules_で検証してください。

#### サプライチェーン攻撃

Node.js アプリケーションに対するサプライチェーン攻撃は、その依存関係（直接的または推移的）のいずれかが侵害されたときに発生します。
これは、アプリケーションが依存関係の仕様を緩く規定している（望ましくないアップデートを許容する）か、仕様によくあるタイプミス（[タイポスクワッティング][] の脆弱性）のいずれかが原因で発生する可能性があります。

攻撃者がアップストリームパッケージを掌握すると、悪意のあるコードを含む新しいバージョンを公開できます。Node.js アプリケーションが、どのバージョンを安全に使用できるかを厳密に規定せずにそのパッケージに依存している場合、パッケージは最新の悪意のあるバージョンに自動的にアップデートされ、アプリケーションが侵害される可能性があります。

`package.json` ファイルで指定された依存関係には、正確なバージョン番号または範囲を指定できます。ただし、依存関係を正確なバージョンに固定する場合、その推移的依存関係自体は固定されません。
これにより、アプリケーションは望ましくない、または予期しないアップデートに対して脆弱なままになります。

考えられる攻撃経路:

- タイポスクワッティング攻撃
- ロックファイルポイズニング
- メンテナーのセキュリティ侵害
- 悪意のあるパッケージ
- 依存関係の混乱

**緩和策**

- `--ignore-scripts` を使用して、npm による任意のスクリプトの実行を防止します。
  - さらに、`npm config set ignore-scripts true` を使用して、これをグローバルに無効化できます。
- 依存関係のバージョンを、範囲指定や変更可能なソースからのバージョンではなく、特定の不変バージョンに固定します。
- すべての依存関係 (直接的および推移的) を固定するロックファイルを使用します。
  - [ロックファイルポイズニングの緩和策][] を使用します。
- [`npm-audit`][] などのツールを用いて、CI による新しい脆弱性のチェックを自動化します。
  - [`Socket`][] などのツールを用いて、静的解析によってパッケージを分析し、ネットワークやファイルシステムへのアクセスなどの危険な動作を見つけることができます。
- `npm install` の代わりに [`npm ci`][] を使用してください。これにより、ロックファイルが強制的に適用され、_package.json_ ファイルとの不整合が発生した場合にエラーが発生します（ロックファイルを黙って無視し、_package.json_ を優先するのではなく）。
- _package.json_ ファイルで、依存関係の名前に誤りやタイプミスがないか注意深く確認してください。

### メモリアクセス違反 (CWE-284)

メモリベースまたはヒープベースの攻撃は、メモリ管理エラーと悪用可能なメモリアロケータの組み合わせに依存します。
他のランタイムと同様に、Node.js はプロジェクトを共有マシンで実行する場合、これらの攻撃に対して脆弱です。
セキュアヒープの使用は、ポインタオーバーランやアンダーランによる機密情報の漏洩を防ぐのに役立ちます。

残念ながら、セキュアヒープは Windows では利用できません。
詳細については、Node.js [セキュアヒープのドキュメント][] を参照してください。

**緩和策**

- アプリケーションに応じて `--secure-heap=n` を使用してください。_n_ は割り当てられた最大バイトサイズです。
- 本番環境アプリを共有マシンで実行しないでください。

### モンキーパッチ (CWE-349)

モンキーパッチとは、実行時にプロパティを変更し、既存の動作を変更することを指します。例:

```js
Array.prototype.push = function (item) {
  // overriding the global [].push
};
```

**緩和策**

`--frozen-intrinsics` フラグは、試験的な[¹][experimental-features] の凍結された組み込み関数を有効にします。これは、すべての組み込み JavaScript オブジェクトと関数が再帰的に凍結されることを意味します。
したがって、以下のスニペットは `Array.prototype.push` のデフォルトの動作を**オーバーライドしません**。

```js
Array.prototype.push = function (item) {
  // overriding the global [].push
};

// Uncaught:
// TypeError <Object <Object <[Object: null prototype] {}>>>:
// Cannot assign to read only property 'push' of object ''
```

ただし、`globalThis` を使用して新しいグローバルを定義し、既存のグローバルを置き換えることができることに注意してください。

```console
> globalThis.foo = 3; foo; // you can still define new globals
3
> globalThis.Array = 4; Array; // However, you can also replace existing globals
4
```

したがって、`Object.freeze(globalThis)` を使用すると、グローバルが置き換えられないことを保証できます。

### プロトタイプ汚染攻撃 (CWE-1321)

Node.js の [脅威モデル][] によれば、攻撃者がユーザー入力を制御することに依存するプロトタイプ汚染は、Node.js コアの脆弱性とは**みなされません**。これは、Node.js がアプリケーションコードによって提供される入力を信頼しているためです。
しかしながら、プロトタイプ汚染は Node.js アプリケーションおよびサードパーティライブラリにとって深刻な脆弱性であり、アプリケーションレベルおよび依存関係レベルで防御策を実装する必要があります。

プロトタイプ汚染とは、\_\_proto\__、\_constructor_、_prototype_、および組み込みプロトタイプから継承されたその他のプロパティの使用を悪用することにより、JavaScript 言語の項目にプロパティを変更または挿入する可能性があることを指します。

<!-- eslint-skip -->

```js
const a = { a: 1, b: 2 };
const data = JSON.parse('{"__proto__": { "polluted": true}}');

const c = Object.assign({}, a, data);
console.log(c.polluted); // true

// Potential DoS
const data2 = JSON.parse('{"__proto__": null}');
const d = Object.assign(a, data2);
d.hasOwnProperty('b'); // Uncaught TypeError: d.hasOwnProperty is not a function
```

これは、JavaScript 言語から継承された潜在的な脆弱性です。

**例**:

- [CVE-2022-21824][] (Node.js)
- [CVE-2018-3721][] (サードパーティライブラリ: Lodash)

その他のシナリオ:

- Web API が、信頼できない JSON リクエストボディを検証なしで共有設定オブジェクトにマージします。攻撃者は `__proto__` プロパティを含むペイロードを送信することで、その過程で多くのオブジェクトに予期しないプロパティを追加し、ロジックのバグやサービス拒否を引き起こします。
- テンプレートレンダリングサービスは、ユーザーが制御するオプションを受け取り、それらをディープマージユーティリティに直接渡します。攻撃者は `Object.prototype` を汚染することで、今後作成されるすべてのテンプレートに予期しない動作をさせ、オブジェクトプロパティの存在に依存するセキュリティチェックをバイパスする可能性があります。

**緩和策**

- [安全でない再帰マージ][] を回避してください。[CVE-2018-16487][] を参照してください。
- 外部/信頼できないリクエストに対して JSON Schema 検証を実装してください。
- `Object.create(null)` を使用して、プロトタイプなしでオブジェクトを作成してください。
- プロトタイプを凍結するには、`Object.freeze(MyObject.prototype)` を使用してください。
- `--disable-proto` フラグを使用して、`Object.prototype.__proto__` プロパティを無効にしてください。
- `Object.hasOwn(obj, keyFromObj)` を使用して、プロパティがプロトタイプではなくオブジェクトに直接存在することを確認してください。
- `Object.prototype` のメソッドの使用は避けてください。

### 制御されていない検索パス要素 (CWE-427)

Node.js の [脅威モデル][] では、Node.js がアクセスできる環境内のファイルシステムは信頼できるものとみなされます。そのため、これらの場所にあるファイルの制御のみに依存する問題は、Node.js コアの脆弱性とは**みなされません**。ただし、これらはデプロイメント全体とサプライチェーンのセキュリティに関連するため、環境を強化し、以下のメカニズムを使用してリスクを軽減する必要があります。

Node.js は [モジュール解決アルゴリズム][] に従ってモジュールをロードします。
したがって、モジュールが要求されるディレクトリ (require) は信頼できるものと想定されます。

つまり、次のようなアプリケーションの動作が想定されます。
以下のディレクトリ構造を前提とします。

- _app/_
  - _server.js_
  - _auth.js_
  - _auth_

server.js が `require('./auth')` を使用する場合、モジュール解決アルゴリズムに従い、_auth.js_ ではなく _auth_ をロードします。

## Node.js の権限モデル

Node.js は、実行時に特定のプロセスに許可される操作を制限できる **権限モデル** を提供します。
このモデルは、Node.js の [脅威モデル][] を補完するものです。

権限モデルを有効にすると（たとえば、`--permission` フラグを使用）、次のような機密性の高い機能へのアクセスを個別に許可または拒否できます。

- ファイルシステムの読み取りと書き込み。
- ネットワークアクセス（インバウンドおよびアウトバウンド）。
- 子プロセスの作成。
- ネイティブアドオンやその他の強力な API の使用。

これにより、悪意のある依存関係や侵害された依存関係、信頼できない構成、または独自のコード内の予期しない動作の影響を抑えることができます。信頼できるコードであっても、明示的に付与した権限の範囲外のアクションを実行できないようにするためです。

最新のフラグとオプションについては、[Node.js の権限に関するドキュメント][] を参照してください。

## 本番環境での試験的機能の使用

試験的機能を本番環境で使用することは推奨されません。
試験的機能は、必要に応じて互換性を破る変更が加えられる可能性があり、その機能は完全に安定しているわけではありません。フィードバックは大歓迎です。

## OpenSSF ツール

[OpenSSF][] は、特に npm パッケージを公開する予定がある場合に非常に役立つ、いくつかのイニシアチブを主導しています。これらのイニシアチブには以下が含まれます。

- [OpenSSF スコアカード][] スコアカードは、一連の自動化されたセキュリティリスクチェックを使用してオープンソースプロジェクトを評価します。これを使用することで、コードベース内の脆弱性と依存関係をプロアクティブに評価し、脆弱性を受け入れるかどうかについて情報に基づいた判断を行うことができます。
- [OpenSSF ベストプラクティス バッジ プログラム][] プロジェクトは、各ベストプラクティスへの準拠状況を記述することで、自主的に自己認証できます。これにより、プロジェクトに追加できるバッジが生成されます。

また、[OpenJS セキュリティ コラボレーション スペース][] を通じて、他のプロジェクトやセキュリティ専門家とコラボレーションすることもできます。

[threat model]: https://github.com/nodejs/node/security/policy#the-nodejs-threat-model
[security guidance issue]: https://github.com/nodejs/security-wg/issues/488
[nodejs guideline]: https://github.com/goldbergyoni/nodebestpractices
[OSSF Best Practices]: https://github.com/ossf/wg-best-practices-os-developers
[Slowloris]: https://en.wikipedia.org/wiki/Slowloris_(computer_security)
[`http.Server`]: https://nodejs.org/api/http.html#class-httpserver
[http docs]: https://nodejs.org/api/http.html
[--inspect switch]: /learn/getting-started/debugging
[same-origin policy]: /learn/getting-started/debugging
[DNS Rebinding wiki]: https://en.wikipedia.org/wiki/DNS_rebinding
[files property]: https://docs.npmjs.com/cli/configuring-npm/package-json#files
[unpublish the package]: https://docs.npmjs.com/unpublishing-packages-from-the-registry
[CWE-444]: https://cwe.mitre.org/data/definitions/444.html
[RFC7230]: https://datatracker.ietf.org/doc/html/rfc7230#section-3
[Node.js permissions documentation]: https://nodejs.org/api/permissions.html#permission-model
[policy mechanism]: https://nodejs.org/api/permissions.html#policies
[typosquatting]: https://en.wikipedia.org/wiki/Typosquatting
[Mitigations for lockfile poisoning]: https://blog.ulisesgascon.com/lockfile-posioned
[`npm-audit`]: https://docs.npmjs.com/cli/commands/npm-audit
[`npm ci`]: https://docs.npmjs.com/cli/v8/commands/npm-ci
[secure-heap documentation]: https://nodejs.org/dist/latest-v18.x/docs/api/cli.html#--secure-heapn
[CVE-2022-21824]: https://www.cvedetails.com/cve/CVE-2022-21824/
[CVE-2018-3721]: https://www.cvedetails.com/cve/CVE-2018-3721/
[insecure recursive merges]: https://gist.github.com/DaniAkash/b3d7159fddcff0a9ee035bd10e34b277#file-unsafe-merge-js
[CVE-2018-16487]: https://www.cve.org/CVERecord?id=CVE-2018-16487
[scrypt]: https://nodejs.org/api/crypto.html#cryptoscryptpassword-salt-keylen-options-callback
[Module Resolution Algorithm]: https://nodejs.org/api/modules.html#modules_all_together
[experimental-features]: #experimental-features-in-production
[`Socket`]: https://socket.dev/
[OpenSSF]: https://openssf.org/
[OpenSSF Scorecard]: https://securityscorecards.dev/
[OpenSSF Best Practices Badge Program]: https://bestpractices.coreinfrastructure.org/en
[OpenJS Security Collaboration Space]: https://github.com/openjs-foundation/security-collab-space
