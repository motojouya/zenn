---
title: "更新系の処理における工夫とオリジナルのIntersect関数"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["設計"]
published: false
---

- domain serviceを作る。それを中心に
- 入力はinput。基本は1つのテーブルを更新する関数を定義する
- 複数のテーブルを更新する際は、controllerからそれだけ呼び出す
- inputはそれぞれ独立させるが、embedすることで同時にecho bindが効く。echoじゃなくてもいい。今後はgorilla/schema使いたい
- inputがtransferを持ち、modelのファクトリ関数を呼び出してドメインルールを適用する
- transferの帰り値は、AggregateRoot,DBdiff,errorなので、返り値、標準出力、標準エラー出力と同じ構成になる
- transferの帰り値を受けて、dbに反映させる。dbに問い合わせる制約は、集約内では同一なはずなので、集約単位でこの手続きを作る
- 手続きがあるので、コードはほぼ書かない。
- あとはdependか。依存している集約を一つの構造体に詰め込んで渡す
- transferの中でAggregateRootを更新するので、それが結果になり、DBへは変更テーブルのみ反映させる


