---
layout: manual_ja
title: Setting / genn.ai
redirect_from: "/config_ja.html"
---

# genn.ai 設定

> ここでは、genn.aiを構成する各種設定について記載します。

## 設定ファイル

genn.aiで使用している主な設定ファイルは下記の2つです。

* gungnir.yaml (client)
* gungnir.yaml (server)


## 稼働モード <a name="mode" class="anchor"></a>

ここでは、genn.aiが稼働する下記3つのモードについて記載します。

* ローカルモード
* 疑似分散モード
* 完全分散モード

### ローカルモード <a name="mode.local" class="anchor"></a>

GungnirServerとTupleStoreServerが、同一のプロセスで稼働します。また、メタ情報の格納・TupleStoreにはMemoryが使用されます。よって、Storm、Kafka、ZooKeeper、MongoDBを別途構築することなくgenn.aiを実行することが可能です。

![standalone](/img/mode_standalone.png)

下記の制限があります。

* メタ情報がメモリに格納されているため、再起動を行うとメタ情報が失われます。
* `FROM`句では、[Memory Spout Processor](dml.html#MEMORY_SPOUT)のみ使用可能です。

Kafka、MongoDB等を別途構築すると、Topologyにて、Kafka、MongoDB等へデータを書き出したり、データを取得したりすることも可能です。外部入出力には、Kafka、MongoDB、HTTP(Solr, Elasticsearch, ...)を用いることが可能です。

起動には下記コマンドを実行します。

    $ cd $GUNGNIR_INSTALL_DIR
    $ ./bin/gungnir-server.sh start ./conf/gungnir-standalone.yaml

gungnir-standalone.yamlには、genn.aiがローカルモード(standalone)で稼働する為の各種設定が記載されています。

停止には下記コマンドを実行します。

    $ cd $GUNGNIR_INSTALL_DIR
    $ ./bin/gungnir-server.sh stop

ローカルモードのGungnirServerに接続するには、`gungnir`コマンドに`-l`オプションを使用して接続することができます。

    $ ./bin/gungnir -l -u root -p gennai

また、下記設定項目を変更することで、メタ情報の格納・TupleStoreを変更することが可能です。

* [metastore](#s.metastore)
* [metastore.mongodb.servers](#s.metastore.mongodb.servers)
* [persistent.emitter](#s.persistent.emitter)

メタ情報の格納をMongoDB、TupleStoreをKafkaに設定すると、それぞれ揮発を防ぐことができます。

![local](/img/mode_local.png)

### 疑似分散モード <a name="mode.pseudo" class="anchor"></a>

GungnirServerとTupleStoreServerが、それぞれ別プロセスとして稼働します。GungnirServer/TupleStoreServerは1つのホストでのみ実行します。ZooKeeperを必要とします。

![pseudo](/img/mode_pseudo.png)

デフォルト設定では、メタ情報はMongoDBに格納され、TupleStoreにはKafkaが使用されます。よってKafka、MongoDBを構築する必要があります。

また、Topologyにて、Kafka、MongoDB等へデータを書き出したり、データを取得したりすることも可能です。外部入出力には、Kafka、MongoDB、HTTP(Solr, Elasticsearch, ...)を用いることが可能です。

起動には下記コマンドを実行します。

    $ cd $GUNGNIR_INSTALL_DIR
    $ ./bin/gungnir-server.sh start

停止には下記コマンドを実行します。

    $ cd $GUNGNIR_INSTALL_DIR
    $ ./bin/gungnir-server.sh stop


### 完全分散モード <a name="mode.distributed" class="anchor"></a>

GungnirServerとTupleStoreServerが、それぞれ別プロセスとして稼働します。GungnirServer/TupleStoreServerをそれぞれ別のホスト、複数のホスト、同一筐体で複数のプロセスとして稼働させることが可能です。

![distributed](/img/mode_distributed.png)

デフォルト設定では、メタ情報はMongoDBに格納され、TupleStoreにはKafkaが使用されます。また、GungnirServer/TupleStoreServerの情報をgenn.aiクラスタで共有する為にZooKeeperを必要とします。

分散モードに接続するクライアントツール(gungnir, post)は、ZooKeeperアンサンブルから常に稼働中のGungnirServer/TupleStoreServerを知ります。従って、一部のGungnirServer/TupleStoreServerが障害によりクラスタから離脱しても、稼働中のサーバにのみアクセスすることで可用性を備えています。

分散モードで稼働させるには、下記設定項目を初期設定から変更する必要があります。

* [cluster.mode](#s.cluster.mode)
* [cluster.zookeeper.servers](#s.cluster.zookeeper.servers)

また、Topologyにて、Kafka、MongoDB等へデータを書き出したり、データを取得したりすることも可能です。外部入出力には、Kafka、MongoDB、HTTP(Solr, Elasticsearch, ...)を用いることが可能です。

それぞれの起動には下記コマンドを実行します。

    $ cd $GUNGNIR_INSTALL_DIR
    $ ./bin/gungnir-server.sh start
    $ ./bin/tuple-store-server.sh start


それぞれの停止には下記コマンドを実行します。

    $ cd $GUNGNIR_INSTALL_DIR
    $ ./bin/gungnir-server.sh stop
    $ ./bin/tuple-store-server.sh stop

## gungnir.yaml (client)

[クライアントツール](cli.html)(gungnir, post)に関する設定項目です。

### GungnirServerに関する設定

#### gungnir.server.host <a name="c.gungnir.server.host" class="anchor"></a>

[クライアントツール](cli.html)(gungnir, post)がアクセスするGungnirServerのHostを、名前解決が可能なホスト名かIPで指定します。GungnirServerが稼働しているホスト以外からのアクセス時に設定をする必要があります。

この設定値が使用されるのは、GungnirServerが **ローカルモード** で稼働している場合です。 **分散モード**(疑似分散/完全分散)で稼働している場合には、ここで設定した値は使用されず、<a href="#c.cluster.zookeeper.servers">cluster.zookeeper.servers</a>で指定したZooKeeperアンサンブルから接続先のGungnirServerの情報(host/port)を取得します。

> Default: "localhost"

#### gungnir.server.port <a name="c.gungnir.server.port" class="anchor"></a>

[クライアントツール](cli.html)(gungnir, post)がアクセスするGungnirServerのPort番号を指定します。GungnirServerの設定において、<a href="#s.gungnir.server.port">gungnir.server.port</a>を変更している場合に設定をする必要があります。

この設定値が使用されるのは、GungnirServerが **ローカルモード** で稼働している場合です。 **分散モード**(疑似分散/完全分散)で稼働している場合には、ここで設定した値は使用されず、<a href="#c.cluster.zookeeper.servers">cluster.zookeeper.servers</a> で指定したZooKeeperアンサンブルから接続先のGungnirServerの情報(host/port)を取得します。

> Default: 7100

### TupleStoreServerに関する設定

#### tuple.store.server.host <a name="c.tuple.store.server.host" class="anchor"></a>

[クライアントツール](cli.html)(gungnir, post)がアクセスするTupleStoreServerのHostを、名前解決が可能なホスト名かIPで指定します。TupleStoreServerが稼働しているホスト以外からのアクセス時に設定をする必要があります。

この設定値が使用されるのは、TupleStoreServerが **ローカルモード** で稼働している場合です。 **分散モード**(疑似分散/完全分散)で稼働している場合には、ここで設定した値は使用されず、<a href="#c.cluster.zookeeper.servers">cluster.zookeeper.servers</a>で指定したZooKeeperアンサンブルから接続先のTupleStoreServerの情報(host/port)を取得します。

> Default: "localhost"

#### tuple.store.server.port <a name="c.tuple.store.server.port" class="anchor"></a>

[クライアントツール](cli.html)(gungnir, post)がアクセスするTupleStoreServerのPort番号を指定します。GungnirServer/TupleStoreServerの設定において、<a href="#s.tuple.store.server.port">tuple.store.server.port</a>を変更している場合に設定をする必要があります。

この設定値が使用されるのは、TupleStoreServerが **ローカルモード** で稼働している場合です。 **分散モード**(疑似分散/完全分散)で稼働している場合には、ここで設定した値は使用されず、<a href="#c.cluster.zookeeper.servers">cluster.zookeeper.servers</a>で指定したZooKeeperアンサンブルから接続先のTupleStoreServerの情報(host/port)を取得します。

> Default: 7200

### 分散モードに関する設定

#### cluster.mode <a name="c.cluster.mode" class="anchor"></a>

設定可能な値は **local** (ローカルモード)/ **distributed** (分散モード)です。

分散モード(疑似分散/完全分散)を指定している場合、<a href="#c.gungnir.server.host">gungnir.server.host</a>, <a href="#c.gungnir.server.port">gungnir.server.port</a>, <a href="#c.tuple.store.server.host">tuple.store.server.host</a>, <a href="#c.tuple.store.server.port">tuple.store.server.port</a>の各設定値は使用されません。[クライアントツール](cli.html)(gungnir, post)が接続するGungnirServer/TupleStoreServerに関する情報は<a href="#c.cluster.zookeeper.servers">cluster.zookeeper.servers</a>で指定したZooKeeperアンサンブルから取得します。

> Default: "distributed"

#### cluster.zookeeper.servers <a name="c.cluster.zookeeper.servers" class="anchor"></a>

GungnirServer/TupleStoreServerの各設定を保存するZooKeeperアンサンブルを構成するZooKeeperサーバをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。

この設定は、分散モード(疑似分散/完全分散)時にのみ使用されます。<a href="#c.gungnir.server.host">gungnir.server.host</a>, <a href="#c.gungnir.server.port">gungnir.server.port</a>, <a href="#c.tuple.store.server.host">tuple.store.server.host</a>, <a href="#c.tuple.store.server.port">tuple.store.server.port</a>の各設定値は、指定したZooKeeperが構成するアンサンブルから取得して使用されます。

> Default: - "localhost:2181"

> Example:
> 
    cluster.zookeeper.servers:
      - "10.0.1.11:2181"
      - "10.0.1.12:2181"
      - "10.0.1.13:2181"

#### cluster.zookeeper.session.timeout <a name="c.cluster.zookeeper.session.timeout" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルと[クライアントツール](cli.html)(gungnir, post)間のセッションタイムアウトまでの時間をミリ秒で指定します。

デフォルト設定ではStormが使用しているセッションタイムアウト時間と同じ値を使用しています。Stormのデフォルト設定値は20000が設定されています。クライアントツールの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.session.timeout}

#### cluster.zookeeper.connection.timeout <a name="c.cluster.zookeeper.connection.timeout" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルと[クライアントツール](cli.html)(gungnir, post)間の接続時におけるタイムアウト時間をミリ秒で指定します。

デフォルト設定ではStormが使用しているコネクションタイムアウト時間と同じ値を使用しています。Stormのデフォルト設定値は15000が設定されています。クライアントツールの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.connection.timeout}

#### cluster.zookeeper.retry.times <a name="c.cluster.zookeeper.retry.times" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルと[クライアントツール](cli.html)(gungnir, post)間の接続が切れた場合、再接続を試行する回数を指定します。

デフォルト設定ではStormが使用している試行回数と同じ値を使用しています。Stormのデフォルト設定値は5が設定されています。クライアントツールの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.retry.times}

#### cluster.zookeeper.retry.interval <a name="c.cluster.zookeeper.retry.interval" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルと[クライアントツール](cli.html)(gungnir, post)間の接続が切れた場合、再接続を試行する間隔をミリ秒で指定します。

デフォルト設定ではStormが使用している間隔時間と同じ値を使用しています。Stormのデフォルト設定値は1000が設定されています。クライアントツールの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.retry.interval}

### Clientに関する設定

#### gungnir.client.response.timeout <a name="c.gungnir.client.response.timeout" class="anchor"></a>

[クライアントツール](cli.html)(gungnir, post)がGungnirServer/TupleStoreServerからのレスポンスを待つ最大許容時間をミリ秒で指定します。指定時間内にGungnirServer/TupleStoreServerからのレスポンスが無い場合、クライアントツールは処理を中断します。

> Default: 10000

### Monitorに関する設定

#### kafka.monitor.zookeeper.servers <a name="c.kafka.monitor.zookeeper.servers" class="anchor"></a>

Monitor機能を使用する際に、[クライアントツール](cli.html)(gungnir)がアクセスするKafkaクラスタ情報を保持しているZooKeeperアンサンブルを構成するZooKeeperサーバをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。

> Default: - "localhost:2181"

## gungnir.yaml (server)

サーバプロセス(GungnirServer, TupleStoreServer)に関する設定項目です。

### GungnirServerに関する設定

#### gungnir.server.port <a name="s.gungnir.server.port" class="anchor"></a>

GungnirServerが待ち受ける(LISTENする)ポート番号を指定します。

> Default: 7100

#### gungnir.server.pid.file <a name="s.gungnir.server.pid.file" class="anchor"></a>

GungnirServerプロセスのPIDを出力するファイルを指定します。出力先はgungnir-serverのインストールディレクトリ直下です。

> Default: gungnir-server.pid 

#### session.timeout.secs <a name="s.session.timeout.secs" class="anchor"></a>

GungnirServerのセッションタイムアウト時間を秒で指定します。[クライアントツール](cli.html)(gungnir)との接続で指定した時間以上、クライアントツールからの操作が無い場合にセッションはタイムアウトします。

> Default: 3600

#### command.processor.cache.size <a name="s.command.processor.cache.size" class="anchor"></a>

GungnirServerにおいて、CommandProcessorのインスタンスをキャッシュするサイズを指定します。

> Default: 1024

### TupleStoreServerに関する設定

#### tuple.store.server.port <a name="s.tuple.store.server.port" class="anchor"></a>

TupleStoreServerが待ち受ける(LISTENする)ポート番号を指定します。

> Default: 7200

#### tuple.store.server.pid.file <a name="s.tuple.store.server.pid.file" class="anchor"></a>

TupleStoreServerのPIDを出力するファイルを指定します。出力先はgungnir-serverのインストールディレクトリ直下です。

**分散モード**(疑似分散/完全分散)での稼働時のみ設定値が有効になります。

> Default: tuple-store-server.pid

#### tracking.cookie.maxage <a name="s.tracking.cookie.maxage" class="anchor"></a>

TupleStoreServerからクライアントに送られるCOOKIEの残存時間を秒で指定します。

> Default: 864000000

#### persistent.deser.queue.size <a name="s.persistent.deser.queue.size" class="anchor"></a>

TupleStoreServerがクライアントからデータを受領後、Kafkaへ書き込みを行う際にシリアライズを行うまでの待機キューのサイズを指定します。シリアライズが遅延している場合、<a href="#s.persistent.deser.parallelism">persistent.deser.parallelism</a>と共に調整を行ってください。

> Default: 1024

#### persistent.deser.parallelism <a name="s.persistent.deser.parallelism" class="anchor"></a>

TupleStoreServerがクライアントから受領したデータをシリアライズする並列度を指定します。シリアライズが遅延している場合、<a href="#s.persistent.deser.queue.size">persistent.deser.queue.size</a>と共に調整を行ってください。

> Default: 32

#### persistent.deserializer <a name="s.persistent.deserializer" class="anchor"></a>

TupleStoreServerがシリアライズする処理クラスを指定します。現時点では、使用可能なクラスは他にありません。

> Default: org.gennai.gungnir.tuple.persistent.JsonPersistentDeserializer

#### persistent.emitter.queue.size <a name="s.persistent.emitter.queue.size" class="anchor"></a>

シリアライズされたTupleをKafkaに書き込む際の待機キューのサイズを指定します。Kafkaへの書き込みが遅延している場合、<a href="#s.persistent.emitter.parallelism">persistent.emitter.parallelism</a>、Kafkaのクラスタ設定と共に調整してください。

> Default: 1024

#### persistent.emitter.parallelism <a name="s.persistent.emitter.parallelism" class="anchor"></a>

シリアライズされたTupleをKafkaに書き込む際の並列度を指定します。Kafkaへの書き込みが遅延している場合、<a href="#s.persistent.emitter.queue.size">persistent.emitter.queue.size</a>、Kafkaのクラスタ設定と共に調整してください。

> Default: 32

#### persistent.emit.tuples.max <a name="s.persistent.emit.tuples.max" class="anchor"></a>

シリアライズされたTupleをKafkaに書き込む際、一度に書き込むTuple数の最大値を指定します。この設定値は、<a href="#s.persistent.emitter.queue.size">persistent.emitter.queue.size</a>でサイズを指定したキューにTupleが複数滞留している場合に使用されます。通常は、キューにTupleが1つでも存在すれば、Kafkaへの書き込み処理が実行されます。

設定値より少ないTuple数でも、Tupleの合計サイズが<a href="#s.persistent.emit.tuples.max.size">persistent.emit.tuples.max.size</a>で設定された値よりも大きくなった場合、Kafkaへの書き込みが実行されます。

> Default: 8

#### persistent.emit.tuples.max.size <a name="s.persistent.emit.tuples.max.size" class="anchor"></a>

シリアライズされたTupleをKafkaに書き込む際、一度に書き込むTupleの合計サイズの最大値を設定します。この設定値は、<a href="#s.persistent.emitter.queue.size">persistent.emitter.queue.size</a>でサイズを指定したキューにTupleが複数滞留している場合に使用されます。通常は、キューにTupleが1つでも存在すれば、Kafkaへの書き込み処理が実行されます。

設定値より小さな合計サイズでも、Tuple数が<a href="#s.persistent.emit.tuples.max">persistent.emit.tuples.max</a>で設定された値に達している場合、Kafkaへの書き込みが実行されます。

> Default: 1024

#### persistent.emitter <a name="s.persistent.emitter" class="anchor"></a>

TupleStoreServerからTopologyへ、Tupleを送信する処理を行うクラスを指定します。デフォルト設定では、TupleStoreServerはKafkaにTupleを書き込みます。Kafkaに書き込みを行うことで、Tupleが一定期間保存されることになります。

gungnir-standalone.yamlを用いたローカルモードで起動すると、<a href="#s.persistent.emitter">persistent.emitter</a>には **InMemoryEmitter** が適用され、Kafkaを起動することなく動作の確認を行えます。ただし<a href="#s.cluster.mode">cluster.mode</a>、<a href="#s.storm.cluster.mode">storm.cluster.mode</a>が共に **local** の場合にのみ使用することができます。

> Default: org.gennai.gungnir.tuple.persistent.KafkaPersistentEmitter

> Example: gungnir-standalone.yamlでの設定
org.gennai.gungnir.tuple.persistent.InMemoryEmitter

### Clusterに関する設定

#### cluster.mode <a name="s.cluster.mode" class="anchor"></a>

GungnirServer/TupleStoreServerの起動モードを指定します。設定可能な値は **local** (ローカルモード)/ **distributed** (分散モード)です。

**ローカルモード** を指定した場合、TupleStoreServerはGungnirServerと同じプロセス内で稼働します。 **分散モード**(疑似分散/完全分散)を指定した場合、TupleStoreServerはGungnirServerと別プロセスで稼働する為、`gungnir-server.sh`とは別に`tuple-store-server.sh`で起動する必要があります。

> Default: "local"

#### cluster.zookeeper.servers <a name="s.cluster.zookeeper.servers" class="anchor"></a>

GungnirServer/TupleStoreServerの各設定を保持しているZooKeeperアンサンブルを構成するZooKeeperサーバをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。分散モード(疑似分散/完全分散)時にのみ使用されます。

> Default: - "localhost:2181"

#### cluster.zookeeper.session.timeout <a name="s.cluster.zookeeper.session.timeout" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルとサーバプロセス(GungnirServer, TupleStoreServer)間のセッションタイムアウトを行う時間をミリ秒で指定します。

デフォルト設定ではStormが使用しているセッションタイムアウト時間と同じ値を使用しています。Stormのデフォルト設定値は20000が設定されています。サーバプロセスの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.session.timeout}

#### cluster.zookeeper.connection.timeout <a name="s.cluster.zookeeper.connection.timeout" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルとサーバプロセス(GungnirServer, TupleStoreServer)間の接続時におけるタイムアウト時間をミリ秒で指定します。

デフォルト設定ではStormが使用しているコネクションタイムアウト時間と同じ値を使用しています。Stormのデフォルト設定値は15000が設定されています。サーバプロセスの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.connection.timeout}

