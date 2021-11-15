# Identity-Aware Proxy でユーザ認証する

簡単なアプリをAppEngineに構築し、IAPでユーザのアクセス制限したり、
ID情報をアプリに渡したりする方法を学ぶ

## はじめに

基本的にユーザ認証はアプリでプログラミングする必要があるけど、
GCPではIAPを使うことで簡単に実装できる
アクセス制限のみやとアプリを修正する必要もないし、
ユーザIDの認識をするにしてもIAPで最小限のコードで実装できる

## 手順

- IAPのページからAPIを有効化する

- 同意画面の作成
  - 内部を選択
  - アプリケーション名、メールアドレス等を入力する

- FlexAPIを無効化する

- IAPの切り替えボタンで有効化する

- AppEngineにチェックマークを付けると、右メニューが出現する
  - 権限を追加する
    - Cloud IAP / IAP-Secured WebApp User ロールを付与

## 確認

AppEngineが出力するURLにアクセスすると、Googleのアカウントでアクセスする画面が出る


## ユーザ情報をとったりするとき

```py
user_email = request.headers.get('X-Goog-Authenticated-User-Email')
user_id = request.headers.get('X-Goog-Authenticated-User-ID')
```

こんな感じでリクエストヘッダから専用の値がとれるようになる

```py
page = render_template('index.html', email=user_email, id=user_id)
```
みたいな感じでtemplateに変数を渡してやれば、index.htmlで{{ email }}で呼び出せる

## 暗号検証

あんまりよくわからなかったのでとばす