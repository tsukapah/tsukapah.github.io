---
title: CloudFormationでCodeDeploy
date: '2016-06-07T19:42:52+09:00'
lastmod: '2022-04-10T17:46:13+09:00'
draft: false
tags:
- AWS
- CloudFormation
- aws-cli
- CodeDeploy
description: こちらの記事は[個人ブログ](https://tech.tsukapah.com/cloudformation-codedeploy/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/598ef327ccc51b4955b6
  qiita_id: 598ef327ccc51b4955b6
---

こちらの記事は[個人ブログ](https://tech.tsukapah.com/cloudformation-codedeploy/)に移行しました

CloudFormationでS3からAutoScalingGroupにデプロイする定義をしてみます。

PackerでAMI作ってるの待ったり、CloudFormationのcreate-stack待ってる間にびゃーっと書いたので間違いとかあったら指摘して下さい。

# 前提
デプロイ先のインスタンスには[CodeDeploy Agentのインストール](http://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/how-to-run-agent.html)をしておいて下さい。
また、S3からデプロイするので、適切にポリシーを定義したIAM Roleも割り当てておく必要があります。
参考までに、こちらで適用したポリシーが以下です。

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:Get*",
        "s3:List*",
        "s3:Put*"
      ],
      "Resource": "*",
      "Effect": "Allow"
    }
  ]
}
```

## このCloudFormation TemplateでできるAWSリソース
+ IAM Role
+ CodeDeploy:Application
+ CodeDeploy:DeploymentGroup

# 手順

## デプロイしたいものを用意する
デプロイしたいからCodeDeployを使うので、まずはデプロイするものを用意します。
とりあえず以下の感じで試します。

```bash
$ tree
.
├── appspec.yml
└── www
    └── index.html

1 directory, 2 files
```

## デプロイを定義する
`appspec.yml`にはデプロイの定義を記載します。

```bash
$ cat appspec.yml
version: 0.0
os: linux
files:
  - source: /
    destination: /tmp/
```
appspec.ymlのあるディレクトリ以下を、/tmpにデプロイする定義にしてみました。
きっと`/tmp/www/index.html`ができるはずです。
特に前処理も後処理もしないのであればこれだけです。

## デプロイするモジュールをアーカイブする
デプロイしたいものを一式アーカイブします。
アーカイブの形式は、zip、tar、tar.gzのどれかです。

```bash
$ zip -r example.zip ./
  adding: appspec.yml (deflated 4%)
  adding: www/ (stored 0%)
  adding: www/index.html (stored 0%)
```

## S3にアップロード
アーカイブをS3にアップロードします。
バケットは事前に作っておいて下さい。

```bash
$ aws s3 cp ./example.zip s3://examplebucket/
upload: ./example.zip to s3://examplebucket/example.zip
```

## templateを書く

とりあえずパラメータで渡さずベタ書きしてみます。

```template.json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "ExampleContentsDeployRole": {
      "Description": "Deploy example",
      "Type": "AWS::IAM::Role",
      "Properties": {
        "AssumeRolePolicyDocument": {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Sid": "",
              "Effect": "Allow",
              "Principal": {
                "Service": "codedeploy.amazonaws.com"
              },
              "Action": "sts:AssumeRole"
            }
          ]
        },
        "ManagedPolicyArns": [
          "arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole"
        ]
      }
    },
    "ExampleContents": {
      "Type": "AWS::CodeDeploy::Application"
    },
    "ExampleContentsDeployGroup": {
      "Type": "AWS::CodeDeploy::DeploymentGroup",
      "Properties": {
        "ApplicationName": {
          "Ref": "ExampleContents"
        },
        "AutoScalingGroups": ["ExampleGroup"],
        "Deployment": {
          "Revision": {
            "RevisionType": "S3",
            "S3Location": {
              "Bucket": "examplebucket",
              "BundleType": "zip",
              "Key": "example.zip"
            }
          }
        },
        "ServiceRoleArn": {
          "Fn::GetAtt": [
            "ExampleContentsDeployRole",
            "Arn"
          ]
        }
      }
    }
  }
}

```

## validate-templateしてみる

```bash
$ aws cloudformation validate-template --template-body "`cat template.json`"
{
    "CapabilitiesReason": "The following resource(s) require capabilities: [AWS::IAM::Role]",
    "Capabilities": [
        "CAPABILITY_IAM"
    ],
    "Parameters": []
}
```
IAM Roleを作るので`CAPABILITY_IAM`つけろよと注意してくれてますね。

## stackを作成する

```bash
$ aws cloudformation create-stack \
	--stack-name testdeploy \
	--template-body "`cat template.json`" \
	--capabilities CAPABILITY_IAM
{
    "StackId": "arn:aws:cloudformation:ap-northeast-1:360022049035:stack/testdeploy/87076e00-2c6c-11e6-82ca-50a68656dc82"
}
```

これでできたと思います。
初回のデプロイに失敗するとstackごとFailedになってしまうので要注意です。
なので、環境構築用のstackとは別にしたほうが失敗時にダメージが少ないかもしれないですね。

