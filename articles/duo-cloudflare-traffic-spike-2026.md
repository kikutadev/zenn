---
title: "DUO瓦割りが84万回遊ばれて、15分で10万リクエスト超え — Cloudflareの使用量が急増した際の対策と振り返り"
emoji: "📈"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "d1", "個人開発", "browsergame"]
published: false
---

![DUO瓦割りとCloudflare Workersのリクエスト急増](/images/duo-cloudflare/duo-cloudflare-15min-thumbnail.webp)

2026年10月25日の夜、Cloudflare Workersの呼び出しが**15分間で132,360回**になりました。当時のWorkers Freeの上限は1日10万回。15分だけで、その回数を超えていました。

同じ頃、ブラウザゲーム「DUO瓦割り」へのアクセスが急増していました。10月25〜26日の完了プレイは**84万回**。折りたたみスマホを瓦のように割って、壊した台数を競うゲームです。

https://games.kikuta.dev/duo-breaker/

アクセスが増えればリクエストも増えますが、調べてみると、それだけではありませんでした。**画像やJavaScriptを返すだけなのにWorkerが動いていた**うえに、**世界累計を表示するだけでD1が86万行も読んでいた**のです。

そこで、Cloudflareの配信設定とD1のSQLを変更しました。どこで余計な処理が発生していたのか、順に振り返ります。

## Workersが367万回、D1は30億行を読んでいた

DUO瓦割りでは、Cloudflare WorkersのAPIでプレイ記録を受け取り、D1に保存しています。ランキングや世界累計もD1から取得する構成です。

10月25〜27日の3日間で、ゲームサイト全体のWorkers呼び出しは**367万回**。D1の読み取りは**30億行**、書き込みは**647万行**でした。DUOだけの数値ではありませんが、サイト全体にかなりの負荷がかかっていました。

![10月25〜27日のWorkers呼び出しの推移](/images/duo-cloudflare/duo-cloudflare-workers-oct25-27-2026.png)

![10月25〜27日のD1読み取り・書き込み行数の推移](/images/duo-cloudflare/duo-cloudflare-d1-oct25-27-2026.png)

当時の無料枠は、Workersが**1日10万回**、D1が**1日500万行の読み取り・10万行の書き込み**です。上限はそれぞれ別に計算されます。

たとえば、APIへのアクセスが1回でも、その中のSQLが86万行を読めば、D1には86万行分の読み取りが計上されます。**Workerを呼ぶ回数を減らすことと、D1が読む行数を減らすことは別の対策**なんですよね。

アクセス集中時にはWorkers Paidへ切り替えました。切り替え時の日割り請求は**$1.95（税込）**でしたが、プランを変えて終わりにはできません。まず配信設定から見直しました。

## 画像を返すだけなのにWorkerが動いていた

見直したのは、`wrangler.jsonc`の`run_worker_first`です。

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

`run_worker_first: true`だと、画像やJavaScript、CSSなどの静的ファイルも、まずWorkerを通ります。

ゲームは画面を開くとスクリプトや画像をまとめて読み込みます。ファイルをそのまま返せばよいだけなのに、その一つひとつでWorkerを実行していたわけです。

そこで、Workerを動かす必要があるURLを指定する方式に変更しました。設定の主要部分はこちらです。

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

HTMLの処理やAPIはWorkerに任せ、それ以外の画像やJavaScriptはStatic Assetsから直接返します。

実は最初の変更では、`/duo-breaker/*`という指定が残っていました。これではゲームのJSや画像もWorkerを通ってしまいます。**パスを指定する場合は、どこまで含まれるのか**も見ないといけません。

設定を修正して、ゲーム素材をWorkerを経由せず配信できるようにしました。

## 世界累計の表示で、86万行も読み取っていた

次はD1です。

DUO瓦割りでは、プレイが終わるたびに破壊台数を`duo_breaker_totals`へ加算しています。世界累計を保存するテーブルは、最初からありました。

ところが後から、「1,000台以上の記録は通常の世界累計から除外する」という処理を追加したときに、除外する台数だけを**毎回プレイ履歴から計算するSQL**にしていました。

```sql
SELECT t.broken_phones - (
  SELECT COALESCE(SUM(r.destroyed), 0)
  FROM duo_breaker_runs r
  WHERE r.destroyed >= 1000
) AS brokenPhones
FROM duo_breaker_totals t
WHERE t.id = 1;
```

結果として返すのは世界累計の数字1つですが、そのために`duo_breaker_runs`を読み直しています。プレイ履歴が増えるほど、読む行数も増えてしまいます。

この処理では、**1回の取得で86万行**を読んでいました。

ここを、除外する台数もあらかじめ合計しておく方法に変更しました。`duo_breaker_totals`に`bot_phones`列を追加し、プレイ結果が確定するときにトリガーで加算する形です。すでに登録されている分は移行時に一度だけ集計しています。

すると、世界累計の取得はこれだけで済みます。

```sql
SELECT broken_phones - bot_phones AS brokenPhones,
       completed_plays AS plays
FROM duo_breaker_totals
WHERE id = 1;
```

もう過去のプレイ履歴を合計し直す必要はありません。**86万行読んでいた処理が、162行で済むようになりました。**

ランキングも似た構造でした。名前を登録したプレイヤーの自己ベストを別テーブルに持ち、全プレイ履歴から毎回ベストを探さなくても順位を出せるようにしています。

## 同じ数字を毎回取りに行かない

もう一つ、世界累計やランキングのGET APIにもキャッシュを入れました。

世界累計は120秒、ランキングは600秒。同じ結果を多くの人が見ている間は、毎回D1を読まなくて済みます。一方、プレイ直後に本人の記録を表示するときなど、すぐに新しい情報が必要な部分は別に扱います。

**静的ファイルはWorkerを通さない、共有できる集計結果はキャッシュする、累計は履歴を毎回集計しない。** それぞれ別のところで無駄を減らす形です。

## まとめ

今回、気を付けるべきだったのは次の2点でした。

- 画像やJavaScriptを返すだけなのに、Workerが実行されていないか。
- SQLは1行しか返さなくても、その裏で大量の行を読んでいないか。

ブラウザゲームは、公開後に急に遊ばれることがあります。Workersの呼び出し回数だけでなく、D1の読み取り行数も見ておくと、アクセスが増えたときに原因を追いやすくなります。

Cloudflare公式の[Static Assetsと`run_worker_first`の設定](https://developers.cloudflare.com/workers/static-assets/binding/)と[D1の料金・使用量](https://developers.cloudflare.com/d1/platform/pricing/)も参考になります。
