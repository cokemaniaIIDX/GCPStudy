# GKEでゲームサーバを動かす

## 目標
- Dockerを使ってOpenArenaのDGSのコンテナ起動する
  - コンテナイメージにはゲームのバイナリと必要なライブラリのみがベースイメージに追加される
- 独立した読み取り専用の永続ディスクボリュームにアセットを格納し、実行時にコンテナにマウント
- KubernetesとAPIを使用して基本的なスケジューラプロセスをセットアッ

## 事前に

OpenArenaのクライアント版をインストールしておく

## アーキテクチャ

フロントとしてKubernetesDGSクラスタを実装
バックエンドとしてスケーリングマネージャを実装
開発用途を意識 → 本番はもっといろいろ実装するはず

## 制約

- リアルタイム対戦なので信頼性が必要
- →通信にはUDPを使う
- DGSプロセスはどれも同じくらいの負荷に分散させる
- 対戦は時間制限
- DGSの起動時間は無視できるレベルなので、事前に準備不要
- カスタマーエクスペリエンスへの影響を最小限にする
- ピークが過ぎたからと言ってVMインスタンスを大戦終了前に終わらせたりしない
- ただし、問題が発生して続行できなくなった場合、対戦状態が失われて新しく開始される
- DGSプロセスは静的アセットを読み込むが、アセットへの書き込みは必要ない

## 手順

### サーバをコンテナ化する

OpenArenaという2012年のゲームを使う

- サーババイナリはクライアント同じコードベースからコンパイルされる
- サーバに不要なものは含まれないようにする
- アセットは別の永続ボリュームから読み込まれるようにする

→このアーキテクチャでは、バイナリのみが置換され、消費するディスク容量も少ないので、
コンテナイメージを素早く配布して、更新の負荷を軽減できる

### 準備

CloudShellが使えないらしいのでGCEをたててSDKを使う

ComputeEngine→作成 アクセス権はすべて

- Docker, kubectl インストール

```
sudo su -

apt-get update

apt-get -y install kubectl

apt-get -y install \
apt-transport-https \
ca-certificates \
curl \
gnupg2 \
software-properties-common

// ここからDockerインストール
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")    $(lsb_release -cs) stable"

sudo apt-get update

sudo apt-get -y install docker-ce

sudo usermod -aG docker $USER

docker run hello-world
```

- ゲームサーバのサンプルデモをDL

```
gsutil -m cp -r gs://spls/gsp133/gke-dedicated-game-server .
```

### コンテナイメージ作成

- リージョン指定

今回はアジアでよさそう

```
export GCR_REGION=asia-northeast1 PROJECT_ID=$PROJECT_ID
printf "$GCR_REGION \n$PROJECT_ID\n"
```

- コンテナイメージ作成

まず、構築するイメージの情報を記述したDockerfileを用意する
こんかいは用意してくれてる

```
cd gke-dedicated-game-server/openarena
```

- ビルド

```
docker build -t \
${GCR_REGION}.gcr.io/${PROJECT_ID}/openarena:0.8.8 .
```

- GCRにプッシュ

```
gcloud docker -- push \
  ${GCR_REGION}.gcr.io/${PROJECT_ID}/openarena:0.8.8
```

### ディスクの作成

通常、ゲームのバイナリは小さくて、アセットが大きくなるらしい
なので、アセットとバイナリは分離して、バイナリのみコンテナイメージを作成することは合理的といえる
アセットは永続ディスクに保存して、DGSコンテナを実行するVMに接続する
アセットを全サーバに配布する手間が省けるうえ、コストのメリットもある

