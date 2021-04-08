# まとめの実践

## 概要
- モノリス型のPetTheoryのシステム(内容不明)を、サーバレスアーキテクチャに置き換える
- ステージング環境→本番環境でデプロイする
- 本番環境内のコンポーネント間で安全なアクセスができるようにする

## 手順

### 環境構築
```sh
gcloud config set project \
  $(gcloud projects list --format='value(PROJECT_ID)' \
  --filter='qwiklabs-gcp')
gcloud config set run/region us-central1
gcloud config set run/platform managed
git clone https://github.com/rosera/pet-theory.git && cd pet-thoery/lab07
```

### ビルド用コマンドメモ
```sh
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/xxxx
```

### デプロイ用コマンドメモ
```sh
gcloud run deploy xxxxx \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/xxxxx \
  --platform managed \
  --region us-central1
  --no-allow-unauthenticated
```

### サービスアカウント作成
```sh
gcloud iam service-accounts create billing-service-sa --display-name "Billing Service Cloud Run"
```

### staging api 0.1 index.js 中身
```js
const express = require('express');

const port = process.env.PORT || 8080;
const app = express();

app.use(express.json());

app.listen(port, () => {
  console.log(`Billing Rest API listening on port ${port}`);
});

app.get('/', async (req, res) => {
  res.json({status: 'Billing Rest API: Online'});

})
```

### frontend-staging index.js
```js
const path = require('path');
const express = require('express');
const hbs = require('hbs');

const app = express();

const pathPublicDirectory = path.join(__dirname, './public');
const pathViews = path.join(__dirname, '/views');
const pathPartials = path.join(__dirname, '/partials');


// Set hbs as the template engine
app.set('view engine', 'hbs');
app.set('views', pathViews);
hbs.registerPartials(pathPartials);

// Set the location of the html templates
app.use(express.static(pathPublicDirectory));

// Initialise the port
const port = process.env.PORT || 8080;

// show the default page
app.get('', (req, res) => {
  res.render('index');
})


// show the default page
app.get('/billing', async(req, res) => {
  try {
    const bills = dataSource.listBillings();
    res.status(200).json( {label: 'bills', records: bills} );

  } catch (error) {
    console.log(`Backend Error: ${error}`);
    res.status(500).json( {status: 'Server error'} );
  }
})



// Listen on a network port
app.listen(port, () => console.log(`Listening on:${port}`))
```

gcloud run deploy frontend-staging-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1 \
  --allow-unauthenticated


gcloud run deploy private-billing-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.2 \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated

gcloud run deploy billing-prod-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-prod-api:0.1 \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated

gcloud iam service-accounts create frontend-service-sa --display-name "Billing Service Cloud Run Invoker"

gcloud run services add-iam-policy-binding frontend-prod-service \ --member=serviceAccount:frontend-service-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --platform managed --region us-central1


gcloud run deploy frontend-prod-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-prod:0.1 \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated