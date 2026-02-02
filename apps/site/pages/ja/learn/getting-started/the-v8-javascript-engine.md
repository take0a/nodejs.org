---
title: The V8 JavaScript Engine
layout: learn
authors: flaviocopes, smfoote, co16353sidak, MylesBorins, LaRuaNa, andys8, ahmadawais, karlhorky, aymen94
---

# V8 JavaScript エンジン

V8 は、Google Chrome を動かす JavaScript エンジンの名前です。Chrome でブラウジングしているときに、JavaScript を受け取り、実行するものです。

V8 は JavaScript エンジンであり、JavaScript コードを解析して実行します。DOM やその他の Web プラットフォーム API（これらはすべてランタイム環境を構成します）はブラウザによって提供されます。

JavaScript エンジンの優れた点は、ホストされているブラウザから独立していることです。この重要な特徴が Node.js の台頭を可能にしました。V8 は 2009 年に Node.js を動かすエンジンとして選ばれ、Node.js の人気が爆発的に高まるにつれて、V8 は現在では JavaScript で書かれた膨大な数のサーバーサイドコードを動かすエンジンとなりました。

Node.js エコシステムは巨大で、Electron などのプロジェクトを含むデスクトップアプリも V8 によって支えられています。

## その他の JavaScript エンジン

他のブラウザには独自の JavaScript エンジンがあります。

- Firefox には [**SpiderMonkey**](https://spidermonkey.dev) が搭載されています。
- Safari には [**JavaScriptCore**](https://developer.apple.com/documentation/javascriptcore) (Nitro とも呼ばれます) が搭載されています。
- Edge は元々 [**Chakra**](https://github.com/Microsoft/ChakraCore) をベースにしていましたが、最近では [Chromium](https://support.microsoft.com/en-us/help/4501095/download-the-new-microsoft-edge-based-on-chromium) と V8 エンジンを使用して再構築されました。

他にも多くのブラウザが存在します。

これらのエンジンはすべて、JavaScript で使用される標準である [ECMA ES-262 標準](https://www.ecma-international.org/publications/standards/Ecma-262.htm) (ECMAScript とも呼ばれる) を実装しています。

## パフォーマンスの追求

V8はC++で書かれており、継続的に改良されています。移植性が高く、Mac、Windows、Linux、その他多くのシステムで動作します。

このV8の紹介では、V8の実装の詳細については触れません。実装の詳細は、より権威のあるサイト（例えば[V8公式サイト](https://v8.dev/)）で確認でき、時間の経過とともに、そして多くの場合、劇的に変化します。

V8は、他のJavaScriptエンジンと同様に、WebとNode.jsエコシステムの高速化のために常に進化しています。

Webでは、長年にわたりパフォーマンスを競い合っており、私たち（ユーザーと開発者）は、年々高速化され最適化されたマシンを手に入れることで、この競争から大きな恩恵を受けています。

## コンパイル

JavaScript は一般的にインタープリタ型言語と考えられていますが、現代の JavaScript エンジンはもはや JavaScript を単に解釈するだけでなく、コンパイルも行います。

これは 2009 年に Firefox 3.5 に SpiderMonkey JavaScript コンパイラが追加されて以来のことで、誰もがこの考え方を採用しました。

JavaScript は、実行速度を高速化するために V8 によって内部的に **ジャストインタイム** (JIT) **コンパイル** によってコンパイルされます。

直感に反するように思えるかもしれませんが、2004 年に Google マップが導入されて以来、JavaScript は数十行のコードを実行する言語から、ブラウザ内で数千行から数十万行に及ぶ完全なアプリケーションへと進化しました。

今では、アプリケーションは数個のフォーム検証ルールや単純なスクリプトではなく、ブラウザ内で何時間も実行できます。

この _新しい世界_ では、JavaScript をコンパイルすることは完全に理にかなっています。JavaScript を _準備_ するのに少し時間がかかるかもしれませんが、完了すると、純粋に解釈されたコードよりもはるかにパフォーマンスが向上するからです。
