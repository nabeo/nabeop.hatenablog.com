---
Title: Terraform で Google Beta API をつかう
Category:
- Google Cloud
- Terraform
Date: 2020-09-30T12:57:01+09:00
URL: https://nabeop.hatenablog.com/entry/2020/09/30/terraform-and-google-beta-api
EditURL: https://blog.hatena.ne.jp/nabeop/nabeop.hatenablog.com/atom/entry/26006613634423914
---

2行でまとめると

* プロバイダは `google` と `google-beta` の両方を定義する
* 必要な時だけ `google-beta` プロバイダをつかうと宣言する

という感じ。

モノによっては Google Beta API をつかう必要がある。Terraform の Google Cloud 向けプロバイダでは API の種別ごとに `google` と `google-beta` というプロバイダが用意されている。`google` と `google-beta` については [Google Provider Versions - Terraform by HashiCorp](https://www.terraform.io/docs/providers/google/guides/provider_versions.html) で説明されている。

具体的な例は以下のとおり。


2つのプロバイダを定義する。
```main.tf
provider "google" {
  credentials = file("account.json")
  project     = "sample"
  region      = "asia-northeast1"
}

provider "google-beta" {
  credentials = file("account.json")
  project     = "sample"
  region      = "asia-northeast1"
}
```

`shared-vpc-host-project`というプロジェクトの定義では `google` プロバイダを使用するが、`shared-vpc-host-project` を共有 VPC として有効にするときは `google-beta` プロバイダを利用する。

```sample.tf
resource "google_project" "shared_vpc_host_project" {
  name       = "shared VPC Host Project"
  project_id = "shared-vpc-host-project"
}

resource "google_compute_shared_vpc_host_project" "shared_vpc_host_project" {
　# フォルダ単位でロールをつけているので beta API をつかう必要がある
  provider = google-beta
  project  = google_project.shared_vpc_host_project.project_id
}
```
