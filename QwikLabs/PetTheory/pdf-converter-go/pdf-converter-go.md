# Go と Cloud Run を使用したPDFの作成メモ

## 概要
PDFコンバータのGo言語版
1回目のやつ(pdf-converter)と3回目のやつ(REST_API)の合わせた復習って感じ

## 手順

### ソースコードを持ってくる

```sh
$ git clone https://github.com/Deleplace/pet-theory.git
$ cd pet-theory/lab03
```

### プログラム作成

```go
package main

import (
      "fmt"
      "io/ioutil"
      "log"
      "net/http"
      "os"
      "os/exec"
      "regexp"
      "strings"
)

func main() {
      http.HandleFunc("/", process)

      port := os.Getenv("PORT")
      if port == "" {
              port = "8080"
              log.Printf("Defaulting to port %s", port)
      }

      log.Printf("Listening on port %s", port)
      err := http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
      log.Fatal(err)
}

func process(w http.ResponseWriter, r *http.Request) {
      log.Println("Serving request")

      if r.Method == "GET" {
              fmt.Fprintln(w, "Ready to process POST requests from Cloud Storage trigger")
              return
      }

      //
      // GCS オブジェクト メタデータを含むリクエスト本文を読み取ります
      //
      gcsInputFile, err1 := readBody(r)
      if err1 != nil {
              log.Printf("Error reading POST data: %v", err1)
              w.WriteHeader(http.StatusBadRequest)
              fmt.Fprintf(w, "Problem with POST data: %v \n", err1)
              return
      }

      //
      // 作業ディレクトリ（同時実行セーフ）
      //
      localDir, errDir := ioutil.TempDir("", "")
      if errDir != nil {
              log.Printf("Error creating local temp dir: %v", errDir)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Could not create a temp directory on server. \n")
              return
      }
      defer os.RemoveAll(localDir)

      //
      // GCS から入力ファイルをダウンロードします
      //
      localInputFile, err2 := download(gcsInputFile, localDir)
      if err2 != nil {
              log.Printf("Error downloading GCS file [%s] from bucket [%s]: %v",
gcsInputFile.Name, gcsInputFile.Bucket, err2)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error downloading GCS file [%s] from bucket [%s]",
gcsInputFile.Name, gcsInputFile.Bucket)
              return
      }

      //
      // LibreOffice を使ってローカルの入力ファイルをローカルの PDF ファイルに変換します。
      //
      localPDFFilePath, err3 := convertToPDF(localInputFile.Name(), localDir)
      if err3 != nil {
              log.Printf("Error converting to PDF: %v", err3)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error converting to PDF.")
              return
      }

      //
      // 生成したばかりの PDF を GCS にアップロードします
      //
      targetBucket := os.Getenv("PDF_BUCKET")
      err4 := upload(localPDFFilePath, targetBucket)
      if err4 != nil {
              log.Printf("Error uploading PDF file to bucket [%s]: %v", targetBucket, err4)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error downloading GCS file [%s] from bucket [%s]",
gcsInputFile.Name, gcsInputFile.Bucket)
              return
      }

      //
      // 元の入力ファイルを GCS から削除します。
      //
      err5 := deleteGCSFile(gcsInputFile.Bucket, gcsInputFile.Name)
      if err5 != nil {
              log.Printf("Error deleting file [%s] from bucket [%s]: %v", gcsInputFile.Name,
gcsInputFile.Bucket, err5)
         // これはブロッキング エラーではありません。
         // PDF は正常に生成され、アップロードされました。
      }

      log.Println("Successfully produced PDF")
      fmt.Fprintln(w, "Successfully produced PDF")
}

func convertToPDF(localFilePath string, localDir string) (resultFilePath string, err error) {
      log.Printf("Converting [%s] to PDF", localFilePath)
      cmd := exec.Command("libreoffice", "--headless", "--convert-to", "pdf",
              "--outdir", localDir,
              localFilePath)
      cmd.Stdout, cmd.Stderr = os.Stdout, os.Stderr
      log.Println(cmd)
      err = cmd.Run()
      if err != nil {
              return "", err
      }

      pdfFilePath := regexp.MustCompile(`\.\w+$`).ReplaceAllString(localFilePath, ".pdf")
      if !strings.HasSuffix(pdfFilePath, ".pdf") {
              pdfFilePath += ".pdf"
      }
      log.Printf("Converted %s to %s", localFilePath, pdfFilePath)
      return pdfFilePath, nil
}
```

