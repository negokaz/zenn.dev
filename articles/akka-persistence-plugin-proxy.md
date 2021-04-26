---
title: "インメモリデータストアを使って Akka Cluster + Akka Persistence のアプリを自動テストする"
emoji: "☄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["akka"]
published: true
---

# はじめに

Akka Cluster を使ってアプリケーションを実装した際、複数ノードでクラスタを構成したときの振る舞いは **[Multi Node Testkit](https://doc.akka.io/docs/akka/current/multi-node-testing.html)** を使ってテストできます。Multi Node Testkit を使うと複数の JVM が起動し、実際にサーバ上でクラスタを構成したのと同じような状況をローカル環境に作りアプリケーションの振る舞いを自動テストできます。

この記事では Akka Cluster と Akka Persistence を組み合わせて実装されたアプリケーションの永続化データをインメモリデータストアに保存することで自動テストを高速化しつつ、Multi Node Testkit を組み合わせた際に起きる、ノード間でデータが共有できない問題に対処する方法を解説します。

# Akka Persistence の永続化にインメモリのデータストアを使う

Akka Persistence の Actor をテストする場合は、イベントやスナップショットといった永続化データをメモリへ書き込むようにすると便利です。

- テスト終了時にメモリへ保存されたデータが自然と消えるためデータクリーニングが不要
- メモリへの書き込みはディスクよりも高速なためテストの実行時間を短縮できる

Akka Persistence は設定ファイルから **Persistence Plugin** を差し替えることでコードを書き換えることなく永続化データの保存先を変更できます。

**akka-persistence-inmemory** という Persistence Plugin を用いることで永続化データをインメモリに保存できます。

https://github.com/dnvriend/akka-persistence-inmemory

`build.sbt` に依存を追加し（上記リンク参照）、テストコードの `applicaiton.conf` で Persistence Plugin を次のように設定します。

```
akka.persistence {
  journal.plugin = "inmemory-journal"
  snapshot-store.plugin = "inmemory-snapshot-store"
}
```

ちなみに、Akka 標準の `akka.persistence.journal.inmem` という Journal Plugin を用いることでもメモリにイベントを書き出せます。ただし、スナップショットをメモリに書き出す機能が標準では提供されないため、前述の Journal Plugin を利用するのがオススメです。

# Multi Node Testkit で akka-persistence-inmemory を使う

Multi Node Testkit で akka-persistence-inmemory を使うと、各クラスタメンバー間で永続化されたイベントやスナップショットを共有できないという問題があります。これは Multi Node Testkit がクラスタのノードを別々の JVM プロセスで起動するためです（プロセス単位でメモリ領域は隔離される）。この問題があると、ノードが 1 台クラッシュしても Akka Persistence で実装した Actor が別のノードで復元されるか、といったことをテストしたい場合に困ります。

このような場合、**[Persistence Plugin Proxy](https://doc.akka.io/docs/akka/2.6/persistence-plugins.html#persistence-plugin-proxy)** を使うとネットワークを経由して特定のノードに永続化データの保存を代理してもらえます。つまり、永続化データの保存先に指定したノードが停止しない限り、全ノードで永続化されたイベントやスナップショットを共有できます。

Persistence Plugin Proxy は Akka Persistence の標準機能として提供されているため、新たな依存ライブラリを追加する必要はありません。

Persistence Plugin Proxy を有効にするには、`application.conf` で Persistence Plugin の設定を次のように変更します。

```
akka.persistence {
  journal.plugin = "akka.persistence.journal.proxy"
  snapshot-store.plugin = "akka.persistence.snapshot-store.proxy"

  journal.proxy {
    start-target-journal = on
    target-journal-plugin = "inmemory-journal"
  }
  snapshot-store.proxy {
    start-target-snapshot-store = on
    target-snapshot-store-plugin = "inmemory-snapshot-store"
  }
}
```

使いたい Journal Plugin を `journal.proxy.target-journal-plugin` に設定し、`journal.plugin` には `"akka.persistence.journal.proxy"` を設定し、スナップショットストアも同様に設定します（journal とは設定するキーが異なるので注意）。

そして、テストコードの初期化時に Persistence Plugin Proxy のターゲットを 1 番目のノードへ向けるよう実装します。

```scala
import akka.persistence.journal.PersistencePluginProxy
// ... 略 ...

object ExampleServiceSpecConfig extends MultiNodeConfig {
  commonConfig(ConfigFactory.load())
  val node1: RoleName = role("node1")
  // ... 略 ...
}

class ExampleServiceSpec extends MultiNodeSpec(ExampleServiceSpecConfig) {
  import ExampleServiceSpecConfig._

  // 1番目のノードは Multi Node Testkit で特別な役割を持っておりシャットダウンできない（するとまずい）ため、
  // 1番目のノードにイベントとスナップショットを永続化するようにしておく
  PersistencePluginProxy.setTargetLocation(system, node(node1).address)

  // ... 略 ...

}
```

1 番目のノードに向ける理由は、テストが動いている間は常に 1 番目のノードが起動していることが期待できるため、イベントやスナップショットの保存先として無難だからです。
なぜ 1 番目のノードをそのようにみなせるかというと Multi Node Testkit において 1 番目のノードは特別な役割を持っており、停止してしまうとテストを正しく継続できないからです。
公式ドキュメントには次のように注意書きが書かれています。

> Don’t issue a shutdown of the first node. The first node is the controller and if it shuts down your test will break.
> **[Multi Node Testing • Akka Documentation](https://doc.akka.io/docs/akka/2.6/multi-node-testing.html#things-to-keep-in-mind)**

次のサンプルプロジェクトの PR から、実際に動かせるコードで Persistence Plugin Proxy を有効にする例を確認できます。

https://github.com/negokaz/sample-akka-inmem-persistence-plugin-proxy/pull/1

実際にこのサンプルプロジェクトのテストを実行する方法は [README](https://github.com/negokaz/sample-akka-inmem-persistence-plugin-proxy) を参照してください。

# 参考資料

- [Persistence Plugins • Akka Documentation](https://doc.akka.io/docs/akka/current/persistence-plugins.html)
