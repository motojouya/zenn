---
title: "Kyselyの利用と工夫と感想"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Kysely', 'TypeScript']
published: true
---

## Intro
TypeScriptのDBアクセスライブラリといえば、今はPrismaが第一に上がりそうだが、Kyselyを使ってみた。
なぜKyselyにしたか、どう使ったか、使ってみてどうだったか記載していく。

## DBアクセスライブラリの選定
選定なんて大層な単語を使っているが、選ぶのは人の考え方次第だ。ただ、選ぶための視点を、少しだけ整理しておきたい。

### DBアクセスライブラリの種類
DBアクセスライブラリはいわゆるパターンものなので、Pattern Of Enterprise Application Architectureをまず確認する。DBアクセスに関するものは以下が挙げられそうだ。
- Table Data Gateway
- Row Data Gateway
- Active Record
- Data Mapper
- Metadata Mapping
- Query Object
- Repository

上記は、あくまでソフトウェアアーキテクチャ上のパターンであり、SQLを投げる以上の意味合いがある。

純粋にSQLを表現という意味では、以下の3のパターンに絞られるのではないか。まず以下について確認していく。
1. JSON Setting
2. Query Builder
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

SQLのシンタックスは文になっており、宣言的な設定のようになっていないので、JSONのほうが直感にデータを絞り込めていると感じるプログラマも多いだろう。実際に、タイピングの量は圧倒的に少なそうで、楽なのは間違いないだろう。
JSONなので、TypeScript上で取り回ししやすく、Queryの発行内容もif分岐で多様に変化させやすく、動的にSQLを操作することもできる。

反面、SQLのシンタックスとはだいぶ違うものなので、どのようなSQLが発行されているか想像しづらく、また最適なSQLになっているかわからないという問題点もある。
パフォーマンスチューニングの際は、どのようなクエリが発行されているか、ログなどから逆算して洗い出してチューニングし、またそれをJSON Settingに変換するということが必要だろう。
RDBのインタフェースはSQLだが、SQLをJSONで表現しようとしても、どうしても仕様上のギャップが出てしまう。

#### Query Builder
区別のためにQuery Builderとしたが、この記事の上ではPoEAAのQuery Objectと同等と考えて問題ない。
SQLを表現するためのObjectを取り回しながらSQLを組み立てていくタイプのものだ。
今回とりあげるKyselyが該当する。

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

JSON SettingではRelationの設定などはあったが、それはQuery Builderでは不要だろう。
ただKyselyについてはテーブルの型は事前に用意しなくてはならない（generationツールがある）。

JSON Settingと比較すると、発行されるSQLが想像しやすい。パフォーマンスチューニングはしやすそうだ。
ただ、JSON Settingより冗長な記載になりがちだ。SQLが苦手なプログラマからするとSQLを覚えてからQuery Builderの使い方を覚えるので、学習コストが高いだろう。
TypeScriptロジックで分岐して、多様なSQLを発行しやすいのは、JSON Settingと同様だ。

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

上記の利点は、本当にSQLを書く感覚で記載できるという点だ。Query BuilderのようにQuery Builderの書き方を覚える必要もない。
また、JSON SettingやQuery BuilderはTypeScript APIで提供されていないRDBの機能は使えない。Raw SQLならば、SQLでしかないので、RDBの制約をそのまま引き継げ、使えるはずだ。

sqlcには動的にSQLを組み立てる機能は備わっていないようだ。SQL自体にそんな機能はないので、素直にSQLを記述するだけならsqlcにも機能はないだろう。
ただ、機能外で文字列をつなぎ合わせて実現している例は見たので、まったく何もできないわけではなさそうだ。

生のSQLを記載するタイプでも、独自にsqlを拡張したDSLを記載して、sqlファイル上でif分岐やforループを実現するタイプのものも存在する。
ただ、筆者がみたことがあるものはjs,tsではなくJavaのものだ。このタイプのものは独自DSLを覚える学習コストが高いと見るべきだろう。

つまり、Raw SQLは動的にSQLを組み立てるのは苦手か、あるいは独自のDSLを覚える必要がある点が難点だろう。

### SQLの種類
SQLをどう表現するか、ライブラリによる違いについて確認したが、どのようなSQLがあるかも整理しておきたい。
DDL、DML、Select文、Update文などもあるが、そういったわかりやすい分類においては、どのライブラリでも機能がある。

ここでは、SQLに対して、2軸の整理をしておきたい。

- 動的にSQLを組み立てるか否か
- SQL自体の実現難易度

前者は自明だろう。後者は以下のように分類したい。以降で解説する。

1. 単一のテーブルに対するPrimary KeyあるいはUnique Keyによる操作
2. 簡易な操作
3. 複雑でチューニング欲求が高まりそうな操作
4. 限定的な機能を用いた操作

