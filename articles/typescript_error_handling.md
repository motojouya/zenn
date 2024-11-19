---
title: "TypeScriptのエラーハンドリングまとめ"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['TypeScript', 'fp-ts', 'NeverThrow']
published: false
---

# Intro
これは[オープンロジアドベントカレンダー](https://qiita.com/advent-calendar/2024/openlogi)1日目の記事です。

TypeScriptというよりJavaScriptで機能的に提供されているエラーハンドリングはtry catch文だろう。
ただ、それには課題があり、それに対して他の言語の考え方を導入して解決したりというのが、よく見られる。

それらを整理し、エラーハンドリングをどのように書くべきか、考える補助になる記事をまとめておきたい。

# 課題
JavaScriptで機能的に提供されているエラーハンドリングはtry catch文であることはすでに述べた。これがTypeScriptになると、TypeScriptが提供する型チェックの網をすり抜けてしまう仕様のため、どうしても機能を活かしきれない。

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

check関数は値を返さない`void`関数なわけだが、`RangeError`になる可能性もある。外側のtry catchを書くべきか否かは、関数の型だけでは判断できない。

ではコードを読めばよいという意見もあるかもしれないが、上記の例は限りなく簡単な例であり、実際には深いコールスタックの中で投げられている例外などは把握できようはずもない。したがって、catch節で`instanceof`で判定するものも、関数の中を見ないと把握できないのだ。
このように関数の型定義を読むだけでは、関数の挙動が想像できないというのが、try catch文の課題だ。

# 整理
上記の仕様をカバーしようと、プログラマレベルでは様々な工夫をしているようだ。
どのやり方がよくて、どれが悪いということではないが、それらのやり方をなるべく網羅してテーブルに乗せ、比較検討したい。当然try catchも含める。

個別のやり方がある。というよりは、いくつかのプログラミングテクニックの複合としてCoding Styleがあるので、その各要素を並べた後、その組み合わせ方の例をいくつか述べていく。

以下の項目について整理していく。
- 用語
- エラーの種類
- エラーの表現
- エラーハンドリング
- returnする値
- エラーの発生場所とハンドリングの場所
- 後処理
- Coding Style

## 用語
この記事のタイトルは`TypeScriptのエラーハンドリングまとめ`だ。
これは、広い意味で`エラー`という単語を用い、エラーをどう扱うかという意味で`ハンドリング`としている。

JavaScriptには、`Error class`があるが、文中で言及する際は`Error class`と表現する。それ以外に`エラー`とした場合は、より広義に意図しない挙動を表現する単語として用いる。

また、`例外`と`エラー`の区別はJavaScriptにおいて曖昧に感じる。
MDNの説明を見ると、throwされ、try catchでハンドリングするものを例外と呼んているようなニュアンスに感じる。
https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Control_flow_and_error_handling#%E4%BE%8B%E5%A4%96%E5%87%A6%E7%90%86%E6%96%87

この文章上は、throwしtry catchすることも、コーディングの選択肢と捉えフラットに評価したい。また、広義には`エラー`という単語があり、それで十分表現できるはずだ。
よって基本的に`例外`という単語は用いず、throwやtry catchについては個別に言及していく。

## エラーの種類
エラーという単語は意図しない挙動としたが、それらも大まかに分類しておくべきだろう。

### TypeScriptのコンパイルエラー
基本的には開発時に修正するものだろう。解決できなければ`@ts-ignore`をつけることもあるかもしれない。
基本的に実行時に発生することは想定しない。

### 利用者が回復不可能なエラー
これはバグや不具合の類になるだろう。筆者は主にWebサービスを開発しているが、HTTP status codeでは500に該当するものだ。
利用者が回復不可能なため、開発者が対応する必要がある。本来は、開発時にすべて取り除かれているのが理想だが、たとえばクラウドサービスの障害によって発生するなど、開発者がコントロール出来ないものもある。

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
- Error class

すべて解説はするが、実用的にはobject,class,Error classだろう。

### boolean
true/falseでエラーか否かを表現できるので、最も簡易だが強力な例とも言える。

以下の例なら、falseがエラー側だろう。
```ts
function isNatural(val: number): boolean {
  return val >= 0;
}
```

だが、trueがエラーになるように実装もできる。

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
  return 'some error happened!';
}
```

### object
プリミティブな値ではなく、objectで包んでやると飛躍的に表現力が向上する。

```ts
type NotFoundError = {
  url_path: string;
  access_user_id: string;
  message: string;
};
```

上記のようにメッセージの他、エラーの原因となった情報を入れることができる。
ただし、objectの同一性の判定は、少々コード量が増える。TypeScriptにはType Guardという機能があり、同一性を判定したあとは、型が効くようにしておきたい。

```ts
function isNotFoundError(err: any): err is NotFoundError {
  if (!err || typeof err !== 'object') {
    return false;
  }

  return err.url_path === 'string' && err.access_user_id === 'string' && err.message === 'string';
}
```

上記の関数を定義しておけば判定できるが、関数を定義するのが面倒だ。
また、NotFoundErrorの場合は、あまり見ない`path`という変数が入っているので判定し易いが、エラーの原因となる値の名称は被ってしまうことも考えられる。

除算と乗算の例を考えてみる。
```ts
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
ただ、こういったケースは、アプリケーションを書いているとあり得ないことではないだろう。

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

