---
title: "Next.js App Routerの利用と工夫と感想"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Next.js', 'App Router', 'Auth.js']
published: true
---

## Intro
Webアプリケーションを作ったのだが、その際にNext.jsをフレームワークとして利用した。
サーバサイドは機能的に補うこともあり、そのあたりどうしたか、また利用しての感想などを記載していく。

途中でRemixもいいなーと調べて比べたりしたので、Remixへの言及も出てくるが、軽く調べた程度なので間違っているかもしれない。
また、ライブラリ自体の詳細な仕様や使い方には言及していないので、そちらは別の記事を参照してほしい。

## 利用ライブラリ
```
next: 14.2.15
next-auth: 5.0.0-beta.22
```
Next.jsはApp Routerを利用した。
またnext-authの5以降がAuth.jsとなる。

RemixではなくNext.js App Router（以降App Router）を選んだのはReact Server Component（以降RSC）を学びたかったから。

## 不足の機能
まずは、Next.jsを使う上で、補っておかなければならない機能について言及する。
App Routerで作成したが、Remixでも同様のケアをすべきものなので、ライブラリ移行しても適用できるはずだ。

Next.jsはWebフロントエンドのためのフレームワークであり、サーバサイドの機能をあまり持たない。
API Routesでリクエストを受け付けることができるが、受け付けるだけでその後のことは何もしてくれない。

今回はリレーショナルデータベースを利用するので、PrismaなりのDBアクセスライブラリは利用するとして、そこに至るまでのアプリケーションの記載を支援する仕組みが必要になる。
以下のようなものがあるだろう。
- Session管理
- リクエストハンドリング
- DI
- トランザクション管理
- ロギング

上記にたいしてどう用意すべきか検討していく。

### Session管理
Session管理は、Auth.jsに任せることにした。
Auth.jsはサーバサイドでsession情報を取り出す役目だが、クライアントサイドでも透過的にサーバからSession情報を取得できるAPIが用意されている。

ただし、基本的にはクライアントサイドでAuth.jsのAPIは用いいないことにした。
Auth.jsはDBスキーマまで口出してくるライブラリで、なるべく変更の影響を小さくしたかった。Auth.jsへの感想は後述する。
また、サーバサイドでも同様で、Sessionから取り出すのはユーザの特定に必要なuser.idのみで、あとの処理では、Auth.jsで指定されたスキーマは参照、更新を一切しないこととした。

クライアントサイドでSession情報を得る際には、App Routerのlayout.tsxでsession情報を取得し、クライアントサイドでReact Contextに登録して子コンポーネントから利用する仕掛けにした。
以下がlayout.tsxの実装例だ。

```tsx
import type { Metadata } from "next";
import "./globals.css";
import { SessionProvider } from "./SessionProvider"; // React Context実装
import { getUser as getUserFromDB } from "../getUser";
import { auth } from "../nextAuthOptions";

export const metadata: Metadata = {
  title: "App",
  description: "Something",
};

export default async function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  // sessionをRSCで取得し、React Contextにbind
  const session = await auth();
  const userId = getUserId(session);
  const user = await getUserFromDB(userId);

  return (
    <html>
      <body >
        <SessionProvider user={user}>
          {children}
        </SessionProvider>
      </body>
    </html>
  );
}
```

サーバサイドでも、userテーブルの他に、特定のユーザを表現するテーブルを別途作り、そこにだけuser_idという紐づけのカラムを用意することで、Auth.jsのスキーマに触れないようにした。
実装としては、後述するリクエストハンドリングを行う部分に実装を差し込むので、コード例はそちらで示す。

2024/11/07追記
以下のスライドによると、layout.tsxで認証しないほうがよいとのことだった。理由としては直感に反してpage.tsx -> layout.tsxが評価され、認証をlayout.tsxに頼っているとpage.tsx側で、認証未/済の両パターンを用意してしまい、ユーザに見えてしまうリスクがあるようだ。
https://speakerdeck.com/zaru_sakuraba/nextjssekiyuritei

今回作ったアプリケーションでは、Page.tsxで表現されている画面で見られて困るものはない。POSTな処理のときはキチンとバリデーション管理しているので、問題ない。
データとして見れてはならないものは、注意して実装しているので大丈夫だろう。
ただし、ベストプラクティスとしては、認証情報はlayout.tsxでbindしないほうがよさそうだ。

