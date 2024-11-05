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

SQLのシンタックスは、文になっており、JSONで書くように設定のようになっていないので、JSONのほうが直感的だと感じるプログラマも多いだろう。
JSONなので、TypeScript上で取り回ししやすく、Queryの発行内容もif分岐で多様に変化させやすい。
反面、SQLのシンタックスとはだいぶ違うものなので、どのようなSQLが発行されているか想像しづらく、また最適なSQLになっているかわからないという問題点もある。

#### Query Builder
区別のためにQuery Builderとしたが、この記事の上ではPoEAAのQuery Objectと同等と考えて問題ない。
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
2. 簡易な操作
3. 複雑な操作
4. 限定的な機能を用いた操作

1,2,3は自明だろう。4は実のところ筆者も想像できない。とにかく難しいSQLとだけ認識しておいてほしい。詳しくは後述する。
上記の4つに分けたのは、SQLを各種ライブラリで実現する際の難易度の順番だ。

#### 1. 単一のテーブルに対するPrimary KeyあるいはUnique Keyによる操作
これは最も簡単な操作だ。どのライブラリでも1行で表現できる。
特に、特定のリソースに対して、更新、削除などする際に表れる。
特定のリソースを`select * from TABLE where id = ?`で取得し、`update TABLE set COLUMN = ? where id = ?`などで更新する。

#### 2. 簡易な操作
簡易な操作という表現は曖昧だが、定義も曖昧なものだ。
具体的にはJSON Settingで実装しやすく、Query Builderでは冗長に感じるものだ。

JoinもWhere句も追加され、1と比較して、2step難易度が上がっているように見えるが、実用上のパターンを検討した上でこうなっている。
1のパターンでは、一つのテーブルに存在確認だけすればよいというケースが多いはずだ。

細々した条件をWhere句に指定する際、えたいデータがあるテーブルだけで完結するケースは少ない。
また逆に、複数テーブルをjoinして、いずれかのテーブルのprimary keyでデータを取得する際も、他のテーブルには細々としたwhere句条件をつけるパターンも多いはずだ。
まったくないとは言わないが、JoinもWhere句も、同時に扱わなければならないものになるというのが、筆者の経験的な感覚だ。

#### 3. 複雑な操作 -> チューニングのモチベーションが高い操作


3は純粋にGroup Byで集約条件が入っているものだ。
ライブラリによっては、テーブル定義をアプリケーション上に記載し、参照して型チェックするが、集約した結果のテーブルは情報がSQLにしかない。
当然扱いづらいはずだ。

#### 4. 限定的な機能を用いた操作
標準的なSQLの機能ではない操作。特定のRDBに限った仕様も存在する。
Raw SQLは当然このあたりを扱えるはずだが、TypeScript APIを用意する必要があるJSON SettingやQuery Builderでは実現が難しい領域だ。

ただ、筆者はそういう機能を使ったことがないし、使わないと実現できない状況に追い込まれたこともない。
特定のRDB固有の機能を使うプロジェクトの存在は否定できないが、こういった状況を想定するかは、プロジェクト次第だろう。

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

上記のものは、JSON Settingがメインだったり、Query Builderがメインだったりもするが、複数の複合であることが多い。

- Prisma
  JSON Setting + Raw SQL
- Drizzle ORM
  JSON Setting + Query Builder
- kysely
  Query Builder
- sqlc
  Raw SQL
- Micro ORM
  JSON Setting + Query Builder
- Type ORM
  JSON Setting + Query Builder

厳密にはRaw SQLはどのライブラリでも扱えそうだが、実用上多用しそうな印象のあるもののみ記載した。
こうしてみると、JSON Setting + Query Builderなライブラリが多い。
上記ライブラリの違いを見ると、リレーションの設定があるか、またActive Record Patternで利用できるのかと言った違いもあるが、今回は言及しない。

JSON Setting、Query Builderは動的SQLに対応しやすく、Raw SQLは難易度の高いSQLに対応し易いはずだ。
そう考えると、Prismaはどちらの機能も抑えており、様々な場面に対応しやすそうだ。

JSON Settingの良さは扱いの簡単さだと考えているが、SQLをチューニングしたくなった際にはどうだろうか？
発行されるSQLがわかりやすいほうがいいはずだ。これは、どれもQuery BuilderなりRaw SQLなりの機能を備えているので、チューニングにも対応できそうだ。

では、動的に組み立てているSQLをチューニングしたくなった際にはどうだろうか。
動的に組み立てているSQLがパフォーマンス上の問題を抱えやすいわけではないが、経験的には動的に組み立てているSQLをチューニングしたことは多い。
これは動的にSQLを組み立てる機能は、アプリケーション上の検索機能で利用され、検索機能はパフォーマンス負荷がかかりやすいという面があるからだ。
もちろんこれを、転置インデックスに変更したり、CQRS構成に変更したりということは考えられるが、できるならチューニングでパフォーマンスが上がったほうが安上がりではある。

