# Pub/Sub , NL , Spanner との連携 - Quiz App

Datastore,Storage,Firebaseに続いて、
バックエンドとの統合を勉強する
Pub/Sub, Natural Language, Spanner と連携するらしい

## 目標

- メッセージを作成してPub/Subトピックにパブリッシュする
- アプリでトピックにサブスクライブしてメッセージを受信する
- Natural Language ML API を利用する
- Spanner のインスタンスを作成してデータを挿入する

## 手順

### 1. 環境整備

- git clone
- 必要ライブラリインストール
- アプリ実行
- ワーカーアプリ起動
今回は、クイズアプリのウェブ部分とコンソールを処理するワーカーの2つのアプリを,
それぞれターミナルを分けて実行する

### 2. コード確認

- pubsub.py
- languageapi.py
- spanner.py
- api.py
- worker.py

### 3. Pub/Sub操作

- トピック作成
コンソールで作成

- サブスクリプション作成
`gcloud pubsub subscriptions create worker-subscription --topic feedback`

- メッセージのパブリッシュ
テストしてみる
`gcloud pubsub topics publish feedback --message "Hello World"`

- サブスクリプションからメッセージ取得
↓のコマンドでメッセージをプルできる
`gcloud beta pubsub subscriptions pull worker-subscription --auto-ack`

### 4. プログラムでメッセージをパブリッシュする

- Pub/Subモジュールインストール
```py:pubsub.py
# モジュールインストール
from google.cloud import pubsub_v1

# クライアントを作成
publisher = pubsub_v1.PublisherClient()

# トピックパスを指定(feedback)
topic_path = publisher.topic_path(project_id, 'feedback')
```

- パブリッシュ用コードを追加

```py:pubsub.py
"""
feedbackをpublishする関数を作成
- feedbackのオブジェクトをJSON表記に変換
- UTF-8にエンコードする
- メッセージをパブリッシュする
- resultを返す
"""
def publish_feedback(feedback):
    payload = json.dumps(feedback, indent=2, sort_keys=True)
    data = payload.encode('utf-8')
    future = publisher.publish(topic_path, data=data)
    return future.result()
```

- アプリケーション側でPub/Subのパブリッシュ機能を使えるようにするコードを追加

```py:api.py
# pubsubをインポート
from quiz.gcp import datastore, pubsub
"""
quiz.gcpはこのプロジェクトのquiz/gcpディレクトリのこと
pubsub は pubsub.py のこと
"""

# アプリからパブリッシュする用の関数
"""
- pubsub.pyのpublish_feedback関数を実行して、result に入れる
- result を Response関数で整形して response に入れる
- response をクライアント(worker)に返す
"""
def publish_feedback(feedback):
    result = pubsub.publish_feedback(feedback)
    response = Response(json.dumps(result, indent=2, sort_keys=True))
    response.headers['Content-Type'] = 'application/json'
    return response
```

これで、アプリでフィードバックを入力して送信したときに、
publish_feedback関数によって、フィードバックが送信されたという通知がpub/subに送られる

- メッセージ作成

実際にアプリでフィードバックを送信してみる

まだこの段階では受信したフィードバックをpullしていないため、workerアプリでは何も起こらない

手動でpullしてみる
```sh
$ gcloud pubsub subscriptions pull worker-subscription --auto-ack
DATA: {"email": "app.dev.student@~~", "feedback": ~~~}
MESSAGE_ID: 000000000000
ORDERING_KEY:
ATTRIBUTES:
```
↑みたいな出力がでる

### 5. プログラムでメッセージをサブスクライブする

- サブスクリプションを作成してメッセージを受信する用のコードを追加

```py:pubsub.py
# クライアントを作成
sub_client = pubsub_v1.SubscriberClient()

# worker-subscription という名の subscriprionを参照するパスを取得する
sub_path = sub_client.subscription_path(project_id, 'worker-subscription')

# メッセージを受信したときにコールバックを呼び出すサブスクリプション関数を作成
def pull_feedback(callback):
    sub_client.subscribe(sub_path, callback=callback)
```

- サブスクライブ機能を使うためのコードを追加

```py:worker.py
# モジュールを読み込む
from quiz.gcp import pubsub

# メッセージの確認応答を行う関数を作成
def pubsub_callback(message):
    message.ack()
    log.info('Message received')
    log.ingo(message)
"""
message には おそらく pub/sub のトピックから送られてくる内容が入るはず
"""

# main()関数でPub/Subサブスクリプションコールバックとして登録する
def main():
    log.info('Worker starting....')
    pubsub.pull_feedback(pubsub_callback)
    while True:
        time.sleep(60)
"""
pubsub.pyで定義しているpull_feedback関数に引数として↑のpubsub_callback関数を返して実行
さっき手動でpullしたのを関数定義したやつかな？ 内容は見損ねた、、
"""
```

- WebAppとWorkerAppを実行してメッセージを作成する

Take Testでフィードバックを送信すると、ワーカーアプリで↓のような出力が出る

```
INFO:root:Worker starting...
INFO:root:Message received
INFO:root:Message {
 data: b'{\n  "email": "app.dev.student@example.org",\n  "fee...'
 ordering_key: ''
 attributes: {}
}
```

