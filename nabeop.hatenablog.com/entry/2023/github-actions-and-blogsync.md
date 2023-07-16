---
Title: Github Actions と blogsync を使ってはてなブログの記事を管理する
Category:
- blogsync
- Github Actions
Date: 2023-07-16T23:50:00+09:00
URL: https://nabeop.hatenablog.com/entry/2023/github-actions-and-blogsync
EditURL: https://blog.hatena.ne.jp/nabeop/nabeop.hatenablog.com/atom/entry/820878482950392555
Draft: true
CustomPath: 2023/github-actions-and-blogsync
---

前々からこのブログを Github で管理したいなーと思っていたけど、なかなか手をつける気になっていなかった。なぜかこの連休中に機運が高まったのでえいやで Github Actions と [blogsync](https://github.com/x-motemen/blogsync) を使った仕組みを導入した。

大筋では

- 新しいエントリを追加する P-R を作る
- 作業ブランチでの作業中は下書き状態にしておく
- P-R をメインブランチにマージするときに公開する

といった運用を想定している。

特に P-R ベースでの作業フローはブログの執筆者のほかに公開前に投稿内容をチェックするような体制を想定している。

## 仕組み

Github Actions のワークフローは以下の3つを実装している。

- 下書き状態で新しい記事をはてなブログに投稿するワークフロー : `blogsync-post.yaml`
    - `blogsync post --draft` を実行している
    - P-R の最初に動くことを想定している
    - `blogsync post` を実行した後は `blogsync/created` ラベルを P-R につける
- はてなブログに投稿している記事を更新するワークフロー : `blogsync-push.yaml`
    - `blogsync push` を実行している
    - P-R 中に記事のコミットがつまれたときに
- 下書き状態の記事を公開状態にするワークフロー : `blogsync-publish.yaml`
    - 記事のメタデータの `Draft` の値を変更して `blogsync push` を実行している
    - P-R がマージされたときに実行される

blogsync がはてなブログの Atom Pub API を使うときの認証情報は設定ファイルか環境変数で扱うことができる。リポジトリに秘密情報は含めたくないので環境変数を使うことになるんだけど、複数の執筆者が想定される環境では、Github の世界とはてなの世界でユーザのマッピングが必要になる。今回は json ファイルで Github の login 名とはてな ID (とその API Key を保存しているシークレットの名前) を定義して、ワークフロー中で取得している。ワークフロー中での取得している様子は以下のとおり。

```yaml
- name: hatenaid
  id: hatena_id
  env:
    PR_AUTHOR: ${{ github.event.pull_request.user.login }}
  run: |
    echo hatena_id=$(jq --arg github_login ${PR_AUTHOR} '.[$github_login].hatena_id' map.json) | tr -d '"' >> $GITHUB_OUTPUT
    echo password_secret_name=$(jq --arg github_login ${PR_AUTHOR} '.[$github_login].password_secret_name' map.json) | tr -d '"' >> $GITHUB_OUTPUT
```

ひとまず、今の状態でしばらく使ってみていまいちなところとかはチマチマ直していくかという気分になっている。今のところ `blogsync pull` に相当するワークフローを作っていないのではてなブログの管理画面などで変更した内容を取り込むときには個別で頑張る必要があるはず。

## 追記

この記事を公開するための準備ではじめて使ったら P-R にラベルを付与するところで、

> GraphQL: Resource not accessible by integration (addLabelsToLabelable)

というエラーが出て失敗してしまった。これは Github token のデフォルトの権限が参照しか付与されていないからだった。リポジトリの Settings から Workflow permissions が「Read repository contents and packages permissions」になっているので「Read and write permissions」に変更するか、ワークフローの定義で個別に指定すればよいはず。面倒なのでデフォルトの権限を「Read and write permissions」にしちゃったけど、本来は `contents: write` と `pull-requests: write` か `issues: write` の権限があれば十分なはず。
