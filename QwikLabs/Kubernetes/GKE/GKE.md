# GKE

## 概要

- app という名前のアプリを例にコンテナをデプロイする
- app   :↓のDockerイメージで構成される
  - monolith    :auth と hello が含まれる
  - auth        :認証されたユーザに対してJWTトークンを生成する
  - hello       :認証されたユーザに挨拶する
  - nginx       :フロントエンド

## 手順

### コード取得

```
gsutil cp -r gs://spls/gsp021/* .
cd orchestrate-with-kubernetes/kubernetes
ls
```

- deployments/
- nginx/
- pods/
- services/
- tls/
- cleanup.sh

### デモ

```sh
kubectl create deployment nginx --image=nginx:1.10.0

kubectl get pods

kubectl expose deployment nginx --port 80 --type LoadBalancer

kubectl get services

curl http://ExternalIP:80
```

