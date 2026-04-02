---
title: AWS Fargateを使用してNginxを起動した(ENI使用)
date: '2017-12-05T21:36:02+09:00'
lastmod: '2017-12-05T21:36:02+09:00'
draft: false
tags:
- AWS
- Docker
- Fargate
description: AWS re:Invent2017でAWSが発表したコンテナホスト(インスタンス)を意識せずにコンテナを管理できるオーケストレーションサービスです。
params:
  qiita_url: https://qiita.com/tsukapah/items/eac3657b902f402af7d7
  qiita_id: eac3657b902f402af7d7
---

# AWS Fargateとは
AWS re:Invent2017でAWSが発表したコンテナホスト(インスタンス)を意識せずにコンテナを管理できるオーケストレーションサービスです。
現時点ではバージニア(us-east-1)リージョンのみでGAとなっています。
ちなみに、re:Inventに参加したので、現地でキーノートを聞いていましたが、同時にEKSも発表され待望のサービスが来た！という感じで私も会場もめちゃくちゃ興奮してました。

# やってみる

## クラスタの作成
リージョンは`バージニア`を選択し、`Elastic Container Service`の画面を開きます。
`クラスターの作成`ボタンを押してクラスタの作成を始めます。
(私の環境は元々別のクラスタが作成してあったので初めての場合表示が違うかもしれません)
![console_aws_amazon_com_ecs_home_region_us-east-1.png](https://qiita-image-store.s3.amazonaws.com/0/32208/24d82d14-c2ec-0665-c464-1f67b6bdc33b.png)

### クラスタテンプレートの選択
以下の3つのクラスタテンプレートが選択できます。

+ Network only
+ EC2 Linux + Networking
+ EC2 Windows + Networking

`Network only`がFargateのようなので、こちらを選択して`次のステップ`へ進みます。
![console_aws_amazon_com_ecs_home_region_us-east-1__1_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/d5eed508-4ac2-9ba1-b06a-2585f798e671.png)

### クラスタの設定
ここではクラスタ名とネットワークの設定をします。
クラスタ名は`FargateTest`にしてみました。
また、このクラスタ用にVPCも合わせて作ってくれるオプションがあったので選択しました。
値はデフォルトのままで`作成`。
![console_aws_amazon_com_ecs_home_region_us-east-1__2_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/09553e5a-4f52-59b4-83c5-3d7744f23b9f.png)

### 起動ステータスの確認
作成が始まるとCloudFormationスタックが走ります。
しばらくするとイイ感じにクラスタができあがります。
![console_aws_amazon_com_ecs_home_region_us-east-1__3_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/f8987abf-3e9f-1eba-1e4d-45252488000e.png)

### クラスタの状態を確認
当然ですが実行中のタスクもサービスもありません。
![console_aws_amazon_com_ecs_home_region_us-east-1__4_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/f56bc4bf-7ab7-74a6-e0cb-6c74afe2d225.png)

## タスク定義作成
新しくNginxを起動するタスクを定義します。
(私の環境は元々別のタスクが登録してあったので初めての場合表示が違うかもしれません)
![console_aws_amazon_com_ecs_home_region_us-east-1__5_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/389c0d57-74a4-8e37-1850-82cc20d81fed.png)

### ローンチタイプの選択
`FARGATE`と`EC2`が選択できるので、当然`FARGATE`を選んで`次のステップ`に進みます。
![console_aws_amazon_com_ecs_home_region_us-east-1__6_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/63b4bd2b-b377-dfab-6d61-df6cba5be18a.png)

### タスク定義の設定
適当にタスク定義に名前をつけます。ここでは`TestNginx`としました。
タスクのサイズ指定でメモリとvCPUを選択できます。
メモリは0.5GBと1GB~30GBまで1GB刻みで調整できるようです。
CPUは0.25、0.5、1、2、4から選択します。
ここではどちらも最小値の0.5GB、0.25vCPUを選択しました。
続いて`コンテナの追加`を行います。
![console_aws_amazon_com_ecs_home_region_us-east-1__7_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/2fafcdb9-5007-443c-8b0e-23b93d1668f3.png)

### コンテナの追加
コンテナ名を適当に定義します。ここでは`TestNginx`にしました。
また、使用するコンテナイメージを指定します。DockerHubで公開されているNginxイメージをそのまま指定しました。
Nginxコンテナはポート80でListenするのでポートマッピングの設定は`80`となります。
その他は特に変更せずに`追加`し、そのまま`作成`します。
![console_aws_amazon_com_ecs_home_region_us-east-1__8_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/c0020f8f-e1aa-44cf-e90e-e40ca1ad7b36.png)

### ローンチステータスの確認
ECSタスクと必要なIAM Roleが定義されたようです。
![console_aws_amazon_com_ecs_home_region_us-east-1__9_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/67e1dacc-dbbb-e412-29c8-a093d738a507.png)

## サービスの作成
クラスタの一覧から今回作成した`FargateTest`を選択します。
![console_aws_amazon_com_ecs_home_region_us-east-1__11_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/4c37ce46-a2e5-f6e0-0ab5-80822067f6c6.png)

### サービスの作成
`サービス`タブ内から`作成`します。
![console_aws_amazon_com_ecs_home_region_us-east-1__12_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/e960c361-0fbb-9a52-af77-7ae11003b3c7.png)

### サービスの設定
Launch Typeは当然`FARGATE`を選択します。
また、タスク定義は先程作成した`TestNginx`を指定します。
サービス名には適当な名前をつけます。ここでは`TestNginx`としました。
![console_aws_amazon_com_ecs_home_region_us-east-1__13_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/bb9f946e-dfac-1c2c-d7c4-c41f81194ee1.png)

### ネットワーク構成の設定
参加するVPCを指定します。たぶん最初に作ったVPC IDが指定されていると思います。
コンテナが起動するサブネットを追加します。AZごとに別れていると思います。
また、セキュリティグループを`編集`することで既存のセキュリティグループを指定したり新規作成の内容を微調整できるようです。デフォルトのままだとポート80をAnyで公開します。
今回はELBを使用せずに直接コンテナに接続してみたいのでELBタイプは`なし`を選択しました。
このまま`次のステップ`に進みます。
![console_aws_amazon_com_ecs_home_region_us-east-1__14_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/0d7eb313-64b9-83aa-6648-e405c34d3121.png)

### オートスケールの設定
オートスケールの設定ができますが、今回は実施しないのでそのまま`次のステップ`に進みます。
![console_aws_amazon_com_ecs_home_region_us-east-1__15_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/9637c094-cabc-035f-3094-bcce38174550.png)

### サービスの確認
内容を確認して`サービスの作成`をします。
![console_aws_amazon_com_ecs_home_region_us-east-1__16_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/620e1136-b974-b682-684e-f56c392d9f79.png)

### 作成ステータスの確認
セキュリティグループの設定やサービスの設定が進みます。
![console_aws_amazon_com_ecs_home_region_us-east-1__17_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/b7845f8b-f9fa-01e6-62cd-de68c7ca083a.png)

## 動作確認
### タスクのステータス確認
クラスタ内のタスクタブでステータスが`RUNNING`になっていることを確認します。少し時間がかかる場合があります。
起動中のタスクIDを選択して詳細を確認します。
![console_aws_amazon_com_ecs_home_region_us-east-1__18_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/9bfd1811-4315-50ab-db10-eec0f6f09553.png)

### タスクの詳細確認
Networkの部分を見ると、ENI idがアタッチされていることが確認できます。
コンテナは外部に対してこのENIで通信します。
ENI IDを選択するとENIの画面に遷移します。
![console_aws_amazon_com_ecs_home_region_us-east-1__19_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/8424f473-ea00-b955-7b73-fa761bf5f1af.png)

### ENIの確認
ENIの画面に遷移したら、IPv4パブリックIPの部分を参照します。
ここに表示されているIPアドレスでコンテナと通信できます。
![us-east-1_console_aws_amazon_com_ec2_v2_home_region_us-east-1.png](https://qiita-image-store.s3.amazonaws.com/0/32208/5dfbf524-4a5e-9f3f-70e0-582f47d21a09.png)

### Nginxにアクセス
ブラウザで先程確認したIPアドレスにアクセスしてみます。
<img width="1267" alt="スクリーンショット 2017-12-05 17.28.17.png" src="https://qiita-image-store.s3.amazonaws.com/0/32208/6fe5612e-88d5-f324-368d-280dc122af4c.png">
無事Nginxが起動していました。

## 後片付け
作成したクラスタを削除します。
...と言いたいところですが、単純に削除したところ途中でエラーが出ました。
サービスを作成した際にできたセキュリティグループが残ってしまうので、CloudFormationStackが削除できないことが原因でした。
おそらく正しい手順は`サービスの削除` -> `セキュリティグループの削除` -> `クラスタの削除`となります。
まさか最後にアンチパターンにぶつかるとは。。。
![console_aws_amazon_com_ecs_home_region_us-east-1__20_.png](https://qiita-image-store.s3.amazonaws.com/0/32208/a63c0b6f-3173-59ad-b87b-ee5ee8f30dec.png)

# 最後に
インスタンスを意識する必要がないので保守範囲が減り、期待通りかなり魅力あるサービスだと感じました。
運用面では監視に課題が出てきそうなのかなーと思いました。
なんにせよ、Terraformへの組み込みと、東京リージョンでのサービス開始が待ち遠しいです。

