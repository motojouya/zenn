---
title: "レビューの体系化について"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["レビュー", "設計書"]
published: true
published_at: 2022-12-20
---

この記事は[オープンロジアドベントカレンダー2022](https://qiita.com/advent-calendar/2022/openlogi)の20日目の記事です。
（本当は19日目でしたが、変わってもらいました。）

## 課題設定
そもそも人のコードを見るのはとても大変です。
コードに限らず、人の考え方を理解するというのは、非常に大変で面倒です。

そんな中で100ファイル超えのでかいPRが投げられるような状況でした。
マジでつらい。と思った自分は、それを改善することにしました。

まずは状況の整理です。
そもそも変更するファイル数や、行数がすくなければレビューするのがつらい状況を脱することができますが、そのゴールを達成するには、技術的には大規模なアーキテクチャの改変などが必要になってきそうです。
新しい機能の追加がなければいいとは思いますが、それではプロダクトの発展がありません。
ということは、レビューの量を減らすというのは難しそうに感じます。

では、量以外で何がつらいのでしょうか？それは一重にレビューの管理ができていないことでした。
100ファイルが一気に来ることで、そのレビューをするための時間の確保をし、レビューの背景を理解した上で、ポイントを抑えて見ていく。
ということをしなくてはなりません。

そこで、レビューを管理しやすいスキームに持っていく必要がありました。

## やったこと

一言で言うと、レビューを分割するようにしました。
というと簡単ですが、分割するための仕組みを作るのが、主にやったことです。

### 設計書
まず、設計フェーズというのを作りました。
現在では、PRDというフェーズもあり、設計作業の意味合いが薄いのですが、この課題があった当時は、何を作りたいのかというのがredmineのチケットにメモ程度しか書かれておらず、redmineかslack上でコミュニケーションをして機能イメージをすり合わせて行く必要がありました。

という背景もあり、設計といいつつ、機能の概要を決めていくような作業です。
具体的には、設計書で機能の概要を決めていくと、ユースケースが決まってきます。
一つの機能を開発する際に、複数のユースケースを定義していく場合に、これによってPull Requestをわけることができるようになります。

### コードリーディング
上記の設計書では、実装イメージをつけることができません。
そこを補うために、開発に入る前に、一緒にコードを読む時間を作りました。

新規作成する際は、似たような機能のコードを参照します。
口頭で伝えつつ、実装者に画面を共有してもらい、その場でメモをコード上に書いてもらうなどしていました。
これによって、レビュー時の実装イメージのずれを少なくし、またついでに改善できるようなリファクタリング可能な場所を示すなどしていました。

### モブプロ
これは現在進行系で進めているのですが、レビューする際のポイントを伝えるのに役立っています。
モブプロを実際の開発プロセスに組み込むところまでは進んでおらず、2時間の間に問題のあるコードをリファクタリングする形にしています。

## 効果
効果としては、レビューの回数は増えましたが、管理は非常にしやすくなりました。
一度にドカンとこず、小出しに来るので、逐次空いた時間にレビューができるようになりました。

また、設計書を作り、設計書のレビューをすることで、全体の整合性の確認と、コード上の実装イメージを持った上でコードレビューに望みやすくなりました。
また、コードリーディングやモブプロをすることで、事前に認識を合わせることで、ずれが少なくなったように感じます。（それでも思ってたのと違うのが来ることはありますが）

### 副次的なやつ
また、副次的な効果として、開発プロセス自体の管理もしやすくなりました。
設計フェーズという、面倒なプロセスも増えたのですが、設計段階で実装のステップを定義することができたので、実際の開発にどれくらいかかるのか、また現在どのステップまで開発が進んでいるのか観測しやすくなりました。

これによって、いままで一人の人間に一つのタスクを丸投げしていたのが、複数で作業しやすくなりました。

## 課題
コードリーディングの部分である程度補っているのですが、ソフトウェアアーキテクチャの検討が不十分に感じます。
既存のコードを直す際にはいいのですが、新規で新しいドメインを作り込む際には検討が必要なはずです。
レビューのコストを下げつつも、その部分も検討していくものが必要に感じています。

## まとめ＆感想
新卒のときはSIerだったので、設計書（とは名ばかりのメモ）は常にある状態だったのですが、スタートアップでは個人の努力で開発プロセスが担保されているのがよくある話だと思います。
オープンロジも10年近くなり、コードベースも大きく、新規開発のコストも少しずつ上がっていく中で、開発速度を一定以上に保つ努力をしていかなくてはなりません。
今後も、成果を出せるエンジニア組織でいられるように、がんばっていきたいなと思います。

## おまけ
思ったより、記事が短かったので、設計を書く際のガイドラインのwikiページを共有しておきます。


```
# 設計書ガイドライン

## 目的

- PRを分けるため。PRを分けるとビッグバンPRが生まれずレビューのリズムが取りやすい
- PRを分けるためには、PRをつなぐ設計資料が必要なため
- そこそこの規模の開発をする場合、チェックすべきポイントが複数あるので部分を切り出して確認していきたいため
- まとめると開発プロセス上でのコミュニケーションを円滑化するため

### 注意事項
あくまでも開発をする上で、スムーズなコミュニケーションを行うためのもの
したがって、開発者以外がわかりやすいようにしたり、永続的なドキュメントとして管理することを必須としない。
むしろ、可能な限り短期間で作成し、開発に早めに着手するほうが優先される。

### 含めないこと
要件定義で行われるべきものは、ここでは定義せず、事前に決まっていることが望ましい。

以下が例
- 機能の背景、目的
- 利用者
- 業務フローの定義
- ユースケースの定義
- 機能の概要

## 基本方針

タスク分解し、分解したタスクを管理していくため
結果として、分解したタスクごとにPRが切られることが望ましい。
レビューの観点においては、PRを分ける根拠としての資料であり、各PRに記載のしづらい情報が網羅されているべき資料であることを見る

### 設計の要否
PRを分ける必要が無いほど小さなタスクにおいて設計を行う必要はない。

基本的には以下のケースにおいては、設計を行うことが望ましい
- 既存機能の修正ではなく、新しい機能の開発である場合
- 画面の新規作成
- 業務フロー上の複数のユースケースにまたがる開発
- 既存機能を置き換えるような開発のケース
- ファイル数が10を超える(ただしこれは目安として)

### 利用メディア
githubのPRで承認フローを定義し、本文はwikiに記載することが望ましい
理由は以下
- wikiで編集履歴を確認できる
- wikiでページを複数用意することで、構造的なドキュメントを作成できる
- github上で記載すると構造的な文章は書けないが、承認フローは開発のやり方で行える

### レビュー
設計のレビューは、初回は対面で行うことが望ましい。
必要な知識や認識に差が出ることが多いため、その場でコミュニケーションしやすいため
また、コードのレビュアは、概要の把握のために設計レビューに参加している必要がある。

### 見積もり
各PRに対してどれくらいかかるかで、終了見込みを立てる。
PR作成以外の作業も見積もって置くと望ましい。
これは、作業計画を立て、scrum的に日々の進捗を感じるための工夫として

### 設計の振り返り
PRを分ける、つまり作業単位を定義することが設計の目的ではあるが、数週間かかるような開発タスクでは、定義漏れも生まれやすい。
隔週や月一程度でよいので、開発がある程度進んだ段階で、設計の漏れや作業の再定義などを行うことを推奨する

### PRの分割の仕方
この時PRの分割としては、いくつか考えられる

- 画面
- ユースケース
- ドメイン
- アプリケーションレイヤ

これらを適切に組み合わせて分割したい。
が、基本的にユースケースが別れる場合は、別のPRが望ましい。

#### ドメイン

ドメインは複雑なものが存在するときや、スキーマ設計を別途行う場合などに別のPRであると望ましい。
複数のユースケースに関わって、重要なドメインがある場合は、そのドメイン部分の開発のみ別PRに切り出すなども考えられる。

#### アプリケーションレイヤ
アプリケーションレイヤで言うと、eloquentやserviceなどのドメインレイヤ、ジョブやcontrollerなどのアプリケーションレイヤ、またフロントのレイヤなどがある。
これらも、分量が多ければ、それらのレイヤごとにPRを分けることもできる。
ただ、レイヤよりも、ユースケースが別れているほうが、PRとして凝集度が高くなる傾向があると考えているので、そのほうが望ましい。

#### 画面
ユースケースよりも画面の単位のほうが大きくなるケースが多いため、ユースケースごとにPRが別れれば画面単位でPRを分けることを考える場面は少ないはず。
ユースケースでわけるほどでもない場合に、画面単位でPRが別れることは有り得る。

## 設計資料

### 画面

画面設計には以下が記載されていることが望ましい。

- 表示項目
- アクション
- 表示項目の編集方法
    - バリデーション
- 画面上のモード変化
    - これはバックエンドにリクエスト中や、データの更新中などの、画面の状態をパターン化する
- 画面遷移
    - 複雑な場合にのみ
- 画面の具体例は必須ではないが、複雑な表示であればワイヤーフレームは必要になるケースが多いはず

ワイヤーフレームが必要な場面は多いと考えられるが、画面イメージが精密に決まっている必要はない
ただし、インフォメーションアーキテクチャの観点で、ふさわしいデータ構造を画面上で表現できるかは検討したほうがよい。
とくにDBのリレーションと違うものが望ましい場合は、データ構造の定義が必要となる

### ユースケース

- 誰が利用するか
- 操作者がどんな場合に利用するか
- インプットパラメータ
    - 詳細まで書く必要はない。例えば、住所なら郵便番号と書く必要はない
- それによるシステムの状態変化。あるいは結果
- 背景
    - 込み入った背景がある場合に

ユースケースが違う場合は、ジョブは分けたほうが望ましい。
その上で、複数のユースケースで利用されるべきロジックは、サービスやeloquentモデルなど、ドメインロジックに定義すべき。

### ドメイン

- DBのスキーマ設計
    - カラムの細い部分は不要
    - テーブルの分割やリレーションは必要
    - 重要なカラム
- ロジックの説明
    - いくつかパターンがあるのであれば、パターンの図示やマトリクスなどを用いて表現する
    - あんまり複雑じゃなければ不要

### 非機能要件
ユースケースに対して設定する
ロギング、アラート、パフォーマンスなど
バッチであればタイミング、実行時間など
その他、その開発に特有の資料があれば、添付する
```
