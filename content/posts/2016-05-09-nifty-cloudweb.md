---
title: NIFTY CloudのWebコンソールからサーバを操作する
date: '2016-05-09T13:52:02+09:00'
lastmod: '2016-05-09T20:39:33+09:00'
draft: false
tags:
- NiftyCloud
description: VMインポートに失敗した時とか、iptablesの設定ミスっちゃったとかでsshできない時、LDAPの設定ミスって一般ユーザでログイン出来ないんだけどrootにssh許してない時とかに有効です。
params:
  qiita_url: https://qiita.com/tsukapah/items/04c12a642ac4f2ac9002
  qiita_id: 04c12a642ac4f2ac9002
---

VMインポートに失敗した時とか、iptablesの設定ミスっちゃったとかでsshできない時、LDAPの設定ミスって一般ユーザでログイン出来ないんだけどrootにssh許してない時とかに有効です。
以下の内容は[公式] (http://cloud.nifty.com/service/console.htm)に書いてあることと変わりません。
# OS
Windows必須
※ Mac、Linuxはブラウザのプラグインがないので使えない、つらい...

# ブラウザ
コンソール|クライアントOS|対応ブラウザ
---|---|---
64bit|64bit|Internet Explorer 10/11、Firefox 最新版、Chrome 最新版
32bit|64bit/32bit|Internet Explorer 10、Firefox 最新版

+ Win7で32bitでChromeが **使えない** ことは確認した
+ Win7で32bitでFireFoxが **使える** ことは確認した
+ ~~IEはなくなry~~

# 通信ポート
20,000以上のハイポート

# 注意点
+ 1サーバ１セッションで、後勝ちな感じっぽいので誰かのセッション奪ってしまうこともありそう

# コンソールエラー内容
http://cloud.nifty.com/guide/cp/login/console_error.htm

