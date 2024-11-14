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
どのやり方がよくて、どれが悪いということではないが、それらのやり方をなるべく網羅してテーブルに乗せ、比較検討したい。当然try catchも含める。

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

### TypeScriptのコンパイルエラー
基本的には開発時に修正するものだろう。解決できなければ@ts-ignoreをつけることもあるかもしれない。
実行時に発生することは想定しない。

### 利用者が回復不可能なエラー
これはバグの類になるだろう。筆者は主にWebサービスを開発しているが、HTTP status codeでは500に該当するものだ。
利用者が回復不可能なため、開発者が対応する必要がある。

### 利用者が回復可能なエラー
開発者が想定し、利用者が原因となって引き起こすエラーだ。
不正な行為や、誤ったアクションによって引き起こされ、正しい使い方をすれば発生しない。

## エラーの表現
エラーの表現としては以下が挙げられる。

- boolean
- number
- string
- object
- class
- Error extends

すべて解説はするが、実用的にはobject,class,Errorだろう。

### boolean
true/falseでエラーか否かを表現できるので、最も簡易だが協力な例とも言える。

以下の例なら、falseがエラー側だろう。
```ts
function isNatural(val: number): boolean {
  return val >= 0;
}
```

ただ、trueがエラーになるように実装もできる

```ts
function isError(val: number): boolean {
  return val < 0;
}
```

### number
エラーコードとして定義されるものになる。
基本的には桁数を決めて整数で定義するだろう。

```ts
const NotFound = 404 as const;
const NoAuth = 403 as const;
const ServerError = 500 as const;
type ErrorCode = typeof NotFound | typeof NoAuth | typeof ServerError;
```

軽量ではあるが、ただの数字なので何を意図しているものなのかわかりづらいかもしれない。
また例はHTTP status codeを想定しているが、status codeでは表現しづらいものも当然ある。

### string
こちらはエラーコード的に定義されているものもあれば、単に文章になっているものもある。

```ts
const NotFound = 'NOT_FOUND' as const;
const NoAuth = 'NO_AUTH' as const;
const ServerError = 'SERVER_ERROR' as const;
type ErrorCode = typeof NotFound | typeof NoAuth | typeof ServerError;
```

エラーコードは型として定義できるが、以下のように任意の文章をエラーとして表現するのは限界があるだろう。
定義した関数のスコープを抜けるとただのstringであり、判別がつかない。

```ts
function someError() {
  return 'some error happend!';
}
```

### object
numberやstringと比較して、objectで包んでやると飛躍的に表現力が向上する。

```ts
type NotFoundError = {
  url_path: string;
  access_user_id: string;
  message: string;
};
```

上記のようにメッセージの他、エラーの原因となった情報を入れることができる。
ただし、objectの同一性の判定は、少々コード量が増える。
TypeScriptにはType Guardという機能があり、同一性を判定したあとは、型が効くようにしておきたい。

```ts
function isNotFoundError(err: any) err is NotFoundError {
  if (!err || typeof err !== 'object') {
    return false;
  }

  return err.url_path === 'string' && err.access_user_id === 'string' && err.message === 'string';
}
```

上記の関数を定義しておけば判定できるが、関数を定義するのが面倒だ。
また、NotFoundErrorの場合は、あまり見ない`path`という変数が入っているので判定し易いが、エラーの原因となる値の名称は被ってしまうことも考えられる。

除算と乗算の例を考えてみる。
```
// 除算はright = 0の際に割り切れないのでエラーとする
type DivisionError = {
  left: number;
  right: number;
  message: string;
};

// 乗算はleftがマイナスで、rightが整数でないときに、虚数になる可能性があり、該当時にエラーとする
type PowError = {
  left: number;
  right: number;
  message: string;
};
```

left, rightという命名が悪いという点はあるが、他に思いつかなかった。
こういったケースは、アプリケーションを書いているとあり得ないことではないだろう。

```ts
type DivisionError = {
  type: 'DIVISION_ERROR';
  left: number;
  right: number;
  message: string;
};
```

例えば、`type`という命名で共通のリテラル型を定義しておくのは手だ。項目名も`type`である必要はない。
`type`の文字列が重複しないように、何らかのルールを設けるか、あるいはエラーを中央集権的に管理してチェックしてもよい。

