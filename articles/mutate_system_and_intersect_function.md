---
title: "更新系の処理における工夫とオリジナルのIntersect関数"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["設計"]
published: false
---

## 導入
参照系の処理については、すでに[記事](TODO)にした。  
今回は、更新系について解説していく。  

個人で開発するなかで、いくつか工夫したことがあり、記事にしたい。  
DDDっぽい処理だが、DDDの教科書的な実装からはずれたものになっているはずだ。  

## 構成
前提として、集約の単位で大きくディレクトリを切る。以前書いたこちらの記事に解説が載っている。  
https://zenn.dev/openlogi/articles/write_uml_for_ddd

そのうえで、その集約の中にディレクトリを切っていく構成を取る。  
これについては、個人のプロジェクトとして開発を行っていたものの中で、テンプレートを抽出したプロジェクトを作った。  
ざっくりもってきただけなので、ビルドも通らない状態だが、大枠の説明としてはちょうどいい。  
https://github.com/motojouya/ddd_go

上記のテンプレートプロジェクトでは、他にもいくつかディレクトリがあるが、今回の登場人物は以下となる。  
- Model  
- Input  
- DomainService(Repository)  
- Controller  

以下にそれぞれの役割を説明する。  

### Model
いわゆるドメインモデルを表現する。ただ、基本的に項目が一致するという前提のもと、DBレコードも表現する。  
アプリケーションとしては、参照透過な関数や、定数定義、守るべきドメインルールなどが実装される。  
既存の概念とそれほど差異はないだろうし、DBレコードと兼用するケースも多いはずなので、それほど誤解を生むものでは無いはず。  

これは、参照系の説明と同じである。  

### Input
更新系においては、入力される情報のためのデータモデルを用意する。  
Modelと併用してもよいが、ModelはDB情報も持つので、入力される情報は別な方が扱いやすい。一つの項目に様々な概念を詰め込むと、読みづらくなるという印象が強い。  
また、Model上はnilがあり得ないが、入力の段階では入力がないことを表現するためにnilを扱ったりもある。このあたりはプロジェクト次第だが、経験的には分けておいたほうが、後々楽になる。  

また、InputはModelに変換する関数を定義する。参照系においても、DBから取得したModelをOutputに変換する関数が必要としたが、簡単なので例示しなかった。  
ただ、InputをModelに変換する際には、不足する情報を補ったりとややこしい。これについては実装の説明の中で解説する。  

### DomainService(Repository)
参照系においてはRepositoryに言及したが、更新系においてはDomain Serviceという概念のほうが説明が楽である。  
ただ筆者は、参照系で説明したRepositoryの中に、ここで説明するDomainServiceを組み込んで混ぜて、Repositoryと呼んでいる。  
どちらも集約ルートの構造体をやり取りするという点は同じであるし、集約内部のロジックを表現するという点についても同様である。  

ドメインモデルというのは、いくつかのドメインルールを持って、値やロジックの正しさが保証されている。  
これらが、ドメインモデル自体が持てる制約であればいいが、中には他のDBレコードとの関係の中に制約があることがある。  
代表的なものは、ユニークキーである。例えば、投稿のtitleは、一つのユーザに対して一意に決まらなくてはならないとする。  
これはDBに問い合わせないとわからない制約だ。  

ドメインモデルに実装できるものはModelに実装するが、DB問い合わせも含めてデータの整合性を保つ場合には、DBアクセスを行える名前空間である必要がある。  
このドメインモデルの作成/更新と、DBに問い合わせる制約をまとめて扱えるのが、Domain Serviceになる。  
まとめて扱うといっても、それぞれの詳細を呼び出して、まとめる役割としたほうがいいかもしれない。  

筆者の定義するDomain Serviceは集約ルートを受け付けて、集約ルートの更新と、DBの更新を行う。  
本当にそれらの処理を呼び出すだけのイメージとしてよい。

### Controller
ここではRepositoryとDomain Serviceを概念としては分けているが、Controllerはそれらの関数を呼び出すことで、ロジックを実現する。  
参照系のRepositoryで取得した集約ルートを、Domain Serviceで更新していくイメージとなる。  

## 実装

### 構造体
参照系の処理と同様、記事を投稿するサービスを想定するが、投稿とそれに対応するタグを想定する。  
以下がModelとなる。  

```go
type PostModel struct {
  Id       int
  Content  string
  AuthorId int
  Tags     []Tag
}

func NewPost(id int, content string, author UserModel) (PostModel, error) {
	// 必要であれば、ここでバリデーションしてドメインルールを表現する
	return PostModel{
		Id:       id,
		Content:  content,
		AuthorId: author.GetId(),
	}, nil
}

type TagModel struct {
  PostId  int
  TagName string
}

func NewTag(tagName string, post PostModel) (TagModel, error) {
	// 必要であれば、ここでバリデーションしてドメインルールを表現する
	return CommentModel{
		PostId:      post.Id,
		TagName:     tagName,
	}, nil
}
```

