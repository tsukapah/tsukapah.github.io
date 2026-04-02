---
title: Elasticsearchクラスタにノードを追加してみる
date: '2017-06-02T19:54:55+09:00'
lastmod: '2017-06-02T19:54:55+09:00'
draft: false
tags:
- Elasticsearch
- Docker
- docker-machine
description: '[こちら](http://qiita.com/tsukapah/items/1df305026f55cdc77b4b)の記事でElasticsearchの複数台クラスタが作れるようになりました。'
params:
  qiita_url: https://qiita.com/tsukapah/items/7b976cb2c59941875ec2
  qiita_id: 7b976cb2c59941875ec2
---

[こちら](http://qiita.com/tsukapah/items/1df305026f55cdc77b4b)の記事でElasticsearchの複数台クラスタが作れるようになりました。
せっかくなので、あとからノードを追加する検証をしてみます。

環境は[こちら](http://qiita.com/tsukapah/items/1df305026f55cdc77b4b)を再利用します。

# 検証手順
## 現状の確認をする

```bash
$ curl $(docker-machine ip elasticsearch1):9200/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
$ curl $(docker-machine ip elasticsearch1):9200/_cat/shards?v
index shard prirep state   docs store ip             node
test  2     r      STARTED    0  130b 192.168.99.105 bmXE705
test  2     p      STARTED    0  130b 192.168.99.106 fyEAVyY
test  1     p      STARTED    0  130b 192.168.99.104 efZ1klG
test  1     r      STARTED    0  130b 192.168.99.106 fyEAVyY
test  4     r      STARTED    0  130b 192.168.99.105 bmXE705
test  4     p      STARTED    0  130b 192.168.99.104 efZ1klG
test  3     p      STARTED    0  130b 192.168.99.105 bmXE705
test  3     r      STARTED    0  130b 192.168.99.104 efZ1klG
test  0     p      STARTED    0  130b 192.168.99.105 bmXE705
test  0     r      STARTED    0  130b 192.168.99.106 fyEAVyY
```

インデックス作成時に特にシャーディングについての設定を実施していないので、
testインデックスのプライマリシャードは5、レプリカシャードは1でインデックスが作成されており、
各シャードは以下のように配置されています。

※プライマリシャード=PS、レプリカシャード=RS

|ホスト|PS0|PS1|PS2|PS3|PS4|RS0|RS1|RS2|RS3|RS4|
|---|---|---|---|---|---|---|---|---|---|---|
|elasticsearch1<br>(192.168.99.104)||○|||○||||○||
|elasticsearch2<br>(192.168.99.105)|○|||○||||○||○|
|elasticsearch3<br>(192.168.99.106)|||○|||○|○||||

## 追加のコンテナホストを起動する

それではElasticsearchクラスタにノードを追加してみます。
まずはコンテナホストを起動します。

```bash
$ docker-machine create --driver virtualbox elasticsearch4
$ docker-machine ssh elasticsearch4 sudo sysctl -w vm.max_map_count=262144
```

## Elasticsearchコンテナを起動する

ノードが増えるので起動オプションの`discovery.zen.ping.unicast.hosts`と`discovery.zen.minimum_master_nodes`の値を追加します。

```bash
$ eval $(docker-machine env elasticsearch4)
$ docker run -d \
  -e "network.host=_eth1:ipv4_" \
  -e "discovery.zen.ping.unicast.hosts=$(docker-machine ip elasticsearch1),$(docker-machine ip elasticsearch2),$(docker-machine ip elasticsearch3)" \
  -e "discovery.zen.minimum_master_nodes=3" \
  -e "xpack.security.enabled=false" \
  -e "xpack.monitoring.enabled=false" \
  -e "xpack.watcher.enabled=false" \
  -e "xpack.graph.enabled=false" \
  -e "xpack.ml.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  --network="host" \
  -p 9200:9200 \
  -p 9300:9300 \
  docker.elastic.co/elasticsearch/elasticsearch:5.4.0
```

## 追加結果を確認する

まずはクラスタの状態を確認します。

```bash:クラスタのステータスを確認
$ curl $(docker-machine ip elasticsearch1):9200/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 4,
  "number_of_data_nodes" : 4,
  "active_primary_shards" : 5,
  "active_shards" : 10,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

以下になっていれば新しいノードがクラスタに参加できています。

+ statusがgreen
+ ノードが3→4
+ データノードが3→4

続いて作成済みのインデックスの状態を確認します。

```bash:インデックスの状態を確認
$ curl $(docker-machine ip elasticsearch1):9200/_cat/shards?v
index shard prirep state   docs store ip             node
test  2     r      STARTED    0  130b 192.168.99.107 TZj8_YT
test  2     p      STARTED    0  130b 192.168.99.106 fyEAVyY
test  1     p      STARTED    0  130b 192.168.99.104 efZ1klG
test  1     r      STARTED    0  130b 192.168.99.106 fyEAVyY
test  4     r      STARTED    0  130b 192.168.99.105 bmXE705
test  4     p      STARTED    0  130b 192.168.99.104 efZ1klG
test  3     p      STARTED    0  130b 192.168.99.105 bmXE705
test  3     r      STARTED    0  130b 192.168.99.104 efZ1klG
test  0     p      STARTED    0  130b 192.168.99.107 TZj8_YT
test  0     r      STARTED    0  130b 192.168.99.106 fyEAVyY
```

以下のように再配置されていることが確認できました。

|ホスト|PS0|PS1|PS2|PS3|PS4|RS0|RS1|RS2|RS3|RS4|
|---|---|---|---|---|---|---|---|---|---|---|
|elasticsearch1<br>(192.168.99.104)||○|||○||||○||
|elasticsearch2<br>(192.168.99.105)||||○||||||○|
|elasticsearch3<br>(192.168.99.106)|||○|||○|○||||
|elasticsearch4<br>(192.168.99.107)|○|||||||○|||

# まとめ
適切にパラメータを指定して起動すれば勝手にノード追加してシャードの再配置までしてくれるのは便利です。
データの永続化をしていないので既存ノードのパラメータを修正できませんでしたが、
永続化してローリングアップデートの要領で`docker stop`と`docker run`を繰り返していけば既存コンテナのパラメータ変更もできそうです。

