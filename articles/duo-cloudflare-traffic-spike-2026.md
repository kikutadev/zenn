---
title: "DUO瓦割りが84万回遊ばれて、15分で10万リクエスト超え — Cloudflareの使用量が急増した際の対策と振り返り"
emoji: "📈"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "d1", "個人開発", "browsergame"]
published: false
---

![DUO瓦割りとCloudflare Workersのリクエスト急増](/images/duo-cloudflare/duo-cloudflare-15min-thumbnail.webp)

2026年10月25〜26日の2日間で、ブラウザゲーム「DUO瓦割り」が**84万回**遊ばれました。折りたたみスマホを瓦のように割って、壊した台数を競うゲームです。

https://games.kikuta.dev/duo-breaker/

アクセスが集中した25日の夜、ゲームサイト全体のCloudflare Workersは**15分間で132,360回**呼び出されました。当時のFreeプランの上限は1日10万回。15分間だけで、その回数を超えていました。

ただ、アクセスが多かったことだけが原因ではありません。調べてみると、画像やJavaScriptを返すだけでもWorkerが動く設定になっていて、D1にも大量の履歴を読み取るSQLがありました。

Workersの配信設定とD1のSQL、それぞれどこを変更したのか紹介します。

## Workersの呼び出し回数とD1の読み取り行数

DUO瓦割りでは、プレイが終わるとWorkersのAPIへスコアを送り、D1に保存しています。世界累計やランキングもAPIから取得しています。

10月25〜27日の3日間では、ゲームサイト全体でWorkersが**367万回**呼び出され、D1は**30億行**を読み取っていました。どちらもDUOだけの数字ではありません。

![10月25〜27日のWorkers呼び出しの推移](/images/duo-cloudflare/duo-cloudflare-workers-oct25-27-2026.png)

![10月25〜27日のD1読み取り・書き込み行数の推移](/images/duo-cloudflare/duo-cloudflare-d1-oct25-27-2026.png)

D1の無料枠は、1日500万行の読み取りと10万行の書き込みでした。Workersの呼び出し回数とは別に計算されます。

Workerを1回しか呼んでいなくても、その中で実行するSQLが大量の行を読めば、D1の使用量は増えます。**Workerを呼び出す回数と、D1が読み取る行数は別**なんですよね。

アクセス集中時にはWorkers Paidへ切り替えました。切り替え時の日割り請求は$1.95（税込）でした。ただ、プランを変えても不要なWorkerの実行は減らないので、配信設定も見直しました。

## 画像を返すだけなのにWorkerが動いていた

原因の一つは、`wrangler.jsonc`の`run_worker_first`です。

もともとは次の設定でした。

```jsonc
{
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "run_worker_first": true
  }
}
```

この`true`があると、HTMLだけでなく画像やJavaScript、CSSも先にWorkerを通ります。

ゲームを開くと、ブラウザは画像やスクリプトをまとめて読み込みます。ファイルをそのまま返すだけなのに、そのたびにWorkerを動かしていたわけです。

そこで、Workerで処理するURLを限定しました。設定の主要部分はこちらです。

```jsonc
{
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "run_worker_first": [
      "/",
      "/*/",
      "/*.html",
      "/api/*"
    ]
  }
}
```

HTMLの処理やAPIはWorkerに通し、ゲームの画像・JavaScript・CSSはStatic Assetsから直接配信する形です。

ただ、変更の途中では`/duo-breaker/*`という指定が残っていました。これだとゲーム内の画像やJavaScriptまで対象になります。**Workerを使うパスは、指定範囲まで見直さないと意味がありません。**

最終的には、この広い指定を外して静的ファイルがWorkerを通らないようにしました。

## 世界累計の表示で86万行も読んでいた

Workerの呼び出し回数とは別に、D1のSQLにも問題がありました。

DUO瓦割りには、プレイが終わるたびに破壊台数を加算する`duo_breaker_totals`という集計テーブルがあります。これだけなら、世界累計を表示するために全プレイを数え直す必要はありません。

ところが後から、1,000台以上の記録を通常の世界累計から除外する処理を追加した際、除外対象の台数を**表示のたびにプレイ履歴から集計するSQL**にしていました。

```sql
SELECT t.broken_phones - (
  SELECT COALESCE(SUM(r.destroyed), 0)
  FROM duo_breaker_runs r
  WHERE r.destroyed >= 1000
) AS brokenPhones
FROM duo_breaker_totals t
WHERE t.id = 1;
```

返すのは世界累計の数字1つですが、その計算のために`duo_breaker_runs`を読み直します。この処理では、**1回の取得で86万行**を読んでいました。

そこで、除外する台数も集計テーブルに保持するよう変更しました。`duo_breaker_totals`に`bot_phones`列を追加し、プレイ結果が確定したときに加算します。過去の記録については、移行時に一度だけ集計しました。

変更後は、世界累計を次のSQLで取得できます。

```sql
SELECT broken_phones - bot_phones AS brokenPhones,
       completed_plays AS plays
FROM duo_breaker_totals
WHERE id = 1;
```

これなら、表示するたびに履歴を読み直さずに済みます。**86万行読んでいた処理が、162行で済むようになりました。**

ランキングも、全プレイ履歴から毎回自己ベストを計算する方式をやめ、名前登録済みプレイヤーのベスト記録を別テーブルに保存する形に変えました。

## 世界累計とランキングにはキャッシュも使う

SQLを軽くしても、同じ世界累計を表示するたびにD1へ問い合わせる必要はありません。

世界累計のGET APIは120秒、ランキングは600秒キャッシュするようにしました。多くの人が同じ結果を見る間は、D1に同じ問い合わせを繰り返さずに済みます。

ただし、プレイ直後に本人の記録を表示するときは、キャッシュされた古い結果を返さないようにしています。

## まとめ

今回のようにアクセスが急増したときは、まず**どのリクエストでWorkerが動いているか**を見るのが大事でした。`run_worker_first`が広すぎると、静的ファイルを返すだけでも実行回数が増えてしまいます。

D1では、SQLが返す結果の件数よりも、**実際に何行読んでいるか**が重要です。合計値の表示で履歴を毎回集計している場合は、更新時に集計値を保存する方法も検討できます。

Cloudflare公式の[Static Assetsと`run_worker_first`の設定](https://developers.cloudflare.com/workers/static-assets/binding/)と[D1の料金・使用量](https://developers.cloudflare.com/d1/platform/pricing/)も参考になります。