バリデーションというのは、ルールの適用である。そのルールの適用は、ドメインモデル自体が実行すべきだろう。  
構造体を作る際に適用できれば、もっとも情報の凝集度が高い状態を維持できるはずだ。  

また、以下がInputとなる。  
Inputは、Modelへの変換関数を実装する。  

```go
type PostInput struct {
  Content  string          `json:"content"`
  AuthorId id              `json:"-"`
}

func (pi PostInput) Transfer(_ PostModel, depends PostDepend, id int) (PostModel, PostModel, error) {
	post, err := NewPost(id, pi.Content, depends.User)
	return post, post, err
}

type TagInput struct {
  TagName string `json:"tag"`
}

type TagInputList struct {
  TagInputs []TagInput `json:"tag_list"`
}

func (til TagInputList) TransferList(post PostModel, _ interface{}) (PostModel, MutationList[TagModel], error) {
	if len(til.TagInputs) > 10 {
		return PostModel{}, MutationList[TagModel]{}, errors.New("Post should have below 10 Tags.")
	}

	matched, inputUnmatched, modelUnmatched := Intersect(til.TagInputs, post.Tags, MatchTagInput)

	var insertTags []TagModel
	for _, input := range(inputUnmatched) {
		insertTag, err := NewTag(post.Id, input.TagName)
		if err != nil {
			return PostModel{}, MutationList[TagModel]{}, err
		}
		insertTags := append(insertTags, insertTag)
	}

	// Tagには更新項目がないが説明のため記載
	var updateTags []TagModel
	for _, zip := range(matched) {
		updateTag, err := UpdateTag(zip.Horizontal, zip.Vertical.TagName)
		if err != nil {
			return PostModel{}, MutationList[TagModel]{}, err
		}
		updateTags := append(updateTags, updateTag)
	}

	post.Comments = slice.Concat(updateTags, insertTags)
	mutation := MutationList[TagModel]{
		Inserts: insertTags,
		Updates: updateTags,
		Deletes: modelUnmatched,
	}

	return post, mutation, nil
}
```

Input->Modelへの変換関数を定義すると書いたが、そのシグネチャがなぜこうなっているかの解説を後回しにしたので、よくわからないものになっているはずだ。  
これらを、解説していく。  

### Intersect関数
まずは関数のシグネチャではなく、内部で利用している`Intersect`関数からだ。  

```go
type Zip[V any, H any] struct {
	Vertical   V
	Horizontal H
}

func Intersect[V any, H any](verticals []V, horizontals []H, predicate func(V, H) bool) ([]Zip[V, H], []V, []H) {

	var matched []Zip[V, H] = []Zip[V, H]{}
	var verticalUnmatched []V = slices.Clone(verticals)
	var horizontalUnmatched []H = slices.Clone(horizontals)

	for vIndex := len(verticalUnmatched) - 1; vIndex >= 0; vIndex-- {
		var vertical = verticalUnmatched[vIndex]

		for hIndex := len(horizontalUnmatched) - 1; hIndex >= 0; hIndex-- {
			var horizontal = horizontalUnmatched[hIndex]

			matched := predicate(vertical, horizontal)
			if matched {
				matched = append([]Zip[V, H]{Vertical: vertical, Horizontal: horizontal}, matched...)
				verticalUnmatched = slices.Delete(verticalUnmatched, vIndex, vIndex+1)
				horizontalUnmatched = slices.Delete(horizontalUnmatched, hIndex, hIndex+1)
				break
			}
		}
	}

	return matched, verticalUnmatched, horizontalUnmatched
}
```

すでに利用例を示しているし、よくあるIntersect関数を拡張した概念であるので、なんのためのものかはわかってもらえるだろう。  
対応するであろう2つのリストから、実際に対応するものと、しないものを振り分ける関数だ。  
一般的なIntersect関数は、筆者が示したものと比較して、`Matched`しか返さない傾向があるが、それではデータベースから削除する対象や、新規追加すべきものがわからないので、片手落ちだと筆者は考えている。  

今回のTagの例でいえば、以下の叙述関数を渡して使うことになる。  

```go
func MatchTagModelInput(input TagInput, model TagModel) bool {
	return input.TagName == model.TagName
}
```

### Transfer関数
Input->Modelへの変換はInput構造体が機能として持っており、独特のシグネチャが定義されている。  
これは汎用的な型として定義している。  

