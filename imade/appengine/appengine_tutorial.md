# AppEngineのチュートリアルをやってみる

[手順](https://cloud.google.com/appengine/docs/standard/python3/building-app?hl=ja)

## 1. AppEngineを有効化

- terraform

```s:app_engine.tf
#============#
# App Engine #
#============#

resource "google_app_engine_application" "imade-gcp" {
  project = var.project
  location_id = "asia-northeast1"
}
```

インスタンスのスペックとかは`app.yaml`で指定するので、
terraformはこれだけ

## 2. webサービスの作成

### 始める前に

- Datastoreでテストを行うために認証情報設定

```
$ gcloud auth application-default login
```

↑これは多分Datastoreへのアクセス権限を使いたいってことやから、
GCEのインスタンスサービスアカウントにDatastore権限を付与すればいいということやと思う
今回はスキップする

### ファイルのコピペ

- ファイル構造

building-an-app/
  app.yaml
  main.py
  requirements.txt
  static/
    script.js
    style.css
  templates/
    index.html

↑の通りファイルを作成する

それぞれ中身は[手順ページ参照](https://cloud.google.com/appengine/docs/standard/python3/building-app/writing-web-service?hl=ja#writing_your_web_service)

### WEBサービスのテスト

Flaskアプリが正常に動作するかテスト

```
$ python -m venv env
$ source env/bin/activate

$ cd building-an-app
$ pip install -r requirements.txt

$ python main.py

→http://localhost:8080にアクセス
```

## 3. AppEngineデプロイ用ファイル作成 app.yaml

```yaml:app.yaml
runtime: python39

handlers:
- url: /static
  static_dir: static
- url: /.*
  script: auto
```

ランタイムと、パスによってディレクトリを指定するルールのみ記載

インスタンスクラスは記述ナシの場合デフォルトの`F1`になるみたい
F1は基本無料なので安心


## 4. デプロイ

```
$ gcloud app deploy

Services to deploy:

descriptor:                  [/home/coke/GCPStudy/imade/appengine/building-an-app/app.yaml]
source:                      [/home/coke/GCPStudy/imade/appengine/building-an-app]
target project:              [imade-gaming-265014]
target service:              [default]
target version:              [20220330t144855]
target url:                  [https://imade-gaming-265014.an.r.appspot.com]
target service account:      [App Engine default service account]


Do you want to continue (Y/n)?  Y

Beginning deployment of service [default]...
Created .gcloudignore file. See `gcloud topic gcloudignore` for details.
╔═══════════════════════════════╗
╠═ Uploading 7 files to Google Cloud Storage                ═╣
╚═══════════════════════════════╝
File upload done.
Updating service [default]...done.                                                                                                                                             
Setting traffic split for service [default]...done.                                                                                                                            
Deployed service [default] to [https://imade-gaming-265014.an.r.appspot.com]

You can stream logs from the command line by running:
  $ gcloud app logs tail -s default

To view your application in the web browser run:
  $ gcloud app browse
```

app.yamlへのパスの確認が出てきてありがたい
なるほど静的ファイルは自動的にGCSに保存されるのか

最初のデプロイは`default`サービスに作成される必要があるので、サービス名はdefault
→そのあとのデプロイではapp.yamlでサービス名を指定できるらしい

### サービス確認

```
$ gcloud app browse

https://imade-gaming-265014.an.r.appspot.com
```

カスタムドメインを使いたい場合はまた[それ用の記事](https://cloud.google.com/appengine/docs/standard/python3/mapping-custom-domains?hl=ja)をみる

## 5. データ処理機能

Datastoreを使って動的なサイトにする

```py:main.py
from google.cloud import datastore

datastore_client = datastore.Client()

def store_time(dt):
    entity = datastore.Entity(key=datastore_client.key('visit'))
    entity.update({
        'timestamp': dt
    })

    datastore_client.put(entity)

def fetch_times(limit):
    query = datastore_client.query(kind='visit')
    query.order = ['-timestamp']

    times = query.fetch(limit=limit)

    return times
```

* def store_time: datetimeを引数にとって、アクセスされた時間をDBの`visit`kind に`timestamp`ドキュメントとして保存
* def fetch_times: limit(=後で10回に設定)を引数にとって、`visit`kindの中身を timestampの降順にクエリをセットし、limit分fetch(=上限設定？)して返す

- rootを修正する

```py:main.py
@app.route('/')
def root():
    UTC = datetime.datetime.now(tz=datetime.timezone.utc)
    JST = datetime.timezone(datetime.timedelta(hours=+9), 'JST')
    dt = UTC.astimezone(JST)
    store_time(dt)

    times = fetch_times(10)

    """ dummy_times = [datetime.datetime(2018, 1, 1, 10, 0, 0),
                   datetime.datetime(2018, 1, 2, 10, 30, 0),
                   datetime.datetime(2018, 1, 3, 11, 0, 0),
                   ] """

    return render_template('index.html', times=times)
```

例はUTCやけどJSTを頑張って取得してきた
これを保存してtemplateに返すようにする

- index

```html:index.html
  <h2>Last 10 visits</h2>
  {% for time in times %}
    <p>{{ time['timestamp'] }}</p>
  {% endfor %}
```

timesはDatastoreのクエリが入ってる？っぽいので、その中のtimestampを出力するように指定

- requirements.txt 追加

```txt:requirements.txt
Flask==2.0.2
google-cloud-datastore==2.4.0
```

### 確認

```
$ pip install -r requirements.txt

$ python main.py

$ http://localhost:8080
→JSTになってない！！怒 もうええわ
```

### デプロイ

```
$ gcloud app deploy
```