また、タグ付きUnion型として定義すれば扱いが楽になる。
`type`の値でtype guardが効くようになるためだ。

```
type DivisionError = {
  type: 'DIVISION_ERROR';
  left: number;
  right: number;
  message: string;
};

type PowError = {
  type: 'POW_ERROR';
  left: number;
  right: number;
  message: string;
};

type ArithmeticError = DivisionError | PowError;

function isArithmeticError(err: any) err is ArithmeticError {
  if (!err || typeof err !== 'object') {
    return false;
  }

  return err.type === 'string';
}

function divideAndPow(value: number, divide: number, pow: number): number | ArithmeticError {
  if (divide === 0) {
    return {
      type: 'DIVISION_ERROR';
      left: value,
      right: divide,
      message: 'division error!'
    };
  }

  // 正しいバリデーションではないが、簡易的に
  if (value < 0) {
    return {
      type: 'POW_ERROR';
      left: value,
      right: pow,
      message: 'pow error!'
    };
  }

  return Math.pow(value / divide, pow);
}

const result = divideAndPow(10, 0, 0.5);
if (isArithmeticError(result)) {
  if (result.type === 'DIVISION_ERROR') {
    console.log('division error!');
  } else if (result.type === 'POW_ERROR') {
    console.log('error error!');
  }
} else {
  console.log(result);
}
```

### class
以下のように定義できる。
```ts
class NotFoundError {
  constructor(
    public readonly url_path: string,
    public readonly access_user_id: string,
    public readonly message: string,
  ) {}
};
```

classで定義することの良さは、型判定が楽になることだ。
```ts
if (err instanceof NotFoundError) {
  // Type Guardが効くのでpathにアクセスできる
  console.log('err is NotFoundError with path: ' + err.path);
}
```

これはobjectと違い、classがTypeScript上だけではなくJavaScript実行時にも型として扱えるためだ。
ただし`instanceof`は、JavaScriptのprototype継承の仕組みを利用し、prototypeツリーのどこかに存在すればそのclassと判定されてしまうので、誤判定の可能性はある。
ただ、これも何らかの開発ルールを設ければ、それほど対処が難しいものでもないだろう。

また、継承の仕組みはエラーのカテゴライズに使えるかもしれない。
```ts
class ArithmeticError {
  constructor(
    public readonly left: number,
    public readonly right: number,
    public readonly message: string,
  ) {}
};

class DivisionError extends ArithmeticError {};
class PowError extends ArithmeticError {};
```

上記の例なら、`error instanceof ArithmeticError`でまとめて判定できるようになる。

### Error extends
classを更に標準ライブラリのError classを継承することもできる。
```ts
class NotFoundError extends Error {
  constructor(
    public readonly url_path: string,
    public readonly access_user_id: string,
    public readonly message: string,
  ) {
    super(message);
    this.name = 'NotFoundError';
  }
};
```

Errorクラスは`message`という項目を引数に取るので、superに与えるほうがよい。
また、Errorクラスはもともと`name`プロパティを持っており`Error`という値なので、区別のために上書きしておくほうがよいだろう。

`instanceof`で同一性を判定できるのはErrorでないクラスと同様だ。
良さとしては、throwした際にstack traceを取得できる点だろう。例外発生時にどのようなコールスタックなのか把握できれば、原因特定は格段に楽になる。

ただ、落とし穴として、Errorクラスのプロパティは列挙可能とならない。
特に何も継承していないクラスでは、`this.prop = value;`とするとpropは列挙可能になるが、Errorクラスはならない。

列挙可能というのは、具体的には`Object.keys()`の返り値に入るということだが、より実用的には列挙可能なプロパティは`JSON.stringify`で出力されるjsonのプロパティとなる。
列挙可能でない項目も`Object.getOwnPropertyNames`なら列挙できるので、`toJSON`関数を実装してやれば、`JSON.stringify`でも項目として出力できる。

```ts
class CustomError extends Error {
  static {
    this.prototype.name = "CustomError";
  }
  toJSON() {
    return Object.getOwnPropertyNames(this).reduce((acc, key) => ({
      ...acc,
      [key]: this[key],
    }), {});
  }
}

class NotFoundError extends CustomError {
  constructor(
    public readonly url_path: string,
    public readonly access_user_id: string,
    public readonly message: string,
  ) {
    super(message);
    this.name = 'NotFoundError';
  }
};
```

