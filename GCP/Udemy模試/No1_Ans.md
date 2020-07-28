# Udemy模試
## 解答、解説

1. 次のうち、Datastoreの特徴として間違っているものは？

   - [ ] Key/Valueペア ルックアップ
   - [ ] SQLライクのクエリ
   - [ ] 複数行トランザクション
   - [x] HBaseファイルシステム
     - HBaseは高スループットのワイドカラム型NoSQL
       - Bigtableの説明
   - [x] リレーショナルデータストレージ
     - Datastoreはドキュメント型NoSQL
   - [ ] ゾーン間データレプリケーション

2. Cloud SQLで提供されるバックアップの種類はどれ？

   - [ ] アーカイブバックアップ
   - [x] 自動バックアップ
   - [x] オンデマンドバックアップ
   - [ ] プリエンプティブルバックアップ
   - [ ] スケジュールバックアップ
   - [ ] フェイルオーバーバックアップ

3. GCPサービスのうち、HL7v2やFHIR、DICOMのようなデータフォーマットを利用するためのデータ統合APIはどれ？

   - [ ] Cloud Claim Endpoints
   - [ ] Cloud Medicare API
   - [ ] Cloud Insurance API
   - [ ] Cloud Health Check
   - [ ] Cloud Claim API
   - [x] Cloud Healthcare API
     - HL7v2,FHIE,DICOM:医療データの規格

4.  Googleのハイブリッドソリューションのうち、
    GCPリソースにアクセスする必要がなく、プロバイダーのサービスを利用して、
    オンプレネットワークとGoogleのエッジネットワークを高スループットで接続できるサービスは？

    - [ ] 共有VPC
    - [x] キャリアピアリング
      - GSuitにつなぐ場合(GCP不要)は、ピアリングを使う
      - プロバイダーを利用するので、キャリアピアリングが正解
    - [ ] ダイレクトピアリング
    - [ ] VPCネットワークピアリング
    - [ ] Dedicated Interconnect
    - [ ] Partner Interconnect

5.  IAMロールの説明として適切なものはどれ？

    - [ ] ロールは、有効なポリシーの集まり
    - [ ] ロールは、GCPには存在しない
    - [x] ロールは、権限の集まり
      - compute.vm.editor とか adminとかの権限の集まりがロール
    - [ ] ロールは、リソースの集まり
    - [ ] ロールは、ポリシーの集まり
    - [ ] ロールは、1つのポリシーのみ含む

6.  Resource Sync や Discoveryを提供するGKEの特徴は？

    - [ ] Cluster Search
    - [x] Multi-Cluster Federation
      - [Kubernetes - Reference](https://kubernetes.io/blog/2018/12/12/kubernetes-federation-evolution/)
    - [ ] TCP/IP
    - [ ] Network Relay
    - [ ] Kubernetes Gateway Protocol
    - [ ] Kube Federation

7.  TPMの説明として最も適切なものはどれ？

    - [ ] TPMとは、ハードウェアレベルで発生するすべてのシステムイベントを保存するハードウェア、ファームウェア、もしくは仮想デバイスのことである。
    - [ ] TPMは、CPUによって効率的な処理が行われるよう、データをRAMにロードされる前に暗号化する。
    - [x] TPMとは、コンピュータのセキュリティを補助するハードウェア、ファームウェア、または仮想デバイスのことであり、高度に特権化された管理者に対して、安全性を提供できるものである。
    - [ ] TPMは、ファームウェアレベルでroot鍵を作成する。

8.  静的・動的両方のルーティングができ、1つのSecureトンネルの両方をサポートするGCPサービスは？

    - [ ] High Availability VPN
    - [ ] Premium Direct Peering
    - [x] Classic VPN
      - Classic VPNのみ静的ルーティングに対応している
    - [ ] HA Direct Peering
    - [ ] Classic Direct Peering
    - [ ] Premium VPN

9.  Bigtableインスタンスは同一ゾーンに複数のクラスタを作成できるか？

    - [ ] yes
    - [x] no
      - Bigtableクラスタは1ゾーンにつき1つ
      - 例えば、asia-northeast1に2つ作るなら、
        - asia-northeast1-aに1つ
        - asia-northeast1-bに1つ
      - とかになる

10. Cloud Storageで1回にアップロードできるオブジェクトのサイズの上限は?

    - [x] 5TB
    - [ ] 1TB
    - [ ] 10TB
    - [ ] 100GB
    - [ ] 1GB
    - [ ] 64TB

11. Cloud Storageでオブジェクトをアップロードするときに指定できるメタデータはどれ？(3つ)

    - [ ] Min-Age
    - [x] Content-Encoding
    - [x] Max-Age
    - [x] Cache-Control
    - [ ] Cache-rule
    - [ ] Cache-Authority

    - ほかにもContent-Disposition,Custom Metadataがある

12. 地理的冗長性について正しい記述はどれ？(2つ)

    - [x] 地理的に冗長なデータは、最低100マイル離れた、少なくとも2つ以上の場所で保存される。
    - [ ] 地理的に冗長なデータは、25～100マイル離れた、少なくとも2つ以上の場所で保存される。
    - [ ] 地理的な冗長性は、同期的に行われる
    - [ ] 地理的に冗長なデータは、少なくとも2つ以上の大陸で保存される。
    - [x] 地理的な冗長性は、非同期的に行われる。
    - [ ] 地理的な冗長性は、半同期的に行われる。

13. GKEのクラスターで設定できる地理的オプションはどれ？(3つ)

    - [x] 単一ゾーン
    - [ ] マルチリージョナル
    - [x] マルチゾ－ン
    - [x] リージョナル
    - [ ] マルチグローバル
    - [ ] グローバル

14. GCEで利用できる支払方法はどれ？(3つ)

    - [x] On Demand
    - [ ] Available
    - [ ] Standard
    - [x] 1 or 3 Year Commit (確約利用割引)
    - [x] Preemptible
    - [ ] Interruptible

15. Cold Line の Cloud Storage は、データの取り出しに何秒かかるか？

    - [ ] 数秒
    - [x] 数ミリ秒
    - [ ] 数時間
    - [ ] 6時間
    - [ ] 数マイクロ秒
    - [ ] 数分

16. 特定のBigtableクラスタにはユニークテーブルやユニークガベージコレクションが備わっている。

    - [ ] yes
    - [x] no