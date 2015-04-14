---
layout: manual_ja
title: DDL / genn.ai
redirect_from: "/ddl_ja.html"
---

# genn.ai DDL

{% include lead.md %}

## CREATE TUPLE <a name="CREATE_TUPLE" class="anchor"></a>

Tupleのスキーマを定義します。

スキーマは

- Tupleの受け取り可否の判定
- JSONTupleからGungnirTuple(genn.ai内部でのタプル表現)への変換
- FROM statements での入力対象の指定

に使用されます。

    CREATE TUPLE schema_name
        (field_name [field_type], ...)
        [PARTITIONED BY parition_field, ...]
        [COMMENT comment]

* schema_name には、Tuple名を指定します。
* field_name には、フィールド名を指定します。
* field_type には、フィールドの型を指定します。省略した場合は、自動判定になります。
* parition_field には、Tupleをパーティション単位に処理する為に、パーティションキーとなるフィールド名を指定します。
PARTITIONED BY clause を省略した場合は、Tupleをランダムに振り分けて処理します。
* comment には、任意の文字列で設定することができます。

> Example:
>
    CREATE TUPLE userAction1 (
        field1 INT,
        field2 STRING,
        field3 STRING,
        field4 LIST<STRING>,
        field5 TINYINT,
        field6 STRUCT<member1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), member2 LIST<BIGINT>>,
        field7 MAP<STRING, BOOLEAN>,
        field8 BOOLEAN,
        field9 TIMESTAMP,
        field10 MAP<INT, STRING>
    )
    PARTITIONED BY field1

### Field Types

フィールドの型には、以下のものを指定します。

* Numeric Types

    TINYINT, SMALLINT, INT, BIGINT, FLOAT, DOUBLE

* Date/Time Types

    TIMESTAMP

* String Types

    STRING

* Misc Types

    BOOLEAN

* Complex Types

    LIST, MAP, STRUCT

#### TINYINT

GungnirTupleでは、JavaのByteとして扱われます。

JSONTupleでは、フィールドの値を数字で記述します。

#### SMALLINT

GungnirTupleでは、JavaのShortとして扱われます。

JSONTupleでは、フィールドの値を数字で記述します。

#### INT

GungnirTupleでは、JavaのIntegerとして扱われます。

JSONTupleでは、フィールドの値を数字で記述します。
このとき該当のフィールドの型をスキーマで省略していた場合は、
記述した数字の桁数に応じて、フィールド値はINT（Integer）もしくはBIGINT（Long）型に変換されます。

#### BIGINT

GungnirTupleでは、JavaのLongとして扱われます。

JSONTupleでは、フィールドの値を数字で記述します。
このとき該当のフィールドの型をスキーマで省略していた場合は、
記述した数字の桁数に応じて、フィールド値はINT（Integer）もしくはBIGINT（Long）型に変換されます。

#### FLOAT

GungnirTupleでは、JavaのFloatとして扱われます。

JSONTupleでは、フィールドの値を数字（小数）で記述します。

#### DOUBLE

GungnirTupleでは、JavaのDoubleとして扱われます。

JSONTupleでは、フィールドの値を数字（小数）で記述します。
このとき該当のフィールドの型をスキーマで省略していた場合は、
フィールド値はDOUBLE型（Doubleに変換されます。

#### TIMESTAMP

* TIMESTAMP

GungnirTupleでは、Javaのjava.util.Dateとして扱われます。

JSONTupleでは、フィールドの値をepoch time（数字）で記述します。


> Example:
>
    field:1382086720

* TIMESTAMP (date_format)

date_format には、日時のフォーマットを指定します。
Javaの[java.text.SimpleDateFormatと同じ書式](
http://docs.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html
)を採用しています。

GungnirTupleでは、Javaのjava.util.Dateとして扱われます。

> Example:
>
    field TIMESTAMP('yyyy-MM-dd HH:mm:ss')

JSONTupleでは、フィールドの値をdate_format で指定したフォーマットで記述します。

> Example:
>
    field:"2013-10-18 18:07:25"

#### STRING

GungnirTupleでは、JavaのStringとして扱われます。

JSONTupleでは、フィールドの値をダブルクォートでくくった文字列で記述します。

#### BOOLEAN

GungnirTupleでは、JavaのBooleanとして扱われます。

JSONTupleでは、フィールドの値を true もしくは false で記述します。

#### LIST

GungnirTupleでは、Javaのjava.util.Listとして扱われます。
LISTの要素の型には、Numeric Types、TIMESTAMP、STRING、BOOLEAN のいずれかを指定します。

> Example:
>
    field LIST<STRING>

JSONTupleでは、フィールドの値をArrays構造で記述します。

> Example:
>
    field:["tokyo","kyoto","osaka"]

#### MAP