### リクエストハンドリング
ここでリクエストハンドリングとは、受け取ったHTTP Requestを解釈、バリデーションし、処理に引き渡すこと。
また処理から受け取ったデータを整形してHTTP Responseを返すこととして、定義する。

よくあるパターンだが、Zodを使って入力値のバリデーションを行い、型の制約を表現した。
また、例外発生時にはHTTP Status Codeをコントロールできる実装もしている。

上記の機能を果たすヘルパー関数を用意した。リクエストハンドリングではなくSession管理だが、Auth.jsの利用もここに実装している。
以下のようなコードになる。

```ts
import { NextRequest, NextResponse } from "next/server";
import type { Session } from "next-auth"; // Auth.js
import { z } from "zod";
import { auth } from "./nextAuthOptions"; // Auth.jsの設定ファイル
import { parse } from "./zodHelper";
import { getUserId } from "./sessionHelper";

type AppRouteHandlerFnContext = {
  params?: Record<string, string | string[]>;
};
type AppRouteHandlerFn = (
  req: NextRequest,
  ctx: AppRouteHandlerFnContext,
) => void | Response | Promise<void | Response>;

export function getRouteHandler<P extends z.SomeZodObject, R>(
  pathSchema: P | null,
  callback: (userId: string, path: z.infer<P> | null) => Promise<R>,
): AppRouteHandlerFn {

  return auth(async (req, { params }) => {

    try {
      const pathArgs = pathSchema ? parse(pathSchema, params) : null; // path parameterをzodにかける

      const userId = getUserId(req.auth); // SessionからuserIdを取得し、他のモジュールからはauth関数を参照しない。

      const result = await callback(userId, pathArgs); // 実際の処理実行

      return NextResponse.json(result);

    } catch (e) {
      if (e instanceof Error) {
        return new NextResponse(e.message, { status: 500 });
      } else {
        return new NextResponse("something happened!", { status: 500 });
      }
    }
  });
}
```

また、利用する際は、以下のように書く。

```ts
import { getRouteHandler } from "./routeHandlerHelper";
import { z } from "zod";

const pathSchema = z.object({
  val: z.coerce.number(),
});

// 実際はもっと複雑な処理だが、呼び出すアプリケーションの機能
const mainFunc = (userId: string, p: z.infer<pathSchema>) => ({ path: p.val });

// API routesの実装はGET,POSTという名前でexportする
export const GET = getRouteHandler(pathSchema, mainFunc);
```

上記はpath parameterのparseの実装だが、実際にはFormData, Request Body, Query Parameterなどのデータも処理できなくてはならない。
基本的にFormData, Request Body, Query Parameterは同時に利用される可能性は少なく、個別に実装すればよい。
以下の3パターンのためのヘルパー関数を用意した。
- Path Parameter + FormData
- Path Parameter + Request Body
- Path Parameter + Query Parameter

また、筆者の認識としては、Server Actionは透過的にサーバサイドの機能を呼び出せる機能であり、サーバサイドにあるという意識が強い。
なのでtsの型チェックがあるからと、リクエストを受け取ったあとのスキーマバリデーションを行わないのが非常に気持ち悪く感じる。
上記のようなリクエストハンドリングを行うヘルパーはServer Actionsのためのものもいくつか用意した。

以下のスライドでもServer Actionsで受け取った値はバリデーションすべきとしてある。
https://speakerdeck.com/zaru_sakuraba/nextjssekiyuritei

### DI
DIは依存性の解決であり、特にIOを生じさせるような処理をメインのロジックから切り離して、テストしやすくプラガブルにするために行う。
ただ、DIはclassベースのプログラミングパラダイムから生まれてきた単語で、今回はclassはかなり限定的に使っているので、厳密な意味でのDIは実装しない。

IOの処理を分離し、テストをしやすくするというのは、classを用いなくても重要なテクニックになるので、それを実現する機能は実装している。
解説の分量が多いので、別パートで解説する。

### トランザクション管理
トランザクションの処理は、フレームワーク的な機能としては用意しなかった。
よくあるパターンとしては、特定のコードをmiddlewareのような機能で囲い、そのmiddleware上でトランザクションが管理されるような形がある。が、そうなっていないので、フレームワーク的な機能ではないということだ。
代わりに、アプリケーションコードの中でトランザクションをコントロールできる関数を用意した。

