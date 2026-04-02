---
title: Dockerを使用した複数ホストでのElasticsearchクラスタの構築検証
date: '2017-06-02T00:33:23+09:00'
lastmod: '2022-04-30T23:55:58+09:00'
draft: false
tags:
- Elasticsearch
- Docker
- docker-machine
description: ※この記事は[個人ブログ](https://tech.tsukapah.com/elasticsearch-cluster-with-docker/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/1df305026f55cdc77b4b
  qiita_id: 1df305026f55cdc77b4b
---

※この記事は[個人ブログ](https://tech.tsukapah.com/elasticsearch-cluster-with-docker/)に移行しました

Elasticsearchは公式でコンテナイメージを提供してくれているのでそれを利用してクラスタを組みたいと思います。
[ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)には`docker-compose.yml`も公開されていますが、docker-composeということはコンテナホストはひとつになるから可用性的にNGだなー、と感じました。(Swarm modeとか使うと解決できるのかな？)
コンテナホストは複数台に分散したいと考えたので、検証してみました。

# 検証環境
+ Mac OS X El Capitan(10.11.6)
+ docker-machine version 0.10.0, build 76ed2a6
+ VirtualBox 5.1.14 r112924 (Qt5.6.2)

# ポイント
+ docker-machineを利用してVirtualBox上に3つのコンテナホストを起動
+ 各コンテナホストに公式のコンテナイメージをデプロイ
+ Dockerのネットワークドライバはhostモードを利用する

# 検証手順
## コンテナホストの準備
### コンテナホストの作成
VirtualBox上にコンテナホストを3つ起動します。

```bash
$ docker-machine create --driver virtualbox elasticsearch1
$ docker-machine create --driver virtualbox elasticsearch2
$ docker-machine create --driver virtualbox elasticsearch3
```

### コンテナホストのメモリマップ上限の引き上げ
[公式](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)にも書いてある通り、設定しないとコンテナが起動しない(開発者モードなら起動するかも)ので設定します。

```bash
$ docker-machine ssh elasticsearch1 sudo sysctl -w vm.max_map_count=262144
$ docker-machine ssh elasticsearch2 sudo sysctl -w vm.max_map_count=262144
$ docker-machine ssh elasticsearch3 sudo sysctl -w vm.max_map_count=262144
```

## Elaticsearchの起動〜動作確認
### Elasticsearchコンテナの起動

Elasticが独自に公開しているコンテナレジストリよりイメージを取得して起動します。
環境変数(`-e`)でパラメータを渡すとElasticsearchの設定を上書きできるのでいきなり起動できます。
検証で渡したパラメータは以下の通り。

|環境変数名|パラメータに渡す値|説明|
|---|---|---|
|network.host|\_eth1:ipv4_|hostモードでコンテナを起動するとホストのNICが全部マッピングされますが、このオプションを指定しないとElasticsearchはeth0のIPアドレスをデフォルトで指定するようです。<br>eth0のIPアドレスはプライベートなIPアドレスが割り当てられていて外部と通信できないのでeth1を明示しています。|
|discovery.zen.ping.unicast.hosts|自分自身以外のコンテナホストのIPアドレスをカンマ区切り|クラスタに参加させるホストを明示します。|
|discovery.zen.minimum_master_nodes|2|`ノード数/2+1`を設定するよう[ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#minimum_master_nodes)にも記載されているので従います。|
|xpack.security.enabled|false|今回の検証には不要なのでX-Packの[Securityを無効化](https://www.elastic.co/guide/en/x-pack/current/security-settings.html)します。|
|xpack.monitoring.enabled|false|今回の検証には不要なのでX-Packの[Monitoringを無効化](https://www.elastic.co/guide/en/x-pack/current/monitoring-settings.html)します。|
|xpack.watcher.enabled|false|今回の検証には不要なのでX-Packの[Watcherを無効化](https://www.elastic.co/guide/en/x-pack/current/notification-settings.html)します。|
|xpack.graph.enabled|false|今回の検証には不要なのでX-Packの[Graphを無効化](https://www.elastic.co/guide/en/x-pack/current/graph-settings.html)します。|
|xpack.ml.enabled|false|今回の検証には不要なのでX-Packの[Machine Learningを無効化](https://www.elastic.co/guide/en/x-pack/current/ml-settings.html)します。|
|ES_JAVA_OPTS|-Xms512m -Xmx512m|Elasticsearchが使用するjavaのヒープサイズを指定します。<br>コンテナホストのメモリを増やすという方法もありますが、3台も2GB以上のホストを起動したらこのMacがゲロ重になるのでこうしました。|

その他にコンテナに渡したオプションは以下の通り

|オプション|説明|
|---|---|
|-d|デタッチドモードでコンテナを起動。<br>説明はいらないですよね？|
|--network="host"|コンテナのネットワークモードを`host`に設定します。<br>コンテナホストのネットワークインターフェースがそのままコンテナ上でも利用されます。|
|-p 9200:9200|Elasticsearchのサービスポートを公開します。<br>`-P(大文字)`でもよかったかも。|
|-p 9300:9300|Elasticsearchがノード間のコミュニケーションで使用するポートを公開します。<br>`-P(大文字)`でもよかったかも。|


```bash:1台目の起動
$ eval $(docker-machine env elasticsearch1)
$ docker run -d \
  -e "network.host=_eth1:ipv4_" \
  -e "discovery.zen.ping.unicast.hosts=$(docker-machine ip elasticsearch2),$(docker-machine ip elasticsearch3)" \
  -e "discovery.zen.minimum_master_nodes=2" \
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

```bash:2台目の起動
$ eval $(docker-machine env elasticsearch2)
$ docker run -d \
  -e "network.host=_eth1:ipv4_" \
  -e "discovery.zen.ping.unicast.hosts=$(docker-machine ip elasticsearch1),$(docker-machine ip elasticsearch3)" \
  -e "discovery.zen.minimum_master_nodes=2" \
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

```bash:3台目の起動
$ eval $(docker-machine env elasticsearch3)
$ docker run -d \
  -e "network.host=_eth1:ipv4_" \
  -e "discovery.zen.ping.unicast.hosts=$(docker-machine ip elasticsearch1),$(docker-machine ip elasticsearch2)" \
  -e "discovery.zen.minimum_master_nodes=2" \
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

### Clusterのステータスを確認する

3台起動したのでクラスタのステータスを確認してみます。

```bash
$ curl $(docker-machine ip elasticsearch1):9200/_cat/health?v
epoch      timestamp cluster        status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1496326996 14:23:16  docker-cluster green           3         3      0   0    0    0        0             0                  -                100.0%
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
```

以下の点が確認できればよいと思います。

+ statusがgreen
+ ノードの数が3
+ データノードの数が3

### インデックスを作って確認してみる

クラスタが組めてそうなのでインデックスを作ってみます。

```bash
$ curl -XPUT $(docker-machine ip elasticsearch1):9200/test?pretty
{
  "acknowledged" : true,
  "shards_acknowledged" : true
}
```

クラスタなので、どのIPアドレスに問い合わせても上記で作成したインデックスができていれば問題ないでしょう。
確認してみます。

```bash:1台目で確認
$ curl $(docker-machine ip elasticsearch1):9200/_cat/indices?v
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test  0iOZud02QHqPMkcDemgdWg   5   1          0            0      1.2kb           650b
```

1台目はできてます。ここはインデックスを作ったときに指定したホストなので当然ですね。

```bash:2台目で確認
$ curl $(docker-machine ip elasticsearch2):9200/_cat/indices?v
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test  0iOZud02QHqPMkcDemgdWg   5   1          0            0      1.2kb           650b
```

2台目でもできてますね。順調。

```bash:3台目で確認
$ curl $(docker-machine ip elasticsearch3):9200/_cat/indices?v
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test  0iOZud02QHqPMkcDemgdWg   5   1          0            0      1.2kb           650b
```

3台目にもありました。バッチリ。

# まとめ

以上で複数ホストにまたがった可用性のあるElasticsearchクラスタをコンテナで起動することができました。
このあとは`cloud-aws`プラグインを使用してEC2上にデプロイする方法などを検証してみようかなーと思っています。(コンテナでもできるのかな？)
~~[Amazon Elasticsearch Service](https://aws.amazon.com/jp/elasticsearch-service/)はVPCに対応してないのが残念ですよね。~~ [^1]


お気づきの点とかあれば優しくご指摘頂ければと思います。

[^1]: [サポートしましたね](https://aws.amazon.com/jp/blogs/news/amazon-elasticsearch-service-now-supports-vpc/)

