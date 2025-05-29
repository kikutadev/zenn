---
title: "VSCodeでキーボード操作に慣れたら超快適な件"
emoji: "⌨️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "keyboard", "keyboardshortcut", "mouse", "touchpad"]
published: true
---

## VSCode でキーボード操作に慣れたら超快適な件

今まで、調べるのが面倒で、タッチパッドで操作してたことも多いです。
最近エディタを全て VSCode に寄せて、ショートカットを覚えて、「VSCode に全部賭けろ」状態にしました。
結果、超快適になりました。
皆様にもぜひ共有したいので、おすすめのキーボードショートカットと、カスタマイズ済のキーボードショートカット（ `keybindings.json`） を紹介します。

### サイドバーをキーボードで操作しよう

まずショートカットを使って慣れていくところから始まります。
サイドバーをうまく使うのが一番大事だと思います。

| 機能                    | shortcut | 一口メモ         | custom  |
| ----------------------- | ------------------------ | ---------------- | ------- |
| Explorer に focus       | command + shift + E      | ファイル探すやつ | default |
| Search に focus         | command + shift + F      | 検索するやつ     | default |
| Source Control に focus | command + shift + G      | git 見るやつ     | custom  |

ちなみに、ショートカットキーを E、F、G と ABC 順に並べてみました。結構覚えやすいのではないでしょうか？
source control については、カスタマイズしてます。
デフォルトだと control+shift+g で、他と統一されてないとわかりづらいので。

### ファイル操作（主に Explorer）

| 機能                 | shortcut | 一口メモ                                                                  |
| -------------------- | ------------------------ | ------------------------------------------------------------------------- |
| 矢印でフォーカス移動 | ↑↓                       | Explorer にフォーカスしてから！                                           |
| フォルダを開閉       | →←                       | 実はあんまり使わないかも                                                  |
| ファイルを開く       | command + ↓              | 調べる前までは、Enter で rename になっちゃうのでクリックして開いてました… |
| ファイルを開く       | command + P              | 超使う！下で解説                                                          |

個人的には、特に `command + P` が一番便利でした。
今まで、ファイルを探す時にフォルダをぽちぽち展開してたんですが、ファイル名がわかってるなら、`command + P` で探して開くだけで良かったんだよね。

### keybindings.json のカスタマイズ

私の VSCode のショートカットをカスタマイズ結果はこちら！
json 内に説明コメントも書いてます。（そのままコピーで使えます）

:::details keybindings.json