PrismaはJSON Setting側はSQLは把握しづらく、Raw SQLは動的SQLが組み立てづらい。両立した機能ではないというのは抑えて置きたい。
検索は複雑なものにならないだろう、あるいは複雑な検索があっても、一つしかないので、都度リソースを投入して直せばいいという考え方もある。
これは作るアプリケーションの特性にもよる。筆者はキャリア的にtoBのシステムが多く、検索画面が複数あり、常に数個のテーブルをJoinしながらsqlにして数百行みたいなプロジェクトは多く経験しているので、Prismaに対して、機能不足という感想を持っているが、そんなアプリケーションどこにあるの？みたいな感覚のプログラマもいるだろう。
それぞれ作るアプリケーションの特性を鑑みて選べばいいだけだ。

今回は筆者はKyselyを採用した。上記の背景を検討したうえで、Query Builderが最も対応力が高いと判断したためだ。
4.特定DB特有のクエリというのは、今回は想定されないので、その部分については懸念していない。もし必要になったとしてもRaw SQLで対応できるだろう。

また、筆者はSQLに慣れているので、特にRelationの設定や、Active Record Patternのような重厚な仕組みは不要で、すべてSQLで完結できる。
SQLで完結できるので、宣言的に記載されたQuery Builderを見れば、どんなSQLを発行しているのか把握でき、コールドリーディング上も利点がある。

## 使い方

以降では、Kyselyの使い方や、使う際の工夫について記載していく。
基本的な使い方についてはKyselyのドキュメントを参照してほしい。

どちらかというと、工夫した内容や、仕様上迷った部分などに言及していく。

### 仕組み
1. sql object
2. 変換
3. 実行

型定義の仕組み
select -> from -> whereではかけない。from -> (select or where)で書くが、これは型を効かせるため。

### Utility
1.単一のテーブルに対するPrimary KeyあるいはUnique Keyによる操作が、Query Builderでは冗長になってしまうことはすでに述べた。
2の簡易なSQLについても冗長になりがちだが、それを支援するツールを作成しようとすると、それはほぼほぼJSON Settingの実装になり、それならばPrismaを使おうとなる。
1についてのみ補う関数を用意した。

また、insert文やupdate文の発行時に現在時刻を設定するための関数というのが、各種RDBには備わっていることが多い。
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
import { Database } from "@/database/type";

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
  // return async function <T extends keyof Database & string>(
  //   tableName: T,
  //   criteria: Partial<Selectable<Database[T]>>
  // ): Promise<Selectable<Database[T]>[]> {
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
  // return async function <T extends keyof Database & string>(
  //   tableName: T,
  //   criteria: Partial<Selectable<Database[T]>>,
  //   updateWith: Updateable<Database[T]>
  // ): Promise<Database[T][]> {
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
  // return async function <T extends keyof Database & string>(
  //   tableName: T,
  //   criteria: Partial<Selectable<Database[T]>>
  // ): Promise<Selectable<Database[T]>[]> {
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

読みづらい。ただ、利用する際には、特定のテーブル名と、それに対応するJSONを与えてあげれば動くはずだ。
select文はread関数が対応するのだが、基本的にequalの条件しか表現できない。これは、primary keyやunique keyで特定レコードを検索する場合のみを想定しているためだ。

各行に@ts-ignoreが記載してある通り、これはコンパイル通らない。
だが、上記の関数を呼び出し元で展開して上げるとコンパイルが通る。テーブル名などで、型を特定していくプロセスが、この名前空間上で表現できないため、この名前空間上で型の整合性を判断できないためと思われる。つまり、実質問題ないが、不安はある状態だと言える。
呼び出し元で、上記の`getSqlNow`関数についても同様の現象が起きる。こちらも実質問題ないが、不安はある。

Kyselyとしては、SQLを表現できればいいはずなので、こういったutilityは少しオプション的な機能かもしれない。
ただ、select文のwhere句には、以下のようにJSONをそのまま渡して、equal条件でのみ絞り込めるようにするKyselyの機能が存在するあたり、Kyselyとしてもある程度utilityを用意しようという意思が見えるように思う。
```
await db
  .selectFrom(tableName)
  // @ts-ignore
  .where((eb) => eb.and(criteria))
  .selectAll()
  .execute();
```

だが、primary keyやunique keyで単一のレコードを取ってくるというケースは、普通のアプリケーションコードにはかなりたくさん出てくる印象がある。
そこで、毎回`selectFrom(TABLE).where('id', '=', ID).selectAll().execute()`とやるのは面倒だし、その関数をテーブル毎に切るのも管理が煩雑だ。