GungnirTupleでは、Javaのjava.util.Mapとして扱われます。
MAPのキーと値の型には、Numeric Types、TIMESTAMP、STRING、BOOLEAN のいずれかを指定します。

> Example:
>
    field MAP<STRING, INT>


JSONTupleでは、フィールドの値をObjects構造で記述します。

> Example:
>
    field:{"078-8220":1, "061-3601":2, "127-0001":3}

キーの型にNumeric Typesを指定した場合は、値をダブルクォートでくくる必要があります。

> Example:
>
    field MAP<INT, DOUBLE> --> field:{"1":0.6, "200":2.3, "560":3.0}

#### STRUCT

GungnirTupleでは、org.gennai.gungnir.tuple.Struct（構造体クラス）として扱われます。
なお、現在は以下の制約があります。

- 値の書き出し(新しい構造体フィールドをAS句で作る、`JOIN`のTOで作る等)はできません。
- [TupleStoreServeから読み込んで結合](dml.html#RestJoin)および[ストリームから読み込んで結合](dml.html#StreamJoin)の条件に使うことはできません。
- [ストリームから読み込んで結合](dml.html#StreamJoin)には、事前にSTRING型等へ書き出しておくことで条件に使用することは可能です。

> Example:
>
    field STRUCT<member0 STRING, member1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), member2 LIST<BIGINT>>


JSONTupleでは、フィールドの値をObjects構造で記述します。

> Example:
>
    field:{member0:"gennai", member1:"2013-10-18 18:28:34", member2:[10000, 20000, 30000]}

### JSONTupleの記述例

上記の userAction1 をJSONTupleで記述した例です。

    {
      field1:12345,
      field2:"xxx",
      field3:"yyy",
      field4:["aaa","bbb","ccc"],
      field5:1,
      field6:{
        member1:"2013-10-18 18:28:34",
        member2:[10000, 20000, 30000]
      },
      field7:{"A":true, "B":false, "C":false},
      field8:true,
      field9:1382086720,
      field10:{"11":"lll", "22":"mmm"}
    }

### 特殊なフィールド

スキーマには、以下の特殊なフィールドを定義することができます。

* &#x5f;tid
* &#x5f;tno
* &#x5f;time

これらの名称はフィールドの予約名なので、通常のフィールド名には使用しないでください。

* &#x5f;tid

    Tracking ID を格納するフィールドです。
    スキーマのフィールドに &#x5f;tid を記述すると、Tracking ID値がTupleに挿入されます。（STRING型になります）

* &#x5f;tno

    Tracking No を格納するフィールドです。
    スキーマのフィールドに &#x5f;tno を記述すると、Tracking No値がTupleに挿入されます。（INT型になります）

* &#x5f;time

    Tupleの受付時間を格納するフィールドです。
    スキーマのフィールドに &#x5f;time を記述すると、受付時間（受付時の現在時間）がTupleに挿入されます。（TIMESTAMP型になります）

> Example:
>
    CREATE TUPLE userAction1 (
        _tno,
        _tid,
        _time,
        field1 INT,
        field2 STRING,
        field3 STRING,
        field4 LIST<STRING>,
        field5 TINYINT,
        field6 STRUCT<member1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), member2 LIST<BIGINT>>,
        field7 MAP<STRING, BOOLEAN>,
        field8 BOOLEAN,
        field9 TIMESTAMP,
        field10 MAP<INT, STRING>
    )
    PARTITIONED BY _tno

### Tracking ID と Tracking No

Tracking ID は、Tupleの投入元を特定する為に使用する一意なID（24桁の文字列）です。
Tracking Noは、Tracking Noを数値化した値（連番）です。

Tracking ID 及び Tracking No は、スキーマに &#x5f;tid もしくは &#x5f;tno フィールドを記述したJSONTupleを受け付けると、自動で生成されます。
生成された Tracking ID と Tracking No は、Tupleに挿入されるとともに、Set-Cookieヘッダで投入元に返却されます。（返却されるのは Tracking ID のみになります）
投入元は、次回のJSONTupleの投入時に、受け取ったTracing ID をCookieヘッダで送信します。
Cookieヘッダで送信された Tracking ID から、JSONTupleが同一の投入元から送られてきたかどうかを判断できます。
送信された Tracking ID で Tracking No を検索し、_tid と &#x5f;tno はTupleに挿入されます。

Tracking ID と Tracking Noは、いずれもTupleの同一性の判定に使用できますが、Tracking No の方が、より視覚的に判断しやすくなっています。

---

## SHOW TUPLES

定義されているTupleスキーマの一覧を表示します。

    SHOW TUPLES

結果はJSONで出力されます。

