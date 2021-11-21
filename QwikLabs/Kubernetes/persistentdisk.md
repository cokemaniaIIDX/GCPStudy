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
kind: PersistentVolumeClaim
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
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
EOF

$ kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: wp-repd-wordpress
  namespace: default
  labels:
    app: wp-repd-wordpress
    chart: wordpress-5.7.1
    heritage: Tiller
    release: wp-repd
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 200Gi
  storageClassName: repd-west1-a-b-c
EOF
```

- 確認

```
$ kubectl get storageclass
$ kubectl get persistentvolumeclaims
```

## WordPress デプロイ

```
$ helm install wp-repd \
--set smtpHost= --set smtpPort= --set smtpUser= \
--set smtpPassword= --set smtpUsername= --set smtpProtocol= \
--set persistence.storageClass=repd-west1-a-b-c \
--set persistence.existingClaim=wp-repd-wordpress \
--set persistence.accessMode=ReadOnlyMany \
stable/wordpress
```

- pod確認

```
$ kubectl get pods
```

- IPアドレス確認

```
while [[ -z $SERVICE_IP ]]; do SERVICE_IP=$(kubectl get svc wp-repd-wordpress -o jsonpath='{.status.loadBalancer.ingress[].ip}');
echo "Waiting for service external IP..."; sleep 2; done; echo
```

- ディスク作成確認

```
while [[ -z $PV ]]; do PV=$(kubectl get pvc wp-repd-wordpress -o jsonpath='{.spec.volumeName}'); echo "Waiting for PV..."; sleep 2; done
kubectl describe pv $PV
```

- パスワード取得

```
$ cat - <<EOF
Username: user
Password: $(kubectl get secret --namespace default wp-repd-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
EOF
```

## ゾーン障害をシミュレーションする

- 現状確認

```
$ kubectl get pods -l app.kubernetes.io/instance=wp-repd -o wide

NAME                                READY   STATUS    RESTARTS   AGE     IP          NODE                                  NOMINATED NODE   READINESS GATES
wp-repd-wordpress-976cf4cd5-49qxf   1/1     Running   0          6m23s   10.84.0.4   gke-repd-default-pool-c91a8b77-fzz5   <none>           <none>
```

- インスタンスグループ削除(ゾーン障害発生)

```
$ gcloud compute instance-groups managed delete ${IG} --zone ${ZONE}
```

- NODEが置き換わっていることを確認

```md
$ kubectl get pods -l app.kubernetes.io/instance=wp-repd -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP          NODE                                  NOMINATED NODE   READINESS GATES
wp-repd-wordpress-976cf4cd5-tvpn6   0/1     Running   1          93s   10.84.1.9   **gke-repd-default-pool-324d3f26-qb3v**   <none>           <none>
```