---
layout: manual_ja
title: 環境準備 / genn.ai
---

# genn.ai 環境準備

> ここでは、genn.aiを使用する為の環境構築の準備について記載します。詳細な設定については[設定](/ja/config.html)/[設定例](/ja/example.html)を参照してください。

genn.aiを利用する環境を準備するには下記の方法があります。

* [ソースコンパイル](#compile)
* [Vagrantを利用](#vagrant)
  * [Vagrantfileから構築](#vagrantfile)
  * [公開Vagrant Boxを利用](#vagrantbox)
* [Dockerを利用](#docker)
  * [公開Dockerイメージを利用](#dockerimage)
  * [Dockerfileから構築](#dockerfile)
    * [all in one](#docker-allinone)
    * [standalone](#docker-standalone)
    * [cluster](#docker-cluster)
* [EC2@AWSを利用](#ec2)
  * [コミュニティAMIを利用](#public-ami)

## ソースコンパイル <a name="compile" class="anchor"></a>

最新版のソースを取得しコンパイルします。JDK、Mavenを必要とします。

    $ git clone https://github.com/TryGennai/gennai
    $ cd gennai/gungnir
    $ mvn clean package -DskipTests=true

利用するモードによりますが、MongoDB, ZooKeeper, Kafka, Stormを別途インストールする必要があります。

詳細は[こちら](https://github.com/TryGennai/gennai/tree/master/gungnir#getting-started-with-gennai)を参照してください。

## Vagrantを利用 <a name="vagrant" class="anchor"></a>

### Vagrantfileから構築 <a name="vagrantfile" class="anchor"></a>

ローカルのVagrantにgenn.ai環境を構築します。

    $ git clone https://github.com/TryGennai/gennai.vagrant
    $ cd gennai.vagrant
    $ vagrant up

詳細は[こちら](https://github.com/TryGennai/gennai.vagrant)を参照してください。

### 公開Vagrant Boxを利用 <a name="vagrantbox" class="anchor"></a>

ローカルのVagrantに既に環境準備されたBoxをダウンロードして起動します。

    $ vagrant box add siniida/gennai
    $ vagrant init siniida/gennai
    $ vagrant up
    $ vagrant ssh

起動されたgenn.ai環境は、[疑似分散モード](/ja/config.html#mode.pseudo)で起動しています。

## Dockerを利用 <a name="docker" class="anchor"></a>

### Dockerイメージを利用 <a name="dockerimage" class="anchor"></a>

DockerHubからイメージをダウンロードして利用します。

    $ docker pull siniida/gennai
    $ docker run -t -i siniida/gennai su - gennai

※ コンテナには十分なメモリを割り当ててください。

コンテナを起動した後は各種サービスを起動する必要があります。

    [gennai@{CONTAINER ID} ~]$ sudo service mongod start
    [gennai@{CONTAINER ID} ~]$ sudo service zookeeper start
    [gennai@{CONTAINER ID} ~]$ sudo serivce kafka start
    [gennai@{CONTAINER ID} ~]$ sudo service storm-nimbus start
    [gennai@{CONTAINER ID} ~]$ sudo service storm-supervisor start
    [gennai@{CONTAINER ID} ~]$ sudo service gungnir-server start
    [gennai@{CONTAINER ID} ~]$ sudo service tuple-store-server start

以降、gungnirコマンドを使用して、gennaiを使用することができます。

ここで起動されるgenn.ai環境は[疑似分散モード](/ja/config.html#mode.pseudo)です。必要に応じて設定を変更し各種サービスを再起動してください。


### Dockerfileから構築 <a name="dockerfile" class="anchor"></a>

`git clone`したDockefileからDockerイメージを構築します。Dockerfileは下記の3種類があります。使用状況に合わせて選択してください。

* **all in one** : 1つのDockerコンテナ上で全てのサービスを稼動
* **standalone** : 1つのDockerコンテナ上でstandaloneモードで稼動
* **cluster** : 複数のDockerコンテナ上にサービスを稼動(docker-composeが必要)

#### all in one <a name="docker-allinone" class="anchor"></a>

    $ git clone https://github.com/siniida/gennai.docker
    $ cd gennai.docker
    $ docker build -t gennai .
    $ docker run -ti --rm gennai /bin/bash

※ コンテナには十分なメモリを割り当ててください。

コンテナを起動した後は各種サービスを起動する必要があります。

    [root@{CONTAINER ID} /]# service mongod start
    [root@{CONTAINER ID} /]# service zookeeper start
    [root@{CONTAINER ID} /]# service kafka start
    [root@{CONTAINER ID} /]# service storm-nimbus start
    [root@{CONTAINER ID} /]# service storm-supervisor start
    [root@{CONTAINER ID} /]# service gungnir-server start
    [root@{CONTAINER ID} /]# service tuple-store-server start
    [root@{CONTAINER ID} /]# /opt/gungnir-client/bin/gungnir -u root -p gennai

ここで起動されるgenn.ai環境は[疑似分散モード](/ja/config.html#mode.pseudo)です。必要に応じて設定を変更し各種サービスを再起動してください。

#### standalone <a name="docker-standalone" class="anchor"></a>

    $ git clone https://github.com/siniida/gennai.docker
    $ cd gennai.docker
    $ git checkout standalone
    $ docker build -t standalone .
    $ docker run -ti --rm standalone
    Pidfile: ./../gungnir-server.pid
    Using config file: ./../conf/gungnir-standalone.yaml
    Starting Gungnir server ... STARTED
    Gungnir server connecting ...
    Gungnir version 0.0.1 build at 20150612-102002
    Welcome root (Account ID: 45b6fefc487547539f720ac974f2157c)
    gungnir>

ここで起動されるgenn.ai環境は[ローカルモード](/ja/config.html#mode.local)です。いくつかの制限事項もありますので注意してください。

#### cluster <a name="docker-cluster" class="anchor"></a>

    $ git clone https://github.com/siniida/gennai.docker
    $ cd gennai.docker
    $ git checkout cluster
    $ docker-compose up

ここで起動されるgenn.ai環境は[完全分散モード](/ja/config.html#mode.distributed)です。

別ターミナルにてクライアントを実行します。

    $ ./client.sh

## EC2@AWSを利用 <a name="ec2" class="anchor"></a>

### コミュニティAMIを利用 <a name="public-ami" class="anchor"></a>

インスタンスの作成時、『コミュニティAMI』からキーワード『gennai』で検索し、『gennai-public-YYYYMMDD』のAMIを使用して起動してください。

※ インスタンスタイプは十分なメモリを搭載したlarge以上を選択してください。

![コミュニティAMI](/img/public-ami.png)

起動後は、gennaiユーザにてログインが可能です。

    $ ssh -i [user.pem] gennai@[public]

デフォルト設定では[疑似分散モード](/ja/config.html#mode.pseudo)で起動されるgenn.ai環境をEC2において利用可能です。

必要に応じて設定を変更し各種サービスを再起動してください。
