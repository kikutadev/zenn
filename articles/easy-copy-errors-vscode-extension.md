---
title: ""
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# ChatGPTに最適なエラー情報をワンショットでコピー！VSCode拡張機能「Easy Copy Errors」実装編 🚀

こんにちは！今回は、開発作業中に遭遇するエラーを生成AIに効率的に伝えるためのVSCode拡張機能「Easy Copy Errors」について紹介します。コーディング中にエラーが出て、「このエラーどう解決すればいいんだろう...ChatGPTに聞いてみよう」と思ったとき、エラー内容をいちいち手動でコピペするのって面倒ですよね😅 そんな悩みを解決する拡張機能を作ってみました！

## はじめに：なぜこの拡張機能が必要なのか

プログラミングをしていると、必ず遭遇するのがエラーメッセージ。特にTypeScriptなどの型エラーは情報量が多く、ChatGPTなどの生成AIに質問する際に「ファイルのどこで」「どんなエラーが」「どんなコードで」発生しているかを正確に伝えることが重要です。

例えば以下のようなエラーが出たとします。

```
Type 'Err' is not assignable to type 'Result'.
```

これだけ見てもどのファイルのどの行で起きたのか、その行のコードがどうなっているのかが分かりません。これをAIに効率よく伝えたい。そんな要望から生まれたのが「Easy Copy Errors」です。

## 実装：TypeScriptでVSCode拡張機能を作る

それでは実装していきましょう！VSCode拡張機能は、Node.jsとTypeScriptを使って作ることができます。

### ①プロジェクトの初期化

まずは拡張機能のプロジェクトを作成します。

```bash
# Yeomanとgenerator-codeをインストール
npm install -g yo generator-code

# プロジェクトを生成
yo code
```

ここでいくつか質問されるので、以下のように回答します。
- New Extension (TypeScript)を選択
- 名前を「easy-copy-errors」にする
- その他は適宜入力

### ②拡張機能のコアロジックを実装

次に、`src/extension.ts`を編集して、エラー情報を取得してコピーする機能を実装します。

```typescript
import * as vscode from 'vscode';
import * as path from 'path';

export function activate(context: vscode.ExtensionContext) {
  console.log('Extension "easy-copy-errors" is now active');

  // コマンドの登録
  let disposable = vscode.commands.registerCommand(
    'easy-copy-errors.copyErrors',
    async () => {
      // アクティブなエディタを取得
      const editor = vscode.window.activeTextEditor;
      if (!editor) {
        vscode.window.showInformationMessage('エディタが開かれていません');
        return;
      }

      // ドキュメント取得
      const document = editor.document;

      // 現在のファイルのDiagnosticsを取得
      const diagnostics = vscode.languages.getDiagnostics(document.uri);

      if (diagnostics.length === 0) {
        vscode.window.showInformationMessage(
          '現在のファイルにエラーはありません'
        );
        return;
      }

      // Diagnosticsをフォーマット
      const formattedDiagnostics = diagnostics
        .map((diag) => formatDiagnostic(diag, document))
        .join('\n\n');

      // クリップボードにコピー
      await vscode.env.clipboard.writeText(formattedDiagnostics);

      vscode.window.showInformationMessage('エラー情報をコピーしました！');
    }
  );

  context.subscriptions.push(disposable);
}
```

そうですね、ここでは主に「コマンドを登録」→「エディタからDiagnosticsを取得」→「フォーマット」→「クリップボードにコピー」という流れです。ぱっと読んでも分かりますね！😊

### ③エラー情報のフォーマット関数を実装

続いて、取得したDiagnostics情報を見やすい形式にフォーマットする関数を実装します。

```typescript
/**
 * 相対パスを取得する関数
 */
function getRelativePath(uri: vscode.Uri): string {
  const workspaceFolder = vscode.workspace.getWorkspaceFolder(uri);
  if (workspaceFolder) {
    return path.relative(workspaceFolder.uri.fsPath, uri.fsPath);
  }
  return uri.fsPath;
}

/**
 * Diagnosticを整形する関数
 */
function formatDiagnostic(
  diagnostic: vscode.Diagnostic,
  document: vscode.TextDocument
): string {
  // 行番号（0からスタートなので+1する）
  const line = diagnostic.range.start.line + 1;

  // エラーが発生している行の内容を取得
  const lineContent = document.lineAt(diagnostic.range.start.line).text;

  // ファイルの相対パス取得
  const relativePath = getRelativePath(document.uri);

  // フォーマット済みの出力を生成
  return `file: ${relativePath}\nLine ${line}:      ${lineContent}\n${diagnostic.message}`;
}
```

この関数では、ファイルの相対パス、エラーのある行番号、その行のコード、そしてエラーメッセージを組み合わせて、AIにとって分かりやすい形式にフォーマットしています。

### ④package.jsonの設定

次に、`package.json`にコマンドとキーボードショートカットを登録します。

```json
"contributes": {
  "commands": [
    {
      "command": "easy-copy-errors.copyErrors",
      "title": "エラー情報をコピー"
    }
  ],
  "keybindings": [
    {
      "command": "easy-copy-errors.copyErrors",
      "key": "ctrl+shift+e",
      "mac": "cmd+shift+e",
      "when": "editorTextFocus"
    }
  ]
}
```

これで`Ctrl+Shift+E`（Macなら`Cmd+Shift+E`）を押すと、エラー情報がコピーできるようになります！便利ですね👍

