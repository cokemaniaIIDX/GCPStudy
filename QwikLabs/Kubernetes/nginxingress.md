# Nginx を GKE の Ingress Controller として使う

KubernetesはHTTPトラフィックをIngressによって処理する

- Ingress
  - Ingress Resource    : 受信トラフィックがサービスにアクセスするためのルールの集合
  - Ingress Controller  : ルールに従ってトラフィックを処理する機能　→ NGINXを使える

- NGINX Ingress Controllerの特徴
  - 対応        : WebSocket,SSL の負荷分散ができる
  - 書き換え    : URIを書き換えて送信できる
  - セッション永続性  : 同じクライアントからのリクエストは常に同じコンテナに渡される(Plusのみ)
  - JWTs        : JSON Web Token が使える(Plusのみ) JWTsはリクエスト認証に使われる

- helmについて
  - helmはKubernetesのパッケージマネージャで、Linuxでのyumとかaptみたいな感じでアプリをインストールしてコンテナにデプロイできる
  - helm用のラボもあるのであとでやってみる

## ゾーン設定

Kubernetesでは、ゾーンの指定が必須
コンテナとか、永続ディスクのプロビジョニング先を指定しないといけないからかな

```
$ gcloud config set compute/zone asia-northeast1-a
```

## クラスタ作成

```
$ gcloud container clusters create nginx-tutorial --num-nodes 2
```

## helm インストール

```
$ helm version
$ helm repo add stable https://charts.helm.sh/stable
$ helm repo update
```

## GKEでアプリをデプロイ

Cloud RepositoryからHelloとかを返すだけの簡単なWebアプリをデプロイする
これはNginxのバックエンドでアクセスされるやつ

```
$ kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:1.0
```

- 公開

```
$ kubectl expose deployment hello-app --port=8080
```

## helmでNginxIngressControllerをデプロイする

GKEのIngress機能として、NginxのIngressControllerを使う

- ingress controllerのデプロイ

```
$ helm install nginx-ingress stable/nginx-ingress --set rbac.create=true
```

```
$ kubectl get service
```

```
$ kubectl get service nginx-ingress-controller
```

## IngressResoueceを構成する

Ingress Controller は Ingress Resourceをみてルートを決めるので、
IngressResourceを設定してあげる必要がある。

```yaml:ingress-resource.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-resource
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /hello
        backend:
          serviceName: hello-app
          servicePort: 8080
```
  - kubernetes.io/ingress.class: nginx  : IngressControllerにNginxを使うことを指定
  - kubernetes.io/ingress.class: gce    : annotationに何も登録がない場合、GCPのL7LBを使う 強制的に使うにはgceを指定する

- 適用

```
$ kubectl apply -f ingress-resource.yaml

$ kubectl get ingress ingress-resource
```

## 確認

```
$ kubectl get service nginx-ingress-controller
→IPが出力されるので、WEBブラウザでアクセス
```