理由としては、コンテナ環境へのデプロイであり、オブジェクトストレージへのアクセス、外部サービスへのアクセスなど、DB以外のIOも多様にある状況においては、一辺倒なトランザクション管理は適切でないと考えたため。
このあたりは、後日DB周りの実装についての記事を書くので、そちらで言及する。

ちなみに以下のようなイメージで使えるようになっている。

```ts
const handle = (db, s3) => async (userId: string, userName: string) => {
  const user = await db.getUser(userId);

  // 他のIOをトランザクション内で行うか否かを選べる
  await s3.put(user);

  // アプリケーションコードの中で、プログラマが意識的にトランザクションを開く
  await db.transact(trx => {
     return await trx.updateUser(userId, userName);
  });

  return await db.getUser(userId);
};
```

### ロギング
ロギングは今回は実装していない。
本当は考えなくてはならないのであげたが、今回は検討していないので解説しない。

## DIモジュールについて
DIモジュールと記載したが、厳密にはDIではない。

### 目的
DIは、様々な目的で利用されるが、そのすべての目的を達成しようとすると、やや重い実装になってしまう。
まずは、DIで実現していたことの整理をする必要がある。

- テストの際に、依存しているモジュールの影響を最小限に抑える
- 実行環境により、依存モジュールを切り換えて実行できるようにする
- 依存しているモジュールに更に依存があれば、再帰的に解決する

最後の再帰的にというのは、目的とは違うかもしれないが、今回実装するうえでは、着目しておかなくてはならない部分だ。
今回の実装は、上記3つのうち最初のテストのためにしか満たしていない。実行環境によって切り替えるのはそもそも今回は不要だった。再帰的うんぬんは後述する。

### 実装
実装としては、以下になった。
GetContext型の設定を用意し、setContextで紐づけて準備しておく。
実行時にはbindContext関数を使ってContextをbindして使う形になる。

```ts
export type GetContext = Record<string, () => unknown>;

export type Context<T extends GetContext> = {
  [K in keyof T]: T[K] extends () => infer C ? C : never;
};

export type ContextFullFunction<T extends GetContext, F> = {
  _context_setting?: T;
  (context: Context<T>): F;
};

// prettier-ignore
export function setContext<T extends GetContext, F>(
  func: ContextFullFunction<T, F>,
  contextSetting: T
): void {
  func._context_setting = contextSetting;
}

export function bindContext<T extends GetContext, F>(func: ContextFullFunction<T, F>): F {
  if (!func._context_setting) {
    throw new Error("programmer should set context!");
  }

  const contextSetting = func._context_setting;

  const context = Object.entries(contextSetting).reduce((acc, [key, val]) => {
    if (typeof val !== "function") {
      throw new Error("programmer should set context function!");
    }

    if (!Object.hasOwn(contextSetting, key)) {
      return acc;
    }

    return {
      ...acc,
      [key]: val(),
    };
  }, {}) as Context<T>;

  return func(context);
}
```

利用イメージとしては、まずはアプリケーションコード上で以下のように設定する。

```ts
import { getDatabase } from "./databaseHelper";
import { ContextFullFunction, setContext } from "./context";

// 注入する依存の実態。直接記載するので依存があるように見えるが、この時点ではまだbindはされていない
const getUserContext = {
  db: () => getDatabase(),
} as const;

export type GetUser = ContextFullFunction<
  typeof getUserContext,
  (userId: string) => Promise<User>
>;
export const getUser: GetUser = ({ db }) => (userId) => db.getUser(userId);

setContext(getUser, getUserContext);
```

上記を以下のように呼び出す

```ts
import { bindContext } from "./context";
import { getUser } from './getUser';

const userId = 1;
const user = await bindContext(getUser)(userId);
```

DIコンテナは大きく、設定ファイルを記載するタイプと、対象classにアノテーションで依存先を表現するタイプがある。
今回はアノテーションをつける形に近く、`getUserContext`という変数で`getUser`関数にbindする設定を行っている。
これは実行されるコードの設定が別ファイルだと、両方参照しないと全容が掴みづらいと考えたためだ。

上記はテストのためと述べた通り、ファイル名前空間上はimportしているので依存していると言える。
ただ、テスト自体は`getUser`関数にたいして`db`変数をモックすればよいので副作用を除外して書けるようになっている。

