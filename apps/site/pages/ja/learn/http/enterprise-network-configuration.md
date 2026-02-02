---
title: Enterprise Network Configuration
layout: learn
authors: joyeecheung
---

# エンタープライズネットワーク構成

## 概要

エンタープライズ環境では、多くの場合、アプリケーションを企業プロキシの背後で動作させ、SSL/TLS 検証にカスタム証明機関 (CA) を使用する必要があります。Node.js は、環境変数とコマンドラインフラグを通じてこれらの要件を組み込みでサポートしているため、多くの場合、サードパーティ製のプロキシライブラリは不要です。

このガイドでは、エンタープライズネットワーク環境で動作するように Node.js アプリケーションを構成する方法について説明します。

- `NODE_USE_ENV_PROXY` 環境変数または `--use-env-proxy` フラグを使用してプロキシを構成する
- `NODE_USE_SYSTEM_CA` 環境変数または `--use-system-ca` フラグを使用してシステムストアから証明機関を追加する

## プロキシ設定

多くの企業環境では、セキュリティと監視のために、外部サービスへのインターネットアクセスをHTTP/HTTPSプロキシ経由でルーティングする必要がある場合があります。そのため、アプリケーションはネットワークリクエストを行う際にこれらのプロキシを認識し、使用する必要があります。

プロキシ設定は、多くの場合、`HTTP_PROXY`、`HTTPS_PROXY`、`NO_PROXY`などの環境変数を介して提供されます。Node.jsは、`NODE_USE_ENV_PROXY`または`--use-env-proxy`が有効になっている場合にこれらをサポートします。これは、`node:http`および`node:https`（v22.21.0またはv24.5.0以降）メソッド、および`fetch()`（v22.21.0またはv24.0.0以降）メソッドで機能します。

例（POSIXシェル）：

```bash
# The proxy settings might be configured in the system by your IT department
# and shared across different tools.
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
export NO_PROXY=localhost,127.0.0.1,.company.com

# To enable it for Node.js applications.
export NODE_USE_ENV_PROXY=1
node app.js
```

あるいは、Node.js v22.21.0 または v24.5.0 以降でコマンドライン フラグ `--use-env-proxy` を使用して有効にします。

```bash
# The proxy settings might be configured in the system by your IT department
# and shared across different tools.
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
export NO_PROXY=localhost,127.0.0.1,.company.com

# To enable it for Node.js applications.
node --use-env-proxy app.js
```

または、`--env-file` を使用してファイルから環境変数を読み込む場合:

```txt
# In .env file
HTTP_PROXY=http://proxy.company.com:8080
HTTPS_PROXY=http://proxy.company.com:8080
NO_PROXY=localhost,127.0.0.1,.company.com
NODE_USE_ENV_PROXY=1
```

```bash
node --env-file ./.env app.js
```

有効にすると、エージェントがオーバーライドされているか、ターゲットが `NO_PROXY` に一致しない限り、`http`、`https`、`fetch()` リクエストはデフォルトで設定されたプロキシを使用します。

### プログラムによるプロキシの設定

プログラムでプロキシを設定するには、エージェントをオーバーライドします。これは現在、`https.request()` と、それを基に構築された `https.get()` などのメソッドでサポートされています。

リクエストごとにエージェントをオーバーライドするには、`http.request()`/`https.request()` などのメソッドで `agent` オプションを使用します。

```cjs
const https = require('node:https');

// Creating a custom agent with custom proxy support.
const agent = new https.Agent({
  proxyEnv: { HTTPS_PROXY: 'http://proxy.company.com:8080' },
});

https.request(
  {
    hostname: 'www.external.com',
    port: 443,
    path: '/',
    agent,
  },
  res => {
    // This request will be proxied through proxy.company.com:8080 using the HTTP protocol.
  }
);
```

```mjs
import https from 'node:https';

// Creating a custom agent with custom proxy support.
const agent = new https.Agent({
  proxyEnv: { HTTPS_PROXY: 'http://proxy.company.com:8080' },
});

https.request(
  {
    hostname: 'www.external.com',
    port: 443,
    path: '/',
    agent,
  },
  res => {
    // This request will be proxied through proxy.company.com:8080 using the HTTP protocol.
  }
);
```

エージェントをグローバルに上書きするには、`http.globalAgent` と `https.globalAgent` をリセットします。

<!-- TODO(joyeecheung): update this when Node.js has a method that supports global configuration for all requesters -->

**注**: グローバル エージェントは `fetch()` に影響しません。

```cjs
const http = require('node:http');
const https = require('node:https');

http.globalAgent = new http.Agent({
  proxyEnv: { HTTP_PROXY: 'http://proxy.company.com:8080' },
});
https.globalAgent = new https.Agent({
  proxyEnv: { HTTPS_PROXY: 'http://proxy.company.com:8080' },
});

// Subsequent requests will all use the configured proxies, unless they override the agent option.
http.request('http://external.com', res => {
  /* ... */
});
https.request('https://external.com', res => {
  /* ... */
});
```

```mjs
import http from 'node:http';
import https from 'node:https';

http.globalAgent = new http.Agent({
  proxyEnv: { HTTP_PROXY: 'http://proxy.company.com:8080' },
});
https.globalAgent = new https.Agent({
  proxyEnv: { HTTPS_PROXY: 'http://proxy.company.com:8080' },
});

// Subsequent requests will all use the configured proxies, unless they override the agent option.
http.request('http://external.com', res => {
  /* ... */
});
https.request('https://external.com', res => {
  /* ... */
});
```

### 認証付きプロキシの使用

