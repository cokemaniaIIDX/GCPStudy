# Kubernetes Engine へのアプリケーションのデプロイ - QUiz App

GKE を使って QuizApp を Kubernetes で動かす

## 手順

### 1. 環境整備

これまでやったやつが自動で実行される
- AppEngine起動
- 環境変数エクスポート
- pip install
- Datastoreエンティティ作成
- Pub/Subトピック作成
- Spannerインスタンス、DB、テーブル作成
- ProjectID出力

NLは解雇・・・！

### 2. コード確認

- frontend , backend のディレクトリを確認(それぞれWEBアプリとワーカーアプリ)

今回はDockerfileメインかな

- frontend , backend それぞれにある Dockerfile を確認する

まだ空でした

### 3. Kubernetes Engine クラスタを作成

コンソールでポチポチ

- 接続

`$ gcloud container clusters get-credentials quiz-cludter --zone us-central1-b --project qwiklabs~~~`
でクラスタへの認証を作成する

`$ kubectl get pods`
で接続
今はまだ何もないので接続するものがないけど、セキュリティが構成できてるのは確認できる

### 4. Container Builder を使って Docker イメージを作成する

- フロント、バック用のDockerfileを作成する

```Dockerfile
FROM gcr.io/google_appengine/python
RUN virtualenv -p python3.7 /env
ENV VIRTUAL_ENV /env
ENV PATH /env/bin:$PATH
ADD requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
ADD . /app
CMD gunicorn -b 0.0.0.0:$PORT quiz:app
```

```Dockerfile
FROM gcr.io/google_appengine/python
RUN virtualenv -p python3.7 /env
ENV VIRTUAL_ENV /env
ENV PATH /env/bin:$PATH
ADD requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
ADD . /app
CMD python -m quiz.console.worker
```

- ContainerBuilderでDockerイメージを作成する

```
$ gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-frontend ./frontend/
$ gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-backend ./backend/
```

これでコンテナレジストリにコンテナが登録される

### 5. Kubernetes のデプロイメントとサービスのリソースを作成する

- デプロイメントファイルを作成

```yaml
# Copyright 2017 Google Inc.
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
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quiz-frontend
  labels:
    app: quiz-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: quiz-app
      tier: frontend
  template:
    metadata:
      labels:
        app: quiz-app
        tier: frontend
    spec:
      containers:
      - name: quiz-frontend
        image: gcr.io/qwiklabs-gcp-03-0e79dfa13dc3/quiz-frontend
        imagePullPolicy: Always
        ports:
        - name: http-server
          containerPort: 8080
        env:
          - name: GCLOUD_PROJECT
            value: qwiklabs-gcp-03-0e79dfa13dc3
          - name: GCLOUD_BUCKET
            value: qwiklabs-gcp-03-0e79dfa13dc3-media
```

```yaml
# Copyright 2017 Google Inc.
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
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quiz-backend
  labels:
    app: quiz-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: quiz-app
      tier: backend
  template:
    metadata:
      labels:
        app: quiz-app
        tier: backend
    spec:
      containers:
      - name: quiz-backend
        image: gcr.io/qwiklabs-gcp-03-0e79dfa13dc3/quiz-backend
        imagePullPolicy: Always
        env:
          - name: GCLOUD_PROJECT
            value: qwiklabs-gcp-03-0e79dfa13dc3
          - name: GCLOUD_BUCKET
            value: qwiklabs-gcp-03-0e79dfa13dc3-media
```

```yaml
# Copyright 2017 Google Inc.
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
apiVersion: v1
kind: Service
metadata:
  name: quiz-frontend
  labels:
    app: quiz-app
    tier: frontend
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: http-server
  selector:
    app: quiz-app
    tier: frontend
```

- デプロイメントとサービスのファイルを実行する

```
$ kubectl create -f ./frontend-deployment.yaml
$ kubectl create -f ./backend-deployment.yaml
$ kubectl create -f ./frontend-service.yaml
```

### 6. クイズアプリのテスト

- デプロイされたリソースを確認

コンソールからたどるか、
`$ kubectl get services`
で表示されるExternalIPをブラウザに打ち込んで、クイズアプリがちゃんと表示されるか確認！