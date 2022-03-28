# Ingress - ドメイン名で複数のServiceにトラフィックを振り分ける

サービスを増やしていった際に、LBのルーティングをどう設定するのかを確認
ホスト名で3つ分けて、それぞれ別のserviceに転送されるようにする

# 気づき

- 今回の作業で一番の気づきになった点 : `BackendConfig で指定するポート`

コンテナネイティブの負荷分散Ingressの場合、
LBのヘルスチェックはNEG内のPodに直接行うことになる
つまり、**BackendConfigのヘルスチェックポートはコンテナがリッスンしているポートを指定する必要がある**
(今回の場合Nginxが8080をリッスンしているので`8080`)

それ以外は過去のやつを3つ並列にコピペしただけでOK

## Cluster 認証

```
$ gcloud container clusters get-gredentials --region=asia-northeast1 ingress-cluster
```

## TLS登録

```
$ cd .certs
$ kubectl create secret tls ingress-tls --cert server.crt --key server.key
```

## Deployment

```yaml:ingress_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-deployment1
spec:
  selector:
    matchLabels:
      app: nginx1
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx1
    spec:
      containers:
      - name: nginx-app-1
        image: "asia-northeast1-docker.pkg.dev/imade-gaming-265014/ingress/ingress-nginx:v1.0"
        ports: 
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthcheck.html
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-deployment2
spec:
  selector:
    matchLabels:
      app: nginx2
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx2
    spec:
      containers:
      - name: nginx-app-2
        image: "asia-northeast1-docker.pkg.dev/imade-gaming-265014/ingress/ingress-nginx:v2.0"
        ports: 
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthcheck.html
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-deployment3
spec:
  selector:
    matchLabels:
      app: nginx3
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx3
    spec:
      containers:
      - name: nginx-app-3
        image: "asia-northeast1-docker.pkg.dev/imade-gaming-265014/ingress/ingress-nginx:v3.0"
        ports: 
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthcheck.html
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Service

```yaml:ingress_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-service1
  annotations:
    cloud.google.com/backend-config: '{"ports": {"8080":"bec1"}}'
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: nginx1
  ports:
  - protocol: TCP
    port: 30001
    targetPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: ingress-service2
  annotations:
    cloud.google.com/backend-config: '{"ports": {"8080":"bec2"}}'
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: nginx2
  ports:
  - protocol: TCP
    port: 30002
    targetPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: ingress-service3
  annotations:
    cloud.google.com/backend-config: '{"ports": {"8080":"bec3"}}'
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: nginx3
  ports:
  - protocol: TCP
    port: 30003
    targetPort: 8080
```

## backendconfig

### 注意点

コンテナネイティブの負荷分散の場合、ヘルスチェックはそのままコンテナのポートで行うようになるので、
backend-config の指定ポートはコンテナのリッスンポートにする必要がある

→`どのコンテナもヘルスチェックのパスとポートが同じの場合`、backendconfigは1つで良いかも
→基本はそれぞれ別のサービスとかコンテナになるし、ヘルスチェックも違ってくると思うから、まぁこれはこれでOK

```yaml:ingress_backendconfig.yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: bec1
spec:
  securityPolicy:
    name: ingress-default
  healthCheck:
    timeoutSec: 30
    checkIntervalSec: 60
    type: HTTP
    port: 8080
    requestPath: /healthcheck.html
    healthyThreshold: 1
    unhealthyThreshold: 5

---
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: bec2
spec:
  securityPolicy:
    name: ingress-default
  healthCheck:
    timeoutSec: 30
    checkIntervalSec: 60
    type: HTTP
    port: 8080
    requestPath: /healthcheck.html
    healthyThreshold: 1
    unhealthyThreshold: 5

---
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: bec3
spec:
  securityPolicy:
    name: ingress-default
  healthCheck:
    timeoutSec: 30
    checkIntervalSec: 60
    type: HTTP
    port: 8080
    requestPath: /healthcheck.html
    healthyThreshold: 1
    unhealthyThreshold: 5
```

## Ingress

それぞれ`/*`でホスト名でサービスを分けるようにしている
もしパスを`/v1`とかに指定したいとかの場合、コンテナ側で/htmlに/v1ディレクトリが存在している必要がある

```yaml:ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    kubernetes.io/ingress.global-static-ip-name: ingress-ip
    kubernetes.io/ingress.allow-http: "false"
spec:
  tls:
  - secretName: ingress-tls
  rules:
  - host: ingress1.<my-domain>
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: ingress-service1
            port:
              number: 30001
  - host: ingress2.<my-domain>
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: ingress-service2
            port:
              number: 30002
  - host: ingress3.<my-domain>
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: ingress-service3
            port:
              number: 30003
```

## DNS登録

下記3レコードを登録する
それぞれ同じLBのIPで名前解決できるようにする

```zone:zoneファイル
ingress1.<my-domain>. 30 IN A <LBのIP>
ingress2.<my-domain>. 30 IN A <LBのIP>
ingress3.<my-domain>. 30 IN A <LBのIP>
```

## 確認

`https://ingress[1-3].<my-domain>/version.html`にそれぞれアクセスして、それぞれのバージョンのHTMLが表示されればOK