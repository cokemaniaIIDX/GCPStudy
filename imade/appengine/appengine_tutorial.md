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

## 6. Firebase連携

https://console.firebase.google.com/?hl=ja
でプロジェクト作成

Authentication → 始める

  ログインプロバイダ メール/パスワード を `有効にする` → 保存

  承認済みドメイン ドメインを追加 → `imade-gaming-265014.an.r.appspot.com`を追加

### firebaseをWEBサービスに追加

firebaseのページでアプリを追加 → コード → アプリ名適当に
~~scriptタグが表示されるのでコピペする~~
※Firebaseで出力されるコードスニペットではうまくいかなかったので
チュートリアルに書いてあるコードのほうを利用する

```html:index.html
<script src="https://www.gstatic.com/firebasejs/7.14.5/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/7.8.0/firebase-auth.js"></script>
<script>
  const firebaseConfig = {
    apiKey: "AIzaSyC6XESRzG4DXBTusAXYEX_aVbTmOUB6PJ8",
    authDomain: "imade--test.firebaseapp.com",
    projectId: "imade--test",
    storageBucket: "imade--test.appspot.com",
    messagingSenderId: "476483753824",
    appId: "1:476483753824:web:0fcfa8c2515c59854c7005"
  };
  firebase.initializeApp(firebaseConfig);
</script>
```

headには↓を記載

```html:index.html
<head>
  <title>Datastore and Firebase Auth Example</title>
  <script src="{{ url_for('static', filename='script.js') }}"></script>
  <link type="text/css" rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
  <script src="https://www.gstatic.com/firebasejs/ui/4.4.0/firebase-ui-auth.js"></script>
  <link type="text/css" rel="stylesheet" href="https://www.gstatic.com/firebasejs/ui/4.4.0/firebase-ui-auth.css" />
</head>
```

中身を修正

```html:index.html
<body>
  <script src="https://www.gstatic.com/firebasejs/7.14.5/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/7.8.0/firebase-auth.js"></script>
  <script>
    const firebaseConfig = {
      apiKey: "AIzaSyC6XESRzG4DXBTusAXYEX_aVbTmOUB6PJ8",
      authDomain: "imade--test.firebaseapp.com",
      projectId: "imade--test",
      storageBucket: "imade--test.appspot.com",
      messagingSenderId: "476483753824",
      appId: "1:476483753824:web:0fcfa8c2515c59854c7005"
    };
    firebase.initializeApp(firebaseConfig);
  </script>

  <h1>Datastore and Firebase Auth Example</h1>

  <div id=""firebaseui-auth-container></div>

  <button id="sign-out" hidden="true">Sign Out</button>

  <div id="login-info" hidden="true">
    <h2>Login info:</h2>
    {% if user_data %}
      <dl>
        <dt>Name</dt><dd>{{ user_data['name'] }}</dd>
        <dt>Email</dt><dd>{{ user_data['email'] }}</dd>
        <td>Last 10 visits</td><dd>
          {% for time in times %}
            <p>{{ time['timestamp'] }}</p>
          {% endfor %}</dd>
      </dl>
    {% elif error_message %}
      <p>Error: {{ error_message }}</p>
    {% endif %}
  </div>
</body>
</html>
```

### 認証用メソッドを追加

```js:script.js
'use strict';

window.addEventListener('load', function () {
  document.getElementById('sign-out').onclick = function () {
    firebase.auth().signOut();
  };

  var uiConfig = {
    signInSuccessUrl: '/',
    signInOptions: [
      firebase.auth.GoogleAuthProvider.PROVIDER_ID,
      firebase.auth.EmailAuthProvider.PROVIDER_ID,
    ],
    tosUrl: 'https://imade-gaming-265014.an.r.appspot.com'
  };

  firebase.auth().onAuthStateChanged(function (user) {
    if (user) {
      document.getElementById('sign-out').hidden = false;
      document.getElementById('login-info').hidden = false;
      console.log(`Signed in as ${user.displayName} (${user.email})`);
      user.getIdToken().then(function (token) {
        document.cookie = "token=" + token;
      });
    } else {
      var ui = new firebaseui.auth.AuthUI(firebase.auth());
      ui.start('#firebaseui-auth-container', uiConfig);
      document.getElementById('sign-out').hidden = true;
      document.getElementById('login-info').hidden = true;
      document.cookie = "token=";
    }
  }, function (error) {
    console.log(error);
    alert('Unable to log in: ' + error)
  });
});
```

