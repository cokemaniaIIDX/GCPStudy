# Kubernetesを利用した負荷分散のテスト

Kubernetesを使ってAppEngineにデプロイした簡単なアプリへ負荷テストを実行する
負荷かけに使うのはLocustというミドルウェア
一般的な方法らしいので参考になりそう

- ワークロード詳細

ClinentがApplicationへユーザ登録(ログイン)、
次に指標やセンサー測定値の報告を開始して、
定期的にサービスに再登録をする ←（ちょっとよくわからん）
みたいな感じ

- Locust

Pythonベースの負荷テストツール
パス`/`でリクエストを分散していく

## 演習内容

1. テスト対象のシンプルウェブアプリをAppEngineに作成
2. Kubernetes Engine を使って 負荷テストフレームワークをデプロイ
3. RESTベースのAPI負荷テストを実施

## 手順

### 各種環境設定

```
PROJECT=$(gcloud config get-value project)
REGION=us-central1
ZONE=${REGION}-a
CLUSTER=gke-load-test
TARGET=${PROJECT}.appspot.com
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```

### サンプルコードを持ってきてDockerイメージ作成

```
$ gsutil -m cp -r gs://spls/gsp182/distributed-load-testing-using-kubernetes .

$ cd distributed-load-testing-using-kubernetes/

$ gcloud builds submit --tag gcr.io/$PROJECT/locust-tasks:latest docker-image/.
```

### ウェブアプリのデプロイ

```
$ gcloud app deploy sample-app/app.yaml
```

### Kubernetesクラスタをデプロイ

```
$ gcloud container clusters create $CLUSTER \
--zone $ZONE --num-nodes=5
```

### 負荷テストのマスター作成

```
sed -i -e "s/\[TARGET_HOST\]/$TARGET/g" kubernetes-config/locust-master-controller.yaml
sed -i -e "s/\[TARGET_HOST\]/$TARGET/g" kubernetes-config/locust-worker-controller.yaml
sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-master-controller.yaml
sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-worker-controller.yaml
```

```
$ kubectl apply -f kubernetes-config/locust-master-controller.yaml

$ kubectl get pods -l app=locust-master
```

```
$ kubectl apply -f kubernetes-config/locust-master-service.yaml

$ kubectl get svc locust-master
```

これでマスターノードができた

### 負荷テストのワーカー作成

```
$ kubectl apply -f kubernetes-config/locust-worker-controller.yaml

$ kubectl get pods -l app=locust-worker

$ kubectl scale deployment/locust-worker --replicas=20

$ kubectl get pods -l app=locust-worker
```


### 負荷試験やってみる

外部IP:8089 を入力してアクセス

ユーザ数と登録速度を適当に決めて実行してみる
→なんか動いてるけど意味わかんね～～～～ｗｗｗ

## 感想

とりあえず、Locustっていう負荷かけソフトがあるのと、
それがマスターノードとワーカーノードで別れてること、
んで、LocustをKubernetesでデプロイすることに成功した。

なんかいも繰り返してkubectlをしてるおかげで結構理解してきたかも