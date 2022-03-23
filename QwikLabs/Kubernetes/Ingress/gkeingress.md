# GKE Ingress オブジェクトを利用して HTTPS 負荷分散する

GKEの`Ingressオブジェクト`を利用した HTTPS LB を作成して、
バックエンドに対する負荷分散を行う

## 仕様

- Nginxコンテナは8080ポートでLISTEN
- Nginxコンテナを展開するServiceは80ポートとpodの8080ポートを接続させる

## 前提

- 確認用VMインスタンスにNGINXをインストール済み
- 80 port を開ける 443も最初の確認時に開ける
  - LBでTLS終端させる想定なのでインスタンスは80のみでOK
- Domainを取得している(名前.com とか GoogleDomain)
- DNSサーバはCloudDNSを使う

## 準備

- TLSを使えるように証明書とドメインを確保
  - TLS証明書(Let's Encryptで発行)
  - ドメイン(Google Domainで購入)

## 手順

### Let's EncryptでTLS証明書発行

[Certbotのインストールガイド](https://certbot.eff.org/instructions?ws=nginx&os=centosrhel8)を見ながらやっていく
`RockyLinux`がなかったので`CentOS8`のやつをみてやったけどうまくいった

snapd というパッケージマネージャのようなものでインストールする方式にかわった模様
[snapdのインストールページ](https://snapcraft.io/docs/installing-snapd)を見て導入する

- snapd install

まずsnapdをインストールする

```
# dnf install -y snapd

# systemctl enable --now snapd.socket
Created symlink /etc/systemd/system/sockets.target.wants/snapd.socket → /usr/lib/systemd/system/snapd.socket.

# ln -s /var/lib/snapd/snap /snap

# systemctl start snapd

# systemctl status snapd
● snapd.service - Snap Daemon
   Loaded: loaded (/usr/lib/systemd/system/snapd.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-03-21 17:17:25 JST; 4s ago
 Main PID: 202608 (snapd)
    Tasks: 8 (limit: 49466)
   Memory: 14.3M
   CGroup: /system.slice/snapd.service
           └─202608 /usr/libexec/snapd/snapd
```

- certbot install

```
# snap install core; snap refresh core
core 16-2.54.4 from Canonical✓ installed
snap "core" has no updates available

# snap install --classic certbot
certbot 1.25.0 from Certbot Project (certbot-eff✓) installed
```

- setup certbot

```
# ln -s /snap/bin/certbot /usr/bin/certbot

# snap set certbot trust-plugin-with-root=ok

# snap install certbot-dns-google
certbot-dns-google 1.25.0 from Certbot Project (certbot-eff✓) installed
```

### Google Cloud でサービスアカウント作成

certbotがCloudDNSのリソースにアクセスできるように
サービスアカウントを作成する

- 名前   : certbot@<project_id>.iam.gserviceaccount.com
- ロール : roles/dns.admin (DNS 管理者)

作成したら、「キー」タブより、「鍵を追加」→「新しい鍵を作成」→「JSON」を選択して「作成」

自動的にPCに保存されるので、ファイルの中身を確認用VMのどこかに保存
(今回は`.secrets/google/certbot_sa.json`として保存した)

### 証明書作成

ワイルドカード証明書(*.example.com)を発行する
発行方法は各DNSサーバによって違うので、それぞれ自分が利用するものを確認する
[CloudDNSを利用する場合の方法はこちら](https://certbot-dns-google.readthedocs.io/en/stable/)
↑のExapmlesに書いてあるコマンドを実行する
※google.jsonはさっき作成&DLしたサービスアカウントの鍵のこと

```
# certbot certonly --dns-google --dns-google-credentials ~/.secrets/google/certbot_sa.json -d *.example.com -i nginx

Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): cokemaniaIIDX@gmail.com ←メールアドレスを入力

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y ←ライセンスを読み、同意する

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y ← 登録メールに団体からのメールを受け取るかどうか
Account registered.
Requesting a certificate for *.example.com
Waiting 60 seconds for DNS changes to propagate

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/example.com/privkey.pem
This certificate expires on 2022-06-19.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

`/etc/letsencrypt/live/<ドメイン>/`に鍵と証明書がインストールされるので、
nginxに持っていく

```
# cp /etc/letsencrypt/live/<ドメイン>/privkey.pem /etc/nginx/tls/server.key
# cp /etc/letsencrypt/live/<ドメイン>/fullchain.pem /etc/nginx/tls/server.crt
```

SSLの設定を追加

```
# vi /etc/nginx/conf.d/tls.conf
server {
    listen 443 ssl;

    ssl_certificate             /etc/nginx/tls/server.crt;
    ssl_certificate_key         /etc/nginx/tls/server.key;
}
```

設定確認し、reload

```
# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# nginx -s reload

# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-03-21 16:41:10 JST; 2h 34min ago
```

- ブラウザで確認

`https://<ドメイン>/`にアクセスしてアクセスできることを確認

![confirm](https_confirm.png)

これで有効なTLS証明書を発行できた

### GKE cluster 作成

- terraform で作成

- 設定:

| 項目              | 設定値           |
| ----------------- | ---------------- |
| cluster_name      | ingress-cluster  |
| location          | asia-northeast1  |
| preemptible       | 有効             |
| node数            | 1                |
| node_pool_name    | ingress_nodepool |
| node_machine_type | e2-medium        |
| service account   | ingress-cluster  |

- サービスアカウントに権限付与

clusterに設定するサービスアカウントは`Artifact Registry`にアクセスできる必要があるので、権限を付与する
権限は`ArtifactRegistry 読み取り (roles/artifactregistry.reader)`

- kubeconfigエントリの生成

↓のコマンドを実行することで、`kubectl`コマンドを実行したときの対象が`さっき作成したクラスタ`になる

```
$ gcloud container clusters get-credentials --region=asia-northeast1 ingress-cluster

$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://35.187.207.69
  name: gke_<PROJECTID>_asia-northeast1_ingress-cluster
contexts:
- context:
    cluster: gke_<PROJECTID>_asia-northeast1_ingress-cluster
    user: gke_<PROJECTID>_asia-northeast1_ingress-cluster
  name: gke_<PROJECTID>_asia-northeast1_ingress-cluster
current-context: gke_<PROJECTID>_asia-northeast1_ingress-cluster
kind: Config
preferences: {}
users:
- name: gke_<PROJECTID>_asia-northeast1_ingress-cluster
  user:
    auth-provider:
      config:
        cmd-args: config config-helper --format=json
        cmd-path: /usr/lib64/google-cloud-sdk/bin/gcloud
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```

### Artifact Registry 作成

- terraform で作成

- 設定:

| 項目            | 設定値          |
| --------------- | --------------- |
| repository_name | ingress         |
| location        | asia-northeast1 |
| format          | DOCKER          |

- 認証の構成

↓のコマンドを実行することで、DockerがArtifact Registryに接続してイメージのpush,pullができるようになる

```sh
$ gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```

### コンテナ作成、GCRにプッシュ

nginxの公式イメージをプルしてきて*いろいろいじった上で*`Google Artifact Registry`にプッシュする

- pull

```
$ docker pull nginx:latest

$ docker images
REPOSITORY                                                TAG       IMAGE ID       CREATED         SIZE
nginx                                                     latest    f2f70adc5d89   3 days ago      142MB
```

#### いじる内容

- nginx.conf
  - port:8080 でLISTENしたいので、nginx.confを修正する
  - if_modified_sice : offにする
    - ↑onだと、変更がなかった時に304のレスポンスを返すようになり、LBのヘルスチェックを不合格にしてしまう
      - GKE Ingress のLBは200のみヘルスチェックを合格させるため

```conf:nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            off;
    if_modified_since   off; # for healthcheck
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       8080 default_server;
        listen       [::]:8080 default_server;
        server_name  _;
        root	     /usr/share/nginx/html;

        include /etc/nginx/default.d/*.conf;

        location / {
	      }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
```

- healtchcheck.html

コンテナ内に配置するhealthcheck用のHTMLファイルを作成

```html:healthcheck.html
<html>
<head>
    <title>healthcheck</title>
</head>
<body>
    healthcheck ok.
</body>
</html>
```

#### dcoker 操作

- Dockerfile 作成

nginxの公式イメージをプルしてきて、
さっき作ったファイルをコンテナ内に配置する設定

```Dockerfile:Dockerfile
FROM nginx:latest
LABEL maintainer="cokemaniaIIDX cokemaniaIIDX@gmail.com"

RUN echo "building nginx server..."

RUN echo "copy conf files"
COPY nginx.conf /etc/nginx/nginx.conf

RUN echo "copy healthcheck file"
COPY healthcheck.html /usr/share/nginx/html/healthcheck.html
```

- docker build実行

```sh
$ docker build . custom-nginx
```

- tagづけ

pull してきた nginx イメージを Artifact Registryにアップロードできるようにタグ付けする

```
$ docker tag nginx asia-northeast1-docker.pkg.dev/<GCP_PROJECT_ID>/ingress/custom-nginx:v2.0

確認
$ docker images
REPOSITORY                                                                TAG       IMAGE ID       CREATED          SIZE
asia-northeast1-docker.pkg.dev/<PROJECTID>/ingress/custom-nginx   v2.0      a3e3c5c36651   45 minutes ago   142MB
```

- push

imageをpush！

```
$ docker push asia-northeast1-docker.pkg.dev/<PROJECTID>/ingress/nginx:v2.0
```

### Deployment の作成

```yaml:ingress_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: app-1
        image: "asia-northeast1-docker.pkg.dev/<PROJECTID>/ingress/custom-nginx:v2.0"
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthcheck.html
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

- 適用

```
$ kubectl apply -f ingress_deployment.yaml
deployment.apps/ingress-deployment created
```

### Service の作成

```yaml:ingress_service.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    cloud.google.com/backend-config: '{"ports":{"8080":"bec"}}'
  name: ingress-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
```

- 適用

```
$ kubectl apply -f ingress_service.yaml
service/ingress-service created
```

### backend config の作成

[公式ドキュメント参照](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-features?hl=ja)

```yaml:ingress_backendconfig.yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: bec
spec:
  healthCheck:
    requestPath: /healthcheck.html
  timeoutSec: 60
```

### Ingress の作成

```yaml:ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
spec:
  rules:
  - http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: ingress-service
            port:
              number: 80
```

- 適用

```
$ kubectl apply -f ingress.yaml
ingress.networking.k8s.io/ingress created
```

※ingressを作成すると、ingressが勝手にLBを作成します
「ネットワークサービス」→「Cloud Load Balancing」から確認してみよう

## 接続テスト

```sh
$ kubectl get ingress
NAME      CLASS    HOSTS   ADDRESS          PORTS   AGE
ingress   <none>   *       <ここにIP>       80      88m

$ kubectl describe ingress ingress
Name:             ingress
Labels:           <none>
Namespace:        default
Address:          <ここにIP>
Default backend:  default-http-backend:80 (10.148.3.2:8080)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /*   ingress-service:80 (10.148.3.19:8080,10.148.3.20:8080,10.148.3.21:8080)
Annotations:  ingress.kubernetes.io/backends: {"k8s-be-30073--850c385e18f8440c":"HEALTHY","k8s-be-32066--850c385e18f8440c":"HEALTHY"} # ← HEALTHYであることを確認する
              ingress.kubernetes.io/forwarding-rule: k8s2-fr-2vr3o74u-default-ingress-fgx1jtuq
              ingress.kubernetes.io/target-proxy: k8s2-tp-2vr3o74u-default-ingress-fgx1jtuq
              ingress.kubernetes.io/url-map: k8s2-um-2vr3o74u-default-ingress-fgx1jtuq
              kubernetes.io/ingress.class: gce
Events:
  Type     Reason     Age                   From                     Message
  ----     ------     ----                  ----                     -------
  Warning  Translate  45m (x16 over 48m)    loadbalancer-controller  Translation failed: invalid ingress spec: service "default/ingress-service" is type "ClusterIP", expected "NodePort" or "LoadBalancer"
  Normal   Sync       44m                   loadbalancer-controller  UrlMap "k8s2-um-2vr3o74u-default-ingress-fgx1jtuq" updated
  Normal   Sync       2m36s (x16 over 89m)  loadbalancer-controller  Scheduled for sync
```

`describe ingress`コマンドで`backend`が`HEALTH`であることが確認できれば、接続可能なはずです
表示されているIPにブラウザからアクセスしてみよう

↓が表示されればOK

![hello, nginx](hello_nginx.png)

## TLS対応させる

### 固定IPを取得

- terraformで作成

IPは`プレミアム`じゃないとうまくいかない
→[参考](https://cloud.google.com/kubernetes-engine/docs/how-to/service-parameters?hl=ja#lb_ip)

また、グローバルIPじゃないと機能しない(=リージョンIPアドレスはIngressでは使えない)
→[参考](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip?hl=ja#use_an_ingress)

```s:google_compute_address.tf
resource "google_compute_global_address" "ingress_ip" {
  name         = "ingress-ip"
  description  = "external IP for Ingress-Cluster"
}
```

### Kubernetes Secret オブジェクト作成

certbotで取得した証明書を`Kubernetes Secret`オブジェクトとして作成

- Kubernetes Secret オブジェクト作成

```sh
$ kubectl create secret tls ingress-tls --cert server.crt --key server.key
secret/ingress-tls created
```

- Ingressに適用

```yaml:ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    kubernetes.io/ingress.global-static-ip-name: ingress-ip
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.allow-http: "false"
spec:
  tls:
  - secretName: ingress-tls
  rules:
  - http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: ingress-service
            port:
              number: 80
```

### Cloud DNS で DNSレコード登録

- コンソールで実施

↓の通りでDNSレコードを登録

```
ingress.<ドメイン>.net 300 IN A <さっき作成した固定IP>
```

## 確認

ブラウザで`https://ingress.<ドメイン>.net/`にアクセス！！

↓が表示されたら完璧です

![hello_nginx_with_https](hello_nginx_with_https.png)