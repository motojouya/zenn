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

上記は、あくまでソフトウェアアーキテクチャ上のパターンであり、SQLを投げる以上の意味合いがある。
純粋にSQLを投げるという意味では、以下の3のパターンに絞られるのではないか。まず以下について確認していく。
1. JSON Setting
2. Query Object
3. Raw SQL

#### JSON Setting
JSON Settingは筆者の造語だが、JSONでSQLを表現するタイプのものだ。
TypeScriptで代表的なライブラリはPrismaだろう。以下はPrismaのGetting Startedのページにあるサンプルだ。

https://www.prisma.io/docs/getting-started/setup-prisma/start-from-scratch/relational-databases/querying-the-database-typescript-postgresql
```ts
async function main() {
  await prisma.user.create({
    data: {
      name: 'Alice',
      email: 'alice@prisma.io',
      posts: {
        create: { title: 'Hello World' },
      },
      profile: {
        create: { bio: 'I like turtles' },
      },
    },
  })

  const allUsers = await prisma.user.findMany({
    include: {
      posts: true,
      profile: true,
    },
  })
  console.dir(allUsers, { depth: null })
}
```

上記のようにQueryを投げる前に、DBのテーブル同士のリレーションやスキーマ情報の登録は事前に行う必要がある。
その後は、上記のようにJSONを指定すると、PrismaがよしなにSQLを発行してくれる。というタイプのものだ。

SQLのシンタックスは、文になっており、JSONで書くように設定のようになっていないので、JSONのほうが直感的だと感じるプログラマも多いだろう。
JSONなので、TypeScript上で取り回ししやすく、Queryの発行内容もif分岐で多様に変化させやすい。
反面、SQLのシンタックスとはだいぶ違うものなので、どのようなSQLが発行されているか想像しづらく、また最適なSQLになっているかわからないという問題点もある。

#### Query Object
SQLを表現するためのObjectを取り回しながらSQLを組み立てていくタイプのものだ。
今回とりあげるKyselyはこちらになる。

https://kysely.dev/docs/getting-started
```ts
import { db } from './database'
import { PersonUpdate, Person, NewPerson } from './types'

export async function findPersonById(id: number) {
  return await db.selectFrom('person')
    .where('id', '=', id)
    .selectAll()
    .executeTakeFirst()
}

export async function findPeople(criteria: Partial<Person>) {
  let query = db.selectFrom('person')

  if (criteria.id) {
    query = query.where('id', '=', criteria.id) // Kysely is immutable, you must re-assign!
  }

  if (criteria.first_name) {
    query = query.where('first_name', '=', criteria.first_name)
  }

  if (criteria.last_name !== undefined) {
    query = query.where(
      'last_name',
      criteria.last_name === null ? 'is' : '=',
      criteria.last_name
    )
  }

  if (criteria.gender) {
    query = query.where('gender', '=', criteria.gender)
  }

  if (criteria.created_at) {
    query = query.where('created_at', '=', criteria.created_at)
  }

  return await query.selectAll().execute()
}

export async function updatePerson(id: number, updateWith: PersonUpdate) {
  await db.updateTable('person').set(updateWith).where('id', '=', id).execute()
}

export async function createPerson(person: NewPerson) {
  return await db.insertInto('person')
    .values(person)
    .returningAll()
    .executeTakeFirstOrThrow()
}

export async function deletePerson(id: number) {
  return await db.deleteFrom('person').where('id', '=', id)
    .returningAll()
    .executeTakeFirst()
}
```

JSON Settingと比較すると、発行されるSQLが想像しやすい。
ただ、JSON Settingより冗長な記載になりがちだ。
JSON Settingと同様、TypeScriptロジックで分岐して、多様なSQLを発行しやすい。

#### Raw SQL
生のSQLを書いて実行するタイプ。
これは実のところ、PrismaにもKyselyにも機能的に備わっているが、生SQLと言えばsqlcが該当するだろう。

https://github.com/sqlc-dev/sqlc-gen-typescript
```sql
-- name: GetAuthor :one
SELECT * FROM authors
WHERE id = $1 LIMIT 1;

-- name: ListAuthors :many
SELECT * FROM authors
ORDER BY name;

-- name: CreateAuthor :one
INSERT INTO authors (
  name, bio
) VALUES (
  $1, $2
)
RETURNING *;

-- name: UpdateAuthor :exec
UPDATE authors
  set name = $2,
  bio = $3
WHERE id = $1;

-- name: DeleteAuthor :exec
DELETE FROM authors
WHERE id = $1;
```

