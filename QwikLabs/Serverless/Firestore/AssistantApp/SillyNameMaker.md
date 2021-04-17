# Silly Name Maker のメモ

## Google Assistant の開発について

Googleアシスタントアプリは、Actionsがプラットフォームの中心となる  
ActionsではHCIと連携できるので、簡単に会話型アプリが開発できる  
中でもよく使われてるのは Dialogflow で、MLとNLUを使ってアプリを開発できる  

HCI:Human Computer Interaction

## Actions の構築

- アクションとは Googleアシスタント用に構築する インタラクション である
- アックションは特定のインテントをサポートし、フルフィルメントによって実行される
- インテント：ユーザが達成したい目標や行いたいタスクのこと
- フルフィルメント：インテントを処理して、対応するアクションを実行するロジック

## コンソール操作

Actions コンソール、DialogFlowコンソールは独自操作が多いので割愛！！

## Function 中身

- Dialogflow インテントがトリガーされる際、（インテントのアクション エリアで宣言されている）インテントのアクション名が、フルフィルメントに対するリクエストで提供されます。このアクション名を使用して、実行するロジックを決めます。
- Dialogflow でユーザー入力からパラメータが解析された場合、フルフィルメントに対する各リクエスト内で、名前でパラメータを見つけてアクセスできます。ここでは、パラメータに後からアクセスできるようにパラメータ名を宣言します。
- この関数により、くだらない名前を生成してアクションが実行されます。この関数は make_name インテントがトリガーされるたびに呼び出され、ユーザー入力からパラメータが使用されて名前が生成されます。

```js
'use strict';

const {dialogflow} = require('actions-on-google');
const functions = require('firebase-functions');

const app = dialogflow({debug: true});

app.intent('make_name', (conv, {color, number}) => {
  conv.close(`Alright, your silly name is ${color} ${number}! ` +
    `I hope you like it. See you next time.`);
});

exports.sillyNameMaker = functions.https.onRequest(app);
```

```json
{
  "name": "silly-name-maker",
  "description": "Find out your silly name!",
  "version": "0.0.1",
  "author": "Google Inc.",
  "engines": {
    "node": "8"
  },
  "dependencies": {
    "actions-on-google": "^2.0.0",
    "firebase-admin": "^4.2.1",
    "firebase-functions": "1.0.0"
  }
}
```