### 再帰的な解決
実装が重くなるという点もそうだが、必要性からも再帰的な解決ができるようには実装しなかった。
以下のコード例では、serviceとdbをbindしているが、実行時に`service.getUser`に`db.getUser`を渡している。
基本的なDIにおいては、serviceの中にdbを注入した状態で、依存が解決されているはずだが、それはしていない。

それが出来ないことを以下に示している。

```ts
import { getDatabase } from "./databaseHelper";
import { ContextFullFunction, setContext } from "./context";
import { getUser } from "./userService";

const getUserContext = {
  db: () => getDatabase(),
  service: () => ({ getUser }),
} as const;

export type GetUser = ContextFullFunction<
  typeof getUserContext,
  (userId: string) => Promise<User>
>;
export const getUser: GetUser = ({ db, service }) => async (userId) => {
  // db.getUserはcallbackとして実行時に与えているが、DIなら事前に注入する実装になるはず
  return service.getUser(db.getUser, userId);
};

setContext(getUser, getUserContext);
```

上記の名前空間が、直接`db.getUser`を使っていることがわかるので、何に依存しているかが明確だ。
これを再帰的に解決しようと、serviceの中に注入してしまうと、serviceの中で、どのようなIOが行われているか、コードを追う必要が出てくる。
この名前空間上で、何が行われているのか表現できたほうが、筆者としてはわかりやすく感じたため、このような実装になっている。

また、上記で依存を解決するものについては、特別な準備が不要だ。
上記の`service`という変数に代入したオブジェクトは、その場で即時的に作っている。`userService#getUser`は、ただの関数なのだ。
DIにおいては、classを用意し、constructorでインスタンス変数を用意するなどのDI用の実装がいるが、ここでは何でもDIできる。

## 感想
ここからは感想を書いていく。定量的な評価よりも感覚的なコメントが多くなっている。

### 書き心地
筆者はReactで開発する体験が素晴らしいと思っていて、感覚としてはほぼほぼ以下の記事で言及されている内容と同じだ。
https://qiita.com/uhyo/items/ff243a5771077aaf4b5b

UIを書く上で、少なからず書き方を覚えていかなくてはならないが、ReactはJavaScriptを書くということの延長で書ける。
だが、App Routerはサーバサイドで実行されるコードと、クライアントサイドで実行されるコードを分けてしまう。
分けてしまうと、変数の参照が切れてしまったりと、ケアが必要になる部分があるはずで、当初はその部分を懸念していた。

だが、クライアントで実行されるコンポーネントは`use client`ディレクティブで、Server Actionsは`use server`ディレクティブで明示的になるようになっているし、ファイル単位で分割もできる。
こういった機能によって、独立した名前空間ごとにサーバ/クライアントが分けやすいように設計されている。
独立した名前空間なので、変数の参照で特に意識することはなく、jsを書いているという感覚でいられるわけだ。

また、RSCからはClient Componentを呼べるが、その逆はできないという制約も、最初は混乱したが、書いていくと比較的自然にそういう風にかけることが多かった。RSCとClient Componentの使い分けも、それほど迷わずに考えられる。
すごいプログラマが設計したものというのは、いい出来になっているものだと感動した。

### Server Actions
最初はすべてURL設計をしてクライアントサイドと、サーバサイドをなるべく疎結合にしようと考えていた。
具体的にはすべてAPI Routesで実装し、Server Actionsを利用しないということだ。

ただ多くの場合において、サーバサイドの機能は、特定の画面でのみ利用される傾向が強い。
当然、疎結合な感じで、RESTfulな設計をした特定URLの機能を、画面が使うという構図が成り立つ画面もあるだろう。
だが、そうでないのであれば逆に画面に対するサーバ実装は専用のもの、つまり密結合な状態を目指してもいいはずだ。

そう考えた際に`画面 -> URL -> サーバ実装`というたどり方が面倒に感じ、取り払いたくなり、Server Actionsを利用した。

以降でも何度かRemixへの言及が出てくるが、API RoutesなしでPOSTな処理を作成することに対しては、Server Actionsのほうが優れていると感じた。
Remixではaction関数がServer Actionに対応するはずだが、こちらは特定の画面に対して1つしか作成できない。
他のaction関数を用意する場合は、URL設計をして別のAPI Routes（Remix上の単語としてはResource Routes）を用意する必要がある。
Server ActionsはURL設計なしでいくらでも作れるので、非常に便利だった。

