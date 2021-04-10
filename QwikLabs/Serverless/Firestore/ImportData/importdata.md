# データをFirestoreにインポートする のメモ

## 概要

- PetTheoryのデータベースにあるデータをFirestoreにインポートする
- CSVをFirestoreに保存できる形式に変換するプログラムを作成して実行
- 顧客データ(PII)を使ってテストするのはよくないので、Fakerというフェイクのデータを生成できるツールを使って試す

## 手順

### Firestoreセットアップ

### データインポート用コード編集

#### packageインストール

- FirestoreとCloudLoggingを使えるようにするためのパッケージをインストール
```sh
npm install @google-cloud/firestore
npm install @google-cloud/logging
```

- ↓のようにpackage.jsonのdependenciesに自動で追加される
```json
...

"dependencies": {
  "@google-cloud/firestore": "^2.4.0",
  "@google-cloud/logging": "^5.4.1",
  "csv-parse": "^4.4.5"
}
```

#### importTestData.js スクリプト修正

- 前のCSVからデータベースへの書き込みスクリプトを、Firestore用に修正する
```js
const {promisify} = require('util');
const parse       = promisify(require('csv-parse'));
const {readFile}  = require('fs').promises;
const {Firestore} = require('@google-cloud/firestore'); // Firestoreのパッケージを読み込む

if (process.argv.length < 3) {
  console.error('Please include a path to a csv file');
  process.exit(1);
}

// ここから追記
const db = new Firestore();

function writeToFirestore(records) {
  const batchCommits = [];
  let batch = db.batch();
  records.forEach((record, i) => {
    var docRef = db.collection('customers').doc(record.email);
    batch.set(docRef, record);
    if ((i + 1) % 500 === 0) {
      console.log(`Writing record ${i + 1}`);
      batchCommits.push(batch.commit());
      batch = db.batch();
    }
  });
  batchCommits.push(batch.commit());
  return Promise.all(batchCommits);
}
// ここまで
/* 
上記のコード スニペットは、新しいデータベース オブジェクトを宣言します。const db
これは、ラボの前半で作成したデータベースを参照します。 = new Firestore();
この関数は、各レコードが順番に処理されるバッチプロセスを使用し、
追加された ID に基づいてドキュメント参照を設定します。
関数の最後に、バッチ コンテンツがデータベースに書き込まれます。
*/

function writeToDatabase(records) {
  records.forEach((record, i) => {
    console.log(`ID: ${record.id} Email: ${record.email} Name: ${record.name} Phone: ${record.phone}`);
  });
  return ;
}

async function importCsv(csvFileName) {
  const fileContents = await readFile(csvFileName, 'utf8');
  const records = await parse(fileContents, { columns: true });
  try {
    await writeToDatabase(records);
  }
  catch (e) {
    console.error(e);
    process.exit(1);
  }
  console.log(`Wrote ${records.length} records`);
}

importCsv(process.argv[2]).catch(e => console.error(e));
```

### テストデータ作成



### データ読み込み



### アクセス許可設定


