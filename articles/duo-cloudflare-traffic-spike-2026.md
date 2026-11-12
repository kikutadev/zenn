---
title: "DUO瓦割りが84万回遊ばれて、15分で10万リクエスト超え — Cloudflareの使用量が急増した際の対策と振り返り"
emoji: "📈"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "d1", "個人開発", "browsergame"]
published: false
---

![DUO瓦割りとCloudflare Workersのリクエスト急増](/images/duo-cloudflare/duo-cloudflare-15min-thumbnail.webp)

ブラウザゲーム「DUO瓦割り」は、10月25〜26日の2日間で84万回遊ばれた。アクセスが集中した10月25日23時頃には、ゲームサイトのCloudflare Workersで**15分間に132,360回**の呼び出しが発生していた。当時のWorkers Freeの上限は1日10万回で、その回数を15分間で上回った。

原因を調べると、画像やJavaScriptを返すだけの通信までWorkerが処理していた。さらに、スコアを保存するD1では、画面に1つの数字を表示するために大量のプレイ履歴を読み取るSQLも問題になった。

この記事では、**Workersの呼び出し回数を減らす配信設定**と、**D1の読み取り行数を減らすSQLの変更**を、実際に使ったコードをもとに紹介する。

## DUO瓦割りの配信構成

DUO瓦割りは、タイミングよくタップして折りたたみスマホを瓦割りのように壊し、破壊した台数を競うブラウザゲーム。プレイ終了時のスコアをD1に保存し、世界累計やランキングをサイトに表示している。

https://games.kikuta.dev/duo-breaker/

ゲームサイトの構成は、フロントエンドの静的ファイルをCloudflare Workers Static Assetsで配信し、スコア保存やランキングなどのAPIにWorkersとD1を使うものだった。ただし当時は、その配信経路が適切に分かれていなかった。

## WorkersとD1で、別々の負荷が増えていた

10月25〜27日の3日間で、ゲームサイト全体のWorkers呼び出しは367万回。D1では30億行の読み取りと647万行の書き込みが記録された。いずれもDUO単独の数値ではない。

![10月25〜27日のWorkers呼び出しの推移](/images/duo-cloudflare/duo-cloudflare-workers-oct25-27-2026.png)

![10月25〜27日のD1読み取り・書き込み行数の推移](/images/duo-cloudflare/duo-cloudflare-d1-oct25-27-2026.png)

急増時にはWorkers Paidに切り替え、日割りで$1.95（税込）が請求された。

Cloudflareでは、次の2つを分けて考える必要がある。

- **Workersの呼び出し回数**：Workerがリクエストを処理した回数。2026年10月時点のFree枠は1日10万回（[公式の制限](https://developers.cloudflare.com/workers/platform/limits/)）。
- **D1の読み取り・書き込み行数**：SQLが読み書きした行の数。Free枠は1日500万行の読み取り、10万行の書き込み（[公式の料金体系](https://developers.cloudflare.com/d1/platform/pricing/)）。

例えば、1回のAPIリクエストでSQLが86万行を読めば、Workersの呼び出しは1回でもD1側では86万行の読み取りになる。したがって、配信経路とSQLはそれぞれ対処する必要があった。

## 対策1：画像やJavaScriptをWorkerに通さない

最初の問題は、`wrangler.jsonc` のStatic Assets設定だった。元の設定は次のとおり。

```jsonc
{
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "run_worker_first": true
  }
}
```

`run_worker_first: true` では、画像・JavaScript・CSSなどの静的ファイルへのアクセスも、まずWorkerを経由する。ゲームは起動時に複数の素材を読み込むため、ページを1回開くだけでもWorkersの呼び出しが増えていた。

そこで、**Workerで処理する必要のあるURLだけを指定する方式**に変更した。設定の主要部分は以下のとおり。

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

HTMLのルーティングやAPIはWorkerに通し、画像・JavaScript・CSSはStatic Assetsから直接返す。`/duo-breaker/*` のような広い指定を残すとゲーム素材まで再びWorkerを通してしまうため、**指定するパスの範囲**が重要だった。

世界累計やランキングのように複数人が同じ結果を見るGET APIにはキャッシュも使った。画面を開くたびに同じSQLを実行しないための対策だ。

## 対策2：累計を表示するたびに履歴を読み直さない

DUO瓦割りの世界累計は、最初から`duo_breaker_totals`という集計テーブルで管理していた。プレイ終了時に合計台数と完了回数を加算する仕組みだ。

問題は、その後に追加した**一定以上のスコアを通常の世界累計から除外する処理**だった。1,000台以上の記録を別区分にするため、画面の累計を取得するたびに、プレイ履歴から除外対象のスコアを合計して差し引いていた。

```sql
SELECT t.broken_phones - (
  SELECT COALESCE(SUM(r.destroyed), 0)
  FROM duo_breaker_runs r
  WHERE r.destroyed >= 1000
) AS brokenPhones
FROM duo_breaker_totals t
WHERE t.id = 1;
```

返す結果は世界累計の1行だけでも、その計算のために`duo_breaker_runs`を調べる。プレイ履歴が増えるほど、読み取り負荷が大きくなる可能性があった。

そこで、除外対象の合計も、プレイ終了時に更新するよう変更した。`duo_breaker_totals`に`bot_phones`を追加し、スコア確定時のトリガーで加算する。既存の除外対象分は移行時に一度だけ集計している。

```sql
CREATE TRIGGER duo_breaker_count_once
AFTER UPDATE OF destroyed ON duo_breaker_runs
WHEN OLD.destroyed IS NULL AND NEW.destroyed IS NOT NULL
BEGIN
  UPDATE duo_breaker_totals
  SET broken_phones = broken_phones + NEW.destroyed,
      completed_plays = completed_plays + 1,
      bot_phones = bot_phones +
        CASE WHEN NEW.destroyed >= 1000 THEN NEW.destroyed ELSE 0 END
  WHERE id = 1;
END;
```

この条件なら、完了前から完了済みへの更新時だけ加算され、終了リクエストの再送による二重計上も防げる。

累計を取得するSQLは次の形になった。

```sql
SELECT broken_phones - bot_phones AS brokenPhones,
       completed_plays AS plays
FROM duo_breaker_totals
WHERE id = 1;
```

こうすると、**表示のたびにプレイ履歴を合計する必要がない**。関連する累計取得処理では、1回あたりのD1読み取りが86万行から162行に減った。

ランキングにも同様の問題があった。全プレイから毎回自己ベストと順位を計算するのではなく、登録プレイヤーの自己ベストを別テーブルに保持し、順位表示ではそこから上位を取り出す方式にした。集計結果をあらかじめ持つことで、履歴の増加に引きずられにくくした。

## まとめ

アクセス増加で最初に見直すべきだったのは、**1回の画面表示で何度Workerが呼ばれるか**と、**1回のAPI処理でD1が何行読むか**の2点だった。

画像やJavaScriptはWorkerに通さず、共有できるAPIの結果はキャッシュする。合計値やランキングを表示するために履歴を毎回計算しているなら、更新時に集計値を保持できないか検討する。どちらも、アクセス数やプレイ履歴が増える前に確認しておきたいポイントだ。
