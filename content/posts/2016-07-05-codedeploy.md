---
title: CodeDeployの上限に引っかかった話
date: '2016-07-05T14:19:55+09:00'
lastmod: '2022-04-10T17:55:53+09:00'
draft: false
tags:
- AWS
- limit
- CodeDeploy
description: ※こちらの記事は[個人ブログ](https://tech.tsukapah.com/limited-codedeploy/)に移行しました。
params:
  qiita_url: https://qiita.com/tsukapah/items/5e72ac521a20b0619208
  qiita_id: 5e72ac521a20b0619208
---

※こちらの記事は[個人ブログ](https://tech.tsukapah.com/limited-codedeploy/)に移行しました。

AutoScaleでEC2を100台にした後にデプロイしてみたらFailしました。
![codedeploy.jpg](https://qiita-image-store.s3.amazonaws.com/0/32208/3e09aa5c-df76-5d69-5166-36c6e7ba35af.jpeg "codedeploy.jpg")
どうやら[Limit](http://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/limits.html)に引っかかったようです。

本番当日だったので泣きそうになりましたが、
まずは落ち着いて[request a limit increase](https://console.aws.amazon.com/support/home#/case/create%3FissueType=service-limit-increase)しつつ、いったんEC2を50台まで減らしてからデプロイし、改めて100台に増やしました。
使うプロダクトの上限は把握したうえで計画的に利用しましょう。
もしくは、本番当日にデプロイしないでも済むように事前に調整しましょう。

