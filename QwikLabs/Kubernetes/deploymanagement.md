# GKEでのデプロイ管理

- 異種混合デプロイの重要性について

アプリケーションを単一の環境にデプロイすると、
+ リソースの上限        : 特にオンプレ
+ 地理的範囲の制限      : ユーザアクセスの集中やネットワークレイテンシを起こす
+ 限られた可用性        : フォールトトレランスや復元力が必要
+ 柔軟性の低いリソース  : リソースが一部に集中してダウンを起こしたりする
+ ベンダーロックイン    : ベンダーによる個別設定などが複雑になって、アプリの移植が困難になる
などの問題が発生する

これらを解決するために、異種混合デプロイが必要

一般的には、
1. マルチクラウドにデプロイ
2. オンプレミスデータの外部接続
3. CI/CD
を実施する

## やってく

- 環境準備

今回はデプロイ手法の勉強なので、GKEの操作関係は軽めでいく

- ソースDL,クラスタ作成

```sh
gcloud config set compute/zone us-central1-a

gsutil -m cp -r gs://spls/gsp053/orchestrate-with-kubernetes .
cd orchestrate-with-kubernetes/kubernetes

gcloud container clusters create bootcamp --num-nodes 5 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

- 詳細確認

```sh
kubectl explain deployment --recursive

kubectl explain deployment.metadata.name
```

- Deployment作成

```sh
cat deployments/auth.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:1.0.0" //→ 2.0.0になっているので1.0.0に変える
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
```

- サービス起動

```
kubectl create -f deployments/auth.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods

kubectl create -f services/auth.yaml
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml

kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
```
デプロイメント.yamlからkubectl create -f でデプロイメントを作成して、
そのデプロイメントをもとに、kubectl create -f で service を起動する

- 確認

```
$ kubectl get services frontend
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
frontend   LoadBalancer   10.115.240.20   34.123.0.18   443:30517/TCP   115s

$ curl -ks https://34.123.0.18
{"message":"Hello"}
```

- Deploymentのスケール

kubectl scale deployment で簡単にスケールできる

```sh
kubectl scale deployment hello --replicas=5

kubectl get pods | grep hello- | wc -l
5

kubectl scale deployment hello --replicas=3

kubectl get pods | grep hello- | wc -l
3
```

## ローリングアップデート

ローリングアップデートでは、デプロイメントファイルを修正した後、
レプリカセットが順次新しく作成されて行って、古いものは落ちていく
徐々に新しいレプリカ数が増えていく

- 主に使うコマンド
  - kubectl `edit` deployment hello
  - kubectl `rollout` `history|pause|resume|status` deployment/hello

editして保存したらいきなり始まる
historyでリビジョンが進んでるのが確認できる
statusは実行したらログが流れて、完了したらkubectl rollout undo deployment/hellotって表示される

## カナリアデプロイ

カナリアデプロイでは、canary用のdeploymentを作成して、今動いてる元のやつと一緒に稼働させる
すると、一部のユーザだけcanaryのdeploymentに接続されるようになる
    →serviceはアプリ名`hello`を指定して実行してるので、dep/hello, dep/hello-canary 両方認識して実行する

- 手順
  - hello-canary.yamlを作成して、helloのバージョンを変える
  - canary.yamlをdeploymentとして作成する

```sh
cat <<EOF > deployments/hello-canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
        track: canary
        # Use ver 2.0.0 so it matches version on service selector
        version: 2.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
```

- 確認

```
kubectl create -f deployments/hello-canary.yaml

kubectl get deployments

curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
→1/4の確率で2.0.0になる
```