#### cluster.zookeeper.retry.times <a name="s.cluster.zookeeper.retry.times" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルとサーバプロセス(GungnirServer, TupleStoreServer)間の接続が切れた場合、再接続を試行する回数を指定します。

デフォルト設定ではStormが使用している試行回数と同じ値を使用しています。Stormのデフォルト設定値は5が設定されています。サーバプロセスの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.retry.times}

#### cluster.zookeeper.retry.interval <a name="s.cluster.zookeeper.retry.interval" class="anchor"></a>

分散モード(疑似分散/完全分散)において、ZooKeeperアンサンブルとサーバプロセス(GungnirServer, TupleStoreServer)間の接続が切れた場合、再接続を試行する間隔をミリ秒で指定します。

デフォルト設定ではStormが使用している間隔時間と同じ値を使用しています。Stormのデフォルト設定値は1000が設定されています。サーバプロセスの設定値のみ変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更をしてください。

> Default: ${storm.zookeeper.retry.interval}

### 管理サーバに関する設定

#### gungnir.admin.server.port <a name="s.gungnir.admin.server.port" class="anchor"></a>

管理サーバが待ち受ける(LISTENする)ポート番号を指定します。この設定を有効にした場合にのみ管理サーバが起動されます。

> Default: 9192

ブラウザもしくはcurl等で、下記のようにサーバプロセスに関するレポートを取得する事ができます。

