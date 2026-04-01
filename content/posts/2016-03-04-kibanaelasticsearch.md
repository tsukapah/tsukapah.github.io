---
title: Kibanaの設定をまるっと別のElasticsearchに移したい(改訂版)
date: '2016-03-04T19:24:33+09:00'
lastmod: '2022-04-09T22:33:11+09:00'
draft: false
tags:
- Elasticsearch
- Kibana
description: ※当記事は[個人ブログ](https://tech.tsukapah.com/new-migrate-kibana-settings/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/91cb093ddfa6ede10b02
  qiita_id: 91cb093ddfa6ede10b02
---

※当記事は[個人ブログ](https://tech.tsukapah.com/new-migrate-kibana-settings/)に移行しました

以前[こんな](http://qiita.com/tsukapah/items/307d05572fdb1cb356b5)記事を書いたんですが、
そもそもKibana自身に設定をexport/importできる機能があることに気がついたので、
使ってみたいと思います。

# 検証環境
[これ](http://qiita.com/tsukapah/items/24bab0f5ab9bb4e12309)で用意。

```elasticsearch
{
  "name" : "Zarek",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.1.0",
    "build_hash" : "72cd1f1a3eee09505e036106146dc1949dc5dc87",
    "build_timestamp" : "2015-11-18T22:40:03Z",
    "build_snapshot" : false,
    "lucene_version" : "5.3.1"
  },
  "tagline" : "You Know, for Search"
}
```
![Kibana Version](https://qiita-image-store.s3.amazonaws.com/0/32208/a461fec4-1fce-277b-4988-713d488c4ed4.png)

# 手順
## Visualizeを作成する
移行するものがないと始まらないのでとりあえずVisualizeを作成します。
ちなみにすでにとあるWebサイトのアクセスログをサンプルとして入れてあるので、
トップページの時系列アクセス数のグラフにしてみます。

1. 上部メニューから`Visualize`を選び、グラフの種類は`Vertical bar chart`を選択します。
1. 入力元を選びます。  
新しいVisualizeを作るので`From a new search`を選びます。
1. Y軸(縦軸)の設定をします。Y-Axisの *Aggregation* を`count`にします。  
というかたぶん最初からそうなってると思います。  
この時点での見た目はこんな感じです。
![Kobito.CaCSV2.png](https://qiita-image-store.s3.amazonaws.com/0/32208/25ea0a05-94b3-d54a-d4e9-6df226b0d0f5.png "Kobito.CaCSV2.png")
1. Select buckets typeから`X-Axis`を選択します。  
これでX軸(横軸)が追加できるようになります。
1. X-Axisの *Aggregation* を`Date Histogram`にします。  
*Field* は`@timestamp`、 *Interval* はお好みに合わせてください。  
今回は`Auto`にしておきます。  
いったんこの状態でグラフを描画してみましょう。  
右向きの三角形を押すとグラフが更新されます。  
この時点での見た目はこんな感じです。
![Kobito.4LNmdn.png](https://qiita-image-store.s3.amazonaws.com/0/32208/f15db381-a70b-79bd-46d2-9c482deea9bc.png "Kobito.4LNmdn.png")
1. 続いてトップページだけにフィルタリングを行います。  
`Add sub-buckets`を押します。  
選択肢が出てきますがとりあえず今回は`Split Bars`にしておきます。  
*Sub Aggregation* から`Filters`を選びます。  
*Filter 1* に絞り込みたい文字列を入力します。  
今回はrequestというFieldの値が`"GET / HTTP/1.1"`に絞り込んでみます。  
テキストボックスに`request:"GET / HTTP/1.1"`を入力します。  
フィルタの設定ができたのでグラフを更新しましょう。 
右向き三角系を押します。  
こんな感じのグラフになりました。
![Kobito.bwZGvU.png](https://qiita-image-store.s3.amazonaws.com/0/32208/0dd6b46a-7c3f-dbea-17da-97305ee93651.png "Kobito.bwZGvU.png")
1. 作成したVisualizeを保存します。  
右上の方にあるフロッピーディスクっぽいボタンを押します[^1]。  
[^1]: どーでもいいですが、きっと若い人はフロッピーディスク見たことないですよね。見たことない人はSDカードっぽい絵のやつがそれです。
テキストボックスに好きな名前を入れて *save* します。  
今回は`TopPageHistgram`にしてみました。  
これで無事Visualizeが保存できました。

## Dashboardを作成する
せっかくなのでDashboardも作っちゃいます。

1. 上部メニューから`Dashboard`を選びます。  
初めてDashboardを作るのであれば*+*ボタンが画面中央あたりにあるので押します。
1. 画面を作ります。  
先ほど作った`TopPageHistgram`が選択できる状態になっているので選びます。  
画面に先ほどのグラフが小さく出てきたと思います。  
あとはグラフの右下をひっぱって好きな大きさにします。  
こんな感じになりました。
![Kobito.oaqexy.png](https://qiita-image-store.s3.amazonaws.com/0/32208/b4a4ea75-1ade-807b-b10f-ea476448a0c5.png "Kobito.oaqexy.png")
1. 作成したDashboardを保存します。  
右上のフロッピーディスクっぽいボタンを押します。  
テキストボックスに好きな名前を入れて *save* します。  
今回は`WebSiteAccess`にしてみました。  
これで無事Dashboardが保存できました。

## Kibanaの設定をexportする
前置きが長くなりましたがいよいよ本題に入ります。

1. 上部メニューから`Setting`を選びます。
2. その下のサブメニューから`Objects`を選びます。
3. `Export Everything`を押すとKibanaの設定がダウンロードできます。  
下のチェックボックスを入れたりする部分で個別にダウンロードすることもできます。  
が、僕の環境では`Export Everything`を押したらいくつかのPOSTリクエストで *500 Internal Server Error*
が出た。。。  
とりあえずVisualizationsだけダウンロードしてみました[^2]。  
[^2]: 解決するのが面倒なので...

タブから`Visualizations`を選択し、`TopPageHistgram`にチェックを入れた状態で`Export`ボタンを押します。  
画面はこんな感じ。
![Kobito.57Lmex.png](https://qiita-image-store.s3.amazonaws.com/0/32208/ae46f819-be63-a274-dd53-f3361ff14430.png "Kobito.57Lmex.png")
*export.json* がダウンロードできたと思います。

```export.json
[
  {
    "_id": "TopPageHistgram",
    "_type": "visualization",
    "_source": {
      "title": "TopPageHistgram",
      "visState": "{\"type\":\"histogram\",\"params\":{\"shareYAxis\":true,\"addTooltip\":true,\"addLegend\":true,\"scale\":\"linear\",\"mode\":\"stacked\",\"times\":[],\"addTimeMarker\":false,\"defaultYExtents\":false,\"setYExtents\":false,\"yAxis\":{}},\"aggs\":[{\"id\":\"1\",\"type\":\"count\",\"schema\":\"metric\",\"params\":{}},{\"id\":\"2\",\"type\":\"date_histogram\",\"schema\":\"segment\",\"params\":{\"field\":\"@timestamp\",\"interval\":\"auto\",\"customInterval\":\"2h\",\"min_doc_count\":1,\"extended_bounds\":{}}},{\"id\":\"3\",\"type\":\"filters\",\"schema\":\"group\",\"params\":{\"filters\":[{\"input\":{\"query\":{\"query_string\":{\"query\":\"request:\\\"GET / HTTP/1.1\\\"\",\"analyze_wildcard\":true}}},\"label\":\"\"}]}}],\"listeners\":{}}",
      "uiStateJSON": "{}",
      "description": "",
      "version": 1,
      "kibanaSavedObjectMeta": {
        "searchSourceJSON": "{\"index\":\"accesslog-*\",\"query\":{\"query_string\":{\"query\":\"*\",\"analyze_wildcard\":true}},\"filter\":[]}"
      }
    }
  }
]
```
これでKibanaの設定が(一部ですが)exportできました。

## Kibanaの設定をimportする
本来は別のelasticスタックで試すのがいいんですが、(用意するのが面倒なので)Visualizationsを削除しちゃいます。
先ほどexportした画面のままになっていると思うので、`TopPageHistgram`にチェックを入れて`Delete`ボタンを押します。
ポップアップが出てほんとにいいか聞いてきますが、`OK`を押します。
上部メニューから`Visualize`を選んでも`TopPageHistgram`はもういません。
ここから、先ほどの設定(export.json)をimportします。

1. 上部メニューから`Setting`を選びます。
2. その下のサブメニューから`Objects`を選びます。
3. `import`ボタンを押すと、Finderあたりが開くので、先ほどの`export.json`を選択します。  
*Visualizations* に先ほど消した`TopPageHistgram`が戻ってきたと思います。  
上部メニューから`Visualize`を選択すると、`TopPageHistgram`が選べます。
無事グラフも見れたと思います。

