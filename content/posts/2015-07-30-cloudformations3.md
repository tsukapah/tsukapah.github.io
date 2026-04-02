---
title: CloudFormationを使ってS3で静的コンテンツを公開する
date: '2015-07-30T19:49:25+09:00'
lastmod: '2022-04-04T14:27:09+09:00'
draft: false
tags:
- AWS
- S3
- CloudFormation
- aws-cli
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/publish-static-content-on-s3-using-cloudformation/)に移行しました。
params:
  qiita_url: https://qiita.com/tsukapah/items/b65ae3794cccad599b32
  qiita_id: b65ae3794cccad599b32
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/publish-static-content-on-s3-using-cloudformation/)に移行しました。

[これ](http://qiita.com/tsukapah/items/96e8402181609518fd3d "aws-cliを使ってS3で静的コンテンツを公開する")と同じ内容をCloudFormationで実現します

#前提
+ aws-cliが入っていること
+ IAMのパーミッションがいい感じに設定されていること
+ Credentialがconfigureもしくは環境変数に設定されてること
	※ `--profile`使ってもいいです

+ default region が ap-northeast-1 で設定してあること
	※ 別にいいんだけど適宜いい感じに読み替えて下さい

+ Mac想定してるのでWindowsの人はEC2使うとかいい感じに読み替えるとかしてください
+ loggingの設定はこの手順には含まれてません


#手順
1. CloudFormationTemplateの作成
2. Stackを作成するShellScriptの作成
3. Stackの作成
4. コンテンツの設置

#CloudFormationTemplateの作成
```public.json
{
  "AWSTemplateFormatVersion" : "2010-09-09",

  "Mappings" : {
    "RegionMap" : {
      "us-east-1"      : { "s3BucketDomain" : "us-east-1.amazonaws.com" },
      "us-west-1"      : { "s3BucketDomain" : "us-west-1.amazonaws.com" },
      "us-west-2"      : { "s3BucketDomain" : "us-west-2.amazonaws.com" },
      "eu-west-1"      : { "s3BucketDomain" : "eu-west-1.amazonaws.com" },
      "sa-east-1"      : { "s3BucketDomain" : "sa-east-1.amazonaws.com" },
      "cu-north-1"     : { "s3BucketDomain" : "cu-north-1.amazonaws.com" },
      "ap-northeast-1" : { "s3BucketDomain" : "ap-northeast-1.amazonaws.com" },
      "ap-southeast-1" : { "s3BucketDomain" : "ap-southeast-1.amazonaws.com" },
      "ap-southeast-2" : { "s3BucketDomain" : "ap-southeast-2.amazonaws.com" }
    }
  },

  "Parameters" : {
    "FQDN" : {
      "Type" : "String",
      "Description" : "The DNS name of service FQDN"
    }
  },

  "Resources" : {
    "S3BucketForWebsite" : {
      "Type" : "AWS::S3::Bucket",
      "Properties" : {
        "AccessControl" : "PublicRead",
        "BucketName" : { "Ref" : "FQDN" },
        "WebsiteConfiguration" : {
          "ErrorDocument" : "error.html",
          "IndexDocument" : "index.html"
        }
      }
    },
    "BucketPolicyForWebsite" : {
      "Type" : "AWS::S3::BucketPolicy",
      "Properties" : {
        "Bucket" : { "Ref" : "S3BucketForWebsite" },
        "PolicyDocument" : {
          "Id" : "PublicRead",
          "Version": "2012-10-17",
          "Statement" : [ {
            "Sid" : "ReadAccess",
            "Action" : [ "s3:GetObject" ],
            "Effect" : "Allow",
            "Resource" : { "Fn::Join" : ["", ["arn:aws:s3:::", { "Ref" : "S3BucketForWebsite" } , "/*" ]]},
            "Principal" : "*"
          } ]
        }
      }
    }
  },

  "Outputs" : {
    "WebsiteURL" : {
      "Value" : { "Fn::GetAtt" : [ "S3BucketForWebsite", "WebsiteURL" ] },
      "Description" : "URL for website hosted on S3"
    }
  }
}
```

#Stackを作成するShellScriptの作成
```s3hosting.sh
BUCKETNAME="hogehogehoget"
aws cloudformation create-stack \
	--stack-name s3website \
	--parameters \
		ParameterKey=FQDN,ParameterValue="${BUCKETNAME}" \
	--template-body "`cat ./public.json`"
```

#Stackの作成
```bash
$ bash s3hosting.sh
{
    "StackId": "arn:aws:cloudformation:ap-northeast-1:928303975683:stack/s3website/0e1b3520-bb1c-11e4-b04c-50a676d45896"
}
```
##Stackの進行状況確認
```bash
$ aws cloudformation describe-stacks
```
`StackStatus`の値が`CREATE_COMPLETE`になれば環境は出来上がっています。

#コンテンツの設置
コンテンツはカレントディレクトリにあるものとして記載しています。
置いてあるファイルは`/index.html`と`/css/index.css`の2つです。

```bash:command
$ SC_DIR=$(pwd)
$ BUCKETNAME="hogehogehoget"
$ aws s3 sync ${SC_DIR}/ s3://${BUCKETNAME}/
```
```bash:出力結果
upload: css/index.css to s3://hogehogehoget/css/index.css
upload: ./index.html to s3://hogehogehoget/index.html
```
##表示確認
```bash:command
$ open http://${BUCKETNAME}.s3-website-ap-northeast-1.amazonaws.com
```

