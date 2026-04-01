---
title: Apex+Terraformでサーバレスアーキテクチャをフルコード化
date: '2019-04-06T00:16:08+09:00'
lastmod: '2019-04-06T00:16:08+09:00'
draft: false
tags:
- AWS
- Apex
- lambda
- Terraform
- APIGateway
description: 私のチームではAWSにおけるインフラの構成管理にTerraformを利用しています。
params:
  qiita_url: https://qiita.com/tsukapah/items/8e0cf03aff49a3c565f8
  qiita_id: 8e0cf03aff49a3c565f8
---

私のチームではAWSにおけるインフラの構成管理にTerraformを利用しています。
ただ、最近は性能がそれほど求められないバッチ処理やAPI等はサーバレスアーキテクチャを採用することが多くなってきました。
しかし、Lambdaファンクションのコード自体のデプロイをTerraformに任せるのがしんどかったりするので、
Lambdaのデプロイは開発チームが利用していたServerless Frameworkに切り分けています。
これには大きな課題があって、`Terraformで作ったS3のイベントをトリガーにするLambdaファンクションを作りたい`、みたいな、後からLambdaを追加する要件に対応しきれずに悩んでいました。
そんなときに`Apex`はTerraformをWrapできるらしいと天から声が降りてきたので試してみることにしてみました。

# 検証に利用した環境
+ MacOS X(10.11.6)
+ Apex(1.0.0-rc2)
+ Terraform(v0.11.11)  
※当エントリではApexやTerraformの導入については触れません

# 作る環境
<img width="1124" alt="APIGateway-Lambda.png" src="https://qiita-image-store.s3.amazonaws.com/0/32208/a88ba059-890b-2b48-e562-9a4e0b664eae.png">
Lambdaで使用するIAM Roleや、API GatewayはTerraformに管理を任せ、LambdaそのものはApexに管理してもらいます。

# ファイル構成
```bash
./project.json # プロジェクト全体の定義(たぶんfunction.jsonで設定を上書きできる気がする)
./functions/hello/function.json # 関数の定義
./functions/hello/hello.py # Lambdaで実行するコード(別にPythonじゃなくてもOK)
./infrastructure/main.tf # Terraformで管理するAWSリソースの定義
```

## ./project.json
たぶんプロジェクト全体の共通設定を記載する感じだと思います。
すいません、ちゃんと調べてません。。。

```json
{
  "name": "apex-sample",
  "description": "apex test 20190405",
  "memory": 128,
  "timeout": 5,
  "role": "arn:aws:iam::{各自のAWS Account ID}:role/apex_sample",
  "environment": {}
}
```

## ./functions/hello/function.json
利用するruntimeや関数名を定義します。
`runtime`と`handler`さえちゃんとしてれば最低限は大丈夫だと思います。

```json
{
    "description": "hello",
    "runtime": "python3.6",
    "handler": "hello.lambda_handler",
    "environment":
    {
        "ENV": "dev"
    }
}
```

## ./functions/hello/hello.py
`hello world`をレスポンスボディに返すだけの処理です。
最低限API Gatewayの仕様に則ったレスポンスを返せれば大丈夫です。

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

def lambda_handler(event, context):
    json = {"statusCode": 200, "body": "hello world"}
    return json
```

### ./infrastructure/main.tf
完全にTerraformのコードです。
めんどくさがって雑な感じに1ファイルにまとめちゃいましたが、module化したりしたほうが管理はしやすいと思います。
`variable "apex_function_hello" {}`でapexから変数を受け取るのがポイントです。
定義しているリソースはざっくりIAM RoleとAPI Gatewayだけです。

```HCL
provider "aws" {
  region = "ap-northeast-1"
}

terraform {
  backend "s3" {
    bucket = "apex-test-bucket"
    key    = "apex/test20190404/terraform.tfstate"
    region = "ap-northeast-1"
  }
}
variable "apex_function_hello" {}

data "aws_region" "current" {}
data "aws_iam_policy_document" "lambda" {
  version = "2012-10-17"
  statement {
    actions = [
      "logs:*"
    ]
    resources = [
      "*"
    ]
    effect = "Allow"
  }
}

resource "aws_iam_policy" "policy" {
  name = "apex_sample"
  path = "/"
  policy = "${data.aws_iam_policy_document.lambda.json}"
}

