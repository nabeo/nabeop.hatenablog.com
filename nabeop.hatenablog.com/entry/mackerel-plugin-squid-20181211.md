---
Title: mackerel-agent-plugins に PR を送った話
Date: 2018-12-11T00:00:00+09:00
URL: https://nabeop.hatenablog.com/entry/mackerel-plugin-squid-20181211
EditURL: https://blog.hatena.ne.jp/nabeop/nabeop.hatenablog.com/atom/entry/10257846132683043889
---

[Mackerel Advent Calendar 2018 - Qiita](https://qiita.com/advent-calendar/2018/mackerel) の 11日目です。昨日は id:koudenpa:detail さんの[.NET で動くアプリケーションを Mackerel で監視できるかな？ ](https://koudenpa.hatenablog.com/entry/2018/12/10/000000)でした。

今回は mackerel-agent-plugin の mackerel-plugin-squid を使おうとしたところ、運用上欲しいメトリックが対象となっていなかったので、エイやと機能を追加して、取り込んでもらったので、その体験談です。

## メトリックとして欲しい項目を考える

今回は既存の plugin に欲しい項目を付け加えるので、コードを読んでみました。

squid には各種情報をとるためのインターフェースがいくつか用意されていて、mackerel-plugin-squid では [`mgr:info`](https://wiki.squid-cache.org/Features/CacheManager/Info?highlight=%28Feature..Squid.Cache.Manager%29) を情報源にしていました。どのような情報が取れるかはリンク先を参照してもらうとして、今回、僕が是非とも加えたいと思ったのは、以下の項目です。

- squid プロセスが使用した CPU 時間
- squid が握っている FD の情報
- squid のキャッシュストレージの使用状況
- squid のメモリ確保状況

それぞれ、`mgr:info` から取得するには以下の項目を参照すれば良さそうです。

- squid プロセスが使用した CPU 時間
    - `Resource usage for squid:` の `CPU Usage, 5 minute avg:`
- squid が握っている FD の情報
    - `File descriptor usage for squid:` あたりを丸ごと
- squid のキャッシュストレージの使用状況
    - `Cache information for squid:` の `Storage Swap size:` や `Storage Mem size:` あたりが使えそう
- squid のメモリ確保状況
    - `Memory accounted for:` の `memPoolAlloc calls:` と `memPoolFree calls:` あたりが使えそう

## 修正箇所を考える

mackerel-agent-plugin の書き方は[整備されたドキュメント](https://mackerel.io/ja/docs/entry/advanced/go-mackerel-plugin)が公開されているので、そこを参考に書き方を調べます。

今回は既存の plugin に修正するので、既存のコードを眺めます。ポイントはグラフ定義部分とメトリック値を入れているところです。

グラフ定義部分は `GraphDefinition()` という関数の中で `graphdef` を返しているだけなので、`graphdef` を眺めてなんとなく雰囲気を掴みます。今回は `Memory accounted for:` の `memPoolAlloc calls:` と `memPoolFree calls:` の値はカウンター値なので、そこだけ気をつけてあげれば良さそうです。

また、メトリック値を取得しているところは僕が最初に確認した時は `FetchMetrics()` の中で TCP ソケットを開いて `GET cache_object://localhost:3128/info HTTP/1.0\n\n` というような文字列を直接流し込んでいました。最近の squid だと `localhost:3128` など cache_manager のポートに HTTP 通信をさせると、取得できるのですが、このプラグンが最初に作られた時は squid 2 系についても考慮する必要があったので、TCP ソケットと直接通信させる必要があったようです。また、値については応答内容を愚直に `regexp.MustCompile()` で正規表現で拾い上げているようでした。

あと、コードを読みながら気づいたのですが、このプラグインにはテストが書かれていなかったので、テストが書きやすい構造に直してあげる必要がありそうです。

## パッチを書いたら PR を出す

というわけで、諸々で書いた結果以下のような PR が出来上がりました。

https://github.com/mackerelio/mackerel-agent-plugins/pull/534

今までの PR の内容をみつつ、雰囲気を感じながら拙い英語で PR を出したところ、わりとさっくりとマージしてもらえました。

mackerel-plugin-squid は [go-mackerel-plugin-helper](https://github.com/mackerelio/go-mackerel-plugin-helper) を使っています。このライブラリはレポジトリの README.md に書かれている通り、現在は [go-mackerel-plugin](https://github.com/mackerelio/go-mackerel-plugin) の使用が推奨されています。実際に最初に参照した開発向けドキュメントでも go-mackerel-plugin の使用を前提としてます。修正するときに一緒にライブラリも変更しようかと思いましたが、今回はあまり時間がとれなかったので、ライブラリの変更を見送ったのが心残りです。

mackerel-agent だけでなく、mackerel-agent-plugins も  Apache License, Version 2.0 で公開されているので、自分が欲しい機能などがあれば積極的に PR を送っていきたいですね!