```ts
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

function isArithmeticError(err: any): err is ArithmeticError {
  if (!err || typeof err !== 'object') {
    return false;
  }

  return typeof err.type === 'string';
}

function divideAndPow(value: number, divide: number, pow: number): number | ArithmeticError {
  if (divide === 0) {
    return {
      type: 'DIVISION_ERROR',
      left: value,
      right: divide,
      message: 'division error!'
    };
  }

  // 正しいバリデーションではないが、簡易的に
  if (value < 0) {
    return {
      type: 'POW_ERROR',
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
  } else {
    console.log('pow error!'); // DIVISION_ERRORでないものはPOW_ERRORでしかないのをtypescriptは知っている
  }
} else {
  console.log(result);
}
```

### class
以下のように定義できる。
classなのでmethodを実装してもよいのだが、データ型としての機能は以下で十分だ。
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
  console.log('err is NotFoundError with path: ' + err.url_path);
}
```

これはobjectと違い、classがTypeScript上だけではなくJavaScript実行時にも型として扱えるためだ。
ただし`instanceof`は、JavaScriptのprototype継承の仕組みを利用し、prototypeツリーのどこかに存在すればそのclassと判定されてしまうので、誤判定の可能性はある。ただ、これも何らかの開発ルールを設ければ、それほど対処が難しいものでもないだろう。

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

### Error class
classを更に標準ライブラリの`Error class`を継承することもできる。
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

`Error class`は`message`という項目を引数に取るので、親コンストラクタに与えるほうがよい。
また、`Error class`はもともと`name`プロパティを持っており、stack traceの出力に使われる。そのままでは`Error`という値なので、区別のために上書きしておくべきだ。

`instanceof`で同一性を判定できるのは`Error class`でないクラスと同様だ。
良さとしては、throwした際にstack traceを取得できる点だろう。例外発生時にどのようなコールスタックなのか把握できれば、原因特定は格段に楽になる。

ただ、落とし穴として、`Error class`のプロパティは列挙可能とならない。
特に何も継承していないクラスでは、`this.prop = value;`とすると`prop`は列挙可能になるが、`Error class`ではならない。

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
また、`JSON.stringify`の第二引数に、`replacer`と呼ばれる関数を与えることができ、そこで値を変換して出力することもできる。特にライブラリから投げられる`Error class`の内容をjsonに変換したいのであれば、こちらを使うべきだろう。

## エラーハンドリング
エラーをどう表現するかは列挙してきたが、それをどうハンドリングするかは、また別の話題だ。

### throw
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

ただ、上記のようにcatch節で、どのエラーなのか判定しなければ、TypeScript上では型安全に使えない。`Error class`や`RangeError class`は`message`プロパティを持っているが、そうでなければ、`message`プロパティがあるかどうかわからない。
throwする際のツラミはここで、`validateInt`の返り値はvoidでエラーの型が出てこない。したがって、どんなエラーが投げられるのか、コードを見に行かなければならない点だ。

反面、node.jsやブラウザ環境では、`Error class`を継承したエラーを投げるとstack traceが取得できる。
これは、エラーの原因の特定には非常に便利なので、bug fix時などには非常に役に立つ。

ちなみにthrowは`Error class`を継承したオブジェクトである必要もない。実用的かどうかはおいておいて、stringなども投げられる。

### return
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

反面、throwは一番直近のtry catch節まで、関数のコールスタックを飛び越えて到達できるので、いちいち認識する必要はない。
ただ、returnする場合は、通過するコールスタックすべてで、エラーがあったか、判定しなくてはならない。

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

## returnする値
エラーの表現は列挙し、それをthrowしたりreturnするとした。
throwするのはエラーだけだが、returnはエラーではなく正常な値もreturnする。
returnする際の値の表現をどうするかは、検討しておくべきだろう。

- boolean
- Union
- Tuple
- object
- class
- Promise

上記のリストを解説していくが、実用的にはUnion,objectあたりだろう。

### boolean
エラーの表現の節でも取り扱ったが、そもそも値を返さない`void`な関数についてはbooleanに変更するだけでもよい。

```ts
function validateInt(val: number): boolean {
  return Number.isInteger(val);
}
```

これならば、`true`で問題なく、`false`でエラーであることが表現できる。

### Union
TypeScriptにはUnion型という表現がある。

```ts
function divide(left: number, right: number): number | RangeError {
  if (left === 0) {
    return new RangeError('zero divide');
  }

  return left / right;
}
```

上記は`Error class`をエラーの表現とし、returnの表現としてUnion型を用いたものだ。
上記であれば、以下のようにエラーを判定できる。

```ts
const result = divide(12, 3);
if (result instanceof RangeError) {
  console.log('Zero Divide!');
} else {
  console.log(result + 10); // numberとして解釈される
}
```

これ以降のものもUnion型を使うのだが、以降はTuple型のUnion、objectのUnion、classのUnionという形でUnionに与える型を限定したやり方になる。
したがって、ここでUnionとしているのは正常な値の型とエラーの型に一貫性のないものとする。

### Tuple
あとにGolang Styleという表現で、コーディングのスタイルを定義して紹介する。
それとは直接的に関係ないのだが、Go言語には関数が多値を返せる仕様になっているようだ。多値を返すために、エラーも正常な値も区別してreturnすることができる。

JavaScriptではそんな機能はないが、配列を返すことで擬似的に表現はできる。
TypeScriptでやるならば、より厳密にTuple型を用いるべきだろう。

```ts
type Result<E, A> = [E, null] | [null, A];

