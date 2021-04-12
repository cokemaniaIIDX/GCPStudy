# CodeBuild と Firebase を利用した Hugo 静的Webサイトのデプロイ のメモ

## 概要

Hugoで作成した静的WebサイトをFirebaseにデプロイする作業をCodeBuildで自動化する  
まず、主導でFirebaseにデプロイしてプロセスを理解して、そのあと自動化する

## 手順

### LinuxインスタンスでHugoインストール

インストール用シェルスクリプトを用意してくれているので、それを実行する  
内容的には、tarをwgetしてきて展開するだけ

### repoとサイト作成

- 今回はソースコードをCloudSourceRepositoryに作成する  
```sh
cd ~
gcloud source repos create my_hugo_site
gcloud source repos clone my_hugo_site
```

- Hugoのコマンドでサイト作成
  - --forceオプションでさっき作ったリポジトリの中に作成できる
```sh
cd ~
/tmp/hugo new site my_hugo_site --force
```

- テーマのインストール
```sh
cd ~/my_hugo_site
git submodule add \
  https://github.com/budparr/gohugo-theme-ananke.git \
  themes/ananke
echo 'theme = "ananke"' >> config.toml
```

- プレビューを見てみる
```sh
cd ~/my_hugo_site
/tmp/hugo server -D --bind 0.0.0.0 --port 8080
```

### Firebaseに追加

- toolのインストール、初期化
```sh
curl -sL https://firebase.tools | bash

cd ~/my_hugo_site
firebase init

/tmp/hugo && firebase deploy
```

### デプロイ自動化

- git 設定
```sh
git config --global user.name "[GIT_NAME]"
git config --global user.email "[GIT_EMAIL]"

cd ~/my_hugo_site
echo "resources" >> .gitignore

git add .
git commit -m "Add app to Cloud Source Repositories"
git push -u origin master
```

### CloudBuildを構成する

#### yamlファイルを持ってくる

- CloudBuildはYAMLファイルを利用してビルドを実行する
- 今回は用意してくれてるので、それを使う
```sh
cd ~/my_hugo_site
cp /tmp/cloudbuild.yaml .
```

```yaml
# Copyright 2020 Google Inc. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

steps:
- name: 'gcr.io/cloud-builders/wget'
  args:
  - '--quiet'
  - '-O'
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

steps:
- name: 'gcr.io/cloud-builders/wget'
  args:
  - '--quiet'
  - '-O'
  - 'firebase'
  - 'https://firebase.tools/bin/linux/latest'

- name: 'gcr.io/cloud-builders/wget'
  args:
  - '--quiet'
  - '-O'
  - 'hugo.tar.gz'
  - 'https://github.com/gohugoio/hugo/releases/download/v${_HUGO_VERSION}/hugo_extended_${_HUGO_VERSION}_Linux-64bit.tar.gz'
  waitFor: ['-']

- name: 'ubuntu:18.04'
  args:
  - 'bash'
  - '-c'
  - |
    mv hugo.tar.gz /tmp
    tar -C /tmp -xzf /tmp/hugo.tar.gz
    mv firebase /tmp
    chmod 755 /tmp/firebase
    /tmp/hugo
    /tmp/firebase deploy --project ${PROJECT_ID} --non-interactive --only hosting -m "Build ${BUILD_ID}"

substitutions:
  _HUGO_VERSION: 0.69.2
```

- 内容
  - 3つのsteps
    - wgetコンテナでfirebase,hugoをインストールする
    - 3つめのステップではUbuntuにHugoとFirebaseをインストールしてサイトを構築する
      - ビルドするたびにコンテナを作成？
        - →デプロイごとにHugoとFirebaseをインストールするので、いつでも最新バージョンを利用できるし、バージョンの指定もできる

### トリガーの作成

CommitされたときにCloudBuildがトリガーされるようにする

#### トリガーの作成

- 名前:commit-to-master-branch
- 説明:Push to master
- イベント:ブランチにpushする
- リポジトリ:my_hugo_site
- ブランチ:^master$
- ビルド構成:CloudBuild構成ファイル
- 場所:/ cloudbuild.yaml

#### サービスアカウントをアップデート

CloudBuilがFirebaseを使ってデプロイするための権限をサービスアカウントに付与する
- Firebaseプロダクト > Firebase Hosting 管理者

### パイプラインのテスト

- ファイル編集

```sh
cd ~/my_hugo_site
```

config.tomlを適当に編集する

- push

```sh
git add .
git commit -m "I updated the site title"
git push -u origin master
```

- cloud buildコンソールで確認
  - 実行されてたらOK！