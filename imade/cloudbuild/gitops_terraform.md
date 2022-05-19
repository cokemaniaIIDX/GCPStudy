# Cloud Build を使って Terraform の GitOps フローを構築する

仕事でCloud Buildを使うことになったので、
CloudBuildの概要や細かい仕様(サービスアカウントの設定やIAMなど)の勉強を兼ねて、
やってみたかった`terraform`の`GitOps`を構築してみる。

おありがたいことに公式がチュートリアルを紹介してくれているので、
こちらを参考に作成していく。

[Terrafrom、Cloud Build、GitOpsを使用してインフラストラクチャをコードとして管理する](https://cloud.google.com/architecture/managing-infrastructure-as-code?hl=ja)

## Cloud Build用サービスアカウント作成

まず、Cloud Build実行用のサービスアカウントを作成する

```s:service_account.tf
module "service_account_for_cloudbuild" {
  source = "../modules/gcp/service_accounts"
  account_id = "cloudbuild"
  display_name = "cloudbuild-service-account"
  description = "Service Account for Cloud Build"
  roles = [
    "roles/editor",
    "roles/viewer",
    "roles/cloudbuild.builds.builder"
  ]
}
```

```
$ terraform apply
```

- roleについて
  - editor : terraformがこのサービスアカウントを利用して各種リソースの作成・削除・変更を行うようになるので、プロジェクトリソースに対する編集者権限を付与する。
  - viewer : terraform plan などを実行する際、クラウド上にあるリソースにアクセスして情報を取得する必要があるが、editorには閲覧権限がないので、viewerも付与する。
  - ↑面倒ならownerを付与すればいいが、過剰になる。
  - cloudbuild.builds.builder : cloudbuildに関する一連の権限を付与してくれる、事前定義のロール。

## cloudbuildのビルド実行yamlを作成

今回はチュートリアルでDLしてきたリポジトリの中にある`cloudbuild.yaml`をまねしつつ作成
terraform用リポジトリの一番頭に配置する

ちなみに自分のは↓こんな感じ

```
$ tree -L 1
.
├── README.md
├── aws
├── cloudbuild.yaml
├── gcp
└── modules

3 directories, 2 files
```

```yaml:cloudbuild.yaml
# Cloud Build - Build 構成ファイル

steps:
- id: 'branch name'
  name: 'alpine'
  entrypoint: 'sh'
  args:
  - '-c'
  - |
      echo "***********************"
      echo "$BRANCH_NAME"
      echo "***********************"

# init 実行
- id: 'tf init'
  name: 'hashicorp/terraform:1.1.9'
  entrypoint: 'sh'
  args:
  - '-c'
  - |
      if [ -d "gcp/" ]; then
        cd gcp
        terraform init
      else
        echo "Error : No such file or directory (gcp)"
        exit 1
        cd ../
      fi

# plan 実行
- id: 'tf plan'
  name: 'hashicorp/terraform:1.1.9'
  entrypoint: 'sh'
  args:
  - '-c'
  - |
      if [ -d "gcp/" ]; then
        cd gcp
        terraform plan
      else
        echo "Error : No such file or directory (gcp)"
        exit 1
        cd ../
      fi
# plan 終了

# apply 実行
- id: 'tf apply'
  name: 'hashicorp/terraform:1.1.9'
  entrypoint: 'sh'
  args:
  - '-c'
  - |
      if [ -d "gcp/" ]; then
        cd gcp
        terraform apply -auto-approve
      else
        echo "***************************** SKIPPING APPLYING *******************************"
        echo "Branch '$BRANCH_NAME' does not represent an oficial environment."
        echo "*******************************************************************************"
      fi
# apply 終了

# ビルド時のオプションはoption: で追加
options:
  logging: CLOUD_LOGGING_ONLY
```

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

## テスト

dev ブランチを作ってプルリクエストを送ってみる

1. GitHubページで鉛筆アイコンで適当に`firewall.tf`をいじる
2. メッセージに適当に入力して、`Create a new branch for this commit and start a pull request`を選択
3. ブランチ名は`dev`にする
4. `Propose changes`をクリック
5. 次のページで`Create pull request`をクリック
6. うまくいくと↓の画像のようにビルド完了のメッセージが出る

![ok](./apply_OK.png)

## 確認

GCP上で実際のファイアウォールルールを見る
→修正が適用されている

# 感想

git push をトリガーに、 terraform init , plan , apply が実行できるようにパイプラインを組むことができた。
ただ、terraformで構築する際、予期せぬDestroyが発生するのがとても怖い
コード修正した後、terraform plan は必須だけど、
plan を手動でやるなら、そのままapply も手動で良くね？と思ってしまう

良い感じなら会社の業務もCICD化しようと思ったけど、
今の使い方でも十分かな。。

# エラー発生時のデバッグ

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

[ここ](https://qiita.com/souhei-etou/items/aa77e178ebe37e916119)で設定方法大まかに説明してくれている

公式は[こっち](https://cloud.google.com/build/docs/configuring-builds/create-basic-configuration)

```yaml:cloudbuild.yaml
# ~~略~~
# apply 終了

# ビルド時のオプションはoption: で追加
options:
  logging: CLOUD_LOGGING_ONLY
```

デフォルトでは**GCSバケットとLoggingの両方にログが出力される**が、
ログ出力用バケットを用意していないので、Loggingのみに出力するように指定しないとエラーでビルドできない

`CLOUD_LOGGING_ONLY`で、Loggingのみにログが出るようになるので、Buildのページでログを確認できる
`GCS_ONLY`でGCSのみにログ出力もできる
この場合、build.logs_bucketオプションを追加して保存先のバケットも指定する必要あり。
  →options:
      build.logs_bucket: bucketname みたいな感じで指定かな？試してないのでわからん
`NONE`でログ出力なしにできるが、デバッグができなくなるので、CLOUD_LOGGING_ONLY推奨。


### ServiceAccountの設定ミス

無事ビルド開始はできたけど、ビルド中にエラーが発生している。

ログ見ると、
terraform plan を実行しようとしたときにserviceaccountにcompute.address.get権限がありませんとかその辺のエラー。
→サービスアカウントに対して閲覧権限、編集権限を付与する

閲覧権限と編集権限のみだと、バケットアクセスとかができないようなので、
`editor`,`viewer`,`cloudbuild.builds.builder` の3つを付与してみた。
→ビルド成功した！