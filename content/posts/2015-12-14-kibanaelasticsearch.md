---
title: Kibanaの設定をまるっと別のElasticsearchに移したい！
date: '2015-12-14T17:16:37+09:00'
lastmod: '2022-04-06T23:09:29+09:00'
draft: false
tags:
- Elasticsearch
- Kibana
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/migrate-kibana-settings/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/307d05572fdb1cb356b5
  qiita_id: 307d05572fdb1cb356b5
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/migrate-kibana-settings/)に移行しました

新しく[こちら](http://qiita.com/tsukapah/items/91cb093ddfa6ede10b02)の記事を書きましたのでそちらを参照することをオススメします。

---

Kibanaの設定は全部Elasticsearchの`.kibana`というインデックスに入っているので、単純にインデックスのお引越しの手順になります。

Snapshot/Restoreを直接APIから実行するのもそんなにむずかしいことではありませんが、
もっと簡単にできそうだったのでESClientというツールを使ってみます。

# 確認した環境
+ Elasticsearch 2.1.0
+ Kibana 4.3.0

# 手順
## ESClientインストール
```bash
$ pip install esclient
※必要に応じてsudoをつけてください
```

## .kibanaインデックスのエクスポート
```bash
$ esdump --url http://<Old Elasticsearch Host>:<Old Elasticsearch Port>/ --indexes .kibana --bzip2 --file kibana.bz2
```

## .kibanaインデックスのインポート
```bash
$ esimport --url http://<New Elasticsearch Host>:<New Elasticsearch Port>/ --file kibana.bz2
```

# 補足
移行対象のデータに日本語が含まれていると、エンコードエラーが出るかもしれません。
その場合、pythonのデフォルトエンコードをUTF8にするなどで回避できます。

---

とりあえず動作確認の環境が欲しければ[こちら](http://qiita.com/tsukapah/items/24bab0f5ab9bb4e12309)もどうぞ。

