# REST API のメモ

## 概要

PetTheoryで、顧客の個人情報データを非公開のまま、保険会社が顧客データにアクセスできるようにしたい
RESTAPIを使って接続する
顧客データはFirestoreに保存する

Go言語を利用する

## 手順
時間が無くなってまうので、要所だけ抽出

### アプリのビルド

#### main.go 作成
```go
package main

import (
  "fmt"
  "log"
  "net/http"
  "os"
)

func main() {
  port := os.Getenv("PORT")
  if port == "" {
      port = "8080"
  }
  http.HandleFunc("/v1/", func(w http.ResponseWriter, r *http.Request) {
      fmt.Fprintf(w, "{status: 'running'}")
  })
  log.Println("Pets REST API listening on port", port)
  if err := http.ListenAndServe(":"+port, nil); err != nil {
      log.Fatalf("Error launching Pets REST API server: %v", err)
  }
}
```

#### Dockerfile作成
```Dockerfile
FROM gcr.io/distroless/base-debian10
WORKDIR /usr/src/app
COPY server .
CMD [ "/usr/src/app/server" ]
```

#### ビルド & デプロイ
```sh
$ go build -o server

$ ls -la

gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1
→Dockerfileとかバイナリserverとかを参照して、コンテナイメージをGCRに保存

gcloud beta run deploy rest-api \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1 \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
→Cloud Run で コンテナを利用可能な状態にする
```

#### ブラウザで確認
https://rest-api-dvmk4q25ca-uc.a.run.app/v1/  
{status: 'running'}  
が表示されたらOK

これにて、RESTAPIが完成となる

### firestore作成

1. コンソールで作成
2. プロジェクトのGCSにデータを移行
    ```
    gsutil cp -r gs://spls/gsp645/2019-10-06T20:10:37_43617 gs://$GOOGLE_CLOUD_PROJECT-customer
    ```
3. GCSからFirebaseに読み込む
    ```
    gcloud beta firestore import gs://$GOOGLE_CLOUD_PROJECT-customer/2019-10-06T20:10:37_43617/
    ```
    ↑アクセス拒否された storage.get権限がない
        →とりあえずバケットを公開にした

### REST API の構成
```
https://rest-api-[hash].a.run.app/v1/customer/22530
```
こんな感じで、顧客IDをURLに入れると、
```json
{
  "status": "success",
  "data": {
    "proposed": 1602,
    "approved": 585,
    "rejected": 489
  }
}
```
その顧客の治療費がJSONで返ってくるようにする

#### プログラム改修

↑の機能を実装するのに、
- Firestoreにアクセスするパッケージと、
- クロスオリジンリソースシェアリング(CORS)を処理するパッケージ
が必要らしい

#### main.go 修正
```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"

	"cloud.google.com/go/firestore"
	"github.com/gorilla/handlers"
	"github.com/gorilla/mux"
	"google.golang.org/api/iterator"
)

var client *firestore.Client

func main() {
	var err error
	ctx := context.Background()
	client, err = firestore.NewClient(ctx, "PROJECT_ID") // ←実際のProjectIDを入力する
	if err != nil {
	log.Fatalf("Error initializing Cloud Firestore client: %v", err)
	}

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	r := mux.NewRouter()
	r.HandleFunc("/v1/", rootHandler)
	r.HandleFunc("/v1/customer/{id}", customerHandler)

	log.Println("Pets REST API listening on port", port)
	cors := handlers.CORS(
		handlers.AllowedHeaders([]string{"X-Requested-With", "Authorization", "Origin"}),
		handlers.AllowedOrigins([]string{"https://storage.googleapis.com"}),
		handlers.AllowedMethods([]string{"GET", "HEAD", "POST", "OPTIONS", "PATCH", "CONNECT"}),
	)
	if err := http.ListenAndServe(":"+port, cors(r)); err != nil {
		log.Fatalf("Error launching Pets REST API server: %v", err)
	}
}

// ハンドラサポート
func rootHandler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "{status: 'running'}")
}

func customerHandler(w http.ResponseWriter, r *http.Request) {
  id := mux.Vars(r)["id"]
  ctx := context.Background()
  customer, err := getCustomer(ctx, id)
  if err != nil {
    w.WriteHeader(http.StatusInternalServerError)
    fmt.Fprintf(w, `{"status": "fail", "data": '%s'}`, err)
    return
  }
  if customer == nil {
    w.WriteHeader(http.StatusNotFound)
    msg := fmt.Sprintf("`Customer \"%s\" not found`", id)
    fmt.Fprintf(w, fmt.Sprintf(`{"status": "fail", "data": {"title": %s}}`, msg))
    return
  }
  amount, err := getAmounts(ctx, customer)
  if err != nil {
    w.WriteHeader(http.StatusInternalServerError)
    fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
    return
  }
  data, err := json.Marshal(amount)
  if err != nil {
    w.WriteHeader(http.StatusInternalServerError)
    fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
    return
  }
  fmt.Fprintf(w, fmt.Sprintf(`{"status": "success", "data": %s}`, data))
}

// カスタマーサポート
type Customer struct {
  Email string `firestore:"email"`
  ID    string `firestore:"id"`
  Name  string `firestore:"name"`
  Phone string `firestore:"phone"`
}

func getCustomer(ctx context.Context, id string) (*Customer, error) {
  query := client.Collection("customers").Where("id", "==", id)
  iter := query.Documents(ctx)

  var c Customer
  for {
    doc, err := iter.Next()
    if err == iterator.Done {
	break
    }
    if err != nil {
	return nil, err
    }
    err = doc.DataTo(&c)
    if err != nil {
	return nil, err
    }
  }
  return &c, nil
}

func getAmounts(ctx context.Context, c *Customer) (map[string]int64, error) {
  if c == nil {
    return map[string]int64{}, fmt.Errorf("Customer should be non-nil: %v", c)
  }
  result := map[string]int64{
    "proposed": 0,
    "approved": 0,
    "rejected": 0,
  }
  query := client.Collection(fmt.Sprintf("customers/%s/treatments", c.Email))
  if query == nil {
    return map[string]int64{}, fmt.Errorf("Query is nil: %v", c)
  }
  iter := query.Documents(ctx)
  for {
    doc, err := iter.Next()
    if err == iterator.Done {
	break
    }
    if err != nil {
	return nil, err
    }
    treatment := doc.Data()
    result[treatment["status"].(string)] += treatment["cost"].(int64)
  }
  return result, nil
}

```

#### ビルド & デプロイ
```sh
go build -o server

gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.2

gcloud beta run deploy rest-api \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.2 \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

### テスト
URLにいろいろIDを入れてみる  
22530 の代わりに次のような別の顧客 ID を URL に入力してみましょう。

- 34216
- 70156（金額はすべてゼロになります）
- 12345（顧客やペットは存在しないため、Query is nil のようなエラーが返されます）

これにて、顧客IDを指定してURLをPOSTすると、Firestoreに保存されている顧客データ(PII)(JSON)を、
公開しない状態で、持ってくることができるRESTAPIの構築が完了！