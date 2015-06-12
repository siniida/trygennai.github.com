---
layout: manual_ja
title: メトリクス設定 / genn.ai
---

## メトリクス基本情報

gennaiでは下記2種類のメトリクス情報を取得することができます。

* Stormにおける処理Tupleに関するメトリクス(<a href="#consumer">ConsumerMetrics</a>)
* TupleStoreServerにおけるTupleに関するメトリクス(<a href="#reporter">ReporterMetrics</a>)

以降では、これらのメトリクスについて記載します。

### ConsumerMetrics <a name="consumer" class="anchor"></a>

Stormにおける処理タプル数等を取得することができます。

取得可能な数値は主に下記の項目があります。

* Component毎の送信キューの状態
* Component毎の受信キューの状態
* Component毎のTuple転送数
* Component毎の失敗Tuple数
* workerの起動時間
* workerが使用するメモリの使用状況

これらのメトリクスの値は、`topology.stats.sample.rate`といったStormの設定に依存します。

### ReporterMetrics <a name="reporter" class="anchor"></a>

TupleStoreServerにおける処理タプル数等を取得することができます。

取得可能な数値は主に下記の項目があります。

* TupleStoreServerが受領したTuple数の総数
* Deser待機キューにおける各種値
* Deser処理に掛かった時間
* Emit待機キューにおける各種値
* EMIT処理に掛かった時間

----

## メトリクス設定

### ConsumerMetrics 設定

Consumer Metrics の取得に使用可能な設定は下記の項目があります。

* LoggingMetricsConsumer
* StatsdMetricsConsumer

#### LoggingMetricsConsumer

Stormの各Supervisorが稼動しているホストの`${STORM_HOME}/logs/metrics.log`に出力されます。

`${GUNGNIR_HOME}/conf/gungnir.yaml`にて下記設定をします。

    topology.metrics.enabled: true
    topology.metrics.consumer: backtype.storm.metrics.LoggingMetricsConsumer
    topology.metrics.consumer.parallelism: 1
    topology.metrics.interval.secs: 60

#### StatsdMetricsConsumer

別途稼動させているStatsDサーバにメトリクス情報を送信します。

`${GUNGNIR_HOME}/conf/gungnir.yaml`にて下記設定をします。

    topology.metrics.enabled: true
    topology.metrics.consumer: org.gennai.gungnir.metrics.StatsdMetricsConsumer
    topology.metrics.consumer.parallelism: 1
    topology.metrics.interval.secs: 60
    metrics.statsd.host: "[StatsD host]"
    metrics.statsd.port: 8125
    metrics.consumer.prefix: "consumer"

### ReporterMetrics 設定

ReporterMetrics の取得に使用可能な設定は下記の項目があります。

* CsvMetricsReporter
* StatsdMetricsReporter

※ ReporterMetricsは複数を同時に使用することが可能です。

#### CsvMetricsReporter

CSVファイル形式で出力します。TupleStoreServerを複数のホストで稼動させている場合、メトリクス情報が各ホストに分散されます。出力されるファイルは下記の種類があります。

* persistent-deser-queue-size.[uid].csv
* persistent-deser-size.[uid].csv
* persistent-deser-time.[uid].csv
* persistent-dispatch-time.[uid].csv
* persistent-emit-count.[uid].csv
* persistent-emit-size.[uid].csv
* persistent-emit-time.[uid].csv
* persistent-emitter-queue-size.[uid].csv
* request-count.csv

`request-count.csv`以外のファイルは、各Topologyを投入したユーザID毎に出力されます。TupleStoreServerが複数のホストで稼動している場合、該当のホストで受領したTupleに関してのメトリクス情報のみ出力されます。

`${GUNGNIR_HOME}/conf/gungnir.yaml`にて下記設定をします。

    metrics.reporter:
      - reporter: org.gennai.gungnir.metrics.CsvMetricsReporter
        interval.secs: 60
        output.dir: ${gungnir.home}/logs

#### StatsdMetricsReporter

別途稼動させたStatsDサーバにメトリクス情報を送信します。TupleStoreServerを複数台で稼動させている場合、メトリクス情報をStatsDサーバに集約することができます。

`${GUNGNIR_HOME}/conf/gungnir.yaml`にて下記設定をします。

    metrics.reporters:
      - reporter: org.gennai.gungnir.metrics.StatsdMetricsReporter
        interval.secs: 60
        statsd.host: ${metrics.statsd.host}
        statsd.port: ${metrics.statsd.port}
        statsd.prefix: ${metrics.reporter.prefix}

上記設定例では、`statsd.host`/`statsd.port`/`statsd.prefix`の設定値をそれぞれ<a href="/ja/config.html#s.metrics.statsd.host">metrics.statsd.host</a>/<a href="/ja/config.html#s.metrics.statsd.port">metrics.statsd.port</a>/<a href="/ja/config.html#s.metrics.reporter.prefix">metrics.reporter.prefix</a>の設定値を参照するようにしています。

----

## メトリクス可視化

StatsdMetricsConsumer/StatsdMetricsReporterを使用することでメトリクス情報を可視化することができます。以降では、これらを用いたメトリクス可視化について記載します。

ここに記載するメトリクス可視化には、下記構成の環境を必要とします。

* gennai
* StatsD
* graphite
* grafana
* Elasticsearch

### 構成概要

各システムの構成概要は下記の通りです。

![metrics-system](/img/metrics-system.png)

StatsDに集約したメトリクス情報は、graphiteへ転送し蓄積します。

各システムのインストール方法・基本的な設定は割愛します。

### gennai設定

`${GUNGNIR_HOME}/conf/gungnir.yaml`に下記設定をします。ReporterMetrics/ConsumerMetrics共に同じStatsDに送信します。

    ### Metrics
    metrics.reporters:
      - reporter: org.gennai.gungnir.metrics.StatsdMetricsReporter
        interval.secs: 60
        statsd.host: ${metrics.statsd.host}
        statsd.port: ${metrics.statsd.port}
        statsd.prefix: ${metrics.reporter.prefix}
    metrics.statsd.host: "172.30.1.123"
    metrics.statsd.port: 8125
    metrics.reporter.prefix: "reporter"
    
    topology.metrics.enabled: true
    topology.metrics.consumer: org.gennai.gungnir.metrics.StatsdMetricsConsumer
    topology.metrics.consumer.parallelism: 1
    topology.metrics.interval.secs: 60
    metrics.consumer.prefix: "consumer"

### StatsD設定

graphiteにメトリクス情報を転送する為の設定をします。また、ConsumerMetrics/ReporterMetrics共に60秒間隔での送信となるので、StatsDにおいてもbackendへの転送を60秒間隔としています。

`config.js`

    {
      graphitePort: 2003,
      graphiteHost: "localhost",
      backends: ["./backends/graphite"],
      flushInterval: 60000,
      deleteGauges: true
    }

### 結果

StatsdMetricsConsumer/StatsdMetricsReporterを使用して、StatsDに各種メトリクス情報を集約しました。StatsDのbackendにgraphiteを設定することで、graphite/grafanaにて可視化されたメトリクス情報を参照することが可能となります。

今回はgraphite/grafanaを用いましたが、StatsDに集約したメトリクス情報はStatsDの機能・設定によって他のシステムに送ることも可能です。
