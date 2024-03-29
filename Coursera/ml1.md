# GCP ML

## 講座内容

- 機械学習の意義
- APIの紹介
- JupyterNotebookを使用したモデル構築

## Googleが選ばれる理由

TensorFlowを活用したMLシステムは、2012ではほぼ0やったけど、2017では4000近くまで増えた
2021は8000~10000くらいかな？

機械学習のシステムパイプラインでは、
データ収集からの整形、MLエンジンへの転送が難しいと思われがちやけど、
大事なのはトレーニングされたモデルで分析して、予測を立てることである
ほとんどのプロジェクトはこの予想段階に到達できてない
これはデータ処理をバッチ、ストリーミング同じ方法で処理できると解決するかもしれない
GCPではDataflowを使っていい感じにデータ処理できる
GCPを、使おう！

## AIファーストとは

- AIとMLの違い

MLはAIの中のツールみたいなもの
ニュートンの法則みたいなもの

- MLの2つの段階

トレーニングする段階
推論する段階

トレーニングは、画像ピクセルデータに正解のラベルをつけたものをいくつも読み込ませて、
この後読み込むデータが元のやつと似てたらOKみたいなのをできるだけ近づけていく作業
推論は、トレーニングで作成したMLのモデルを使ってあらゆる予測を立てること

データサイエンティストは、トレーニング作業に全力を注ぎたくなるけど、
それでは意味なくて、本番環境を意識して、推論を実行可能にする「運用化」まで用意できるようにするのが大切

- GCPの機械学習

MLのモデルはニューラルネットワークを使う
NNは昔はいろんな問題があって使われなかったけど、今はめっちゃ使われてるやつらしい

翻訳、画像分類、音声認識など、ニューラルネットワークが得意

GoogleでもあらゆるプロダクトでMLが使われているけど、
それぞれ1プロダクトに1モデルというわけではなく、いろいろなモデルを使っている

1つのモデルからいろいろ分析するのは、やめようね！

- ヒューリスティックルールの置換

MLは反復データを学習させることはわかったが、
これはどうやってGoogleみたいに活用すればいいのか？
どんな種類の問題がMLによって解決できるのか？

Google検索を例にとる
これはまがいもなくGoogleの主要サービスだが、
最初の仕様は、入力された文字と、それを期待する答えに対して、あらゆるルールを手動でコードに追加していっていた
例えば、Giantsと検索されたときに、ユーザがベイエリアの場合SanFranciscoGiants,ニューヨークの場合はNewYorkGiantsのように
ただ、これは膨大なコード量になって、メンテもむずい
ここでMLを使って見たらしい
手書きルールを超えて、自動でルールをNNが覚えていって、
検索品質が向上していったらしい
しかも、ユーザがどんどん使うので、継続的に改善ができている

つまり、どんな問題がMLで解決できるかというと、
ルールを使っているすべての問題らしい