> Example:
    http://127.0.0.1:9192/stats.txt

取得したレポートでは下記項目等を確認する事ができます。

* リクエスト数
* リクエストのレイテンシ


#### gungnir.admin.server.backlog <a name="s.gungnir.admin.server.backlog" class="anchor"></a>

クライアントからの接続が、HTTPサーバ(管理サーバ用)に受け入れられるのを待機するキューに入れるTCP接続の最大数を指定します。

> Default: 100

### Stormに関する設定

#### storm.cluster.mode <a name="s.storm.cluster.mode" class="anchor"></a>

Stromの稼働モードを指定します。指定可能な値は **local** (Stormのworkerを、GungnirServerのプロセス内にスレッドで起動する)もしくは、 **distributed** (稼働しているStormクラスタを利用する)です。

> Default: "local"

#### storm.nimbus.host <a name="s.storm.nimbus.host" class="anchor"></a>

稼働しているStormクラスタのNimbusホストを、名前解決ができるホスト名もしくはIPで指定します。<a href="#s.storm.cluster.mode">storm.cluster.mode</a>が **distributed** である場合にのみ使用されます。

> Default: "localhost"

### Topologyに関する設定

#### topology.workers <a name="s.topology.workers" class="anchor"></a>

投入するTopologyが稼働するStormのworker数のデフォルト値を指定します。`SET`文を使用することで、各Topology毎に変更することが可能です。

