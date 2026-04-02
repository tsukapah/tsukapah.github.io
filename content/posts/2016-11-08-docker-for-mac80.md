---
title: Docker for Macでローカルの80番ポートにマッピングしたらつまづいた
date: '2016-11-08T15:47:05+09:00'
lastmod: '2022-04-17T23:00:20+09:00'
draft: false
tags:
- Mac
- Docker
description: ※当故事は[個人ブログ](https://tech.tsukapah.com/docker-for-mac-port80/)に移行しました
params:
  qiita_url: https://qiita.com/tsukapah/items/3f0660ad7a27a4585bf3
  qiita_id: 3f0660ad7a27a4585bf3
---

※当故事は[個人ブログ](https://tech.tsukapah.com/docker-for-mac-port80/)に移行しました

Macを新しくしたのでdocker-machineからDocker for Macに乗り換えてみた。
nginxコンテナをローカルの80番にマッピングして動作確認しようとしたら微妙につまづいたのでメモ。

```bash
$ docker run --name nginx -d -p 80:80 nginx
241b1dc789fe4d699af662c872ca3703053b77715e56ef307919fa4fb5457d6b
$ curl localhost
curl: (52) Empty reply from server
```

おや？レスポンスが返ってこない。
リクエストヘッダを見てみる。

```bash
$ curl -v localhost
* Rebuilt URL to: localhost/
*   Trying ::1...
* Connected to localhost (::1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.43.0
> Accept: */*
>
* Empty reply from server
* Connection #0 to host localhost left intact
curl: (52) Empty reply from server
```

`* Connected to localhost (::1) port 80 (#0)`とのことなのでIPv6アドレスにリクエスト投げてるっぽい。
hostsを確認。

```bash
$ cat /etc/hosts
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost
```

IPv6アドレスをコメントアウトし再トライ。

```bash
$ cat /etc/hosts
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
#::1             localhost
```
```bash
$ curl -v localhost
* Rebuilt URL to: localhost/
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.43.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.11.5
< Date: Tue, 08 Nov 2016 05:27:01 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Tue, 11 Oct 2016 15:03:01 GMT
< Connection: keep-alive
< ETag: "57fcff25-264"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host localhost left intact
```

期待通り返ってきた。
localhostをIPv6で名前解決できないとダメな理由がない限り、このIPv6エントリはコメントアウトでいいと思います。