> Result:
>
    [
      {
        "name":"userAction1",
        "topologies":["547d88dd0cf2c20a93d59e9f"],
        "owner":"user@genn.ai",
        "createTime":"2013-10-18T02:14:00.241Z",
        "comment": "comment for tuple."
      },
      {
        "name":"userAction2",
        "topologies":[],
        "owner":"user@genn.ai",
        "createTime":"2013-10-17T02:16:34.898Z"
      }
    ]

* name は、Tuple名です。
* topologies は、該当のTupleを使用しているTopologyIDの配列です。
* owner は、Tupleを作成したユーザのユーザ名です。
* createTime は、Tupleを作成した日時です。
* comment は、Tupleに設定したコメントです。

---

## DESC TUPLE

定義されているTupleスキーマの情報を表示します。

    DESC TUPLE schema_name


* schema_name には、Tuple名を指定します。

> Example:
>
    DESC TUPLE userAction1


結果はJSON形式で出力されます。

> Result:
>
    {
      "name":"userAction2",
      "fields":{
        "field1":{"type":"BIGINT"},
        "field2":{"type":"STRING"},
        "field3":{"type":"STRING"}
      },
      "topologies": ["547d88dd0cf2c20a93d59e9f"],
      "partitioned":["field1"],
      "owner":"user@genn.ai",
      "createTime":"2013-09-13T01:35:55.667Z",
      "comment": "comment for tuple."
    }


* name は、Tuple名です。
* fields は、Tupleのフィールドの一覧です。
* topologies は、該当のTupleを使用しているTopologyIDの配列です。
* partitioned は、Tupleのパーティションキーフィールドです。
* owner は、Tupleを作成したユーザのユーザ名です。
* createTime は、Tupleを作成した日時です。
* comment は、Tupleに設定されているコメントです。

---

## DROP TUPLE

Tupleスキーマを削除します。
どのトポロジからも使われてないもののみ削除可能です。

    DROP TUPLE schema_name

* schema_name には、Tuple名を指定します。

> Example:
>
    DROP TUPLE userAction1


---

## CREATE VIEW

TupleのViewを定義します。Tupleスキーマを別名で定義できます。


    CREATE VIEW view_schema_name AS FROM tuple_schema_name FILTER condition [COMMENT comment];


* view_schema_name には、View名を指定します。
* tuple_schema_name には、元となるTuple名を指定します。
* condition には、TupleをViewにひもづける条件を指定します。
* comment には、任意の文字列で設定することができます。

> Example:
>
    CREATE VIEW viewAction1 AS FROM userAction1 FILTER field3 = 'CATEGORY-1'
    CREATE VIEW viewAction2 AS FROM userAction1 FILTER field3 = 'CATEGORY-2'
    CREATE VIEW viewAction3 AS FROM userAction1 FILTER field3 = 'CATEGORY-3'

userAction1のTupleスキーマをもとに、field3フィールドの値ごとに３つのviewを定義しています。

---

## SHOW VIEWS

定義されているViewの一覧を表示します。

    SHOW VIEWS


結果はJSONで出力されます。

> Result:
>
    [
      {
        "name":"viewAction1",
        "topologies": ["547d88dd0cf2c20a93d59e9f"],
        "owner":"user@genn.ai",
        "createTime":"2013-10-19T03:19:22.241Z"
      },
      {
        "name":"viewAction2",
        "topologies": ["547d88dd0cf2c20a93d59e9f"],
        "owner":"user@genn.ai",
      　"createTime":"2013-10-19T03:19:56.898Z"
      },
      {
        "name":"viewAction3",
        "topologies": [],
        "owner":"user@genn.ai",
        "createTime":"2013-10-19T03:19:34.898Z",
        "comment": "comment for view."
      }
    ]


* name は、View名です。
* topologies は、該当のViewを使用しているTopologyIDの配列です。
* owner は、Viewを作成したユーザのユーザ名です。
* createTime は、Viewを作成した日時です。
* comment は、Viewに設定したコメントです。

---

## DESC VIEW

定義されているViewの情報を表示します。

    DESC VIEW schema_name

* schema_name には、View名を指定します。

> Example:
>
    DESC VIEW viewAction1


結果はJSON形式で出力されます。

> Result:
>
    {
      "name":"viewAction1",
      "from":"userAction1",
      "filter":"field3 = CATEGORY-1",
      "topologies": [],
      "owner":"user@genn.ai",
      "createTime":"2013-10-19T03:19:22.241Z",
      "comment": "comment for view."
    }

* name は、View名です。
* from は、Viewの元となるTuple名です。
* filter は、TupleをViewにひもづけている条件です。
* topologies は、該当のViewを使用しているTopologyIDの配列です。
* owner は、Viewを作成したユーザのユーザ名です。
* createTime は、Viewを作成した日時です。
* comment は、Viewに設定されているコメントです。

---

## DROP VIEW

Viewを削除します。

    DROP VIEW schema_name

* schema_name には、View名を指定します。

> Example:
>
    DROP VIEW viewAction1

