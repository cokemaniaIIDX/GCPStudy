# Doker のメモ

## Dockerfile の書式

```Dockerfile
cat > Dockerfile <<EOF
# 上位イメージとして正式な Node ランタイムを使用します
FROM node:6

# コンテナの作業ディレクトリを /app に設定します
WORKDIR /app

# 現行ディレクトリの内容をコンテナの /app にコピーします
ADD . /app

# コンテナのポート 80 で外部からアクセスできるようにします
EXPOSE 80

# コンテナの起動時に node を使用して app.js を実行します
CMD ["node", "app.js"]
EOF
```

## できたこと

- Docker の基本の基本が学べた
- 正直細かいコマンドは調べたらすぐ出てくるし本格的にやり始めてから覚えればいいかな
- 昔に一回Docker勉強したので、内容はすんなり頭に入った