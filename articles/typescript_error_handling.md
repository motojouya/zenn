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

## 用語
この記事のタイトルは`TypeScriptのエラーハンドリングまとめ`だ。
これは、広い意味で`エラー`という単語を用い、エラーをどう扱うかという意味で`ハンドリング`としている。

JavaScriptには、`Error class`があるが、文中で言及する際は`Error class`と表現する。
それ以外に`エラー`とした場合は、より広義に意図しない挙動を表現する単語として用いる。

また、`例外`と`エラー`の区別はJavaScriptにおいて曖昧に感じる。
MDNの説明を見ると、throwされ、try catchでハンドリングするものを例外と読んでいるようなニュアンスに感じる。
https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Control_flow_and_error_handling#%E4%BE%8B%E5%A4%96%E5%87%A6%E7%90%86%E6%96%87

この文章上は、throwすることも、try catchすることも、コーディングの選択肢と捉えフラットに評価したい。
また、広義には`エラー`という単語があり、それで十分表現できるはずだ。
よって基本的に`例外`という単語は用いず、throwやtry catchについては個別に言及していく。

## エラーの種類
エラーという単語は意図しない挙動としたが、それらも大まかに分類しておくべきだろう。

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
- Error class

すべて解説はするが、実用的にはobject,class,Errorだろう。

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