#### 1. 単一のテーブルに対するPrimary KeyあるいはUnique Keyによる操作
これは最も簡単な操作で、筆者の経験的には、特定のレコードに対して、更新、削除などする際に表れる。
特定のレコードを`select * from TABLE where id = ?`で取得し、`update TABLE set COLUMN = ? where id = ?`などで更新するタイプのものだ。

簡単な操作なので、どのライブラリでも実現できるだろう。ただし、JSON Setting以外はどうしても冗長に見えるはずだ。
Prisma: `const user = await prisma.user.findUnique({ where: { id }});`
Kysely: `const user = await kysely.selectFrom('user').where('id', '=', id).selectAll().execute();`

評価としては以下だろうか
- JSON Setting: 得意
- Query Builder: 可能
- Raw SQL: 可能

#### 2. 簡易な操作
簡易な操作という表現は曖昧だ。より具体的には、チューニング欲求が高まりそうにない操作と定義したほうがいいだろう。
つまり2と3の違いは、チューニング欲求が高まりそうか否かだけの違いだ。

チューニング欲求が高まりそうというのも曖昧な表現だ。そんなもの見たこともないというプログラマもいるかもしれない。
例えば、2,3のテーブルをそれぞれの主キー外部キーでJoinし、where条件もintegerカラムでの大小や、日付カラムでの期間指定のみ。というようなSQLはチューニング欲求は高くないだろう。
だが、テーブルのJoinする外部キーの候補が複数あったり、Joinするテーブルが集約カラムだったり、select句にCase文が入っていたり、where条件もandとorのネストが5重になっていたりした場合、将来的にチューニングしなければならない可能性は高まるだろう。

簡易な操作のSQLは、JSON Settingで実装しやすく、Query Builderでは冗長に感じるはずだ。
書く際のタイピングの手間で言えば、絶対にJSON Settingのほうが楽だ。
チューニングにする必要もないほど自明なSQLなら、JSON Settingで書かない理由がない。

評価
- JSON Setting: 得意
- Query Builder: 可能
- Raw SQL: 可能

#### 3. 複雑でチューニング欲求が高まりそうな操作
3は、2より難しい操作だ。
チューニング欲求が高まりそうだと言っても、必ずパフォーマンスチューニングするわけではない。ただし、複雑な問い合わせであればあるほど、発行されるSQLがどのようなものか把握したいだろう。
PrismaでJSONから変換されるsqlはだいぶ良くなってきていると聞いたが、それでも直感的にどのようなsqlが発行されているかわかるQuery BuilderやRaw SQLのほうが信頼性が高い。

評価
- JSON Setting: 苦手
- Query Builder: 可能
- Raw SQL: 可能

#### 4. 限定的な機能を用いた操作
標準的なSQLの機能ではない操作。特定のRDBに限った仕様も存在する。
Raw SQLは当然このあたりを扱えるはずだが、TypeScript APIを用意する必要があるJSON SettingやQuery Builderでは実現が難しい領域だ。

ただ実のところ、筆者はそういう機能を使った経験がない。特定のRDB固有の機能を使うプロジェクトは存在するが、こういった状況を想定するかは、プロジェクト次第だろう。

評価
- JSON Setting: 苦手
- Query Builder: 苦手
- Raw SQL: 可能

### 比較
TypeScriptのDBアクセスライブラリは様々あり、JSON Settingがメインだったり、Query Builderがメインだったりもするが、複数の機能の複合であることが多い。

- Prisma
  JSON Setting + Raw SQL
- kysely
  Query Builder
- sqlc
  Raw SQL
- Drizzle ORM
  JSON Setting + Query Builder
- Micro ORM
  JSON Setting + Query Builder
- Type ORM
  JSON Setting + Query Builder

けっこうある。上記はTypeScriptのみで開発されているものに絞っているので、JavaScriptで開発されTypeScriptの型情報が存在するものにまで広げると、もっと挙げられる。
また、厳密にはRaw SQLはどのライブラリでも扱えそうだが、実用上多用しそうな印象のあるもののみ記載した。

すべての言及するわけにはいかないので、ここからは2024年後半でTypeScriptで最も勢いのあるPrisma（という認識だがどうだろうか）と、今回取り上げるKyselyに注目する。
動的に組み立てないSQLについては以下の表のように考ればよいだろう。

|            | Prisma | Kysely |
|------------|--------|--------|
| 1.主キー   | 得意   | 可能   |
| 2.簡易     | 得意   | 可能   |
| 3.複雑     | 可能   | 可能   |
| 4.限定機能 | 可能   | 可能   |

