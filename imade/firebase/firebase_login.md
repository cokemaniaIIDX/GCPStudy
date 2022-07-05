# ssh 先の linux マシン上で firebase login する方法

firebase login を実行し、出力されたURLにアクセスして認証しようとすると、
localhostにアクセスできませんといわれて、ログインできない。
`firebase login を実行するOS上`でしかログインできない模様。

今回みたいなssh先のlinux上でfirebase loginする場合は、↓のコマンドでログインする。

```
$ firebase login:ci --no-localhost

To sign in to the Firebase CLI:

1. Take note of your session ID:

   7AF72

2. Visit the URL below on any device and follow the instructions to get your code:

   https://auth.firebase.tools/login?code_challenge=xxxxxxxxxxxxxxxxxx....

3. Paste or enter the authorization code below once you have it:

? Enter authorization code: ## ログイン後出力されるコードをここに入力する
```

ログインできるとtokenIDが出力されるので、環境変数に設定する。

```
✔  Success! Use this token to login on a CI server:

1//0eqI**************************tokenコード****************************

Example: firebase deploy --token "$FIREBASE_TOKEN"


$ export FIREBASE_TOKEN=1//0eqI**************************tokenコード****************************
```

```
$ firebase init --token "$FIREBASE_TOKEN"

     ######## #### ########  ######## ########     ###     ######  ########
     ##        ##  ##     ## ##       ##     ##  ##   ##  ##       ##
     ######    ##  ########  ######   ########  #########  ######  ######
     ##        ##  ##    ##  ##       ##     ## ##     ##       ## ##
     ##       #### ##     ## ######## ########  ##     ##  ######  ########
```

↑が表示されて、そのあとの操作を要求されたらOK
以降はそれぞれ用途ごとに初期化を実行していく。