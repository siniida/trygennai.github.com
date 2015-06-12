---
layout: manual_ja
title: CLI / genn.ai
redirect_from: "/cli_ja.html"
---

# genn.ai CLI

> ここでは、genn.aiを操作するためのコマンドラインツールであるgungnir、及びデータ投入ツールであるpostスクリプトについて記載します。

## Gungnir Command Line Options

    $ gungnir options
    usage: gungnir
     -e <quoted-command-string>   Command from command line
     -f <filename>                Command from file
     -h                           Print help information
     -l                           Connect to cluster of local mode
     -p <password>                Password to use when connecting to the
                                  gungnir server
     -u <username>                Username to use when connecting to the
                                  gungnir server

* 60分以上操作がない場合、セッションは切断されます。

---

## Gungnir Interactive Shell Commands

コマンドの終了には";"を使用します。

### quit | exit

Gungnir CLIを終了します。

    gungnir> quit;

---

### EXPLAIN

Topologyの実行計画を表示します。

    gungnir> EXPLAIN [topology_name];

* topology_nameは、`SUBMIT TOPOLOGY`で設定した名称を指定します。
* topology_nameを省略すると、現セッションで最後に投入したTopologyの実行計画を表示します。

> Example:
>
    gungnir> CREATE TUPLE userAction (field1 INT, field2 STRING);
    OK
    gungnir> FROM userAction AS ua1 USING kafka_spout() INTO s1;
    OK
    gungnir> FROM s1 FILTER field1 >= 10 INTO s2;
    OK
    gungnir> FROM s1 FILTER field1 < 10 INTO s3;
    OK
    gungnir> EXPLAIN;
    SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])
     -S-> PARTITION_1
    PARTITION_1(identity grouping)
     -S-> FILTER_2
     -S-> FILTER_3
    FILTER_2(field1 >= 10)
    FILTER_3(field1 < 10)

実行計画から、Tupleを投入した際にオペレータがどのように呼ばれるかを確認することができます。
それぞれのオペレータをVertex、それらをつなぐStreamをEdgeとするGraph構造になっています。
上記の例では、SPOUT_0・PARTITION_1・FILTER_2・FILTER_3の４つのオペレータをStreamで連結しています。
それぞれのオペレータの下の行に、そのオペレータからのびているStreamが表示されています。
SPOUT_0からのびているStreamは１つで、PARTION_1に接続しています。
PARTITION_1からのびているStreamは２つで、それぞれFILTER_2とFILTER_3に接続しています。

#### EXPLAIN EXTENDED

Topologyのより詳細な実行計画を表示します。

    gungnir> EXPLAIN EXTENDED [topology_name];

* topology_nameは、`SUBMIT TOPOLOGY`で設定した名称を指定します。
* topology_nameを省略すると、現セッションで最後に投入したTopologyの詳細な実行計画を表示します。


> Example:
>
    gungnir> FROM userAction AS ua1 USING kafka_spout() INTO s1;
    OK
    gungnir> FROM s1 FILTER field1 >= 10 INTO s2;
    OK
    gungnir> FROM s1 FILTER field1 < 10 INTO s3;
    OK
    gungnir> EXPLAIN EXTENDED;
    Explain:
      SPOUT_0(kafka_spout(), [userAction(field1 INT, field2 STRING)]) parallelism=1
        -S-> PARTITION_1
      PARTITION_1(partitioned grouping(userAction(shuffle)))
        -S-> FILTER_2
        -S-> FILTER_3
      FILTER_2(field1 >= 10) parallelism=1
      FILTER_3(field1 < 10) parallelism=1
    Stream edges:
      SPOUT_0
        incoming: -
        outgoing: PARTITION_1
      PARTITION_1
        incoming: SPOUT_0
        outgoing: FILTER_2, FILTER_3
      FILTER_2
        incoming: PARTITION_1
        outgoing: -
      FILTER_3
        incoming: PARTITION_1
        outgoing: -
    Output fields:
      SPOUT_0 {userAction=[field1, field2]}
      PARTITION_1 {userAction=[field1, field2]}
      FILTER_2 {userAction=[field1, field2]}
      FILTER_3 {userAction=[field1, field2]}
    Group fields:
    Components:
      EXEC_SPOUT {
        SPOUT_0 -SingleDispatcher-> PARTITION_1
      } parallelism=1
      EXEC_BOLT_1 {
        PARTITION_1 -MultiDispatcher[-SingleDispatcher-> FILTER_2, -SingleDispatcher-> FILTER_3]
      } parallelism=1
    Topology:
      EXEC_SPOUT
        -PARTITION_1-> EXEC_BOLT_1
      EXEC_BOLT_1