これは、今までSQLの難易度カテゴリの得意不得意と、Prisma、Kyselyの機能から導いたものだ。
Prismaは1,2はJSON Settingで対応するので`得意`だが、3,4はRaw SQLで対応するため、`可能`としている。kyselyは4のみRaw SQLとなるがすべて`可能`になる。
動的に組み立てないならPrismaのほうが圧倒的だ。

ただ、これが動的に組み立てるSQLならばどうだろうか。
1は動的に組み立てることはないだろうし、4で動的に組み立てることを実現するにはいずれにしろコストがかかるので検討対象としない。2については、PrismaはJSON Settingで柔軟に対応できるが、3については、JSON Settingではチューニング欲求にたいして機能が不十分だし、Raw SQLでは動的に組み立てることに対して不十分だ。

つまり、3.複雑でチューニング欲求が高まりそうな操作のSQLを、動的に組み立てる場合は、Kyselyのほうが優位であるはずだ。

### 評価
PrismaとKyselyを比較してみたが、比較軸は他にもある。TypeScriptの型による支援がどこまであるか。APIが使いやすいか。対応しているSQLの幅。Migrationのやりやすさ。
そういったものをすべて比較して評価する必要があるが、今回着目したのは、動的にSQLを組み立てるか否か、SQLの実現難易度の2軸だけだ。
また、他のDBアクセスライブラリが気になるのであれば、上記の表に書き加え、動的SQLはどうか検討してみてほしい。

そして、その2軸であっても、読者によって印象は様々だろう。
SQLの実現難易度で設定した3.複雑でチューニング欲求が高まりそうな操作も、PrismaのJSON Settingで十分対応できるという認識の方もいるだろう。そもそもそういったクエリはチューニングするのではなく、転置インデックスやCQRS構成に変更すべきだし、そのための開発余力も取ってあるという方もいるかもしれない。チューニング欲求という表現をしたが、欲求であって実際にチューニングをしなければならない状況に陥るかは、事業の成長や、インフラコストなど様々な要因も絡んでくる。

したがって、挙げた2軸についても、アプリケーションの特性によって選ぶべきだろう。
動的に組み立てるSQLが複数あり、複雑になる可能性はあるのか？あるいは開発余力はあるが、開発優先度的にとにかくリリースしたいのか。
それらによって決めればよい。多くのケースでPrismaが最適という結論もでるだろう。

ただ、筆者は動的に組み立てるSQLに対してパフォーマンスチューニングをしたことが何度もある。そしてPrismaの開発者が素晴らしいことは知っているが、JSON SettingからSQLを組み立てるライブラリ機能を手放しで信じるというのは怖い。
また、一つのアプリケーション上に、動的に組み立てるSQLが両手以上の数あるプロジェクトも経験しているし、それらのSQLに対して複雑さを感じている。これは動的にSQLを組み立てる機能は、アプリケーション上の検索機能で利用され、検索機能はパフォーマンス負荷がかかりやすいという面があるからだ。
筆者は、Kyselyを選ぶべきだと結論づけるべきプロジェクトも多く存在するという肌感がある。

読者には、上記2軸以外、またPrisma、Kysely以外も検討した上で、適切なライブラリ選択をしてほしい。

今回、筆者はQuery Builderを覚えればだいたいのことが出来てしまう対応範囲の広さや、Kyselyの型のきかせ方などが好ましいと考え、Kyselyを覚えたくて使った。
Query Builderの機能性で言えば、上記で挙げたいくつかのライブラリの中でも群を抜いている印象がある。

## 使い方

以降では、Kyselyの使い方や、使う際の工夫について記載していく。
ただ、基本的な使い方についてはKyselyのドキュメントを参照してほしい。どちらかというと、工夫した内容や、仕様上迷った部分などに言及していく。

### 仕組み
Kyselyを使う上で、ちょくちょ迷うことがあった。迷わないようにするために、Kyselyの仕組みをある程度知っていたほうがいいだろう。

Kyselyの内部に注目すると、以下のステップで実行されていることがわかる。
1. Query Builderの組み立て
2. Query BuilderからPrepared Statementに変換
3. 実行

1のQuery Builderで組み立てる部分は当然アプリケーションを書くプログラマがやる部分だ。Query Builderで指定していく。
2,3は、`kysely#executeQuery`の実装を見るとわかるのだが、compileされてPrepared Statementに変換された後、executeされる。
https://github.com/kysely-org/kysely/blob/master/src/kysely.ts#L377

execute関数は他にもあるので`kysely#executeQuery`だけではないが、他でも同様のことをやっている。

プログラマが最も意識すべきは、1で組み上げるQuery Builderがどうなっているかだろう。
大前提として、以下のページに示されている通り、スキーマを表現する型を用意して置かなければならない。
https://kysely.dev/docs/getting-started#types

ただ、自分で書かなくてもgeneratorツールがあるので、Migrationと二重管理になることはないだろう。
これで各テーブルの型が手に入り、これがすべての型の基になる。

