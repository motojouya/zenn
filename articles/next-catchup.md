---
title: "フロントエンドにキャッチアップするならNext.jsと思った"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Next.js']
published: false
---

## intro

実際は、プログラマとしての書きやすさも重要

next.jsのススメ
https://future-architect.github.io/articles/20240228a/

## UIライブラリ
reactと他のUIライブラリとの比較
https://qiita.com/uhyo/items/ff243a5771077aaf4b5b

- qwik
  - クロージャの参照が切れるがその仕組
    https://zenn.dev/hedrall/articles/c15276e01725e0
  - server componentsは、ファイル単位という制約を実質持ち込める

## reactのメタフレームワーク
vs remix
- form
  - server components & server action
    - 自由度は高いが、知識がいる
  - loader
    - わかりやすいが画面に従属
  - nested layoutでの実現
  - 無限スクロールではremixならできるがnextはroute handler必要
- web標準
  - 要は実装しない。余計なことをしない
  - fetchの件はたしかに嫌
- SPAモード
  - github pagesでは、404をどう実装するかの違い
  - 404の実装はremixのほうが楽
  - github pages以外の場合ではいずれにしろ、nginxがいる
  - できるできないではなく、ファイルの配置の違いでしかない
  - 認証
    remix-authとnext-authの違い。db schema,役割の切り方
- 無限スクロールのfetch
  server actionsを使うやり方もありそう
  https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_1_interactive_fetch
  ただ、postでデータ取得するの抵抗ある

## 技術選択
特定の要件があるなら、それを選べばいい
- 一休remix
  https://user-first.ikyu.co.jp/entry/2023/12/15/093427
- デザイナじゃないので、reactやりづらいと言われたら言い返せない。

## キャッチアップするなら
- react前提
- reactの最新に触れて置きたい
- 逆にremixはnext.jsの知識を使えて、その逆はキャッチアップが多い
  - remixがよいフレームワークであるとも言えそう

## outro

