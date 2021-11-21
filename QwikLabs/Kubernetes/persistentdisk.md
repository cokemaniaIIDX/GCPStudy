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
$ helm repo add stable https://charts.helm.sh/stable

$ helm repo update
```

- Storage Class 作成

```
$ kubectl apply -f - <<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: repd-west1-a-b-c
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
  zones: us-west1-a, us-west1-b, us-west1-c
EOF
```

- 永続ボリュームの要求を作成

```
$ kubectl apply -f - <<EOF
kind: PersistentBolumeClaim
apiVersion: v1
metadata:
  name: data-wp-repd-mariadb-0
  namespace: default
  labels:
    app: mariadb
    component: master
    release: wp-repd
spec:
  accessModes:
    - ReadOnlyMany
  resoueces:
    requests:
      storage: 8Gi
  storageClassName: standard
EOF
```

- 確認

```
$ kubectl get storageclass
$ kubectl get persistentvolumeclaims
```

## WordPress デプロイ

