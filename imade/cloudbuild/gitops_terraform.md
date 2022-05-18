# Cloud Build を使って Terraform の GitOps フローを構築する

仕事でCloud Buildを使うことになったので、
CloudBuildの概要や細かい仕様(サービスアカウントの設定やIAMなど)の勉強を兼ねて、
やってみたかった`terraform`の`GitOps`を構築したいと思います！

おありがたいことに公式がチュートリアルを紹介してくれているので、
こちらを参考に作成していきます。

[Terrafrom、Cloud Build、GitOpsを使用してインフラストラクチャをコードとして管理する](https://cloud.google.com/architecture/managing-infrastructure-as-code?hl=ja)

## Cloud Build用サービスアカウント作成

```s:service_account.tf
module "service_account_for_cloudbuild" {
  source = "../modules/gcp/service_accounts"
  account_id = "cloudbuild"
  display_name = "cloudbuild-service-account"
  description = "Service Account for Cloud Build"
  roles = [
    "roles/cloudbuild.builds.builder"
  ]
}
```

```
$ terraform apply
```

## cloudbuildのビルド実行yamlを作成

今回はチュートリアルでDLしてきたリポジトリの中にある`cloudbuild.yaml`をそのままコピって来た
`solutions-terraform-cloudbuild-gitops/cloudbuild.yaml`

## Cloud Build と GitHubリポジトリの接続

GitHubのアプリ`Google Cloud Build`を使う

このアプリによって、Gitリポジトリのブランチへのpushを検知して、
terraformマニフェストを自動apply実行してくれるみたい。

1. [アプリページ](https://github.com/marketplace/google-cloud-build)画面下部の`Setup with Google Cloud Build`をクリック
2. Install Google Cloud Build のページが開く
3. terraform 関連のリポジトリのみCloud Build接続したいので、`Only select repositories`を選択
4. リポジトリ`imade-terraform`を選択
5. `Install`をクリック
6. Googleログインを促されるので、ログインする
7. Cloud Build と連携するみたいなボタンが出るので押す (スッと飛ばしちゃってどんなのか忘れた)
8. Cloud Build の画面に飛ばされる
9. リポジトリを選択画面が出るので、`imade-terraform`を選択 (imade-terraformしか出てこない)
10. 「私は、接続サービスを提供～～～」にチェックボックスを入れる
11. `接続`を押す
12. 接続されたので、`トリガーを作成する`でトリガーを作成する (サンプルトリガーを作るボタンもある こっちでとりあえず作って後で編集でもいいかも)
  1.  名前               : `push-to-branch`
  2.  リージョン         : `グローバル` (※`asia-northeast1`にしようかと思ったけど、それだと連携したGitHubリポジトリを選択できなくなる)
  3.  リポジトリ         : `imade-terraform (GitHub アプリ)`
  4.  ブランチ           : `.*(任意のブランチ)`
  5.  サービスアカウント : `cloudbuild@imade-gaming-265014.iam.gserviceaccount.com`
  6.  `作成`クリック
  1.  ↑というかこれもterraformで作りたいw

## テスト

dev ブランチを作ってプルリクエストを送ってみる

1. GitHubページで鉛筆アイコンで適当に`firewall.tf`をいじる
2. メッセージに適当に入力して、`Create a new branch for this commit and start a pull request`を選択
3. ブランチ名は`dev`にする
4. `Propose changes`をクリック
5. 次のページで`Create pull request`をクリック
6. エラー出る。

## Cloud Buildのデバッグ

### Build構成ファイルの配置忘れ

エラーメッセージ : `We could not find a valid build file. Please ensure that your repo contains a cloudbuild or a Dockerfile.`

チュートリアルのリポジトリは全部用意してくれてたけど、
自分でやる場合は`cloudbuild.yaml`が必要なのを忘れてた。

チュートリアルのリポジトリにあるcloudbuild.yamlをそのままコピって持ってくる。

### ServiceAccountの設定ミス

エラーメッセージ : 
```
'build.service_account' is specified, the build must either (a) specify 'build.logs_bucket' (b) use the CLOUD_LOGGING_ONLY logging option, or (c) use the NONE logging option
```

サービスアカウントを指定してビルドを実行する場合、ビルドログの設定を行う必要があるらしい

- エラー内容詳細
  - ログ吐き出し用のバケットの指定 (specify build.logs_bucket)
  - CLOUD_LOGGING_ONLYのオプションを指定 (use the CLOUD_LOGGING_ONLY logging option)
  - ↑2つの設定、もしくは ログを吐かないようにする (or use the NONE logging option)

この設定は`cloudbuild.yaml`で指定するっぽいかな

```yaml:cloudbuild.yaml

```