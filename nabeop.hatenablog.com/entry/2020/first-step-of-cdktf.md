---
Title: cdktf に入門した話
Category:
- Terraform
- Google Cloud
- cdktf
Date: 2020-12-02T09:35:00+09:00
URL: https://nabeop.hatenablog.com/entry/2020/first-step-of-cdktf
EditURL: https://blog.hatena.ne.jp/nabeop/nabeop.hatenablog.com/atom/entry/26006613657721920
---

本記事ははてなエンジニア Advent Calendar 2020 の2日目の記事です。昨日は id:miki_bene:detail さんの[ページのコンテンツから離れたタイミングのブラウザイベントの選び方 - ドキュメントを見たほうが早い](https://benevolent0505.hatenablog.com/entry/2020/12/01/093000)でした

[https://qiita.com/advent-calendar/2020/hatena:embed:cite]

現職になってからクラウドプロバイダーは AWS が中心だったので、TypeScript で AWS CDK をつかっていました。最近になって Google Cloud にも興味が出てきたので、Terraform も選択肢として入ってくるようになりました。ただ、TypeScript でインフラを構成することになれていると HCL では表現力が足りないなーという課題感も感じていました。

そんなときに CDK の Terraform 版である cdktf の存在を知ったので、Terraform に入門するついでにお試しで cdktf にも入門してみました。

cdktf はこのエントリの執筆時点で v0.0.18 でまだまだ開発版という位置付けで正式版になったらいろいろと変わると思います。以下は v0.0.18 での知見です。

## cdktf の始め方

HashiCorp のドキュメントに始め方があります。

[https://learn.hashicorp.com/tutorials/terraform/cdktf-install:embed:cite]

CDK と同じように TypeScript 向けのプロジェクトは以下のように初期化してテンプレートが生成されます。

```
cdktf init --template=typescript --local
```

`cdktf init` した直後のディレクトリは CDK と同じような感じです。

```
$ ls -a
./                 .gitignore         main.d.ts          node_modules/      tsconfig.json
../                cdktf.json         main.js            package-lock.json
.gen/              help               main.ts            package.json
$
```

唯一、 `.gen` というディレクトリだけが見慣れない感じだったので中身を確認したところ、初期状態では `cdktf.json` で `terraformProviders` に `aws` が指定されている関係で Terraform の AWS プロバイダ向けと思われる ts が `.gen/providers/aws` 以下に配置されていました。

あと、`help` というファイルに cdktf での開発に必要なコマンドが書かれていて便利でした。

## Google Cloud 向けに環境を整える

初期状態では AWS を想定されていますが、今回は Google Cloud を対象としたいので Terraform プロバイダを変更する必要があります。Terraform プロバイダ設定は `cdktf.json` で以下のように書き換えます。

```cdktf.json
{
  "language": "typescript",
  "app": "npm run --silent compile && node main.js",
  "terraformProviders": [
    "google@~> 3.0"
  ]
}
```

このままでは `.gen` 以下には AWS 向けのプロバイダが入っているので `help` にかいてあるとおり Google Cloud のプロバイダを導入します。

```
npm i -a @cdktf/provider-google
npm run get
```

## cdktf のある生活

Google Cloud 向けのクレデンシャル情報を用意したりと Terraform / Google Cloud としてのお作法はありますが、だいたい AWS CDK と同じように開発ができます。

つまり、

* ts ファイルで記述して
* `npm run compile` して、ts ファイルから js ファイルと d.ts ファイルを生成して、
* `npm run diff` で差分を確認して、
* `npm run deploy` して適用する

という感じです。

`npm run diff` が `terraform plan`、`npm run deploy` が `terraform apply` に相当しています。

また、`package.json` の内容などを特に変更しなければ `cdktf.out` が Terraform としての working directory という扱いになっていそうです。

```
$ ls -a cdktf.out
./           ../          .terraform/  cdk.tf.json  plan
$
```

たとえば、`cdktf.out/plan` は `terraform plan -out plan` したときに生成されるファイルと同じ感じでした。つまり、`cdktf.out` ディレクトリに移動したら `terrafom show plan` で plan 時の情報を確認できたりします。`cdktf.out/plan` を使って差分を確認する様子は別エントリで書いています。

[https://nabeop.hatenablog.com/entry/2020/10/cdktf-diff:embed:cite]

## Terraform プロバイダーとの対応関係

`.gen/providers/google` の中にプロバイダーのリソースやデータソースに対応するクラスが定義されています。

また、各クラスの実装はプロバイダーのスキーマから生成されており、以下のような対応関係になっているみたいです。

* リソース
    * リソース名についているプロバイダーの文字列を削除して、アッパーキャメルケースに変換
* データソース
    * データソース名に `Data` プレフィックスをつけて、アッパーキャメルケースに変換
    
つまり、

* [`google_project` リソース](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project)を使いたい場合は [`Project` クラス](https://github.com/terraform-cdk-providers/cdktf-provider-google/blob/v0.0.57/src/project.ts)を使う
* [`google_project` データソース](https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/project)を使いたい場合は [`DataGoogleProject` クラス](https://github.com/terraform-cdk-providers/cdktf-provider-google/blob/v0.0.57/src/data-google-project.ts)を使う

というような感じです。

## snapshot テストしたい

せっかく TypeScript で IaC をしているので、テストも書いておきたいところです。AWS CDK を使っている時は snapshot テストだけは整えるようにして、バージョンアップによる不意の変更がわかるようにしています。cdktf でも同じようにしたいので調べてみたところ、[`cdktf` のなかに `Testing` クラスが定義されていました](https://github.com/hashicorp/terraform-cdk/blob/4c2bd1af58823b264db3d0112fb25824f5b99ed9/packages/cdktf/lib/testing.ts)。

試しに jest 周りを整えて、`main.test.ts` を書いてみたら snapshot テストができました。

```main.test.ts
import { SandboxStack } from './main';
import { Testing } from 'cdktf';

describe ("sandbox-project", () => {
  test("snaphost test", () => {
    const app = Testing.app();
    const stack = new SandboxStack(app, 'test-stack', {
      credentials: '{}',
    });
    const results = Testing.synth(stack);
    expect(results).toMatchSnapshot();
  });
});
```

## cdktf を触ってみた感想

まだまだ開発版なので足りていない機能はあるという前提ですが、以下のような感想です。

* TypeScript の資産が活かせるので開発はすんなり始められそう
    * Terraform プロバイダーから半ば機械的に生成された ts ファイルをベースにしているので、HCL を書いている感はある
* AWS CDK に慣れた身では `cdktf diff` の出力内容だけだと、どのような変更が行われるかわからず不安になる
    * 変更があるリソースはわかるけど、変更内容まではわからないというところが厳しかった

[cdktf のレポジトリ](https://github.com/hashicorp/terraform-cdk/)をみると活発に開発されていて、プロジェクトの[ロードマップ](https://github.com/hashicorp/terraform-cdk/projects/1)もあるので正式版のリリースを気長に待とうかという気持ちです。

明日は id:dekokun:detail さんです。
