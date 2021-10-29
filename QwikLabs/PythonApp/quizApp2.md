# Cloud Storage と連携する - Quiz App

## クイズアプリの画像をアップする機能で、保存先をCloudStorageにする

DjangoではAWS S3に画像を保存する方法をかじったけど、
今回のやつをやればGCP版を輸入できる・・・！？

前回時間オーバーで最後の確認ができなかったので今回からはやはり
一回目を通して内容メモしてから、動作確認がてらやる感じで

## 手順

### 1. 準備

- リポジトリクローン

```
$ git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

- prepare_environment.sh実行

```
$ cd training-data-analyst/courses/developingapps/python/cloudstorage/start
$ . prepare_environment.sh
```

- サーバ起動

```
$ python run_server.py
```

- コード確認

同じPythonフレームワークやからFlaskが読める・・・！読めるぞ！
Django極めよう・・・

### 2. バケット作成

```
gsutil mb gs://$DEVSHELL_PROJECT_ID-media
export GCLOUD_BUCKET=$DEVSHELL_PROJECT_ID-media
echoe $GCLOUD_BUCKET
```

### 3. バケットに画像を保存するためのコードを作る

- モジュールインポート
- クライアント作成
- バケット参照を作成

```py
from google.cloud import storage

bucket_name = os.getenv('GCLOUD_BUCKET')
storage_client = storage.Client()
bucket = storage_client.get_bucket(bucket_name)
```


### 4. ファイルを送信するためのコードを作る

- バケット内のblobオブジェクトへの参照を取得する
- blobオブジェクトを使って画像をアップロードする
- ファイルを公開する
- blobの公開URLを返す

```py
    blob = bucket.blob(image_file.filename)
    blob.upload_from_storing(
        image_file.read(),
        content_type=image_file.content_type
    )
    if public:
        blob.make_public()
    return blob.public_url
```

### 5. Storageの機能を使うためのコードを作る

- storageクライアントをインポート
- upload_file関数にファイルをアップして、公開URLを変数に代入する
- 公開URLをreturnで返す
- save_question関数にimage_fileが存在するかを確認するifテストを作る
  - 存在する場合 : upload_file関数を読んでエンティティimageUrlにURLを割り当てる
  - 存在しない場合 : imageUrlエンティティに空の文字列を割り当てる

```py
def upload_file(image_file, public):
    if not image_file:
        return None

    # TODO: Use the storage client to Upload the file
    # The second argument is a boolean

    public_url = storage.upload_file(
        image_file,
        public
    )

    # END TODO

    # TODO: Return the public URL
    # for the object

    return public_url

    # END TODO

"""
uploads file into google cloud storage
- call method to upload file (public=true)
- call datastore helper method to save question
"""
def save_question(data, image_file):

    # TODO: If there is an image file, then upload it
    # And assign the result to a new Datastore property imageUrl
    # If there isn't, assign an empty string
    
    if image_file:
        date['iamgeUrl'] = str(
            upload_file(image_file, True)
        )
    else:
        data['imageUrl'] = u''

    # END TODO

    data['correctAnswer'] = int(data['correctAnswer'])
    datastore.save_question(data)
    return
```

### 6. アプリを実行してStorageオブジェクトが作成されるか確認

確認できた、
URL末尾に /api/quizzes/gcp を追加してアクセスすると、
追加した質問のJSONが出力もされた

ただ、実際の質問では画像は表示されなかったから意味不明な質問になった、、、
まぁいいや

## 感想

ほぼDjangoでS3保存やったのとおなじかんじでできた！
これはいいラボや