### ビルド & デプロイ

#### Goのビルド
```sh
$ go build -o server
```
↑これでGoによるプログラミングが、serverって名前のバイナリファイルになる

#### Dockerfile作成
```Dockerfile
FROM debian:buster
RUN apt-get update -y \
  && apt-get install -y libreoffice \
  && apt-get clean
WORKDIR /usr/src/app
COPY server .
CMD [ "./server" ]
```
Dockerコンテナとしてコンテナ化できるようDockerfileを作る  
その時、Goのアプリと、Libreofficeを同時にコンテナ化するので、  
apt-getでLibreofficeをインストールするコマンドを書いておく

#### Cloud Buildでビルド
```sh
$ gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```
これでDockerコンテナがGCRに保存される

#### Cloud Run でデプロイする
```sh
$ gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-central1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed
```
- 内容
  - pfd-converterっていう名前でRunのデプロイするよ
  - イメージはgcr.io/~~ を使うよ
  - プラットフォームはmanageのものをつかうよ
  - リージョンはus-central1だよ
  - メモリを2G使うようにするよ
  - 認可されていないものは拒否するよ
  - 環境変数を設定するよ[PFD_BUCKET=$GOOGLE_CLOUD_PROJECT-processed]

これでRunのアプリが完成する

### サービスアカウント作成

このままでは通信許可がないので、  
RunのアプリがPub/Subを通してGCSからデータを持ってきたり、GCSにデータを保存したりできない  
→許可ポリシーを持つサービスアカウントを作成してそれをアタッチする

#### Pub/Sub通知作成
```sh
$ gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload
```
gsutilコマンドで、uploadバケットにファイルが保存されたら、new-docというラベルがついたPub/Sub通知を作成する

#### サービストリガー役のサービスアカウント作成
```sh
$ gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```
これは単純にサービスアカウントを作っただけ

#### 権限付与
```sh
$ gcloud run services add-iam-policy-binding pdf-converter \
  --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
  --role=roles/run.invoker \
  --region us-central1 \
  --platform managed
```
- 内容
  - CloudRunでデプロイされてるpdf-converterにiamポリシーバインディングを付与するよ
  - メンバーにはサービスアカウント指定するよ
  - ロールはRunの実行者だよ
  - リージョンはus-central1だよ
  - プラットフォームはマネージのものだよ

おそらくこのままだとPub/Sub認証トークンってのが作成できなくて、Pub/Subが利用できない

#### Pub/Sub認証トークンを作成できるようにする
```sh
$ PROJECT_NUMBER=$(gcloud projects list \
  --format="value(PROJECT_NUMBER)" \
  --filter="$GOOGLE_CLOUD_PROJECT")

$ gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gsericeaccount.com \
  --role=roles/iam.serviceAccountTokenCreator
```

### アプリケーションテスト
```sh
$ SERVICE_URL=$(gcloud run services describe pdf-converter \
  --platform managed \
  --region us-central1 \
  --format "value(status.url)")

$ echo $SERVICE_URL

$ curl -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

これでファイルをPDFに変換するRunアプリのURL(エンドポイント)が作成される  
まだ、GCSにアップロードされても通知が作成されるだけでアプリに変換前データを送れないので、  
そのためのトリガーを作成する

### Cloud Storage 側にトリガーを作成する
```sh
$ gcloud pubsub subscriptions create pdf-conv-sub \
  --topic new-doc
  --push-endpoint=$SERVICE_URL \
  --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```
- 内容
  - pubsubの機能で、pdf-conv-subっていう名前のサブスクライバーを作成するよ
  - new-docっていうトピックを持っている通知が来たら作動するよ
  - プッシュ先のエンドポイントは$SERVICE_URLだよ
  - プッシュするときに利用するサービスアカウントはpubsub-cloud~~だよ

### アップロードテスト
```sh
$ gsutil -m cp -r gs://spls/gsp762/* gs://$GOOGLE_CLOUD_PROJECT-upload
```

実行後即uploadのバケットを見ると未処理のものが保存されていく  
更新を押すと、どんどん処理されて行って削除されていく  
processedのほうを見るとPDFに変換されたファイルが保存されている