```go
type TransferableWithId[Agr any, Dpd any, Nd Keyed] interface {
	Transfer(Agr, Dpd, int) (Agr, Nd, error)
}

type TransferableList[Agr any, Dpd any, Nd any] interface {
	TransferList(Agr, Dpd) (Agr, MutationList[Nd], error)
}
```

`TransferableWithId`は、Entity Keyを持つ構造体を作成する際に利用する型だ。  
Postを作る際に、集約ルートはPostなので、`Agr = Post`。User集約の情報に依存しているので、`PostDepend`が`User`を持ち`Dpd = PostDepend`となる。
```go
type PostDepend struct {
	User User
}
```

引数である各種の値は、自身が持っている。Postを作る際に必要な値を、以下の3つに分類し、それぞれを引数に受け取れるようにすることで、変換を実現している。  
- 自分自身の集約ルート
- 依存している他の集約の集約ルート
- 引数

またEntity Keyは今回はアプリケーション内部で発行するし、発行の際に副作用がでるので、別物として引数に取れるようになっている。  

`TransferList`は集約ルートが、リスト構造のオブジェクトを保持し、それを一度に更新したい場合に利用できるものだ。  
すでに解説したIntersect関数を使って、insert対象、update対象、delete対象を明らかにする。  

これらの関数がおもしろいのは、その関数の型である。
- `Transfer(Agr, Dpd, int) (Agr, Nd, error)`
- `TransferList(Agr, Dpd) (Agr, MutationList[Nd], error)`

入力値が抽象化されているのは当然だが、返り値が3つある。これは以下のように捉えられそうだ。  
- 関数の返り値: Agr
- 正常動作時の副作用: Nd,MutationList[Nd]
- 異常動作じの副作用: error

どこかで見た分類だなと思ったが、shellの出力の概念と似ている。関数の返り値、標準出力、標準エラー出力。  
ここではアプリケーションなので、InputがModelに反映された結果として、`Agr`があり、それらのDBに反映させるべき差分が`Nd`,`MutationList[Nd]`となり、エラーの場合は`error`となる。  
この関数の型は、意図せずとしてこうなったのだが、面白い視点を与えてくれた。  

### Helper関数
Transfer関数がgenericsな関数であるのは、helper関数を使って、特定の処理を簡単に実装するためである。  
そのhelper関数を説明する。  

```go
func CreateWithId[Agr any, Dpd any, Nd database.Keyed, In TransferableWithId[Agr, Dpd, Nd]](
	db database.Executor,
	aggregate Agr,
	depend Dpd,
	input In,
) (Agr, error) {

	var zeroAgr Agr
	var err error

	var zeroNd Nd
	num, err := db.GetMax(zeroNd) // 対象のテーブルにアクセスしてidの最大値を取得
	if err != nil {
		return zeroAgr, err
	}

	newAgr, node, err := input.Transfer(aggregate, depend, num)
	if err != nil {
		return zeroAgr, err
	}

	keys := node.Keys()
	rowNode, err := db.Get(node, keys...) // gorpを使っている。主キーで検索して主キーが一致するレコードがあるかを検査している。
	if err != nil {
		return zeroAgr, err
	}

	if existNode, ok := rowNode.(*Nd); ok && existNode != nil {
		return zeroAgr, errors.New("record already exists")
	}

	err = db.Insert(&node)
	if err != nil {
		return zeroAgr, err
	}

	return newAgr, nil
}

func Mutate[Agr any, Dpd any, Nd any, In TransferableList[Agr, Dpd, Nd]](
	db gorp.SqlExecutor,
	aggregate Agr,
	depend Dpd,
	input In,
) (Agr, error) {

	var zeroAgr Agr

	newAgr, mutationList, err := input.TransferList(aggregate, depend)
	if err != nil {
		return zeroAgr, err
	}

	if len(mutationList.Dels) > 0 {
		_, err := db.Delete(mutationList.Deletes...)
		if err != nil {
			return zeroAgr, err
		}
	}

	if len(mutationList.Upds) > 0 {
		_, err := db.Update(mutationList.Updates...)
		if err != nil {
			return zeroAgr, err
		}
	}

	if len(mutationList.Inss) > 0 {
		err := db.Insert(mutationList.Inserts...)
		if err != nil {
			return zeroAgr, err
		}
	}

	return newAgr, nil
}
```

上記は抽象化されているので、どんなデータモデルでも受け付けることができる。  
制約としては、特定のデータモデルしか更新できないようにしていることだ。  
筆者は、DomainServiceの実装において、一つのデータモデルの更新に集中したほうがいいという考えだ。  
複数のデータモデルを検証しながら、最後に一気にDB更新を行うロジックというのは、ごちゃごちゃしがちという印象がある。  

