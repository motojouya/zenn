---
title: "Data Transfer Objectを拡張してクリーンアーキテクチャの同心円の図を立体的に捉える"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["設計"]
published: false
---

# Data Transfer Objectを拡張してクリーンアーキテクチャの同心円の図を立体的に捉える

## 導入
ぶっちゃけクリーンアーキテクチャの書籍は1ページも読んだことがないのだが、自分で考えた実装パターンの説明にちょうどよさそうなので、概念を拝借したい。  
Webの記事などはいろいろなものを読んでいるので、そう理解が間違っているとは思っていないが、間違っていたらご指摘願いたい。  

ということで、個人的に有用だなと思った実装パターンを検討したので、解説していく。  
まずは用いる単語を整理した後に、どのような課題を解決したいのか、解決案としての実装パターン、それを適用したときのアーキテクチャ概要などに言及した後に、運用上でてきそうな課題、Pros/Consなどを述べていく。  

## 言葉の整理

### Data Transfer Object (DTO)
`Pattern of Enterprise Application Architecture`いわゆるPoEAAだが、日本語訳を有志で公開されているページがあったので引用させていただく。  
[データ転送オブジェクト](https://bliki-ja.github.io/pofeaa/DataTransferObject)  

リモートファサードのような、ネットワークあるいはプロセス跨ぎのコミュニケーションを行う場面が想定されているようだ。  
細かい関数を何度も呼び出すのは使用感として悪いので、ファサードを用意し、それらの値をまとめて返却する。その時にまとめる役割がData Transfer Object(以降DTO)であると。  

個人的な解釈としては、内部の細かい関数は関連が弱そうだなという印象がある。つまりDTOの内部の値もそれぞれが関連が弱く、ロジックで扱うような関連の強いデータの集まりとは少し違う印象だ。  
また、シリアライズ可能ともある。これはロジックを持たない値と捉えても良さそうではある。  

複数人に更新されてしまうので、ソースとして信頼度は低いが、Wikipediaにもページがあり、そちらには`DTO が自身のデータの格納と取り出し機能しか持たない`ともある。  
[Wikipedia DTO](https://ja.wikipedia.org/wiki/Data_Transfer_Object)  

関連が強くないデータとも述べたが、関連はロジックにおいて必要になるので、Transferするだけと捉えると、関連が強くないのではなく単にプリミティブな値を持つと捉えることもできるかもしれない。鶏が先か卵が先か。  
また現代においては、OOP以外のパラダイム、モダンな言語機能などもあるので、当時の状況で実装可能なパターンとして存在していたのかもしれない。  

これ以上は詳しい方に解説いただきたいが、よく聞くのは、`ロジックを持たない`、それゆえ`データを運ぶ`役割のみに利用されるという説明だ。  

とりあえず、なんとなくのイメージはついたのでは無いだろうか。  
本記事のタイトルは、`Data Transfer Objectを拡張してクリーンアーキテクチャの同心円の図を立体的に捉える`だが、このDTOの概念を拡張した新しい概念を定義し、より役割を持たせて、さらに利用場面を限定するような実装パターンの紹介である。  

### クリーンアーキテクチャの同心円の図
クリーンアーキテクチャといえば、この図。この図こそクリーンアーキテクチャ。という印象があるのでは無いだろうか。  
https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg
[The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)から引用

書籍を読んでいないので厳密な言及はさけさせていただくが、zennにも数多あるクリーンアーキテクチャの解説記事によると、依存関係の方向性を定めることが重要と言及されることが多い。  

図では4つのレイヤを定義している。  
- Enterprise Business Rules
  - Entities
- Application Business Rules
  - Use Cases
- Interface Adapters
  - Controllers
  - Gateways
  - Presenters
- Frameworks & Drivers
  - Web
  - Devices
  - DB
  - UI
  - External Interfaces

このあたりは筆者個人では直感的にこの依存関係は理想的と感じる。  
ただ、上記の中で、本記事で利用させていただく単語は`Entity(Entities)`のみだ。他は筆者が定義が曖昧なので利用は避ける。ただ、`Use Cases`や`Controllers`のようなコードは出てくるだろう。これは単に`手続きロジック`と呼びたい。  

図によると`Entity`は、どこにも依存しないプレーンなオブジェクトだ。データ構造に付随するロジックを持つ。どこにも依存していないので、気にせず変更することができる。逆に言えば、Entityの変更は他のモジュールにも波及しやすそうではある。  
`Entity`を取り回して扱うコードを、ここでは`手続きロジック`と呼びたい。`Entity`だけでは実現が難しい動的に確認すべき制約を実現したり、データの保存を指示するなどのロジックを持つものだ。  
クリーンアーキテクチャの図で言うと、`Use Cases`や`Controllers`に該当しそうだが、本記事では厳密な定義が不要で、かつ依存の方向性のみが明確になっていればよいので、`手続きロジック`と呼ばせてもらう。  

本記事のタイトルは、`Data Transfer Objectを拡張してクリーンアーキテクチャの同心円の図を立体的に捉える`だが、クリーンアーキテクチャの図で最も重要であろう依存関係はそのままに、新しい構造を提案する形になる。  
クリーンアーキテクチャの同心円の図の傾向は踏襲しつつ、別のクラス図も後半で提示する。  

## 背景としての課題
新しい実装概念を利用するには背景となる課題が必要だ。まずは、これまでで定義した`Entity`と`手続きロジック`のコードを以下に示す。  
サンプルコードはすべてGoを用いる。  

まずは`Entity`だ。  

```Go
package entity

import (
    "strings"
    "errors"
)

type Name string

func NewName(name string) (Name, error) {
    var trimmed = strings.TrimSpace(name)

    if trimmed == "" {
        return Name(""), errors.New("name should not be empty")
    }

    var length = len([]rune(trimmed))
    if length < 1 || length > 255 {
        return Name(""), errors.New("name must be between 1 and 255 characters")
    }

    return Name(trimmed), nil
}

type User struct {
    Id   int
    Name Name
    Age  int
}

func NewUser(id int, name Name, age int) *User {
    return &User{
        Id:   id,
        Name: name,
        Age:  age,
    }
}
```

TODO Value Objectは言及したほうがよさそう。インタフェース側のデータ構造と分離させるモチベーションとして強い型で表現したい、それによって計算ロジックに集中できるようにしたいあたりが理由なはず。
逆にこういう詰替えが不要な環境には、本記事で説明するパターンは不要かもしれない。値を入れて、取り回される分には、Entityは何にも依存していないので。

`Name`はいわゆる`Value Object`と呼ばれるものだとしてよいだろう。255文字の制約を設け、型で表現される形を取っている。  
また、以下は`手続きロジック`だが、Google翻訳で手続きは`procedure`だったので、package名としている。  

```Go
package procedure

import (
    "entity"
    "request"
    "idCreator"
    "repository"
)

type UserCreate struct {
    UserRepository repository.UserRepository
}

func NewUserCreate(repo repository.UserRepository) *UserCreate {
    return &UserCreate{
        UserRepository: repo,
    }
}

func (creator UserCreate) Execute(request request.UserCreateRequest) (*entity.User, error) {
    id, err := idCreator.create()

    name, err = entity.NewName(request.Name)
    if err != nil
        return nil, err
    }

    age = request.Name

    user = entity.NewUser(id, name, age)

    if err = creator.UserRepository.Save(user); err != nil {
        return nil, err
    }

    return user, nil
}
```

コードは省いているが、`UserCreateRequest`はプレーンなオブジェクトで、DTOと捉えてもいい。  
`idCreator`なるモジュールはidは発行してくれるモジュールだ。本記事の本筋とは関係ないので、簡易な形を取っている。  
`repository`は、`Entity`をDBに保存するモジュールだ。これは後から別途でてくるが、現段階では、この程度の説明で進める。  

`UserCreateRequest`がDTOだとして、たとえばWebアクセスで入ってきたパラメータを保持しているとしよう。  
あるいは上記のコードサンプルと違うが、直接HttpRequestを表現するオブジェクトからの取得としてもいい。  

その取得時に、サンプルコードでは6行使っている。筆者はこの記述に冗長さを感じているのが、課題感の根本だ。  
```Go
    name, err = entity.NewName(request.Name)
    if err != nil
        return nil, err
    }
    age = request.Name
    user = entity.NewUser(id, name, age)
```

サンプルで用いた`User`構造体は、項目が3つしかないが、これが100項目を持つ構造体だとしたらどうだろうか？100項目がフラットに並ばず、階層構造的になっていたとしても、その部分のハンドリングは非常にめんどうだろう。  
すくなくとも、手続き的な記述を記載する`procedure`の名前空間にこの記述を置くのは、場違いである感覚が強い。  
この課題を解決したい。  

## 実装
ここからが、この記事の本題となる。先程、説明を省いた`UserCreateRequest`の実装から見ていく。

```Go
type UserCreateRequest struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}
```

こんな感じだろうか。Go言語は構造体のtag情報(上記では`json:"name"`の部分)を利用してHttp Requestの中身を、構造体にマッピングするライブラリが多く存在する。  
`UserCreateRequest`の場合は、jsonデータを想定しているが、formの値などもマッピングして利用できる。  

この`UserCreateRequest`を拡張して使う。具体的にこうする。  

```Go
package request

import (
    "entity"
)

type UserCreateRequest struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func (req UserCreateRequest) GetUser(id int) (*entity.User, error) {
    name, err = entity.NewName(req.Name)
    if err != nil
        return nil, err
    }

    age = request.Name

    user = entity.NewUser(id, name, age)
}
```

事前に冗長だと述べていた関数をそのままこちらに移してきた形となる。これを利用すると`手続きロジック`はこう書ける。  

```Go
package procedure

import (
    "entity"
    "request"
    "idCreator"
    "repository"
)

type UserCreate struct {
    UserRepository repository.UserRepository
}

func NewUserCreate(repo repository.UserRepository) *UserCreate {
    return &UserCreate{
        UserRepository: repo,
    }
}

func (creator UserCreate) Execute(request request.UserCreateRequest) (*entity.User, error) {
    id, err := idCreator.create()

    user, err := request.GetUser(id)
    if err != nil {
        return nil, err
    }

    if err = creator.UserRepository.Save(user); err != nil {
        return nil, err
    }

    return user, nil
}
```

どうだろうか。`User`モデル内の個別の要素、特に`Name`は変換時に長さチェックが入るわけだが、そういった記述を含めて`request`パッケージに追いやったことで、細かい記述が消え、処理の流れの粒度が揃ったように感じられる。  
今までは`Name`だの`Age`だのという情報が混ざっていたが、修正後は`request`から`User`を取得して`repository`で保存する。という文章を表すコードに近くなった。

また、`request`packageのメソッドである`GetUser`が`error`を出す可能性があるのであれば、それは入力値の問題だろう。それも`procedure`のコードを見ることで想像できそうだ。  
バリデーション自体は、`entity`packageの`NewName`関数が担っている。文字数の制限というのは`Name`型の制約だからだ。そのロジック自体はどこにも漏れていない。単に、`request`packageで、`entity`packageの構造体に変換しただけだ。  

繰り返しになるが、`User`オブジェクトは項目が3つしかない。項目数が多いものを扱うときには大変になるはずだ。階層構造的な構造体とするのであれば、構造体ごとに変換メソッドを持たせて呼び出すこともできる。  
`手続きロジック`の中で個別の項目を取り扱うよりは見通しがよくなるはずだ。  

## そもそも詰め替えしないという選択肢について
TODO

## Outboundな処理への適用
これまでは、入力パラメータに`UserCreateRequest`というデータモデルを適用し、変換するという形をもって実装パターンとしてきた。言うなればInboundなデータ処理で適用させてきたわけだ。  
これが入力パラメータ(Inbound)ではなく、OutboundなDB schemaのデータモデルとするのであれば、どのようにできるだろうか？  

これまで`entity`は、どこにも依存しないデータモデルとしてきた。このあたりは考え方次第だが、`entity`自体がDB schemaを表現してもいいだろう。  
Goの場合は、データをマッピングする文化があるので、以下のようにすることもできる。  
```Go
type User struct {
    Id   int    `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}
```

その場合、`Name`が`string`となっているのは大きな違いでもある。独自定義型に変換できるライブラリもあるかもしれないが、すくなくとも前に解説した`NewName`関数によるバリデーションを暗黙的に行ってくれるものはないだろう。  
DB schemaのデータモデルを表現するのであれば、基本的には組み込み型を使うことになる。そして、`Name`型を使いたいのであれば、DB Schemaのデータモデルと`Entity`は別のものとして存在させ、変換するというやり方もあるはずだ。  

具体的にEntityとDBのデータモデルを変換する関数をDB側に持たせてみる。  

```Go
package db

import (
    "entity"
)

type User struct {
    Id   int    `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}

func FromEntity(entity *entity.User) *User {
    return &User{
        Id:   entity.Id,
        Name: string(entity.Id),
        Age:  entity.Id,
    }
}

func (user User) ToEntity() (*entity.User, error) {
    name, err := entity.NewName(user.Name)
    if err != nil {
        return nil, err
    }

    return entity.NewUser(user.Id, name, user.Age), nil
}
```

TODO repositoryの意味も定義が必要そう。
こういう構造になってくると、`repository`は存在意義が怪しい。ここで`repository`は`Entity`を引き受けて`Entity`で返す役割を持っているはずのものだからだ。  
DB schemaのデータモデル自身が、Entityへの変換関数を持っているのであれば、わざわざ`Repository`にEntityを渡す必要はなく、単にEntityをdbモデルに変換し、直接DBアクセス関数を呼んでやればよい。  

```Go
package procedure

import (
    "entity"
    "request"
    "idCreator"
    "db"
    // "repository"
)

type UserCreate struct {
    UserRepository repository.UserRepository
}

func NewUserCreate(repo repository.UserRepository) *UserCreate {
    return &UserCreate{
        UserRepository: repo,
    }
}

func (creator UserCreate) Execute(request request.UserCreateRequest) (*entity.User, error) {
    id, err := idCreator.create()

    user, err := request.GetUser(id)
    if err != nil {
        return nil, err
    }

    dbUser = db.FromEntity(user)
    err = db.Insert(&dbUser)
    if err != nil {
        return nil, err
    }

    return user, nil
}
```

つまり解釈としては、RepositoryがEntityを出し入れするというのであれば、RepositoryというのはDBアクセス役割の`Insert`関数と、変換役割の`FromEntity`関数を混ぜて使っているものと考えることもできそうだ。  
仮にdbに`ToEntity`関数を実装しないのであれば、`repository#GetUser`はこんな実装なのではないだろうか。  

```Go
package repository

import (
    "entity"
    "db"
)

func GetUser(id int) (*entity.User, error) {
    dbUser, err := db.GetUser(id)
    if err != nil {
        return nil, err
    }

    name, err := entity.NewName(dbUser.Name)
    if err != nil {
        return nil, err
    }

    return entity.NewUser(dbUser.Id, name, dbUser.Age), nil
}
```

上記はDB schemaのデータモデル取得と、変換の処理が混ざっている状態だ。  
これが`Name`型がないのであれば、こんなふうな解釈や工夫をする必要は無いのかもしれない。ただ、ひとたびValue Objectを運用しようとするときには、変換のロジックが必ず何処かに挟まるはずだ。  
そして、その変換とデータ取得が混ざった状態のコードは、責務の多いコードといえるだろう。現段階でどちらがいいとは言わないが、repositoryを使えば責務が混ざるが、DTO(ここでは`db.User`)で変換ロジックを実装すれば、混ざらないようにすることができる。  

## inbound/outboundに適用したときの、アーキテクチャをclass図で解説。つまりこれが図を立体的に見せる。という解説。

## outbound側は、プラガブルにしたいが、運用上付け替えるテストや、プロダクト成長時の移行時の付け替え機能の利用などの実例や考え方
## このパターンを適用する際にどう考えるべきか
## 締め

