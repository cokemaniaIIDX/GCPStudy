# データをFirestoreにインポートする のメモ

## 概要

- PetTheoryのデータベースにあるデータをFirestoreにインポートする
- CSVをFirestoreに保存できる形式に変換するプログラムを作成して実行
- 顧客データ(PII)を使ってテストするのはよくないので、Fakerというフェイクのデータを生成できるツールを使って試す

## 手順

### Firestoreセットアップ

コンソールでネイティブモードを有効にする

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
const {Logging} = require('@google-cloud/logging');     // Loggingにログが出るようにする

const logName = 'pet-thoery-logs-importTestData';

// logging クライアントの作成
const logging = new Logging();
const log = logging.log(logName);

const resource = {
    type: 'global',
};

if (process.argv.length < 3) {
  console.error('Please include a path to a csv file');
  process.exit(1);
}

// ここから
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
// ここまで追記
/* 
上記のコード スニペットは、新しいデータベース オブジェクトを宣言します。const db = new Firestore();
これは、ラボの前半で作成したデータベースを参照します。
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
// ↑これは古いほうなので使わない

async function importCsv(csvFileName) {
  const fileContents = await readFile(csvFileName, 'utf8');
  const records = await parse(fileContents, { columns: true });
  try {
    await writeToFirestore(records);
    // await writeToDatabase(records); // 古いほうはコメントアウトする
  }
  catch (e) {
    console.error(e);
    process.exit(1);
  }
  console.log(`Wrote ${records.length} records`);
  
  // テキスト ログエントリ
  success_message = `Success: importTestData - Wrote ${records.length} records`
  const entry = log.entry({resource: resource}, {message: `${success_message}`});
  log.write([entry]);
}

importCsv(process.argv[2]).catch(e => console.error(e));
```

### テストデータ作成

- faker を使ってテスト用データを作成する
```js
const fs = require('fs');
const faker = require('faker');
const {Logging} = require('@google-cloud/logging'); // Loggingが使えるように追加

const logName = 'pet-theory-logs-createTestData';

// Logging クライアントの作成
const logging = new Logging();
const log = logging.log(logName);

const resource = {
  // この例では、単純化のために「global」リソースをターゲットにしています。
  type: 'global',
};

function getRandomCustomerEmail(firstName, lastName) {
  const provider = faker.internet.domainName();
  const email = faker.internet.email(firstName, lastName, provider);
  return email.toLowerCase();
}

async function createTestData(recordCount) {
  const fileName = `customers_${recordCount}.csv`;
  var f = fs.createWriteStream(fileName);
  f.write('id,name,email,phone\n')
  for (let i=0; i<recordCount; i++) {
    const id = faker.random.number();
    const firstName = faker.name.firstName();
    const lastName = faker.name.lastName();
    const name = `${firstName} ${lastName}`;
    const email = getRandomCustomerEmail(firstName, lastName);
    const phone = faker.phone.phoneNumber();
    f.write(`${id},${name},${email},${phone}\n`);
  }
  console.log(`Created file ${fileName} containing ${recordCount} records.`);

  // テキスト ログエントリ
  const success_message = `Success: createTestData - Created file ${fileName} containing ${recordCount} records.`
  const entry = log.entry({resource: resource}, {name: `${fileName}`, recordCount: `${recordCount}`, message: `${success_message}`});
  log.write([entry]);
}

recordCount = parseInt(process.argv[2]);
if (process.argv.length != 3 || recordCount < 1 || isNaN(recordCount)) {
  console.error('Include the number of test data records to create. Example:');
  console.error('    node createTestData.js 100');
  process.exit(1);
}

createTestData(recordCount);
```

### データ読み込み

```sh
node importTestData customers_1000.csv
```
→Firestoreにデータが入ってる！

### アクセス許可設定

- デベロッパーに編集権限を与えたくないので、閲覧権限のみ与える
```sh
$ gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:[EMAIL] --role=roles/logging.viewer

$ gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:[EMAIL] --role roles/source.writer
```