resource "aws_iam_role" "lambda" {
  name = "apex_sample"
  description = ""
  assume_role_policy = <<EOF
{
  "Version":"2012-10-17",
  "Statement":
  [
    {
      "Effect":"Allow",
      "Principal":
      {
        "Service":"lambda.amazonaws.com"
      },
      "Action":"sts:AssumeRole"
    }
  ]
}
EOF
  force_detach_policies = false
  path = "/"
  max_session_duration = 3600
}

resource "aws_iam_role_policy_attachment" "lambda" {
  role       = "${aws_iam_role.lambda.name}"
  policy_arn = "${aws_iam_policy.policy.arn}"
}

resource "aws_api_gateway_rest_api" "rest_api" {
    name = "apex_sample"
    description = "Created by AWS Lambda"
    endpoint_configuration {
        types = ["REGIONAL"]
    }
    minimum_compression_size = -1
    api_key_source = "HEADER"
}

resource "aws_api_gateway_resource" "proxy" {
  rest_api_id = "${aws_api_gateway_rest_api.rest_api.id}"
  parent_id   = "${aws_api_gateway_rest_api.rest_api.root_resource_id}"
  path_part   = "{proxy+}"
}

resource "aws_api_gateway_method" "proxy" {
  rest_api_id   = "${aws_api_gateway_rest_api.rest_api.id}"
  resource_id   = "${aws_api_gateway_resource.proxy.id}"
  http_method   = "ANY"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "lambda" {
  rest_api_id = "${aws_api_gateway_rest_api.rest_api.id}"
  resource_id = "${aws_api_gateway_method.proxy.resource_id}"
  http_method = "${aws_api_gateway_method.proxy.http_method}"

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = "arn:aws:apigateway:${data.aws_region.current.name}:lambda:path/2015-03-31/functions/${var.apex_function_hello}/invocations"
}

resource "aws_api_gateway_deployment" "apigw" {
  depends_on = [
    "aws_api_gateway_integration.lambda",
  ]

  rest_api_id = "${aws_api_gateway_rest_api.rest_api.id}"
  stage_name  = "default"
}

resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = "${var.apex_function_hello}"
  principal     = "apigateway.amazonaws.com"
  source_arn = "${aws_api_gateway_deployment.apigw.execution_arn}/*/*"
}

output "base_url" {
  value = "${aws_api_gateway_deployment.apigw.invoke_url}"
}
```

# デプロイ
## 環境変数の設定
```bash
$ export AWS_ACCESS_KEY_ID=hogehoge
$ export AWS_SECRET_ACCESS_KEY=piyopiyo
$ export AWS_DEFAULT_REGION=ap-northeast-1
$ export AWS_REGION=ap-northeast-1
```

## Terraformバックエンドの初期化
```bash
$ apex infra init
```
※`apex infra`コマンドがterraformコマンドをwrapしてくれてるようです。

## IAM Roleのデプロイ
API GatewayをデプロイするにはLambdaが必要、LambdaをデプロイするためにはIAM Roleが必要。
ということで、まずはIAM Roleだけデプロイします。

```bash
$ apex infra apply -target aws_iam_policy.policy -target aws_iam_role.lambda -target aws_iam_role_policy_attachment.lambda
```
すぐに`var.apex_function_hello`の値を入力するように促されますが、IAM Roleでは特に使用していないので適当に入力してEnterしちゃいます。

## Lambdaのデプロイ

```bash
$ apex deploy
   • creating function         env= function=hello
   • created alias current     env= function=hello version=1
   • function created          env= function=hello name=apex-sample_hello version=1
```

## 残りのAWSリソースのデプロイ

```bash
$ apex infra apply
```
ここではLambdaファンクションがデプロイ済みなので`var.apex_function_hello`の値は聞かれません。

# 動作確認
これですべてのリソースがデプロイできたはずなので動作を確認してみます。

```
$ curl https://budfnr6xp6.execute-api.ap-northeast-1.amazonaws.com/default/apex-sample_hello
hello world
```
ちゃんとレスポンスが返ってきました。
これならTerraformのコードを作り込んでやればVPCに依存するリソース等との共存も可能になりそうですね。

---
# 後片付け(環境の破棄)
```bash
$ apex infra destroy
$ apex delete
```
それぞれプロンプトでは`yes`と答えればOK。

# 参考にしたサイト
+ [Apex公式サイト](https://apex.run/)
+ [Terraform公式サイト(Serverless Applications with AWS Lambda and API Gateway)](https://learn.hashicorp.com/terraform/aws/lambda-api-gateway)

