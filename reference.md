# GCPドキュメントのまとめ

## Computing

### Compute Engine
- ストレージオプション
  - ゾーン永続HDD,SSD
  - リージョン永続HDD,SSD
    - リージョン内の2つのゾーンに同期レプリケーションされる
  - ローカルSSD
    - 一番スループットが高い
    - VM削除と同時に削除される想定のため、暗号化に対応していない
    - インスタンスあたりの最大容量は3TB
  - CloudStorage
  - Filestore

### App Engine
- Standard
  - 中でサンドボックス(コンテナ？)が動いてる
  - 0インスタンスにスケールできる
  - 最後の処理から15分後、もしくはシャットダウンした15分後に課金が止まる
  - マシンタイプがある(Bシリーズ,Fシリーズ)
  - 使える言語
    - Python
    - Java
    - Node.js
    - PHP
    - Ruby
    - Go

- Frexible
  - 中でComputeEngineインスタンスが動いてる
    - つまり、SSHできる
  - CPU,メモリは任意で決めれる
  - 使える言語
    - Standard＋.NET
    - 独自のDockerイメージとかも使える

### Kubernetes Engine
- Serviceの種類
  - ClusterIP
    - 内部クライアントが内部の固定IPアドレスにリクエストを送信する
    - クラスタ内のクライアントが、特定のPodにアクセスする場合
  - NodePort
    - クライアントはServiceが指定した１つ以上のnodePort値を使用して、ノードのIPにリクエストを送信する
    - ClusterIPの拡張版
    - 外部クライアントは、クラスタノードの外部IPの、nodePortのポートでServiceを呼び出す
  - LoadBalancer
    - クライアントがネットワークロードバランサのIPにリクエストを送信する
    - NodePortの拡張版
    - 外部クライアントは、LBの外部IPでServiceを呼び出す
    - ポートはPorts:portの値
  - ExternalName
    - 内部クライアントが、外部DNS名のエイリアスとして、ServiceのDNS名を使用する
    - 内部クライアントがmy-xn-service.default.svc.cluster.localにアクセスすると、example.comにリダイレクトされる
    - ExternalNameタイプは特異で、一連のPodに関連付けられてないし固定IPがない
    - 内部DNS名から外部DNS名のマッピングに使われる
  - Headless
    - Podのグループ化を行うが、固定IPは必要ないときに使う


## Storage

### Cloud Storage
- 可用性
  - Standard
    - Multi(Dual)-Regionで99.99%を超える
    - Regionalで99.99%
  - そのほか(Nearline,Coldline,Archive)
    - Multi(Dual)-Regionで99.95%
    - Regionalで99.90%

- すべてのクラス
  - 無制限ストレージ
  - 耐久性 99.999999999%
  - 地理的な冗長性

### Datastore
- コンポーネント
  - 種類
    - RDBMSでいうテーブル
  - エンティティ
    - 行
  - プロパティ
    - 列
  - キー
    - 主キー


## 負荷分散
- マネージドインスタンスグループ
  - 自動修復、負荷分散、自動スケーリング、自動更新、ステートフルなどの機能をサポート
  - 単一ゾーンまたはリージョンで作成できる

- 非マネージドインスタンスグループ
  - 異種インスタンスをまとめる
  - スケーリングとか自動修復はできない
  - 同一ゾーン内のみしか作れない


## Serverless Computing

### Cloud Functions
- Cloud Functionsの料金は、関数の実行期間、関数の呼び出し回数、関数に対してプロビジョニングされたリソースの数に応じて決まる
  - 呼び出し回数
    - 最初の200万回は無料
    - 以降、100万単位で$0.4
  - 実行期間
    - 関数がリクエストを受け取ってから完了するまでの期間
    - 100ミリ秒単位で測定される
    - メモリとCPUの使用量でも変動する
      - メモリ:512MB,CPU:800MHz → $0.000000925　等


## セキュリティ アイデンティティ

### Cloud IAM 
- メンバー
  - Googleアカウント
  - サービスアカウント
  - Googleグループ
  - G Suite または Cloud Identity ドメイン


## ハイブリッド

### Interconnect

### Peering
- Googleのエッジネットワークと接続する
- エッジロケーションは現在142箇所　東京7箇所、大阪2箇所
- ダイレクトピアリングはGoogleCloudの外部にある
  - →よって、G Suiteへアクセスする必要がなければInterconnectを使う

## その他
- GoogleCloudのプロジェクトやリソースの管理方法
  - Console
  - Comand Line Interface(gcloud)
    - Cloud SDK
    - Cloud Shell
  - クライアントライブラリ(API)
    - Node.js や Python から呼び出す
  - Cloud Console モバイルアプリ
  - Cloud tools for PowerShell
    - Windowsサーバ用