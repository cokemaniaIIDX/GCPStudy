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