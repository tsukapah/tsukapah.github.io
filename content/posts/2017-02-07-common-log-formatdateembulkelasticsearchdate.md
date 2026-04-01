---
title: Common Log Formatのdate型ログをEmbulkでElasticsearchにdate型で入れたメモ
date: '2017-02-07T21:53:46+09:00'
lastmod: '2022-04-21T17:18:34+09:00'
draft: false
tags:
- Elasticsearch
- Embulk
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/common-log-format-elasticsearch-embulk/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/66cdea20724f13513f9d
  qiita_id: 66cdea20724f13513f9d
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/common-log-format-elasticsearch-embulk/)に移行しました

なんか苦労したのでメモ。
#読み込むログ

```log.json
{"time_local":"05/Feb/2017:03:38:34 +0900","hoge":"piyo"}
```

NginxやApacheでよく見かけるフォーマット。

#Embulkの設定

Elasticsearch的にUTCで入れたほうが扱いやすいので`timestamp`型で取り込んでUTCに戻してあげる。

```config.yml
in:
  type: file
  path_prefix: /path/to/json/
  parser:
    type: jsonl
    charset: UTF-8
    newline: LF
    columns:
      - { name: "time_local", type: "timestamp", format: "%d/%b/%Y:%T %z" }
      - { name: "hoge", type: "string" }
out:
  type: elasticsearch_ruby
  cluster_name: elasticsearch
  mode: normal
  nodes:
    - {host: "es_host", port: 9200}
  index: jsonlog
  index_type: log
```

このEmbulkの定義で読み込むと`time_local`の値は以下の感じで出力される。

```
2017-02-04 18:38:34 UTC
```

#Elasticsearchのインデックステンプレート

EmbulkでUTCに戻すのはいいが、このまま突っ込むだけだとElasticsearchはdate型としては受け入れてくれない。
突っ込む前にフォーマットを定義してマッピングしておかないと、文字列としてドキュメントが登録されてしまう。

```template.json
{
  "template": "jsonlog",
  "mappings": {
    "log": {
      "properties": {
        "time_local": {
          "type": "date",
          "format": "YYYY-MM-dd HH:mm:ss' UTC'"
        },
        "hoge": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```

```bash
$ curl -XPUT es_host:9200/_template/jsonlog?pretty -d "$(cat template.json)"
```

あとは上の設定でEmbulkを使ってbulkインポートしてあげれば時系列データとして意味のあるログになる。

