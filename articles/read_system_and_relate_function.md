---
title: "参照系の処理における工夫とRelate関数"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["設計"]
published: false
---

## 導入
筆者は自由にコードを書きたいので、自由に設計して自分が書きやすいように書いてきたのだが、いつのまにかDDDっぽい構造になっていた。  
あくまで、ぽいだけであり、細部は結構違うし、こだわりのルールなどもあるのだが、DDDってこんなふうにできたのかもなと思うなどした。  

参照する処理において、データベースからデータを引っ張ってくるモジュールとしては、Repositoryが定義されることが多いように思う。  
Repositoryの返すオブジェクトは、ドメインモデルであると言及される。また、筆者は集約を表現するものとしても扱いたい。  

これらの概念を説明し、参照系の処理において、どのようにRepositoryを定義し、利用し、工夫するかを説明していく。  
更新系の処理については、別途記事を書く予定。  

## 構成
前提として、集約の単位で大きくディレクトリを切る。以前書いたこちらの記事に解説が載っている。  
https://zenn.dev/openlogi/articles/write_uml_for_ddd

そのうえで、その集約の中にディレクトリを切っていく構成を取る。  
これについては、個人のプロジェクトとして開発を行っていたものの中で、テンプレートを抽出したプロジェクトを作った。  
ざっくりもってきただけなので、ビルドも通らない状態だが、大枠の説明としてはちょうどいい。  
https://github.com/motojouya/ddd_go

上記のテンプレートプロジェクトでは、他にもいくつかディレクトリがあるが、今回の登場人物は以下となる。  
- model  
- output  
- repository  
- controller  

以下にそれぞれの役割を説明する。  

### Model
いわゆるドメインモデルを表現する。ただ、基本的に項目が一致するという前提のもと、DBレコードも表現する。  
アプリケーションとしては、参照透過な関数や、定数定義、守るべきドメインルールなどが実装される。  
既存の概念とそれほど差異はないだろうし、DBレコードと兼用するケースも多いはずなので、それほど誤解を生むものでは無いはず。  

### Output
Modelで表現しない、アプリケーションのreturn値。webならresponseに該当するデータモデル。  

前提として、集約を大きく分けると言及した。Modelの構造もそれに準ずる形となり、他の集約の内部は知ることができない。  
知ってしまうと、ソフトウェア境界が曖昧になり、改修の際の影響範囲がわかりづらくなる。これは筆者の経験上、どこかできっちりと境界を引いたほうがいいという経験上の判断である。  

ただ、webのresponseにおいては、集約を跨いで項目を関連づけて値を返す場合もある。  
Modelは強い制約の元、しっかりと守られたものとして定義し、アプリケーションのreturn値は別の構造体を定義し、ModelからOutputに変換するという方法を取った。  
そのためのOutputモジュールである。  

DDDの文脈では、参照系の処理と、更新系の処理を完全に分離し、参照系モデルと更新系モデルを分ける方法が取られることもある。  
今回定義したModelは、更新系でのみ利用し、参照系はOutputのみしか使わない。という構成だ。  
その場合、今回定義したOutputモジュールは、DBレコードの情報をしることになり得るので、それはそれで情報が散らばるなという印象があり、今回はドメインモデルはModel、出力モデルはOutputとし、変換するという形を取った。  

### Repository
Modelをデータストアから取得する処理を定義する。  
データストアは、DBがわかりやすいのでそれをイメージしてもらってよいが、それに限らない。  
重要なポイントとしては、集約を返すということだ。  

Modelも当然集約を表現することになる。
以下のようなデータモデルを表現する際に、`PostRepisotry`が返すのは、`Post.Comments`に`Comment`を埋め込んだ状態の`Post`構造体ということだ。  
```go
type Post struct {
  Id       int
  Content  string
  AuthorId int
  Comments []Comment
}

type Comment struct {
  Id          int
  PostId      int
  Content     string
  CommenterId int
}
```

この場合、`Post`が集約ルートとなるので、集約ルート以下の構造体は埋め込まれて取得される。  
この記事では、この埋め込むという機能自体をRepositoryの主な役割と定義している。  

