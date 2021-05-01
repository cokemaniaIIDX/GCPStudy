# Jenkinsを使用したCD のメモ

## このラボで学ぶもの

- Kubernetes Engine で Jenkins クラスターを作成する
- Jenkins のインストール
  - helm
  - credensial設定
- Jenkins の機能を試す
- Jenkins のパイプラインを作成してビルド&デプロイを自動化する
  - SourceRepositoryとの連携


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