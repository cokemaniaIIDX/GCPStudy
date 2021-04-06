# email service メモ

## 概要

- Pet Theoryでは、試験室から戻ってきた医学的検査結果の処理に時間がかかっているので、改善したい
- 処理内容:
    - 検査結果が返ってくると、ペットの飼い主にスマホでメールを作成して送信する
- この検査結果を伝えるプロセスを自動化したい

## アーキテクチャ

- 外部の検査会社からの結果をHTTP(S)で受信する
- 受信確認を検査の試験室に送る
- 飼い主に検査結果をメールとSMSで送信する

## Pub/Subトピックの作成

今回のシステムを通して利用するPub/Subのトピックを作成する  
このトピックを介してそれぞれのサービスが稼働する仕組み
```sh
gcloud pubsub topics create new-lab-report
```

## 検査報告書サービスのビルド

### コード準備

#### リポジトリをDL
```sh
git clone https://github.com/rosera/pet-theory.git
```

#### パッケージインストール
```sh
npm install express
npm install body-parser
npm install @google-cloud/pubsub
```
→package.jsonの中に、dependenciesとして追加される

#### package.json編集
```json
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```

#### index.js作成
```js
const {PubSub} = require('@google-cloud/pubsub');
const pubsub = new PubSub();
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;

app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {
  try {
    const labReport = req.body;             // POSTリクエストから検査項目を抽出する
    await publishPubSubMessage(labReport);  // 検査項目を含むPub/Subメッセージをパブリッシュする
    res.status(204).send();
  }
  catch (ex) {
    console.log(ex);
    res.status(500).send(ex);
  }
})

async function publishPubSubMessage(labReport) {
  const buffer = Buffer.from(JSON.stringify(labReport));
  await pubsub.topic('new-lab-report').publish(buffer);
}
```

#### Dockerfile作成
```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

### デプロイ

スクリプトdeploy.shを作成して、実行可能にする
```sh
中身
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service
gcloud run deploy lab-report-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated

実行可能にする
chmod u+x deploy.sh

実行
./deploy.sh
```

これで、検査報告書サービスがHTTP経由で医学検査結果を取得できるようになった

### サービスのテスト

#### 環境変数設定
```sh
export LAB_REPORT_SERVICE_URL=$(gcloud run services describe lab-report-service --platform managed --region us-central1 --format="value(status.address.url)")

echo $LAB_REPORT_SERVICE_URL
```
デプロイ結果のURLを変数に格納する

#### POSTテスト用スクリプト作成
```sh
vi post-reports.sh
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 12}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 34}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 56}" \
  $LAB_REPORT_SERVICE_URL &

chmod u+x post-reports.sh
```
id:12,34,56の検査結果をPOSTで送信するてきな内容かな


#### テスト
```sh
./post-reports.sh
```
→CloudRunのlab-report-serviceのログで確認
→204コードが帰ってきていたらOK

## メールサービスの作成

### パッケージインストール
```sh
npm install express
npm install body-parser
```

### package.json追記
```json
"start": "node index.js",
```

### index.js
```js
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());