上記の利点は、本当にSQLを書く感覚で記載できるという点だ。
生のSQLに近い形であれば、sqlファイル上で、if分岐を入れてロジックを表現し、多様なSQLを発行できるタイプのものもある（TypeScriptであるかは不明）。
ただ、sqlcは本当に生のSQLに近い形のようで、動的にSQLを組み立てる機能は備わっていないようだ。
おそらく、上記の文字列に、何らかの文字列を連結して実行することは可能なはずで、動的SQLは実現できるが、機能外でのケアのようだ。
sqlcについてはドキュメントのみの情報なので、間違っていたら指摘いただきたい。

### SQLの種類
SQLをどう表現するか、ライブラリによる違いについて確認したが、どのようなSQLがあるかも整理しておきたい。
DDL、DML、Select文、Update文などもあるが、そういったわかりやすい分類においては、どのライブラリでも機能がある。

以下のように考えたい
1. 単一のテーブルに対するPrimary KeyあるいはUnique Keyによる操作
2. 複数のテーブルをJoinし、Where句に条件を指定しての操作
3. Group Byなどの集約条件を含んだ操作
4. 限定的な機能を用いた操作

1,2,3は自明だろう。4は実のところ筆者も想像できない。とにかく難しいSQLとだけ認識しておいてほしい。詳しくは後述する。
上記の4つに分けたのは、SQLを各種ライブラリで実現する際の難易度の順番だ。

#### 1. 単一のテーブルに対するPrimary KeyあるいはUnique Keyによる操作
これは最も簡単な操作だ。どのライブラリでも1行で表現できる。
特に、特定のリソースに対して、更新、削除などする際に表れる。
特定のリソースを`select * from TABLE where id = ?`で取得し、`update TABLE set COLUMN = ? where id = ?`などで更新する。

#### 2. 複数のテーブルをJoinし、Where句に条件を指定しての操作
JoinもWhere句も追加され、1と比較して、2step難易度が上がっているように見えるが、実用上のパターンを検討した上でこうなっている。
1のパターンでは、一つのテーブルに存在確認だけすればよいというケースが多いはずだ。

細々した条件をWhere句に指定する際、えたいデータがあるテーブルだけで完結するケースは少ない。
また逆に、複数テーブルをjoinして、いずれかのテーブルのprimary keyでデータを取得する際も、他のテーブルには細々としたwhere句条件をつけるパターンも多いはずだ。
まったくないとは言わないが、JoinもWhere句も、同時に扱わなければならないものになるというのが、筆者の経験的な感覚だ。

#### 3. Group Byなどの集約条件を含んだ操作
3は純粋にGroup Byで集約条件が入っているものだ。
ライブラリによっては、テーブル定義をアプリケーション上に記載し、参照して型チェックするが、集約した結果のテーブルは情報がSQLにしかない。
当然扱いづらいはずだ。

#### 4. 限定的な機能を用いた操作
こちらは存在するかわかっていないが、ある想定は持っていたほうがいいだろう。
具体的にはJSON SettingやQuery Objectでは表現しきれないような、ややこしい、あるいは特定のRDBの機能に特化したSQLだ。
経験的に筆者は扱ったことはないので存在するのかわからないが、Raw SQLでしか扱えないクエリというのは、あってもおかしくないだろう。

#### 動的SQL
また、上記4つの分類ではなく、動的に組み立てるか、宣言的に配置して扱えるかという分け方もある。
こちらも、各種ライブラリを評価する上では抑えておきたい点になる。

### 比較
TypeScriptのDBアクセスライブラリは以下のようなものがあるだろう。
- Prisma
- Drizzle ORM
- kysely
- sqlc
- Micro ORM
- Type ORM

けっこうある。上記はTypeScriptで記載されているものに絞っているので、JavaScriptで記載されTypeScriptの型情報が存在するものにまで広げると、もっと挙げられる。
Sequelizeも半分ぐらいはもうTypeScriptのようだが、ここでは除外した。

上記のものは、JSON Settingがメインだったり、Query Objectがメインだったりもするが、複数の複合であることが多い。