function divide(left: number, right: number): Result<RangeError, number> {
  if (left === 0) {
    return [new RangeError('zero divide'), null];
  }

  const calcResult = left / right;

  return [null, calcResult];
}
```

以下のように判定できる。

```ts
const [err, calcResult] = divide(12, 3);
if (!err) {
  console.log(calcResult + 10);
}
```

TODO 要確認 croakerのコードベースの問題？
ただ、筆者の環境では上記ではType Guardが効かず、`typeof calcResult = null | number`と判定されるケースがあった。
nullはタグ付きUnion型のdiscriminatorとなれる型なので、判別可能なはずで、原因はわからない。

ただ、その場合も、明示的にdiscriminatorを入れてやれば治った。

```ts
type Result<E, A> = [true, E, null] | [false, null, A];

const [hasErr, err, calcResult] = divide(12, 3);
if (hasErr) {
  console.log(calcResult + 10); // コンパイルエラーにならない
}
```

また、Type Guardが効くので以下のように表現してもよい。

```ts
type Result<E, A> = [true, E] | [false, A];
```

### object
Tupleは順番で値の位置を確認するが、名前でアクセスできたほうが便利かもしれない。そういった場合はobjectという選択肢がある。

```ts
type Result<E, A> =
| {
  hasError: true,
  error: E;
}
| {
  hasError: false,
  data: A;
};

function divide(left: number, right: number): Result<RangeError, number> {
  if (left === 0) {
    return {
      hasError: true,
      error: new RangeError('zero divide'),
    };
  }

  const calcResult = left / right;

  return {
    hasError: false,
    data: calcResult,
  };
}
```

上記関数も、少し記述が冗長になってきた。以下のようにhelperを定義すると楽かもしれない。

```ts
function <E>error(error: E) {
  return {
    hasError: true,
    error,
  } as const;
}

function <A>success(data: A) {
  return {
    hasError: false,
    data,
  } as const;
}

function divide(left: number, right: number): Result<RangeError, number> {
  if (left === 0) {
    return error(new RangeError('zero divide'));
  }

  const calcResult = left / right;

  return success(calcResult);
}
```

以下のように判定できる

```ts
const result = divide(12, 3);
if (!result.hasError) {
  console.log(result.data + 10);
}
```

### class
helper関数を定義しなくても、classでいいのでは？と思った読者もいるだろう。
JavaScriptにおけるclassの実態はFunctionで関数なので、そう発想するのは自然な流れだろう。

```ts
class CustomError<E> {
  public readonly hasError = true;
  constructor(public readonly error: E) {}
}

class Success<A> {
  public readonly hasError = false;
  constructor(public readonly data: A) {}
}

type Result<E, A> = CustomError<E> | Success<A>;

function divide(left: number, right: number): Result<RangeError, number> {
  if (left === 0) {
    return new CustomError(new RangeError('zero divide'));
  }

  const calcResult = left / right;

  return new Success(calcResult);
}
```

以下のように判定できる。

```ts
const result = divide(12, 3);
if (!result.hasError) {
  test(result.data);
}

function test(val: number) {
  console.log(val + 10);
}
```

classで実装することのメリットは、super classで共通のメソッドを用意できることだ。
しかし、objectの例でも、当該のobjectを引数に受け取る関数を用意すれば、構造的には同じことができる。
もちろん、読んだ感触は違ってくるが、そういった関数、メソッドについては、大きな違いがあるわけではないので、ここでは言及しない。

### Promise
Promise型を返すという方法もある。

```ts
function divide(left: number, right: number): Promise<number> {
  if (left === 0) {
    return Promise.reject(new RangeError('zero divide'));
  }

  return Promise.resolve(left / right);
}
```

ただ、この場合は、エラーの型は消える。関数のシグネチャにも現れていないだろう。
以下のように、catch関数の中で、type guardを更に追加で入れてやらないと型判定されない。しかし、catch関数の場合は、`err`の型はanyになるので、コンパイルエラーとはならないようだ。

```ts
const result = divide(12, 3);

