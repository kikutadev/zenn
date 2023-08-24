---
title: "NotionのテーブルにNotion APIを使ってリレーションを一括追加するGoogle Colabファイルを公開しました"
emoji: "🔗"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["notion", "notionapi", "googlecolab", "python", "api"]
published: true
---

### gist のリンク

https://gist.github.com/kikutadev/25b4f9fc725b14d2bcb4103182f78539

### Colab のリンク

https://colab.research.google.com/gist/kikutadev/25b4f9fc725b14d2bcb4103182f78539/notion-api-batch-relation.ipynb

### 使い方

以下の 5 ステップで進めていきます。

1. NOTION_TOKEN の取得
2. Notion でインテグレーションを追加する
3. database_id の取得
4. プロパティの編集
5. 実行

#### 1. NOTION_TOKEN の取得

https://notion.so/my-integrations から取得してください。

`secret`から始まる文字列です。これをコピーして、colab の`NOTION_TOKEN`に貼り付けてください。

#### 2. Notion でインテグレーションを追加する

作成したインテグレーションを以下のように追加します。
![](https://storage.googleapis.com/zenn-user-upload/2ef71db4f29f-20230825.png)

#### 3. database_id の取得

notion.so で、リレーションを追加したいデータベースを開きます。

URL の`database/`の後ろの文字列が`database_id`です。

ここではリレーション元が`MAIN_DATABASE_ID`、リレーション先が`SUB_DATABASE_ID`とします。
これらをコピーして、colab の`MAIN_DATABASE_ID`と`SUB_DATABASE_ID`に貼り付けてください。

#### 4. プロパティの編集

以下の 3 つのプロパティを編集していきます。

- `SUB_DATABASE_relation_property_name`
  - リレーションを追加したいプロパティ名を入力してください。

:::message
`①基となるメインデータベースの全アイテムのIDを一括取得して、別のデータベースを作成しリレーションを追加`のみを試したい場合は、ここまでで OK です。
:::

- `SUB_DATABASE_property_name_searching_by`
  - メインデータベースを検索するのに使用するプロパティ名を入力してください。
- `MAIN_DATABASE_property_name_to_find`
  - 検索したいメインデータベースのプロパティ名を入力してください。

例を示しておきます。
以下の Sub Database の`RelationToMain`を追加したいとします。

![](https://storage.googleapis.com/zenn-user-upload/68289736643c-20230825.png)
![](https://storage.googleapis.com/zenn-user-upload/4c656742541c-20230825.png)

- `SUB_DATABASE_relation_property_name`は、以下でいうところの`RelationToMain`です。
- そして、検索元のプロパティ名である`SUB_DATABASE_property_name_searching_by`は、`Name`です。
- 検索したいメインデータベースのプロパティ名である`MAIN_DATABASE_property_name_to_find`は、`MAIN_NAME`です。

これらを入力したらこうなります。

![](https://storage.googleapis.com/zenn-user-upload/0ec1c22aabe3-20230825.png)

#### 5. 実行

こちらで準備は完了ですので、実行してみましょう。

② の検索する方をやってみます。先ほどのデータベースに以下のように、同じ名前のアイテムを追加してみます。
![](https://storage.googleapis.com/zenn-user-upload/986b63d60ed7-20230825.png)
![](https://storage.googleapis.com/zenn-user-upload/b98817943558-20230825.png)

##### こちらで実行すると、、、

無事追加されました！
![](https://storage.googleapis.com/zenn-user-upload/9c1235e0acdf-20230825.png)

一度リレーションが追加されたら、後は Excel でいう`VLOOKUP`のように、ロールアップの機能により、リレーション先のプロパティを取得できます。
以下は、`MAIN_DATABASE`にステータスというプロパティを追加し、`SUB_DATABASE`にロールアップで`MAIN_DATABASE`のステータスを取得した例です。
![](https://storage.googleapis.com/zenn-user-upload/e9e414947b05-20230825.png)
![](https://storage.googleapis.com/zenn-user-upload/7300fc97b918-20230825.png)

### まとめ

ここでは以下の 5 手順を紹介しました。

1. NOTION_TOKEN の取得
2. Notion でインテグレーションを追加する
3. database_id の取得
4. プロパティの編集
5. 実行

ここでは Name というプロパティで取得しましたが、少し手直しすれば、アイデア次第で色々発展させられるのではないかと思います。
ぜひ活用してみてください。

### gist のリンク

https://gist.github.com/kikutadev/25b4f9fc725b14d2bcb4103182f78539

### Colab のリンク

https://colab.research.google.com/gist/kikutadev/25b4f9fc725b14d2bcb4103182f78539/notion-api-batch-relation.ipynb