> Default: 1

#### topology.status.check.times <a name="s.topology.status.check.times" class="anchor"></a>

投入されたTopologyの起動もしくは停止時に、Topologyの状態を確認する回数の最大値を指定します。GungnirServerは、この設定値と<a href="#s.topology.status.check.interval">topology.status.check.interval</a>の設定値を乗じた時間内に、Topologyの状態変更を検知できない場合、処理をロールバックします。

> Default: 20

#### topology.status.check.interval <a name="s.topology.status.check.interval" class="anchor"></a>

投入されたTopologyの起動もしくは停止時に、Topologyの状態を確認する間隔をミリ秒で指定します。GungnirServerは、この設定値と<a href="#s.topology.status.check.times">topology.status.check.times</a>の設定値を乗じた時間内に、Topologyの状態変更を検知できない場合、処理をロールバックします。

> Default: 2000

#### default.parallelism <a name="s.default.parallelism" class="anchor"></a>

Operatorの並列度のデフォルト値を指定します。クエリに`parallelism`句を記述している場合、クエリに記述された値が使用されます。クエリで`parallelism`句が記述されていないOperatorには、設定値が適用されます。この設定項目は`SET`文でTopology毎に設定する事が可能です。

> Default: 1

#### gungnir.local.dir <a name="s.gungnir.local.dir" class="anchor"></a>

