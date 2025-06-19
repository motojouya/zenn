---
title: "リスト操作の科学"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "Go"]
published: false
---

# Intro
先日、このスライドを見たのだが、最近は自分もリスト操作に対して、どう実装するか考えることが多く、記事にしておきたい。
https://speakerdeck.com/syumai/go-generics-slice-manipulation

上記の記事はGo言語で、リスト操作する際に、for文ではなく、mapやfilterのような関数を紹介している。
この記事では、それらの関数、for文、またリストを引数に取る関数を実装する際のことなど、検討していきたい。

サンプルコードはTypeScriptだったり、Go言語だったりするが、そこはご容赦いただきたい。

# やらないこと
この記事はリスト操作についてなので、無限ループや、アプリケーションから見て無限に見えるリストは扱わない。
後者については、後でちょっとだけ言及する。
前者は以下のようなものだ。

```typescript
for (;;) {
  if (confirm("終了しますか？")) {
    break;
  }
}
```

上記を1行で書くと、以下のようになるわけだが、chrome devtoolsのコンソールで実行すると、無限に確認ダイアログが表示される。
`for (;;) { if (confirm("終了しますか？")) { break; } }`

while文でも実現できるが、for文は上記のようなこともできる。こういった使い方には言及しない。

# a
- 実装ルール
  - for文
  - map, filter, reduce
  - find
  - some, every
  - intersect, grouping
  - immutable文脈ではeachは使わない
  - そもそもIOがあるなら、for文。名前空間を切りたくない。returnしたいなど
- 関数を切る
  - リストだけ持つオブジェクトの問題点
  - モデルの振る舞いとしてのリスト操作の想定
  - golangのsambar/loでは多すぎる
  - 実用関数の実装
- 落ち穂ひろい
  - dbの値はsortして、無限長を想定 -> cascading reducer
  - リスト内包表記

# Outro
