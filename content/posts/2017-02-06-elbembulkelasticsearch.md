---
title: ELBのログをEmbulkでElasticsearchに入れたメモ
date: '2017-02-06T22:07:45+09:00'
lastmod: '2022-04-21T17:13:35+09:00'
draft: false
tags:
- AWS
- S3
- elb
- Elasticsearch
- Embulk
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/elb-elasticsearch-with-embulk/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/e04950c8e73d354efbfd
  qiita_id: e04950c8e73d354efbfd
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/elb-elasticsearch-with-embulk/)に移行しました

# 前提

Elasticsearch : 5.2.0
Embulk : v0.8.16

ELBのログについては[こちら](http://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/classic/access-log-collection.html)を参照

# Elasticsearch Mapping
## Elasticsearchのインデックステンプレートを定義する

以下をelblog.jsonとして保存

```elblog.json
{
  "template": "elblog",
  "mappings": {
    "log": {
      "properties": {
        "timestamp": {
          "type": "date"
        },
        "elb": {
          "type": "string",
          "index": "not_analyzed"
        },
        "client_port": {
          "type": "string",
          "index": "not_analyzed"
        },
        "backend_port": {
          "type": "string",
          "index": "not_analyzed"
        },
        "request_processing_time": {
          "type": "float"
        },
        "backend_processing_time": {
          "type": "float"
        },
        "response_processing_time": {
          "type": "float"
        },
        "elb_status_code": {
          "type": "string",
          "index": "not_analyzed"
        },
        "backend_status_code": {
          "type": "string",
          "index": "not_analyzed"
        },
        "received_bytes": {
          "type": "long"
        },
        "sent_bytes": {
          "type": "long"
        },
        "request": {
          "type": "string",
          "index": "not_analyzed"
        },
        "user_agent": {
          "type": "string",
          "index": "not_analyzed"
        },
        "ssl_cipher": {
          "type": "string",
          "index": "not_analyzed"
        },
        "ssl_protocol": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```

## Elasticsearchにインデックステンプレートを登録する
\<es_host>は適宜自分の環境に合わせること

```bash
$ curl -XPUT <es_host>:9200/_template/elblog -d "$(cat elblog.json)"
```

# Embulkでデータの投入
## Embulkのプラグイン導入

```bash
$ embulk gem install embulk-input-s3
$ embulk gem install embulk-output-elasticsearch_ruby
```

## Embulkの設定ファイル作成
以下をconfig.ymlとして保存
\<Bucket_Name>などは適宜自分の環境に合わせること

```config.yml
in:
  type: s3
  bucket: <Bucket_Name>
  path_prefix: hogehoge/AWSLogs/123456789012/elasticloadbalancing/ap-northeast-1/2017/02/05/
  endpoint: s3-ap-northeast-1.amazonaws.com
  access_key_id: HOGEHOGEHOGEHOGE
  secret_access_key: HOGEHOGEHOGEHOGE
  parser:
    charset: UTF-8
    newline: LF
    type: csv
    delimiter: ' '
    quote: ''
    escape: ''
    trim_if_not_quoted: false
    skip_header_lines: 0
    allow_extra_columns: true
    allow_optional_columns: false
    columns:
      - name: timestamp
        type: string
      - name: elb
        type: string
      - name: client_port
        type: string
      - name: backend_port
        type: string
      - name: request_processing_time
        type: double
      - name: backend_processing_time
        type: double
      - name: response_processing_time
        type: double
      - name: elb_status_code
        type: string
      - name: backend_status_code
        type: string
      - name: received_bytes
        type: double
      - name: sent_bytes
        type: double
      - name: request
        type: string
      - name: user_agent
        type: string
      - name: ssl_cipher
        type: string
      - name: ssl_protocol
        type: string
out:
  type: elasticsearch_ruby
  mode: normal
  nodes:
    - {host: "<es_host>", port: 9200}
  index: elb_log
  index_type: log
```

## データの投入

```bash
$ embulk run ./config.yml
```