- Prisma
  JSON Setting + Raw SQL
- Drizzle ORM
  JSON Setting + Query Object
- kysely
  Query Object
- sqlc
  Raw SQL
- Micro ORM
  JSON Setting + Query Object
- Type ORM
  JSON Setting + Query Object

厳密にはRaw SQLはどのライブラリでも扱えそうだが、実用上多用しそうな印象のあるもののみ記載した。
こうしてみると、JSON Setting + Query Objectなライブラリが多い。
上記ライブラリの違いを見ると、リレーションの設定があるか、またActive Record Patternで利用できるのかと言った違いもあるが、今回は言及しない。

JSON Setting、Query Objectは動的SQLに対応しやすく、Raw SQLは難易度の高いSQLに対応し易いはずだ。
そう考えると、Prismaはどちらの機能も抑えており、様々な場面に対応しやすそうだ。

JSON Settingの良さは扱いの簡単さだと考えているが、SQLをチューニングしたくなった際にはどうだろうか？
発行されるSQLがわかりやすいほうがいいはずだ。これは、どれもQuery ObjectなりRaw SQLなりの機能を備えているので、チューニングにも対応できそうだ。

では、動的に組み立てているSQLをチューニングしたくなった際にはどうだろうか。
動的に組み立てているSQLがパフォーマンス上の問題を抱えやすいわけではないが、経験的には動的に組み立てているSQLをチューニングしたことは多い。
これは動的にSQLを組み立てる機能は、アプリケーション上の検索機能で利用され、検索機能はパフォーマンス負荷がかかりやすいという面があるからだ。
もちろんこれを、転置インデックスに変更したり、CQRS構成に変更したりということは考えられるが、できるならチューニングでパフォーマンスが上がったほうが安上がりではある。

PrismaはJSON Setting側はSQLは把握しづらく、Raw SQLは動的SQLが組み立てづらい。両立した機能ではないというのは抑えて置きたい。
検索は複雑なものにならないだろう、あるいは複雑な検索があっても、一つしかないので、都度リソースを投入して直せばいいという考え方もある。
これは作るアプリケーションの特性にもよる。筆者はキャリア的にtoBのシステムが多く、検索画面が複数あり、常に数個のテーブルをJoinしながらsqlにして数百行みたいなプロジェクトは多く経験しているので、Prismaに対して、機能不足という感想を持っているが、そんなアプリケーションどこにあるの？みたいな感覚のプログラマもいるだろう。
それぞれ作るアプリケーションの特性を鑑みて選べばいいだけだ。

今回は筆者はKyselyを採用した。上記の背景を検討したうえで、Query Objectが最も対応力が高いと判断したためだ。
4.特定DB特有のクエリというのは、今回は想定されないので、その部分については懸念していない。もし必要になったとしてもRaw SQLで対応できるだろう。

また、筆者はSQLに慣れているので、特にRelationの設定や、Active Record Patternのような重厚な仕組みは不要で、すべてSQLで完結できる。
SQLで完結できるので、宣言的に記載されたQuery Objectを見れば、どんなSQLを発行しているのか把握でき、コールドリーディング上も利点がある。

## 使い方

以降では、Kyselyの使い方や、使う際の工夫について記載していく。
基本的な使い方についてはKyselyのドキュメントを参照してほしい。

どちらかというと、工夫した内容や、仕様上迷った部分などに言及していく。

### 仕組み

### utility
実装と利用。ts-ignore

### SQL単位でファイル管理
DIの記事を参照
散らばった関数をユースケースで集めてbindするという発想
transactionの管理をユースケースで行う
kyselyに限らず、リポジトリではなく、独立したsql queryを管理するという考え方において有効

### SQLの分割管理
SQL自体を組み合わせるかどうかという点。
複数のSQLを一つのファイルに入れないというのは上で。

kyselyで
- しづらいがsqlって分割管理しづらいので問題ない
- ただ、utilityを作るのに難しい
  - ts-ignore満載

aliasを付けると、定義したテーブルの型をコピーしてaliasを同じ型として定義する。
つまりクエリが長くなるほど、型定義がややこしくなる。手で書ききるのは難しい。

### Transaction
transaction管理が例外のみ
早く入ってほしい
https://github.com/kysely-org/kysely/pull/962
現在0.27.4だが、0.28.0入りそう

### Now()

## Outro

