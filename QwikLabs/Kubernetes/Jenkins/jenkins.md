# Jenkinsを使用したCD のメモ

## このラボで学ぶもの

- Kubernetes Engine で Jenkins クラスターを作成する
- Jenkins のインストール
  - helm
  - credensial設定
- Jenkins の機能を試す
- Jenkins のパイプラインを作成してビルド&デプロイを自動化する
  - SourceRepositoryとの連携


## リポジトリクローン

```sh
git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
```

## クラスタ作成

- Jenkinsが稼働するクラスタを作成する
  - ノード数:2
  - マシンタイプ:n1-standard-2
  - アクセススコープ:SourceRepo(rw),cloudplatform(container registry用)

```sh
$ gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"
```

- 認証情報取得
```sh
$ gcloud container clusters get-credentials jenkins-cd

$ kubectl cluster-info
```

## Helm でJenkinsをインストールする

```sh
$ helm repo add jenkins https://charts.jenkins.io

$ helm repo update

$ helm install cd jenkins/jenkins -f jenkins/values.yaml --version 1.2.2 --wait
NAME: cd
LAST DEPLOYED: Thu Apr 29 10:27:41 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace default cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080
  kubectl --namespace default port-forward $POD_NAME 8080:8080

3. Login with the password from step 1 and the username: admin


For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
```


## クラスター設定

### jenkinsサービスアカウント作成

```sh
$ kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins
```

### ポート転送

```sh
$ export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```

## Jenkins接続

### パスワード取得

admin のパスワード
```sh
$ printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```
Cloud Shellの8080でプレビューでJenkinsのUI画面にいける


## アプリケーションのデプロイ

### 名前空間作成

```sh
$ kubectl create ns production
```

### デプロイ作成

作成した名前空間上にデプロイする
```sh
$ kubectl apply -f k8s/production -n production

$ kubectl apply -f k8s/canary -n production

$ kubectl apply -f k8s/services -n production
```

この状態だと、本番環境とカナリア環境が1つずつ作成されている状態  
本番のフロントエンドをスケールアップする
```sh
$ kubectl scale deployment gceme-frontend-production -n production --replicas 4
```
これで、フロントエンドが本番:カナリア=4:1になって、20%のユーザがカナリアに接続するようになる


## Jenkins パイプライン作成

### Source Repository作成

```sh
$ gcloud source repos create default

$ git init

$ git config credential.helper gcloud.sh

$ git remote add origin [作成した repo の URL]

$ git config --global user.email "適当に設定"

$ git config --global user.name "適当に設定"

$ git add .

$ git commit -m "Initial commit"

$ git push origin master
```

### サービスアカウントの認証情報を追加

Jenkinsがソースリポジトリにアクセスできるようにするためには認証情報を構築する必要がある  
Jenkinsは、自身が稼働しているGKEクラスタのサービスアカウント認証情報を使用して、SourceRepoからコードをDLする

操作は、JenkinsUIで行う(日本語と英語が入り混じっててわかりづらい)
   1. Jenkinsの管理 → Manage Credentials
   2. Jenkins を選択
   3. Global Credentials を選択
   4. Add Credentials を選択
   5. ドロップダウンから Google ServiceAccout from metadata を選択

### Jenkins jobの作成

- UI
  1. 新しいジョブの作成
  2. sample-app というプロジェクト名をつけて Multibranch Pipeline を選択
  3. Branch Soueces で、 Add Source して git を選択
  4. Source Repo の RepoURL を入力
  5. Credentials ドロップダウンからさっき作成した認証情報を選択
  6. Scan Multibranch Pipeline Triggers の Periodically if not othewise run をオンにして interval を 1minuteにする
  7. 保存

保存するとジョブが実行されるけど、初回は失敗する模様  
気にせず、つぎの手順をやる


これで、SourceRepoにpushされたのをトリガーとして、アプリをビルド、デプロイを自動でやってくれる

## 開発環境作成

### ブランチの作成

```sh
$ git checkout -b new-feature
```

### アプリの更新

バージョン番号とか、ラベルの色を変えたりする
```sh
$ vi Jenkinsfile
$ vi html.go
$ vi main.go
```

### デプロイ開始

```sh
$ git add Jenkinsfile html.go main.go
$ git commit -m "Version 2.0.0"
$ git push origin new-feature
```

JenkinsUIを確認すると、なんか始まる  
kubectl --namespace=new-feature apply... が確認出来たらOK
完了までは結構時間かかる

### 確認

開発環境では、ロードバランサの外部IPは使わずにkubectl プロキシを使う  
プロキシを使うとKubernetesAPIが自ら認証を行って  
ローカルマシンからのリクエストをクラスタ内にリダイレクトしてくれる  
サービスがインターネットに公開されることなく確認作業ができる
```sh
$ kukbectl proxy &
↑すぐ終了して次のなんかが実行される → Ctl * C で終了する
```

```sh
$ curl \
http://localhost:8001/api/v1/namespaces/new-feature/services/gceme-frontend:80/proxy/version

{"version" , "2.0.0"}が返される
```


## カナリアリリースをデプロイ

### ブランチ作成

```sh
$ git checkout -b canary
$ git push origin canary
```
JenkinsUIでデプロイを確認する

### 確認

デプロイ完了後、確認してみる
```sh
$ export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

$ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
```
5回に1回くらいの確率で version2.0.0がでてくる


## 本番環境にデプロイ

### ブランチ作成

```sh
$ git checkout master
$ git merge canary
$ git push origin master
```

### 確認
```sh
$ export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

$ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
```
全部が2.0.0になってる！

`kubectl get service gceme-frontend -n production`でIP取得してブラウザで見てみても、ラベルがオレンジに変わってる！