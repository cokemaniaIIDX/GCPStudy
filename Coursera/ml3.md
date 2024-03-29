# ML勉強

## MLはデータが最重要

## 機械学習のトレーニングフェーズ

機械学習も、アプリケーションを同じように、
継続的にMLモデルのトレーニング、評価、検証、モニタリングなどを行っていく必要がある
これを、DevOpsをもじってMLOpsという

ただし、MLは継続してトレーニングしていくと次第に複雑になってくるし、
しかも予想性能も落ちてくる可能性があるらしい
MLのモデルで重要なのは、簡素なものである必要があるらしい
簡素なモデルやと、予想結果も精度が高くなるらしい

継続的なモデルのトレーニングを続けて行くためには、
手動でのトレーニングから始めることになると思うが、
次第にトレーニング、デプロイ、インテグレーションを自動化していくと、運用効率が改善される

## ML問題のフレーム化

例えば製造業の需要予測では、
- 予測したいのは何か？
  - 来月の販売台数
- どんなデータが必要か？
  - 先月までの販売台数
  - 返品個数や他社製品の販売価格
- APIは何が必要か？
  - 部品のIDや予想したい月など

こんな感じで考えていくらしい
やはり難しい

## MLによるビジネス

機械学習ではモデルの複雑さよりモデルの量が大事
簡素なモデルでは短いサイクルで失敗を繰り返して、多くのアイデアを試したほうが成功率が上がる

多くの企業が利用しているデータは構造化されていないので、
MLパイプラインを使って構造化して、単純なデータになったものをモデルへの入力に使うようにする

MLの成功のカギはユーザの満足度
ユーザがMLとかかわることになったシーンを予測して、
それに対する適切な対応を取れるように予測できるMLにする必要がある

キーワード的には、やはり簡単なMLモデルをたくさん用意すること、
データが多ければ多いほど予想がうまくいくこと
継続的に改善していくためにどうすればいいかを考えること
が大事そうかなと思った