---
title: Asynchronous flow control
layout: learn
authors: aug2uag, ovflowd
---

# 非同期フロー制御

> この投稿の内容は、[Mixu の Node.js ブック](http://book.mixu.net/node/ch7.html) に大きく影響を受けています。

JavaScript は、ビューがレンダリングされる「メイン」スレッドではノンブロッキングになるように設計されています。ブラウザにおいてこれがいかに重要であるかは想像に難くないでしょう。メインスレッドがブロックされると、エンドユーザーが恐れる悪名高い「フリーズ」が発生し、他のイベントがディスパッチされなくなり、例えばデータ取得が失われるといった事態に陥ります。

これは、関数型プログラミングスタイルでしか解決できない特有の制約を生み出します。ここでコールバックが役立ちます。

しかし、コールバックは複雑な処理では扱いが困難になることがあります。その結果、「コールバック地獄」に陥ることがよくあります。コールバックを含む複数のネストされた関数によって、コードの可読性、デバッグ、整理などが困難になるのです。

```js
async1(function (input, result1) {
  async2(function (result2) {
    async3(function (result3) {
      async4(function (result4) {
        async5(function (output) {
          // do something with output
        });
      });
    });
  });
});
```

もちろん、現実世界では `result1`、`result2` などを処理するために追加のコード行が必要になる可能性が高いため、この問題の長さと複雑さにより、上記の例よりもはるかに複雑なコードになることがよくあります。

**ここで _関数_ が大いに役立ちます。より複雑な操作は、多くの関数で構成されます。**

1. イニシエータースタイル / 入力
2. ミドルウェア
3. ターミネータ

**「イニシエータースタイル / 入力」は、シーケンスの最初の関数です。この関数は、操作の元の入力（存在する場合）を受け取ります。操作は実行可能な一連の関数であり、元の入力は主に次のようになります。**

1. グローバル環境内の変数
2. 引数ありまたはなしの直接呼び出し
3. ファイルシステムまたはネットワーク要求によって取得された値

ネットワーク要求とは、外部ネットワーク、同じネットワーク上の別のアプリケーション、または同じネットワークまたは外部ネットワーク上のアプリケーション自身によって開始された着信要求のことです。

ミドルウェア関数は別の関数を返し、ターミネータ関数はコールバックを呼び出します。以下は、ネットワークまたはファイルシステムリクエストへのフローを示しています。これらの値はすべてメモリ内で利用可能なため、レイテンシは0です。

```js
function final(someInput, callback) {
  callback(`${someInput} and terminated by executing callback `);
}

function middleware(someInput, callback) {
  return final(`${someInput} touched by middleware `, callback);
}

function initiate() {
  const someInput = 'hello this is a function ';
  middleware(someInput, function (result) {
    console.log(result);
    // requires callback to `return` result
  });
}

initiate();
```

## 状態管理

関数は状態に依存する場合とそうでない場合があります。状態依存性は、関数の入力やその他の変数が外部関数に依存している場合に発生します。

**このように、状態管理には主に2つの戦略があります。**

1. 関数に変数を直接渡す。
2. キャッシュ、セッション、ファイル、データベース、ネットワーク、その他の外部ソースから変数の値を取得する。

グローバル変数については触れていません。グローバル変数による状態管理は、しばしばずさんなアンチパターンとなり、状態の保証が困難、あるいは不可能になります。複雑なプログラムでは、可能な限りグローバル変数の使用を避けるべきです。

## 制御フロー

オブジェクトがメモリ上で利用可能な場合、反復処理が可能であり、制御フローは変更されません。

```js
function getSong() {
  let _song = '';
  let i = 100;
  for (i; i > 0; i -= 1) {
    _song += `${i} beers on the wall, you take one down and pass it around, ${
      i - 1
    } bottles of beer on the wall\n`;
    if (i === 1) {
      _song += "Hey let's get some more beer";
    }
  }

  return _song;
}

function singSong(_song) {
  if (!_song) {
    throw new Error("song is '' empty, FEED ME A SONG!");
  }

  console.log(_song);
}

const song = getSong();
// this will work
singSong(song);
```

ただし、データがメモリ外に存在する場合、反復は機能しなくなります。

```js
function getSong() {
  let _song = '';
  let i = 100;
  for (i; i > 0; i -= 1) {
    setTimeout(function () {
      _song += `${i} beers on the wall, you take one down and pass it around, ${
        i - 1
      } bottles of beer on the wall\n`;
      if (i === 1) {
        _song += "Hey let's get some more beer";
      }
    }, 0);
  }

  return _song;
}

function singSong(_song) {
  if (!_song) {
    throw new Error("song is '' empty, FEED ME A SONG!");
  }

  console.log(_song);
}

const song = getSong('beer');
// this will not work
singSong(song);
// Uncaught Error: song is '' empty, FEED ME A SONG!
```

なぜこのようなことが起こるのでしょうか？`setTimeout` は、CPU に命令をバス上の別の場所に保存するよう指示し、データは後で取得するようにスケジュール設定します。関数が 0 ミリ秒の時点で再び実行され、CPU がバスから命令を取得して実行するまでに、数千 CPU サイクルが経過します。唯一の問題は、song ('') が数千サイクル前に返されていたことです。

ファイルシステムやネットワーク要求の処理でも同じ状況が発生します。メインスレッドを不確定な時間ブロックすることはできません。そのため、コールバックを使用して、制御された方法で時間内にコードの実行をスケジュールします。

ほぼすべての操作は、次の 3 つのパターンで実行できます。

1. **直列:** 関数は厳密に順番に実行されます。これは `for` ループに最も似ています。

```js
// operations defined elsewhere and ready to execute
const operations = [
  { func: function1, args: args1 },
  { func: function2, args: args2 },
  { func: function3, args: args3 },
];

function executeFunctionWithArgs(operation, callback) {
  // executes function
  const { args, func } = operation;
  func(args, callback);
}

function serialProcedure(operation) {
  if (!operation) {
    process.exit(0); // finished
  }

  executeFunctionWithArgs(operation, function (result) {
    // continue AFTER callback
    serialProcedure(operations.shift());
  });
}

serialProcedure(operations.shift());
```

2. **連続実行制限:** 関数は厳密に順番に実行されますが、実行回数には制限があります。これは、大きなリストを処理する必要があるものの、正常に処理できる項目の数に上限を設けたい場合に便利です。

```js
let successCount = 0;

function final() {
  console.log(`dispatched ${successCount} emails`);
  console.log('finished');
}

function dispatch(recipient, callback) {
  // `sendMail` is a hypothetical SMTP client
  sendMail(
    {
      subject: 'Dinner tonight',
      message: 'We have lots of cabbage on the plate. You coming?',
      smtp: recipient.email,
    },
    callback
  );
}

function sendOneMillionEmailsOnly() {
  getListOfTenMillionGreatEmails(function (err, bigList) {
    if (err) {
      throw err;
    }

    function serial(recipient) {
      if (!recipient || successCount >= 1000000) {
        return final();
      }

      dispatch(recipient, function (_err) {
        if (!_err) {
          successCount += 1;
        }

        serial(bigList.pop());
      });
    }

    serial(bigList.pop());
  });
}

sendOneMillionEmailsOnly();
```

3. **完全な並列:** 1,000,000 人のメール受信者のリストにメールを送信する場合など、順序が問題にならない場合。

```js
let count = 0;
let success = 0;
const failed = [];
const recipients = [
  { name: 'Bart', email: 'bart@tld' },
  { name: 'Marge', email: 'marge@tld' },
  { name: 'Homer', email: 'homer@tld' },
  { name: 'Lisa', email: 'lisa@tld' },
  { name: 'Maggie', email: 'maggie@tld' },
];

function dispatch(recipient, callback) {
  // `sendMail` is a hypothetical SMTP client
  sendMail(
    {
      subject: 'Dinner tonight',
      message: 'We have lots of cabbage on the plate. You coming?',
      smtp: recipient.email,
    },
    callback
  );
}

function final(result) {
  console.log(`Result: ${result.count} attempts \
      & ${result.success} succeeded emails`);
  if (result.failed.length) {
    console.log(`Failed to send to: \
        \n${result.failed.join('\n')}\n`);
  }
}

recipients.forEach(function (recipient) {
  dispatch(recipient, function (err) {
    if (!err) {
      success += 1;
    } else {
      failed.push(recipient.name);
    }
    count += 1;

    if (count === recipients.length) {
      final({
        count,
        success,
        failed,
      });
    }
  });
});
```

それぞれに独自のユースケース、メリット、そして問題点があり、実際に試したり、詳細を読んだりすることができます。最も重要なのは、操作をモジュール化し、コールバックを使用することです。少しでも疑問を感じたら、すべてをミドルウェアとして扱ってください。
