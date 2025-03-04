---
title: "【Short Tips】Slidevで作ったスライドをNext.jsで表示させる"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [slidev,nextjs]
published: false
---

# Slidevで作ったスライドをNext.jsで表示させるには

イメージはこんな感じで埋め込み表示させる。

https://kikuta.dev/slide

## Step 1: Slidevで作成したスライドをビルドする

```bash
npx slidev build --base /slide
```

:::message
`--base` オプションでビルドしたスライドのパスを指定する。
ここで`--base`を指定しないと、スライドのパスが`/`になってしまい、後述のiframeからのルーティングがうまくいかない。
:::

## Step 2: Next.jsのpublicディレクトリにビルドしたスライドを配置する

ビルドしたスライドをNext.jsのpublicディレクトリに配置する。
`dist`というディレクトリにビルドされるので、`public`ディレクトリに`slide`という名前で配置する。

## Step 3: Next.jsでスライドを表示するページを作成する

かなり適当だが、iframeで表示させるだけのページを作成する。
tailwindcssでスタイルを当てている。

```tsx
import React from "react";

import { getMetadata } from "@/lib/getMetadata";

export default function SlidePage() {
  return <iframe src="slide/index.html" className="m-2 size-full grow" />;
}

export const metadata = getMetadata({
  type: "website",
  pagePath: "/slide",
  title: "slide",
  description: "kikuta.devのスライドページです。",
});
```

## 完成

これで、Slidevで作成したスライドをNext.jsで表示させることができる。
![alt text](/images/download.jpg)