result
  .then(calcResult => console.log(calcResult + 10))
  .catch(err => {
    if (err instanceof RangeError) {
       console.log('zero divide!');
    } else {
       console.log('到達不能');
    }
  });
```

Promiseはtry catch節で扱うこともできる。エラーの型が消えるのは同じだ。
ただし、catch節の場合は、eがunknown型となるので、コンパイルエラーとなり、より安全に処理できる。

```ts
const result = divide(12, 3);

try {
  const awaitedResult = await result;
  console.log(awaitedResult + 10);

} catch (e) {
  if (e instanceof RangeError) {
     console.log('zero divide!' + e.message);
  } else {
     console.log('到達不能');
  }
}
```

## エラーの発生場所とハンドリングの場所

### ライブラリ
ライブラリで発生するエラーは基本的には`Error class`だろう。
ただschema validatorのzodなんかだと、`Error class`を投げるのではなく、`success: boolean`というプロパティを持つobjectを返すやり方も提供してくれる。

アプリケーションコード上でも、エラーの表現は`Error class`で、throwするハンドリングで扱う場合は問題ないが、returnしたい場合は扱いづらい。
ライブラリは利用時にwrapしてやるのがいいだろう。直接ライブラリのAPIを使うのではなく、それらを扱う共通の関数を定義して提供するほうがよい。それならば、ライブラリの変更の影響を抑えられるし、wrapしているので、その中でtry catchしてやれば、returnするのも簡単だ。

ライブラリというよりフレームワーク的になってくるが、ライブラリから実行されるコードの場合、`Error class`をthrowしないと想定する挙動にならないものも存在する。
たとえば、HTTP status codeを500で返してほしいとか、DBのトランザクションをrollbackしてほしいと言った場合には、アプリケーションコードから`Error class`をthrowする必要がある。

### アプリケーション
アプリケーションコードでは、より自由に定義すればよい。
ただし、ライブラリを使うこともあるし、ライブラリに使ってもらうコードを書くこともある点には注意だ。

アプリケーションを書いていると、様々な関数を定義することになる。対象的なのは、末端の単一の役割を持つような関数と、ロジック全体を表現するような関数があるということだ。
前者は処理の独立性、扱いやすさを追求する形になるだろう。逆に後者は、様々な関数やモジュールを組み合わせて、システムの要件を実現する。

前者で発生するエラーは限られたものだろう。反面、後者はありとあらゆるエラーが発生する可能性もある。
これは後述するCoding Styleを選択する際に効いてくる。限られたエラーだけならば、単純なコードだけでも可読性が損なわれないが、ありとあらゆるエラーが発生するのであれば、何らかの仕組みがあったほうが扱いやすいだろう。

## 後処理
エラーが発生した場合に、どうするかも検討しておく。
どれか一つを後処理として実行するのではなく、複数のアクションを起こす場合のほうが多いだろう。

### 変換
エラーを何らかの形に変換することはよくあるだろう。
関数は文脈がある。例えば、DBレイヤでのSQLのシンタックスエラーは、呼び出し元からすれば、与える引数のバリデーションをすり抜けた結果と捉えるかもしれない。
エラーにはそれらの文脈を伴ってエスカレーションされるべきなので、エラーを変換するケースもある。

### フィードバック
これはエラーを利用者に表示するということだが、よりプログラマ視点で、呼び出し元の関数にエラーを返すこともフィードバックと言えるだろう。
また利用者に表示する内容は、プログラム内部の事情など関係ないので、利用者にわかりやすいメッセージに変換することになる。

### ログ
エラーが発生した際にログに記録しておき、開発者が確認することで不具合に気づきやすくなる。

### 握り潰す
エラーが発生しても問題としないこともある。あるいは、特定の関数の文脈ではエラーであっても、大枠での処理の全体からみれば、エラーでないというケースもある。

## CodingStyle
今まで、エラーハンドリングの要素について、様々な状況を検討してきた。
それらの要素には相性があり、お互いを活かせるやり方として、要素を組み合わせたCoding Styleがあるはずだ。

筆者が考えたもの（つまり検討が浅い）ものもあるが、4つのstyleを挙げてみる。挙げた4つ以外にも、相性のよいやり方があるかもしれないので、読者にも検討してみてほしい。

- Try Catch Style
- Promise Chain Style
- Golang Style
- Railway Oriented Style

また、上記の4つの方法は、同期/非同期どちらのコードにも適用できる。
同期/非同期の違いは、それほど意識するようなものではないので、基本的には同期のパターンのコードで言及していく。

### Try Catch Style
最も基本的なやり方だろう。
`Error class`をエラーとしてthrowして、try catch文でハンドリングする。
複数のコールスタックを飛び越えて行くので、途中でcatchする必要はない。

- エラー
  `Error class`
- ハンドリング
  throw

```ts
function divide(left: number, right: number): number {
  if (right === 0) {
    throw new RangeError('zero divide!');
  }

  return left / right;
}