const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`Email Service: Report ${labReport.id} trying...`);
    sendEmail();
    console.log(`Email Service: Report ${labReport.id} success :-)`);
    res.status(204).send();
  }
  catch (ex) {
    console.log(`Email Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})

function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}

function sendEmail() {
  console.log('Sending email');
}
```
- Pub/Sub メッセージを解読し、sendEmail() 関数の呼び出しを試みます。
- 呼び出しに成功し、例外がスローされなければ、ステータス コード 204 を返し、メッセージが処理されたことを Pub/Sub に通知します。
- 例外が発生した場合、サービスはステータス コード 500 を返し、メッセージが処理されなかったことを Pub/Sub に通知します。Pub/Sub は、後でサービスにもう一度メッセージを配信する必要があります。

### Dockerfile作成
```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

### デプロイ

#### スクリプト
```deploy.sh
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/email-service

gcloud run deploy email-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/email-service \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated
```

```
chmod u+x deploy.sh
./deploy.sh
```

### Pub/Subにメースサービスをトリガーさせる

#### サービスアカウント作成
```sh
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```

#### サービスアカウントにメールサービスを呼び出すための権限を付与する
```sh
gcloud run services add-iam-policy-binding email-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-central1 --platform managed
```

#### プロジェクト番号格納
```
PROJECT_NUMBER=$(gcloud projects list --filter="qwiklabs-gcp" --format='value(PROJECT_NUMBER)')
```

#### Pub/Sub認証トークンを作成できるようにする
```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
```
→おそらく、Pub/Subでの通信時に利用する認証かな

#### メールサービスデプロイ時に出力されるURLを変数に格納する
```
EMAIL_SERVICE_URL=$(gcloud run services describe email-service --platform managed --region us-central1 --format="value(status.address.url)")
```

#### Pub/Subのサブスクリプションに登録する
```
gcloud pubsub subscriptions create email-service-sub --topic new-lab-report --push-endpoint=$EMAIL_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```
こうすることで、new-lab-reportにメッセージが投稿されると、email-serviceがサブスクリプションとして実行される

### テスト

#### post-reports.sh実行
```
~/pet-theory/lab05/lab-service/post-reports.sh
```
→CloudRunのログに出力される  
今回はプロトタイプなのでこれでOK  
おそらく本物は何らかの方法でメールの送信を実行するんやと思う

## SMSサービス

### いつもの
```
npm install express
npm install body-parser

"start": "node index.js",追記
```
index.js
```js
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());

const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`SMS Service: Report ${labReport.id} trying...`);
    sendSms();

    console.log(`SMS Service: Report ${labReport.id} success :-)`);
    res.status(204).send();
  }
  catch (ex) {
    console.log(`SMS Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})

function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}

function sendSms() {
  console.log('Sending SMS');
}
```

```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

```sh
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service

gcloud run deploy sms-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated
```

### SMSサービスのPub/Sub設定

#### 権限付与
```
gcloud run services add-iam-policy-binding sms-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-central1 --platform managed
```
email同様、cloudrunにpubsubのsms-serviceをトリガーするiam権限を付与 

#### 環境変数設定
```
SMS_SERVICE_URL=$(gcloud run services describe sms-service --platform managed --region us-central1 --format="value(status.address.url)")

echo $SMS_SERVICE_URL
```

#### サブスクリプション作成
```
gcloud pubsub subscriptions create sms-service-sub --topic new-lab-report --push-endpoint=$SMS_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

### テスト
```
~/pet-theory/lab05/lab-service/post-reports.sh
```
emailのときと同じようなのが表示されてたらOK


## システム復元性テスト

### スクリプトにエラーを仕込む
```js
...

function sendEmail() {
  throw 'Email server is down';
  console.log('Sending email');
}
...
```

### 再デプロイ
```
./deploy.sh
```

### テスト
```
~/pet-theory/lab05/lab-service/post-reports.sh
```
→500エラーが起こり続ける

元に戻したらエラーが起こらなくなる


## 要点
1. 複数のサービスが直接お互いを呼び出すのではなく、Pub/Sub を介して相互に非同期通信を行うことで、システムの復元性を高めることができます。

2. 検査報告書サービスは、Pub/Sub を使用することで他のサービスとは独立してトリガーされます。たとえば、顧客が別のメッセージング サービスを通して検査結果を受け取りたい場合でも、検査報告書サービスを更新することなくその機能を追加できます。

3. 再試行の処理は Cloud Pub/Sub が担当するので、サービスによる再試行は不要です。サービスは、成功または失敗を示すステータス コードを返すだけです。

4. サービスが停止した場合でも Pub/Sub が再試行を続けるので、サービスがオンラインに戻ったときにシステムを自動的に回復できます。