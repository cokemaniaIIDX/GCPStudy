# firebase Emulatorを使う

認証機能などのテストを行うのに、FirebaseEmulatorが有用。
これを使えば本番環境のアプリのDBとかに変更を加えずに、認証機能とかのテストができるようになる。
例えば、ログイン機能の検証で実際のアプリにユーザを作成せずとも認証機能がうまく動くかどうかテストできる。

## init

まずfirebase emulatorをinitする必要がある。

```
// ログインしてない場合はログインする
$ firebase login:ci --no-localhost

// init
$ firebase init emulators

     ######## #### ########  ######## ########     ###     ######  ########
     ##        ##  ##     ## ##       ##     ##  ##   ##  ##       ##
     ######    ##  ########  ######   ########  #########  ######  ######
     ##        ##  ##    ##  ##       ##     ## ##     ##       ## ##
     ##       #### ##     ## ######## ########  ##     ##  ######  ########

You're about to initialize a Firebase project in this directory:

  /home/coke/imade-portal/frontend

Before we get started, keep in mind:

  * You are initializing within an existing Firebase project directory


=== Project Setup

First, let's associate this project directory with a Firebase project.
You can create multiple project aliases by running firebase use --add, 
but for now we'll just set up a default project.

i  Using project imade-portal (imade-portal)

=== Emulators Setup
? Which Firebase emulators do you want to set up? Press Space to select emulators, then Enter to confirm your choices. Authentication Emulator

↑Authentication Emulatorのみ選択。 ほかのやつは利用したくなったときにまたインストールしよう。

? Which port do you want to use for the auth emulator? 9099

↑デフォルトの9099ポートを使用。Authは9099がデフォでほかのやつはほかのポートを使う。

? Would you like to enable the Emulator UI? Yes

↑EmulatorUIを有効にするかどうか。とりあえずYesにした。

? Which port do you want to use for the Emulator UI (leave empty to use any available port)? 

↑EmulatorUIの起動ポートをどうするか。 空白にしとけば開いているポートをよしなに使ってくれるっぽい。

? Would you like to download the emulators now? Yes

↑Emulatorsを今DLするかどうか。 しないという選択は要るのか・・・？

i  ui: downloading ui-v1.7.0.zip...

i  Writing configuration info to firebase.json...
i  Writing project information to .firebaserc...

✔  Firebase initialization complete!
```

## Emulator起動

```
$ firebase emulators:start
i  emulators: Starting emulators: auth, hosting
i  hosting: Serving hosting files from: build
✔  hosting: Local server: http://localhost:5000
i  ui: downloading ui-v1.7.0.zip...
Progress: =========================================================================================================================================================================> (100% of 5MB)
i  ui: Emulator UI logging to ui-debug.log

┌─────────────────────────────────────────────────────────────┐
│ ✔  All emulators ready! It is now safe to connect your app. │
│ i  View Emulator UI at http://localhost:4000                │
└─────────────────────────────────────────────────────────────┘

┌────────────────┬────────────────┬────────────────────────────┐
│ Emulator       │ Host:Port      │ View in Emulator UI        │
├────────────────┼────────────────┼────────────────────────────┤
│ Authentication │ localhost:9099 │ http://localhost:4000/auth │
├────────────────┼────────────────┼────────────────────────────┤
│ Hosting        │ localhost:5000 │ n/a                        │
└────────────────┴────────────────┴────────────────────────────┘
  Emulator Hub running at localhost:4400
  Other reserved ports: 4500

Issues? Report them at https://github.com/firebase/firebase-tools/issues and attach the *-debug.log files.
```

`$ firebase emulators:start`でエミュレータが起動する。
エミュレータを起動した状態で`http://localhost:4000`にアクセスすると、
FirebaseエミュレータのUIページにアクセスできる。

## 動作確認してみる

Auth.tsxにログイン機能を追加して、
エミュレータでユーザ登録されてるかどうかを確認。

### エミュレータモード起動

```tsx:Auth.tsx
import { getAuth } from 'firebase/auth';
import { app } from './firebase';
import { connectAuthEmulator, createUserWithEmailAndPassword, signInWithEmailAndPassword } from 'firebase/auth';
import { stringify } from '@firebase/util';

const auth = getAuth(app);
connectAuthEmulator(auth, "http://localhost:9099");
```

↑これを追加することで、Reactプロジェクトがエミュレータ実行モードになる。
画面下に`Running in emulator mode. Do not use with production credentials.`と表示されるようになる。

### ユーザ登録機能追加

ユーザ新規登録フォーム(Formik)のonSubmitに、createUserInWithEmailAndPassword()を実行するように設定。

```tsx:Auth.tsx
<Formik
  initialValues={{
    email: '',
    password: '',
    rememberMe: false,
  }}
  validationSchema={validationSchema}
  onSubmit={(values, { setSubmitting }) => {
    setTimeout(() => {
      signInWithEmailAndPassword(auth, values.email, values.password)
        .then((userCredential) => {
          const user = userCredential.user;
          const email = stringify(user.email);
          props.SetEmail(email)
        })
        .catch((error) => {
          const errorCode = error.code;
          const errorMessage = error.message;
          alert(errorMessage);
        })
      setSubmitting(false);
    }, 400);
  }}
>
```

これで、フォームに入力されたEmailとPasswordでFirebaseにユーザ登録されるようになる。
ログインもこれと同じ感じで実装すれば良き良き。

## おわりに

認証機能の動作確認が簡単にできたし、
テストユーザの作成削除がGUIでスムーズにできたしとても良き。