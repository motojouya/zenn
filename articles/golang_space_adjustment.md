---
title: "go fmtに逆らって列揃えする方法があるんだぜ"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go"]
published: false
---

`/* */`を挟むだけだぜ

## Intro
ちょっとおもしろい雰囲気のタイトルで釣って、中身の薄い記事を読ませるのはダサいのでこれくらいにしておきたい。  
ちなみに中身は本当に大したことない。↑の一文で分かる人はそっとじしたほうがいい。  

最近Go言語を勉強しはじめていて、とても楽しいし、`go fmt`でフォーマットしてくれるのは非常に快適だ。  
だが、どうしても`go fmt`でフォーマットしてほしくないものもある。以降の文章では、どういう状況のときに必要なのか、どのようにするのかを解説していく。  

## 課題
`go fmt`は非常に便利なツールで、Go言語の見た目を揃えてくれる。  
その際に、空いたスペースを削除してくれる。厳密なルールは分かっていないが、structの定義において、型の記述の行位置はスペースを使って揃えるなど、かなりいい感じにしてくれる。  

だが、例えば、以下のようなものは揃えてくれない。
```go
// 揃ってる！
type User struct {
    Id           int       `db:"id,primarykey,autoincrement"`
    Email        string    `db:"email"`
    Name         string    `db:"name"`
    RegisterDate time.Time `db:"register_date"`
    UpdateDate   time.Time `db:"update_date"`
}

// 揃ってる！
func NewUser(id int, email string, name string, registerDate time.Time, updateDate time.Time) User {
    return User{
        Id:           id,
        Email:        email,
        Name:         name,
        RegisterDate: registerDate,
        UpdateDate:   updateDate,
    }
}

var now = time.Now()

// 揃ってない…
var records = []User{
    //      id, email, name, register_date, update_date
    NewUser(1, "test@example.com", "John Do", now, now),
    NewUser(2, "test.test@example.com", "Nihon Taro", now, now),
    NewUser(3, "test.test.test@example.com", "Nanashi Gonbe", now, now),
}
```

最後の`records`を定義するところが揃っていない。  
そもそも、リストの中身をこんな風に定義することはあんまりないので、基本は問題ない。  

だが、DBアクセスする関数をテストするときに、データを用意する必要があった。  
上記は、usersテーブルの中身になるわけだが、そのテストデータを、`user_test.go`の中に定義して、ファイル上でテストデータもテストロジックも完結するようにしたい。  

また、テーブルのデータというのは、cliでもguiクライアントでも、列が揃えられて出力されることもおおい。見た目上そちらに寄せたいというのも人情だろう。  
なので、上記の揃ってない部分を揃えたいわけだ。  

## 対策
冒頭と同じだが`/* */`を挟む。

```go
// 揃ってる！
var records = []User{
    //      id,email                             ,name                ,register_date,update_date
    NewUser(1 ,"test@example.com" /*           */,"John Do" /*      */, now /*    */, now /*  */),
    NewUser(2 ,"test.test@example.com" /*      */,"Nihon Taro" /*   */, now /*    */, now /*  */),
    NewUser(3 ,"test.test.test@example.com" /* */,"Nanashi Gonbe" /**/, now /*    */, now /*  */),
                                        // ↑コメントの前はスペースが挟まる
}
```

これで、列が揃う。zennのシンタックスハイライトが効いてるので伝わると思うが、コメント部分は暗い色のフォントになる。現代的なエディタであれば、だいたい同様だろう。  
したがって、`/* */`の部分は目立たない。  

コード上に記載しているが、`/* */`の前には必ずスペースが挟まる。  
スペースを入れ忘れて、列揃えをがんばって`go fmt`すると、ズレてしまうので注意が必要だ。  

## Outro
基本は`go fmt`に従うべきだろう。列揃えのためにスペースを入力していくのも非常に面倒だし、そのコードを後から改修する人の気持ちを想像するとなんとも言えない。  

だが、DBアクセスするテストデータを簡単に用意し、視覚的にテーブルのカラムの値が一目瞭然になっているのは、強いメリットのようにも感じる。  
テストコードだからいいだろうという気持ちもある。  

読者には用法用量を守って使ってみてほしいぜ。  