以下は、特定のユーザが投稿できるか、また画像をアップロードできるかの権限を管理するRoleテーブルとJoinするクエリだ。
内容を見ると、`From -> select -> where`の順に書かれているのがわかる。select句、where句の順序はどちらでもいいが、kyselyではfrom句は最初に定義しなくてはならない。これは、from句で定義したaliasの型定義を利用するためだ。
以降の解説はコードコメントを参照してほしい。

```ts
import { Kysely } from "kysely";
import { Database } from "./databaseType"; // テーブルの型定義で、クエリを記述する前に定義しておくもの

type UserWithRole = Omit<Database['user'], 'name'> & {
  role_name: string;
  role_post: boolean;
  role_post_file: boolean;
}

export type GetUser = (db: Kysely<Database>) => (userId: string) => Promise<UserWithRole[]>;
export const getUser: GetUser = (db) => async (userId) => {

  return db
    .selectFrom("user as u") // Database型の内部にあるuser型を、uという名前で再定義している
    .innerJoin("role as r", "u.role_id", "r.role_id") // Database['role'] -> Database['r'] をし、u.role_id, r.role_idがどちらも同じ型であることを型チェック
    .select([
      "u.user_id as user_id", // u型はuser型から引き継いでuser_idというpropertyがあることを知っている
      "u.name as user_name", // u.name型は、クエリの結果(仮にResultとするが)、Result.user_nameに引き継がれる
      "u.created_date as created_date",
      "u.updated_date as updated_date",
      "r.name as role_name",
      "r.post as role_post",
      "r.post_file as role_post_file",
    ])
    .where("u.user_id", "=", userId) // '=' も内部的にリテラル型で定義されており、u.user_idと変数userIdが同じ型であることを検査している
    .execute();
};
```

型の整合性を保ったり、型情報からproperty名を提案したりというのが、上記の仕組みを持って実現されている。
Kyselyのトップページの映像は、こういったタネがある。
https://kysely.dev/

ちなみにテーブル指定する際にaliasを使っている部分の型定義がどうなっているか気になる方もいるかもしれない。
以下の`AnyAliasedTable<DB>`型で分割されて管理されている。
https://github.com/kysely-org/kysely/blob/master/src/parser/table-parser.ts#L28

### Utility
SQLの難易度定義において1.単一のテーブルに対するPrimary KeyあるいはUnique Keyによる操作が、Query Builderでは冗長になってしまうことはすでに述べた。
2の簡易な操作についても冗長になるケースは考えられるが、それを支援するツールを作成しようとすると、それはほぼほぼJSON Settingの実装になり、それならばPrismaを使おうとなる。
したがって、1についてのみ補う関数を用意した。

また、insert文やupdate文の発行時に現在時刻を設定するための関数というのが、各種RDBには備わっていることが多いし、経験上、利用頻度も高い。
今回はsqlite3を利用しているので`datetime('now', 'localtime')`になるのだが、これはRDB内の関数なので、Kyselyにはこれを表現するものが存在しない。
こちらも用意した。

```ts
import {
  Kysely,
  Insertable,
  Selectable,
  Updateable,
  InsertObject,
  UpdateObject,
  AnyColumn,
  ExtractTypeFromReferenceExpression,
  ReferenceExpression,
  OperandValueExpressionOrList,
  FilterObject,
  sql,
} from "kysely";
import { Database } from "databaseType";

export const getSqlNow = (db: Kysely<Database>) => () => db.fn("datetime", [sql`'now'`, sql`'localtime'`]);

export function create(db: Kysely<Database>) {
  return async function <T extends keyof Database & string>(
    tableName: T,
    newRecords: ReadonlyArray<Insertable<Database[T]>>,
  ): Promise<Selectable<Database[T]>[]> {
    return await db.insertInto(tableName).values(newRecords).returningAll().execute();
  };
}

export function read(db: Kysely<Database>) {
  return async function <T extends keyof Database & string>(
    tableName: T,
    criteria: FilterObject<Database, T>,
  ): Promise<Selectable<Database[T]>[]> {
    // @ts-ignore
    return await db
      .selectFrom(tableName)
      // @ts-ignore
      .where((eb) => eb.and(criteria))
      .selectAll()
      .execute();
  };
}

export function update(db: Kysely<Database>) {
  return async function <T extends keyof Database & string>(
    tableName: T,
    criteria: FilterObject<Database, T>,
    updateWith: UpdateObject<Database, T>,
  ): Promise<Selectable<Database[T]>[]> {
    // @ts-ignore
    return await db
      .updateTable(tableName)
      // @ts-ignore
      .set(updateWith)
      // @ts-ignore
      .where((eb) => eb.and(criteria))
      .returningAll()
      .execute();
  };
}

export function destroy(db: Kysely<Database>) {
  return async function <T extends keyof Database & string>(
    tableName: T,
    criteria: FilterObject<Database, T>,
  ): Promise<Selectable<Database[T]>[]> {
    // @ts-ignore
    return await db
      .deleteFrom(tableName)
      // @ts-ignore
      .where((eb) => eb.and(criteria))
      .returningAll()
      .execute();
  };
}
```

