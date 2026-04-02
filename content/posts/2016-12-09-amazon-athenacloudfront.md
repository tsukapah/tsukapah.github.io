---
title: Amazon AthenaでCloudFrontのログにクエリを投げたメモ
date: '2016-12-09T18:57:01+09:00'
lastmod: '2022-04-17T22:57:53+09:00'
draft: false
tags:
- AWS
- CloudFront
- Athena
description: ※当記事は[個人ブログ](https://tech.tsukapah.com/athena-cloudfront/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/bbc750d7f2c92b29fe29
  qiita_id: bbc750d7f2c92b29fe29
---

※当記事は[個人ブログ](https://tech.tsukapah.com/athena-cloudfront/)に移行しました

AWS re:Invent 2016 で発表された新サービス Amazon Athenaを早速(?)使ってみました。
S3にログを置いたままサーバレスでクエリが発行できるので集計等が捗ります。
すでに同様の記事も見かけましたが、デリミタの指定でうまくいかずにハマったのでメモ。
なお、CloudFrontのLogging設定はこの記事では触れません。

# Databaseの作成
```sql
CREATE DATABASE CloudFrontLogs;
```

# Tableの作成
マネージメントコンソール上でも定義できますが、
DDL書くほうが楽な気がしたので、2つくらいカラム定義したあたりでそっとキャンセルしました。

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS <TableName> (
  date DATE,
  time STRING,
  xEdgeLocation STRING,
  scBytes INT,
  cIp STRING,
  csMethod STRING,
  csHost STRING,
  csUriStem STRING,
  scStatus INT,
  csReferer STRING,
  csUserAgent STRING,
  csUriQuery STRING,
  csCookie STRING,
  xEdgeResultType STRING,
  xEdgeRequestId STRING,
  xHostHeader STRING,
  csProtocol STRING,
  csBytes INT,
  timeTaken INT,
  xForwardedFor STRING,
  sslProtocol STRING,
  sslCipher STRING,
  xEdgeResponseResultType STRING,
  csProtocolVersion STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = "\t",
  'field.delim' = "\t"
) LOCATION 's3://<LogBacketName>/<LogPrefix>/'
;
```

CloudFrontのログフォーマットはTSVです。
`serialization.format`と`field.delim`を`'  ' --タブ文字`みたいに表記している記事がいくつかありましたが、[ドキュメント](http://docs.aws.amazon.com/athena/latest/ug/ddl/create-table.html)を見たら普通に`"\t"`でよいと書いてありました。
カラム名はお好みに合わせてよしなに変更してください。

# Queryの発行
```sql
SELECT * FROM <TableName> limit 10;
```
期待通りResultsが返ってくればOK。

