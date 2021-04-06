# pdf-converter のメモ

## アーキテクチャ

- サーバレスアーキテクチャ Cloud Run を使う
- Storage BucketにアップロードしたファイルをPDFに変換し、違うバケットに保存する

## 内容
- Node JSを使う
- Cloud Build でコンテナをビルドする
- Cloud Run でPDFに変換するサービスを作成する
- Storageの イベント処理を使う

## 手順

### Pet Theory のソースをDL
```
$ git clone https://github.com/rosera/pet-theory.git
```

### パッケージインストール
```
npm install express
npm install body-parser
npm install child_process
npm install @google-cloud/storage
```

### ビルドする
```
gcloud builds submit \
    --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-cnverter
```

### デプロイする
```
gcloud beta run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated

Service [pdf-converter] revision [pdf-converter-00001] has been deployed and is serving 100 percent of traffic at https://pdf-converter-[hash].a.run.app
```

### 出力されるURLに対して環境変数を設定する
```
SERVICE_URL=$(gcloud beta run services describe pdf-converter --platform managed --region us-central1 --format="value(status.url)")

$ echo $SERVICE_URL
https://pdf-converter-xmfdp4nhiq-uc.a.run.app
```

### テストのPOSTリクエストを送ってみる
```sh
curl -X POST $SERVICE_URL

<html><head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<title>403 Forbidden</title>
</head>
<body text=#000000 bgcolor=#ffffff>
<h1>Error: Forbidden</h1>
<h2>Your client does not have permission to get URL <code>/</code> from this server.</h2>
<h2></h2>
</body></html>
→申し訳ないが匿名ユーザはNG
```

```sh
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL


OK
```

### Storage バケット作成
```sh
$ gsutil mb gs://$GOOGLE_CLOUD_PROJECT-upload
$ gsutil mb gs://$GOOGLE_CLOUD_PROJECT-processed
```
アップロード用のバケットと、変換したPDFを入れる用のバケットを作る

### Pub/Subへの通知を作成
```sh
$ gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload

Created Cloud Pub/Sub topic projects/qwiklabs-gcp-01-90f7d300ded1/topics/new-doc
Created notification config projects/_/buckets/qwiklabs-gcp-01-90f7d300ded1-upload/notificationConfigs/1
```
new-docというラベルの付いた通知が作成される

### Pub/SubがCloudRunをトリガーするためのサービスアカウント作成
```sh
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```

### サービスアカウントにPDFコンバータを呼び出す権限を付与する
pdf-converterという名前のiamポリシーバインディングを、さっき作ったサービスアカウント"pubsub-cloud-run-invoker"に追加する
中身(role)は、Runの実行権限(run.invoker)
```sh
gcloud beta run services add-iam-policy-binding pdf-converter --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --platform managed --region us-central1
```

### Projectナンバーを変数に設定
```
gcloud projects list
805582018042

PROJECT_NUMBER=805582018042
```

### Pub/Sub認証トークンを利用できるようにする
```sh
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
```
なんか必要らしい


