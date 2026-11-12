---
title: "DUO瓦割りが84万回遊ばれて、15分で10万リクエスト超え — Cloudflareの使用量が急増した際の対策と振り返り"
emoji: "📈"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "d1", "個人開発", "browsergame"]
published: false
---

![DUO瓦割りとCloudflare Workersのリクエスト急増](/images/duo-cloudflare/duo-cloudflare-15min-thumbnail.webp)

2026年10月25日の夜、ブラウザゲーム「DUO瓦割り」を公開していたゲームサイトで、**Cloudflare Workersが15分間に132,360回**呼び出された。Workers Freeの上限は当時1日10万回。15分だけでその回数を超えていた。

DUO瓦割りは、タイミングよくタップして折りたたみスマホを瓦のように割り、破壊台数を競うブラウザゲームだ。10月25〜26日の完了プレイ数は**84万回**に達した。

https://games.kikuta.dev/duo-breaker/

アクセスが増えて分かった問題は、大きく2つあった。

- **Workerの呼び出し回数が多い**：画像やJavaScriptなど、本来そのまま配信できるファイルまでWorkerを通っていた。
- **D1で読み取る行数が多い**：世界累計を1回表示するだけで、プレイ履歴から大量の行を読み取っていた。

どちらもCloudflareの使用量が増える原因だが、直す場所は別だった。今回は、実際に変更した設定とSQLを紹介する。

## WorkersとD1の使用量が急増した

ゲームでは、スコアやランキング、世界累計をCloudflare WorkersのAPI経由でD1に保存・取得していた。サイトのHTMLやゲームの画像・JavaScriptはWorkers Static Assetsから配信していた。

10月25〜27日のゲームサイト全体のWorkers呼び出しは**367万回**。D1では**30億行の読み取り、647万行の書き込み**が記録された。

![10月25〜27日のWorkers呼び出しの推移](/images/duo-cloudflare/duo-cloudflare-workers-oct25-27-2026.png)

![10月25〜27日のD1読み取り・書き込み行数の推移](/images/duo-cloudflare/duo-cloudflare-d1-oct25-27-2026.png)

当時の無料枠は、Workersが**1日10万回の呼び出し**、D1が**1日500万行の読み取り・10万行の書き込み**だった。いずれもゲーム単位ではなく、アカウント全体での制限だ。

ここで重要なのは、**Workersの呼び出し回数とD1の読み取り行数は別のもの**だということ。Workerを1回呼ぶだけでも、その中で実行したSQLが大量の履歴を読むなら、D1の使用量は大きくなる。

アクセス集中時にはWorkers Paidにも切り替えた。日割りの請求は$1.95（税込）だった。ただし、プランの変更だけで処理の無駄がなくなるわけではない。配信経路とSQLをそれぞれ見直した。

## 画像やJavaScriptまでWorkerを通していた

最初に見直したのは`wrangler.jsonc`の`run_worker_first`だった。

当初は次のように設定していた。

```jsonc
{
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "run_worker_first": true
  }
}
```

これだと、HTMLだけでなく画像・JavaScript・CSSを返すリクエストでも、先にWorkerが実行される。ゲームはページを開くと複数の画像やスクリプトを読み込むため、**1回遊ぶための通信が何回ものWorker呼び出しになっていた**。

そこで、Workerが必要なURLを指定する方式に変更した。設定の主要部分は次のとおり。

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

HTMLの書き換えやAPIなどはWorkerに通し、画像・JavaScript・CSSはStatic Assetsから直接返す。

ただし、最初の変更では`/duo-breaker/*`のような広いパス指定が残っており、ゲーム素材もWorkerを経由していた。そこで指定範囲を見直し、実際に静的ファイルが直接配信されるようにした。**`run_worker_first`は有効・無効だけでなく、どのURLに適用するかが重要**だった。

## 世界累計の表示で86万行を読んでいた

もう一つはD1側の問題だ。

DUO瓦割りでは、プレイ終了時に破壊台数を`duo_breaker_totals`へ加算していた。つまり、**世界累計を保持するテーブルは最初から存在していた**。

ところが、後から「1,000台以上壊したスコアは通常の世界累計に含めない」という区分を追加した際、除外対象の台数を**表示のたびにプレイ履歴から合計するSQL**にしてしまった。

```sql
SELECT t.broken_phones - (
  SELECT COALESCE(SUM(r.destroyed), 0)
  FROM duo_breaker_runs r
  WHERE r.destroyed >= 1000
) AS brokenPhones
FROM duo_breaker_totals t
WHERE t.id = 1;
```

このSQLが返すのは累計の数字1つ。それでも、`SUM`の計算では`duo_breaker_runs`を読む。アクセスが集中したうえにプレイ履歴が増え、**1回の取得で86万行**を読む状態になっていた。

解決方法は、除外する台数もあらかじめ合計しておくことだった。`duo_breaker_totals`に`bot_phones`列を追加し、プレイ結果が確定したときにトリガーで更新する。過去の除外対象分は移行時に一度集計した。

変更後、世界累計を取得するSQLは次のようになった。

```sql
SELECT broken_phones - bot_phones AS brokenPhones,
       completed_plays AS plays
FROM duo_breaker_totals
WHERE id = 1;
```

これなら、ページを開くたびにプレイ履歴を合計し直す必要がない。対象の読み取り行数は、**86万行から162行**に減った。

ランキングも同様に、全プレイから毎回自己ベストを計算するのをやめ、名前登録済みプレイヤーの自己ベストを専用テーブルで持つ形に変更した。

## 繰り返し呼ばれるAPIはキャッシュする

世界累計やランキングは、多くの人が同じ内容を見る。毎回D1に問い合わせる必要はない。

そこで、世界累計とランキングのGET APIには短時間のキャッシュを設定した。世界累計は120秒、ランキングは600秒。プレイ直後の本人の記録など、すぐに最新値が必要な操作は別に扱った。

これで、画面を開くたびに同じ問い合わせが繰り返されるのを抑えた。Workersの配信経路を直すことと、**Workerが呼ばれた後にD1へ何度アクセスするかを減らすこと**は、それぞれ別の対策になる。

## 振り返り

今回の経験で見直すことになったのは、**「ファイルを返すためにWorkerを起動していないか」と「1回のSQLで何行読んでいるか」**だった。

Cloudflareでは、画面の表示が正常でも裏側で大量の処理が走ることがある。とくにゲームのようにアクセスが一度に増えるサービスでは、プレイ回数だけでなく、Workersの呼び出しとD1の行数を別々に見る必要がある。

設定を確認するなら[Workers Static Assetsの`run_worker_first`](https://developers.cloudflare.com/workers/static-assets/binding/)、DBの使用量を調べるなら[D1の料金と行数の考え方](https://developers.cloudflare.com/d1/platform/pricing/)が参考になる。
