# Firebaseを利用したWebアプリのメモ

## 概要

- ユーザが情報をログに記録して、予約をリアルタイムでスケジュールできるアプリの作成
  - Firebaseプロジェクトの作成
  - Firebaseセキュリティルールを構成する
  - Googleログインをアプリに追加する
  - ユーザの連絡先情報登録用のDBを構築する
  - ユーザページを作成して、予約スケジュールを可能にするコードのデプロイ
  - Firebaseのリアルタイムアップデートを使ってみる

## 手順

### 環境のプロビジョニング

- GCPコンソールでFirestore,FirebaseAPIとかを有効にする
- Firebaseコンソールでアプリの登録を行う
- Firebase CLIをインストールして設定

### 認証の作成

- firestore.rulesで設定する
  - 今回は確認するだけ

### アプリのデプロイ

- デプロイ
```sh
firebase deploy
```

### 顧客ページの作成

- customer.js作成
```js
let user;

firebase.auth().onAuthStateChanged(function(newUser) {
  user = newUser;
  if (user) {
    const db = firebase.firestore();
    db.collection("customers").doc(user.email).onSnapshot(function(doc) {
      const cust = doc.data();
      if (cust) {
        document.getElementById('customerName').setAttribute('value', cust.name);
        document.getElementById('customerPhone').setAttribute('value', cust.phone);
      }
      document.getElementById('customerEmail').innerText = user.email;
    });
  }
});

document.getElementById('saveProfile').addEventListener('click', function(ev) {
  const db = firebase.firestore();
  var docRef = db.collection('customers').doc(user.email);
  docRef.set({
    name: document.getElementById('customerName').value,
    email: user.email,
    phone: document.getElementById('customerPhone').value,
  })
})

```

- css 作成
```css
body { background: #ECEFF1; color: rgba(0,0,0,0.87); font-family: Roboto, Helvetica, Arial, sans-serif; margin: 0; padding: 0; }
#message { background: white; max-width: 360px; margin: 100px auto 16px; padding: 32px 24px 16px; border-radius: 3px; }
#message h3 { color: #888; font-weight: normal; font-size: 16px; margin: 16px 0 12px; }
#message h2 { color: #ffa100; font-weight: bold; font-size: 16px; margin: 0 0 8px; }
#message h1 { font-size: 22px; font-weight: 300; color: rgba(0,0,0,0.6); margin: 0 0 16px;}
#message p { line-height: 140%; margin: 16px 0 24px; font-size: 14px; }
#message a { display: block; text-align: center; background: #039be5; text-transform: uppercase; text-decoration: none; color: white; padding: 16px; border-radius: 4px; }
#message, #message a { box-shadow: 0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24); }
#load { color: rgba(0,0,0,0.4); text-align: center; font-size: 13px; }
@media (max-width: 600px) {
  body, #message { margin-top: 0; background: white; box-shadow: none; }
  body { border-top: 16px solid #ffa100; }
}
```

- 再デプロイ
```
firebase deploy
```

### 予約スケジュールページの作成

- appointments.html作成
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Pet Theory appointments</title>
    <script src="/__/firebase/6.4.2/firebase-app.js"></script>
    <script src="/__/firebase/6.4.2/firebase-auth.js"></script>
    <script src="/__/firebase/6.4.2/firebase-firestore.js"></script>
    <script src="/__/firebase/init.js"></script>
    <link type="text/css" rel="stylesheet" href="styles.css" />
  </head>
  <body>
    <div id="message">
      <h2>Scheduled appointments</h2>
      <div id="appointments"></div>
      <hr/>
      <h2>Schedule a new appointment</h2>
      <select id="timeslots">
        <option value="0">Choose time</option>
      </select>
      <br><br>
      <button id="makeAppointment">Schedule</button>
    </div>
    <script src="appointments.js"></script>
  </body>
</html>
```

- appointments.js 作成
```js
let user;

firebase.auth().onAuthStateChanged(function(newUser) {
  user = newUser;
  if (user) {
    const db = firebase.firestore();
    const appColl = db.collection('customers').doc(user.email).collection('appointments');
    appColl.orderBy('time').onSnapshot(function(snapshot) {
      const div = document.getElementById('appointments');
      div.innerHTML = '';
      snapshot.docs.forEach(appointment => {
        div.innerHTML += formatDate(appointment.data().time) + '<br/>';
      })
      if (div.innerHTML == '') {
        div.innerHTML = 'No appointments scheduled';
      }
    });
  }
});

const timeslots = document.getElementById('timeslots');
getOpenTimes().forEach(time => {
  timeslots.add(new Option(formatDate(time), time));
});

document.getElementById('makeAppointment').addEventListener('click', function(ev) {
  const millis = parseInt(timeslots.selectedOptions[0].value);
  if (millis > 0) {
    const db = firebase.firestore();
    db.collection('customers').doc(user.email).collection('appointments').add({
      time: millis
    })
    timeslots.remove(timeslots.selectedIndex);
  }
})

function getOpenTimes() {
  const retVal = [];
  let startDate = new Date();
  startDate.setMinutes(0);
  startDate.setSeconds(0);
  startDate.setMilliseconds(0);
  let millis = startDate.getTime();
  while (retVal.length < 5) {
    const hours = Math.floor(Math.random() * 5) + 1;
    millis += hours * 3600 * 1000;
    if (isDuringOfficeHours(millis)) {
      retVal.push(millis);
    }
  }
  return retVal;
}

function isDuringOfficeHours(millis) {
  const aDate = new Date(millis);
  return aDate.getDay() != 0 && aDate.getDay() != 6 &&
         aDate.getHours() >= 9 && aDate.getHours() <= 17;
}

function formatDate(millis) {
  const aDate = new Date(millis);
  const days = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];
  const months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
                  'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
  return days[aDate.getDay()] + ' ' + aDate.getDate() + ' ' +
         months[aDate.getMonth()] + ', ' + aDate.getHours() + ':' +
         (aDate.getMinutes() < 10? '0'+aDate.getMinutes(): aDate.getMinutes());
}
```

- 再デプロイ
```
firebase deploy
```

