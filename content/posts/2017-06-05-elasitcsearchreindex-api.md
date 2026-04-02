---
title: ElasitcsearchのReindex APIを使ってみた
date: '2017-06-05T16:55:29+09:00'
lastmod: '2017-12-28T16:02:44+09:00'
draft: false
tags:
- Elasticsearch
- Docker
- docker-machine
description: '[Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-reindex.html)が便利そうだったので検証してみました。'
params:
  qiita_url: https://qiita.com/tsukapah/items/9cb26539bd59da983896
  qiita_id: 9cb26539bd59da983896
---

[Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-reindex.html)が便利そうだったので検証してみました。

# 検証環境
+ Mac OS X El Capitan(10.11.6)
+ Docker version 17.03.1-ce, build c6d412e
+ docker-machine version 0.10.0, build 76ed2a6
+ VirtualBox 5.1.14 r112924 (Qt5.6.2)

# 検証開始
## コンテナホストの起動
```bash
$ docker-machine create elasticsearch --driver virtualbox
```

## Elasticsearchコンテナの起動

```bash
$ eval $(docker-machine env elasticsearch)
$ docker run -d \
	-e "http.host=0.0.0.0" \
	-e "transport.host=127.0.0.1" \
	-e "xpack.security.enabled=false" \
	-e "xpack.monitoring.enabled=false" \
	-e "xpack.watcher.enabled=false" \
	-e "xpack.graph.enabled=false" \
	-e "xpack.ml.enabled=false" \
	-e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
	-p 9200:9200 \
	-p 9300:9300 \
	--name elasticsearch \
	docker.elastic.co/elasticsearch/elasticsearch:5.4.1
$ curl $(docker-machine ip elasticsearch):9200
{
  "name" : "3VeUTYk",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "BCo9uLM_SZqThWdYIMO9AA",
  "version" : {
    "number" : "5.4.1",
    "build_hash" : "2cfe0df",
    "build_date" : "2017-05-29T16:05:51.443Z",
    "build_snapshot" : false,
    "lucene_version" : "6.5.1"
  },
  "tagline" : "You Know, for Search"
}
```

## インデックスの作成
ひとまず細かい設定はしない状態のインデックスを作成してみます。

```bash
$ curl -XPUT $(docker-machine ip elasticsearch):9200/test1?pretty
{
  "acknowledged" : true,
  "shards_acknowledged" : true
}
$ curl $(docker-machine ip elasticsearch):9200/_cat/shards?v
index shard prirep state      docs store ip        node
test1 3     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 3     r      UNASSIGNED
test1 4     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 4     r      UNASSIGNED
test1 1     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 1     r      UNASSIGNED
test1 2     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 2     r      UNASSIGNED
test1 0     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 0     r      UNASSIGNED
```

インデックス *test1* が作成され、5つのプライマリシャードが作成できました。
1ノードクラスタなのでレプリカシャードはUNASSIGNEDです。

## データ投入
まずは1件データを投入してみたいと思うので、投入用のデータを作成します。

```json:test1data.json
{
    "user" : "tsukapah",
    "message" : "I have a pen."
}
```

このドキュメントをtest1インデックスに投入します。

```bash
# データ投入
$ curl -XPOST $(docker-machine ip elasticsearch):9200/test1/ppap?pretty --data @test1data.json
{
  "_index" : "test1",
  "_type" : "ppap",
  "_id" : "AVx2XNnewZBo_GQ4tz7S",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "created" : true
}
# データの確認
$ curl $(docker-machine ip elasticsearch):9200/test1/ppap/AVx2XNnewZBo_GQ4tz7S?pretty
{
  "_index" : "test1",
  "_type" : "ppap",
  "_id" : "AVx2XNnewZBo_GQ4tz7S",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "user" : "tsukapah",
    "message" : "I have a pen."
  }
}
```

入りました。これからこのインデックスを作り直したいと思います。

# インデックステンプレートの作成

作り直すインデックスにはインデックステンプレートを適用してみたいと思います。
プライマリシャードを6つにしてみようと思います。

新しいインデックス名：test2
プライマリシャードの数：6

```json:template.json
{
    "template" : "test2",
    "version": 1,
    "settings": {
      "number_of_shards": 6
    }
}
```

テンプレートを適用してみます。

```bash
# テンプレートの適用
$ curl $(docker-machine ip elasticsearch):9200/_template/test2 --data @template.json
{"acknowledged":true}
# テンプレートの確認
$ curl $(docker-machine ip elasticsearch):9200/_template?pretty
{
  "test2" : {
    "order" : 0,
    "version" : 1,
    "template" : "test2",
    "settings" : {
      "index" : {
        "number_of_shards" : "6"
      }
    },
    "mappings" : { },
    "aliases" : { }
  }
}
```

# インデックスの再作成

Reindexする処理内容をjsonで定義します。
test1からtest2にインデックスを移し替えるので以下のようになります。

```json:reindex.json
{
  "source": {
    "index": "test1"
  },
  "dest": {
    "index": "test2"
  }
}
```

Reindexしてみます。

```bash
$ curl -XPOST $(docker-machine ip elasticsearch):9200/_reindex?pretty --data @reindex.json
{
  "took" : 308,
  "timed_out" : false,
  "total" : 1,
  "updated" : 0,
  "created" : 1,
  "deleted" : 0,
  "batches" : 1,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```

新しいインデックスにドキュメントがあることを確認します。

```bash
$ curl $(docker-machine ip elasticsearch):9200/test2/ppap/AVx2XNnewZBo_GQ4tz7S?pretty
{
  "_index" : "test2",
  "_type" : "ppap",
  "_id" : "AVx2XNnewZBo_GQ4tz7S",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "user" : "tsukapah",
    "message" : "I have a pen."
  }
}
```

シャードの状態を確認します。

```bash
$ curl $(docker-machine ip elasticsearch):9200/_cat/shards?v
index shard prirep state      docs store ip        node
test1 3     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 3     r      UNASSIGNED
test1 4     p      STARTED       1   4kb 127.0.0.1 3VeUTYk
test1 4     r      UNASSIGNED
test1 1     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 1     r      UNASSIGNED
test1 2     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 2     r      UNASSIGNED
test1 0     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test1 0     r      UNASSIGNED
test2 5     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test2 5     r      UNASSIGNED
test2 3     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test2 3     r      UNASSIGNED
test2 4     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test2 4     r      UNASSIGNED
test2 1     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test2 1     r      UNASSIGNED
test2 2     p      STARTED       1   4kb 127.0.0.1 3VeUTYk
test2 2     r      UNASSIGNED
test2 0     p      STARTED       0  130b 127.0.0.1 3VeUTYk
test2 0     r      UNASSIGNED
```

test2インデックスが0〜5の6つのシャードで構成されていることが確認できました。

# まとめ
便利！
Elasticsearchはマッピングの設定やシャーディングの設定はインデックス作成時にしかできません。
今回検証したReindex APIと[Index Aliases API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html)を利用すればアプリケーションコードに変更を加えずに簡単にインデックスの作り直しができそうですね。

今回はドキュメントが1つだけだったので瞬殺で終わりましたが、ドキュメントが多くなった場合はそれなりに時間かかるんでしょうかねー。

