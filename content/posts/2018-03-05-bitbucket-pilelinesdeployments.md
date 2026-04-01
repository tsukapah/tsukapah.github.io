---
title: Bitbucket Pilelines/Deployments入門
date: '2018-03-05T17:25:53+09:00'
lastmod: '2018-03-05T17:25:53+09:00'
draft: false
tags:
- AWS
- S3
- Bitbucket
- CloudFront
- BitbucketPipelines
description: Amazon S3とCloudFrontを使用してスタティックなWebコンテンツを配信・運用しているんですが、
params:
  qiita_url: https://qiita.com/tsukapah/items/48d6f2882fe1db4d3238
  qiita_id: 48d6f2882fe1db4d3238
---

# はじめに
Amazon S3とCloudFrontを使用してスタティックなWebコンテンツを配信・運用しているんですが、
「~~AWSのコンソールポチポチするのストレス~~更新時の運用を楽にしたいんじゃ！」というディレクターやデザイナーの要望に答えるためにBitbucket Pipelinesを使用して自動化してみました。
スタティックなファイルのデプロイなので、PipelinesとしてもDeploymentsとしても入門レベルの内容だと思います。

# 前提
+ S3+CloudFrontでコンテンツ配信できる環境が整っていること
+ BitBucketのGitリポジトリがあること

# 手順
## 1. IAMユーザの作成
Bitbucket Pipelinesがデプロイに使用するIAMユーザを作成します。
IAMには`S3のオブジェクトを更新するのに必要`な権限と、`CloudFrontのInvalidationリクエストができる`権限が必要です。
ポリシーそのものは適宜作成してください。

ユーザを作成したら、アクセスキーの生成をしてキーペアを取得してください。

## 2. BitBucketの設定
### 2.1 Pipelinesの有効化
リポジトリの左メニューより　`設定` > PIPELINES `Setting` > `Enable Pipelines`を有効にします。

### 2.2 環境変数の設定
リポジトリの左メニューより　`設定` > PIPELINES `Environment variables`に進み、以下の環境変数を定義します。

|Variable name|Value|Secured|
|---|---|---|
|AWS_ACCESS_KEY_ID|手順1で取得したACCESS_KEY_ID|:heavy_check_mark:|
|AWS_SECRET_ACCESS_KEY|手順1で取得したSECRET_ACCESS_KEY|:heavy_check_mark:|
|CONTENTS_BUCKET|デプロイ先のバケット名||
|DISTRIBUTION_ID|CloudFront distributionのID||

なんらかの理由でS3のバケットが変更になったり、CloudFrontのDistributionが変わったりした場合でもGitのコミットなしで変更できるようにキーペア以外も環境変数で定義してみました。

### 2.3 Pipelineの定義

以下のファイルをリポジトリ直下にコミットします。

```bitbucket-pipelines.yml
image: python:3.6.4

pipelines:
  branches:
    master:
      - step:
          name: Print Message
          caches:
            - pip
          script:
            - echo "Deploy process to production environment started"
      - step:
          name: Deploy to Production(push to S3 and cache invalidation)
          deployment: production
          trigger: manual
          caches:
            - pip
          script:
            # Install
            - pip install awscli
            # Push to S3
            - aws s3 sync . s3://${CONTENTS_BUCKET}
              --delete
              --exclude '.git/*'
              --exclude '.gitignore'
              --exclude 'README.md'
              --exclude 'bitbucket-pipelines.yml'
            # Requesting cache invalidation
            - aws cloudfront create-invalidation --distribution-id ${DISTRIBUTION_ID} --path '/*'
```

この内容の場合、リモートリポジトリ(Bitbucket)のmasterブランチにマージされるとPipelineが走り出します。
`trigger: manual`は最初の`step`には記述できないようなので、`echo`でメッセージを出力するだけのstepを記述し、2つめの`step`に処理を記載しています。

これでデプロイはPipelinesの画面からボタンを押すだけでできるようになり、また、Deploymentsの画面からどのコミットがデプロイされているのかを追えるようになるので便利です。