function isArithmeticError(err: any) err is ArithmeticError {
  if (!err || typeof err !== 'object') {
    return false;
  }

  return typeof err.type === 'string';
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
また、`Error class`はもともと`name`プロパティを持っており`Error`という値なので、区別のために上書きしておくべきだ。

`instanceof`で同一性を判定できるのは`Error class`でないクラスと同様だ。
良さとしては、throwした際にstack traceを取得できる点だろう。例外発生時にどのようなコールスタックなのか把握できれば、原因特定は格段に楽になる。

ただ、落とし穴として、`Error class`のプロパティは列挙可能とならない。
特に何も継承していないクラスでは、`this.prop = value;`とするとpropは列挙可能になるが、`Error class`はならない。

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
特にライブラリから投げられる`Error class`の内容をjsonに変換したいのであれば、こちらを使うべきだろう。

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

ただ、上記のようにcatch節で、どのエラーなのか判定しなければ、TypeScript上では型安全に使えない。
`Error class`や`RangeError class`は`message`プロパティを持っているが、そうでなければ、`message`プロパティがあるかどうかわからない。
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
エラーの表現は列挙し、それをthrowしたりreturnするとした。
throwするのはエラーだけだが、returnはエラーではなく正常な値もreturnする。
returnする際の値の表現をどうするかは、検討しておくべきだろう。

- boolean
- union
- tuple
- object
- class
- optional/maybe
- result/either

数が多いので、事前に列挙しておく。実用的にはunion,object,classあたりだろう。

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
const ERROR_ZERO_DIVIDE = 'ZERO_DIVIDE' as const;
type ErrorString = typeof ERROR_ZERO_DIVIDE;

function divide(left: number, right: number): number | ErrorString {
  if (left === 0) {
    return ERROR_ZERO_DIVIDE;
  }

  return left / right;
}
```

上記は文字列リテラルをエラーの表現とし、returnの表現としてUnionを用いたものだ。
上記であれば、以下のようにエラーを判定できる。

```ts
const result = divide(12, 3);
if (result === ERROR_ZERO_DIVIDE) {
  console.log('Zero Divide!');
}
```

### Tuple
あとにGolang Styleという表現で、プログラミングのスタイルを定義して紹介する。
それとは直接的に関係ないのだが、Go言語には関数が多値を返せる仕様になっているようだ。
多値を返すために、エラーも正常な値も区別してreturnすることができる。

JavaScriptではそんな機能はないが、配列を返すことで擬似的に表現はできる。
TypeScriptでやるならば、より厳密にTupleを用いるべきだろう。

```ts
const ERROR_ZERO_DIVIDE = 'ZERO_DIVIDE' as const;
type ErrorString = typeof ERROR_ZERO_DIVIDE;

type Result<E, A> = [E, null] | [null, A];

function divide(left: number, right: number): Result<ErrorString, number> {
  if (left === 0) {
    return [ERROR_ZERO_DIVIDE, null];
  }

  const calcResult = left / right;

  return [null, calcResult];
}
```

ただ、Tupleを用いる場合、上記ではたりない。

```ts
const [err, calcResult] = divide(12, 3);
if (!err) {
  console.log(calcResult + 10); // コンパイルエラー
}
```

上記では、TypeScriptのType Guardが働かず、`typeof calcResult = null | number`と判定されるためだ。
これは`Result`型にリテラル型の値が入っていないため、`Result`型がタグ付きUnionとして表現されていないためである。
```ts
type Result<E, A> = [E, null] | [null, A];
```

こうすれば治る。

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
TupleであってもUnionにタグがなければならない。とすれば、それはもうobjectでよいのでは？と思った読者もいるだろう。

```ts
const ERROR_ZERO_DIVIDE = 'ZERO_DIVIDE' as const;
type ErrorString = typeof ERROR_ZERO_DIVIDE;

type Result<E, A> =
| {
  hasError: true,
  error: E;
}
| {
  hasError: false,
  data: A;
};

function divide(left: number, right: number): Result<ErrorString, number> {
  if (left === 0) {
    return {
      hasError: true,
      error: ERROR_ZERO_DIVIDE,
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
  };
}

function <A>success(data: A) {
  return {
    hasError: false,
    data,
  };
}

function divide(left: number, right: number): Result<ErrorString, number> {
  if (left === 0) {
    return error(ERROR_ZERO_DIVIDE);
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
JavaScriptにおけるclassの実態はFunction classで関数なので、そう発想するのは自然な流れだろう。

TODO type guard効くか、ちゃんと調べておきたい

```ts
const ERROR_ZERO_DIVIDE = 'ZERO_DIVIDE' as const;
type ErrorString = typeof ERROR_ZERO_DIVIDE;

class CustomError<E> {
  public hasError = true;
  constructor(public readonly error: E) {}
}

class Success<A> {
  public hasError = false;
  constructor(public readonly data: A) {}
}

type Result<E, A> = CustomError<E> | Success<A>;

function divide(left: number, right: number): Result<ErrorString, number> {
  if (left === 0) {
    return new CustomError<ErrorString>(ERROR_ZERO_DIVIDE);
  }

  const calcResult = left / right;

  return new Success<number>(calcResult);
}
```

以下のように判定できる。これはobjectの例と同じだ。

```ts
const result = divide(12, 3);
if (!result.hasError) {
  console.log(result.data + 10);
}
```

上記の例は、継承などは利用していないが、super classで共通のメソッドを用意してもよい。
ただ、objectの例でも、当該のobjectを引数に受け取る関数を用意すれば、構造的には同じことができる。
もちろん、読んだ感触は違ってくるが、そういった関数、メソッドについては、大きな違いがあるわけではないので、ここでは言及しない。

## エラーの発生とハンドリングの場所

### ライブラリ
ライブラリで発生するエラーは基本的には`Error class`だろう。
ただschema validatorのzodなんかだと、`Error class`を投げるのではなく、`success: boolean`というプロパティを持つobjectを返すやり方も提供してくれる。

アプリケーションでも、エラーの表現は`Error class`で、throwするハンドリングで扱う場合は問題ないが、returnする際は扱いづらい。
ライブラリは利用時にwrapしてやるのがいいだろう。直接ライブラリのAPIを使うのではなく、それらを扱う共通の関数を定義して提供するほうがよい。
それならば、ライブラリの変更の影響を抑えられるし、wrapしているので、その中でtry catchしてやれば、returnするのも簡単だ。

ライブラリというよりフレームワーク的になってくるが、ライブラリから実行されるコードの場合、`Error class`をthrowしないと想定する挙動にならないものも存在する。
たとえば、HTTP status codeを500で返してほしいとか、DBのトランザクションをrollbackしてほしいと言った場合には、アプリケーションコードから`Error class`をthrowする必要がある。

### アプリケーション
アプリケーションコードでは、より自由に定義すればよい。
ただし、ライブラリを使うこともあるし、ライブラリに使ってもらうコードを書くこともある点には注意だ。

アプリケーションを書いていると、様々な関数を定義することになる。
対象的なのは、末端の単一の役割を持つような関数と、ロジック全体を表現するような関数があるということだ。
前者は処理の独立性、扱いやすさを追求する形になるだろう。逆に後者は、様々な関数やモジュールを組み合わせて、システムの要件を実現する。

前者で発生するエラーは限られたものだろう。反面、後者はありとあらゆるエラーが発生する可能性もある。

## 後処理
エラーが発生した場合に、どうするかも検討しておく。
どれか一つを後処理として実行するのではなく、複数のアクションを起こす場合のほうが多いだろう。

### 変換
エラーを何らかの形に変換することはよくあるだろう。
関数は文脈がある。例えば、DBレイヤでのSQLのシンタックスエラーは、呼び出し元からすれば、与える引数のバリデーションをすり抜けた結果と捉えるかもしれない。
エラーにはそれらの文脈を伴ってエスカレーションされるべきなので、エラーを変換するケースもある。

### フィードバック
これはエラーを利用者に表示するということだ。
表示の内容は、プログラム内部の事情など関係ないので、利用者にわかりやすいメッセージに変換することになる。

### ログ
エラーが発生した際にログに記録しておき、開発者が確認することで不具合に気づきやすくなる。

### 握り潰す
エラーが発生しても、問題としないこともままある。あるいは、特定の関数の文脈ではエラーであっても、大枠での処理の全体からみれば、エラーでないというケースもある。

## Coding Style
今まで、エラーハンドリングの要素について、様々な状況を検討してきた。
それらの要素には相性があり、お互いを活かせるやり方として、要素を組み合わせたCoding Styleがあるはずだ。

筆者が考えたもの（つまり検討が浅い）ものもあるが、3つのstyleを挙げてみる。挙げた3つ以外にも、相性のよいやり方があるかもしれないので、読者にも検討してみてほしい。

### Try Catch Style

###  Golang Style

###  Railway Oriented Style

## railway oriented styleにできるライブラリ

### Promise
- thenでつなぐ
- 非同期実行の場合でも型が消えるのは同じ。なのでrejectには型がつかない

### fp-ts
- fp-ts
- neverthrow
- effect

試したPR
https://github.com/motojouya/croaker/pull/42

### neverthrow

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
->結局Promiseのチェーンと同じなので、railway orientedと表現すべきでは。
https://typescript-jp.gitbook.io/deep-dive/type-system/exceptions#anatahaerwosursuruhaarimasen

# 利用
組み合わせて使う。
筆者は、こういうときはこう。こういうときはこう。というような感じ。webアプリ。

# Outro
エラーハンドリングについて、なるべく網羅的に解説してきた。
読者には、特定のコードベースにおいて、エラーハンドリングをどうするか、一定のルールを持って運用できる手助けになればと思う。

また、網羅的と言いつつ漏れがあったり、説明に間違いがある場合は追記、修正したい。その際は連絡いただきたい。

