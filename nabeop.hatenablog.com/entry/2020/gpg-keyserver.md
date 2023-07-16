---
Title: gnupg の鍵サーバで hkps://hkps.pool.sks-keyservers.net を使いたい
Category:
- gpg
Date: 2020-08-21T00:00:00+09:00
URL: https://nabeop.hatenablog.com/entry/2020/gpg-keyserver
EditURL: https://blog.hatena.ne.jp/nabeop/nabeop.hatenablog.com/atom/entry/26006613617139692
---

**[注意] !!!sks-keyservers.net は終了しているのでこの設定は意味がありません!!! [追記]**


<!-- more -->



homebrew で gnupg をインストールした直後だと以下のように失敗してしまう。

```
% gpg --keyserver hkps://hkps.pool.sks-keyservers.net --send-keys C08813444158EAD96E4D8226F6B09CB92E3DF175
gpg: sending key F6B09CB92E3DF175 to hkps://hkps.pool.sks-keyservers.net
gpg: keyserver send failed: General error
gpg: keyserver send failed: General error
[1]    53395 exit 2     gpg --send-keys C08813444158EAD96E4D8226F6B09CB92E3DF175
%
```

TLS 無しで試してみると成功する。

```
% gpg --keyserver hkp://hkps.pool.sks-keyservers.net --send-keys C08813444158EAD96E4D8226F6B09CB92E3DF175
gpg: sending key F6B09CB92E3DF175 to hkp://hkps.pool.sks-keyservers.net
%
```

サーバ証明書の検証で失敗しているぽい。こういうときはサーバ証明書を教えてあげる必要がある。

まずは、`dirmngr` に `hkps.pool.sks-keyservers.net` で使っている証明書を明示させる。

homebrew で gnupg をインストールしたら `$( brew --prefix )/share/gnupg/sks-keyservers.netCA.pem` にサーバ証明書が存在するはず。このファイルを `dirmngr` が鍵サーバの証明書として扱えるようにする。

具体的には `~/.gnupg/dirmngr.conf` に以下を記述して、 `gpgconf --reload dirmngr` で設定の再読み込みをさせるだけで良い。

```
hkp-cacert /usr/local/share/gnupg/sks-keyservers.netCA.pem
```

あとは `gpg --keyserver hkps://hkps.pool.sks-keyservers.net --send-keys C08813444158EAD96E4D8226F6B09CB92E3DF175` が成功することを確認したら、gnupg の設定ファイル(`~/.gnupg/gpg.conf`) で以下のように鍵サーバとして `hkps://hkps.pool.sks-keyservers.net` を指定する。

```
keyserver hkps://hkps.pool.sks-keyservers.net
```
