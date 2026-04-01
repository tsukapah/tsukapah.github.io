---
title: 複数行をまとめて置換したかったのでsedとpythonで比べてみた
date: '2018-02-08T22:13:00+09:00'
lastmod: '2018-02-11T17:44:33+09:00'
draft: false
tags:
- Python
- sed
description: '```before.txt'
params:
  qiita_url: https://qiita.com/tsukapah/items/f2235a38592a7082662b
  qiita_id: f2235a38592a7082662b
---

# やりたいこと

```before.txt
aiueo
kakikukeko
sashisuseso
tachitsuteto
naninuneno
hahifuheho
mamimumemo
yayuyo
wawon
```
例えば上記のようなファイルがあったとして、以下のように置換したいとします。

```after.txt
aiueo
gagigugeko
wawon
```

つまり、`kakikukeko`の行から`yayuyo`の行までを`gagigugego`に置き換えたいわけです。

## やってみる

### sedの場合

まずは下記のようなsedコマンドに渡すコマンドをファイルに書きます。

```sed.txt
:a
N
s#kakikukeko.*yayuyo\n#gagigugego\n#
Ta
P
D
```

実行

```bash
$ sed -f sed.txt before.txt
aiueo
gagigugego
wawon
```

置き換わりました。
でも`sed.txt`がワケワカメです。

#### 解説
コマンド|解説
---|---
:a | `T`コマンド用に`a`というラベルを定義してます。
N | 次の行を処理対象に追加します。<br>sedは基本的には単一行の処理をするものですが、これで複数行の処理を実現させます。
s#kakikukeko.*yayuyo\ｎ#gagigugego\ｎ#|置換のためのメイン処理です。<br>Nコマンドの結果複数行が処理対象に含まれます。
Ta|上記のメイン処理が失敗したらここ(ラベル`a`)に飛びます。
P|処理対象の改行以前を出力します。
D|Pで出力した部分を処理対象から削除して、最初のラベル`a`に戻ります。

なんとなく解説してみましたが、やっぱりワケワカメです。
直感的にわからないのであとから見ても何がしたいのかよくわからないと思います。

### Pythonの場合

以下のPythonスクリプトを書きます。

```python:sed.py
import sys
import re

if __name__ == '__main__':
    before_str=r'kakikukeko.*yayuyo'
    after_str=r'gagigugego'

    f = open(sys.argv[1],'r')
    body = f.read()
    print re.sub(before_str, after_str, body, flags=re.DOTALL)
    f.close()
```

実行

```bash
$ python sed.py ./before.txt
aiueo
gagigugego
wawon

```

置き換わりました。
解説要らない程度には直感的です。

#### 解説
コマンド|解説
---|---
import sys|引数を受け取るために`sys`モジュールをインポートします。
import re|正規表現を取り扱うために`re`モジュールをインポートします。
before_str=r'kakikukeko.*yayuyo'|`before_str`に置換前の文字列を正規表現で入れます。
after_str=r'gagigugego'|`after_str`には置換後の文字列を入れます。
f = open(sys.argv[1],'r')<br>body = f.read()|ファイルの中身をまるごと変数に読み込ます。
print re.sub(before_str, after_str, body, flags=re.DOTALL)|`body`の中見から`before_str`に一致する文字列を`after_str`に置換して標準出力に出します。<br>`flags=re.DOTALL`を指定したことで、正規表現の`.`に改行コード`\n`も含まれるようになり、複数行が置換対象になります。
f.close()|ファイルを閉じます。

---
以上。
もうsedは使わないかもしれない。

## ...ということを書いたところ(2018.02.09追記)

@shiracamus さんよりコメントで以下のsedコマンドでいける、とご指摘頂きました。ありがとうございます。
なるほど`c`オプションを理解できてませんでした。

```bash
$ sed '/kakikukeko/,/yayuyo/cgagigugego' before.txt
aiueo
gagigugego
wawon
```

`c`オプションは `開始行,終了行c置換後の文字列` でまとめて複数行を置き換えることができるようです。
開始行/終了行には行番号か正規表現が使用できるようなので、この例は正規表現を利用したケースということになります。

おれはまだsedの本気に触れてなかった。。。
ということで、やっぱりまだsedも使うかもしれない。

# やりたいこと②(2018.02.09追記)
後方参照したいケースもあります。
例えば以下のようなファイルがあります。

```before2.html
<html>
 <head>
  <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
  <!-- test -->
  <!-- テスト -->
  <!-- 試験 -->
 </head>
 <body>
 </body>
</html>
```

このファイルの中から`head`タグの中見だけ取り出したいような場合です。
期待値的には以下の感じです。

```after2.html
  <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
  <!-- test -->
  <!-- テスト -->
  <!-- 試験 -->
```

## やってみる
### sedの場合

まずは下記のようなsedコマンドに渡すコマンドをファイルに書きます。
先程のケースとほとんど同じですが、`s`コマンドの正規表現を含む置換処理が違います。

```sed2.txt
:a
N
s#<html>.*<head>\(.*\)</head>.*</html>#\1#
Ta
P
D
```

実行

```bash
$ sed -f sed2.txt before2.html

  <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
  <!-- test -->
  <!-- テスト -->
  <!-- 試験 -->

```

#### 解説

基本的な解説は上記のとおりなので割愛します。

コマンド|解説
---|---
`s#<html>.*<head>\(.*\)</head>.*</html>#\1#`|正規表現でheadタグの開始タグと終了タグの間の文字列をグルーピングし、後方参照で置換後の文字列に渡します。

このケースだとsedの`c`オプションは使えないのでやっぱりこの書き方になるかなーと思います。

### pythonの場合

以下のPythonスクリプトを書きます。

```python:sed2.py
import sys
import re

if __name__ == '__main__':
    str=r'<html>.*<head>(.*)</head>.*</html>'

    f = open(sys.argv[1],'r')
    body = f.read()
    print re.search(str, body, flags=re.DOTALL).group(1)
    f.close()
```

実行

```bash
$ python sed2.py ./before2.html

  <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
  <!-- test -->
  <!-- テスト -->
  <!-- 試験 -->


```

#### 解説
コマンド|解説
---|---
print re.search(str, body, flags=re.DOTALL).group(1)|正規表現のうち、グルーピングされてる値の1番目を標準出力に出力します。

---
ということで、後方参照するならpythonのほうが直感的でメンテナンスもしやすそうです。

