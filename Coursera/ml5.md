# Googleの機械学習の取り組み

## MLの意外性

やはり、MLで大事なのは、MLのトレーニングに集中するのではなくて、
適切なデータ収集に力を入れたほうがいいということ
MLのインフラを構築すると、自動で繰り返しにモデルをトレーニングできるので、
KPIの定義に費やす時間などが短縮される

## 秘訣

- やはりデータを集めないうちはMLを語る資格がない
- MLシステムのトレーニングのループにしっかりと人員を配置する
- 間違った最適化を行って、悪いコンテンツを提示する可能性があるので、しっかりみはっとく
- 他のMLシステムを見て、作れそうと勘違してはいけない
- 画像認識、音声認識とかの認識機能を自社製で作ろうとしなくてよい
- MLの作成はめちゃくちゃ大変やけど、一回完成すれば他の会社はまねできないし、いろいろとシステムが改善していくはず


## MLのビジネスプロセス

非MLソリューションからMLソリューションに移行する方法

1. 一人の貢献者 : 社長1人による一つの仕事
2. 委任         : 一人でできなくなってきたので他の人に任せる
3. デジタル化   : 委任した仕事をマニュアル化して、繰り返しできるようになってきたら、それをデジタルでできるようにする
4. ビッグデータ : 自動化したシステムからえられるデータを収集して、改善案を考える
5. 機械学習     : えられたビッグデータを活用してMLをトレーニングし、改善作業も自動化する

↑この一連の流れを理解、実践する
一つのプロセスでも欠けるとよくない

## まとめ

- 最初からすごいMLを作ろうとしない データをしっかり集めながらじっくりやってく必要がある
- プロダクトと、フィードバックでの改善のループを意識 また、そのループはすべて序同化していくこと