読みづらい。ただ、利用する際には、特定のテーブル名と、それに対応するJSONを与えてあげれば動く。
```ts
const user = await read('user', { user_id: 1 });
```
補足すると、select文はread関数が対応するのだが、基本的にequalの条件しか表現できない。これは、primary keyやunique keyで特定レコードを検索する場合のみを想定しているためだ。

各行に@ts-ignoreが記載してある通り、これは型チェックを無視しないとコンパイル通らない。だが、上記の関数を呼び出し元で展開してやるとコンパイルが通る。
テーブル名などで、型を特定していくプロセスが、この名前空間上で表現できないため、この名前空間上で型の整合性を判断できないためと思われる。つまり、実質問題ないが、不安はある状態だと言える。
上記の`getSqlNow`関数についても、同様の現象が起きる。こちらも実質問題ないが、不安はある。

Kyselyとしては、SQLを表現できればいいはずなので、こういったutilityは少しオプション的な扱いかもしれない。
ただ、select文のwhere句には、以下のようにJSONをそのまま渡して、equal条件でのみ絞り込めるようにするKyselyの機能が存在するあたり、Kyselyとしてもある程度utilityを用意しようという意思が見えるように思う。
```ts
await db
  .selectFrom(tableName)
  // @ts-ignore
  .where((eb) => eb.and(criteria)) // criteriaはjsonでそのまま渡している
  .selectAll()
  .execute();
```

だが、primary keyやunique keyで単一のレコードを取ってくるというケースは、普通のアプリケーションコードにはかなりたくさん出てくる印象がある。
そこで、毎回`selectFrom(TABLE).where('id', '=', ID).selectAll().execute()`とやるのは面倒だし、その関数をテーブル毎に作るのも管理が煩雑だ。

今回は無理やりコンパイルするコードで実現したが、このあたり何かいいアイデアがあればと思う。
このあたりを強く意識するのであれば、Query BuilderではなくJSON Settingを使うべきだろう。

### SQLの分割管理
Utilityでの解説を見て感じた読者もいるかもしれないが、KyselyでSQLを部分的に管理するのは、骨が折れそうな印象だ。
これを例を持って示しつつ、筆者はむしろ良さだと考えているので、それを説明する。

例えば、以下のようなSQLを想定する。

```sql
select p.contents
  from post as p
 inner join comment as c
         on p.post_id = c.post_id
 where c.user_id = ?
     ;
```

パフォーマンス的な問題が出やすいため筆者は推奨しないが、このsqlは実はこうも表現できる。

```sql
select p.contents
  from post as p
 where p.post_id in (select c.post_id from comment as c where c.user_id = ?)
     ;
```

上記は推奨しないが、SQLの以下の部分が分割して管理できそうな印象には同意してもらえるはずだ。
`select c.post_id from comment as c where c.user_id = ?`

TypeScriptで管理しているコード上も、上記のように分割して管理したいものもあるかもしれない。
だが、Kyselyでは難しい印象を持った。

例を示そう。
以下は、最初のパフォーマンス的に問題ないものだ。

```ts
import { Kysely, SelectQueryBuilder } from "kysely";
import { Database } from "databaseType";

type Post = {
  post_user_id: string;
  post_contents: string;
}

export type GetPost = (db: Kysely<Database>) => (userId: string) => Promise<Post[]>;
export const getPost: GetPost = (db) => async (userId) => {
  return await db
    .selectFrom("post as p")
    .innerJoin("comment as c", "p.post_id", "c.post_id")
    .select((eb) => [
      "p.user_id as post_user_id",
      "p.contents as post_contents",
    ])
    .where("p.deleted_date", "is", null)
    .where("c.deleted_date", "is", null)
    .where("c.user_id", "=", userId)
    .orderBy(["p.posted_date desc"])
    .execute();
};
```

これは、こう書き換えられる。

```ts
import { Kysely, SelectQueryBuilder } from "kysely";

// ... コード省略

export const getPost: GetPost = (db) => async (userId) => {
  return await db
    .selectFrom("post as p")
    .select((eb) => [
      "p.user_id as post_user_id",
      "p.contents as post_contents",
    ])
    .where("p.deleted_date", "is", null)
    .where(
      "p.post_id",
      "in",
      getCommentId(db).where("c.user_id", "=", userId)
    )
    .orderBy(["p.posted_date desc"])
    .execute();
};

type GetCommentId = (db: Kysely<Database>) => SelectQueryBuilder<Database & { c: Database['comment']; }, "c", { post_id: number; }>;
const getCommentId: GetCommentId = (db) => {
  return db
    .selectFrom('comment as c')
    .where("c.deleted_date", "is", null)
    .select(['c.post_id as post_id']);
};
```

