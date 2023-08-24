---
title: "NextjsのServer ActionsとGoogle Spreadsheetでお手軽お問い合わせフォーム"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "ServerActions", "GoogleSpreadsheet", "お問い合わせフォーム"]
published: false
---

# 結論：こんな感じでお問い合わせを送付できるように

![](https://storage.googleapis.com/zenn-user-upload/4562c685543f-20230821.gif)

一部省略していますが、POST するコードはこちらです。（詳細は後ほど）
10 行程度で Google SpreadSheet(以下 GSS)に POST できちゃいます。簡単！

```ts
export async function sendInquiry(data: FormData) {
  // バリデーションを行う（後ほど説明）
  // Spreadsheetの諸々の認証は省略（後ほど説明）

  const doc = new GoogleSpreadsheet(sheetId, jwt);
  await doc.loadInfo();
  const inquirySheet = doc.sheetsByTitle["inquiry"];

  const newRow = {
    name: data.get("name") as string,
    email: data.get("email") as string,
    message: data.get("message") as string,
  };
  await inquirySheet.addRow(newRow);
}
```

実際の動作はこちらで確認できます。

https://kikuta.dev/inquiry

:::message
なお現在 UI が残念（送信したかどうか分からない、連打できてしまう、メッセージの幅が小さい等、変な部分が多い点はご了承ください。
**連打・大量送信はなるべくしないように**お願いします 😞
:::

## 注意書き

:::message alert
筆者は App Router 初心者（Pages Router も初心者）なので、ミスがありましたらコメント等いただけると幸いです。

:::

### 使用したバージョン

```json
"next": "^13.4.13",
```

# 作業手順

では、具体的にどうすれば Server Actions で Google Spreadsheet に POST できるか説明していきます。

## Google Spreadsheet のセットアップ

私のブログに[タイムライン](https://kikuta.dev/timeline#all)があるのですが、catnose99 さんのレポを参考にセットアップしました。こちらにセットアップ手順が記載されているので、参考にしていただければと思います。
なお、手順としては、`Google Consoleでのセットアップ、Google Spreadsheetの作成、envに記載`といった内容になります。

https://github.com/catnose99/timeline#configure-keys

### 実装

実装ですが、以下に番号をつけて説明します。なお、実際の実装では zod でバリデーションを行なっています。
①`"use server"`により server actions であることを明示します。なお関数の中にも書いていますが、どちらか片方があれば問題ない気もします（？）。
②`JWT`により認証します。
③FormData を受け取り、FormData の get メソッドで値を取得し、オブジェクトにします。
④Google Spreadsheet 内に行を追加します(POST)。

:::details page.tsx

```ts:app/inquiry/sendInquiry.ts
"use server"; //①

import { JWT } from "google-auth-library";//②
import { GoogleSpreadsheet } from "google-spreadsheet";

export async function sendInquiry(data: FormData) {//③
 "use server"; //①

 const SCOPES = [
   "https://www.googleapis.com/auth/spreadsheets",
   "https://www.googleapis.com/auth/drive.file",
 ];
 const sheetId = process.env.SHEET_ID;
 const clientEmail = process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL;
 const privateKey = process.env.GOOGLE_PRIVATE_KEY?.replace(/\\n/gm, "\n");

 const jwt = new JWT({ //②
   email: clientEmail,
   key: privateKey,
   scopes: SCOPES,
 });

 // envファイルからの読み込みが問題ないかバリデーション
 if (
   typeof sheetId === "undefined" ||
   typeof clientEmail === "undefined" ||
   typeof privateKey === "undefined"
 ) {
   console.error(
     'env "SHEET_ID", "GOOGLE_SERVICE_ACCOUNT_EMAIL", "GOOGLE_PRIVATE_KEY" are required.',
   );
   process.exit(1);
 }

 const doc = new GoogleSpreadsheet(sheetId, jwt);

 await doc.loadInfo();
 const inquirySheet = doc.sheetsByTitle["inquiry"];

 const newRow = {
   name: data.get("name") as string,//③
   email: data.get("email") as string,
   message: data.get("message") as string,
 };
 await inquirySheet.addRow(newRow);//④
}

```

:::

## Server Actions のセットアップ

### next.config.js のセット

`next.config.js`に以下を追記します。

```js:next.config.js
const nextConfig = {
  experimental: {
    serverActions: true,
  }
};
```

### フォームのページを作成

ポイントは以下の 2 つです。
①server actions をインポートする
②name 属性を設定する

tailwindcss を使用しているので、tailwindcss のクラスが多くなっていますが、正直ほとんど HTML です。
なお、`"use client"`を先頭に付与すれば、useState や React Hook Form を使用することもできます。

:::details page.tsx

```tsx:app/inquiry/page.tsx
import React from "react";

import { sendInquiry } from "./sendInquiry";

export default function InquiryPage() {
  return (
      <form
        action={sendInquiry}//①
        className="m-4 min-w-[90vw] rounded bg-gray-600 p-6 shadow-2xl sm:min-w-[600px]"
      >
        <div className="mb-4">
          <label htmlFor="name" className="mb-2 block text-sm font-bold">
            名前:
          </label>
          <input
            type="text"
            id="name"
            name="name"//②
            className="w-full rounded border bg-gray-800 px-3 py-2 text-sm"
          />
        </div>
        <div className="mb-4">
          <label htmlFor="email" className="mb-2 block text-sm font-bold">
            メールアドレス*:
          </label>
          <input
            type="email"
            id="email"
            name="email"//②
            className="w-full rounded border bg-gray-800 px-3 py-2 text-sm"
          />
        </div>
        <div className="mb-4">
          <label htmlFor="message" className="mb-2 block text-sm font-bold">
            メッセージ*:
          </label>
          <textarea
            id="message"
            name="message"//②
            className="w-full rounded border bg-gray-800 px-3 py-2 text-sm"
          />
        </div>
        <div>
          <button
            type="submit"
            className="rounded bg-rose-500 px-4 py-2 text-white hover:bg-rose-700"
          >
            送信
          </button>
        </div>
      </form>
  );
}
```

:::

## 以上で完了

以上で、Google Spreadsheet に POST できるようになります。触ったファイルは、config 系を除けば page.tsx と sendInquiry.ts の 2 つだけです。簡単！

## 追加メモ：action={sendInquiry}とは？FormData とは？get メソッドとは？

私がつまづいた部分をもう少しメモしておきます。

従来の React だと action 属性は使わず、`onSubmit`とかだったと思います。Server Action では、action 属性を使用します。action 属性には、今回は`sendInquiry`を指定しています。

そして、この action 属性に渡される関数の引数に FormData が渡されます。
FormData は、HTMLFormElement 内の入力要素の値を取得するためのオブジェクトと考えれば良いのではないかと思います。
値取得には、**get メソッド**と各 input 要素の name 属性を使用します。

Server Actions を使うまではどういうイメージか全く分かりませんでしたが、実際に使ってみるととても簡単でした。

### （余談）FormData の型定義

action 属性の型定義を辿っていくと、FormData の型定義があります。
途中で`DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_FORM_ACTIONS`とかいう型を飛び越える必要がありなんか怖いですが、React 公式なので大丈夫でしょう（）

![](https://storage.googleapis.com/zenn-user-upload/b7d7cf80b51a-20230821.png)

:::details @types/react/index.d.ts

```ts:@types/react/index.d.ts
    interface FormHTMLAttributes<T> extends HTMLAttributes<T> {
        acceptCharset?: string | undefined;
        action?:
            | string
            | undefined
            | DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_FORM_ACTIONS[keyof DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_FORM_ACTIONS];
        autoComplete?: string | undefined;
        encType?: string | undefined;
        method?: string | undefined;
        name?: string | undefined;
        noValidate?: boolean | undefined;
        target?: string | undefined;
    }
```

:::

:::details react/experimentals.d.ts

```ts:react/experimentals.d.ts
    interface DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_FORM_ACTIONS {
        functions: (formData: FormData) => void;
    }
```

:::

:::details typescript/lib/lib.dom.d.ts

```ts:typescript/lib/lib.dom.d.ts
interface FormData {
    /** [MDN Reference](https://developer.mozilla.org/docs/Web/API/FormData/append) */
    append(name: string, value: string | Blob): void;
    append(name: string, value: string): void;
    append(name: string, blobValue: Blob, filename?: string): void;
    /** [MDN Reference](https://developer.mozilla.org/docs/Web/API/FormData/delete) */
    delete(name: string): void;
    /** [MDN Reference](https://developer.mozilla.org/docs/Web/API/FormData/get) */
    get(name: string): FormDataEntryValue | null;
    /** [MDN Reference](https://developer.mozilla.org/docs/Web/API/FormData/getAll) */
    getAll(name: string): FormDataEntryValue[];
    /** [MDN Reference](https://developer.mozilla.org/docs/Web/API/FormData/has) */
    has(name: string): boolean;
    /** [MDN Reference](https://developer.mozilla.org/docs/Web/API/FormData/set) */
    set(name: string, value: string | Blob): void;
    set(name: string, value: string): void;
    set(name: string, blobValue: Blob, filename?: string): void;
    forEach(callbackfn: (value: FormDataEntryValue, key: string, parent: FormData) => void, thisArg?: any): void;
}

```

:::

## 現時点でよく分かっていないこと

form の action 属性に渡す`FormData`は、input 要素の name 属性をキーとしたオブジェクトになるのですが、型定義を外から渡すことができないと思われます。
なので、React Hook Form などを使用するにしても、結局送信する段階で`data.get("name")`といった形で取得するので、完全な型安全は今の所無理そう、とか、プロパティの抜け漏れとかが発生しそう、とか思いました（解決方法があれば教えてください）。

なので、`"use client"`を先頭に付与して、React Hook Form を使用してみたのですが、自分では実装方法が分からず、一旦完全な Server Components の作成方法をご紹介しました。