function middleFunc(first: number, second: number): number {

  // なんらかの処理

  const result = divide(first, second);

  // なんらかの処理

  return result;
}

function topLevelFunc() {
  try {
    const result = middleFunc(12, 2);
    console.log(result);
  } catch (e) {
    if (e instanceof RangeError) {
       console.log(e.message);
       return;
    }
    if (e instanceof Error) {
       console.log(e.message);
       return;
    }
    console.log('something happened!');
  }
}
```

これはすでに述べた通り、エラーの型が表現されていないので、実装を読まなければ型がわからない。またcatch節でそうやって調べた型でType Guardしなければ、messageすら読めないというところだ。
代わりにmiddleFuncのような中間のコールスタックでは何もする必要がない。

### Promise Chain Style
こちらも型が効かないパターンではあるが、Promiseで扱うこともできる。
エラーの表現は何でもよいのだが、一旦objectで表現する。エラーはreturnし、returnされるのはPromiseだ。

- エラー
  任意
- ハンドリング
  return Promise

```ts
type DivisionError = {
  left: number;
  right: number;
  message: string;
};

function isDivisionError(err: any): err is DivisionError {
  if (!err || typeof err !== 'object') {
    return false;
  }

  return err.left === 'number' && err.right === 'number' && err.message === 'string';
}

function divide(left: number, right: number): Promise<number> {
  if (right === 0) {
    return Promise.reject({
      left,
      right,
      message: 'zero divide!',
    });
  }

  return Promise.resolve(left / right);
}

function middleFunc(first: number, second: number): Promise<number> {

  // なんらかの処理

  const result = divide(first, second);
  return result.then(num => {

    // なんらかの処理

    return num;
  });
}

function topLevelFunc() {
  const result = middleFunc(12, 2);
  result
    .then(num => {
      console.log(num);
    })
    .catch(e => {
      if (isDivisionError(e)) {
         console.log(e.message);
         return;
      }
      console.log('something happened!');
    });
}
```

Try Catch Styleと同様、エラーの型は消えるので、type guardで検査してやる必要がある。
また、Chainでつなぐと、中間のmiddleFuncもPromiseを意識しなくてはならなくなる。

これなら、return表現をPromiseにしたとしてもtry catch文で処理するほうが現実的だろう。
これは提案するStyleの中で最も採用理由が薄いものだ。だが、後述するRailway Oriented Styleと似ており、わかりやすさのためにも挙げておく。

###  Golang Style
Golang Styleは筆者が勝手に呼んでいるものなので、他にいい命名があったら教えてほしい。
Go言語はエラーをreturnすると聞いたのでそう呼んでいるが、ここでreturnするのはUnion型なので、Go言語の多値というイメージとは違う。
Early Return Styleと呼ぼうかとも思ったが、Early ReturnはStyleというよりTechniqueというイメージなので、エラーの表現や返し方を含めて呼ぶにはふさわしくない。

実装は、エラーはclass、あるいは`Error class`で表現してreturnする。returnするのはUnion型だ。

- エラー
  class
- ハンドリング
  return Union

```ts
class DivisionError {
  constructor(
    public readonly left: number,
    public readonly right: number,
    public readonly message: string,
  ) {}
};

function divide(left: number, right: number): number | DivisionError {
  if (right === 0) {
    return new DivisionError(left, right, 'zero divide!');
  }

  return left / right;
}

function middleFunc(first: number, second: number): number | DivisionError {

  // なんらかの処理

  const result = divide(first, second);
  if (result instanceof DivisionError) {
    // もし必要であれば、middleFuncの文脈に沿ったエラーに変換してreturnする
    return result;
  }

  // なんらかの処理

  return result;
}