<a href="/ja/dml.html#TupleJoin">複数のTupleをJOINする</a>際に、一時的にTupleをファイルに格納することができます。格納しておくファイルは指定したディレクトリに作成されます。`${gungnir.home}`からの相対パスとなります。

> Default: "gungnir-local"

### Metastoreに関する設定

#### metastore <a name="s.metastore" class="anchor"></a>

GungnirServerのメタ情報を格納するのに使用するクラスを指定します。

設定可能な値は、デフォルト設定の **MongoDbMetaStore** と **InMemoryMetaStore** です。 **InMemoryMetaStore** はGungnirServerが起動している間のみ利用可能なMetastoreです。GungnirServerを再起動すると各メタ情報は消失します。メタ情報を永続的にするには **MongoDbMetaStore** を使用してください。 **MongoDbMetaStore** を使用する場合、<a href="#s.metastore.mongodb.servers">metastore.mongodb.servers</a>と共に指定する必要があります。

> Default: org.gennai.gungnir.metastore.MongoDbMetaStore

#### metastore.mongodb.servers <a name="s.metastore.mongodb.servers" class="anchor"></a>

メタ情報の格納に **MongoDbMetaStore** を使用する場合に、接続先のMongoDBのホストをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。

> Default: - "localhost:27017"

### TupleStoreに関する設定

