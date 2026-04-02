---
title: ローカルにAmazonSQS互換のMQを起動して使ってみる
date: '2016-11-23T23:31:54+09:00'
lastmod: '2022-04-17T22:59:05+09:00'
draft: false
tags:
- Docker
- aws-cli
- sqs
- ElasticMQ
description: ※当記事は[個人ブログ](https://tech.tsukapah.com/elasticmq/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/0ba64ac734e885f6e8e8
  qiita_id: 0ba64ac734e885f6e8e8
---

※当記事は[個人ブログ](https://tech.tsukapah.com/elasticmq/)に移行しました

最近[ElasticMQ](https://github.com/adamw/elasticmq)というのを知りました。
Amazon SQS-compatible interface ということで、Amazon SQSを使ったアプリケーション開発が捗りそうです。
ちょうど試そうと思った今日(2016/11/23)にVersion 0.11.0になっていました。
java6以降が入ってれば動くようです。
今回はDockerで動かして、aws-cliで接続してみたいと思います。

# Dockerfileの作成
```Dockerfile
FROM java:8

ADD https://s3-eu-west-1.amazonaws.com/softwaremill-public/elasticmq-server-0.11.0.jar /elasticmq/elasticmq-server-0.11.0.jar
EXPOSE 9324
ENTRYPOINT ["java","-jar","/elasticmq/elasticmq-server-0.11.0.jar"]
```

# コンテナイメージのビルド
```bash
$ docker build -t tsukapah/elasticmq:0.11.0 .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM java:8
8: Pulling from library/java
386a066cd84a: Pull complete
75ea84187083: Pull complete
88b459c9f665: Pull complete
690dbea0e4ca: Pull complete
7e401cdd6f18: Pull complete
a58186ddf9a0: Pull complete
49999ed55bc4: Pull complete
eb40561aad8f: Pull complete
Digest: sha256:dd0fc686a5584c0c7f3e50dd84ddc42fae400c27a21d8ca98dad190aff5e9d52
Status: Downloaded newer image for java:8
 ---> 861e95c114d6
Step 2 : ADD https://s3-eu-west-1.amazonaws.com/softwaremill-public/elasticmq-server-0.11.0.jar /elasticmq/elasticmq-server-0.11.0.jar
Downloading  30.2 MB/30.2 MB
 ---> 55fa5752ace4
Removing intermediate container 2529a37411e1
Step 3 : EXPOSE 9324
 ---> Running in 0465d32da4e8
 ---> 052963c601f8
Removing intermediate container 0465d32da4e8
Step 4 : ENTRYPOINT java -jar /elasticmq/elasticmq-server-0.11.0.jar
 ---> Running in e283caeeb0b2
 ---> 3f6b0738e570
Removing intermediate container e283caeeb0b2
Successfully built 3f6b0738e570
```

# コンテナの起動
```bash
docker run -d --name=elasticmq -p 9324:9324 tsukapah/elasticmq:0.11.0
082c9ffb6c1048615b72fabf1440241f77f9decbf74adb142fae75b0bcacfd2c
```
どうやら起動したみたいです。

# aws-cliを使って接続してみる
```bash
$ aws sqs list-queues --endpoint-url http://localhost:9324
```
なんも出力されない。まぁQueueがないので当然なんだけど。
ちなみにCredentialやDefaultRegionが設定されていないと以下の感じでエラーが出ます。
ダミーでもいいので設定しましょう。

```bash
$ aws sqs list-queues --endpoint-url http://localhost:9324
You must specify a region. You can also configure your region by running "aws configure".
```

## Queueの作成
```bash
$ aws sqs create-queue --queue-name test --endpoint-url http://localhost:9324
{
    "QueueUrl": "http://localhost:9324/queue/test"
}
```
お、どうやらできたっぽい。

## Queueの確認
```bash
$ aws sqs list-queues --endpoint-url http://localhost:9324
{
    "QueueUrls": [
        "http://localhost:9324/queue/test"
    ]
}
```

## メッセージの投入
```bash
$ aws sqs send-message --queue-url http://localhost:9324/queue/test --message-body "Information about the largest city in Any Region." --endpoint-url http://localhost:9324
{
    "MD5OfMessageBody": "51b0a3256d59467f973009b739163aa0",
    "MD5OfMessageAttributes": "d41d8cd98f00b204e9800998ecf8427e",
    "MessageId": "8df11eeb-e83c-4076-8b63-1edec5265402"
}
```
入ったみたいです。
`--queue-url`と`--endpoint-url`が冗長な感じで気持ち悪いですが、どっちも必要です。

## メッセージの取得
```bash
$ aws sqs receive-message --queue-url http://localhost:9324/queue/test --endpoint-url http://localhost:9324
{
    "Messages": [
        {
            "Body": "Information about the largest city in Any Region.",
            "MD5OfMessageAttributes": "d41d8cd98f00b204e9800998ecf8427e",
            "ReceiptHandle": "8df11eeb-e83c-4076-8b63-1edec5265402#0f0d12eb-dd37-4ac8-a11d-a8c41420eca0",
            "MD5OfBody": "51b0a3256d59467f973009b739163aa0",
            "MessageId": "8df11eeb-e83c-4076-8b63-1edec5265402"
        }
    ]
}
```
無事取り出せました。

---
簡単でした。

