---
layout: manual_ja
title: DML / genn.ai
redirect_from: "/dml_ja.html"
---

# genn.ai DML

{% include lead.md %}

## FROM

FROM を使い、Tupleの入力元をスキーマとともに指定します。

### TupleStoreServerからの入力 <a name="FROM_REST" class="anchor"></a>

    FROM schema_name AS schema_alias, ... USING spout_processor


* schema_name には、Tuple名もしくはView名を指定します。
* schema_alias には、クエリ内で使用するTupleもしくはViewの別名を指定します。
* spout_processor には、読み込みに使用するプロセッサを指定します。

> Example:
>
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout()


TupleStoreServerからの入力は、一つのTopologyに対して一つしか定義できません。

#### Kafka Spout Processor <a name="KAFKA_SPOUT" class="anchor"></a>

TupleをKafkaから読み込みます。
[DDL](ddl.html) で説明されている`CREATE TUPLE`にて作成したスキーマからの入力Tuple(RESTで投入)もKafka経由で読み上げます。

    kafka_spout([offset, [force]])

* offset には、Topicの読み出し位置を指定します。指定可能な値は下記の通りです。
  * -1 or 省略: Topicの末尾から読み出し
  * -2: Topicの先頭から読み出し
* force に`true`を指定すると、Topologyの再起同時に保存されているオフセット情報を利用せず先頭からデータを読み込みます。


> Example: Topicの先頭から読み出し
>
    FROM userAction1 USING kafka_spout(-2)

> Example: 再起同時に必ずTopicの先頭から読み出し
>
    FROM userAction1 USING kafka_spout(-2, true)

#### Memory Spout Processor <a name="MEMORY_SPOUT" class="anchor"></a>

