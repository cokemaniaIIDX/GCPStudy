# Manage Deploy のメモ

## 演習内容

- kubectlツール
- yamlファイルでのデプロイ
- リリース、更新、スケーリング
- デプロイスタイルの変更

## Deploymentの詳細

`kubectl explain deployment`を使うことでdeploymentオブジェクトに関する情報が表示される  
Deploymentの構造とか書くフィールドの機能を理解するのに役立つ

`kubectl explain deployment --recursive`でフィールをすべて確認

- 例
  - deployment.metadata.name
  - spec.replicas

## Deploymentのスケール

`kubectl scale deployment`コマンドでスケールできる

```sh
$ kubectl scale deployment hello --replicas=5
```

## ローリングアップデート

`kubectl edit deployment hello`でローリングアップデートができる

`kubectl rollout pause deployment/hello`で一時停止できる 再開は`resume` ステータスは`status`

- ロールバック

新しいバージョンでバグった場合にロールバックする方法

- `rollout`コマンドを使う

```sh
$ kubectl rollout undo deployment/hello

$ kubectl rollout history deployment/hello
```

### ログ

- ローリングアップデート
```sh
$ kubectl edit deployment hello
エディタが開くのでコンテナバージョンを2.0.0に変更して保存
deployment.apps/hello edited
↑edit開始

$ kubectl get replicaset
NAME                  DESIRED   CURRENT   READY   AGE
auth-7cffdb8677       1         1         1       6m56s
frontend-7b4b97c4dc   1         1         1       4m29s
hello-687bb4b8f4      3         3         3       5m35s
hello-d99c798f        1         1         1       10s
↑新しいバージョンが作成されてる

$ kubectl get replicaset
NAME                  DESIRED   CURRENT   READY   AGE
auth-7cffdb8677       1         1         1       7m27s
frontend-7b4b97c4dc   1         1         1       5m
hello-687bb4b8f4      0         0         0       6m6s
hello-d99c798f        3         3         3       41s
↑新しいバージョンにすべて移行

$ kubectl rollout history deployment/hello
deployment.apps/hello
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
↑リビジョンが進む
```

- ロールバック
```sh
$ kubectl rollout undo deployment/hello
deployment.apps/hello rolled back
↑ロールバック開始

$ kubectl rollout history deployment/hello
deployment.apps/hello
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
↑リビジョンが進む

$ kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
auth-7cffdb8677-7wc8h           kelseyhightower/auth:1.0.0
frontend-7b4b97c4dc-xdnwc               nginx:1.9.14
hello-687bb4b8f4-9gdpf          kelseyhightower/hello:1.0.0
hello-687bb4b8f4-r6sz2          kelseyhightower/hello:1.0.0
hello-687bb4b8f4-v547j          kelseyhightower/hello:1.0.0
hello-d99c798f-kv9mk            kelseyhightower/hello:2.0.0
↑今のコンテナ(2.0.0)はそのままで、新しいもの(ver1.0.0)が増える

$ kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
auth-7cffdb8677-7wc8h           kelseyhightower/auth:1.0.0
frontend-7b4b97c4dc-xdnwc               nginx:1.9.14
hello-687bb4b8f4-9gdpf          kelseyhightower/hello:1.0.0
hello-687bb4b8f4-r6sz2          kelseyhightower/hello:1.0.0
hello-687bb4b8f4-v547j          kelseyhightower/hello:1.0.0
↑1.0.0にロールバック完了
```

## カナリアデプロイ

本番環境で一部のユーザを対象に新しいデプロイをテストするときは、カナリアデプロイを使用する

```yaml
~~
app: hello
track: canary
version: 2.0.0
~~
```

### 状態

今、deploymentsにcanaryのdeploymentが追加されたので、  
サービスhello は、トラフィックをhelloとhello-canaryの両方に割り振る  
(hello:3 , hello-canary:1なので1/4のユーザのみ新機能にアクセスする状態)

- 確認
```sh
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"1.0.0"}
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"2.0.0"}
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"1.0.0"}
```

### セッションアフィニティ

カナリアデプロイ中に、ユーザがUIの変更にぶち当たると困惑するので、  
同じユーザは常に同じバージョンで処理してほしい → セッションアフィニティを使う

```yaml
spec:
  sessionAffinity: ClientIP
  selector:
    app: "hello"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

## Blue/Greenデプロイ

古いBlueバージョンと新しいGreenバージョンの2つのデプロイを作成してLBで振り分ける

- 概要
旧環境と新環境を両方用意して、kubectl apply で切り替える感じ