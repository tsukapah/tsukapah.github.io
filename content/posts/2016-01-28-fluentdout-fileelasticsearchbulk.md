---
title: Fluentdのout_fileプラグインで出力したファイルからElasticsearchにbulkインポートしたくなることってよくありますよね？
date: '2016-01-28T15:07:03+09:00'
lastmod: '2022-04-07T23:38:04+09:00'
draft: false
tags:
- Bash
- Fluentd
- Elasticsearch
description: ※当記事は[個人ブログ](https://tech.tsukapah.com/bulk-import-with-bash/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/044b8d633da49314ed36
  qiita_id: 044b8d633da49314ed36
---

※当記事は[個人ブログ](https://tech.tsukapah.com/bulk-import-with-bash/)に移行しました

bashのワンライナーで強引にbulkインポート用のフォーマットに書き換えちゃいます。
以下のコマンド結果をファイルにリダイレクトするとか、curlの引数にコマンド置換で渡すでもいいです。コマンドが長いのでたいていファイルに書きだしちゃいますが。
全部同じインデックス、ドキュメントタイプにすることを前提にしてます。
あと、時刻のフォーマットを書き換えてる正規表現については必要があれば変えてください。(3つ目の`-e`のとこです)


```bash
$ awk -F '\t' '{print $1","$3}' <対象ファイル> | \
	sed -e 's/{//g' \
		-e 's/^/{"@timestamp":/g' \
		-e 's/\([0-9]\{4\}\)\/\([0-9]\{2\}\)\/\([0-9]\{2\}\) \([0-9]\{2\}:[0-9]\{2\}:[0-9]\{2\}\) \(+[0-9]\{4\}\)/"\1-\2-\3T\4\5"/g' \
		-e 'i\{"index":{}}\'
```

まぁ、他にいくらでも方法あると思いますが...

fluent-plugin-elasticsearchとかで最初から入れろよって話もありますが、それはそれ。

