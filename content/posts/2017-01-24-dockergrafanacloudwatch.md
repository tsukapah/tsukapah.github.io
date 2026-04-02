---
title: Docker+Grafanaでアカウントを横断してCloudWatchを可視化する
date: '2017-01-24T19:39:34+09:00'
lastmod: '2022-04-18T23:26:29+09:00'
draft: false
tags:
- AWS
- CloudWatch
- Docker
- aws-cli
- grafana
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/docker-grafana/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/257c3744763dbc0accf4
  qiita_id: 257c3744763dbc0accf4
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/docker-grafana/)に移行しました

ちょっとしたモニタリングだったらCloudWatchでも十分。
ただ、普段コマンドラインツールで操作していることもあるし、複数アカウントを持っていたりするのでマネージメントコンソールだと使い勝手が悪い。
そんな悩みをGrafanaが解決してくれたので自分用にメモメモ。
# 必要なもの
+ AWSのIAM User
+ aws-cli
+ Docker for Mac(Docker Machineでも可)

# credentialsの登録

自分が普段使っているMacでは、aws-cliを使ってprofileをいくつも登録してある。
登録するときはこんな感じ。
`profile`名は適宜変更すること。
すでに作成済みの場合は実施不要。

```bash
$ aws configure --profile hogehoge
AWS Access Key ID [None]: aaaaa
AWS Secret Access Key [None]: bbbbb
Default region name [None]:
Default output format [None]:
```

すると、以下の感じで必要な情報が登録されているはず。

```bash
$ cat ~/.aws/credentials
[hogehoge]
aws_access_key_id = aaaaa
aws_secret_access_key = bbbbb
```

credentialの取り扱いには気をつけること！

# DockerでGrafanaを起動

上記で作成したcredentialsを`-v`オプションでGrafanaコンテナに渡す。
公式イメージあるので捗る。
Grafanaは3000ポートでListenしているのでそれもあわせてマッピング。

```bash
$ docker run -d -p 3000:3000 -v ~/.aws/credentials:/usr/share/grafana/.aws/credentials --name grafana grafana/grafana
3d7de10094f1d84dbedd8a77fb55ef5c997ccdde31722a114bcd25388b1a8b3d
```

起動した。

# Grafanaにデータを追加

接続してみる。

```bash
$ open http://localhost:3000
```

![screencapture-localhost-3000-login-1484041461957.png](https://qiita-image-store.s3.amazonaws.com/0/32208/89b5cc36-ef2d-de38-2854-fe5e86ed7500.png "screencapture-localhost-3000-login-1484041461957.png")

デフォルトのログインID/PWは`admin/admin`。
ログインするとこんな感じ。
![Kobito.cKv8Tt.png](https://qiita-image-store.s3.amazonaws.com/0/32208/7ea01852-2ca2-daed-844d-d63f92e0af53.png "Kobito.cKv8Tt.png")
なんか前見たときよりUIがよくなってるかも。
ちょっと目立つ感じになってる`Add data source`ボタンを押す。

![Kobito.b6Dvv3.png](https://qiita-image-store.s3.amazonaws.com/0/32208/f8a13f8e-721e-965c-5d0e-5d97d08b9366.png "Kobito.b6Dvv3.png")
こんな画面になる。
Nameには任意のData Sourceの名前をつけてる。
Typeのプルダウンから`CloudWatch`を選択。
Auth Providerは`Credentials file`以外にも`Access & secret key`やARNが選択できるが、変更せずに`Credentials file`を指定する。
Default Regionは適宜選択すること。
Custom Metrics namespaceはなにかあれば入力するが、とりあえず空のままでOK。
左下の`Add`ボタンを押して、`Success`と表示されれば登録できてるっぽい。

![Kobito.KIPEF2.png](https://qiita-image-store.s3.amazonaws.com/0/32208/cf34f400-3211-b981-fce3-4d0fbe213c9f.png "Kobito.KIPEF2.png")
こんな感じで登録できた。

あとはdashboardでポチポチグラフを追加していい感じの画面を作ればOK。
![Kobito.TV43uj.png](https://qiita-image-store.s3.amazonaws.com/0/32208/794fceed-20b8-1b24-ddea-675974f952a4.png "Kobito.TV43uj.png")

---

Dockerで起動しているのでこのままだとデータの永続化とかはできてないけど、とりあえず簡単に起動できて便利。