### new-docでメッセージが公開されたときにPDFコンバータを実行できるようにSubscriptionを作成する
```sh
gcloud beta pubsub subscriptions create pdf-conv-sub --topic new-doc --push-endpoint=$SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

### Storageにアップされた時にCloudRunがトリガーされるか確認
```sh
gsutil -m cp gs://spls/gsp644/* gs://$GOOGLE_CLOUD_PROJECT-upload
```

### Logging確認

OKの後に↓みたいなの(アップロードされたファイルの詳細)が表示されている
```json
{
  "textPayload": "file: {\"kind\":\"storage#object\",\"id\":\"qwiklabs-gcp-01-90f7d300ded1-upload/file_example_XLS_50.xls/1617690268826640\",\"selfLink\":\"https://www.googleapis.com/storage/v1/b/qwiklabs-gcp-01-90f7d300ded1-upload/o/file_example_XLS_50.xls\",\"name\":\"file_example_XLS_50.xls\",\"bucket\":\"qwiklabs-gcp-01-90f7d300ded1-upload\",\"generation\":\"1617690268826640\",\"metageneration\":\"1\",\"contentType\":\"application/vnd.ms-excel\",\"timeCreated\":\"2021-04-06T06:24:28.888Z\",\"updated\":\"2021-04-06T06:24:28.888Z\",\"storageClass\":\"STANDARD\",\"timeStorageClassUpdated\":\"2021-04-06T06:24:28.888Z\",\"size\":\"13824\",\"md5Hash\":\"1zLu832UFyLIsjIhvxcxYQ==\",\"mediaLink\":\"https://www.googleapis.com/download/storage/v1/b/qwiklabs-gcp-01-90f7d300ded1-upload/o/file_example_XLS_50.xls?generation=1617690268826640&alt=media\",\"contentLanguage\":\"en\",\"crc32c\":\"MlcwJQ==\",\"etag\":\"CJD40e796O8CEAE=\"}",
  "insertId": "606bfe9e00021219ae89158e",
  "resource": {
    "type": "cloud_run_revision",
    "labels": {
      "configuration_name": "pdf-converter",
      "service_name": "pdf-converter",
      "location": "us-central1",
      "project_id": "qwiklabs-gcp-01-90f7d300ded1",
      "revision_name": "pdf-converter-00001-han"
    }
  },
  "timestamp": "2021-04-06T06:24:30.135705Z",
  "labels": {
    "instanceId": "00bf4bf02d23ef4872d3d348e6514bfea45b7adf8a8684bd0e995ef589ba4917d8c739929f3b6b1539d06aa08ae73eabc5ab2a5d3e8b1ecd3cef6b50f8"
  },
  "logName": "projects/qwiklabs-gcp-01-90f7d300ded1/logs/run.googleapis.com%2Fstdout",
  "receiveTimestamp": "2021-04-06T06:24:30.474239713Z"
}
```

---
ここまでで、
- Storageにファイルをアップしたら、new-docというラベルがついた通意をPub/Subに送信
- new-docのラベルがついた通知が来たら、CloudRunのコンテナが実行して払いだされるURLにPOSTリクエストを実行するサブスクリプションが作成される
- まだ、変換することはできない

### Dockerコンテナについて
- Cloud Runはコンテナを使うため、その形式でアプリを用意する必要がある
  - アプリケーションのDockerfileマニフェストを作成する必要がある
- LibreOfficeを使用するので、それをインストールするコマンドをコンテナに含める
  - apt-get update -y && apt-get install -y libreoffice && apt-get clean
- 今の段階では、LibreOfficeはコンテナに含まれていないので追加する必要がある

### Dockerfileに追記
下記行を追記する(FROM node:12の下)
```Dockerfile
RUN apt-get update -y \
    && apt-get install -y libreoffice \
    && apt-get clean
```

### index.jsに追記
```js
const {promisify} = require('util');
const {Storage}   = require('@google-cloud/storage');
const exec        = promisify(require('child_process').exec);
const storage     = new Storage();
```

### app.post 置き換え
```js
app.post('/', async (req, res) => {
  try {
    const file = decodeBase64Json(req.body.message.data);
    await downloadFile(file.bucket, file.name);
    const pdfFileName = await convertFile(file.name);
    await uploadFile(process.env.PDF_BUCKET, pdfFileName);
    await deleteFile(file.bucket, file.name);
  }
  catch (ex) {
    console.log(`Error: ${ex}`);
  }
  res.set('Content-Type', 'text/plain');
  res.send('\n\nOK\n\n');
})
```

#### 詳細
- Pub/Sub 通知からファイルの詳細を抽出します。
- Cloud Storage からローカルのハードドライブにファイルをダウンロードします。これは実際には物理ディスクではなく、ディスクのように動作する仮想メモリのセクションです。
- ダウンロードしたファイルを PDF に変換します。
- PDF ファイルを Cloud Storage にアップロードします。環境変数 process.env.PDF_BUCKET には、PDF を書き込む - Cloud Storage バケットの名前が含まれています。以下のサービスをデプロイするときに、この変数に値を割り当てます。
- 元のファイルを Cloud Storage から削除します。

### コンテナをビルド
```sh
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```

### デプロイ
```sh
gcloud beta run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-central1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed
```

### 確認
```sh
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

