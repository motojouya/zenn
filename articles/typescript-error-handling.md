---
title: "TypeScriptでのエラー処理をまとめる"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['typescript', 'fp-ts']
published: false
---

## intro

## エラーと例外
定義
- エラー
- 例外

## エラーの種類
- ユーザが対処できるか否か
- typescriptに怒られるか否か
- IOがからむか否か

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

