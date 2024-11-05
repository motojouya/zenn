---
title: "Kyselyの利用と工夫と感想"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Kysely', 'TypeScript']
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

SQLのシンタックスは文になっており、宣言的な設定のようになっていないので、JSONのほうが直感的だと感じるプログラマも多いだろう。
JSONなので、TypeScript上で取り回ししやすく、Queryの発行内容もif分岐で多様に変化させやすく、動的にSQLを操作することもできる。
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
生のSQLに近い形であれば、sqlファイル上で、DSLとしてif分岐を入れてロジックを表現し、多様なSQLを発行できるタイプのものもある。ただし、筆者が知るものはTypeScriptではなくJavaのものだ。

sqlcは本当に生のSQLに近い形のようで、動的にSQLを組み立てる機能は備わっていないようだ。（使ってないので曖昧だが検索すると難しいという記事が多い）
おそらく、上記の文字列に、何らかの文字列を連結して実行することは可能なはずで、動的SQL自体は実現できそうだが、機能外でのケアのようだ。

つまり、Raw SQLは動的にSQLを組み立てるのは苦手か、あるいは独自のDSLを覚える必要がある点が難点だろう。SQL上のDSLは、TypeScriptのAPIよりも学習の負荷が高いはずだ。
逆にJSON SettingやQuery BuilderはTypeScript APIで提供されていないRDBの機能は使えない。
Raw SQLならば、SQLでしかないので、RDBの制約をそのまま引き継げ、使えるはずだ。

sqlcについてはドキュメントのみの情報なので、間違っていたら指摘いただきたい。

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
Prismaなら`const user = await prisma.user.find(id);`と記載するところだが、Kyselyなら`const user = await kysely.selectFrom('user').where('id', '=', id).selectAll().execute();`となる。

#### 2. 簡易な操作
簡易な操作という表現は曖昧だ。より具体的には、チューニング欲求が高まりそうにない操作と定義したほうがいいだろう。
つまり2と3の違いは、チューニング欲求が高まりそうか否かだけの違いだ。

チューニング欲求が高まりそうというのも曖昧な表現だ。そんなもの見たこともないというプログラマもいるかもしれない。
例えば、2,3のテーブルをそれぞれの主キー外部キーでJoinし、where条件もintegerカラムでの大小や、日付カラムでの期間指定のみ。というようなSQLはチューニング欲求は高くないだろう。
だが、テーブルのJoinする外部キーの候補が複数あったり、Joinするテーブルが集約カラムだったり、select句にCase文が入っていたり、where条件もandとorのネストが5重になっていたりした場合、将来的にチューニングしなければならない可能性は高まるだろう。



具体的にはJSON Settingで実装しやすく、Query Builderでは冗長に感じるものだ。

JoinもWhere句も追加され、1と比較して、2step難易度が上がっているように見えるが、実用上のパターンを検討した上でこうなっている。
1のパターンでは、一つのテーブルに存在確認だけすればよいというケースが多いはずだ。

細々した条件をWhere句に指定する際、えたいデータがあるテーブルだけで完結するケースは少ない。
また逆に、複数テーブルをjoinして、いずれかのテーブルのprimary keyでデータを取得する際も、他のテーブルには細々としたwhere句条件をつけるパターンも多いはずだ。
まったくないとは言わないが、JoinもWhere句も、同時に扱わなければならないものになるというのが、筆者の経験的な感覚だ。

#### 3. 複雑でチューニング欲求が高まりそうな操作


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
TypeScriptのDBアクセスライブラリは様々あり、JSON Settingがメインだったり、Query Builderがメインだったりもするが、複数の機能の複合であることが多い。

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

けっこうある。上記はTypeScriptで開発されているものに絞っているので、JavaScriptで開発されTypeScriptの型情報が存在するものにまで広げると、もっと挙げられる。
Sequelizeも半分ぐらいはもうTypeScriptのようだが、ここでは除外した。
また、厳密にはRaw SQLはどのライブラリでも扱えそうだが、実用上多用しそうな印象のあるもののみ記載した。

すべての言及するわけにはいかないので、ここからは2024年後半でTypeScriptで最も勢いのあるPrisma（という認識だがどうだろうか）と、今回取り上げるKyselyに注目する。
動的に組み立てないSQLについては以下の表のように考ればよいだろう。

