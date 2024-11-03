---
title: "Kyselyの利用と工夫と感想"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['kysely', 'TypeScript']
published: false
---

## Intro
TypeScriptのDBアクセスライブラリといえば、今はPrismaが第一に上がりそうだが、Kyselyを使ってみた。
なぜKyselyにしたか、どう使ったか、使ってみてどうだったか記載していく。

## DBアクセスライブラリの選定
選定なんて大層な単語を使っているが、選ぶのは人の考え方次第だ。
ただ、選ぶための基準となる考え方を、少し整理しておきたい。

### DBアクセスライブラリの種類
いわゆるパターンものなので、Pattern Of Enterprise Application Architectureをまず確認する。
- Table Data Gateway
- Row Data Gateway
- Active Record
- Data Mapper
- Metadata Mapping
- Query Object
- Repository

よくわからないのと、ほとんど生のSQLを書いて利用するというようなパターンが見当たらなかった。
改めて整理すると、以下のように分類しておきたい。
- Data Mapper
  オブジェクトとDBのレコードをマッピングし利用する。
  TypeScriptではPrismaのことをData Mapperと呼びたい
- Query Object
  QueryをJavaScriptから作っていくタイプのもの
  TypeScriptでは今回取り上げるKyselyが該当する
- SQL Template
  SQLに近い構文を書いてPlace Holderを埋めていくタイプのもの
  TypeScriptだとsqlcを指す
- Active Record
  Active RecordはData Mapperの派生で、マップしたオブジェクトにそのまま保存や取得メソッドが生えているタイプのもの

Active RecordはData Mapperの派生であり、DBアクセスするという以上に様々な機能があり、DBアクセスの文脈だけでは言及がしづらい。
基本的にはData Mapperと同じなので、そちらと同じとみなして取り扱う。

良し悪しではなく、サンプルコードを見せてどういうものか、認識を揃えたほうがいいか

### SQLの種類




## DBアクセスの種類
- data record pattern?
- 他にもパターンあるはず。gof?
- repository
- active record

それぞれの良し悪し

- DBレイヤでどこまでするか
  - すなおに投げるだけ
  - ちょっとデータの整形とかも
- jsで実装するか
  - js
  - dsl
  - json
- relationを表現するか
- sqlを意識するか

{table|row}gatewayが、純粋にレコードを取ってくる仕組み。
data mapperというのは、処理のためにデータを取ってくる。
両者の間にはギャップがあって、後者のほうがよりアプリケーションのために整形とかいろいろするが、前者はただただ取ってくるだけ。
active recordはdata mapperと比較する概念で、オブジェクト自体が多機能であるか、dbアクセスが分離されているかの違い。
query builderは、queryを組み立てるという意味で、active record上でも、data mapper上でも使えるはず。
data mapperの代表例はrepositoryになるはず。
という整理
あとはsql templateもある

active recordはこき下ろす。

## ライブラリ
- prisma
- drizzle
- kysely
- knex

上記の整理の上で、repositoryとactive record、query builderなどを組み合わせて、ライブラリが成り立っている。
一つの概念では開発者体験を網羅できないので、その組み合わせ自体から評価できるはず

## sql
1. テーブルにprimary keyでアクセス
2. join
3. where句へのややこしい条件
4. group by
5. 動的なクエリ

where句へのややこしい条件がある場合は、joinが必要なことも多い

kyselyでwhere句をややこしくできるようなutilityを用意するなら、relationの仕組みもいる。
utilityとしては、1のみ実現できればよい。

別軸として、チューニングするか否かちう区別もある

## utility
実装と利用。ts-ignore

## DI
DIの記事を参照
散らばった関数をユースケースで集めてbindするという発想
transactionの管理をユースケースで行う
kyselyに限らず、リポジトリではなく、独立したsql queryを管理するという考え方において有効

## 分割管理
kyselyで
- しづらいがsqlって分割管理しづらいので問題ない
- ただ、utilityを作るのに難しい
  - ts-ignore満載

### 型がややこしい
aliasを付けると、定義したテーブルの型をコピーしてaliasを同じ型として定義する。
つまりクエリが長くなるほど、型定義がややこしくなる。手で書ききるのは難しい。

## その他
transaction管理が例外のみ
早く入ってほしい
https://github.com/kysely-org/kysely/pull/962

## outro

