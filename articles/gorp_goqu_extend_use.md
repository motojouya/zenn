---
title: "gorpとgoquを拡張してDBアクセスする"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "ORM"]
published: false
---

## 導入
Go言語のORMといえば、[GORM](https://github.com/go-gorm/gorm)が有名なところだろう。会社でGoの経験のある同僚には、[SqlBoiler](https://github.com/aarondl/sqlboiler)をおすすめされた。  
だが、どちらも筆者の手に馴染みそうな道具とはならなそうだったので、探したところ、gorpとgoquを組み合わせるに至った。  
[upper/db](https://github.com/upper/db)も筆者にfitしそうではあるが、goquの表現力が素晴らしく、gorpの節制的なAPIも気に入ったので、gorp+goquとした。  

https://github.com/go-gorp/gorp

https://github.com/doug-martin/goqu

最近はGo言語が気に入ってGoばかり書いているが、以前はTypeScriptをトレーニングしていて、その際は[kysely](https://github.com/kysely-org/kysely)というライブラリを使い、記事にした。  
TypeScriptの柔軟性を利用して、goquよりも表現力が高く、素晴らしいライブラリなので参考にしてほしい。この記事はkysely紹介記事の、gorp+goqu版である。  
https://zenn.dev/motojouya/articles/kysely_use_scratch_impression

## DBアクセスライブラリに求めるもの
[kyselyの紹介記事](https://github.com/kysely-org/kysely)と似たような内容になってしまうが、筆者がDBアクセスライブラリに求めるものをおさらいしておきたい。  

そもそも大前提として、DBアクセスの詳細は隠蔽されていたほうがいい。  
より具体的には、DB操作というのは、特定の関数をinterfaceとして、引数を与えることで指示が出せる状態が望ましい。  

これによって、テストを行う際に、DBアクセス部分は切り離して独立させ、DBアクセスと切り離してテストができる。  
DBアクセスを行うモジュールは独立しているので、単体テストの段階でDBアクセスを行ってテストできる。  

ちなみに筆者はdockertestを利用して、DBアクセスモジュールの単体テストを実装している。  
https://github.com/ory/dockertest

### 反例

これがどういうことかというと、以下のAPIではないということだ。  
```go
product, err := gorm.G[Product](db).Where("id = ?", 1).First(ctx)
```

上記はgormのサイトの[トップページ](https://gorm.io/docs/index.html)に記載された使い方の中で、データを取得する方法の1行だ。  
何が問題かというと、DBを取得するのに`gorm.G[Product](db)`から、2回関数をチェーンしている。これが2回であればいいが、複数関数をチェーンする場合、モックするのが非常に難しくなってしまう。  

これを解消するためには、以下のようなinterfaceと関数を定義してやるという方法がある。  

```go
type DatabaseAccess struct {
  db *sql.DB
}

func (da DatabaseAccess) GetProductById(ctx context.Context, id int) (Product, error) {
  return gorm.G[Product](da.db).Where("id = ?", id).First(ctx)
}
```

だが、せっかくDBアクセスライブラリを使っているのに、毎回これらをラップして定義し直すのは、骨が折れるだろう。  
なんのためのDBアクセスライブラリなのか。と思ってしまわないでもない。  

関数をチェーンするのがダメならば、より汎用的な表現の引数を受け入れるようにできれば？という考え方もある。  
こちらはGoではなくTypeScriptのライブラリで申し訳ないが、prismaだ。  
https://www.prisma.io/docs/orm/prisma-client/queries/filtering-and-sorting

```ts
const users = await prisma.user.findMany({
  where: {
    email: {
      endsWith: "prisma.io",
    },
  },
});
```

これならば関数チェーンがひどくなることなどないだろう。関数のinterfaceもシンプルだし、独立したモジュールとして存在できる。  
ただし、上記の例で言えば、`findMany`にわたすjsonで何でもできてしまい、DBアクセスモジュール側の役割が非常に薄くなる。  
ロジックとしては、where句を表現するjsonがここで重要なロジックであり、これをテストしたいわけだ。そしてそれはデータベースにアクセスしてテストしたほうがいい。  
つまり利用側でjsonを定義しても、そのjsonの中身がテストで精査されないので、テストの観点が漏れてしまう。  

これも解消するのであれば、DBアクセスモジュールの関数定義を以下として、jsonを定義する部分を隠蔽すべきだろう。  
```ts
const getUserByEmailEndsWith = (emailSuffix: string) => User
```

こうなると、先程のGORMの例と同じストーリーとなり、筆者としてはバッドエンドだ。  
物語と同様、プログラミングコードの良さも人によって違うだけなので、ここでのバッドエンドというのは、筆者の視点である。  
GORMもprismaも他の課題を非常にうまく解決してくれる素晴らしいライブラリである。Not For Meなだけなことは、断っておきたい。  

## 複雑なSQL
複雑なsqlを表現する場合は、どんな場合においても、ラップすると思った方もいるのではないだろうか。  

```sql
select post.id
     , post.title
     , post.content
     , post.tag
  from post as p
  left outer join comment as c
          on p.id = c.post_id
 where c.user_id = <user_id>
   and c.content like '%テスト%'
     ;
```

上記は、SNSで特定のユーザが`テスト`という単語を含むコメントをつけた投稿を探すクエリだ。  
筆者は、このクエリを実現するsqlが、DBアクセスモジュールの外側に定義されていると辛いなと感じる。  
このクエリの機能は、DBにアクセスする単体テストで品質が保たれてほしいし、独立したモジュールとして切り替え可能で、利用側のモジュールからはモックされて単体テストが実現されてほしい。  

そして、その関数は、どんなライブラリを使ったとしても、簡単には実現できないだろう。  
反例の中で説明してきた、わざわざDBアクセスライブラリをラップして、モジュールを切る必要が出てくる。  

でも、そこは徒労感があまりない。それは複雑なものだからだろう。複雑だから、ロジックを独立させて、切り替え可能にし、単独で機能の保証をしたいのだ。  
こういったケースにおいては、DBアクセスモジュールを切ることは、むしろ歓迎すべきことだ。  

## 簡単なsqlの定義
複雑なsqlの反対は、簡単なsqlだろう。複雑なsqlというのは多様なので、一言で言い表せないだろう。だって複雑なのだから。  

だが、簡単なsqlの定義も、考えるとどこに線を引くかで悩ましい問題ではある。  
- 単一のテーブルにしかアクセスしない
- 集計クエリやwindow関数がない

上記でもいいが、where句は非常に多様だ。
```sql
select * from post where created_date >= <from_date> and created_date < <to_date>
```

上記のsqlを実現するだけでも、Gormなら以下のようにチェーンが伸びる。  
```go
product, err := gorm.G[Product](db).Where("created_date >= ?", fromDate).Where("created_date < ?", toDate).First(ctx)
```

また、こういったクエリは、後からwhere句に条件が追加されやすい。様々なユースケースに合うように、カスタマイズされてしまい、結局複雑になる。  
```go
type DatabaseAccess struct {
  db *sql.DB
}

func (da DatabaseAccess) GetProductByDate(ctx context.Context, fromDate time.Time, toDate time.Time, title string) (Product, error) {
  q := gorm.G[Product](db).
    Where("created_date >= ?", fromDate).
    Where("created_date < ?", toDate);

  if title != "" {
    q = q.Where("title = ?", title).
  }

  return q.First(ctx)
}
```

では、どこに線を引くかというところだが、筆者の基準としてはここだろうと考える。
- 単一のテーブルにしかアクセスしない
- with句、group by句、order by句がない
- テーブルの主キー、あるいはユニークキーの一致のみが条件となる
- select,update,delete,insertはすべて対象

ドメイン駆動設計のRepositoryなどは、以下のinterfaceが定義されることが多いだろう。  
```go
type PostRepository interface {
  GetPostById(id int) Post
}
```

これであれば、今後拡張されることもない。また、非常に多くの場面で利用される。  
プロジェクトによっては、一つのテーブルに複数のkeyがあり、ユニークキーによってレコードを特定することもあると思うが、こちらについても簡単なsqlのスコープ内だ。  

Select文、つまりReadな処理についてのみ言及してきたがUpdateやDeleteなどのMutateな処理についても同様だ。  
筆者の経験上、テーブルのレコードを一気に更新するケースというのは存在するが、それほど多くはない。  
基本的には、主キーをwhere句に、updateやdeleteを行う。insertはbulk insertされることはあるが、1レコードずつ行う場合のほうが圧倒的に多い。  

ここまで簡単なsqlの定義を行ってきたが、この簡単なsqlは、開発プロジェクトの中で頻繁に出てくる。  
簡単なsqlを`PostRepositry`のようなモジュールを定義せずとも簡単に実現できて、複雑なsqlは`PostRepository`でラップして実現する。というのが、筆者の基本方針だ。  

簡単なsqlは、非常に簡単なので単体テストで品質保証を行うモチベーションが低いので、メタコードで実現されていても構わない。  
むしろ開発プロジェクトで頻繁にでてくるので、メタコードで再利用性高く実装されていてほしい。  

## gorp
gorpの何が素晴らしいかというと、ここまで論じてきた簡単なsqlを実現するinterfaceが揃っていることだ。  
実のところ、筆者としてはもう一歩足りない部分があるのだが、それも簡単に拡張して、実現できる。  

以下のinterfaceである。  
https://github.com/go-gorp/gorp/blob/main/gorp.go#L92-L109
```go
type SqlExecutor interface {
	WithContext(ctx context.Context) SqlExecutor
	Get(i interface{}, keys ...interface{}) (interface{}, error)
	Insert(list ...interface{}) error
	Update(list ...interface{}) (int64, error)
	Delete(list ...interface{}) (int64, error)
	Exec(query string, args ...interface{}) (sql.Result, error)
	Select(i interface{}, query string, args ...interface{}) ([]interface{}, error)
	SelectInt(query string, args ...interface{}) (int64, error)
	SelectNullInt(query string, args ...interface{}) (sql.NullInt64, error)
	SelectFloat(query string, args ...interface{}) (float64, error)
	SelectNullFloat(query string, args ...interface{}) (sql.NullFloat64, error)
	SelectStr(query string, args ...interface{}) (string, error)
	SelectNullStr(query string, args ...interface{}) (sql.NullString, error)
	SelectOne(holder interface{}, query string, args ...interface{}) error
	Query(query string, args ...interface{}) (*sql.Rows, error)
	QueryRow(query string, args ...interface{}) *sql.Row
}
```

## goqu





ソフトウェアのトレーニングとして、会社のリポジトリを借りて、会社のシステムの作り直しを勝手に行っていた。
会社で使うかは知ったことではないが、自分で書いた部分のコードで、ドメイン知識が関係ない部分はプロジェクトテンプレートとして以下のプロジェクトにコミットしている。
https://github.com/motojouya/ddd_go



