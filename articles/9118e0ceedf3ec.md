---
title: "神経使う運用手作業の負担を Markdown Console で軽減する"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "markdown", "個人開発"]
published: true
---

システム運用で本番環境を触る手作業は、メンバーを心身ともに疲弊させ、ミスを誘います。

- 手順を誤ると障害に直結するプレッシャー
- 使い慣れた設備が揃っていない作業エリア
- （作業内容によっては）夜勤での作業

本番作業をトラブルなく進めるには、有識者による作業内容のレビュー、十分なリハーサル、そして作業者の集中力の維持が重要です。

本番作業の準備と実施を効率化し、ヒューマンエラーによるトラブルを減らすことを目的に、VSCode の拡張として **Markdown Console** を作りました。

https://marketplace.visualstudio.com/items?itemName=negokaz.markdown-console

Markdown Console は下記を実現します。

- Markdown に記述された作業ステップ（コードブロック）をワンクリックで実行でき、作業者の集中力を節約します
- 実行結果（成功/失敗）が可視化され、予期しないエラーが発生したら即座に気付けます
- 手順書は Markdown ファイルのため、コードレビューと同じ感覚でレビューできます
- 手順書に記載された接続先ホスト名などを変数化して差し替えることができるため、異なる環境でのリハーサルを容易に実施できます

![](/images/9118e0ceedf3ec.md/markdown-consoe-demo.gif)

さらに、作業後に役立つ次のような機能も提供します。

- 作業記録を HTML 形式でエクスポートできます

この作業記録からはいつ何が実行され、どのような結果だったかを簡単に確認できます。また、作業の証跡としてだけではなく、他のメンバーが類似作業を実施する際の参考資料としても活用できます。

## 手順書の作り方

どのように手順書を書くのかを簡単にご紹介します。

Markdown Console で実行できる手順書を作成するには `.con.md` という拡張子のファイルを作成します。この手順書は Markdown 形式で文章を記述できます。ディレクトリを右クリックして以下のメニューを選択することでもファイルを作成できます。

![](/images/9118e0ceedf3ec.md/context-menu.png)

SSH を使ってリモートサーバー上でコマンドを実行したい場合は次のように記述します。
```shell
#@cmd:[ssh user1@app1]
echo "Hello, World!"
```

1行目の記述が重要で、`@cmd` にコードブロックを実行するためのコマンドを定義します。`@cmd` よりも前の文字列は無視されるため、必要に応じてコメントアウトすることでシンタックスハイライトが汚れるのを防げます。

このコードブロックを Markdown Console で実行すると、裏では `ssh` コマンドが実行されます。コードブロックの内容はデフォルトでは**コマンドの最後の引数**として指定され、ターミナル上で以下のコマンドを実行したのと同じ意味になります。
```shell
ssh user1@app1 'echo "Hello, World!"'
```

:::message
`@cmd` に指定するコマンドは Markdown Console を実行する端末にインストールされているものでないといけません。必要なものは事前にインストールしておきましょう。
:::

## 手順書の実行

手順書を実行する「Console ビュー」を開くには、`.con.md` ファイルの編集画面から以下のボタンを押下します。

![](/images/9118e0ceedf3ec.md/open-console.png)

Console ビュー上で `Run` ボタンを押下してコードブロックを実行してみると、次のようになります。

![](/images/9118e0ceedf3ec.md/run-hello-world.gif)

コマンドの実行が成功した場合はグリーン、失敗した場合はレッドで表示されます。コマンドの成功・失敗はコマンドの終了ステータスが `0` の場合は成功とみなされ、それ以外では失敗とみなされます。

## `@cmd` に指定できるコマンドの例

コードブロックの `@cmd` に指定するコマンドには任意のものが指定できます。

ローカルで bash スクリプトを実行したり、
```shell
#@cmd:[bash -c]
echo "Hello, World!"
```

Windows 環境で Powershell スクリプトを実行したり、
```powershell
#@cmd:[Powershell -Command]
Write-Output "Hello, World!"
```

SSH 経由で `mysql` コマンドを実行し SQL を実行することもできます。
```sql
--@cmd:[ssh mysql@203.0.113.2 mysql -u user1 -p -e]
SELECT * FROM CUSTOMER;
```

手順書内に記述するユーザー名やホスト名は変数化できます。

```bash
#@cmd:[ssh {{app.user}}@{{app.host}}] 👈
echo "Hello, World!"
```

```yaml:markdown-console.yml
variables:
    app:
        user: user1
        host: app1
```

Markdown Console の設定ファイルである `markdown-console.yml` ファイルを作成すると上記の変数に何を埋め込むかを定義できます。
また、`markdown-console_*.yml`（`*` は任意の文字列）というファイルを作成すると、設定の一部を上書きできます。作業リハーサルで検証環境の接続情報に書き換えれば、本番環境で実施する予定の作業がそっくりそのまま検証環境で実行できるようになります。

## GitHub Copilot との相性

Markdown Console の手順書は、ほぼプレーンな Markdown なので、[GitHub Copilot](https://copilot.github.com/) の支援を受けられます。簡単な手順であれば、次のように GitHub Copilot がコードブロックの内容を自動生成してくれます。

![](/images/9118e0ceedf3ec.md/github-copilot.gif)

コードブロックの `@cmd` もサジェストしてくれるのは非常に便利ですね。

## さいごに

ここでご紹介した以外にも、以下のような機能があります。

- 標準入力からコードブロックの内容を流し込む
- 仮想端末（pty）を有効にして `vim` などのグラフィカルな UI を持つコマンドを実行できるようにする
- コードブロックを実行する際の環境変数を設定する

詳細は[リファレンスガイド](https://github.com/negokaz/vscode-markdown-console/blob/main/docs/ja/reference-guide.md)を確認してください。

もし不具合や機能追加の要望があれば、[Issues](https://github.com/negokaz/vscode-markdown-console/issues) からお知らせください。