### トークンの検証、復号バックエンド作成

```py:main.py
firebase_request_adapter = requests.Request()
@app.route('/')
def root():
    id_token = request.cookies.get("token")
    error_message = None
    claims = None
    times = None

    if id_token:
        try:
            claims = google.oauth2.id_token.verify_firebase_token(
                id_token, firebase_request_adapter)
        except ValueError as exc:
            error_message = str(exc)

    UTC = datetime.datetime.now(tz=datetime.timezone.utc)
    JST = datetime.timezone(datetime.timedelta(hours=+9), 'JST')
    dt = UTC.astimezone(JST)
    store_time(dt)
    times = fetch_times(10)

    return render_template(
        'index.html',
        user_data=claims, error_message=error_message, times=times)

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=8080, debug=True)
```

### 確認

- requirements.txt追加

```
$ pip install -r requirements.txt

$ python main.py
```

→認証ボタンが出てくる

### デプロイ

```
$ gcloud app deploy

$ gcloud app browse
```

## 7. データのパーソナライズ

1. store, fetchメソッド更新

```py:main.py
datastore_client = datastore.Client()

def store_time(email, dt):
    entity = datastore.Entity(key=datastore_client.key('User', email, 'visit'))
    entity.update({
        'timestamp': dt
    })

    datastore_client.put(entity)

def fetch_times(email, limit):
    ancestor = datastore_client.key('User', email)
    query = datastore_client.query(kind='visit', ancestor=ancestor)
    query.order = ['-timestamp']

    times = query.fetch(limit=limit)

    return times
```

データの保存メソッドとデータの整形メソッドにユーザとEmailを追加
visitエンティティに対して`ancestor`を関連付ける
※ancestorがユーザ独自で決める変数なのか、Datastore固有のものなのかは不明

2. rootメソッド更新

```py:main.py
firebase_request_adapter = requests.Request()
@app.route('/')
def root():
    id_token = request.cookies.get("token")
    error_message = None
    claims = None
    times = None

    if id_token:
        try:
            claims = google.oauth2.id_token.verify_firebase_token(
                id_token, firebase_request_adapter)
            UTC = datetime.datetime.now(tz=datetime.timezone.utc)
            JST = datetime.timezone(datetime.timedelta(hours=+9), 'JST')
            dt = UTC.astimezone(JST)
            store_time(claims['email'], dt)
            times = fetch_times(claims['email'], 10)
        except ValueError as exc:
            error_message = str(exc)

    return render_template(
        'index.html',
        user_data=claims, error_message=error_message, times=times)
```

tokenを検証して得た`claims変数`の中にある`email`をさっき更新したメソッドの引数に渡してあげる

### インデックスの構成

Datastoreではインデックスというのに基づいてクエリを作成している
ただ、祖先があるエンティティとかの複雑なエンティティについては自動生成のインデックスが使えないので、
手動でインデックスを作成して読み込ませる必要がある

```yaml:index.yaml
indexes:

- kind: visit
  ancestor: yes
  properties:
  - name: timestamp
    direction: desc
```

Datastoreにデプロイ

```
$ gcloud datastore indexes create index.yaml
```

→コンソールで確認

### 確認

```
$ python main.py

http://localhost:8080/
```

思った通り、Emailログインは未登録の場合新規作成になる
この辺は理解進んだなぁ

### デプロイ

```
$ gcloud app deploy

$ gcloud app browse
```

# 完了！

完全に理解したｗ