見てもらってわかるように`GetCommentId`の型がやたら複雑だ。これは最も簡単な例なのでこの程度だが、joinするテーブルが増え、selectするカラムが増えすると、もっとすごいものを書く必要がある。
もちろん型を書いてもいいのだが、多少重複コードがあっても`getCommentId`を切り出さずに、以下のように一息で書きたいという気持ちにさせてくれる。

```ts
export const getPost: GetPost = (db) => async (userId) => {
  return await db
    .selectFrom("post as p")
    .select((eb) => [
      "p.user_id as post_user_id",
      "p.contents as post_contents",
    ])
    .where("p.deleted_date", "is", null)
    .where(
      "p.post_id",
      "in",
      eb => eb
        .selectFrom('comment as c')
        .where("c.user_id", "=", userId)
        .where("c.deleted_date", "is", null)
        .select(['c.post_id as post_id'])
    )
    .orderBy(["p.posted_date desc"])
    .execute();
};
```

ただ、筆者個人の意見としては、これは逆に良さでもあると考える。

まず、上記のように`in`句を使った絞り込みはパフォーマンス上の問題が出やすく、基本的にはjoinでやったほうがいい場面が多いはずなので、部分管理する必要性がそもそも低い。

次に、そもそもsqlというのは関数と構造上違うということだ。
jsの関数は、独立して処理が終わり、呼び出し元に帰る。つまり、呼び出される側 -> 呼び出し元という順番が存在する。
しかしsqlというのは、sql文全体をオプティマイザが評価し、そこから最適な実行計画を呼び出すので、部分的に切り出したsqlが実行されたあとに、呼び出し側が実行されるわけではない。そしてそもそもそういう構造だとsqlはパフォーマンスが悪化するはずだ。
したがって、オプティマイザ視点では、sqlを部分的に評価して組み合わせるのではなく、全体を見てトップダウンで評価をくださなければならない。
そしてプログラマは、パフォーマンスチューニングしたければ、オプティマイザ視点を持つべきである。

最後に往々にして、切り出したところで、参照されるのが1箇所か2箇所では、コスパが悪い。2箇所程度なら重複して書いても、grepしてすぐリストアップできるだろう。
そして、何箇所からも参照されているのであれば、それこそキチンと型を書いて部分管理すればよいのだ。

つまり、型が書きづらくて、SQLの分割管理のモチベーションが上がらないが、それでも意味があるなら分割するようにすればいい。
こういう気持ちにさせてくれるので、結果的にSQLが良い状態で保てるのではないかと思う。
これがKyselyのSQLが分割管理しづらいのはよいという根拠だ。

ただ、現代においてはRDBのオプティマイザが進化し、`in`句を利用したクエリのパフォーマンスも問題ない可能性もある。
しかし筆者は、すべてのケースでオプティマイザが理想的な実行計画を出すという前提でsqlを書くのではなく、なるべくオプティマイザにわかりやすいsqlのほうがよいというポリシーを持っているので、上記のような説明をした。
RDB内部の進化については、筆者がキャッチアップできていない領域なので、間違っていたら指摘してほしい。

### SQL単位でファイル管理
実行されるSQLは、1クエリ1ファイルで管理されていたほうがいいだろう。
Kyselyで表現するSQLについても、独立した名前空間で管理されていたほうがよいと考える。これはSQLを分割管理しづらいので、そもそも複数のクエリを、同じ名前空間で管理するモチベーションが低いというのもある。

したがって、今回はクエリ事にファイルを分けて管理した。
クエリ事にファイルを分けると、使う側でそれらを集めてつかわなければならない。集めて使う際に、`Kysely`オブジェクトの参照など、考えることがある。
それをサポートする`getDatabase`という関数を定義して利用した。

```ts
import { getDatabase } from "./database";
import { getUser } from './query/getUser';
import { getPost } from './query/getPost';

export async function getUser(user_id: string): [User, Post[]] {

  const db = getDatabase({ getUser, getPost }, null);

  const user = await db.getUser(user_id);
  const posts = await db.getPost(user_id);

  return [user, posts];
};
```

上記は最も簡単な例だが、他にもアクセスするクエリがあるのであれば、`getDatabase`に与えているJSONにクエリを追加していく。
`query/getUser`は`Kysely`オブジェクトを引数として受け取るが、`getDatabase`の中ですでにbindされているので、あとは純粋な引数の`user_id`だけ与えればよい。

