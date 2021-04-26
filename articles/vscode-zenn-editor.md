---
title: "【VS Code】Zenn の記事をワンクリックでプレビューする"
emoji: "🔍"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "zenn"]
published: true
---

# はじめに

VS Code で Zenn の記事を執筆する際は [Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli) を使って記事をプレビューすることで仕上がりを確認できます。このプレビューは Web ブラウザで表示する方式のため、執筆にどのようなエディタを使ってもプレビューできるという利点があります。

私が Zenn の記事を執筆する際は書きながら仕上がりを確認するため VS Code のウィンドウを左に、Zenn CLI のプレビューが表示されたブラウザのウィンドウを右に並べるという方法をとっていました。
Zenn CLI を起動し、ブラウザを開き、ウィンドウの位置を調整し、編集したい記事を開く…

![](https://storage.googleapis.com/zenn-user-upload/4a7vgjyzzegn06sk3idmyn7cfv4l)

Zenn の執筆体験は素晴らしいです。が、もっと手間を減らして記事を書き始める心理的障壁を減らしたいと思いました。記事を編集中にワンクリックでプレビューできると嬉しいですね。

幸いにも VS Code には拡張機能を追加するしくみがあります。

# VS Code Zenn Editor

というわけで、Zenn CLI のプレビューを VS Code 内に表示する拡張機能を作りました。

https://marketplace.visualstudio.com/items?itemName=negokaz.zenn-editor

この拡張を使うと編集中の記事をワンクリックでプレビューできます！

![](https://storage.googleapis.com/zenn-user-upload/iu0i5xw122sn8u35sqzxo6ijrx1t)
*プレビューの様子*

アクティブな Markdown ファイルの切り替えに追従してプレビューされる記事が切り替わるようになっています。

## インストール方法

[Visual Studio Code Marketplace](https://marketplace.visualstudio.com/items?itemName=negokaz.zenn-editor) からインストールするか、コマンドパレットから「Extensions: Install Extensions」を選んで `zenn-editor` と検索して「Install」を押すとインストールできます。

## 使い方

`articles` か `books` 配下にある（Zenn の記事になる）Markdown ファイルを開いて、右上にある次のアイコンをクリックするとプレビューが開きます。

![](https://storage.googleapis.com/zenn-user-upload/p7tuqxb9y4txhgfepyy47ei6naa9)
*プレビューを表示するアイコン*

## 不具合報告・機能要望

不具合の報告や追加機能の要望は GitHub から issue を作成いただけると幸いです。

https://github.com/negokaz/vscode-zenn-editor/issues

# おまけ：アーキテクチャ

どのような方式でプレビューを実現しているのかを簡単に書き残しておきます。

大まかにいうと、拡張機能から Zenn CLI のプレビューをバックグラウンドで起動し、それを VS Code の Webview で表示するという方式です。

![](https://storage.googleapis.com/zenn-user-upload/7sbrt9h9hn8e1lc1uge2m5sv8d2x =400x)

工夫ポイントとしてはスクリーンの横幅を節約するためサイドバーを隠しました。
サイドバーにあるページ切り替えのメニューは Markdown ファイルと連動したプレビューの切り替えが機能的に代替となるためサイドバーがなくても問題ないという判断です。

これを実現するためには、Zenn CLI のプレビューを参照するために Webview の中に埋め込んだ `<iframe>` の中のコンテンツを編集する必要があります。しかしながら、VS Code の Webview のオリジンと Zenn CLI のプレビューのオリジンが異なるため、CORS の制約を受け `<iframe>` 内のコンテンツを編集できません。

そこで、Webview と Zenn CLI のプレビューの間にリバースプロキシを挟むことにしました。リバースプロキシがプレビューへの踏み台となるページ `/__proxy_index`（実際とは異なります）を提供し、このページ以外の URL へのアクセスは Zenn CLI のプレビューへ転送します。`/__proxy_index` 上では `<iframe>` を使ってプレビューしたいページを参照しますが、オリジン（ホスト名とポート）はリバースプロキシのものにします。

![](https://storage.googleapis.com/zenn-user-upload/w16p9ypgz962lk8k630fnbaugl3i =600x)

こうすることで晴れて(?) `/__proxy_index` とプレビューのページが同じオリジンになり、`/__proxy_index` から `<iframe>` 内のプレビューページを編集できるようになったため、サイドバーを隠すことができました。

アクティブな Markdown ファイルに連動したプレビューの切り替えなどは、CORS の制約を受けない [`postMessage`](https://developer.mozilla.org/ja/docs/Web/API/Window/postMessage) を用いて制御しています。
