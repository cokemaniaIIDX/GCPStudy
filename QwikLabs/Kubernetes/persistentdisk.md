# リージョン永続ディスクにアプリをデプロイする

リージョン永続ディスクを使って、可用性の高いWordPressをデプロイする
手動でポッドを落としてみて、障害復旧性をテストする

## クラスタ作成

- 環境変数設定

```
$ CLUSTER_VERSION=$(gcloud container get-server-config --region us-west1 --format='value(validMasterVersions[0])')

$ export CLOUDSDK_CONTAINER_USE_V1_API_CLIENT=false

$ echo $CLUSTER_VERSION $CLOUDSDK_CONTAINER_USE_V1_API_CLIENT
```

- クラスタ作成

```
$ gcloud container clusters create repd \
--cluster-version=${CLUSTER_VERSION} \
--machine-type=n1-standard-4 \
--region=us-west1 \
--num-nodes=1 \
--node-locations=us-west1-a,us-west1-b,us-west1-c
```

## リージョンディスクを使ってアプリをデプロイ

- リポジトリ追加

```

```