これをDIの仕組みと組み合わせてあげれば、`db`オブジェクトを`getUser`関数に注入して使える。
DIの仕組みについては検討し、以下の記事で言及している内容で実装した。
https://zenn.dev/motojouya/articles/app_router_use_scratch_impression#di%E3%83%A2%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AB%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6

`getDatabase`関数の実装については、トランザクションへのケアもあるので、次のパートで言及する。

### Transaction管理
Transaction管理でよくあるのは、middlewareあるいはAOPなど、何らかの関数をラップして、トランザクションを管理するというものが多い。
ただ、これでは当該関数の最初から最後までをTransactionとしてしまい、細かな管理が難しくなる。

昔はファイルと言えばサーバ上に保存したものだが、現代でクラウド上で保存するならオブジェクトストレージになるだろう。サービスも多様化し、外部APIを叩いて機能を実現しているものも少なくない。
アプリケーションの外にでてIOがあるのはデータベースだけではない。より細かなTransaction管理が過去よりも求められるようになっているはずだ。

筆者は今回、アプリケーションコードの中で、プログラマが任意のタイミングでTransactionを開ける関数を用意した。
使用感としては以下のような感じだ。

```ts
import { getDatabase } from "./database";
import { getUser } from './query/getUser';
import { updateUser } from './query/updateUser';

export async function updateUser(user_id: string): User {

  const db = getDatabase({ getUser }, { updateUser });

  const user = db.getUser(user_id);

  // 任意のタイミングでトランザクションを開始/終了できる
  db.transact(trx => {
    trx.updateUser(user_id, { ... });
  });

  return db.getUser(user_id);
};
```

上記コードでは`db`オブジェクトが`transact`関数を持っているわけだが、`getDatabase`の第一引数はTransactionの外、第二引数はTransactionの内側で実行されるものだ。
前段で紹介したファイル名前空間ごとに分けたクエリをまとめる機能も持っているので、オブジェクトにまとめて引き渡す形を取る。

長いが、`getDatabase`関数の実装を示す。

```ts
import Sqlite from "better-sqlite3"; // コード例はsqlite3だが基本的にSQLインタフェースなら何でも動く
import { Kysely, SqliteDialect } from "kysely";
import { Database } from "./databaseType"; // Kyselyを利用する上では、スキーマ定義を型として定義する必要がある（generationツールもある）。その型定義を参照している

export type GetQuery = Record<string, (db: Kysely<Database>) => unknown>;

let db: Kysely<Database>;

export type Query<Q extends GetQuery> = {
  [K in keyof Q]: Q[K] extends (db: Kysely<Database>) => infer F ? F : never;
};

export type GetKysely = () => Kysely<Database>;
export const getKysely: GetKysely = () => {
  return new Kysely<Database>({
    dialect: new SqliteDialect({
      database: new Sqlite(process.env.SQLITE_FILE),
    }),
  });
};

export type Transact<T extends GetQuery> = <R>(callback: (trx: Query<T>) => Promise<R>) => Promise<R>;

export type DB<Q extends GetQuery, T extends GetQuery> = Query<Q> & {
  transact: Record<never, never> extends T ? undefined : Transact<T>;
};

export function getDatabase<T extends GetQuery>(queries: null, transactionQueries: T): DB<Record<never, never>, T>;
export function getDatabase<Q extends GetQuery>(queries: Q, transactionQueries: null): DB<Q, Record<never, never>>;
export function getDatabase<Q extends GetQuery, T extends GetQuery>(queries: Q, transactionQueries: T): DB<Q, T>;
export function getDatabase<Q extends GetQuery, T extends GetQuery>(
  queries: Q | null,
  transactionQueries: T | null,
): DB<Q, T> {
  if (!db) {
    db = getKysely();
  }
  let dbAccess = {};

  if (queries) {
    dbAccess = getQueries(db, queries, dbAccess);
  }

  if (transactionQueries) {
    dbAccess = {
      ...dbAccess,
      transact: getTransact(db, transactionQueries),
    };
  }

  return dbAccess as DB<Q, T>;
}

function getTransact<T extends GetQuery>(db: Kysely<Database>, queries: T): Transact<T> {
  return async function <R>(callback: (trx: Query<T>) => Promise<R>): Promise<R> {
    try {
      return db.transaction().execute((trx) => {
        const transactedQueries = getQueries(trx, queries, {});
        return callback(transactedQueries);
      });

    } catch (e) {
      if (e instanceof Error) {
        console.log('database error!');
      }
      throw e;
    }
  };
}

function getQueries<T extends GetQuery>(db: Kysely<Database>, queries: T, acc: object): Query<T> {
  return Object.entries(queries).reduce((acc, [key, val]) => {
    if (typeof val !== "function") {
      throw new Error("programmer should set context function!");
    }

    if (!Object.hasOwn(queries, key)) {
      return acc;
    }

    return {
      ...acc,
      [key]: val(db),
    };
  }, acc) as Query<T>;
}
```