```
region=asia-northeast1
zone_1=${region}-b
gcloud config set compute/region ${region}

// Diskの設定用のとりあえずインスタンスを作成
gcloud compute instances create openarena-asset-builder \
--machine-type f1-micro \
--image-family debian-9 \
--image-project debian-cloud \
--zone ${zone_1}

// 永続ディスク作成
gcloud compute disks create openarena-assets \
--size=50GB --type=pd-ssd \
--description="OpenArena data disk. Mount read-only at /usr/share/games/openarena/baseoa/" \
--zone ${zone_1}

// アタッチ
gcloud compute instances attach-disk openarena-asset-builder \
--disk openarena-assets --zone ${zone_1}
```

- SSHして設定

```
// まずマウントしないと使えない
sudo lsblk

sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
sudo mkdir -p /usr/share/games/openarena/baseoa/
sudo mount -o discard,defaults /dev/sdb /usr/share/games/openarena/baseoa/

// サーバインストール
sudo apt-get update
sudo apt-get -y install openarena-server

// ゲームアセットをコピーしてくる
sudo gsutil cp gs://qwiklabs-assets/single-match.cfg /usr/share/games/openarena/baseoa/single-match.cfg

// 終わったらマウント解除して終了
sudo umount -f -l /user/share/games/openarena/baseoa/
sudo shutdown -h now
```

- メインのほうでVM削除

```
echo $zone_1

gcloud compute instances delete openarena-asset-builder --zone ${zone_1}
```

### Kuernetes クラスタ準備

- n1-standardに作る
- 本番用途ではn1-highcpuが適してるかも
  - 同時実行予定のDGSポッド数を増やせる
  - ノード数に限りがあるので、上限を増やせる
- クラスタ内のリソースを増減する場合は、インスタンス単位で増減するのが簡単
- 今回はvCPUは2で十分

- ネットワーク、firewallルールを作成

```
gcloud compute networks create game
gcloud compute firewall-rules create openarena-dgs --network game \
--allow udp:27961-28061
```

- クラスタ作成

```
gcloud container clusters create openarena-cluster \
--num-nodes 3 \
--network game \
--machine-type=n1-standard-2 \
--zone=${zone_1}
```

- 認証情報設定

```
gcloud container clusters get-credentials openarena-cluster --zone ${zone_1}
```

- メモ

今回のラボでは--disable-addons フラグでLoadBalancing機能とかが無効になってる
本番を作るときはこの辺も意識する必要があるらしい
[インスタンスグループ](https://cloud.google.com/compute/docs/instance-groups/)

- 永続ディスクを構成

一般的なDGSではゲームアセットに対する書き込みは不要
→バグりそうやし確かに
なので、ポッドごとに読み取り専用のアセットを含む永続ディスクをマウントする

ボリュームをKubernetesポッドに含めるには、
PersistentVolumeClaimリソースが必要
PersistentVolumeClaimにはPersistentVolumeClassがいるので、
両方適用する
これはラボやから用意してくれてるけど、本来は自分で作る もしくは配布されてるのを持ってくる

```
kubectl apply -f k8s/asset-volume.yaml
kubectl apply -f k8s/asset-volumeclaim.yaml

kubectl get persistentVolume
kubectl get persistentVolumeClaim
```

- Pod構成

OpenArenaでは1ゲームに制限時間があるので、時間終了とともにコンテナも終了する
これをもとにしてスケーリングポリシーを作れる
→マイクラやとどうするか、、、特にスケーリングはええか

- ゲームサーバのプロセス管理について

ベストプラクティスではできる限りDGSはマッチメーカーやスケーリングマネージャと直接通信を行わず、
KubernetesAPIに状態を公開してAPI経由で処理を行うようにする
外部クライアントも、直接サーバにクエリを送信せず、KubernetesエンドポイントからDGSの状態を読みとる


### スケーリングマネージャ設定

スケーリングマネージャはシンプルに、現在にDGS負荷に基づいて仮想マシン数をスケーリングする
内容は用意されてるのをもらって使う

```
export GCR_REGION=asia-northeast1
export PROJECT_ID=PROJECT_ID

cd ../scaling-manager
chmod +x build-and-push.sh
source ./build-and-push.sh
```