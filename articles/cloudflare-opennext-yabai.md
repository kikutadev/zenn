---
title: "Cloudflare OpenNext Nextjsスタックの爆速デプロイがヤバい件。"
emoji: "🚀"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Cloudflare", "OpenNext", "nextjs", "workers"]
published: false
---
# OpenNext用のCloudflareアダプターを使用して、Next.jsアプリをCloudflare Workersにデプロイしてみた

いや、めっちゃ早いし、めっちゃ楽だった。

これを実行するだけで...
```bash
npm create cloudflare@latest -- PROJECTNAME --framework=next --platform=workers
```

プロジェクトができた。
https://web.kikutadev.workers.dev/