function topLevelFunc() {

  const result = middleFunc(12, 2);

  if (result instanceof DivisionError) {
    console.log(result.message);

  } else {
    console.log(result);
  }
}
```

察しのよい読者は気づいていると思うが、筆者がおすすめするやり方だ。なんならこの長い記事は、この段落のためにあると言ってもよい。
middleFuncのようなコールスタックの途中の関数であっても、エラーを意識してif分岐を書かなくてはならないのは面倒だが、確実に型が反映されるので、topLevelFuncでもエラー型を意識できる。
TypeScript的にUnion型は特徴的だが、エラーをreturnしている以外は至ってなんの変哲もないJavaScriptコードになる。

エラーをclassで表現しているのは、instanceofでtype guardが効くためだ。わざわざユーザ定義のtype guard関数を用意しなくてもよい。
蛇足だが、筆者はTypeScriptのコーディングにおいて、ほとんどclassは利用しないのだが、このエラーハンドリングにおいては、TypeScriptでもJavaScriptでも型表現として利用できるclassは便利に使っている。

`Error class`をエラーとして用いてもよく、ダメな理由はない。
後述するが、Try Catch Styleはどうしても併用する必要があり、したがって`Error class`はそちらで利用したいので区別したほうがわかりやすい。
区別のために、継承ツリー上に目印になるようなsuper classを定義してもよいが、わざわざ`Error class`を継承する理由もないというのが、単純なclassをオススメする理由だ。

###  Railway Oriented Style
こちらについては以下の記事に詳しい。
https://buildersbox.corp-sansan.com/entry/2024/03/26/110000

Domain Modeling Made Functionalという書籍でRailway Oriented Programmingという名前で紹介されている。
筆者は恥ずかしながら未読なので、用語や解説が間違っていたら指摘いただきたい。

Railway Oriented Programmingと銘打つからにはかなり特徴的であり、Programming Paradigmとして、コードベース全体に浸透させるべきものかもしれない。
ただここの段落では、Styleの一例として切り出せるもののみをピックアップして、Railway Oriented Styleと呼んで紹介する。コードベースの一部にのみ適用することも可能なStyleとしての説明だ。

Railway Oriented Styleを行うには、いくつかutility関数が必要であり、実際にはそれらが用意されたライブラリを利用することになるだろう。
以下の2つのライブラリで説明したい。
- NeverThrow
- fp-ts

ただ、上記のライブラリの利用例にはいる前に、基本的な考え方は抑えておきたい。

エラーの表現は自由だが、エラーはreturnし、returnするのはobjectで記載されている例が多いように思う。
以下のように表現し、returnする。例ではエラーは`Error class`で表現する。

- エラー
  任意
- ハンドリング
  return object

```ts
type Failure<E> = {
  isOk: false;
  error: E;
};

type Success<A> = {
  isOk: true;
  data: A;
};

type Result<E, A> = Failure<E> | Success<A>;

function divide(left: number, right: number): Result<RangeError, number> {
  if (right === 0) {
    return {
      isOk: false,
      error: new RangeError('zero divide!'),
    };
  }

  return {
    isOk: true,
    data: left / right,
  };
}

function pow(left: number, right: number): Result<RangeError, number> {
  if (left < 0) {
    return {
      isOk: false,
      error: new RangeError('Imaginary Number Possible!'),
    };
  }

  return {
    isOk: true,
    data: Math.pow(left, right),
  };
}
```

上記が下ごしらえとなるが、Promise Chain Styleの例のように、ここからutility関数を使って、より便利にハンドリングしていくというのが特徴だ。
簡易的な実装なら、以下のような感じになるだろう。

```ts
function pipe<E1, A1, E2, A2>(func: (data: A1) => Result<E1 | E2, A2>) {
  return function (beforeResult: Result<E1, A1>): Result<E1 | E2, A2> {
    if (!beforeResult.isOk) {
      return beforeResult;
    }
    return func(beforeResult.data);
  }
}

function callerFunc(val: number): Result<RangeError, number> {
  // 筆者が下手なせいで型引数が長いが、型引数は読み飛ばして構わない
  const divided = pipe<RangeError, number, RangeError, number>((calcVal) => divide(calcVal, 2))({ isOk: true, data: val });
  const powed = pipe<RangeError, number, RangeError, number>((calcVal) => pow(calcVal, 0.5))(divided);
  return powed;
}
```

Railwayというのは、始点から終点に流れる複線の線路が、複線どうしを移動できるイメージだ。
ここで複線の片方は正常系、もう一方はエラーだ。そして、今回紹介しているRailwayは行ったり来たりは出来ず、正常系からエラーに移動することはできるが、逆はできない。

上記の例をみると、正常な場合にはcallerFuncの最初から最後まで実行されるのは明確だろう。
ただエラーの場合にも、pipe関数を呼ぶcallerFuncではearly returnされずに最後まで実行される。これはpipe関数の中でbeforeResultが`isOk=false`の場合に次の関数を実行せずにエラーをreturnしているためだ。
つまり、callerFuncの中で、pipe関数で流れを繋いでおり、コードの頭から最後まで流れている。始点から終点までRailが流れる。
そして、pipe関数の中の制御で、エラーが来た場合は、常にエラー側の処理となり、正常系にはならない。

より具体的に、divideでエラーになれば、`pipe(pow)`でもそのエラーとなって、callerFuncの返り値になる。
divideでもpowでも計算ができれば、正常な値がcallerFuncの返り値になる。
この文章の上では、この概念をRailway Oriented Styleとして定義する。

#### NeverThrow
NeverThrowはPromise Chain Styleに似た形になるだろう。
ただPromiseと違い、ちゃんとエラーの型が効くので安心だ。

```ts
import type { Result } from 'neverthrow';
import { err, ok } from 'neverthrow';

function divide(right: number): (left: number) => Result<number, RangeError> {
  return function (left: number): Result<number, RangeError> {
    if (right === 0) {
      return err(new RangeError('zero divide!'))
    }

    return ok(left / right);
  }
}