`Kysely.transaction.execute`関数は内部的に、例外をcatchしたときにRollbackを行う形になっている。
最もオーソドックスな方法ではあるが、最近はRustなどの影響を受けてエラーを例外ではなく、値としてReturnしたいというプログラマもいるはずだ。
そういった意味では、Rollbackの方法が例外だけでは機能不足な印象がある。

これは実は今後以下のPRでケアされるはずだ。
https://github.com/kysely-org/kysely/pull/962

Kyselyは記事時点で0.27.4だが、0.28に入りそうだ。
上記PRは、Rollbackだけではなくsave pointにも対応しているようなので、より細かな制御が行えるだろう。

### 迷ったところ
sqlite3での日付計算で以下の式を実現しようとしたが、その実現方法で迷ったのと、内部の仕組みを調べたので、記載しておく。
```sql
datetime('now', 'localtime', '-1 days')
```

#### 失敗案1
最初、以下のように実装した。
```ts
const days = 1;
const datetimeSql = db.fn("datetime", ["now", "localtime", `-${days} days`]);
```

だが、SQLとしては以下が出力され、うまく動かない。
```sql
datetime(`now`, `localtime`, `-1 days`)
```

実現したいものとしては、`datatime`関数は文字列の値を受け取る。その文字列に固定の`now`や`localtime`を与えると、実際の時間に変換してくれる機能がある。つまり`now`、`localtime`はただの文字列の値だ。
だが、文字列の値として表現したい場合はシングルクォートで、バッククオートで囲むとカラム名という意味になる。これはSQL標準での仕様だ。
つまり、この案で出力されるSQLでは`now`や`localtime`はカラム名となってしまい、そんなカラム名はないので失敗する。

#### 解決案
失敗案1で素直に文字列を与えるとカラム名になってしまうkyselyの仕様なので、以下のように`value`関数で囲って上げればよい。

```ts
const days = 1;
const datetimeSql = db.fn("datetime", [sql.value("now"), sql.value("localtime"), sql.value(`-${days} days`)]);
```

これで想定どおりのSQLが得られる。

#### 失敗案2
すでに解決案を示したが、もう一つ失敗例を示して、内部の仕様の解説も行う。

以下は、fn関数で`datetime`関数を表現するのではなく、引数を含む関数全体をsql文にしてしまおうという案だ。
全体をsql文にするには、タグ付きテンプレートリテラルで`sql`というものが用意されているので、利用する。

ただし、以下はうまく動かない。

```ts
const days = 1;
const datetimeSql = sql`datetime('now', 'localtime', '-${days} days')`;
```

上記は、以下のsqlとして出力される。

```sql
datetime('now', 'localtime', '-? days')
```

kyselyは、TypeScriptのAPIを用いると、値として引数に渡したもの、カラム名として引数に渡したものを区別していて、カラム名はPrepared Statementに組み込まれ、値は`?`プレースホルダで管理される。
これは、sql injectionを防ぐための基本的なしくみだ。
`sql`タグ付きテンプレートリテラルでも、テンプレート上で参照している変数を、値として認識して`?`プレースホルダとして管理しようとする。

これを理解していないと、上記のような使い方をしてしまう。
`days`という変数をSQL上の値として扱おうとしているため、`?`プレースホルダに変換される。
そして上記のsql文では`?`プレースホルダがシングルクォートの内部に入ってしまい、プレースホルダとして機能せず、プレースホルダが見つからなくて失敗するということが起きる。

失敗案2のようにsqlタグ付きテンプレートでやるならば以下のようにすればよい。

```ts
const days = 1;
const minusDays = `-${days} days`;
const datetimeSql = sql`datetime('now', 'localtime', ${minusDays})`;
```

これならば以下のようなprepared statementが発行されるので、うまく動く。

```sql
datetime('now', 'localtime', ?)
```

ただ、基本的にはfn関数のほうが、各文字列をカラムなのか値なのか個別に認識しながら発行されるので、より安全だろう。
この仕様は、仕組みがわからなくてコードを読んで理解したのだが、内部が知れるよい機会だった。

## Outro
Kyselyを選択する理由、使い方、難しい部分などを解説してきた。

筆者はHTMLのテンプレートエンジンとしてReactが好きだが、他のテンプレートエンジンと違って、JavaScriptの機能で組み立てており、独自DSLが他のやり方より圧倒的に少ない点が気に入っている。
KyselyもQuery Builderなので、JavaScriptの機能で組み立て、発行されるSQLもイメージが付きやすい。どこかReactに似ているなと感じている。

