# Kubernetes と VisionAPI を利用した 画像のラベル付け

Redditの/r/aww ってとこにある？画像を表示して、ラベルもつける
アプリの中身の内容がほとんどないので、実質クラスタの作り方をみるだけで
Visionの内容は皆無

## 手順

- クラスタ起動
- リポジトリDL
- make&デプロイ
- 確認

## Makeでやってることを確認

```
awwvision ディレクトリで make all を実行して、すべての内容をビルドしてデプロイします。

make all
このプロセスの一環として、Docker イメージがビルドされ、Google Container Registry のプライベート コンテナ レジストリにアップロードされます。さらに、テンプレートから yaml ファイルが生成され、プロジェクト固有の情報が入力されて、それらを使ってこのラボの Kubernetes リソース（redis、webapp、worker）がデプロイされます。
```

- Makefileを見てみる

```
.PHONY: all
all: redis webapp worker

.PHONY: delete
delete: delete-redis delete-webapp delete-worker

.PHONY: redis
redis:
        $(MAKE) -C redis all

.PHONY: webapp
webapp:
        $(MAKE) -C webapp all

.PHONY: worker
worker:
        $(MAKE) -C worker all

.PHONY: delete-redis
delete-redis:
        $(MAKE) -C redis delete

.PHONY: delete-webapp
delete-webapp:
        $(MAKE) -C webapp delete

.PHONY: delete-worker
delete-worker:
        $(MAKE) -C worker delete
```
意味不明。。。


- redis/Makefile

```
.PHONY: all
all: deploy

.PHONY: deploy
deploy:
        kubectl create -f spec.yaml

.PHONY: delete
delete:
        kubectl delete --ignore-not-found -f spec.yaml
```
はぁ～なるほど、サブディレクトリのほうで実行コマンドを指定している感じか

- spec.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
spec:
  ports:
  - port: 6379
    targetPort: redis-server
  selector:
    app: redis
    role: master

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: master
    spec:
      containers:
      - image: redis
        name: redis-master
        ports:
        - name: redis-server
          containerPort: 6379
```
Redisを使ったアプリも作ってみたいな

## 感想

このラボで学べることはあんまりなかったけど、
VisionAPIを使って画像をラベル分けは最近のアプリには要素で取り入れると面白そう