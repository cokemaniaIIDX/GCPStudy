### 環境構築
```sh
gcloud auth list
gcloud config list project
gcloud config set project \
  $(gcloud projects list --format='value(PROJECT_ID)' \
  --filter='qwiklabs-gcp')
gcloud config set run/region us-central1
gcloud config set run/platform managed
git clone https://github.com/rosera/pet-theory.git && cd pet-theory/lab07
```

OK

### タスク1
```sh
cd unit-api-billing

gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.1

gcloud run deploy public-billing-service \
--image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.1 \
--allow-unauthenticated
```

OK

### タスク2
```sh
cd ../staging-frontend-billing

gcloud builds submit \
--tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1

gcloud run deploy frontend-staging-service \
--image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1 \
--allow-unauthenticated

gcloud run services list
```

### タスク3
```sh
gcloud run services delete public-billing-service

gcloud run services list

cd ../staging-api-billing

gcloud builds submit \
--tag gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.2

gcloud run deploy private-billing-service \
--image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.2 \
--no-allow-unauthenticated

BILLING_SERVICE=private-billing-service
echo $BILLING_SERVICE

BILLING_URL=$(gcloud run services describe $BILLING_SERVICE \
  --platform managed \
  --region us-central1 \
  --format "value(status.url)")

curl -X get -H "Authorization: Bearer $(gcloud auth print-identity-token)" $BILLING_URL
```

### タスク4
```sh
cd ../
gcloud iam service-accounts create billing-service-sa --display-name "Billing Service Cloud Run"
```

OK

### タスク5
```sh
cd prod-api-billing

gcloud builds submit \
--tag gcr.io/$GOOGLE_CLOUD_PROJECT/billing-prod-api:0.1

gcloud run deploy billing-prod-service \
--image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-prod-api:0.1 \
--no-allow-unauthenticated \
--service-account=billing-service-sa@qwiklabs-gcp-00-a4bc030f6777.iam.gserviceaccount.com

PROD_BILLING_SERVICE=billing-prod-service

PROD_BILLING_URL=$(gcloud run services \
  describe $PROD_BILLING_SERVICE \
  --platform managed \
  --region us-central1 \
  --format "value(status.url)")

curl -X get -H "Authorization: Bearer \
  $(gcloud auth print-identity-token)" \
  $PROD_BILLING_URL
```

### タスク6
```sh
cd ../
gcloud iam service-accounts create frontend-service-sa --display-name "Billing Service Cloud Run Invoker"

gcloud beta run services add-iam-policy-binding private-billing-service --member=serviceAccount:frontend-service-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --platform managed --region us-central1
```

### タスク7
```
cd prod-frontend-billing

gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-prod:0.1

gcloud run deploy frontend-prod-service --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-prod:0.1 --allow-unauthenticated \
--service-account=frontend-service-sa@qwiklabs-gcp-00-a4bc030f6777.iam.gserviceaccount.com