各項目は下記の通りです。

* Explain: 実行計画(EXPLAINの結果と同様)
* Stream edges: 各オペレータ毎のIN/OUT
* Output fields: 各オペレータが出力するTupleの形式
* Group fields: グループキーに用いているフィールド
* Components: オペレータのStormのComponentにおける配置状況
* Topology: Componentsの連結

---

### SUBMIT TOPOLOGY

Topologyを登録して起動します。

    gungnir> SUBMIT TOPOLOGY topology_name [COMMENT "comment"];

* topology_name には一意となる任意の文字列を指定します。
* topology_name は英数字およびアンダースコアを使用できます。
* comment には、任意の文字列で設定することができます。

> Example:
>
    gungnir> FROM userAction AS ua1 USING kafka_spout() FILTER field1 = 10 INTO s1;
    OK
    gungnir> EXPLAIN;
    …
    gungnir> SUBMIT TOPOLOGY test_topology_1 COMMENT "テスト";
    OK
    Starting ... Done
    {
      "id":"54753ae70cf2422ae5af8e1e",
      "name":"test_topology_1",
      "status":"RUNNING",
      "owner":"user@genn.ai",
      "createTime":"2014-11-26T02:28:55.612Z",
      "comment":"テスト",
      "summary":{
        "name":"gungnir_54753ae70cf2422ae5af8e1e",
        "status":"ACTIVE",
        "uptimeSecs":1,
        "numWorkers":1,
        "numExecutors":3,
        "numTasks":3
      }
    }

`SUBMIT TOPOLOGY` の実行後、表示されるJSONは `DESC TOPOLOGY` を実行して得られる内容と同一です。

---

### DESC TOPOLOGY

登録したTopologyの情報を表示します。

    gungnir> DESC TOPOLOGY [topology_name];

* 結果はJSON形式で出力されます。
* topology_name を省略した場合は、現在のセッションで最後に投入した Topology の情報を表示します。

> Example:
>
    gungnir> DESC TOPOLOGY;
    {
      "id":"54753ae70cf2422ae5af8e1e",
      "name":"test_topology_1",
      "status":"RUNNING",
      "owner":"user@genn.ai",
      "createTime":"2014-11-26T02:28:55.612Z",
      "comment":"テスト",
      "summary":{
        "name":"gungnir_54753ae70cf2422ae5af8e1e",
        "status":"ACTIVE",
        "uptimeSecs":403,
        "numWorkers":1,
        "numExecutors":3,
        "numTasks":3
      }
    }

* id は、Topology IDです。
* name は、`SUBMIT TOPOLOGY`で設定した名称です。
* status は、Topologyの状態です。
Topologyの状態に応じて、STARTING（起動中）-> RUNNING（実行中）-> STOPPING（停止中）-> STOPPED（停止状態）と変化します。
* owner は、Topologyを登録したユーザのユーザ名です。
* createTime は、Topologyを登録した日時です。
* summary は、Topologyの起動情報です。status がRUNNINGの場合のみに表示されます。

特定のTopologyの情報を表示したい場合は、情報を表示したいTopologyの名称を指定します。

    gungnir> DESC TOPOLOGY topology_name;

> Example:
>
    gungnir> DESC TOPOLOGY test_topology_1;
    {"id":"54753ae70cf2422ae5af8e1e", ...}

---

### SHOW TOPOLOGIES

登録したTopologyの一覧を表示します。

    gungnir> SHOW TOPOLOGIES;

結果はJSON形式の配列で出力されます。

> Example:
>
    gungnir> SHOW TOPOLOGIES;
    [
      {
        "id":"54753ae70cf2422ae5af8e1e",
        "name":"test_topology_1",
        "status":"RUNNING",
        "owner":"user@genn.ai",
        "createTime":"2014-11-26T02:28:55.612Z"
      }
    ]

* id は、Topology IDです。
* name は、`SUBMIT TOPOLOGY`で設定した名称です。
* status は、Topologyの状態です。
Topologyの状態に応じて、STARTING（起動中）-> RUNNING（実行中）-> STOPPING（停止中）-> STOPPED（停止状態）と変化します。
* owner は、Topologyを登録したユーザのユーザ名です。
* createTime は、Topologyを登録した日時です。

---

### STOP TOPOLOGY

起動している Topology を停止します。

    gungnir> STOP TOPOLOGY topology_name;

* topology_name には、停止するTopologyの名称を指定します。

> Example:
>
    gungnir> STOP TOPOLOGY test_topology_1;
    OK
    Stopping ...... Done
    {
      "id":"54753ae70cf2422ae5af8e1e",
      "name":"test_topology_1",
      "status":"STOPPED",
      "owner":"user@genn.ai",
      "createTime":"2014-11-26T02:28:55.612Z"
    }

---

### START TOPOLOGY

停止している Topology を再起動します。

    gungnir> START TOPOLOGY topology_name;

* topology_name には、起動するTopologyの名称を指定します。

> Example:
>
    gungnir> START TOPOLOGY test_topology_1;
    OK
    Starting ... Done
    {
      "id":"54753ae70cf2422ae5af8e1e",
      "name":"test_topology_1",
      "status":"RUNNING",
      "owner":"user@genn.ai",
      "createTime":"2014-11-26T02:28:55.612Z",
      "summary":{
        "name":"gungnir_54753ae70cf2422ae5af8e1e",
        "status":"ACTIVE",
        "uptimeSecs":2,
        "numWorkers":1,
        "numExecutors":3,
        "numTasks":3
      }
    }

---

### DROP TOPOLOGY

登録したTopologyを削除します。

    gungnir> DROP TOPOLOGY topology_name;

* topology_name には、削除するTopologyの名称を指定します。

Topologyは停止している必要があります。削除する前に `DESC TOPOLOGY` を実行して、Topologyの状態が STOPPED になっていることを確認してください。

> Example:
>
    gungnir> DESC TOPOLOGY test_topology_1;  <-- 停止しているかを確認
    {"id":"54753ae70cf2422ae5af8e1e","name":"test_topology_1","status":"STOPPED",...}
    gungnir> DROP TOPOLOGY test_topology_1;
    OK

---

### CLEAR

登録したクエリをメモリから消去します。
クエリ（FROM〜）は、TopologyをSUBMITするまでメモリに置かれます。
クエリを打ちなおしたい場合は、CLEARコマンドを実行してメモリをクリアしてください。

    gungnir> CLEAR;

> Example:
>
    gungnir> EXPLAIN;
    SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])
    …
    gungnir> CLEAR;
    OK
    gungnir> EXPLAIN;
    FAILED: Operator isn't registered

---

### SET

Topologyのプロパティを設定、確認することができます。

    gungnir> SET property_name[ = property_value];

property_valueを省略した場合、該当のプロパティに設定されている値を表示します。

設定したプロパティは、 `SUBMIT TOPOLOGY` もしくは `START TOPOLOGY` で、起動するTopologyに反映されます。
プロパティは、 `STOP TOPOLOGY` で初期化されてしまうので、 `START TOPOLOGY` を実行する前に再設定する必要があります。

設定可能なプロパティは下記の通りです。

* default.parallelism
* kafka.emit.brokers
* kafka.emit.required.acks
* kafka.monitor.brokers
* kafka.monitor.required.acks
* kafka.spout.topic.replication.factor
* mongo.fetch.servers
* mongo.persist.servers
* monitor.enabled
* topology.metrics.enabled
* topology.metrics.interval.secs
* topology.workers

設定したプロパティはクライアントのセッションが閉じられるまで保持されます。一度のセッションで複数のTopologyをSUBMITする場合には、注意してください。

SUBMITしたTopologyに設定されたプロパティは変更することができません。プロパティを変更する場合には、`STOP TOPOLOGY`、`DROP TOPOLOGY`を実施後、再度SETにてプロパティの設定を行い、Topologyを再設定してください。

> Example:
>
    gungnir> SET monitor.enabled = true;

Monitorログを出力するように、プロパティを設定します。

---

### POST

Topologyの動作確認の為に、TopologyにJOINTupleを送信します。

    gungnir> POST tuple_name json_tuple;

* tuple_name には、送信するTuple名を指定します。
* join_tuple には、送信するJSONTupleを記述します。

応答にSet-Cookieヘッダが返された場合は、その内容がCLIのメモリCookieに保存され、保存されたCookieは、次回のPOSTコマンドの実行時にCookieヘッダとして送信されます。

> Example:
>
    gungnir> POST userAction {field1:10,field2:"test"};
    POST /gungnir/v0.1/547539ed0cf2422ae5af8e1c/userAction/json
    OK

#### Interactive Mode

