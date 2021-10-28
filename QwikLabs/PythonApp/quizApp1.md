# Datastoreと連携する - Quiz App

## クイズアプリをホスティングする

このラボはFlaskで構築されたPythonのサーバアプリを作成するみたい

クイズの内容を記入して登録するサーバサイドアプリと、
クイズの答えを選択して答えるクライアントサイドアプリで構成されている

今回はAppEngine上で稼働させて、Datastoreにデータを保存する連携を学ぶ

## エンティティの登録

アプリが正常に動くようにdatastore.pyを修繕していく必要がある

- `os`モジュールをインストールする
- osモジュールを使って`GCLOUD_PROJECT`環境変数を取得する
- `google.cloud`パッケージから`datastore`モジュールをインポートする
- `datastore_client`という名前の`datastore.Client`クライアントオブジェクトを宣言する

```py
# TODO: os モジュールのインポート
import os
from flask import current_app
from google.cloud import datastore

project_id = os.getenv('GCLOUD_PROJECT')
datastore_client = datastore.Client(project_id)
```
- os.getenv('xxx') : python稼働中のosの環境変数から値を取得する
- import datastore : googleからdatastore関連のモジュールをインポート
- datastore_client : クライアントオブジェクトというものを作成している

つづき

- Datastore クライアント オブジェクトを使用して、種類が `Question` の Datastore エンティティのキーを作成する。
- そのキーを使用して、質問の Datastore エンティティを作成する。
- ウェブ アプリケーションのフォームから提供される値の辞書の項目を反復処理する。
- ループの本体で、キーと値を Datastore エンティティ オブジェクトに割り当てる。
- Datastore クライアントを使用してデータを保存する。

```py
"""
各質問のエンティティを作成および永続化する
Datastore キーは、リレーショナル データベースの主キーに相当
キーの主な作成方法は 2 つある:
1. 種類を指定し、Datastore で一意の数値 ID を生成する
2. 種類と一意の文字列 ID を指定する
"""
def save_question(question):
    key = datastore_client.key('Question')
    q_entity = datastore.Entity(key=key)
    for q_prop, q_val in question.items():
        q_entity[q_prop] = q_val
    datastore_client.put(q_entity)
```
- この関数が呼び出されるときに、questionという引数が渡される
- この引数は、フォームから送信されてくるクイズ内容の辞書型のデータ
- key = datastore_client.key('Question') : データストアクライアントのkeyの値がQuestionのものをkeyという変数に格納
- q_entity = datastore.Entity(key=key) : key が 上で指定したkeyのものを、データストアエンティティとしてq_entityに格納
- for ~~ q_val : フォームから送られてきた質問内容のkey-valueを反復処理でエンティティに割り当てる
- datastore_client.put(q_entity) : エンティティを保存

## クイズを登録

ここまでで、Datastoreにデータを保存する関数が完成したので実際にアプリで登録してみる

| フォームの項目 | 値                                              |
| :------------- | :---------------------------------------------- |
| Author         | [自分の名前]                                    |
| Quiz           | Google Cloud Platform                           |
| Title          | Which company owns GCP?                         |
| Answer 1       | Amazon                                          |
| Answer 2       | Google（[Answer 2] のラジオボタンを選択します） |
| Answer 3       | IBM                                             |
| Answer 4       | Microsoft                                       |

saveボタンで送信

- Datastoreを確認

エンティティが作成されていることが確認できる

## エンティティの取得

```py
"""
該当するクイズの Question エンティティ リストを返します
- クイズ名でフィルタ、gcp にデフォルト設定
- ページングなし
- ID プロパティとしてエンティティ キーに追加
- redact が true ならば、correctAnswer プロパティを各エンティティから削除
"""
def list_entities(quiz='gcp', redact=True):
    query = datastore_client.query(kind='Question')
    query.add_filter('quiz', '=', quiz)
    results =list(query.fetch())
    for result in results:
        result['id'] = result.key.id
    if redact:
        for result in results:
            del result['correctAnswer']
    return results
"""
```