function pow(right: number): (left: number) => Result<number, RangeError> {
  return function (left: number): Result<number, RangeError> {
    if (left < 0) {
      return err(new RangeError('imaginary number possible!'))
    }

    return ok(Math.pow(left, right));
  }
}

function callerFunc(val: number): Result<number, RangeError> {
  return ok(val + 10)
    .andThen(divide(2))
    .andThen(pow(0.5));
}
```

上記の例は簡単なコードだ。divideもpowもnumberを受け取って、numberを返すので、つなぐことができる。つまり基本的にはつなぐ関数の前後で、返り値と引数の型を一致させる必要がある。
では例えば、divideとpowの返り値を、掛け算するようなコードはどうすればよいのか。

途中の計算結果を、変数で保持するようなコードはNeverThrowにはサポートがないようだった。だからと言ってできないわけではなく、以下のようなhelperを用意すれば可能だ。
ちょっと型定義がややこしいが、fp-tsのbindの実装を参考にしている。もっといいやり方があるかもしれない。

```ts
function bind<N extends string, O extends object, E2, A>(key: Exclude<N, keyof O>, func: (data: O) => Result<A, E2>) {
  return function(beforeData: O): Result<{ [K in N | keyof O]: K extends keyof O ? O[K] : A }, E2> {
    return func(beforeData).map(a => ({
      ...beforeData,
      [key]: a,
    } as { [K in N | keyof O]: K extends keyof O ? O[K] : A })); // 筆者ではうまいやり方がわからずasを利用
  }
}

function callerFunc(val: number): Result<number, RangeError> {
  return ok({ val: val + 10 })
    .andThen(bind('divided', ({ val }) => divide(2)(val)))
    .andThen(bind('powed', ({ val }) => pow(0.5)(val)))
    .map(({ divided, powed }) => (divided * powed));
}
```

1点補足しておくと、returnする値をNeverThrowで定義したwrapper関数を利用して作っているため、ソースコードのありとあらゆるところで、NeverThrowに依存することになる。
これが嫌な場合は、エラーをthrowするコードを`fromThrowable`関数で囲ってやることで、エラーがthrowされたら`err`、正常な値なら`ok`として扱うことができる。

紹介したのは基本的な実装方法だが、NeverThrowではもっといろいろなことができるので、興味がある読者は調べて見てほしい。

#### fp-ts
fp-tsはchainでつなぐタイプではない。名前からも想像がつく通り、より関数型パラダイム寄りのライブラリだ。
こちらには、NeverThrowにはなかった`bind`関数が実装されているので、途中の計算結果を保持するのも簡単だ。

```ts
import { pipe } from "fp-ts/function";
import { Either, left, right, bindW, map } from 'fp-ts/Either';

function divide(rightNum: number) {
  return function (leftNum: number): Either<RangeError, number> {
    if (rightNum === 0) {
      return left(new RangeError('zero divide!'))
    }

    return right(leftNum / rightNum);
  }
}

function pow(rightNum: number) {
  return function (leftNum: number): Either<RangeError, number> {
    if (leftNum < 0) {
      return left(new RangeError('Imaginary Number Possible!'))
    }

    return right(Math.pow(leftNum, rightNum));
  }
}

function callerFunc(val: number): Either<RangeError, number> {
  return pipe(
    bindW('divided', () => divide(2)(val)),
    bindW('powed', () => pow(0.5)(val)),
    map(({ divided, powed }) => (divided * powed)),
  );
}
```

Result型に属するものはEither、あるいはTaskEitherになるだろう。ここではEither型の例をあげる。leftがエラーで、rightが正常値だ。rightは右という意味だが、正しいという意味でもあり、かかっている。Result型の順番も`Either<typeof left, typeof right>`となる。
fp-tsにもエラーをthrowする関数を扱うためのhelperがある。上記で利用しているEitherならtryCatch関数が利用できそうだ。

fp-tsについても、もっと様々なことができるので興味がある読者は調べて使ってみてほしい。

### Style 比較
様々なエラーハンドリングを見てきたが、Styleとしては、何を選んでもいいだろう。
関数型プログラミングパラダイムに慣れている開発者はfp-tsを選ぶだろうし、そもそも何も工夫しない標準的な書き方のほうがブレがないというならTry Catch Styleを選ぶだろう。（筆者はコミュニティの場末で、ひっそりとGolang Styleで開発したい。）

開発者の指向性はそれぞれでいいので論じるつもりはないが、コード量、特に行数についてはfp-tsなり、NeverThrowを使ったほうが少なくなりそうな予感がする。
筆者は、TypeScriptで自分用のwebアプリケーションを書いたので、そこでGolang Styleとfp-tsでコード量がどうなるかを比較した。Mergeしなかったが、以下がそのPRだ。
https://github.com/motojouya/croaker/pull/42

コードの詳細は説明しないが、行数にして対象の関数はGolang Styleで39行、fp-tsで38行になった。prettierの設定は120文字にしているので、折り返ししすぎているということはないだろう。予想に反して、行数はそれほど変わらなかった。
この1例だけで評価するのは公平ではないので結論とはしなくないが、行数の節約のためにfp-tsを導入したいという理由は、少し弱い意見となるかもしれない。

PRを診てもらうほうが比較としてはわかりやすいが、念の為コードも乗せておく。

:::details Golang Style vs fp-ts

- Golang Style
```ts
export const postCroak: PostCroak =
  ({ db, local, fetcher }) =>
  (identifier) =>
  async (text, thread) => {
    const trimedContents = trimContents(text);
    if (trimedContents instanceof InvalidArgumentsFail) {
      return trimedContents;
    }

    const nullableThread = nullableId("thread", thread);
    if (nullableThread instanceof InvalidArgumentsFail) {
      return nullableThread;
    }

    const croaker = await getCroaker(identifier, !!nullableThread, local, db);
    if (croaker instanceof AuthorityFail) {
      return croaker;
    }

    const createCroak = {
      croaker_id: croaker.croaker_id,
      contents: trimedContents,
      thread: nullableThread || undefined,
    };

    const links = await getOgps(fetcher, trimedContents);
    if (links instanceof FetchAccessFail) {
      return links;
    }

    const croak = await db.transact((trx) => trx.createTextCroak(createCroak, links));

    return {
      ...croak,
      croaker_name: croaker.croaker_name,
      has_thread: false,
      files: [],
    };
  };
