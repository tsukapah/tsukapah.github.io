---
title: Dockerで Elasticsearch + Kibana を動かしてみた
date: '2015-12-11T13:32:26+09:00'
lastmod: '2022-04-05T22:50:26+09:00'
draft: false
tags:
- Elasticsearch
- Docker
- Kibana
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/docker-elasticsearch-kibana/)に移行しました。
params:
  qiita_url: https://qiita.com/tsukapah/items/24bab0f5ab9bb4e12309
  qiita_id: 24bab0f5ab9bb4e12309
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/docker-elasticsearch-kibana/)に移行しました。

~~Docker HubにElasticsearchとKibanaの公式イメージがあるのでそれを使ってみました。~~
[こちら](https://github.com/docker-library/elasticsearch)のREADMEに以下の記載がありました。

> This is the Git repo of the Docker Hub image for Elasticsearch. See the Docker Hub page for the full readme on how to use this Docker image and for information regarding contributing and issues. The Docker Hub image is not prepared in collaboration with nor supported by upstream Elastic which provides its own official image.

Elastic社が独自に提供するイメージが公式みたいなのでそちらに変更します。

# 前提
Dockerの実行環境があること。
Dockerの実行環境構築は色んな人が書いてくれてるので割愛します。
参考までに、こちらの環境はMac OS X + VirtualBox + docker-machineです。
※Docker for Macを導入した際、この手順だと微妙なところがあったので修正しました。 ~~ついでなので使用するコンテナのバージョンを5.0.0に変更しました。(2016/11/4)~~
Elastic社が公開しているレジストリを参照するように変更し、バージョンを5.4.1に変更しました。また、レガシーなlink機能の利用をやめ、Bridgeネットワークを利用するように変更しました。(2017/6/3)


# 手順
## コンテナ実行環境の確認
※Docker for Mac等、ローカルにDocker環境がある場合は不要

```bash
$ docker-machine env test-machine
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/tsukapah/.docker/machine/machines/test-machine"
export DOCKER_MACHINE_NAME="test-machine"
# Run this command to configure your shell:
# eval "$(docker-machine env test-machine)"
```

## Bridgeネットワークの作成

```bash
$ docker network create elasticsearch --driver bridge
1c00368f055cd86e40da5db8b9e41211abc2821763b604a2583c054c6ecbcb22
```

## Elasticsearchコンテナの起動
```bash
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
	--network="elasticsearch" \
	docker.elastic.co/elasticsearch/elasticsearch:5.4.1
931bceaa78bf8129c59989116dc1aa2698382315246a5635b06b9b787e5c1f77
```

## Elasticsearch動作確認
```
$ curl $(docker-machine ip test-machine):9200
{
  "name" : "6Y9ph9_",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "IWB-HjK9Q3-x5OS6Fh4rPg",
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

※Docker for Mac等の場合は宛先はlocalhost:9200とかになります。

## Kibanaコンテナの起動
```bash
$ docker run -d \
	--name kibana \
	-p 5601:5601 \
	-e "ELASTICSEARCH_URL=http://elasticsearch:9200" \
	-e "xpack.graph.enabled=false" \
	-e "xpack.security.enabled=false" \
	-e "xpack.ml.enabled=false" \
	--network="elasticsearch" \
	docker.elastic.co/kibana/kibana:5.4.1
81db2e8f947ae24ef1cb50dbabc6b13e0fe62233c36a2c47de725fe048152457
```

## Kibanaに接続してみる
```bash
$ open http://$(docker-machine ip test-machine):5601
```

※Docker for Mac等の場合は宛先はlocalhost:5601とかになります。

こんな感じでエラーが出ないで初期画面が表示されれば成功。
![Kibana5.0.0.png](https://qiita-image-store.s3.amazonaws.com/0/32208/4cd881d1-4e5d-6f27-4a28-a9fd1623f931.png "Kibana5.0.0.png")

あとはもう好きなようにデータをElasticsearchに投げつけちゃいましょう。
X-PackのSecurityをfalseにしているので認証無しでデータを投入できます。

# 雑感
いままでEC2でガチャガチャやってたことを思うと、超簡単でした。
一度docker pullでイメージ手元に持ってしまえばインターネットにつながってなくてもいろいろ試せてうれしいですね。
Elasticsearch + Dockerについてはクラスタ名の変更とか ~~クラスタリングとか~~ データ永続化とかをおいおいやってみようと思います。
KibanaはElasticsearchのバージョン依存が強いので、最新のイメージを使わない場合はバージョン管理を気をつけたほうが良さそうですね。Docker Composeを利用することでコードで管理できると思います。

※[クラスタリングについて書きました](http://qiita.com/tsukapah/items/1df305026f55cdc77b4b)