他にも`Error.prototype.toJSON`に実装することもできるが、できれば標準ライブラリの挙動は換えたくないだろう。
また、`JSON.stringify`の第二引数に、`replacer`と呼ばれる関数を与えることができ、そこで値を変換して出力することもできる。
特にライブラリから投げられるErrorの内容をjsonに変換したいのであれば、こちらを使うべきだろう。

## エラーハンドリング
エラーをどう表現するかは列挙してきたが、それをどうハンドリングするかは、また別の話題だ。

### throw error
エラーをthrowするパターンだ。最もオーソドックスな方法だろう。
発生箇所でthrowし、判定箇所でtry catchする。

```ts
function validateInt(val: number): void {
  if (!Number.isInteger(val)) {
    throw new RangeError('not integer! value: ' + val);
  }
}

try {
  validateInt(0.3);
} catch (e) {
  if (e instanceof RangeError) {
    console.log('RangeError! message: ' + e.message);
  }

  if (e instanceof Error) {
    console.log('Some Error! message: ' + e.message);
  }

  console.log('something happened!', e);
}
```

ただ、上記のようにcatch節で、どのエラーなのか判定しなければ、TypeScript上では型安全に使えない。
`Error`や`RangeError`は`message`プロパティを持っているが、そうでなければ、`message`プロパティがあるかどうかわからない。
throwする際のツラミはここで、`validateInt`の返り値はvoidでエラーの型が出てこない。したがって、どんなエラーが投げられるのか、コードを見に行かなければならない点だ。

反面、node.jsやブラウザ環境では、`Error`クラスを継承したエラーを投げるとstack traceが取得できる。
これは、エラーの原因の特定には非常に便利なので、bug fix時などには非常に役に立つ。

ちなみにthrowは`Error`クラスを継承したオブジェクトである必要もない。実用的かどうかはおいておいて、stringなども投げられる。

### return error
returnすると型で表現されるので、Type Guardは実装しやすくなるはずだ。

```ts
function validateInt(val: number): boolean {
  return Number.isInteger(val);
}

const validateResult = validateInt(0.3);
if (validateResult) {
  console.log('int!');
} else {
  console.log('not int...');
}
```

反面、throwは一番直近のtry catch節まで、関数のコールスタックを飛び越えて到達できるので、築一認識する必要はない。
ただ、returnする場合は、通貨するコールスタックすべてで、エラーがあったか、判定しなくてはならない。

```ts
function validateInt(val: number): boolean {
  return Number.isInteger(val);
}

function validateNatural(val: number): boolean {
  if (!validateInt(val)) { // いちいち判定の分岐を書く必要がある
    return false;
  }
  return val >= 0;
}

const validateResult = validateNatural(-1);
if (validateResult) {
  console.log('natural!');
} else {
  console.log('not natural...');
}
```

## return anything
- union
- tuple
- object
- optional/maybe
- result/either
- other class

## 発生場所
- ライブラリ
  wrapしたほうがいい。
  throw Errorであることが基本なので、Errorのpropを列挙できない問題があるが、列挙が必要か検討したほうがいい
  ライブラリ側でエラーを期待してrollbackしたり、500を返すことも
- アプリケーションコード
  - 末端の機能的な関数
  - IOのある関数
  - 参照等価な関数
    throwすると参照等価ではなくなる
  - IOや作用を起こす関数をまとめ上げる関数
  - DIなど、アプリケーションのロジックはないが、実行の準備をする関数

## コーディングスタイル
- try catch style
- golang style
- railway oriented style

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

TODO callbackで解決するパターンもあるっぽい。メジャーなイメージないけど
https://typescript-jp.gitbook.io/deep-dive/type-system/exceptions#anatahaerwosursuruhaarimasen

# 利用
組み合わせて使う。
筆者は、こういうときはこう。こういうときはこう。というような感じ。webアプリ。

## outro
エラーハンドリングについて、なるべく網羅的に解説してきた。
読者には、特定のコードベースにおいて、エラーハンドリングをどうするか、一定のルールを持って運用できる手助けになればと思う。

また、網羅的と言いつつ漏れがあったり、説明に間違いがある場合は追記、修正したい。その際は連絡いただきたい。