#### kafka.brokers <a name="s.kafka.brokers" class="anchor"></a>

TupleStoreに使用するKafkaのBrokerをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。

> Default: - "localhost:9092"

#### kafka.required.acks <a name="s.kafka.required.acks" class="anchor"></a>

TupleStoreServerがKafkaにTupleを書き込む際に、どの時点で応答を返すかを設定します。設定可能な値は 0, 1, -1 です。
詳細は[Kafkaのドキュメント](http://kafka.apache.org/documentation.html)を参照してください。

> Default: 1

#### kafka.producer.type <a name="s.kafka.producer.type" class="anchor"></a>

TupleStoreServerがKafkaにTupleを書き込む際の処理を指定します。同期(sync)/非同期(async)を指定できます。

> Default: "sync"

#### kafka.auto.commit.interval <a name="s.kafka.auto.commit.interval" class="anchor"></a>

TupleStoreServerがKafkaに書き込みを行う際に、AutoCommitを実行する間隔をミリ秒で指定します。

> Default: 10000

#### kafka.zookeeper.servers <a name="s.kafka.zookeeper.servers" class="anchor"></a>

TupleStoreServerが書き込むKafkaのクラスタ情報を保持するZooKeeperアンサンブルを構成するZooKeeperサーバをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。

> Default: - "localhost:2181"

#### kafka.zookeeper.session.timeout <a name="s.kafka.zookeeper.session.timeout" class="anchor"></a>

TupleStoreServerが、Kafkaのクラスタ情報を保持するZooKeeperアンサンブルとのセッションを、タイムアウトする時間をミリ秒で指定します。

> Default: 6000

#### kafka.zookeeper.connection.timeout <a name="s.kafka.zookeeper.connection.timeout" class="anchor"></a>

TupleStoreServerが、Kafkaのクラスタ情報を保持するZooKeeperアンサンブルに接続を試みる際のタイムアウト時間をミリ秒で指定します。

デフォルト設定ではGungnirServerがStormのクラスタ情報を保持するZooKeeperアンサンブルとの設定と同じ値を使用します。Stormのデフォルト設定値は15000が設定されています。サーバプロセスの設定値のみを変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更してください。

> Default: ${storm.zookeeper.connection.timeout}

#### kafka.zookeeper.retry.times <a name="s.kafka.zookeeper.retry.times" class="anchor"></a>

Kafkaクラスタの情報を保持するZooKeeperアンサンブル間とのセッションが切断された場合に、再接続を試行する回数を指定します。

デフォルト設定ではStormが使用している試行回数と同じ値を使用しています。Stormのデフォルト設定値は5が設定されています。サーバプロセスの設定値のみを変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更してください。

> Default: ${storm.zookeeper.retry.times}

#### kafka.zookeeper.retry.interval <a name="s.kafka.zookeeper.retry.interval" class="anchor"></a>

Kafkaクラスタの情報を保持するZooKeeperアンサンブル間とのセッションが切断された場合に、再接続を試行する間隔をミリ秒で指定します。

デフォルト設定ではStormが使用している間隔時間と同じ値を使用しています。Stormのデフォルト設定値は1000が設定されています。サーバプロセスの設定値のみを変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更してください。

> Default: ${storm.zookeeper.retry.interval}

### Monitorに関する設定

#### monitor.enabled <a name="s.monitor.enabled" class="anchor"></a>

Monitor機能を使用可能とするかを指定します。

> Default: false

#### kafka.monitor.brokers <a name="s.kafka.monitor.brokers" class="anchor"></a>

Monitor機能に使用するKafkaクラスタのBrokerをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。

> Default: - "localhost:9092"

#### kafka.monitor.required.acks <a name="s.kafka.monitor.required.acks" class="anchor"></a>

Monitor機能を使用する際に、KafkaクラスタからAckをどのタイミングで受け取るかを設定します。詳細に関しては[Kafkaのドキュメント](http://kafka.apache.org/documentation.html)を参照してください。

> Default: 1

#### kafka.monitor.auto.commit.interval <a name="s.kafka.monitor.auto.commit.interval" class="anchor"></a>

Monitor機能において、KafkaのAutoCommitを実行する間隔をミリ秒で指定します。

> Default: 10000

#### kafka.monitor.zookeeper.servers <a name="s.kafka.monitor.zookeeper.servers" class="anchor"></a>

Monitor機能を使用する際にアクセスするKafkaクラスタの情報を保持するZooKeeperアンサンブルをリスト形式で指定します。

> Default: - "localhost:2181"

#### kafka.monitor.zookeeper.session.timeout <a name="s.kafka.monitor.zookeeper.session.timeout" class="anchor"></a>

Monitor機能を使用する際に、Kafkaクラスタの情報を保持するZooKeeperアンサンブルとの接続に適用されるセッションタイムアウトをミリ秒で指定します。

デフォルト設定ではStormが使用しているセッションタイムアウト時間と同じ値を使用しています。Stormのデフォルト設定値は20000が設定されています。Monitor機能使用時の設定値のみを変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更してください。

> Default: ${storm.zookeeper.session.timeout}

#### kafka.monitor.zookeeper.connection.timeout <a name="s.kafka.monitor.zookeeper.connection.timeout" class="anchor"></a>

Monitor機能を使用する際に、アクセスするKafkaクラスタの情報を保持するZooKeeperアンサンブルとの接続におけるタイムアウト時間をミリ秒で指定します。

デフォルト設定ではStormが使用しているコネクションタイムアウト時間と同じ値を使用しています。Stormのデフォルト設定値は15000が設定されています。Monitor機能使用時の設定値のみを変更したい場合は、この設定項目を変更してください。Stormの設定値と共に変更したい場合は、Stormをインストールしたディレクトリ配下にあるconf/storm.yamlにて変更してください。

> Default: ${storm.zookeeper.connection.timeout}

#### kafka.monitor.zookeeper.sync.timeout <a name="s.kafka.monitor.zookeeper.sync.timeout" class="anchor"></a>

Monitor機能を使用する際に、アクセスするKafkaクラスタの情報を保持するZooKeeperアンサンブルとの接続に適用される同期処理時間のタイムアウトをミリ秒で指定します。

> Default: 2000

### Operatorに関する設定

#### spout.operator.queue.size <a name="s.spout.operator.queue.size" class="anchor"></a>

`SPOUT`オペレータが内部に保持するキューのサイズを指定します。

> Default: 1024

#### emit.operator.queue.size <a name="s.emit.operator.queue.size" class="anchor"></a>

`EMIT`オペレータが内部に保持するキューのサイズを指定します。

> Default: 1024

#### emit.operator.emit.tuples.max <a name="s.emit.operator.emit.tuples.max" class="anchor"></a>

`EMIT`オペレータが保持するキューに同時に複数のタプルが到着している場合、一度に書き出すタプルの最大数を指定します。

> Default: 8

#### tuplejoin.seek.size <a name="s.tuplejoin.seek.size" class="anchor"></a>

TupleJoin時に、メモリもしくはファイル上に一時的に保存している待ち受けTupleの保存期間が、設定時間を経過しているかの確認をTupleが到着する度に実行します。その際に一度に確認するTuple数を指定します。

> Default: 8

### Processorに関する設定

#### kafka.spout.fetch.size <a name="s.kafka.spout.fetch.size" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>が、Kafkaから一度に取得するデータのサイズを指定します。

> Default: 1048576

#### kafka.spout.fetch.interval <a name="s.kafka.spout.fetch.interval" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>が、Kafkaからデータを取得する間隔をミリ秒で指定します。

> Default: 1000

#### kafka.spout.offset.behind.max <a name="s.kafka.spout.offset.behind.max" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>において、オフセットが書き込まれない最大の許容時間を指定します。

> Default: 9223372036854775807

#### kafka.spout.state.update.interval <a name="s.kafka.spout.state.update.interval" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>が、状態を更新する間隔をミリ秒で指定します。

> Default: 2000

#### kafka.spout.topic.replication.factor <a name="s.kafka.spout.topic.replication.factor" class="anchor"></a>

Topologyの投入時に、<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>がKafkaに作成するトピックのレプリケーション数を指定します。

> Default: 1

#### kafka.spout.read.brokers.retry.times <a name="s.kafka.spout.read.brokers.retry.times" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>がKafkaからデータを取得する際に、担当するパーティションのリーダー情報を取得するのに試行する回数を指定します。

> Default: 5

#### kafka.spout.read.brokers.retry.interval <a name="s.kafka.spout.read.brokers.retry.interval" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>がKafkaからデータを取得する際に、担当するパーティションのリーダー情報を取得するのに試行する間隔をミリ秒で指定します。

> Default: 1000

#### kafka.spout.partition.operation.retry.times <a name="s.kafka.spout.partition.operation.retry.times" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>がKafkaからデータを取得する時、Kafkaクラスタの状態によってパーティション情報が変更されている場合があります。その際に、<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>にて再度Kafkaのパーティション情報を取得する試行回数を指定します。

> Default: 5

#### kafka.spout.partition.operation.retry.interval <a name="s.kafka.spout.partition.operation.retry.interval" class="anchor"></a>

<a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>がKafkaからデータを取得する時、Kafkaクラスタの状態によってパーティション情報が変更されている場合があります。その際に、 <a href="/ja/dml.html#KAFKA_SPOUT">Kafka Spout Processor</a>にて再度Kafkaのパーティション情報を取得する試行の間隔時間をミリ秒で指定します。

> Default: 1000

#### mongo.fetch.servers <a name="s.mongo.fetch.servers" class="anchor"></a>

`JOIN`句において<a href="/ja/dml.html#MONGO_FETCH">Mongo Fetch Processor</a>を用いる際に、結合するデータの読み込み元となるMongoDBサーバをリスト形式で指定します。`[host|IP]:[port]`の書式で指定します。

> Default: - "localhost:27017"

#### mongo.fetch.cache.size <a name="s.mongo.fetch.cache.size" class="anchor"></a>

`JOIN`句において<a href="/ja/dml.html#MONGO_FETCH">Mongo Fetch Processor</a>使用時の、キャッシュを保持する件数の最大値を指定します。

> Default: 1024

#### web.fetch.cache.size <a name="s.web.fetch.cache.size" class="anchor"></a>

`JOIN`句において<a href="/ja/dml.html#WEB_FETCH">Web Fetch Processor</a>使用時の、キャッシュを保持する件数の最大値を指定します。

> Default: 1024

#### jdbc.fetch.driver <a name="s.jdbc.fetch.driver" class="anchor"></a>

`JOIN`句において<a href="/ja/dml.html#JDBC_FETCH">JDBC Fetch Processor</a>を用いる際に、接続先となるRDBに応じたドライバーを指定します。

> Default: "com.mysql.jdbc.Driver"

MySQL以外に接続をする場合には、ドライバーを組み込む必要があります。

#### jdbc.fetch.cache.size <a name="s.jdbc.fetch.cache.size" class="anchor"></a>

`JOIN`句において<a href="/ja/dml.html#JDBC_FETCH">JDBC Fetch Processor</a>使用時の、キャッシュを保持する件数の最大値を指定します。

> Default: 1024

#### kafka.emit.brokers <a name="s.kafka.emit.brokers" class="anchor"></a>

`EMIT`句において<a href="/ja/dml.html#KAFKA_EMIT">Kafka Emit Processor</a>を用いた際に、出力先となるKafkaクラスタのBrokerをリスト形式で指定します。

> Default: - "localhost:9092"

#### kafka.emit.required.acks <a name="s.kafka.emit.required.acks" class="anchor"></a>

`EMIT`句において<a href="/ja/dml.html#KAFKA_EMIT">Kafka Emit Processor</a>を用いる際に、出力先となるKafkaクラスタからAckをどのタイミングで受け取るかを指定します。詳細に関しては[Kafkaのドキュメント](http://kafka.apache.org/documentation.html)を参照してください。

> Default: 1

#### mongo.persist.servers <a name="s.mongo.persist.servers" class="anchor"></a>

`EMIT`句において<a href="/ja/dml.html#MONGO_PERSIST">Mongo Persist Processor</a> を用いる際に、出力先となるMongoDBサーバをリスト形式で指定します。

> Default: - "localhost:27017"

#### jdbc.persist.driver <a name="s.jdbc.persist.driver" class="anchor"></a>

`EMIT`句において<a href="/ja/dml.html#JDBC_PERSIST">JDBC Persist Processor</a>を用いる際に、出力先となるRDBに応じたドライバーを指定します。

> Default: "com.mysql.jdbc.Driver"

MySQL以外に接続をする場合には、ドライバーを組み込む必要があります。

### Metricsに関する設定

#### metrics.reporters <a name="s.metrics.reporters" class="anchor"></a>

GungnirServer/TupleStoreServerにおけるメトリクス情報を取得するクラス、取得間隔秒、出力先を指定します。デフォルトは、メトリクス情報を取得しません。

> Example:
>
    - reporter: org.gennai.gungnir.metrics.CsvMetricsReporter
      interval.secs: 60
      output.dir: ${gungnir.home}/logs

`reporter`に設定可能なクラスは下記の通りです。

* org.gennai.gungnir.metrics.CsvMetricsReporter
* org.gennai.gungnir.metrics.ConsoleMetricsReporter
* org.gennai.gungnir.metrics.StatsdMetricsReporter

詳細は<a href="/ja/metrics.html">メトリクス設定</a>の項目を参照してください。

#### topology.metrics.enabled <a name="s.topology.metrics.enabled" class="anchor"></a>

StormのTopologyに関するメトリクス情報を取得するかを指定します。

> Default: false

#### topology.metrics.consumer <a name="s.topology.metrics.consumer" class="anchor"></a>

StormのTopologyに関するメトリクス情報を取得するクラスを指定します。

> Default: backtype.storm.metric.LoggingMetricsConsumer

現在指定可能な設定は下記の通りです。

* backtype.storm.metrics.LoggingMetricsConsumer
* org.gennai.gungnir.metrics.StatsdMetricsConsumer

詳細は<a href="/ja/metrics.html">メトリクス設定</a>の項目を参照してください。

#### topology.metrics.consumer.parallelism <a name="s.topology.metrics.consumer.parallelism" class="anchor"></a>

StormのTopologyに関するメトリクス情報を取得する実行処理の並列度を指定します。

> Default: 1

#### topology.metrics.interval.secs <a name="s.topology.metrics.interval.secs" class="anchor"></a>

StormのTopologyに関するメトリクス情報を取得する間隔を秒で指定します。

> Default: 60

#### metrics.statsd.host <a name="s.metrics.statsd.host" class="anchor"></a>

メトリクス情報を送信するStatsDが稼動しているホストを指定します。

> Default: "localhost"

#### metrics.statsd.port <a name="s.metrics.statsd.port" class="anchor"></a>

メトリクス情報を送信するStatsDが稼動しているポート番号を指定します。

> Default: 8125

#### metrics.consumer.prefix <a name="s.metrics.consumer.prefix" class="anchor"></a>

メトリクス情報を外部に送信する際に、ConsumerMetricsに付与するプレフィックスを指定します。

> Default: "storm.metrics"

#### metrics.reporter.prefix <a name="s.metrics.reporter.prefix" class="anchor"></a>

メトリクス情報を外部に送信する際に、ReporterMetricsに付与するプレフィックスを指定します。

> Default: "tuplestore"
