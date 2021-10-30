# Firebase と連携 - Quiz App

QuizアプリにFirebaseを利用した認証機能を追加して、
ユーザ登録、ログインしないとクイズに答えられないようにする

## 目標

- FirebaseとGCPプロジェクトを連携する
- クライアント側のアプリにFirebaseの構成を追加する
- PythonコードでFirebaseをクライアントアプリと統合させる

## 手順

### 1. 環境準備

- リポジトリクローン
- アプリの実行

### 2. コード確認

- index.html

Angular.jsで作られたSPAの模様

- qiq-login-template.html

ログインページ これもAngular.js

- qiq-login.js

Angular.jsによるログインに関する関数
見ても正直意味わからん

### 3. Firebaseを操作する

- プロジェクト作成
- 認証方法でメール/パスワードを有効にする
- 認証ドメインで、AppEngineアプリのドメインをコピーして追加する

### 4. アプリをFirebaseと統合する

- firebaseで発行されるscriptタグをindex.htmlに張り付ける
- Firebaseライブラリを追加する

firebase-app.jsの行の下に追加ってあるけどないぞ、、、？

### 5. 確認

登録用のフォームがないぞ、、、？
バージョンが古い・・・！！！

## 感想

突然Angular使いだすし、認証の確認もできんかったし
クソやんけ！

Firebase自体は前やったサーバレスアプリのほうで試せたから、まぁいいか、、、