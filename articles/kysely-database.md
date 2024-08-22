---
title: "Kyselyを使ってみた"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['kysely', 'typescript']
published: false
---

## intro

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

## ライブラリ
- prisma
- drizzle
- kysely
- knex

## sql
1. テーブルにprimary keyでアクセス
2. join
3. where句へのややこしい条件

where句へのややこしい条件がある場合は、joinが必要なことも多い

kyselyでwhere句をややこしくできるようなutilityを用意するなら、relationの仕組みもいる。
utilityとしては、1のみ実現できればよい。

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