プロキシで認証が​​必要な場合は、プロキシURLに認証情報を含めます。

```bash
export HTTPS_PROXY=http://username:password@proxy.company.com:8080
```

**セキュリティに関する注意**: env ファイルに認証情報をコミットすることは避けてください。シークレットマネージャーとプログラムによる設定を推奨します。

### プロキシバイパス設定

`NO_PROXY` 変数は以下をサポートします。

- `*` - すべてのホストでプロキシをバイパス
- `company.com` - ホスト名の完全一致
- `.company.com` - ドメインサフィックスの一致（`sub.company.com` に一致）
- `*.company.com` - ワイルドカードドメインの一致
- `192.168.1.100` - IP アドレスの完全一致
- `192.168.1.1-192.168.1.100` - IP アドレスの範囲
- `company.com:8080` - 特定のポート番号を持つホスト名

ターゲットが `NO_PROXY` に一致する場合、リクエストはプロキシをバイパスします。

## 証明機関の設定

デフォルトでは、Node.js は Mozilla にバンドルされているルート CA を使用し、OS ストアを参照しません。多くのエンタープライズ環境では、内部 CA が OS ストアにインストールされており、社内サービスへの接続時に信頼されることが想定されています。これらの CA によって署名された証明書への接続は、次のようなエラーで検証に失敗する可能性があります。

```
Error: self signed certificate in certificate chain
```

Node.js v22.15.0、v23.9.0、v24.0.0 以降では、システムの証明書ストアを使用してこれらのカスタム CA を信頼するように Node.js を構成できます。

### システムストアからのCA証明書の追加

- 環境変数から: `NODE_USE_SYSTEM_CA=1 node app.js`
- コマンドラインフラグから: `node --use-system-ca app.js`

有効にすると、Node.jsはシステムCAを読み込み、バンドルされているCAに加えてTLS検証に使用します。

Node.jsは、プラットフォームに応じて異なる場所から証明書を読み取ります。

- Windows: Windows証明書ストア（Windows Crypto API経由）
- macOS: macOSキーチェーン
- Linux: OpenSSLのデフォルト（通常は`SSL_CERT_FILE`/`SSL_CERT_DIR`経由、またはOpenSSLのビルドに応じて`/etc/ssl/cert.pem`や`/etc/ssl/certs/`などのパス）

Node.jsはChromiumと同様のポリシーに従います。詳細については、[Node.js のドキュメント](https://nodejs.org/api/cli.html#--use-system-ca)を参照してください。

### CA 証明書の追加

システムストアに依存せずに特定の CA 証明書を追加するには:

```bash
export NODE_EXTRA_CA_CERTS=/path/to/company-ca-bundle.pem
node app.js
```

ファイルには、PEM エンコードされた証明書が 1 つ以上含まれている必要があります。

#### オプションの組み合わせ

`NODE_USE_SYSTEM_CA` と `NODE_EXTRA_CA_CERTS` を組み合わせることができます。

```bash
export NODE_USE_SYSTEM_CA=1
export NODE_EXTRA_CA_CERTS=/path/to/additional-cas.pem
node app.js
```

両方を有効にすると、Node.js はバンドルされた CA、システム CA、および `NODE_EXTRA_CA_CERTS` で指定された追加の証明書を信頼します。

### CA 証明書をプログラムで設定する

#### グローバル CA 証明書を設定する

[`tls.getCACertificates()`](https://nodejs.org/api/tls.html#tlsgetcacertificatestype) と [`tls.setDefaultCACertificates()`](https://nodejs.org/api/tls.html#tlssetdefaultcacertificatescerts) を使用して、グローバル CA 証明書を設定します。たとえば、システム証明書をデフォルトストアに追加するには、次のようにします。

```cjs
const https = require('node:https');
const tls = require('node:tls');
const currentCerts = tls.getCACertificates('default');
const systemCerts = tls.getCACertificates('system');
tls.setDefaultCACertificates([...currentCerts, ...systemCerts]);

// Subsequent requests use system certificates during verification.
https.get('https://internal.company.com', res => {
  /* ... */
});
fetch('https://internal.company.com').then(res => {
  /* ... */
});
```

```mjs
import https from 'node:https';
import tls from 'node:tls';
const currentCerts = tls.getCACertificates('default');
const systemCerts = tls.getCACertificates('system');
tls.setDefaultCACertificates([...currentCerts, ...systemCerts]);

// Subsequent requests use system certificates during verification.
https.get('https://internal.company.com', res => {
  /* ... */
});
fetch('https://internal.company.com').then(res => {
  /* ... */
});
```

#### 個々のリクエストごとに CA 証明書を設定する

リクエストごとに CA 証明書をオーバーライドするには、`ca` オプションを使用します。これは現在、`tls.connect()`/`https.request()` と、それらに基づいて構築された `https.get()` などのメソッドでのみサポートされています。

```cjs
const https = require('node:https');
const specialCerts = ['-----BEGIN CERTIFICATE-----\n...'];
https.get(
  {
    hostname: 'internal.company.com',
    port: 443,
    path: '/',
    method: 'GET',
    // The `ca` option replaces defaults; concatenate bundled certs if needed.
    ca: specialCerts,
  },
  res => {
    /* ... */
  }
);
```

```mjs
import https from 'node:https';
const specialCerts = ['-----BEGIN CERTIFICATE-----\n...'];
https.get(
  {
    hostname: 'internal.company.com',
    port: 443,
    path: '/',
    method: 'GET',
    // The `ca` option replaces defaults; concatenate bundled certs if needed.
    ca: specialCerts,
  },
  res => {
    /* ... */
  }
);
```