CRUDでも、データを取得する処理はRのみだが、更新する処理はCUDで3つもある。
データはコンポーネントと1対1になっているほうがよいはずなので、必然的にデータ取得処理は1つでよい、あるいは1つに寄せていきたいはずだが、更新はそうではないはずだ。

### RSC
そもそもRSCを学びたくてNext.jsを使った。書き心地でも言及したが、最初はどういう機能なのかドキュメントを読んだが、今では自然に利用できる。

画面表示の際に、今まで`useEffect`で取得していたデータを、自然な流れでデータを取得して利用できるのは便利だった。
反面、クライアントサイドでは利用できないので、その場合はAPI RoutesでEndpointを作り、Client Componentから`useEffect`で取得しなくてはならない。

今回無限スクロールの実装があったので、初期表示の際にデータ取得するだけではなく、スクロール時にデータ取得する必要があった。
こういった画面ではRSCではなくClient ComponentからAPI Routeを利用する必要がある。

Remixでのデータ取得にはloader関数があり、特定の画面にloader関数が紐づいているようなので、わざわざAPI Routesを別に作らなくても無限スクロールの実装ができるようだ。
更新ではなく、データを取得するという点については、Remixのほうが便利そうだと感じた。
RSCの機能としては、無限スクロールの最適化のための機能ではないので、こういった言及は公平ではないが、少なくともApp Routerでは無限スクロールのためにAPI Routesの実装が必要になる。

### Remixのloaderとactionについて
App Routerを利用したが、Remixも使ってみたいという気持ちを持っていたので、常に比較しながら使っていた。Remixの機能についても少し補足しておく。

Remixのloaderとactionは、同じ名前空間でexportするコンポーネントがあれば、そのコンポーネント用のGET処理、POST処理としてURL設計不要で利用できる。
逆にexportするコンポーネントがなければ、ただただGET、POST用のAPI Routesとしての実装になる。
一つの関数を使い方で機能が変わるのは、ちょっとややこしいかもしれないが、実際の実装としてはほぼ同じになりそうで、逆にわかりやすいと感じた。

App Routerは、こういった機能設計にはなっていない。

### Auth.js
Auth.jsを見て最初にギョッとしたのは、DBスキーマの強制だ。
以下はPrismaの例だが、指定されたスキーマを作らなくてはならない。
https://authjs.dev/getting-started/adapters/prisma

RemixならばAuth.jsの利用もできるが、remix-authもある。
remix-authならばスキーマを指定しない。反面、DBに認可の連携情報を記録するのは、自分で実装しなければならないようだった。
https://remix.run/resources/remix-auth

基本的にはスキーマ設計というのはアプリケーションエンジニアのコントロール領域だという考えから、remix-authのほうがポジティブに感じる。
ただ、IDプロバイダーからの情報や、セッション管理の情報などは、基本的なデータモデルがあるはずで、たしかに理想的なスキーマは指定の形に落ち着くかもしれない。また、スキーマを改修する際も、拡張情報ならば別テーブルを用意したほうが理想的になる場面も多そうだ。

つまりAuth.jsの指定するスキーマを改修することは基本的に想定しなくていい。
最初はギョッとしたが、Auth.jsでも実害はなさそうだという印象を持った。

### TypeScriptの型レベルプログラミング
DIの実装においては、厳密には違うかもしれないが、初めていわゆる型レベルプログラミングというものを実装した。
入力の型から、出力の型を導出する技術で、関数のロジックの表現ができてしまう。
今回はそういったことは実装していないが、四則演算や、任意の文字列の整形なども、型として表現できるようだ。
かなり強力な機能なので、何らかの制限が必要だろうなと感じた。

制限としては、任意の数字や文字列ではなく、たとえば1-100までの整数や、特定の文字列だけ受け取れる。というような型の制限の上で、実装するべきだと考えている。
ただ、ちょろっと触っただけなので、どういう場面で、どう効果的なのか、今後も着目していきたい。

## Outro
RSCは非常におもしろい機能で、利用してみてとても学びがあった。何を補っておくべきかも整理できたので良かった。
ただ、今はNext.jsよりもRemixのほうが機能が絞られて扱いやすそうだなという印象を持っている。

使ってみたらそんなことはなく、隣の芝生が青く見えていただけということもあるので、次回はRemixを学んでみたい。
今後も、React周辺の事情にはキャッチアップしていきたい。

