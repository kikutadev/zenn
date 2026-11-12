---
title: "Cloudflare Workersの無料枠に達したときの緊急回避策をまとめたよ"
emoji: "🚨"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "githubpages", "unity", "browsergame"]
published: false
---

ブラウザゲーム「[GAPLINE](https://games.kikuta.dev/gapline/)」を公開した翌日、ゲームサイトのCloudflare Workersが無料枠に達し、**Error 1027**が発生した。GAPLINEをGitHub Pagesへ一時退避し、翌日にはCloudflareでの配信を再開した。

このときの経験を振り返りながら、Workersの無料枠に達したときの緊急回避策と、同じ事態を避けるために見直した配信設定をTipsとしてまとめる。

## Workersの無料枠に達した

Cloudflare Workersは、リクエストを受けたときにサーバー側のコードを実行できるサービスだ。ブラウザゲームではランキングやアクセス解析などのAPIに使える。一方、ゲームのHTMLや画像、JavaScript、WASMは、コードを実行せずにファイルをそのまま返せる。

Cloudflare WorkersのFreeプランでは、**Workerの呼び出しはアカウント全体で1日10万回まで**。作品ごとに10万回ではない。上限はUTCの午前0時、日本時間では午前9時にリセットされる。上限を超えると、Workerのルート設定によってはError 1027が表示される。

一方、Workers Static Assetsからファイルを直接配信するリクエストは無料で、回数制限もない。静的ファイルの配信にもWorkerを優先実行するよう設定していると、その分はWorkerの呼び出し回数に入る。無料枠を考えるときは、ゲームが何回遊ばれたかだけでなく、何回Workerを実行したかを確認する必要がある。

11月5日の運用記録では、UTC午前0時以降にゲームサイトのWorkerが**95,035回**呼び出されていた。ほどなく1日10万回の上限に達し、Error 1027が発生した。

Cloudflareの管理画面で11月5〜6日の2日間を確認すると、ゲームサイト用Workerの呼び出しは約28.8万回、アセットリクエストは約27.5万回、エラーは約4,640件だった。2日間の合計なので、1日10万回という制限とそのまま比較する数字ではない。これらは共通Workerの数値であり、GAPLINE単体のアクセス数ではない。また、エラー総数がすべてError 1027だったわけではない。

![Cloudflare Workersのメトリクス画面。11月5〜6日の呼び出し288.19k、アセットリクエスト275.42kと時間別グラフ](/images/cloudflare-workers-free-limit-emergency/workers-metrics.png)

*Cloudflare管理画面で後日確認したメトリクス。対象期間は11月5日0時〜7日0時（日本時間）。「呼び出し」「アセットリクエスト」は別の指標で、どちらもGAPLINE専用の値ではない。*

## なぜ静的ファイルの配信でWorkerを使っていたのか

GAPLINEを含む複数のブラウザゲームを、ひとつの `kikuta-games` Workerで公開していた。サイトでは、ゲームのHTMLや一部の実行ファイルを返す前にもWorkerを動かす設定が残っていた。

Cloudflare WorkersのStatic Assetsには `assets.run_worker_first` という設定がある。指定したURLでは、静的ファイルが存在しても先にWorkerを呼ぶ。この機能はHTMLを書き換えたり認証処理を入れたりするには便利だが、静的に返すだけのファイルまでWorkerを実行すれば、無料枠を消費する。

Unity WebGLのゲームは、プレイ画面を開く際にHTMLだけでなく、JavaScript、WASM、ゲームデータなど複数のファイルを取得する。静的ファイルにWorkerを通す設定があると、プレイ回数とWorkerの呼び出し回数は一致しない。

同じ時期にはアクセス解析の実装変更もあり、イベント受信に伴うD1の書き込みが増えていた。したがって、Workersの上限超過をGAPLINE単独の人気によるものとは断定できない。まず見直すべきだったのは、複数作品で共有するWorkerをどの経路で起動していたかだった。

## GitHub Pagesへ一時退避した

上限に達したWorkerのコードでリダイレクトを返そうとしても、Worker自体が正常に呼び出せなければ転送できない。そこで、GAPLINEだけを静的なリダイレクト設定でGitHub Pagesへ移した。

公開サイトの `_redirects` には次のルールを追加した。

```text
/gapline https://kikutadev.github.io/gapline-pages/ 302
/gapline/ https://kikutadev.github.io/gapline-pages/ 302
/gapline/* https://kikutadev.github.io/gapline-pages/:splat 302
```

さらに `wrangler.jsonc` の `assets.run_worker_first` に `!/gapline/*` を追加し、GAPLINEへのリクエストでWorkerを先に実行しないようにした。

HTTP 302は一時的な転送を示す。ゲームサイトの `/gapline/` から入った利用者を、GitHub Pagesで配信していたGAPLINEへ誘導する構成だ。正式な公開先を恒久的に変えたわけではない。

## Cloudflareへ戻し、配信方法を変更した

無料枠のリセット後、GAPLINEをCloudflareで直接配信する構成に戻した。そこで終わらせず、ゲームサイト全体の配信方法も整理した。

まず、アクセス解析用のJavaScriptをHTMLへ追加する処理は、リクエストのたびにWorkerで書き換える方式から、ビルド時に組み込む方式へ変更した。HTMLを表示するだけならWorkerでの処理は不要になる。

次に、HTML、画像、JavaScript、WASMなどは、原則としてWorkers Static Assetsから直接返すようにした。Workerを優先実行する経路は、アクセス解析やゲームの記録など、サーバー側の処理が必要なAPIへ絞った。

```text
ゲームのHTML・画像・JavaScript・WASM
  → Workers Static Assetsから直接配信

アクセス解析・ランキングなどのAPI
  → Workerを起動して処理
```

GAPLINEのWebGLファイルにも配信上の調整が必要だった。Unityの圧縮済み `.unityweb` を正しく読み込むため、`Content-Encoding: gzip` とファイル形式に合った `Content-Type` を返す設定にした。もともとはR2経由で配信していたが、圧縮済みファイルをStatic Assetsへ移した。

Cloudflare WorkersのStatic Assetsには**1ファイル25MiB**の上限がある。当時のGAPLINEの圧縮済みデータは約11.5MB、WASMは約8.0MBで、この上限内に収まっていた。非圧縮ファイルへ戻すのではなく、gzipを維持して配信した。

## 障害後に確認するようになったこと

静的ファイルの配信とAPIの実行を分けると、ゲーム本体のダウンロードがそのままWorkerの呼び出しを増やす構成を避けられる。

今回の退避で有効だったのは、転送処理を上限に達したWorkerに依存させなかったことだ。一方、障害当時のリクエストを作品別・経路別にすべて分類できていたわけではなく、GAPLINEが何回のWorker呼び出しを発生させたかは分からない。次に同じ状況を調べるときは、Worker全体のメトリクスに加えて、どのURLをWorkerが処理しているかも確認する必要がある。

### Cloudflareの公式資料

- [Workers Freeの1日10万回制限とError 1027](https://developers.cloudflare.com/workers/platform/limits/)
- [Static Assetsの料金と制限](https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/)
- [run_worker_firstの設定方法](https://developers.cloudflare.com/workers/static-assets/binding/)
- [リダイレクトの設定](https://developers.cloudflare.com/pages/configuration/redirects/)