今回は無理やりコンパイルするコードで実現したが、このあたり何かいいアイデアがあればと思う。
このあたりを強く意識するのであれば、Query BuilderではなくJSON Settingを使うべきだろう。

### SQLの分割管理
Utilityでの解説を見て感じた読者もいるかもしれないが、KyselyでSQLを部分的に管理するのは、骨が折れそうな印象だ。

例えば、以下のようなSQLを想定する。

```sql
select posts.title
  from posts
 inner join comments
         on posts.id = comments.post_id
 where comments.user_id = ?
     ;
```

パフォーマンス的な問題が出やすいため筆者は推奨しないが、このsqlは実はこうも表現できる。

```sql
select title
  from posts
 where id in (select post_id from comments where user_id = ?)
     ;
```

上記は推奨しないが、SQLの以下の部分が分割して管理できそうな印象には同意してもらえると思う。
`select post_id from comments where user_id = ?`

TypeScriptで管理しているコード上も、上記のように分割して管理したいものもあるかもしれない。
だが、Kyselyでは難しい印象を持った。

それは、Utilityの説明でも触れたように、テーブル名などの情報が揃っていないと、型推論が決定的にならないため、あるいは型定義が煩雑になるためだ。
Kysely内部の型はかなりしっかりexportされているので、部分的に分割したKyselyのオブジェクトも型を表現できるとは思うが、記載は冗長になりそうだ。
これはKyselyが、SQLの型をしっかり管理し、テーブルだけではなく、テーブルをsql上で部分的にpick upしたview的なものですら型として管理しているためだ。

ただ、筆者個人の意見としては、これは逆に良さでもあると考える。
これはまだ論理的に説明できるほど整理しているものではなく、感覚的な意見だが、SQLを分割管理するのは、そもそも非常に難しい。
これは、上記のコードを以下のように表現すると、ヒントがあるように思う。

```sql
select title
  from posts as p
 where exists (select post_id from comments as c where c.user_id = ? and c.post_id = p.id)
     ;
```

SQLの仕様上、部分的に切り出したいcommentsテーブルへのクエリが、外側のpostsに依存している。
利用される側が、利用する側のidの情報をしらなければならないという、パラドックスがある。
これを解決するための仕組みを用意しなければならないだろう。

また、そもそも宣言的にデータを表現するSQLという文を、分割して部分適用するという事自体が、使い方として間違ってそうという感覚もある。
まだ、整理できてないことを記載して申し訳ないが、とにかくSQLを分割管理することが難しいということに共感は得られただろうか？

難しいという前提に立つのであれば、そもそも分割管理し辛い仕組みのほうが、分割管理するのではなく、多少コードが重複したとしても、それぞれSQLを宣言的に記載したいと思うはずだ。
そういったコードの状態を目指すほうが、筆者としてはいいのではないかと考えている。

### SQL単位でファイル管理
実行されるSQLは、ファイル単位で管理されていたほうがいいだろう。
Kyselyで表現するSQLについても、独立した名前空間で管理されていたほうがよいと考える。
これはSQLを分割管理しづらいので、そもそも複数のクエリを、同じ名前空間で管理するモチベーションが低いというのもある。

したがって、今回はクエリ事にファイルを分けて管理した。
クエリ事にファイルを分けると、使う側でそれらを集めてつかわなければならない。集めて使う際に、`Kysely`オブジェクトの参照など、考えることがある。
それをサポートする`getDatabase`という関数を定義して利用した。

```ts
import { getDatabase } from "./database";
import { getUser } from './query/getUser';

export async function getUser(user_id: string): User {

  const db = getDatabase({ getUser }, null);

  return db.getUser(user_id);
};
```

上記は、最も簡単な例だが、他にもアクセスするクエリがあるのであれば、`getDatabase`に与えているJSONに、クエリを追加していけばよい。
また、コード上で、直接DBコネクションを取得しているので、DIのような仕組みで依存性を解決したい場合は、`getDatabasse`を関数の外で呼び出してdbオブジェクトを注入すればよいはずだ。

getDatabase関数の実装については、トランザクションへのケアもあるので、次のパートで言及する。

### Transaction管理

```ts
import Sqlite from "better-sqlite3";
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

// T extends Record<never, never> だと普通に成立するので逆にしておく
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

  return dbAccess as DB<Q, T>; // FIXME as!
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

```ts
import { getDatabase } from "./database";
import { getUser } from './query/getUser';
import { updateUser } from './query/updateUser';

export async function updateUser(user_id: string): User {

  const db = getDatabase({ getUser }, { updateUser });

  const user = db.getUser(user_id);

  db.transact(trx => {
    trx.updateUser(user_id, { ... });
  });

  return db.getUser(user_id);
};
```

transaction管理が例外のみ
早く入ってほしい
https://github.com/kysely-org/kysely/pull/962
現在0.27.4だが、0.28.0入りそう

## Outro

