# Challenge ラボ

## シナリオ

Cloud Run → REST API → Firestore

## 手順

### 環境設定

```sh
$ gcloud config set run/region us-central1
$ gcloud config set run/platform managed
$ git clone https://github.com/rosera/pet-theory.git
```

### Firestoreへのデータインポート

- 用意してもらってるNetflixのデータを用意してもらってるスクリプトでインポートする

```sh
$ npm install
$ node index.js netflix_titles_original.csv
```

### REST API 作成

- Cloud Run でAPIを作成する

#### ビルド&デプロイ

```sh
$ cd pet-theory/lab06/firebase-rest-api/solution-01
$ gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1
$ gcloud run deploy netflix-dataset-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1 \
  --platform managed \
  --allow-unauthenticated
```

#### テスト

```sh
$ export SERVICE_URL=$(gcloud run services describe netflix-dataset-service --format "value(status.url)")
$ curl -X GET $SERVICE_URL
```

### APIをアップデート

```sh
$ cd ~/pet-theory/lab06/firebase-rest-api/solution-02
$ gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.2
$ gcloud run deploy netflix-dataset-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.2 \
  --allow-unauthenticated
```

#### テスト

```
$ curl -X GET $SERVICE_URL/2019
```

### フロントエンドのデプロイ

```sh
$ cd ~/pet-theory/lab06/firebase-frontend
$ gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1
$ gcloud run deploy frontend-staging-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1 \
  --allow-unauthenticated
```

### 本番フロントエンドのデプロイ

```sh
$ cd ~/pet-theory/lab06/firebase-frontend/public

$ vi app.js
const REST_API_SERVICE = "https://netflix-dataset-service-4bmbzx3lga-uc.a.run.app/2020"
↑を追記して元のコンストラクタをコメントアウトする

$ cd ..
$ gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-production:0.1
$ gcloud run deploy frontend-production-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-production:0.1 \
  --allow-unauthenticated
```