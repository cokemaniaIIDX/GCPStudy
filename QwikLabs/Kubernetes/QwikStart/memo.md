# GKE の Qwik Start

## 概要

- GKEのクラスタ作成
- 認証情報取得
- アプリのデプロイ
- クラスタの削除

### 認証情報の取得

```sh
$ gcloud container clusters get-credentials imade-cluster
Fetching cluster endpoint and auth data.
kubeconfig entry generated for imade-cluster.
```

### アプリのデプロイ

#### デプロイメントを作成する

```sh
$ kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
deployment.apps/hello-server created
```

- デプロイメントは kubectl で作成する
- コンテナイメージは、google-samples のhello-app:1.0

#### サービスを expose する

```sh
$ kubectl expose deployment hello-server --type=LoadBalancer --port 8080
service/hello-server exposed
```

- デプロイメントを expose するとWEBに公開される
- type は LoadBalancer が基本かな

### 確認

```sh
kubectl get service
```
→Public IPをコピー

ブラウザで
```
http://[Public IP]:8080
```

hello-appが表示されたらOK


### 削除

```
$ gcloud container clusters delete imade-cluster
The following clusters will be deleted.
 - [imade-cluster] in [us-central1-a]

Do you want to continue (Y/n)?  y

Deleting cluster imade-cluster...done.
Deleted [https://container.googleapis.com/v1/projects/qwiklabs-gcp-00-122919c4dc5e/zones/us-central1-a/clusters/imade-cluster].
```

### まとめ

- gcloud コマンドで GKEクラスタを起動するのを確認できた
- get-credentialsはやったことなかったのでなぜ必要なのか後で確認してみたい
- 資格勉強後やと内容がすんなり入ってきてよかった