```json
// Place your key bindings in this file to override the defaults
[
  // divとかを囲うときに使うやつ。Next Edit Suggestionがあるからいらないかも
  {
    "key": "shift+cmd+w",
    "command": "editor.emmet.action.wrapWithAbbreviation"
  },
  // ウィンドウを閉じるときに使うやつを無効化。タブ閉じる時にcommand+wを長押ししちゃうので、ウィンドウまで閉じるのがストレスでした。
  {
    "key": "shift+cmd+w",
    "command": "-workbench.action.closeWindow"
  },
  // タブを閉じるときに使うやつを無効化。タブ閉じる時にcommand+wを長押ししちゃうので、ウィンドウまで閉じるのがストレスでした。
  {
    "key": "cmd+w",
    "command": "-workbench.action.closeWindow",
    "when": "!editorIsOpen && !multipleEditorGroups"
  },
  // ファイル作成
  {
    "key": "cmd+n",
    "command": "explorer.newFile"
  },
  // untitledFileは、たまにしか使わないので変更
  {
    "key": "cmd+n",
    "command": "-workbench.action.files.newUntitledFile"
  },
  // untitledFileは、たまにしか使わないので変更
  {
    "key": "shift+alt+cmd+n",
    "command": "workbench.action.files.newUntitledFile"
  },
  // フォルダ作成。
  {
    "key": "alt+cmd+n",
    "command": "explorer.newFolder"
  },
  // scmにフォーカス
  {
    "key": "shift+cmd+g",
    "command": "workbench.view.scm",
    "when": "workbench.scm.active"
  },
  // scmにフォーカスするため、無効化
  {
    "key": "shift+cmd+g",
    "command": "-editor.action.previousMatchFindAction",
    "when": "editorFocus"
  },
  // scmにフォーカスするため、無効化
  {
    "key": "shift+cmd+g",
    "command": "-workbench.action.terminal.openDetectedLink",
    "when": "accessibleViewIsShown && terminalHasBeenCreated && accessibleViewCurrentProviderId == 'terminal'"
  },
  // デフォルトのscmにフォーカスは無効化
  {
    "key": "ctrl+shift+g",
    "command": "-workbench.view.scm",
    "when": "workbench.scm.active"
  },
  // commit messageの生成。g→gの順
  // commit messageを生成するコマンド（cursor）
  {
    "key": "alt+cmd+g alt+cmd+g",
    "command": "cursor.generateGitCommitMessage"
  },
  // commit messageを生成するコマンド（windsurf）
  {
    "key": "alt+cmd+g alt+cmd+g",
    "command": "windsurf.generateCommitMessage"
  },
  // commit messageを生成するコマンド（github copilot）
  {
    "key": "alt+cmd+g alt+cmd+g",
    "command": "github.copilot.git.generateCommitMessage"
  },
  // staging
  {
    "key": "alt+cmd+g alt+cmd+s",
    "command": "git.stage"
  },
  // commit git commitなので、g→cの順
  {
    "key": "alt+cmd+g alt+cmd+c",
    "command": "git.commit",
    "when": "!operationInProgress"
  },
  // diffを開く。openなので、g→oの順
  {
    "key": "alt+cmd+g alt+cmd+o",
    "command": "git.openChange"
  },
  // git checkout。branchを選ぶイメージで、g→bの順
  {
    "key": "alt+cmd+g alt+cmd+b",
    "command": "git.checkout"
  },
  // easy copy errorsというVSCode拡張機能のショートカットをカスタマイズしています。
  {
    "key": "alt+cmd+e",
    "command": "easy-copy-errors.copyErrors"
  },
  {
    "key": "alt+cmd+d",
    "command": "easy-copy-errors.removeDiffMarkers"
  },
  // cmd+bでサイドバーの開閉ですが、マークダウンだとこれで上書きされちゃうので変更。
  {
    "key": "shift+cmd+b",
    "command": "markdown.extension.editing.toggleBold",
    "when": "editorTextFocus && !editorReadonly && editorLangId =~ /^markdown$|^rmd$|^quarto$/"
  },
  // cmd+bでboldをオンオフするショートカットを無効化
  {
    "key": "cmd+b",
    "command": "-markdown.extension.editing.toggleBold",
    "when": "editorTextFocus && !editorReadonly && editorLangId =~ /^markdown$|^rmd$|^quarto$/"
  },
  // pinするときのやつ。
  {
    "key": "cmd+k cmd+enter",
    "command": "workbench.action.pinEditor",
    "when": "!activeEditorIsPinned"
  },
  // command+k押した後にshift押しながらenterってトリッキーすぎたので変えました。
  {
    "key": "cmd+k shift+enter",
    "command": "-workbench.action.pinEditor",
    "when": "!activeEditorIsPinned"
  },
  // 同上
  {
    "key": "cmd+k cmd+enter",
    "command": "workbench.action.unpinEditor",
    "when": "activeEditorIsPinned"
  },
  // 同上
  {
    "key": "cmd+k shift+enter",
    "command": "-workbench.action.unpinEditor",
    "when": "activeEditorIsPinned"
  },
  // 補完候補を開く
  {
    "key": "cmd+space",
    "command": "editor.action.triggerSuggest",
    "when": "editorHasCompletionItemProvider && textInputFocus && !editorReadonly && !suggestWidgetVisible"
  },
  // 補完候補の、cmd+iは無効化
  {
    "key": "cmd+i",
    "command": "-editor.action.triggerSuggest",
    "when": "editorHasCompletionItemProvider && textInputFocus && !editorReadonly && !suggestWidgetVisible"
  }
]
```

:::

カスタマイズ内容で特徴的なところをかいつまんで説明します。

SCM（Git）に関するショートカットをいい感じに整理してます。だいたいのことはショートカットでできるように。

- `cmd+alt+g` の後に g でコミットメッセージ生成：これめっちゃ便利です。
- `cmd+alt+g` の後に c でコミット
- `cmd+alt+g` の後に o で diff
- `cmd+alt+g` の後に b でブランチ切り替え
- `cmd+alt+g` の後に s でステージング

:::details コミットメッセージ生成の補足
windsurf, cursor, vscode の分を `keybindings.json` に追加してあります。
私は windsurf が next edit suggestion が制限なし無料なので、メインで使ってます。
ただし、コミットメッセージ生成は windsurf だと有料で、github copilot だと無料なので、同じプロジェクトを windsurf と vscode で開き、git を使う時だけ vscode でコミットしてます。
お金かかるの嫌なので 😢
:::

### 後書き

その他にも使っているショートカットはあるのですが、力尽きてしまいました。
久々に生成 AI に頼らず書いたのですが、結構疲れました。。
次は superwhisper で音声入力で書いてみようかな。
