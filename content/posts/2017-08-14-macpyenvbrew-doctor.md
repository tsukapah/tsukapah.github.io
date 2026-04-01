---
title: Macにpyenvを入れたらbrew doctorで怒られるようになったので対応したメモ
date: '2017-08-14T15:39:37+09:00'
lastmod: '2017-08-14T15:39:37+09:00'
draft: false
tags:
- Bash
- Mac
- homebrew
- brew
- pyenv
description: Ansibleの1.9系と2.0系を両方使いたくてpyenvを導入したらHomebrewに怒られた。
params:
  qiita_url: https://qiita.com/tsukapah/items/40462aa2311ce6269571
  qiita_id: 40462aa2311ce6269571
---

Ansibleの1.9系と2.0系を両方使いたくてpyenvを導入したらHomebrewに怒られた。

```bash
$ brew doctor
Please note that these warnings are just used to help the Homebrew maintainers
with debugging if you file an issue. If everything you use Homebrew for is
working fine: please don't worry and just ignore them. Thanks!

Warning: "config" scripts exist outside your system or Homebrew directories.
`./configure` scripts often look for *-config scripts to determine if
software packages are installed, and what additional flags to use when
compiling and linking.

Having additional scripts in your path can confuse software installed via
Homebrew if the config script overrides a system or Homebrew provided
script of the same name. We found the following "config" scripts:
  /Users/tsukapah/.pyenv/shims/python-config
  /Users/tsukapah/.pyenv/shims/python2-config
  /Users/tsukapah/.pyenv/shims/python2.7-config
```

PATHが通ってるところとかHomebrew関連のディレクトリに管理外の`*-config`が悪影響及ぼすかもしれないからどうにかしてねー、って感じでしょうか。

実際にPATHはこうなってた。

```bash
$ echo ${PATH}
/Users/tsukapah/.pyenv/shims:/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

しっかり先頭にいる。
ファイル消すわけにもいかんしPATHをどうにかする方向で検討。

brewコマンドを使うときだけpyenv用のPATHが通らなくなればいいので、brewコマンドの`alias`を以下の感じで.bash_profileの最後に設定しておけば解決。

```.bash_profile
alias brew="env PATH=${PATH/\/Users\/${USER}\/\.pyenv\/shims:/} brew"
```

あとはこの.bash_profileを読み込み直してあげればHomebrewの機嫌も直ってメデタシ。