この記事では、`TransferableWithId`,`TransferableList`の2つの紹介に絞ったが、実際には他にもいくつかパターンがある。  
ただ、どれもgenericsの機能で抽象化され、様々な集約で同様の処理として利用している。  
helper関数の流れは以下だ。
1. 必要であればidの発行
2. Transfer関数を実行して、集約ルートを、引数で更新する
3. DBに反映させる

idの発行が必要だったり、必要なかったり。3のDBの反映の際にレコードがすでに存在すればエラーとするか、更新するのか。また複数レコードの更新を役割とするか。  
だいたい、それくらいのパターンしかない。ただ、特定の集約によっては、id以外に副作用を伴って取得する値（例えば日付とか）が存在する。  
また、DomainServiceは、DBに問い合わせる制約を表現すると、冒頭で言及した。  
そういった処理が必要であれば、無理に共通化せず、別途その集約のためにhelper関数を実装する。  

### DomainService
TODO interfaceだけ先に示してもいいかもしれない。
DomainServiceのinterfaceは以下になる。この実装は、今まで示してきた関数を組み合わせることで簡単に実現できる。  

```go
type PostService interface {
	CreatePost(user UserModel, input PostInput) (PostModel, error)
	UpdateTags(post Post, input TagInputList) (PostModel, error)
}
```


```go
type postService struct {
	gorp.SqlExecutor
}

func (ps postService) CreatePost(user UserModel, input PostInput) (PostModel, error) {
	var post PostModel
	depend := PostDepend{User: user}
	CreateWithId(ps.SqlExecutor, post, depend, input)
}

func (ps postService) UpdateTags(post Post, input TagInputList) (PostModel, error) {
	var depend interface{}
	Mutate(ps.SqlExecutor, post, depend, input)
}
```

実装としてはこれだけだ。  
DomainServiceとして、UserModelを引数に取るので、PostDependに包むなどの処理はあるが、基本的にロジックがない。  
ここまで削れるなら、このDomainService自体はテストが不要だと考えている。その代わりに、細かいパーツをテストしてロジックを担保する。  

このDomainServiceは、対象の集約ルートとInputを受け付ける。  
ドメインロジック全体をパッケージする役割なので、ドメインモデルを作る動作と、DB操作がパッケージされている。Controllerは、これを呼び出すだけになる。  

### Controller

```go
type AllPostInput {
	PostInput
	TagInputList
	UserInput
}

type PostController struct {
	PostService
	UserRepository
}

func (pc PostController) CreatePost(input AllPostInput) (PostOutput, error) {
	user, err := pc.UserRepository.GetUserById(input.UserInput.Id)
	if err != nil {
		return PostOutput{}, err
	}

	post, err := pc.PostService.CreatePost(user, input.PostInput)
	if err != nil {
		return PostOutput{}, err
	}

	post, err := pc.PostService.UpdateTags(post, input.TagInputList)
	if err != nil {
		return PostOutput{}, err
	}

	return TransferPost(posts), nil
}
```

PostServiceは、PostとTagを同時には更新しないので、controllerで呼び出す必要がある。  
したがって、基本的にはControllerを囲む形でDBのトランザクションを貼るべきだろう。  
PostServiceが一発で両方更新すれば便利かもしれないが、再利用性は下がる。また、先に示したhelper関数のようなものも使えないか、非常に複雑になるかだ。  
処理を単純に保つことで、再利用性を高めつつ、コード量も減らす工夫である。  
入力のInputは、webの入力やjson入力をbindする形になるが、embedして使えば、一度にbindすることができる。そのために、例挙したcontrollerでは`AllPostInput`を用意した。  

上記はPostを作成する処理だったが、更新の際は、`PostRepository`から、Postを取得してきて、それを更新する形になるだろう。  
その時呼び出す、PostServiceのメソッドシグネチャは以下のようになるだろう。  
`UpdatePost(post Post, input PostInput) (PostModel, error)`

これで、取得したPostを更新することができる。  

## まとめ
すでに述べたことだが、今回紹介したDomainServiceは、前回の参照系の記事で紹介したRepositoryの機能としている。  
どちらも、集約ルートとなる構造体をin/outとしてやり取りする。  
筆者としては、このRepositoryとDomainServiceをあわせた機能は、集約ルートとDBのデータをまとめて振る舞いを定義しているもののように感じる。  

これは、DDDで言及される、アプリケーション層が、DB操作とドメインロジックを併せ持つ、ドメイン層を操作するという構図にもなる。  
教科書的なDDDではないのだが、もしかしたら読者の参考になるかもしれない。  

