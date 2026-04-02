---
title: よく使うのでDocker ComposeでElasticsearchとKibanaを一撃で起動する
date: '2016-11-06T23:24:01+09:00'
lastmod: '2022-04-12T22:28:25+09:00'
draft: false
tags:
- Mac
- Elasticsearch
- Docker
- Kibana
- docker-compose
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/docker-compose-elasticsearch-kibana/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/dc193ef5825f45897898
  qiita_id: dc193ef5825f45897898
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/docker-compose-elasticsearch-kibana/)に移行しました

Docker Composeは複数のコンテナをまとめて定義・実行するためのツールです。
Docker Composeを使わないで個別に起動する方法は[こちら](http://qiita.com/tsukapah/items/24bab0f5ab9bb4e12309)です。

# 実行環境
![DockerEngine.png](/images/b1a56210-672a-88ea-e5cf-09ca1ee46c6c.png "DockerEngine.png")

```bash
$ uname -v
Darwin Kernel Version 15.6.0: Thu Sep  1 15:01:16 PDT 2016; root:xnu-3248.60.11~2/RELEASE_X86_64
$ docker --version
Docker version 1.12.1, build 6f9534c
$ docker-compose --version
docker-compose version 1.8.0, build f3628c7
```

# クラスタの定義
```docker-compose.yml
kibana:
  image: kibana
  links:
    - elasticsearch:elasticsearch
  ports:
    - 5601:5601

elasticsearch:
  image: elasticsearch
  ports:
    - 9200:9200
    - 9300:9300
```

# クラスタ起動(デタッチモード)
```bash
$ docker-compose up -d
Creating kibanacluster_elasticsearch_1
Creating kibanacluster_kibana_1
```

# 起動状態の確認
```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                            NAMES
eecb811cb144        kibana              "/docker-entrypoint.s"   17 minutes ago      Up 17 minutes       0.0.0.0:5601->5601/tcp                           kibanacluster_kibana_1
1c60a47ffd10        elasticsearch       "/docker-entrypoint.s"   17 minutes ago      Up 17 minutes       0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   kibanacluster_elasticsearch_1
```

# 接続確認
```bash
$ curl localhost:9200
{
  "name" : "shyHA8G",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "9I82GX3XSRCaPzRwXwD04A",
  "version" : {
    "number" : "5.0.0",
    "build_hash" : "253032b",
    "build_date" : "2016-10-26T05:11:34.737Z",
    "build_snapshot" : false,
    "lucene_version" : "6.2.0"
  },
  "tagline" : "You Know, for Search"
}
$ open http://localhost:5601
```
![Kibana5.0.0.png](/images/4cd881d1-4e5d-6f27-4a28-a9fd1623f931.png "Kibana5.0.0.png")

# クラスタの停止
```bash
$ docker-compose stop
Stopping kibanacluster_kibana_1 ... done
Stopping kibanacluster_elasticsearch_1 ... done
```

# コンテナの削除
```bash
$ docker-compose rm
Going to remove kibanacluster_kibana_1, kibanacluster_elasticsearch_1
Are you sure? [yN] y
Removing kibanacluster_kibana_1 ... done
Removing kibanacluster_elasticsearch_1 ... done
```

