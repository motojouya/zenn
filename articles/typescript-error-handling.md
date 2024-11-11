---
title: "TypeScriptのエラーハンドリングまとめ"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['TypeScript', 'fp-ts', 'neverthrow']
published: false
---

# Intro
これは[オープンロジアドベントカレンダー](https://qiita.com/advent-calendar/2024/openlogi)1日目の記事です。

TypeScriptというよりJavaScriptで機能的に提供されているエラーハンドリングはtry catch文だろう。
ただ、それには課題があり、それに対して他の言語の考え方を導入して解決したりというのが、よく見られる。

それらを整理し、エラーハンドリングをどのように書くべきか、考える補助になる記事をまとめて置きたい。

TODO callbackで解決するパターンもあるっぽい。メジャーなイメージないけど
https://typescript-jp.gitbook.io/deep-dive/type-system/exceptions#anatahaerwosursuruhaarimasen

# 課題
JavaScriptで機能的に提供されているエラーハンドリングはtry catch文であることはすでに述べた。
これがTypeScriptになり、TypeScriptが提供する型チェックの網をすり抜けてしまう仕様のため、どうしてもTypeScriptの機能を活かせない。

以下はMDNのRangeErrorの解説ページの内容だが、型を付けてみたものだ。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/RangeError
```ts
function check(n: number): void {
  if (!(n >= -500 && n <= 500)) {
    throw new RangeError("The argument must be between -500 and 500.");
  }
}

try {
  check(2000);
} catch (error) {
  if (error instanceof RangeError) {
    // Handle the error
  }
}
```

check関数は値を返さない`void`関数なわけだが、RangeErrorになる可能性もある。
外側のtry catchを書くべきか否かは、関数の型だけでは判断できない。

ではコードを読めばよいという意見もあるかもしれないが、上記の例は限りなく簡単な例であり、実際には更に深いcall stackの中で投げられている例外などは把握できようはずもない。
したがって、catch節で`instanceof`で判定するものも、関数の中を見ないと把握できないのだ。
このように関数のシグネチャを読むだけでは、関数の挙動が想像できないというのが、try catch文の課題だ。

# 整理
上記の仕様をカバーしようと、プログラマレベルでは様々な工夫をしているようだ。
どのやり方がよくて、どれが悪いということではないが、それらのやり方をなるべくすべてテーブルに乗せ、比較検討したい。
当然try catchも含めて比較検討する。

個別のやり方がある。というよりは、いくつかのプログラミングテクニックの複合としてコーディングスタイルが生まれているので、その各要素を並べた後、その組み合わせ方の例をいくつか述べていく。

## エラーと例外
まずは、単語について整理したい。
JavaScriptでは、標準classとしてErrorがある。エラーだ。
例外はどちらかというと、`例外処理`という単語で、try catch文を示すことがおおいのではないだろうか。

上記の定義に則るのであれば、この記事では基本的に例外という単語は用いない。

エラーについても`Error`クラスを継承していればエラーなのか？という点も曖昧だ。
エラーについては、何らかの不都合と定義してもいいかもしれない。少なくとも本記事では、`Error`クラスを継承していることを、エラーの条件とはしない。
`Error`クラスについては、後で出てくるので、明確に`Error`クラスと表現する。

エラーについては、幅の広い単語なので、次節で更に言及する

## エラーの種類
エラーと例外を区別したが、エラーにも様々あるだろう。

- アプリケーションの利用者の意図通りにならない挙動
- ライブラリが投げる`Error`クラス
- TypeScriptのコンパイルエラー

どれもエラーだ。

## エラーの表現
- numbar
- string
- object
- class
- error class

## エラーハンドリング
- return error
- throw error

## return anything
- union
- tuple
- object
- optional/maybe
- result/either
- other class

## コーディングスタイル
- try catch
- early return
- railway oriented

## railway orientedの適用
- ユースケース
- サービス
- IO
- utility

長いところに使いたい

## fp-ts
- fp-ts
- neverthrow
- effect

試したPR
https://github.com/motojouya/croaker/pull/42

## neverthrow

補うコード。これがあればfp-tsと同等のことができそう
```ts
const constant = (key, func) => (carry) => {

  if (carry.isError) {
    return carry;
  }

  const result = func(carry);

  if (result.isError) {
    return result;
  }

  return new Success({
    ...carry,
    [key]: result.result,
  });
}
```

## outro

