# AppEngineのチュートリアルをやってみる

[手順](https://cloud.google.com/appengine/docs/standard/python3/building-app?hl=ja)

## 1. AppEngineを有効化

- terraform

```s:app_engine.tf
#============#
# App Engine #
#============#

resource "google_app_engine_application" "imade-gcp" {
  project = var.project
  location_id = "asia-northeast1"
}
```

インスタンスのスペックとかは`app.yaml`で指定するので、
terraformはこれだけ

## 2. webサービスの作成

### 始める前に

- Datastoreでテストを行うために認証情報設定

```
$ gcloud auth application-default login
```

↑これは多分Datastoreへのアクセス権限を使いたいってことやから、
GCEのインスタンスサービスアカウントにDatastore権限を付与すればいいということやと思う
今回はスキップする

### ファイルのコピペ

- ファイル構造

building-an-app/
  app.yaml
  main.py
  requirements.txt
  static/
    script.js
    style.css
  templates/
    index.html

↑の通りファイルを作成する