POSTコマンドには、インタラクティブにJSONTupleを編集できるモードがあります。
json_tuple を指定せずにPOSTコマンドを実行すると、送信するJSONTupleの入力を求められます。
JSONTupleのすべてのフィールドの入力が完了すると、編集したJSONTupleを送信します。

    gungnir> POST userAction;
    field1 (INT): 12345
    field2 (STRING): test
    POST userAction {"field1":12345,"field2":"test"}
    POST /gungnir/v0.1/547539ed0cf2422ae5af8e1c/userAction/json
    OK

LIST, MAP, STRUCT型のフィールドの編集は、以下のように行います。

    CREATE TUPLE userAction2 (f1 STRING, f2 LIST<INT>, f3 MAP<STRING, INT>, f4 STRUCT<m1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), m2 BOOLEAN>);

> Example:
>
    gungnir> POST userAction2;
    f1 (STRING): test
    f2 (LIST)
      0 (INT): 1
      1 (INT): 2
      2 (INT): 3
      3 (INT):   <-- [Enter key]を押して、LISTの編集を終了
    f3 (MAP)
      key (STRING): k1
      value (INT): v1
      value (INT): 1
      key (STRING): k2
      value (INT): 2
      key (STRING): k3
      value (INT): 3
      key (STRING):   <-- keyの入力で [Enter key]を押して、MAPの編集を終了
    f4 (STRUCT)
      m1 (TIMESTAMP(yyyy-MM-dd HH:mm:ss)): 2013-10-19 22:02:24
      m2 (BOOLEAN): false
>
    POST userAction2 {"f1":"test","f2":[1,2,3],"f3":{"k1":1,"k2":2,"k3":3},"f4":{"m1":"2013-10-19 22:02:24","m2":false}}
    POST /gungnir/v0.1/547539ed0cf2422ae5af8e1c/userAction2/json
    OK

---

### COOKIE

メモリCookieの内容を表示します。
メモリCookieは、CLIの終了とともにメモリから削除されます。

    gungnir> COOKIE;

> Exmaple:
>
    gungnir> COOKIE;
    {"name":"TID","value":"52558684e4b0f241c72d0365","domain":null,"path":null,"comment":null,"commentUrl":null,"discard":false,"ports":[],"maxAge":864000000,"version":1,"secure":false,"httpOnly":false}]

---

### COOKIE CLEAR

メモリCookieの内容を削除します。
メモリCookieには、Serverから受け取ったTracking IDが格納されている為、メモリCookieの内容を削除することによって
Serverから新たなTracking IDを受け取ることができます。

    gungnir> COOKIE CLEAR;

> Example:
>
    gungnir> COOKIE CLEAR;
    OK
    gungnir> COOKIE;
    []

---

### MONITOR <a name="monitor" class="anchor"></a>

Topologyの動作確認の為に、TupleがTopologyの中をどのように流れたをMonitor機能で確認します。
Tupleが流れた際に出力されるMonitorログを表示することによって、確認を行います。

    gungnir> MONITOR topology_id;

* 確認するTopologyのTopology IDを指定します。

Monitor機能は、以下の手順で使用します。

1. Monitorログを出力するようにプロパティを設定して、該当のTopologyをSUBMITします。
2. 1で起動したTopologyのTopology IDを指定して、MONITORコマンドを実行します。
3. 別のCLIを起動し、POSTコマンドを使ってJOINTupleを送信します。JOINTupleは、curlコマンド等を使って送信することもできます。
4. MONITORコマンドを実行しているCLIに、3で送信したTupleが流れることによって出力されたMonitorログが表示されます。

> Example:
>
    gungnir> SET monitor.enabled = true;
    OK
    gungnir> SUBMIT TOPOLOGY;
    OK
    gungnir> DESC TOPOLOGY;
    {"id":"5261606ee4b099995d4f460f","status":"RUNNING","owner":"user@genn.ai","createTime":"2013-10-18T16:23:09.901Z","summary":{"name":"gungnir_5261606ee4b099995d4f460f","status":"ACTIVE","uptimeSecs":403,"numWorkers":1,"numExecutors":3,"numTasks":3}}
    gungnir> MONITOR 5261606ee4b099995d4f460f;

別のCLIを開いてPOSTコマンドを実行します。

    gungnir> POST userAction {"field1":1234,"field2":"test"};

