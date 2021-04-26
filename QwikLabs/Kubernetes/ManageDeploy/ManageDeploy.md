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

## カナリアデプロイ

本番環境で一部のユーザを対象に新しいデプロイをテストするときは、カナリアデプロイを使用する

```yaml
~~
app: hello
track: canary
version: 2.0.0
~~
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