この埋め込むという処理は、ORMによっては勝手にやってくれる。GORMなんかはまさにその例ではあるが、ORMにまるっと任せてしまうと、ORMのAPIを一度呼び出すごとに、複数のSQLが発行されてしまい、挙動がわかりづらくなってしまう。  
発行されるsqlも隠蔽されて、チューニングがやりづらいというのが、筆者の個人的な経験である。もちろん、そういったORMは便利なので使うときは使うのだが、今回はこの部分を分解し、個別にsqlを評価できる形を取る。  

また、sqlの発行自体は、データアクセスレイヤを定義して、それを呼び出す形を取るので、Repositoryは、DBアクセスの詳細をしらないという形を取った。  

### Controller
Repositoryを呼び出して、Modelを取得し、各ModelをOutputに変換して、各Outputを紐づける。  
これらの処理はそれぞれの機能を呼び出すだけなので、それらの機能を呼び出すモジュールという立ち位置にある。  

更新系の処理でもcontrollerを利用するが、今回については参照系の処理のみ言及する。  

## 実装
すでに例に上げた`Post`と`Comment`がある、Post集約を実例に取って実装を説明していく。  
また、User集約が別途あり、Outputでは関連付けられる。  

### 構造体
構造体はすでに紹介したが、再掲する。  
以下がModelとなる。  

```go
type PostModel struct {
  Id       int
  Content  string
  AuthorId int
  Comments []CommentModel
}

type CommentModel struct {
  Id          int
  PostId      int
  Content     string
  CommenterId int
}
```

以下がOutputとなる。  

```go
type PostOutput struct {
  Id       int             `json:"id"`
  Content  string          `json:"content"`
  AuthorId id              `json:"-"`
  Author   *UserOutput     `json:"author,omitempty"`
  Comments []CommentOutput `json:""`
}

type CommentOutput struct {
  Id          int         `json:"id"`
  PostId      int          `json:"-"`
  Content     string      `json:"content"`
  Commenter   *UserOutput `json:"commenter,omitempty"`
  CommenterId int         `json:"-"`
}
```

他にもModel->Outputと変換する関数も必要だが、簡単なものなので、説明は省く。  
Outputには、User集約のOutputを保持できるようになっている。また、`AuthorId`は、Goの構造体としては値を持つが、jsonとしては不要なはずなので出力されない。  
User集約のOutputは、controllerによって紐づけが不要な場合も想定されるので、nullableでありomitemptyとしている。  

### 関連付け
Model->Outputの関連付けは単純なので省いたが、関連付けについては、工夫している。  
具体的には、以下の関数を利用している。  

```go
func Relate[B any, L any](branches []B, leaves []L, relateIfCase func(B, L) B) []B {

	if len(branches) == 0 {
		return branches
	}

	var related = make([]B, 0, len(branches))

	for _, branch := range branches {
		var workingBranch = branch

		for index, leaf := range leaves {
			workingBranch = relateIfCase(workingBranch, leaf)
		}

		related = append(related, workingBranch)
	}

	return related
}
```

いわゆるORMでいうところのRelationを表現させるための関数だ。  
これは、以下のように利用する。  

```go
func RelatePostComment(post PostModel, comment CommentModel) PostModel {
	if post.Id == comment.PostId {
		post.Comments = append(post.Comments, comment)
	}

	return post
}

relatedPosts := Relate(posts, comments, RelatePostComment)
```

これは同一集約内で`PostModel`と`CommentModel`を紐づける処理として定義したが、集約同士を紐づける際、つまり`PostOutput`と`UserOutput`を紐づける場合においても、同様の関数、同様の概念で実行できる。  

### Repository
Repositoryはデータストアから取得してきたデータを上記で説明した関連付けの機能を用いて、返す処理になる。

```go
func (pr postRepository) GetPostById(postIds []int) ([]PostModel, error) {
	posts, err := pr.DataStore.GetPostById(postIds)
	if err != nil {
		return nil, err
	}

	comments, err := pr.DataStore.GetCommentByPostId(postIds)
	if err != nil {
		return nil, err
	}

	return Relate(posts, comments, RelatePostComment), nil
}
```

これが、Postの取得パターンが増えてきたらどうすべきだろうか。  
Postの絞り込み部分は実装が違いそうだが、Commentの取得と関連付けは同じになりそうだ。  
関連付けたオブジェクトを絞り込むこともあるが、集約ルートから見て同一性を重要視するのであれば、同じになるはずである。  

