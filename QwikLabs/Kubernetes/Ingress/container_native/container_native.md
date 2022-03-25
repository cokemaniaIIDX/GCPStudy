# 【GKE】コンテナネイティブの負荷分散Ingressを構築する【ingress】

GKE Ingressの負荷分散では、

- `マネージドインスタンスグループ(MIG)`にあるGCEインスタンス内のPodに`kube-proxyを介して`トラフィックを分散する従来型の負荷分散
- `ネットワークエンドポイントグループ(NEG)`にあるPodに`直接トラフィックを分散`するコンテナネイティブの負荷分散

の2種類があります

今回紹介するコンテナネイティブの負荷分散は、従来のものと比べて以下のメリットがあります

- ネットワークパフォーマンスの改善
  - LBが直接Podと通信を行うため、proxyを介す従来型と比べてネットワークホップが少なく、レイテンシとスループットの改善が見込める
- 可視性の向上
  - LB-Pod間のレイテンシを確認できるため、NEGレベルでのトラブルシューティングが容易になる また、ヘルスチェックも容易になる
- 高度なLB機能のサポート
  - Cloud Armor(WAF), Cloud CDN(CDN), Identity-Aware Proxy との統合ができる

CloudArmorでアクセス制限をかけたいため、コンテナネイティブのクラスタを作ることになりました

# 実際に構築

では作成していきます
基本的に[前回の記事](../gkeingress_new.md)と同じ手順です

## 要件

https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing?hl=ja#requirements

- VPCネイティブのGKEクラスタである必要があります

> コンテナ ネイティブのロード バランシングを使用するには、クラスタが VPC ネイティブである必要があります。

https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing?hl=ja#create_cluster

- 有効なエイリアスIPを使用したクラスタである必要があります

> コンテナ ネイティブの負荷分散を使用するには、有効なエイリアス IP を使用したクラスタを作成する必要があります。

## クラスタ作成

上記の要件を踏まえて下記の通り設定して作成します
※↓表はTerraformで作成する際の設定Key-Valueです

| 項目                   | 設定値           |
| ---------------------- | ---------------- |
| cluster_name           | ingress-cluster  |
| location               | asia-northeast1  |
| preemptible            | 有効             |
| node数                 | 1                |
| node_pool_name         | ingress_nodepool |
| node_machine_type      | e2-medium        |
| service account        | ingress-cluster  |
| **networking_mode**    | `VPC_NATIVE`     |
| **cluster ipv4 range** | `192.168.0.0/16` |
| **service ipv4 range** | `172.16.0.0/16`  |

※注意: cluster ip range , service ip range は VPC内の既存のサブネットのrangeと被らないように設定する必要があります
例えば、 VPC内のネットワーク subnet1 が `10.0.0.0/24`の場合、`10.0.0.0/16`などは使えません(/24の範囲は/16の中に含まれているため)

### 認証情報取得

kubectl を実行できるように認証情報を取得します

```
$ gcloud container clusters get-credentials --region=asia-northeast1 ingress-cluster
```

## Cloud Armor ルール作成

クラスタへのIP制限を行うために`CloudArmor`を作成します (Cloud Armorは`AWS WAF`と同じような機能を提供しています)

内容:

| 項目             | 設定値1                                              | 設定値2                      |
| ---------------- | ---------------------------------------------------- | ---------------------------- |
| 名前             | ingress-default                                      | -                            |
| アクション(許可) | IP範囲(<家のネットのIP>), IP範囲(開発サーバの外部IP) | 優先度:1000                  |
| アクション(拒否) | IP範囲(* すべて)                                     | 優先度: 2147483647(一番低い) |

設定したルールはこの後設定する`backendconfig`で適用します

## Kubernetes操作

### Deployment 作成、適用

```yaml:ingress_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-app-1
        image: "asia-northeast1-docker.pkg.dev/<PROJECT_ID>/ingress/custom-nginx:v1.0"
        ports: 
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthcheck.html
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

nginxのPodで、8080ポートをリッスンします
ヘルスチェック用にhealthcheck.htmlを用意しています

- 適用

```
$ kubectl apply -f ingress_deployment.yaml
```

### Service 作成、適用

```yaml:ingress_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-service
  annotations:
    cloud.google.com/backend-config: '{"ports": {"80":"bec"}}'
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

- **cloud.google.com/backend-config**:
  - ヘルスチェック用にバックエンドコンフィグを用意します(バックエンドコンフィグを使わない場合、ヘルスチェックは常にパス`/`になるため)
- **cloud.google.com/neg:**:
  - ingressがネットワークエンドポイントグループを利用するコンテナネイティブの負荷分散を利用することを明示します
- **type**:
  - 従来のLBでは`NodePort`を指定しましたが、コンテナネイティブの場合は`ClusterIP`を指定します

- 適用

```
$ kubectl apply -f ingress_service.yaml
```

### backendconfig 作成、適用

```yaml:ingress_backendconfig.yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: bec
spec:
  securityPolicy:
    name: ingress-default
  healthCheck:
    timeoutSec: 30
    checkIntervalSec: 60
    type: HTTP
    port: 80
    requestPath: /healthcheck.html
    healthyThreshold: 1
    unhealthyThreshold: 5
```

- **securityPolicy**: 先ほど作成したCloudArmorルールを指定します
- **healthCheck**: `/healthcheck.html`を見てヘルスチェックを行うように指定します

### ingress 作成、適用

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
  - host: ingress.<ドメイン>.net
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: ingress-service
            port:
              number: 80
```

- ingress.global-static-ip-name: 固定外部IPは前回の記事で使ったものと同様です
- kubernetes.io/ingress.allow-http: HTTPは拒否するようにします

## 確認

```
$ kubectl describe ingress ingress | grep backends
Annotations: ingress.kubernetes.io/backends: {"k8s-be-32540--fd36cae49148e94e":"HEALTHY","k8s1-fd36cae4-default-ingress-service-80-c3fb220f":"HEALTHY"}
```

HEALTHYであることが確認出来たら、LBのドメインにブラウザでアクセスしてみます

`https://ingress.<ドメイン>.com/`

nginxのウェルカムページが表示されればOK！

# おわりに

今回は[前回の記事](../gkeingress_new.md)を応用して、`コンテナネイティブ`のIngress負荷分散を作成しました

コンテナネイティブのほうが従来のものよりメリットが多いし、これを使わない手はないかと思います
Cloud Armor, Cloud CDNと統合できるのが重要かなと思います

次は新機能のGatewayを試してみようと思います

では、また