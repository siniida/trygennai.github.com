---
layout: manual_ja
title: クエリのデバッグ / genn.ai
---

# クエリのデバッグ

> ここでは、genn.aiのクエリをデバッグする方法を記載します。

## 環境

クエリのデバッグを行うには[ローカルモード](/ja/config.html#mode.local)で起動したgenn.aiを必要とします。ローカルモードの環境構築については[環境準備](/ja/environment.html)を参照してください。

※ ローカルモードには、MongoDB, ZooKeeper, Kafka, Stormをインストールする必要はありません。

---

## 1. GungnirServerを起動

ローカルモードで起動します。

    $ cd ${GUNGNIR_SERVER_DIR}
    $ ./bin/gungnir-server.sh start conf/gungnir-standalone.yaml

## 2. コンソールで接続

    $ cd ${GUNGNIR_CLIENT_DIR}
    $ ./bin/gungnir -l -u root -p gennai

## 3. Tupleのスキーマ定義

任意のTupleを定義します。詳細に関しては[DDL](/ja/ddl.html)のページを参照してください。

    gungnir> CREATE TUPLE tuple1(id INT, content STRING);
    OK
    gungnir> 

## 4. デバッグ設定

`SET`句を用いて、DEBUGをONにします。

    gungnir> SET debug = on;
    OK
    gungnir> 

## 5. Topologyの投入

入力元に`memory_spout()`、出力に`log()`を指定します。

    gungnir> FROM test USING memory_spout()
          -> EMIT * USING log();
    OK
    gungnir>
    gungnir> SUBMIT TOPOLOGY test;
    OK
    Starting ... Done
    {"id":"63....","name":"test","status":"RUNNING","owner":...}

※ 別途MongoDB, Kafkaを構築している場合には、`kafka_spout()`,`mongo_persist()`等の田の入出力プロセッサでも使用可能です。

## 6. Tupleの投入

手順3で定義したTupleのデータを投入します。

    gungnir> POST tuple1;
    id (INT): 123
    content (STRING): hiraga
    POST tuple1 {"id":123,"content":"hiraga"}
    POST /gungnir/v0.1/e3a272bb16754933a0936b0274ca084a/tuple1/json
    OK
    gungnir>

※ 例では[Interactive Mode](/ja/cli.html#interactive)で投入していますが、JSON形式での投入も可能です。

## 7. ログの確認

`LOG`コマンドで、Tupleが通った経路を確認します。

    gungnir> LOG;
    [test] SPOUT_0 -> PARTITION_1  ( tuple1:[123,"hiraga"] )
    [test] PARTITION_1 -> EMIT_2  ( tuple1:[123,"hiraga"] )
    [test EMIT_2] {"id":123,"content":"hiraga"}

---

## 注意事項

1. `SET debug = on;`を設定すると、セッションが閉じられるまで有効になります。有効である間に起動した全てのTopologyに設定が反映されますので、不要となった場合には`SET debug = off;`で無効にするか、セッションを一度閉じてください。
1. デバッグモード(`SET debug = on;`)で起動したTopologyは、`SET debug = off;`を設定してもデバッグモードを解除できません。Topologyを削除し、`SET debug = off;`を設定した後に再度Topologyを登録してください。
1. Topologyを`STOP TOPOLOGY`->`START TOPOLOGY`した場合、サーバとコンソール間の接続が一度切れてしまう為、Topologyからの応答を拾い漏らしてしまう場合があります。POSTを何度か実行すると再接続されます。
1. `SET debug = on;`および`memory_spout()`, `log()`はパフォーマンスを大きく低下させてしまう為、クエリのデバッグ時以外では使用しないでください。
