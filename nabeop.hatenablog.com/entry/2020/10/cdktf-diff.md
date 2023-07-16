---
Title: cdktf diff の内容を詳しく調べる
Category:
- Terraform
- cdktf
Date: 2020-10-06T14:52:33+09:00
URL: https://nabeop.hatenablog.com/entry/2020/10/cdktf-diff
EditURL: https://blog.hatena.ne.jp/nabeop/nabeop.hatenablog.com/atom/entry/26006613637204016
---

最近は Terraform を触っているんだけど、AWS-CDK に慣れた身としては HCL を直接書くのは厳しいので、cdktf を試している。cdktf とは AWS-CDK のように Terraform の HCL を Typescript などで記述できるツールです。

[https://github.com/hashicorp/terraform-cdk:embed:cite]

cdktf を使うにあたり厳しさを感じていたのが `cdktf diff` で変更されるリソースの内容がわかりにくいという問題があった。cdktf の使い方とかは別エントリで書こうと思っていたんだけど、terraform と cdktf を行き来するなかで現状における解決策を見つけたのでメモしておく。

まず、`cdktf diff` したら `cdktf.out` ディレクトリが作成される。このディレクトリは `terraform init` して `terraform plan -out plan` した内容が出力されている。`cdktf.out` の中身は以下のようになっている。

```
cdktf.out
├── .terraform
│   └── plugins
│       ├── registry.terraform.io
│       │   └── hashicorp
│       │       └── google
│       │           └── 3.42.0
│       │               └── darwin_amd64
│       └── selections.json
├── cdk.tf.json
└── plan
```

`cdktf.out/cdk.tf.json` は謎だけど、 `cdktf.out/plan` が今回の本丸。

`cdktf.out` ディレクトリを working ディレクトリとして、`terraform show plan` すると `terraform plan` したときの出力が得られる。

```
$ cd cdktf.out
$ terraform show plan

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:

Terraform will perform the following actions:

Plan: 0 to add, 0 to change, 0 to destroy.
$
```

本当は AWS-CDK 並にいい感じに `cdktf diff` が出力して欲しいところだけど、正式リリース前なのでこんなもんでしょという感じではある。