### ⑤設定オプションの追加

ユーザーがフォーマットをカスタマイズできるように、設定オプションも追加しておきましょう。

```json
"configuration": {
  "title": "Easy Copy Errors",
  "properties": {
    "easyCopyErrors.useNewFormat": {
      "type": "boolean",
      "default": true,
      "description": "ファイルパスとコード行内容を含む新しいフォーマットを使用する"
    },
    "easyCopyErrors.format": {
      "type": "string",
      "default": "[${severity}] Line ${line}, Column ${column}: ${message}",
      "description": "エラーメッセージのフォーマット（useNewFormatがfalseの場合に使用）"
    }
  }
}
```

この設定を活用するように、先ほどの`formatDiagnostic`関数を拡張します。

```typescript
function formatDiagnostic(
  diagnostic: vscode.Diagnostic,
  document: vscode.TextDocument
): string {
  const config = vscode.workspace.getConfiguration('easyCopyErrors');
  const useNewFormat = config.get('useNewFormat') ?? true;

  // 行番号とその他の情報取得
  const line = diagnostic.range.start.line + 1;
  const column = diagnostic.range.start.character + 1;
  const message = diagnostic.message;
  const lineContent = document.lineAt(diagnostic.range.start.line).text;
  const relativePath = getRelativePath(document.uri);

  if (useNewFormat) {
    // 新しいフォーマット（AI向け）
    return `file: ${relativePath}\nLine ${line}:      ${lineContent}\n${message}`;
  } else {
    // カスタムフォーマット
    const format = config.get('format') ||
      '[${severity}] Line ${line}, Column ${column}: ${message}';

    return format
      .replace(/\${line}/g, line.toString())
      .replace(/\${column}/g, column.toString())
      .replace(/\${message}/g, message)
      .replace(/\${lineContent}/g, lineContent)
      .replace(/\${relativePath}/g, relativePath);
  }
}
```

## エラーに遭遇した時の対応策

拡張機能の開発中、いくつかのエラーに遭遇しました。ここではその対処法を紹介します。

### ①VSIXファイルがインストールできない

```
Error: Unable to install extension 'undefined_publisher.easy-copy-errors' as it is not compatible with VS Code '1.96.2'.
```

このエラーが出た場合は、`package.json`の`engines.vscode`フィールドのバージョンとVS Codeのバージョンが一致していない可能性があります。以下のように修正します。

```json
"engines": {
  "vscode": "^1.96.0"
}
```

また、`publisher`フィールドが未定義の場合も同様のエラーが表示されることがあります。個人利用なら適当な値でOKです。

```json
"publisher": "your-name",
```

### ②@types/vscodeのバージョン不一致エラー

```
ERROR  @types/vscode ^1.97.0 greater than engines.vscode ^1.9.0. Consider upgrade engines.vscode or use an older @types/vscode version
```

このエラーは、型定義のバージョンとエンジンのバージョンが一致していないときに発生します。解決策は2つ：

1. `package.json`の`engines.vscode`を型定義に合わせる
2. `@types/vscode`のバージョンを下げる

```bash
# 選択肢2の場合（非推奨）
npm install --save-dev @types/vscode@^1.96.0
```

ただし、①の方法（エンジンバージョンを型定義に合わせる）が推奨です。

## 生成AIとの連携：どう使うのか

この拡張機能を使えば、エラーが発生した時に`Ctrl+Shift+E`を押すだけで、以下のような情報がクリップボードにコピーされます。

```
file: src/components/Button.tsx
Line 42:      return {label}
Property 'handlClick' does not exist. Did you mean 'handleClick'? ts(2551)
```

これをそのままChatGPTなどの生成AIに貼り付けるだけで、AIはエラーの発生場所やコード内容を正確に把握できます。

例えばこんな感じで聞くことができます👇

```
以下のTypeScriptのエラーを解決する方法を教えてください。

file: src/components/Button.tsx
Line 42:      return {label}
Property 'handlClick' does not exist. Did you mean 'handleClick'? ts(2551)
```

AIは「handlClick」が「handleClick」の打ち間違いであることをすぐに理解し、適切な解決策を提案してくれるでしょう！

## まとめ：生産性向上のための小さな一歩

今回紹介した「Easy Copy Errors」拡張機能は、開発中に遭遇するエラーをAIに効率的に伝えるための小さなツールです。ショートカットキー一つでエラー情報をコピーできるようになり、AIとのやり取りがスムーズになります。

この拡張機能のポイントは以下の通りです：
- ファイルパスを含める（どこで起きたエラーか分かる）
- エラーが発生した行のコードを含める（コンテキストが理解しやすい）
- エラーメッセージをそのまま含める（正確な情報が伝わる）

これらの情報が揃っていると、AIは問題の本質をより正確に理解し、適切な解決策を提案しやすくなります。

ぜひみなさんも、この拡張機能を使って生成AIとの開発ワークフローを効率化してみてください！質問や改善案があれば、ぜひコメントで教えてくださいね😊

もう手動でエラー情報をコピペする時代は終わりました。キーボードショートカット一つで、AIとのスムーズなコミュニケーションを始めましょう！

---

ちなみに、この拡張機能はMITライセンスで公開していますので、自由に利用・改変していただけます。GitHubリポジトリへのスター⭐もお待ちしています！

