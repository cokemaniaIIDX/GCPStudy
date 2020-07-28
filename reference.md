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
- 非マネージドインスタンスグループ
  - 異種インスタンスをまとめる
  - スケーリングとか自動修復はできない
  - 同一ゾーン内のみしか作れない
