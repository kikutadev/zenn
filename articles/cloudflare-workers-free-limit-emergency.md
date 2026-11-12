---
title: "Cloudflare Workersの無料枠に達したときの緊急回避策をまとめたよ"
emoji: "🚨"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "githubpages", "unity", "browsergame"]
published: false
---

ブラウザゲーム「[GAPLINE](https://games.kikuta.dev/gapline/)」を公開した翌日、ゲームサイトのCloudflare Workersが無料枠の上限に達してしまい、**Error 1027**が発生しました。

Workerが動かなくなったときに使った退避策と、翌日に配信を復旧させてから見直したStatic Assetsの設定をまとめます。

もし今まさにError 1027が出て困っているなら、やるべき流れは次のとおりです。

1. **静的リダイレクトで別ホスティング（GitHub Pagesなど）へ一時退避する**（Workerはもう呼べないので、Workerを通さない静的な転送を使います）
2. **枠がリセットされたら配信を戻す**（日本時間午前9時にリセットされます）
3. **Static Assetsの設定を見直して、静的ファイルでWorkerを消費しない構成に変える**

順を追って、何が起きてどう直したのかを振り返っていきます。

## Workersの無料枠に達して何が起きたか

Cloudflare Workersは、リクエストに応じてサーバー側の処理を実行できるサービスです。ブラウザゲームだとランキングやアクセス解析などのAPIに使えます。一方で、ゲームのHTMLや画像、JavaScript、WASMといったファイルは、コードを動かさずにそのまま返せば足ります。

Cloudflare WorkersのFreeプランでは、**Workerの呼び出しはアカウント全体で1日10万回まで**という制限があります。作品ごとではなくアカウント合算です。上限はUTC午前0時、日本時間だと午前9時にリセットされます。この上限を超えると、Workerのルート設定によってはError 1027が表示されてアクセスできなくなります。

ところが、Workers Static Assetsからファイルを直接返すリクエストは無料で、回数制限もありません。静的ファイルの配信までWorkerを優先実行する設定になっていると、ファイルの取得ごとにWorkerの呼び出し回数がカウントされてしまいます。

11月5日の記録では、UTC午前0時を過ぎてからゲームサイトのWorkerが**95,035回**呼び出されていました。そして、ほどなく1日10万回の上限に達してError 1027が発生しました。

後からCloudflareの管理画面で11月5〜6日の2日間を確認したところ、ゲームサイト用Workerの呼び出しは28.8万回、アセットリクエストは27.5万回、エラーは4,640件でした。2日間の合算なので1日10万回の制限と直接比べる数字ではありませんが、かなりの回数が動いていました。ただし、これらは共通Worker全体の数値で、GAPLINE単体のアクセス数ではありません。エラー総数のすべてがError 1027だったわけでもありません。

![Cloudflare Workersのメトリクス画面。11月5〜6日の呼び出し288.19k、アセットリクエスト275.42kと時間別グラフ](/images/cloudflare-workers-free-limit-emergency/workers-metrics.png)

*Cloudflare管理画面で後日確認したメトリクス。対象期間は11月5日0時〜7日0時（日本時間）。「呼び出し」「アセットリクエスト」は別の指標で、どちらもGAPLINE専用の値ではない。*

## なぜ静的ファイルの配信でWorkerを消費していたのか

原因を調べると、配信設定の噛み合わせに理由がありました。

もともとGAPLINEを含む複数のブラウザゲームを、ひとつの `kikuta-games` Workerで配信していました。そのサイトで、ゲームのHTMLや一部の実行ファイルを返す前にもWorkerを動かす設定が残っていたんです。

Cloudflare WorkersのStatic Assetsには `assets.run_worker_first` という設定項目があります。ここに対象のパスを指定すると、静的ファイルがあっても先にWorkerを呼び出します。HTMLの書き換えや認証を挟むには便利な機能ですが、ただ静的ファイルを返すだけの通信までWorkerを起動してしまうと、あっという間に無料枠を使い切ってしまいます。

Unity WebGLのゲームは、起動時にHTMLだけでなくJavaScript、WASM、ゲームデータなど複数のファイルを一気に読み込みます。静的ファイルにWorkerを通す設定があると、1回遊ばれただけで何回もWorkerが呼ばれてしまうわけです。

ただ、同じ時期にはアクセス解析の実装を変更していて、イベント受信に伴うD1への書き込みも増えていました。そのため、Workersの上限超過がGAPLINE単独の人気によるものだったとは断定できません。いずれにしても、複数作品で共有しているWorkerをどのURLで起動させていたかが、一番の見直しポイントでした。

## 【緊急対処】GitHub Pagesへ静的リダイレクトで一時退避する

Error 1027が出ているときは、すでにWorkerの呼び出し上限を超えている状態です。そのため、Workerのコード内にリダイレクト処理を書いてデプロイしようとしても、そもそもWorker自体が実行されないので転送できません。

そこで、Workerを介さずに動く静的なリダイレクト設定を使って、GAPLINEだけをGitHub Pagesへ一時退避させることにしました。

公開サイトの `_redirects` に次のルールを追加します。

```text
/gapline https://kikutadev.github.io/gapline-pages/ 302
/gapline/ https://kikutadev.github.io/gapline-pages/ 302
/gapline/* https://kikutadev.github.io/gapline-pages/:splat 302
```

あわせて、`wrangler.jsonc` の `assets.run_worker_first` に `!/gapline/*` を指定し、GAPLINEへのアクセスでWorkerを先に実行しないように除外しました。

HTTP 302は一時的な転送を表すステータスコードです。ゲームサイトの `/gapline/` に来た読者を、退避先として用意したGitHub PagesのGAPLINEへそのまま誘導できます。正式なURLを変えずに、その場をしのぐ退避策としてうまく動いてくれました。

## 【復旧と恒久対策】Cloudflareへ戻して配信設定を根本から見直す

日本時間午前9時（UTC午前0時）を迎えて無料枠のリセットを確認したあと、GAPLINEの配信をCloudflareへ戻しました。ただ戻すだけでは再発してしまうので、ゲームサイト全体の配信構成を根本から整理しました。

今までリクエストのたびにWorkerでHTMLへアクセス解析用JavaScriptを注入していたんですが、ビルド時にHTMLへ組み込む方式へ変更しました。HTMLを表示するだけならWorkerを通す必要はなく、静的ファイルとして返すだけで良かったんですよね。

そのうえで、ファイルの役割ごとに配信経路をきれいに分けました。HTML、画像、JavaScript、WASMなどは、すべてWorkers Static Assetsから直接返すようにし、Workerを優先実行する経路は、アクセス解析やゲームのスコア保存などサーバー処理が必要なAPIだけに絞り込みました。

```text
ゲームのHTML・画像・JavaScript・WASM
  → Workers Static Assetsから直接配信

アクセス解析・ランキングなどのAPI
  → Workerを起動して処理
```

また、GAPLINEのWebGLファイル配信にも少し調整を入れました。Unityの圧縮済み `.unityweb` を正しくブラウザに読み込ませるため、`Content-Encoding: gzip` と形式に合わせた `Content-Type` を返す設定にしています。もともとR2経由で配信していましたが、圧縮ファイルをStatic Assetsへ移しました。

Workers Static Assetsには**1ファイル25MiB**の上限があります。当時のGAPLINEは圧縮済みデータが11.5MB、WASMが8.0MBだったので、この上限内にしっかり収まりました。非圧縮に戻すことなく、gzip圧縮を保ったまま配信できています。

## 結果と学んだこと

静的ファイルの配信とAPIの実行経路を分けたことで、ゲームを起動してファイルをダウンロードする通信がWorkerの呼び出し回数を浪費することはなくなりました。

今回のトラブルで特に役立ったのは、上限超過で動かないWorkerに頼らず、静的リダイレクトを使ってトラフィックを別ホスティングへ逃がせたことです。

障害当時のアクセスログを作品別・URL別にすべて追えていたわけではないため、GAPLINE単体が実際に何回の呼び出しを起こしていたのかまでは分かりません。ですが、Worker全体のメトリクスに加えて、「どのURLでWorkerが起動する設定になっているか」も確認しておく必要があります。

### 参考リンク

- [Workers Freeの1日10万回制限とError 1027](https://developers.cloudflare.com/workers/platform/limits/)
- [Static Assetsの料金と制限](https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/)
- [run_worker_firstの設定方法](https://developers.cloudflare.com/workers/static-assets/binding/)
- [リダイレクトの設定](https://developers.cloudflare.com/pages/configuration/redirects/)