[ローカルモード](config.html#mode.local)における動作確認用として、Kafkaを経由せずに、メモリからTupleを読み込みます。

    memory_spout()

Memory Spout Processorを使用するには、GungnirServerの下記設定項目が共に **local** となっている必要があります([ローカルモード](config.html#mode.local)での稼働)。

* [cluster.mode](config.html#s.cluster.mode)
* [storm.cluster.mode](config.html#s.storm.cluster.mode)

### ストリームからの入力 <a name="FROM_STREAM" class="anchor"></a>

    FROM stream_name[(tuple_name, ...) AS tuple_alias], ...

* stream_name には、 [INTO](dml.html#INTO) もしくは [EMIT](dml.html#EMIT_INTERNAL) で作られたストリームを指定します。
* tuple_name には、Tupleを指定します。Tupleを指定すると、ストリーム中の該当のTupleのみを読み込むようになります。
* tuple_alias には、任意のTuple名を指定します。Tuple名を指定すると、以降のストリーム中でのTuple名を変更することができます。

> Example: ストリームからすべてのTupleを読み込む場合
>
    FROM s1, s2

> Example:s1ストリームからua1, ua2タプルのみ読み込み、s2ストリームからはv1タプルのみ読み込む場合
>
    FROM s1(ua1, ua2), s2(v1)
>
* ここで使われているua1やua2は`CREATE TUPLE`で作られたタプル名、s1はそれらタプルが流れてくるストリーム名です。

### JOIN <a name="TupleJoin" class="anchor"></a>

複数のTupleをフィールドの値を元に結合します。

結合対象のTupleが未到達の場合、先に到着しているTupleはメモリに保持され結合対象のTupleの到着を待機します。未結合のTupleが多くなると想定される場合、`USING file_cache()`構文を追加することで、Tupleをファイルに一時的に格納することができます。
ファイルキャッシュは、<a href="/ja/config.html#s.gungnir.local.dir">gungnir.local.dir</a>で指定したディレクトリに作成されます。

`USING file_cache()`構文を省略した場合、もしくは`USING memory_cache()`を明示的に指定した場合には、Tupleはメモリに保存されます。

#### TupleStoreServerからTupleを読み込んで結合 <a name="RestJoin" class="anchor"></a>

TupleStoreServerからTupleを読み込む時点で、複数のTupleを結合します。

    FROM (schema_name_1
      JOIN schema_name_2 ON join_condition
      TO join_field
      EXPIRE period
      [USING file_cache() | memory_cache()]
    ) AS schema_alias USING spout_processor

    join_condition:
    schema_name_1.key_field = schema_name_2.key_field

    join_field:
    schema_name_1.join_field [AS field_alias, ...]

* schema_name_1, schema_name_2は結合したいTupleを指定します。
* join_conditionにはTupleの結合条件を指定します。複数条件の場合にはANDで指定します。
* join_fieldには結合後のTupleが保持するフィールドを指定します。結合後のフィールド名称が被らない場合には、ワイルドカードを使用する事やaliasを省略することが可能です。
* periodには、Tupleを保持する期間を指定します。指定した時間経過後にJoin対象のTupleが読み込まれると、Tuple Joinは実行されません。

> Example: ua1, ua2, ua3の3つのTupleを結合して、ua4のTupleを作成
>
    FROM (ua1
      JOIN ua2 ON ua1.field1 = ua2.field3
      JOIN ua3 ON ua1.field2 = ua3.field5
      TO ua1.*, ua2.field4 AS field4, ua3.field7 AS field7
      EXPIRE 1min
    ) AS ua4 USING kafka_spout()

#### ストリームからTupleを読み込んで結合 <a name="StreamJoin" class="anchor"></a>

ストリームから、一部のTupleに対してTupleを結合します。

    FROM (stream_name_1(tuple_name_1)
      JOIN stream_name_2(tuple_name_2) ON join_condition
      TO join_field
      EXPIRE period
      [USING file_cache()]
    ) AS schema_alias

    join_condition:
    tuple_name_1.key_field = tuple_name_2.key_field

    join_field:
    tuple_name_1.join_field [AS field_alias, ...]

* tuple_nameを指定する必要があります。
* join_fieldには結合後のTupleが保持するフィールドを指定します。結合後のフィールド名称が被らない場合には、ワイルドカードを使用する事やaliasを省略することが可能です。
* periodには、Tupleを保持する期間を指定します。指定した時間経過後にJoin対象のTupleが読み込まれても、Tuple Joinは実行されません。

> Example: ua1とua2をストリームs1から読み込み、ua3をストリームs2から読み込んで結合
>
    FROM (s1(ua1)
      JOIN s1(ua2) ON ua1.field1 = ua2.field3
      JOIN s2(ua3) ON ua1.field2 = ua3.field5
      TO ua1.*, ua2.field4 AS field4, ua3.field7 AS field7
    ) AS ua4

---

#### Column:「スキーマとタプル」

gennaiには、スキーマ(Schema)とタプル(Tuple)という似た定義の言葉が存在します。

##### スキーマ(Schema)

TupleStoreServerが受領するJSON形式のデータをタプルに変換する為のルールのようなものです。[`CREATE TUPLE`](ddl.html#CREATE_TUPLE)で定義されます。

##### タプル(Tuple)

トポロジ(Topology)を流れるデータは、`JOIN`や`EACH`といった各オペレータにて、フィールドそのものや、その保持する値を追加・削除・編集され、タプルの形状を様々に変化させていきます。当初定義されているスキーマとはまったくの別物になることもありますし、スキーマと同じフィールド定義でストリームの終端まで達することもあります。

以上より当ドキュメント内では、DDLで作成する定義および`FROM`でKafkaよりデータを取得する際にはスキーマ(Schema)を使用し、それ以降においてはタプル(Tuple)を使用しています。

---

## INTO <a name="INTO" class="anchor"></a>

INTO では、ストリームを出力する(分岐する)ことができます。

    INTO stream_name

* stream_name には、出力するストリーム名を指定します。

> Example:
>
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1

INTO を使って出力したストリームは、FROM で読み込むことが出来ます。

### 分岐, 合流

分岐の例を以下に示します。

    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1;
    FROM s1(ua1) ...
    FROM s1(ua2) ...
    FROM s1(v1) ...

合流の例を以下に示します。

    FROM s1(ua1) ... INTO s2;
    FROM s1(ua2) ... INTO s3;
    FROM s1(v1) ... INTO s4;
    FROM s2, s3, s4 ...

---

## JOIN

JOINは、外部データをフィールドとしてTupleに結合します。

    JOIN join_fields USING fetch_processor

    join_fields:
    join_field [, ...]

* join_fields には、結合するフィールドを全て指定します。
フィールドはTupleに追加されます。

fetch_processorにて複数のドキュメントが返却された場合、Tupleは返却されたドキュメントと同数複製されます。

JOINは、`GROUP BY`されたストリームの中で使用することはできません。`GROUP BY`の実行前にJOINをするか、`END GROUP`もしくは`TO STREAM`で`GROUP BY`のストリームから抜けた後にJOINを実行してください。


### データストアとの結合

#### Mongo Fetch Processor <a name="MONGO_FETCH" class="anchor"></a>

MongoDB上にあるデータとの結合が可能です。MongoDBにおけるfind()を実行します。

    JOIN .. USING mongo_fetch(db_name, collection_name, mongo_query, fields, sort, limit, expire)

* db_name には、入力とするDB名を指定します。
* collection_name には、入力とするCollection名を指定します。
* find_query には、MongoDBにおけるfind()で指定するのと同様の検索条件を指定します。MongoDBの各種関数は使用できません。
`@フィールド名`もしくは`@フィールド名!`で、クエリにTupleのフィールドを組み込む事ができます(プレースホルダ)。
`@`をクエリに含む場合は、`\@`のようにエスケープしてください。
* fields には、MongoDBにおけるfind()で取得するフィールド名を配列形式で指定します。
* sort には、MongoDBから取得するデータのソート条件を指定します。省略可能です。
* limit には、MongoDBにおけるfind()の取得ドキュメント数を指定します。省略可能です。
* expire には、MongoDBから受け取られたデータをキャッシュする時間を指定します。省略するとキャッシュは行われません。

> Example: ソート条件とリミットを指定
>
    JOIN field10, field11
      USING mongo_fetch(
        'db1',
        'col1',
        '{$and: [{code1: @field1}, {code2: @field2}, {del: 0}]}',
        ['name', 'type'],
        '{code1: -1}',
        1,
        1min
      )

> Example: ソート条件のみ指定
>
    JOIN field10, field11
      USING mongo_fetch(
        'db1',
        'col1',
        '{$and: [{code1: @field1}, {code2: @field2}, {del: 0}]}',
        ['name', 'type'],
        '{code1: -1}',
        1min
      )

> Example: リミット条件のみ指定
>
    JOIN field10, field11
      USING mongo_fetch(
        'db1',
        'col1',
        '{$and: [{code1: @field1}, {code2: @field2}, {del: 0}]}',
        ['name', 'type'],
        1,
        1min
      )

#### JDBC Fetch Processor <a name="JDBC_FETCH" class="anchor"></a>

RDB上にあるデータとの結合が可能です。デフォルトではDriverはMySQLになっています。

    JOIN .. USING jdbc_fetch(connect, user, password, query, expire)

* connect には、RDBへの接続文字列を指定します。
* user には、RDBにログインする為のユーザを指定します。
* password には、RDBにログインするユーザのパスワードを指定します。
* query には、RDBに対して実行するクエリを指定します。`@フィールド名`もしくは`@フィールド名!`で、クエリにTupleのフィールドを組み込むことができます(プレースホルダ)。`@`をクエリに含む場合は、`\@`のようにエスケープしてください。
* expire には、RDBから取得したデータをキャッシュする時間を指定します。省略するとキャッシュは行われません。

> Example:
>
    JOIN f2, f3 USING jdbc_fetch(
      'jdbc:mysql://localhost/db1',
      'gennai',
      'gennai',
      'SELECT field2, field3 FROM test WHERE field1 = @f1'
    )

### Webサービスとの結合

#### Web Fetch Processor <a name="WEB_FETCH" class="anchor"></a>

JSON形式のレスポンスを返すWebサービスとの結合か可能です。
結合条件をクエリパラメータに変換し、指定したURLにアクセスします。
取得したJSON形式のレスポンスデータの一部を抜き出し、Tupleと結合します。

    JOIN .. USING web_fetch(url, path, fields, expire)

* url には、データを取得するURLを指定します。プレースホルダでTupleのフィールドを指定する事ができます。
* path には、返却されるJSONの親ノードを指定します。
* fields には、取得するノード名を配列形式で指定します。
* expire には、結果をキャッシュする時間を指定します。省略するとキャッシュは行われません。

> Example: Solrのデータと結合する
>
    JOIN field10, field11
      USING web_fetch(
        'http://localhost:3000/solr/select?q=books.title:@field1+AND+books.author:@field2&fl=id,price&wt=json',
        'response.docs',
        ['id', 'price'],
        10min
      )
>
> レスポンスデータ: pathで指定したパスをルートパスとしてJSONのパースを行います。
>
    {
      response:{
        docs:[
          {　←★ここから読み始める
            id:"978-1423103349",
            title:"The Sea of Monsters",
            author:"Rick Riordan",
            price:6.49
          }
        ]
      }
    }

### キャッシュ

Fetch Processorにてキャッシュしている間に実行されたJOIN句は、キャッシュから結合フィールドを取得し、Tupleに結合します。
指定した時間を過ぎると、再度データを取得し、キャッシュが更新されます。

キャッシュは結合キーごとに保存されます。

---

## FILTER

FILTER は、単一のTupleに対して通過の可否を判定します。

    FILTER condition

* condition に、フィルタの条件を指定します。

condition の符号には、以下のものを指定します。

* &#61; もしくは &#61;&#61;
* <> もしくは !&#61;
* &gt;
* &gt;=
* &lt;
* &lt;=
* LIKE
* REGEXP
* IN, ALL, BETWEEN
* AND
* OR
* NOT

### &#61;, &#61;&#61;, <>, !&#61;, >, >&#61;, <, <&#61;

論理比較を行います。

> Example:
>
    field1 >= 10

### LIKE, REGEXP

LIKEで使用できるワイルドカードは、"%"（複数文字）と"_"（一文字）です。

> Example:
>
    field2 LIKE 't%'

REGEXPで使用できる正規表現は、[java/util/regex/Patternと同じ書式](http://docs.oracle.com/javase/6/docs/api/java/util/regex/Pattern.html)
を採用しています。

> Example:
>
    field3 REGEXP '^[A-Z]{2}-[0-9]{4}$'

### IN, ALL, BETWEEN

INは、LISTフィールドに対して値が一つでも含まれているかを調べます。

> Example:
>
    field4 IN ('tokyo', 'kyoto', 'osaka')

ALLは、LISTフィールドに対して値がすべて含まれているかを調べます。

> Example:
>
    field4 ALL ('tokyo', 'kyoto', 'osaka')

BETWEENは、INT型での範囲を指定します。

> Example:
>
    field1 10 AND 100


### AND, OR, NOT

AND, OR, NOT は入れ子にすることが可能です。優先順位はNOT, AND, ORの順に処理されます。
優先順位は、()を使用して変更できます。

> Example:
>
    field1 <= 30 AND (field5 BETWEEN 10 AND 100 OR field2 LIKE 'A%')

### その他(比較)

以下、基本型以外での比較方法を示します。

#### STRUCT型フィールドの比較

TupleのフィールドがSTRUCT型の場合は、フィールド値を以下のように比較します。

> Example:
>
    field6.member3 = 100

#### LIST型フィールドの比較

TupleのフィールドがLIST型の場合は、フィールド値を以下のように比較します。

> Example:
>
    field4[0] = 'tokyo'

#### MAP型フィールドの比較

TupleのフィールドがMAP型の場合は、フィールドの値を以下のように比較します。

> Example:
>
    field7['visa'] = true


#### 定数

条件に指定できる定数は、以下になります。

* 文字列

    シングルクォートまたはダブルクォートでくくった文字列

* INT値

    number only

    > Example:
    >
      2147483647

* DOUBLE値

    number.number

    > Example:
    >
      12.5

* BIGINT値

    numberL

    > Example:
    >
      9223372036854775807L

* SMALLINT値

    numberS

    > Example:
    >
      32767S

* TINYINT値

    numberY

    > Example:
    >
      255Y

* FLOAT値

    number.numberF

    > Example:
    >
      12.5F

* BOOLEAN値

    > Example:
    >
      true|false


---

### FILTER GROUP

FILTER GROUP は、複数のTupleに対してTupleの通過を判定します。

    FILTER GROUP EXPIRE period [STATE TO state_field]
      condition, ...


* period には、フィルタの状態を保持する期間を指定します。
* state_field には、フィルタの状態を出力するフィールド名を指定します。フィルタを通過すると、指定したフィールド名で
Tupleに状態フィールドを追加します。STATE TO clause を省略した場合は、状態フィールドは追加しません。
* condition には、フィルタ条件を指定します。カンマ区切りで、複数のTupleに対する条件を指定します。
カンマ区切りで指定した条件をすべて満たせば、Tupleはフィルタを通過します。
すべての条件を満たした場合、最後に到着したTupleがフィルタを通過し、フィルタの状態は初期化されます。

> Example:
>
    FILTER GROUP EXPIRE 7DAYS STATE TO fg_state
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member2 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'


 ua1, ua2, ua3の３つのTupleに対してフィルタを実行します。
 条件１と条件２（※）の両方を満たせば、Tupleはフィルタを通過します。
 （※）条件１はua1に対するフィルタで、条件２はua2とua3に対するフィルタです。

 条件２は、ua2とua3の条件を"OR"で指定しているので、ua1かつua2の条件を満たすか、ua1かつua3の条件を満たすことで
 すべての条件は満たされます。

 条件１もしくは条件２のいずれかを満たした状態を７日間保持します。
 例えば、条件１を最後に満たした状態で、条件２の条件を満たすまでの期間が７日以内であれば、条件１と条件２は満たされますが、
 ８日以上の日数が経過していた場合、条件１の状態は初期化されてしまう為、条件２のみ満たしている状態になります。

 state_fieldに"fg_state"を指定しているので、フィルタの状態を"fg_state"フィールドとしてTupleに追加します。
 fg_stateは、条件１と条件２のそれぞれを満たした日時が格納されます。
 条件の数と等しいTIMESTAMPのLISTになります。

#### 状態の保持

period を用いて、フィルタの状態をどれだけ保持するか、を指定します。

* 秒で指定

    number(SECONDS|SEC)

    > Example:
    >
      30SECONDS

* 分で指定

    number(MINUTES|MIN)

    > Example:
    >
      55MIN

* 時間で指定

    number(HOURS|H)

    > Example:
    >
      55HOURS

* 日で指定

    number(DAYS|D)

    > Example:
    >
      15DAYS

---

## EACH

EACH は、Tupleの集計や編集を実行します。

    EACH expr, ...

* exprには、集計関数や編集関数、四則演算、フィールドのアクセサを指定します。

### フィールドのアクセサ

> Example:
>
    EACH field1, field6.member1 AS field10, field7['visa'] AS visa

field1はそのまま、field6.member1をfield10フィールドへ、field7&#91;'visa'&#93;をvisaフィールドへ抽出します。

### 関数

`EACH`句では下記関数を使用することができます。

* <a href="#total_functions">集計関数</a>
* <a href="#edit_functions">編集関数</a>
* <a href="#operation_functions">四則演算</a>
* <a href="#math_functions">数学関数</a>
* <a href="#distance_functions">距離計算</a>

---

## LIMIT

Tupleの流れを制限します。
現在、時間による制限と、タプル数による制限の方法があります。

    LIMIT [FIRST|LAST] period

* periodには、時間もしくはTuple数を指定します。
* FIRSTは、指定範囲内に最初に到着したTupleを後続の処理に渡します。
* LASTは、指定範囲内の最後に到着したTupleを後続の処理に渡します。

### 時間による流量制限

* 時間の起点は、最初にTupleが到着した時間です。
* 起点は指定時間が経過した以降にリセットされます。

最初に到着したTupleを後続に送り、以降30分はTupleを送らない。

> Example:
>
    LIMIT FIRST EVERY 30min

Tupleが最初に到着してから、30分間に到着した最後のTupleを後続に送る。

> Example:
>
    LIMIT LAST EVERY 30min

* 時間の起点は、最初にTupleが届いた時になります。起点は30分経過後にリセットされます。
* 30分経過したかどうかは、30分経過後に初めてTupleが届いた時点で判断されます。　　

### Tuple数による流量制限

5件のTupleの中で、最初に到着したTupleを後続に送る。

> Example:
>
    LIMIT FIRST EVERY 5

5件のTupleの中で、最後に到着したTupleを後続に送る。

> Example:
>
    LIMIT LAST EVERY 5

 * いずれもカウンタは５件受け取るごとにクリアされます。

#### 補足

LIMITを使うことによって、
LIMIT FIRSTであれば、１度EMIT（通知）したTupleを一定時間EMITさせないように制限したり、
LIMIT LASTであれば、最初にアクションし始めてから、４時間の間で一番最後に行ったアクションを抽出したりすることができます。

ただし、LIMIT LASTの場合、Tupleの通過のトリガーが、時間経過後に初めてきたTupleになるので、通過までタイムラグが発生します。
例えば、30分間の最後にTupleが届き、次のTupleが届くまでに１週間かかった場合、その最後のTupleが送られるのは１週間後になります。

---

## SLIDE

集計関数によるスライド集計を実行します。
間隔は時間もしくはTuple数を指定することが可能です。
スライドはTupleの到着時にのみ実行されます。

    SLIDE LENGTH period [BY time_field| count] expr

* periodには、スライド時間もしくはタプル数を指定します。
* exprには、集計関数や編集関数、四則演算、フィールドのアクセサを指定します。

### 時間でスライド

    SLIDE LENGTH period BY time_field expr

* time_fieldには、起点となるTIMESTAMP型のフィールドを指定します。

> Example:
>
    SLIDE LENGTH 10sec BY _time sum(field1) AS new_field

"LENGTH スライド時間 BY 起点となるフィールド名 集計関数"
といった書式で指定します。

* 起点となるフィールド名には、Timestamp型のフィールドを指定します。
  _timeフィールドも指定できます。（タプルに_timeフィールドを指定していた場合）
* 集計関数は複数指定できます。現状では、sum(), avg(), count()の３つの集計関数が使用できます。

### Tuple数でスライド

> Exapmle:
>
    SLIDE LENGTH 100 sum(field2) AS new_field

LENGTHにスライドするTupleの数を指定します。

スライドはTupleの到着時にのみ実行されます。（スライドによる再計算も到着時のみ）

処理的には、Tupleが到着する度にTupleをウィンドウ領域に貯めつつ、都度集計を行なっていきます。
その際、ウィンドウ幅から除外されたTuple（※）を集計結果から減算します。

（※）スライドして集計対象から外れたTuple


![Alt text](/img/underconstruction.png)
ウィンドウ領域に保存するTupleは、集計に必要なフィールドのみを選択しています。
スライドに使用する領域は現在メモリになっていますが、将来的に外部DBへの差し替えを予定。

### 関数

`SLIDE`句では下記関数が使用できます。

* <a href="#total_functions">集計関数</a>
* <a href="#edit_functions">編集関数</a>
* <a href="#operation_functions">四則演算</a>
* <a href="#math_functions">数学関数</a>
* <a href="#distance_functions">距離計算</a>
* <a href="#list_functions">リストフィールド操作関数</a>
* <a href="#stack_functions">スタック関数</a>

---

## SNAPSHOT

一定間隔による集計を実行します。間隔は時間もしくはTuple数を指定することが可能です。

    SNAPSHOT EVERY period expr

* periodには、時間・Tuple数を指定します。
* exprには、集計関数や編集関数、四則演算、フィールドアクセサを指定します。

### 時間ベースでの実行

* cron形式でも指定することができます。
* 集計値は指定時間毎にリセットされます。

cronの書式は、[Quartz](http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/crontrigger)の書式を採用しています。

> Example: 7分毎に実行
>
    SNAPSHOT EVERY 7min sum(field1) AS new_field
    SNAPSHOT EVERY "* */7 * * * ? *" sum(field1) AS new_field

集計はTupleが送られてくる度に実行されますが、集計結果は７分に１回だけ次の処理に送られます。
下段の例は、時間をcron形式で指定したものです。（注：ダブルクォートでくくる必要があります）
７分の間に１度もTupleが送られてこなかった場合は、集計結果が生成されない為、Tupleは次の処理に送られません。
集計結果は７分毎にリセットされます。

> Example: 毎日0時に日次の集計結果を返す
>
    SNAPSHOT EVERY "0 0 0 * * ? *" sum(bbb) AS s, count() AS c

### Tuple数ベースでの実行

* 指定したTuple数の集計を実行し、後続の処理に結果を送ります。

> Example:
>
    SNAPSHOT EVERY 10 sum(field1) AS new_field

指定した回数分のTupleが届いたタイミングで、集計結果が次の処理に送られます。
集計結果は10件毎にリセットされます。

### 関数

`SNAPSHOT`句では下記関数が使用できます。

* <a href="#total_functions">集計関数</a>
* <a href="#edit_functions">編集関数</a>
* <a href="#operation_functions">四則演算</a>
* <a href="#math_functions">数学関数</a>
* <a href="#distance_functions">距離計算</a>
* <a href="#list_functions">リストフィールド操作関数</a>
* <a href="#stack_functions">スタック関数</a>

---

## GROUP

BEGIN GROUP ... END GROUP で囲まれたクエリを、グループで実行します。

    BEGIN GROUP BY field, ...
      [END GROUP | TO STREAM]

* field には、グループ化するフィールドの名前を指定します。

> Example: EACH をグループ毎に実行
>
    BEGIN GROUP BY user_name
    EACH user_name, count() AS gc1
    EMIT * USING mongo_persist('db1', 'col2', 'user_name');
    END GROUP
>
> user_name ごと（ユーザごと）にTupleがカウントされます。


> Example: FILTER GROUP をグループ毎に実行
>
    BEGIN GROUP BY user_name
    FILTER GROUP EXPIRE 1DAYS
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member3 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'
    EMIT * USING mongo_persist('db1', 'col3')
    END GROUP
>
> user_nameごとに（ユーザごとに）フィルタが判定されます。
> 特定のユーザが条件１と条件２を満たしているかを判定し、FILTER GROUP の状態はユーザごとに保持されます。

### GROUP のネスト

GROUPはネストできます。

> Example:
>
    EACH ...  <- グループ化せずに実行
    BEGIN GROUP BY date
      EACH ...  <- date ごとに実行される
      BEGIN GROUP BY area
        EACH ...  <- date + area ごとに実行される
      END GROUP
    END GROUP
    EACH ...  <- グループ化せずに実行

TO STREAM で、すべてのグループを解除します。

> Example:
>
    EACH ...  <- グループ化せずに実行
    BEGIN GROUP BY date
      EACH ...  <- date ごとに実行される
      BEGIN GROUP BY area
        EACH ...  <- date + area ごとに実行される
    TO STREAM
    EACH ...  <- グループ化せずに実行

END GROUP と TO STREAM は、グループ化を解除する必要がなければ省略可能です。

---

## EMIT <a name="EMIT" class="anchor"></a>

EMITは、Tupleをトポロジの外部へ出力します。

  EMIT output_field, ... USING emit_processor

* output_field には、出力するフィールド名を指定します。ワイルドカード（&#42;）を指定できます。
* emit_processor には、出力に使用するプロセッサを指定します。

> Example:
>
    EMIT field1, field2, field3 USING mongo_persist('db1', 'col1')

### 外部への出力

#### Kafka Emit Processor <a name="KAFKA_EMIT" class="anchor"></a>

TupleをKafkaに出力します。

    kafka_emit(topic_name[, mode])


* topic_name には、出力するTopic名を指定します。topic_name は [プロセッサ変数](dml.html#procparam) に対応しています。
* mode には、jsonもしくはcsvを指定できます。省略した場合はjsonが適用されます。

> Example:
>
    kafka_emit('topic1')

また、Kafka上の書き出しは標準ではJSONになりますが、CSVでの書き出しも可能です。
このとき、出力の並びはEXPLAINで確認できる並びとなります。

> Example:
>
    EMIT * USING kafka_emit('topic1', 'csv');


#### Mongo Persist Processor <a name="MONGO_PERSIST" class="anchor"></a>

TupleをMongoDBに出力します。

    mongo_persist(db_name, collection_name [, index [, key_names]])


* db_name には、出力するDB名を指定します。db_name は [プロセッサ変数](dml.html#procparam) に対応しています。
* collection_name には、出力するCollection名を指定します。collection_name は [プロセッサ変数](dml.html#procparam) に対応しています。
* index には、`true`もしくは`false`を指定します。対象のcollectionにインデックスを作成します。省略するとインデックスは作成されません。
* key_names には、出力するキーのフィールド名を指定します。複合キーの場合は配列で指定してください。
 key_names を指定した場合、出力はキーに対してupdateされます。
 key_names を指定しなかった場合は、出力はinsertになります。

> Example:
>
    mongo_persist('db1', 'col1')  <- insert
    mongo_persist('db1', 'col1', true, 'field2') <- field2 をキーとしてupdate、インデックスを作成
    mongo_persist('db1', 'col1', false, ['field2', 'field3']) <- field2 + field3 を複合キーとしてupdate、インデックスを作成しない

#### JDBC Persist Processor <a name="JDBC_PERSIST" class="anchor"></a>

TupleをRDBに出力します。 

    jdbc_persist(connect, user, password, query)

* connect には、RDBへの接続文字列を指定します。
* user には、RDBにログインする為のユーザを指定します。
* password には、RDBにログインするユーザのパスワードを指定します。
* query には、RDBに対して実行するクエリを指定します。`@フィールド名`もしくは`@フィールド名!`で、クエリにTupleのフィールドを組み込むことができます(プレースホルダ)。`@`をクエリに含む場合は、`\@`のようにエスケープしてください。

> Example:
>
    EMIT * USING jdbc_persist(
      'jdbc:mysql://localhost/db1',
      'gennai',
      'gennai',
      'INSERT INTO output (field1, field2) VALUES (@f1, @f2)'
    )

#### Web Emit Processor

Tupleを他のRESTサーバに向けて送信します。

    web_emit(url_name)


 * url_name には、出力する先のURLを指定します。

現在、Elasticsearchに格納する場合、第二引数に"es"を指定すると、[blukインサート](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/bulk.html) に対応した書式で投げ込みます(第三引数にメタデータを指定して下さい)。

> Example:
>
    EMIT * USING web_emit('http://localhost:9200/_bulk', 'es', {'index':'test', 'type':'xyz'});

### 内部への出力 <a name="EMIT_INTERNAL" class="anchor"></a>

#### Schema Persist Processor

Tupleを同一アカウントの他のスキーマに出力します。

    EMIT fields TO schema_name;

* schema_name には、出力先のスキーマの名称を指定します。
* 出力先のスキーマと型、順序を一致させる必要があります。

> Example: スキーマ定義
>
    CREATE TUPLE tuple1 (aaa STRING, bbb INT, ccc INT, ddd STRING, _time);
    CREATE TUPLE tuple2 (aaa STRING, bbb INT, ccc INT, ddd STRING);
    CREATE TUPLE tuple3 (bbb INT, ccc INT, ddd STRING);
> Example 出力側
>
    FROM tuple1, tuple2 USING kafka_spout()
    ...
    EMIT bbb, ccc, ddd TO tuple3;

> Example: 入力側
>
    FROM tuple3 USING kafka_spout()
    ...


#### プロセッサ変数 <a name="procparam" class="anchor"></a>

Emit Processor の出力先の名称には、以下のプロセッサ変数を含めることができます。

${TOPOLOGY_ID} は、起動中のTopology IDに置き換えられます。
${ACCOUNT_ID} は、Topologyを起動したユーザのAccount IDに置き換えられます。

> Example:
>
    kafka_emit('topic_${TOPOLOGY_ID}')

---

## 関数 <a name="functions" class="anchor"></a>

### 集計関数 <a name="total_functions" class="anchor"></a>

#### count

Tupleの到着数をカウントします。

    count([DISTINCT] [field])

* fieldを指定すると、指定したフィールドが存在するTupleのみカウント対象となります。
* DISTINCTを指定すると、指定したフィールドの一意なTupleをカウントします。

> Example: 到着したTupleを全てカウント
>
    EACH count() AS cnt1

> Example: 到着したTupleのうち、field1フィールドが存在するTupleをカウント
>
    EACH count(field1) AS cnt2

> Example: 到着したTupleのうち、field1フィールドが存在し、かつ一意なTupleをカウント
>
    EACH count(DISTINCT field1) AS cnt3

> Example: 複数のフィールドに対して、一意なTupleをカウント
>
    EACH count(DISTINCT concat(field1, ifnull(field2, -))) AS cnt4

#### sum

到着したフィールドの値を合計します。

    sum([DISTINCT] field)

* fieldの指定は必須です。指定したフィールドが存在するTupleの値を合計します。
* DISTINCTを指定すると、指定したフィールドの一意なTupleの値を合計します。

> Example: 到着したTupleのうち、field1フィールドが存在するTupleの値を合計
>
    EACH sum(field1) AS sum1

> Example: 到着したTupleのうち、field1フィールドが存在し、かつ一意なTupleの値を合計
>
    EACH sum(DISTINCT field1) AS sum2

#### avg

到着したフィールドの値の平均を計算します。

    avg([DISTINCT] field)

* fieldの指定は必須です。指定したフィールドが存在するTupleの値の平均を計算します。
* DISTINCTを指定すると、指定したフィールドの一意なTupleの値の平均を計算します。

> Example: 到着したTupleのうち、field1フィールドが存在するTupleの値の平均を計算
>
    EACH avg(field1) AS avg1

> Example: 到着したTupleのうち、field1フィールドが存在し、かつ一意なTupleの値の平均を計算
>
    EACH avg(DISTINCT field1) AS avg2

### 編集関数 <a name="edit_functions" class="anchor"></a>

#### ifnull

フィールドの値がNULLであれば、代替の値で置き換えます。

> Example:
>
    EACH ifnull(field1, 0) AS field1

#### concat

STRING型のフィールドの値を連結したフィールドを作成します。

> Example:
>
    EACH concat(field1, '-', field2) AS new_field

#### split

フィールドの値を区切り文字列もしくは正規表現によってリストに変換します。

    split(string str, string pattern)

* strには、対象フィールドを指定します。
* patternには、区切り文字列もしくは正規表現で指定します。

> Example:
>
    EACH split(field1, ',') AS new_list

#### regexp_extract

フィールドの値から正規表現によって値を抜き出します。

    regexp_extract(string subject, string pattern, int index)

* subjectには、対象フィールドを指定します。
* patternには、正規表現パターンを指定します。
* indexには、抜き出すグループの番号を指定します。

> Example:
>
    EACH regexp_extract(field1, '(\d{4})-(\d{2})-(\d{2})', 1) AS new_field

#### parse_url

フィールドの値(URL)をパースして、一部を返します。

    parse_url(string urlString, string partToExtract [, string keyToExtract])

* urlStringには、パースするURLフィールドを指定します。
* partToExtractには、下記のいずれかを指定します。
    * HOST
    * PATH
    * QUERY
    * REF
    * PROTOCOL
    * AUTHORITY
    * FILE
    * USERINFO
* keyToExtractには、partToExtractがQUERYの場合、取得するパラメータ名称を指定します。

> Example:
>
    EACH parse_url(urlField, 'QUERY', 'k1') AS new_field

#### cast

フィールドの値を指定した型に変換します。

    cast(expr as type)

* exprには、対象フィールドを指定します。
* typeには、下記の型のみ指定可能です。
    * TIMESTAMP
    * TINYINT
    * SMALLINT
    * INT
    * BIGINT
    * FLOAT
    * DOUBLE
    * BOOLEAN

> Example:
>
    EACH cast(field1 AS TIMESTAMP('yyyyMMddHHmmss')) AS new_field

![Alt text](/img/underconstruction.png)
定数に関しては、関数の種類が増えてきてから改めて記述する予定です。

#### date_format

TIMESTAMP型のフィールドを、フォーマットした文字列(STRING)に変換します。

    date_foramt(field AS string)

* fieldには、TIMESTAMP型のフィールドを指定します。
* stringには、パターン文字列を指定します。

パターン文字列には、[java/text/SimpleDateFormat](https://docs.oracle.com/javase/jp/6/api/java/text/SimpleDateFormat.html)と同じ書式を採用しています。

> Example:
>
    EACH date_format(field1, 'yyyy-MM-dd-HH-mm') field2

> Example: 日時文字列(yyyyMMdd)をTIMESTAMP型に変換し、日付のみ取得
>
    EACH date_format(cast(field1 AS TIMESTAMP('yyyyMMdd')) AS 'dd') AS field2

### 四則演算 <a name="operation_functions" class="anchor"></a>

数値型フィールドでは下記の四則演算が可能です。

* 加算(+)
* 減算(-)
* 乗算(\*)
* 除算(/)
* 除算(DIV)
* 剰余(MOD, %)

#### 数値型フィールド同士の演算

数値型フィールド同士での四則演算が可能です。

> Example:
>
    EACH field1 + field2 AS field3
    EACH field1 DIV field2 AS field3

#### 数値型フィールドと数値の演算

数値型フィールドと数値の四則演算が可能です。

> Example:
>
    EACH field1 * 2 AS field3

#### 関数を組み合わせた演算

数値型フィールドと集計関数を組み合わせた四則演算が可能です。

> Example:
>
    EACH sum(field1 * 10) * count() AS field3

#### 演算の優先順位

括弧を用いて、四則演算の優先順位を変更する事が可能です。

> Example:
>
    EACH field1 * (field2 + 123) AS field3

### 数学関数 <a name="math_functions" class="anchor"></a>

#### sqrt

引数で指定された数値フィールドの平方根を計算します。

> Example:
>
    EACH sqrt(field1) AS field2

#### sin

引数で指定された数値フィールドをラジアンで表した角度として、正弦(sin)を計算します。

> Example:
>
    EACH sin(field1) AS field3

#### cos

引数で指定された数値フィールドをラジアンで表した角度として、余弦(cos)を計算します。

> Example:
>
    EACH cos(field1) AS field4

#### tan

引数で指定された数値フィールドをラジアンで表した角度として、正接(tan)を計算します。

> Example:
>
    EACH tan(field1) AS field5

### 距離計算 <a name="distance_functions" class="anchor"></a>

#### distance

Double型のフィールドを、始点X座標、始点Y座標、終点X座標、終点Y座標の順に指定し、始点と終点の距離を計算します。

> Example:
>
    EACH distance(x1, y1, x2, y2) AS dis

### リストフィールド操作関数 <a name="list_functions" class="anchor"></a>

#### size

リスト型フィールドのサイズを取得します。

    size(field)

> Example:
>
    EACH size(field2) AS list_size

#### slice

リスト型フィールドから指定範囲を取得します。

    slice(field, begin[, end])

* begin には、取得開始位置を指定します。
* end には、取得完了位置を指定します。省略すると末尾まで取得します。

> Example: 3番目から最後まで取得
>
    slice(field1, 2)

> Example: 先頭から3番目まで取得
>
    slice(field1, 0, 2)

> Example: 先頭から、最後から2番目までを取得
>
    slice(field1, 0, -1)

> Example: 最後から5番目以降を取得
>
    slice(field1, -4)

### スタック関数 <a name="stack_functions" class="anchor"></a>

フィールドの値をスタックすることができます。 `SLIDE`句と`LIMIT`句でのみ使用可能です。

#### collect_list

    collect_list(field)

重複しているフィールドの値もスタックします。フィールド値は順番通りに保持されます。

#### collect_set

    collect_set(field)

重複しているフィールドの値はスタックされません。重複したフィールドの値がある場合、順番は保持されません。