```go
func (pr postRepository) GetPostById(postIds []int) ([]PostModel, error) {
	posts, err := pr.DataStore.GetPostById(postIds)
	if err != nil {
		return nil, err
	}

	return fetchComment(pr, posts)
}

func (pr postRepository) GetPostByTitle(title string) ([]PostModel, error) {
	posts, err := pr.DataStore.GetPostByTitle(title)
	if err != nil {
		return nil, err
	}

	return fetchComment(pr, posts)
}

func fetchComment(pr postRepository, posts []Post) ([]PostModel, error) {
	postIds := make([]PostModel, 0, len(posts))
	for _, post := range(posts) {
		postIds = append(postIds, post.Id)
	}

	comments, err := pr.DataStore.GetCommentByPostId(postIds)
	if err != nil {
		return nil, err
	}

	return Relate(posts, comments, RelatePostComment), nil
}
```

これならば、紐づけの処理は完全に共通化できる。集約の内部は、集約の外側に漏れないので、Commentを取得してPostに紐づけるのは、ここにしか定義されない。  
もちろん、GORMなどのORMを使えば、もっと便利に短くかけるのは間違いない。  
ただ、Postを取得する部分が、非常に複雑になった場合など、そのsql部分だけ切り出してテストができたら、挙動が見えやすくなる。この実装にも一定の良さは有るだろう。  

また、DDDっぽいと記事の冒頭で言及した通り、エンティティキーでの取得の返り値がsliceになっている。  
`GetPostById(postIds []int) ([]Post, error)`の部分である。  
通常、`Post`や`*Post`を返すと思うが、この理由は次のcontrollerで言及する。  

### controller
controllerではrepositoryを呼び出して、outputの紐づけを行う。  

```go
func (pc PostController) GetPostById(postIds []int) ([]PostOutput, error) {
	posts, err pc.PostRepository.GetPostById(postIds)
	if err != nil {
		return nil, err
	}

	var userIds []int
	for _, post := range(posts) {
		userIds = append(userIds, post.AuthorId)
		for _, comment := range(post.Comments) {
			userIds = append(userIds, comment.CommenterId)
		}
	}

	users, err pc.UserRepository.GetUserById(userIds)
	if err != nil {
		return nil, err
	}

	postOutputs := TransferPost(posts)
	userOutputs := TransferUser(users)

	return Relate(postOutputs, userOutputs, RelatePostUser), nil
}
```

上記のuserIdを取ってくる処理がcontrollerとしては邪魔くさいが、適宜Modelなどに外だし可能だ。説明のためにすべて書いている。  
Userとの紐づけを必要としないのであれば、usersを取得する必要はない。そこはcontrollerでcontrolできる。  

そして、このように他の集約のデータをcontrollerで紐づけるために、Repositoryから複数オブジェクトをsliceで返している。  
単一のオブジェクトしか取得できない場合、N+1問題が発生してしまうためだ。  

上記のcontrollerであれば、sqlの発行は3回で固定される。post,comment,userだ。  
発行されるsqlは、少ないほどよいが、とはいえ1,2回増えたところでレイテンシーにはそれほど影響はない。  
それよりも問題なのは、どれくらい発行されるか、コントロール不能になることだ。それがN+1問題である。  

## まとめ
正直、この記事を書いていてGORMでいいかもしれん。と何度かよぎった。  
ただ、GORMのようなライブラリの仕組みがどうなっているか知ることには意味があるし、クエリを一つ一つ管理しやすい形にもっていけば、チューニングなどはやりやすくなるだろう。  

筆者としては、細かい部品を組み合わせていく形のほうが、好きではある。  
また、今回の例で示した`PostOutput`と`UserOutput`の紐づけは、GORMを使っていると、逆にやりづらいかもしれない。Relate関数がGORMの内部にしか無いからだ。  
Relate関数は筆者の実装は遅そうだが、機能の概念としてははGORM内部で使われているものと似たようなものだろう。  

今回は参照系の処理についての記事だが、別の記事で更新系の処理についても解説する。  
読者の参考になれば嬉しい。  