```

- fp-ts
```ts
export const postCroak: PostCroak =
  ({ db, local, fetcher }) =>
  (identifier) =>
  (text, thread) =>
    pipe(
      TE.Do,
      TE.bindW("trimedContents", () => TE.fromEither(trimContents(text))),
      TE.bindW("nullableThread", () => TE.fromEither(nullableIdFP("thread", thread))),
      TE.bindW(
        "croaker",
        ({ nullableThread }) =>
          () =>
            getCroaker(identifier, !!nullableThread, local, db),
      ),
      TE.bindW(
        "links",
        ({ trimedContents }) =>
          () =>
            getOgps(fetcher, trimedContents),
      ),
      TE.bindW("croakData", ({ croaker, trimedContents, nullableThread }) =>
        TE.right({
          croaker_id: croaker.croaker_id,
          contents: trimedContents,
          thread: nullableThread || undefined,
        }),
      ),
      TE.bindW("croak", ({ croakData, links }) =>
        TE.rightTask(() => db.transact((trx) => trx.createTextCroak(croakData, links))),
      ),
      TE.map(({ croak, croaker }) => ({
        ...croak,
        croaker_name: croaker.croaker_name,
        has_thread: false,
        files: [],
      })),
      TE.toUnion,
    )();
```

:::

# 検討
エラーハンドリングについて、様々なStyleを挙げてきた。筆者はPrivateのコードはGolang Styleで書いているが、実際にはどうしてもTry Catch Styleが発生する場面が存在する。

エラーの発生場所に立ち返ってみると、ライブラリで発生するものは大抵のものが`Error class`をthrowする実装になっている。これを何処かでtry catchしなくてはならない。
筆者は基本的にライブラリはwrapして利用するので、wrapするコード上でtry catchし、エラーの形式を変換している。
また、ライブラリによっては、トランザクションをrollbackするために、`Error class`をthrowしなくてはならないものも存在する。

また、エラーの種類に立ち返ってみると、アプリケーションの利用者が回復不可能なエラーというのは、そもそも開発者が想定できていないものでもある。
想定可能なものは利用者が回復可能な形で実装すべきだが、仕様上の検討漏れや、クラウドサービスの障害など、アプリケーションで想定しづらいものも存在する。
これらは、そもそも想定できないのだから、対処としてはトップレベルの関数でTry Catch Styleで扱うほかない。

つまり、エラーの種類や、発生場所によってはTry Catch Styleで実装する必要があるということだ。
まとめると、[#エラーの発生場所とハンドリングの場所](#エラーの発生場所とハンドリングの場所)や[#エラーの種類](#エラーの種類)や[#後処理](#後処理)のやり方によって、[#エラーの表現](#エラーの表現)と[#エラーハンドリング](#エラーハンドリング)、場合によって[#returnする値](#returnする値)を検討する必要がある。
また、それらを集約する形で[#Coding Style](#CodingStyle)があり、参考にしたい。

読者も上記を検討した上で、TypeScriptを書いてほしい。面倒ならTry Catch Styleに統一するのも方法だろう。

# Outro
エラーハンドリングについて、なるべく網羅的に解説してきた。
読者には、特定のコードベースにおいて、エラーハンドリングをどうするか、一定のルールを持って運用できる手助けになればと思う。

また、網羅的と言いつつ漏れがあったり、説明に間違いがある場合は追記、修正したい。その際は連絡いただきたい。