Monitorログが出力されます。

    gungnir> MONITOR 5261606ee4b099995d4f460f;
    {"time":"2013-10-19T14:12:36.856Z","source":"SPOUT_0","target":"PARTITION_1","tuple":{"tupleName":"userAction","values":[1234,"test"]}}
    {"time":"2013-10-19T14:12:36.880Z","source":"PARTITION_1","target":"FILTER_2","tuple":{"tupleName":"userAction","values":[1234,"test"]}}
    {"time":"2013-10-19T14:12:36.953Z","source":"PARTITION_1","target":"FILTER_3","tuple":{"tupleName":"userAction","values":[1234,"test"]}}

source のオペレータから target のオペレータに向かって、Tupleが流れているのが確認できます。tuple に、流れたTupleの内容が表示されています。

---

### DESC CLUSTER

Stormのクラスタ情報を取得します。

    gungnir> DESC CLUSTER;

* rootユーザのみ実行可能です。

> Example:
>
    {
      "mode":"distributed",
      "nimbus":{"uptimeSecs":9894},
      "supervisors":[
        {
          "id":"4e39bbf4-8e74-4842-8d3a-9e768aea41b4",
          "host":"10.0.1.23",
          "uptimeSecs":9887,
          "numWorkers":4,
          "numUsedWorkers":0
        },
        {
          "id":"5ae76a62-a850-4912-bdb3-22f0895e9161",
          "host":"10.0.1.243",
          "uptimeSecs":9888,
          "numWorkers":4,
          "numUsedWorkers":1
        }
      ],
      "topologies":[
        {
          "name":"gungnir_54757345e4b02392c29483e1",
          "status":"ACTIVE",
          "uptimeSecs":66,
          "numWorkers":1,
          "numExecutors":3,
          "numTasks":3
        }
      ]
    }

* JSON形式で情報を取得します。
* mode は、起動しているStormのモードです(distributed|local)。
* nimbus, supervisorsは、Stormに関する情報です。
* topologies は、稼働中のTopology情報です。

---

### CREATE USER

ユーザアカウントを作成します。

    gungnir> CREATE USER 'user' IDENTIFIED BY 'password';

* userには、作成するユーザアカウントの名称を指定します。
* userには、英数字、アンダースコア(_)、アットマーク(@)、.等を使用できます。
* userが英数字、アンダースコア(_)のみで構成される場合、シングルクォートを省略することが可能です。
* passwordには、作成するユーザアカウントのログインパスワードを指定します。
* rootユーザのみ実行可能です。

> Example: gennaiユーザをパスワードgennaiで作成
>
    gungnir> CREATE USER gennai IDENTIFIED BY 'gennai';
    OK
> Example: gennai@example.comユーザをパスワードgennaiで作成
>
    gungnir> CREATE USER 'gennai@example.com' IDENTIFIED BY 'gennai';
    OK

---

### ALTER USER

ユーザアカウントのパスワードを変更します。

    gungnir> ALTER USER user IDENTIFIED BY 'password';

* userには、変更するユーザアカウントの名称を指定します。
* passwordには、変更後のパスワードを指定します。
* rootユーザでは、すべてのユーザアカウントのパスワードを変更できます。
* root以外のユーザでは、ログインしているユーザアカウントのパスワードのみの変更ができます。

> Example: gennaiユーザのパスワードをgennai2に変更する
>
    gungnir> ALTER USER gennai IDENTIFIED BY 'gennai2';
    OK

---

### DROP USER

ユーザアカウントを削除します。

    gungnir> DROP USER user;

* userには、削除するユーザアカウントの名称を指定します。
* rootユーザのみ実行可能です。
* rootユーザを削除することはできません。
* 対象のユーザアカウントが作成したTUPLE/TOPOLOGYが存在する場合は、ユーザカウントを削除できません。

> Example: gennaiユーザを削除する
>
    gungnir> DROP USER gennai;
    OK


---

### SHOW USERS

ユーザアカウントの一覧をJSON形式で表示します。

    gungnir> SHOW USERS;

* rootユーザのみ実行可能です。

> Example:
>
    gungnir> SHOW USERS;
    [
      {
        "id":"547d16890cf2089ea8c0367d",
        "name":"root",
        "createTime":"2014-12-02T01:31:53.797Z"
      },
      {
        "id":"547d169e0cf2089ea8c0367e",
        "name":"gennai",
        "createTime":"2014-12-02T01:32:14.294Z"
        "lastModifyTime":"2014-12-02T05:45:04.068Z"
      },
      {
        "id":"547d519f0cf2089ea8c03687",
        "name":"gennai@genn.ai",
        "createTime":"2014-12-02T05:43:59.293Z"
      }
    ]

