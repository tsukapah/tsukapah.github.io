---
title: DockerのFluentd logging driverを通したltsv形式アクセスログをパースする
date: '2016-12-05T12:37:50+09:00'
lastmod: '2022-04-19T10:45:42+09:00'
draft: false
tags:
- nginx
- Fluentd
- Docker
description: ※当記事は[個人ブログ](https://tech.tsukapah.com/docker-fluentd-logging-driver/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/7ebefb525a06aae61171
  qiita_id: 7ebefb525a06aae61171
---

※当記事は[個人ブログ](https://tech.tsukapah.com/docker-fluentd-logging-driver/)に移行しました

Docker 1.8からは[Fluentd logging driver](https://docs.docker.com/engine/admin/logging/fluentd/)が実装されているのでFluentdでログをゴニョゴニョするのが簡単にできます。jsonの`log`要素がソースコンテナのログになるようです。
そして、話は変わりますがNginxのアクセスログフォーマットをLTSVにしている人は結構多いのではないかと思います。
Fluentd logging driver経由で受け取ったjsonの1オブジェクトに入れられたLTSVのログをFluentdでパースして取り扱いがしやすいようにアウトプットしてみようと思います。
内容としては[Fluentdでも紹介されている](http://www.fluentd.org/guides/recipes/docker-logging)通り`fluent-plugin-parser`を使うだけです。


#受取側のFluentdコンテナを用意

せっかくなので受信側もコンテナで用意します。

##Fluentdの設定(fluent.conf)を作成
```fluent.conf
<source>
  @type forward
</source>

<filter docker.**>
  @type parser
  format ltsv
  key_name log
  reserve_data true
</filter>

<match docker.**>
  @type stdout
</match>
```

Fluentd logging driverはデフォルトで`docker.`をタグプレフィックスにして出力してきます。
今回はデフォルトのまま使います。
ちゃんとやるならrewrite_tag_filterとかで`source`の値ごとにタグを別けてあげるべきなんでしょうが今回はそこは重要じゃないので省きます。

##Fluentdコンテナを起動する
```bash
$ docker run -d -p 24224:24224 --name fluentd -v $(pwd)/fluent.conf:/fluentd/etc/fluent.conf fluent/fluentd:latest
d2ef7c0dd0baf70fb766f38ed4ec26127d8ea5bc5b26b46fb13a04b78c3f684a
```

#ログを渡すNginxコンテナを用意

##Nginxの設定(nginx.conf)を作成
```nginx.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format ltsv 'time:$time_iso8601\t'
                    'remote_addr:$remote_addr\t'
                    'request_method:$request_method\t'
                    'request_length:$request_length\t'
                    'request_uri:$request_uri\t'
                    'https:$https\t'
                    'uri:$uri\t'
                    'query_string:$query_string\t'
                    'status:$status\t'
                    'bytes_sent:$bytes_sent\t'
                    'body_bytes_sent:$body_bytes_sent\t'
                    'referer:$http_referer\t'
                    'useragent:$http_user_agent\t'
                    'forwardedfor:$http_x_forwarded_for\t'
                    'request_time:$request_time\t'
                    'upstream_response_time:$upstream_response_time';

    access_log  /var/log/nginx/access.log  ltsv;

    sendfile        on;

    keepalive_timeout  65;

    include /etc/nginx/conf.d/*.conf;
}

```
ログフォーマット以外はほぼデフォルトです。


##Nginxコンテナの起動
Docker Hubで公開されている公式のnginxイメージを使います。
このイメージはデフォルトのアクセスログが標準出力、エラーログが標準エラー出力のシンボリックリンクになっているのでなにも考えないでもコンテナのログとして扱われます。
Fluentd logging driverはデフォルトで`localhost:24224`を出力先に指定しています。

```bash
$ docker run -d --log-driver=fluentd --name nginx -p 80:80 -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf nginx
59d3d85ac863763940f979c907dfc851c4fe381c87076f5d13611196e8847e5a
```

#動作確認
##Nginxコンテナにリクエストしてみる
```bash
$ curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
...以下省略
```

##Fluentdコンテナのログを確認してみる
```bash
$ docker logs fluentd | tail -1
2016-12-02 14:19:45 +0000 docker.59d3d85ac863: {"container_name":"/nginx","source":"stdout","log":"time:2016-12-02T14:19:45+00:00\tremote_addr:172.17.0.1\trequest_method:GET\trequest_length:73\trequest_uri:/\thttps:\turi:/index.html\tquery_string:-\tstatus:200\tbytes_sent:850\tbody_bytes_sent:612\treferer:-\tuseragent:curl/7.43.0\tforwardedfor:-\trequest_time:0.000\tupstream_response_time:-","container_id":"59d3d85ac863763940f979c907dfc851c4fe381c87076f5d13611196e8847e5a","remote_addr":"172.17.0.1","request_method":"GET","request_length":"73","request_uri":"/","https":"","uri":"/index.html","query_string":"-","status":"200","bytes_sent":"850","body_bytes_sent":"612","referer":"-","useragent":"curl/7.43.0","forwardedfor":"-","request_time":"0.000","upstream_response_time":"-"}
```

出てますね。
jqを使って見やすく...

```bash
$ docker logs fluentd | tail -1 | cut -d " " -f5- | jq .
{
  "container_name": "/nginx",
  "source": "stdout",
  "log": "time:2016-12-02T14:19:45+00:00\tremote_addr:172.17.0.1\trequest_method:GET\trequest_length:73\trequest_uri:/\thttps:\turi:/index.html\tquery_string:-\tstatus:200\tbytes_sent:850\tbody_bytes_sent:612\treferer:-\tuseragent:curl/7.43.0\tforwardedfor:-\trequest_time:0.000\tupstream_response_time:-",
  "container_id": "59d3d85ac863763940f979c907dfc851c4fe381c87076f5d13611196e8847e5a",
  "remote_addr": "172.17.0.1",
  "request_method": "GET",
  "request_length": "73",
  "request_uri": "/",
  "https": "",
  "uri": "/index.html",
  "query_string": "-",
  "status": "200",
  "bytes_sent": "850",
  "body_bytes_sent": "612",
  "referer": "-",
  "useragent": "curl/7.43.0",
  "forwardedfor": "-",
  "request_time": "0.000",
  "upstream_response_time": "-"
}
```

ちゃんとjsonになって扱いやすそうになりました。
ちなみにパースしないとこうなります。

```bash
$ docker logs fluentd | tail -1 | cut -d " " -f5- | jq .
{
  "container_id": "59d3d85ac863763940f979c907dfc851c4fe381c87076f5d13611196e8847e5a",
  "container_name": "/nginx",
  "source": "stdout",
  "log": "time:2016-12-02T14:15:57+00:00\tremote_addr:172.17.0.1\trequest_method:GET\trequest_length:73\trequest_uri:/\thttps:\turi:/index.html\tquery_string:-\tstatus:200\tbytes_sent:850\tbody_bytes_sent:612\treferer:-\tuseragent:curl/7.43.0\tforwardedfor:-\trequest_time:0.000\tupstream_response_time:-"
}
```

これでいい感じに集計とかもできそうです。

#Docker Composeにしてみる

## `docker-compose.yml`作成
ここではイメージのタグを細かく指定しつつ、fluentdのタグをちょっといじってみました。

```docker-compose.yml
version: '2'
services:
  fluentd:
    image: fluent/fluentd:v0.12-latest
    ports:
      - 24224:24224
    volumes:
      - ./fluent.conf:/fluentd/etc/fluent.conf

  nginx:
    image: nginx:1.11.6
    ports:
      - 80:80
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    logging:
      driver: fluentd
      options:
        tag: "docker.nginx.{{.Name}}"
```

## docker-composeで起動

```bash
$ docker-compose up -d
Pulling fluentd (fluent/fluentd:v0.12-latest)...
v0.12-latest: Pulling from fluent/fluentd
3690ec4760f9: Already exists
76d1b89b475a: Pull complete
bcd369573a88: Pull complete
65a3b4aef657: Pull complete
f5518d832e29: Pull complete
b0448a46bc91: Pull complete
a15afbb59fec: Pull complete
b468936212bd: Pull complete
Digest: sha256:fb1b1f28bbec8d0364ae151af56123c1db86d0bb096909bd5876c0fe8bbbe84a
Status: Downloaded newer image for fluent/fluentd:v0.12-latest
Pulling nginx (nginx:1.11.6)...
1.11.6: Pulling from library/nginx
386a066cd84a: Already exists
386dc9762af9: Pull complete
d685e39ac8a4: Pull complete
Digest: sha256:3861a20a81e4ba699859fe0724dc6afb2ce82d21cd1ddc27fff6ec76e4c2824e
Status: Downloaded newer image for nginx:1.11.6
Creating testfluentd_fluentd_1
Creating testfluentd_nginx_1
```

## 確認

### Nginxコンテナにリクエストしてみる

```bash
$ curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
...以下省略
```

### Fluentdコンテナのログを確認してみる

```bash
$ docker logs testfluentd_fluentd_1 | tail -1
2016-12-03 14:45:24 +0000 docker.nginx.testfluentd_nginx_1: {"container_id":"e118fe1156d668514d5213181c305b60aaf8381f3a05e66e8d1690b8f4b53357","container_name":"/testfluentd_nginx_1","source":"stdout","log":"time:2016-12-03T14:45:24+00:00\tremote_addr:172.18.0.1\trequest_method:GET\trequest_length:73\trequest_uri:/\thttps:\turi:/index.html\tquery_string:-\tstatus:200\tbytes_sent:850\tbody_bytes_sent:612\treferer:-\tuseragent:curl/7.43.0\tforwardedfor:-\trequest_time:0.000\tupstream_response_time:-","remote_addr":"172.18.0.1","request_method":"GET","request_length":"73","request_uri":"/","https":"","uri":"/index.html","query_string":"-","status":"200","bytes_sent":"850","body_bytes_sent":"612","referer":"-","useragent":"curl/7.43.0","forwardedfor":"-","request_time":"0.000","upstream_response_time":"-"}
```
期待通りのタグで出てますね。

---

コンテナで環境のポータビリティがよくなりましたが、ログの運用は重要です。
障害調査をしようと思ったらコンテナごとログがなくなってた、なんてことにならないように、ログ設計はしっかりとしたいところなのでこれからも色々やってみます。