|            | Prisma | Kysely |
|------------|--------|--------|
| 1.主キー   | strong |   weak |
| 2.簡易     | strong | enough |
| 3.複雑     | enough | strong |
| 4.限定機能 | enough | enough |

strong: 得意なので実装が楽
enough: 十分な機能があるが得意というわけではない
  weak: 苦手なので実装が面倒

Prismaは1,2はJSON Settingで対応し、3,4はRaw SQLで対応する。kyselyは4のみRaw SQLだろう。（Raw SQLはどのライブラリにもある）

これが動的に組み立てるSQLならばどうだろうか。
1は動的に組み立てることはないだろうし、4で動的に組み立てることを実現するにはいずれにしろコストがかかるので検討対象としない。
2については、PrismaはJSON Settingで柔軟に対応できるが、3については、JSON Settingではチューニング欲求にたいして機能が不十分だし、Raw SQLでは動的に組み立てることに対して不十分だ。

つまり、動的なSQLに対しては、3においてPrismaではweakになる。対してKyselyはstrongのままだ。

### 評価
PrismaとKyselyを比較してみたが、比較軸は他にもある。TypeScriptの型による支援がどこまであるか。APIが使いやすいか。対応しているSQLの幅。Migrationのやりやすさ。
そういったものをすべて比較して評価する必要があるが、今回着目したのは、動的にSQLを組み立てるか否か、SQLの実現難易度の2軸だけだ。
また、他のDBアクセスライブラリが気になるのであれば、上記の表に書き加え、動的SQLはどうか検討してみてほしい。

そして、その2軸であっても、読者によって印象は様々だろう。
SQLの実現難易度で設定した3.複雑でチューニング欲求が高まりそうな操作も、PrismaのJSON Settingで十分対応できるという認識の方もいるだろう。
そもそもそういったクエリはチューニングするのではなく、転置インデックスやCQRS構成に変更すべきだし、そのための開発余力も取ってあるという方もいるかもしれない。
チューニング欲求という表現をしたが、欲求であって実際にチューニングをしなければならない状況に陥るかは、事業の成長や、インフラコストなど様々な要因も絡んでくる。

したがって、挙げた2軸についても、アプリケーションの特性によって選ぶべきだろう。
動的に組み立てるSQLが複数あり、複雑になる可能性はあるのか？あるいは開発余力はあるが、開発優先度的にとにかくリリースしたいのか。
それらによって決めればよい。多くのケースでPrismaが最適という結論もでるだろう。

ただ、筆者は動的に組み立てるSQLに対してパフォーマンスチューニングをしたことが何度もある。
そしてPrismaの開発者が素晴らしいことは知っているが、JSON SettingからSQLを組み立てるライブラリ機能を手放しで信じるというのは怖い。
また、一つのアプリケーション上に、動的に組み立てるSQLが両手以上の数あるプロジェクトも経験しているし、それらのSQLに対して複雑さを感じている。
これは動的にSQLを組み立てる機能は、アプリケーション上の検索機能で利用され、検索機能はパフォーマンス負荷がかかりやすいという面があるからだ。
筆者は、Kyselyを選ぶべきだと結論づけるべきプロジェクトも多く存在するという肌感がある。

読者には、上記2軸以外、またPrisma、Kysely以外も検討した上で、適切なライブラリ選択をしてほしい。

今回、筆者はKyselyの型のきかせ方、Query Builderを覚えればだいたいのことが出来てしまう対応範囲の広さなどが好ましいと考え、Kyselyを覚えたくて使った。
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

読みづらい。ただ、利用する際には、特定のテーブル名と、それに対応するJSONを与えてあげれば動く。
```ts
const user = read('user', { user_id: 1 });
```
補足すると、select文はread関数が対応するのだが、基本的にequalの条件しか表現できない。これは、primary keyやunique keyで特定レコードを検索する場合のみを想定しているためだ。

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
import { getPost } from './query/getPost';

export async function getUser(user_id: string): [User, Post[]] {

  const db = getDatabase({ getUser, getPost }, null);

  const user = await db.getUser(user_id);
  const posts = await db.getPost(user_id);

  return [user, posts];
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

