---
title: "Git のコミットプレフィックスを絵文字で彩る（自動で）"
emoji: "🎨"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["git"]
published: true
---

# はじめに

Git のコミットメッセージにプレフィックスを付けることで、次のような効果があると言われています。

- メッセージからコミットの内容を予想しやすくなりレビュアーの負荷を下げられる
- プレフィックスを意識することでコミット粒度を自然と適度に分割できるようになる
- 後でログからコミットを探しやすくなる

[【今日からできる】コミットメッセージに 「プレフィックス」 をつけるだけで、開発効率が上がった話 - Qiita](https://qiita.com/numanomanu/items/45dd285b286a1f7280ed)

私自身も実践し、効果を実感していたのですが、`feat:` などの文字だけだと直感的にどのプレフィックスなのか把握できないし、プレフィックスに絵文字を付けたいなーと思いました。

しかし、コマンドラインで絵文字を入力するのは手間がかかります。
GitHub では絵文字の代わりに Emoji code を書くことで絵文字を表示するという機能がありますが、これを覚えるのは辛いし、コマンドラインでログを見たときに絵文字が表示されないという点が私にとってはイマイチでした。

| Emoji code  | 絵文字 |
| ---------------- | ----- |
| `:sparkles:`     | ✨     |
| `:construction:` | 🚧    |
| `:books:`        | 📚    |

[Basic writing and formatting syntax - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/github/writing-on-github/basic-writing-and-formatting-syntax#using-emoji)

そこで、以前 **プレフィックスを書くと自動で絵文字を付けてくれる** ツールを作りました。

# Git Fancy Message Prefix

ツールはGitHubで公開しています。Gitの [prepare-commit-msg](https://git-scm.com/docs/githooks#_prepare_commit_msg) フックスクリプトとして実装しました。

👉 [negokaz/git-fancy-message-prefix: A Git prepare-commit-msg hook for fancy commit message](https://github.com/negokaz/git-fancy-message-prefix)

:::message
所属するチームで、定められた prepare-commit-msg フックスクリプトがある場合は使用しないでください。他のフックスクリプトと併用できるようには作っていません。
:::

このフックスクリプトを使うと、コミットログがこんな感じになります。

![](https://storage.googleapis.com/zenn-user-upload/xwvcqfrachasppizhxkzl5svpcui)

（趣味で作ってる別のツールのコミットログです）

## どんな機能があるのか？

`git commit -m` でコミットメッセージの先頭にプレフィックスを付けると、自動でプレフィックスに対応する絵文字が先頭に追加されます。例えば、`feat:` プレフィックスには ✨ が追加されます。

![](https://github.com/negokaz/git-fancy-message-prefix/raw/master/docs/img/commit-with-message.gif)

`-m` オプションなしで `git commit` した場合は次のようにテンプレートが表示されます。
テンプレートのコメントアウトを解除してコミットメッセージを書けば絵文字とプレフィックスの入力が省けます。

![](https://github.com/negokaz/git-fancy-message-prefix/raw/master/docs/img/commit-template.gif)

デフォルトでは次のようなプレフィックスと絵文字が設定されています。

| プレフィックス     | 絵文字            | 説明                                           |
|-------------------|-------------------|-------------------------------------|
| `feat:`     | ✨ (`\U2728`)   | 新機能追加                                        |
| `fix:`      | 🐞 (`\U1f41e`) | バグ修正                                         |
| `doc:`      | 📚 (`\U1f4da`) | ドキュメントのみの変更                                  |
| `style:`    | 💄 (`\U1f484`) | プログラムの動きに影響を与えない変更（インデントの調整やフォーマッタにかけた場合など）|
| `refactor:` | ⛏️ (`\U1f528`) | バグ修正や新機能追加以外のコード修正                           |
| `perf:`     | 🚀 (`\U1f680`) | パフォーマンス改善のためのコード修正                           |
| `test:`     | 🚨 (`\U1f6a8`) | テストの追加や既存テストの修正                              |
| `chore:`    | 👷 (`\U1f477`) | ビルドプロセスやドキュメント生成のような補助ツールやライブラリの変更           |
| `merge:`    | 🔀 (`\U1f500`) | マージコミットにはこの絵文字が付きます                          |

スクリプトの `template` 関数の中を編集すれば、プレフィックスと対応する絵文字のカスタマイズができます。

> ```bash
> function templates {
> # format:
> #
> #   prefix:   emoji(code)   description
> #
> # Full Emoji List: https://unicode.org/emoji/charts/full-emoji-list.html
> cat <<EOF
> feat:     \U2728    新機能追加
> fix:      \U1f41e   バグ修正
> doc:      \U1f4da   ドキュメントのみの変更
> style:    \U1f484   プログラムの動きに影響を与えない変更\n(インデントの調整やフォーマッタにかけた場合など)
> refactor: \U1f528   バグ修正や新機能追加以外のコード修正
> perf:     \U1f680   パフォーマンス改善のためのコード修正
> test:     \U1f6a8   テストの追加や既存テストの修正
> chore:    \U1f477   ビルドプロセスやドキュメント生成のような補助ツールやライブラリの変更
> merge:    \U1f500
> EOF
> # "merge:" is a special prefix to create merge commit message.
> }
> ```
> [git-fancy-message-prefix/prepare-commit-msg.ja](https://github.com/negokaz/git-fancy-message-prefix/blob/08e25f19e261365002bd25f0c578146760e3c825/prepare-commit-msg.ja#L6-L24)

本プロジェクトは MIT ライセンスで公開しているため、自由にコピーしてカスタマイズし、ご自身のチーム内で共有頂けます。

## インストール方法

次のコマンドを git のワーキングディレクトリ（`.git` があるディレクトリ）で実行してください。Windows の場合は Git Bash で実行してください。

```bash
curl https://raw.githubusercontent.com/negokaz/git-fancy-message-prefix/master/prepare-commit-msg.ja -o .git/hooks/prepare-commit-msg && chmod +x .git/hooks/prepare-commit-msg
```

テンプレートが英語のものもあります。詳しくは [こちら](https://github.com/negokaz/git-fancy-message-prefix#install) を参照してください。

# おわりに

コミットメッセージのプレフィックスに絵文字が追加されたことで、プレフィックスを読まなくても直感的にどのようなコミットなのかが把握できるようになりました。コミットログが勝手に賑やかになっていくので、コミットするときになんとなくテンションが上がります。

もし気に入って頂けたら使ってみてください。