今のところ、Pub/Subを利用するメリットはないように見える
ワーカーアプリでログ出したところで、何の効果もないので

次以降、このSubscriptionを NLAPIや、Spannerと連携していく

### 6. Natural Language API を使う

Natural Language API によって、送信されたフィードバックの中身を解析して、
フィードバックに込められた感情を数値化する

- NL API を呼び出すコードを追加

```py:languageapi.py
# モジュールのインポート
from google.cloud import language_v1

# クライアント作成
lang_client = language_v1.LanguageServieClient()

# analyze関数でテキストのsentimentをスコアとして返す
def analyze(text):
    doc = language_v1.types.Document(content=text, type_="PLAIN_TEXT")
    sentiment = lang_client.analyze_sentiment(document=doc).document_sentiment
    return sentiment.score
```

- NL API 機能を使うためのコードを追加

```py:worker.py
# モジュールを読み込む
from quiz.gcp import pubsub, languageapi

# pubsub_callback関数に追記する
def pubsub_callback(message):
    message.ack()
    log.info('Message received')
    log.info(message)
    data = json.loads(message.data)
    # Pub/Subから受け取ったメッセージの中の、feedbackトピックのdataをstr型として受け取って、languageapi.analyze関数に渡す
    score = languageapi.analyze(str(data['feedback']))
    # ログ出力 '{}'.format(score)はpythonでの変数の表記方法
    log.info('Score: {}'.format(score))
    # scoreに新しいスコアプロパティを割り当てる(？ よくわからない作業)
    # →わかった。 languageapi実行するまでは data オブジェクトに score っていうプロパティがないから、
    # ここで新しくついかしてるんや
    data['score'] = score
    # これでspannerに保存できるようになる
```

- アプリを実行してNLAPIをテスト

TakeTestでフィードバック送信
感情が分かりやすい文を入れてみる
ワーカーアプリにて出力に
`INFO:root:Score: -0.8999999761581421`が出現 ←はWhat the fuck! を入れてみた -になってて草

### 7. Spannerにデータを保存する

感情スコアを含めた、フィードバックのデータをSpannerに保存していく

- Spanner インスタンスを作成する

コンソールでポチポチ

- データベースとテーブルを作成

コンソールでポチポチ

スキーマは↓
```sql
CREATE TABLE Feedback (
    feedbackId STRING(100) NOT NULL,
    email STRING(100),
    quiz STRING(20),
    feedback STRING(MAX),
    rating INT64,
    score FLOAT64,
    timestamp INT64
)
PRIMARY KEY (feedbackId);
```

- データ保存用のコードを追加

```py:spanner.py
# モジュール読み込み
from google.cloud import spanner

# クライアント作成
spanner_client = spanner.Client()

# インスタンスを指定
instance = spanner_client.instance('quiz-instance')

# データベースを指定
database = instance.database('quiz-database')

# 関数で with ブロックを使ってdatabase.batchオブジェクトを作成する
# batchオブジェクトを使うと、Spannerに対して複数のオペレーションを実行できる
def save_feedback(data):
    with database.batch() as batch:
        # Email,クイズ名,日付の値を使って、キー(feedback_id)の作成
        # reverse_email関数 : app.dev.student@example.com → com_example_student_dev_app
        # com_google_sbtkzks_place_202110311606 みたいなキーになる
        feedback_id = '{}_{}_{}'.format(
            reverse_email(data['email']),
            data['quiz'],
            data['timestamp']
        )
        # レコードをSpannerに挿入
        batch.insert(
            table='feedback',
            column=(
                'feedbackId',
                'email',
                'quiz',
                'timestamp',
                'rating',
                'score',
                'feedback'
            ),
            value=[
                (
                    feedback_id,
                    data['email'],
                    data['quiz'],
                    data['timestamp'],
                    data['rating'],
                    data['score'],
                    data['feedback']
                )
            ]
        )
```

- Spanner機能を使うためのコードを追加

```py:worker.py
# モジュールを追加インポート
from quiz.gcp import pubsub, languageapi, spanner

# call_back関数に追記
def pubsub_callback(message):
    message.ack()
    log.info('Message received')
    log.info(message)
    data = json.loads(message.data)
    score = languageapi.analyze(str(data['feedback']))
    log.info('Score: {}'.format(score))
    data['score'] = score
    # spanner.pyで作ったsave_feedback関数を実行して、データを挿入する
    spanner.save_feedback(data)
    # 完了したらログ出力
    log.info('Feedback saved')
```

- アプリを実行してSpannerをテスト

例によってTakeTestでフィードバック送信

Spannerでデータを見てみると、
feedbackId ~ score が保存されている！

## 感想

かなり実践的かつ、NLAPIも使える面白いラボやった

これまで4つ実践ラボをやってきて、
GCPではモジュールをインポートして、クライアントを作成して、その中のプロパティとかを取得する
っていう一連の流れがあることに気づいた

NLAPIも面白かった
感情分析して、数値で返ってくるので、この値が低い順とかでソートしたらフィードバックを効率よく確認できそう