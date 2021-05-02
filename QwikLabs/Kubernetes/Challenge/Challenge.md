# CHallenge のメモ(手順)

- やること
  - Dockerイメージを作成してDockerfileに保存
  - Dockerイメージのテスト
  - DockerイメージをContainerRepositoryにPush
  - イメージを更新してDeploymentにPush
  - Jenkinsパイプラインで自動化

`source <(gsutil cat gs://cloud-training/gsp318/marking/setup_marking.sh)`で進行状況をチェックできる

## Task1

- source repo からリポジトリをクローンしてくる
```
$ git clone https://source.developers.google.com/p/qwiklabs-gcp-03-4c5b064134cf/r/valkyrie-app
```

- Dockerイメージを作成する
  - valkyrie-app/に作成
```sh
$ vi Dockerfile
FROM golang:1.10
WORKDIR /go/src/app
COPY source .
RUN go install -v
ENTRYPOINT ["app","-single=true","-port=8080"]
```

- Dockerfileを使用して、タグv0.0.1を持つvalkyrie-appという名前のDockerイメージを作成
```sh
$ docker build -t valkyrie-app:v0.0.1 .
```

- 進行状況チェック実行
```
$ . step1.sh
```

## Task2

- Dockerイメージのテスト
```sh
$ docker run -p 8080:8080 --name valkyrie-app valkyrie-app:v0.0.1 &
```

- 進行状況チェック
```
$ . step2.sh
```

## Task3

- GCRにPushする
```sh
$ docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/valkyrie-app:v0.0.1 .
$ docker push gcr.io/$GOOGLE_CLOUD_PROJECT/valkyrie-app:v0.0.1
```


## Task4

GKEでデプロイメントを作成して公開する  

- valkyrie-app/k8sに、deployment.yaml と service.yamlがあるので、これを使ってクラスタ,デプロイメントを作成する
```sh
$ gcloud config set compute/zone us-east1-b

$ gcloud container clusters create valkyrie-dev \
--num-nodes 2 \
--machine-type n1-standard-1 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"

$ kubectl apply -f k8s/deployment.yaml
$ kubectl apply -f k8s/service.yaml
```
↑やっぱり既にありましたわ us-east1-dにあった


- 認証情報を取得する
```sh
$ gcloud container clusters get-credentials valkyrie-dev
```

- deployment.yaml と service.yaml を修正する
```sh
$ vi deployoment.yaml
image: IMAGE_HERE を image: gcr.io/$GOOGLE_CLOUD_PROJECT/valkyrie-app:v0.0.1 に変更

$ vi service.yaml
変えなくてよさそう
```

- デプロイする
```sh
$ kubectl apply -f k8s/deployment.yaml
$ kubectl apply -f k8s/servicce.yaml

確認
$ kubectl get svc
```


## Task5

- レプリカ数を3に増やす
```sh
$ kubectl scale deployment valkyrie-dev --replicas 3

確認
$ kubectl describe deployment valkyrie-dev
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
```

- Kurtとかいうやつの変更をmasterにマージする
  - 多分、html.go のdivのblueがgreenになっている
```sh
$ git merge origin/kurt-dev
```

- v0.0.2としてビルドして、GCRにPush、valkyrie-devクラスタで再デプロイする
```sh
$ vi k8s/deployment.yaml
v0.0.1 → v0.0.2 にする
$ docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/valkyrie-app:v0.0.2 .
$ docker push gcr.io/$GOOGLE_CLOUD_PROJECT/valkyrie-app:v0.0.2
$ kubectl apply -f k8s/deployment.yaml
```

- カードが緑色になっているので確認
`kubectl get svc`のExternalIPをウェブブラウザで見てみる


## Tak6

Jenkinsで自動化する  
すでにJenkinsコンテナがデプロイされている

- パスワード取得
```sh
$ printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

- コンソールに接続
```sh
Task1で起動したコンテナがポートを使用して邪魔になるので、停止する
$ docker ps
$ docker kill 81c137fb2420(コンテナID)

$ export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")

$ kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```

- ウェブでプレビューする
  - adminユーザをさっき取得したパスワードでログイン
  - Install suggested plugins で簡易インストール
  - Jenkinsの管理 → Manage Credentials
    - jenkins → グローバルドメイン → 認証情報を追加
      - google service account from metadata
  - 新規ジョブ作成
    - item name:valkyrie-app
    - Multibranch Pipeline でOK
    - Branch Sources → Add source → Git
      - 認証情報:さっきつくったやつ
      - リポジトリ:https://source.developers.google.com/p/qwiklabs-gcp-03-4c5b064134cf/r/valkyrie-app
        (Source RepoからURLコピーしてくる)
      - ソースコードの`*/master`ブランチを参照するようにパイプラインジョブを作成する
    - 一応Scan Multibranch Pipline Triggersをチェックして 間隔を1minuteにしておく

- Jenkinsファイルを修正
```sh
$ vi Jenkinsfile
YOUR_PROJECTの所をプロジェクトIDで置き換える

$ vi source/html.go
green → orange 変更
```

- git 操作
```sh
$ git add Jenkinsfile source/html.go

$ git config --global user.email "sbtkzks@gmail.com"
$ git config --global user.name "coke@qwiklabs"

$ git commit -m "version0.0.2"
$ git push origin master
```

## ビルドでエラーが出たので

調査してみた  
masterブランチをクリックすると、グラフみたいなのが見れる  
赤色になってるのが失敗で、マウスホバーするとログボタンがあるのでクリック  
ログが表示されて、何故失敗したかがわかる  
今回は、Dockerfileをコミットしてなかったから失敗した  
Dockerfileもコミットしてプッシュしたらまたビルド開始  
今度はうまくいった