---

## Gungnir Command Batchmode

gungnirコマンドのバッチモードを使用します。

### from command line

短文のクエリには-eオプションで実行が可能です。

    $ gungnir -u [user] -p [pass] -e "query"

実行するクエリはダブルクォーテーションで括る必要があります。

> Example:
>
    $ gungnir -u gennai -p gennai -e "DESC USER;"
    Gungnir server connecting ...
    Gungnir version 0.0.1 build at 20150114-013107
    Welcome gennai (Account ID: 549a345d0cf2a9c38ec5789a)
    {"id":"549a345d0cf2a9c38ec5789a","name":"gennai","createTime":"2014-12-24T03:34:53.195Z"}
    $

---

### from  query file

複数のクエリを実行するにはクエリファイルを別途用意し、gungnirコマンドの-fオプションでクエリファイルを指定します。

    $ gungnir -u [user] -p [pass] -f [query file]

> Example:
>
    $ cat test.q
    FROM userAction AS ua1 USING kafka_spout() INTO s1;
    FROM s1 FILTER field1 >= 10 INTO s2;
    FROM s1 FILTER field1 < 10 INTO s3;
    SUBMIT TOPOLOGY test;
    $
    $ gungnir -u [user] -p [pass] -f test.q
    ...

クエリファイルに記述された全クエリが実行され、上記ExampleではTopologyが実行された状態になります。

クエリファイルでは下記コマンドを使用することができません。

* POST
* COOKIE
* MONITOR

## Post Script

postスクリプトは、コマンドラインからTupleStoreServerにTupleの送信を行います。テストやベンチマークツールとしての利用を想定しています。

postスクリプトでは、ユーザログインを行っていないため、送信先のTuple定義が存在するかチェックをしていません。

## Post Script Command Line Options

    $ post options
    usage: post [-options] [JSON tuple]
     -a <accountid>               Account ID to use when posting to the tuple
                                  store server
     -f <filename>                JSON tuple from files
     -h                           Print help information
     -l <response-time-limits>    Response time limits (secs)
     -n <number-of-times>         number of times
     -p <number-of-parallelism>   Number of parallelism
     -q <size-of-send-queue>      Size of send queue
     -s                           Print statistics information
     -t <tuplename>               Tuple name to use when posting to the tuple
                                  store server
     -v                           Make the operation more talkative

## Post Script Examples

ここではpostスクリプトの使用例を記載します。

### 引数に指定されたTupleを送信

    $ post -a 549a345d0cf2a9c38ec5789a -t test -v '{t1: 1}'
    POST /gungnir/v0.1/549a345d0cf2a9c38ec5789a/test/json
    
    account ID:       549a345d0cf2a9c38ec5789a
    tuple name:       test
    parallelism:      1
    times:            1
    queue size:       8192
    time limits(sec): 10
    
    HTTP/1.1 204 No Content
    Content-Length: 0
    Date: Thu, 15 Jan 2015 04:57:22 GMT
    Finished 0 requests
    
    Time(sec):          309746.296
    Completed requests: 0
    Failed requests:    0
    Timeout requests:   0
    $

### 引数に指定されたファイルを送信

    $ cat post.data
    {aaa:"aaa1", bbb:10, ccc:100, ddd:"ddd1"}
    {aaa:"aaa2", bbb:10, ccc:100, ddd:"ddd2"}
    {aaa:"aaa3", bbb:10, ccc:100, ddd:"ddd3"}
    $ ./bin/post -a 549a345d0cf2a9c38ec5789a -t tuple1 -v -f post.data

### 標準入力からTupleを流し込み

    $ cat post.data | ./bin/post -a 5465e7d3e4b008da18561803 -t tuple1 -p 2


### ベンチマークツールとしての利用

    $ ./bin/post -a 5465e7d3e4b008da18561803 -t tuple1 -n 50 -p 3 '{aaa:"aaa3", bbb:10, ccc:100, ddd:"ddd3"}'

上記例では、指定したTupleを50回×3並列で送信されます。従って、計150個のTupleが送信されます。

    $ ./bin/post -a 5465e7d3e4b008da18561803 -t tuple1 -n 50 -p 3 -f post.data

上記例ではファイルを指定しているので、ファイルに記述されているTuple数×50回×3並列で送信されます。従って、ファイルに記述されているTuple数が10であったとすると、10×50×3=1500個のTupleが送信されます。
