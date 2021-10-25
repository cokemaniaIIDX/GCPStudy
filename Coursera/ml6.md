# インクルーシブML

## バイアス

人間の認識にはバイアスがかかっていて、
例えば、靴を想像してと言われたら、スニーカーを想像する人もいれば、ヒールを靴と想像する人もいる
機械学習でもこういうバイアスは発生する
MLは人間によって作成されたデータに基づいて学習しているから、
データに人間のバイアスがかかっているために、影響されてMLもバイアスがかかってしまう

物理学者がどんな顔をしているかという例を学習させるとすると、
歴代の物理学者の顔をデータとして読み込ませていくけど、
そうすると物理学者はほとんど男なので、男性に偏ってしまう

## 包括性の評価

MLの包括性(ちゃんとしてるかどうか？)を評価する際につかえる方法に、
`混同行列(confusion matrix)`がある

混同行列を使ってMLを評価する際、
「陽性positive」「陰性negative」を調べていく

たとえば、人間の顔を認識する時、
データのラベルとMLの判定をそれぞれチェックする
- 真陽性 : ラベルに顔があって、MLも顔と認識             (顔がはっきり見えている写真)
- 真陰性 : ラベルに顔がなく、MLも顔じゃないと認識       (動物の顔の写真)
- 偽陽性 : ラベルに顔がないのに、MLが顔と認識           (人間の顔の彫刻の写真)
- 偽陰性 : ラベルに顔があるのに、MLが顔じゃないと認識   (マスクや服で一部隠れた顔写真)

偽陰性、偽陽性が発生した場合、ここに注目してMLのトレーニングを進めていく

## 統計判断指標とトレードオフ

混同行列を作成したら、顔の認識以外にもいろいろな指標を計算してMLの精度を上げていく
MLの包括性を上げるためには、偽陽性(FP)率や偽陰性(FN)率を計算するという方法がある

偽陰性と偽陽性、どっちが低くなればいいか？
これは、場合による

精度悪いけど検出率が高いもの、
精度高いけど、検出率(検出数)が低いもの
どっちを使えばいいかはトレードオフの関係にある

画像のプライバシーのためにぼかすかどうかの判断するとすると、
偽陽性FP率が高い場合、ぼかし対象が多くなって不鮮明になる ← ぼかすべきと判断するご検出が多いため
偽陰性FN率が高い場合、ぼかし対象が少なくなってプライバシーがさらされる

スパムメールフィルターを例にとると、
偽陽性率が高い場合、スパム判定が多くなるので、友達や家族からのメールも削除されてしまう
偽陰性率が高い場合、スパム判定が少ないので、スパムが受信ボックスに来てうっとうしい

FP、FNどちらを取るかはトレードオフの関係なので、ちょうどいいところを模索していく

## 機会均等

Wikiによると、機会均等とは、
重要な仕事は最も「優秀な者」にゆだねられるべきであって、人種、信条、性別等の非合理的理由によって仕事の依頼をゆだねるべきではないということ

MLでの例 : ローンを許可するかどうかの判断
ローンの支払いの信用度を数値化してグラフにして、閾値を決める
閾値以下の人はローンを却下するように決める

実際の可否結果では、
返済不履行なのに許可された人や、履行可能なのにローン拒否された人とかが出てくる

ここで閾値の最善を考えると、
履行可能で、ローン許可される人の割合を最大にすることが必要

ただし、ローンの可否数に関して正しい決定の数を最大にする点以外にも考慮する場合がある

ローンの種類によっては、他のローンより収入が大きくなる場合がある
収入を最大化することに重きを置いた場合、最大の閾値を決めるとすると、また違う結果になってくる

ローン許可の正確な決定数の閾値と、利益最大化を考えた時の閾値は同じになっているか？
この質問に答えるのは難しい

ここで機会の均等を考える

正の出力の資格がある人は、それ以外のグループで正に分類される人と同じ機会が与えられる必要がある

例えば、アメリカでは年齢によって従業員を差別してはいけないらしいので、
各年代毎で、ローンの可否に、真陽性率が変わってしまってはいけない

ちょっと結局意味わからなかったけど、どのグループでも真陽性率が一致している必要があるらしい