---
title: "Next.js App RouterとRemixを比べた後に使い方を検討する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Next.js', 'App Router', 'Remix', 'Auth.js', 'remix-auth']
published: false
---

## Intro
Webアプリケーションを作ったのだが、その際にNext.jsをフレームワークとして利用した。
Remixと比較して選定した理由、使い方、足りない部分をどう補ったかなどを記載していく。

個別のライブラリについて詳しくは解説しないので、使い方や詳細な仕様については、別の記事を参照してほしい。

## Next.js App Router vs Remix

### React
筆者はReactで開発する体験が素晴らしいと思っているので、ブラウザUIは基本的にはReactで書きたい。
ほぼほぼ以下の記事で言及されている内容と同じことを思っている。
https://qiita.com/uhyo/items/ff243a5771077aaf4b5b

Reactのメタフレームワークは、メジャーどころではNext.jsとRemixだろう。
ここからは、Next.js App Router（以降App Router）とRemixを比較して行く。

### Web Standard
RemixとNext.jsを比較したときに、槍玉に上がりやすいのは、[Remix](https://remix.run/)のトップページに書かれている以下の文言だろう。
`Focused on web standards and modern web app UX, you’re simply going to build better websites`

特にNext.jsはfetchのパッチを当てていると言及される。
Next.jsのfetchは、以下のページで説明されているが、基本的な標準APIとしては使えつつも、別のオプションが使えるという仕様になっている。
https://nextjs.org/docs/app/api-reference/functions/fetch

つまり基本的な挙動は変更させていない。実のところそんなに変なことはやっていないなという印象を持った。
もちろんそれならば`nextFetch`みたいな関数を切り、それを使ってもらうように誘導できなかったのか？という疑問は当然残る。実害ではなく必要性を論点にすればネガティブな印象になる。

詳細に迫るのはこのあたりで止めておくが、Web標準じゃないからとNext.jsを嫌いすぎる必要はないと感じた。
ただ、Web標準を遵守しますというポリシーを掲げているフレームワークのほうが安心できる。将来にわたって混乱が少なそうなのはポリシーを掲げている側だろう。

### Path設計を不要にする技術
Next.jsでもRemixでも、API Routesを実装し、そのPathにリクエストを送れば、サーバサイドの機能を利用できる。
ただ、どうしてもPath設計(URL設計)が必要になり煩雑だ。

すでに開発を終えた時点での感想だが、最初はすべてPath設計をしてクライアントサイドと、サーバサイドをなるべく疎結合にしようと考えていた。
ただ多くの場合において、サーバサイドの機能は、特定の画面でのみ利用される傾向が強い。当然、疎結合な感じで、RESTfulな設計をした特定Pathの機能を、画面が使うという構図が成り立つ画面もあるだろう。だが、そうでないのであれば逆に画面に対するサーバ実装は専用のもの、つまり密結合な状態を目指してもいいはずだ。
そう考えた際に`画面 -> URL -> サーバ実装`というたどり方が面倒に感じ、取り払いたくなる。

この章では、Path設計をせずに、画面とサーバ実装を直接つなぐ機能について、言及していく。
API Routesを切ってしまえば実装できるものなので、ないと困る機能ではないが、便利機能として評価したい。

データ取得するGET処理と、データ更新するPOST処理でそれぞれ機能が違う。
- App Router
  GET: React Server Component(以降はRSC)
  POST: Server Actions
- Remix
  GET: loader関数
  POST: action関数

#### GET
まず言及しなくてはならないのはRSCだろう。
https://ja.react.dev/reference/rsc/server-components

RSCは、実のところPath設計を不要にする技術ではない。サーバサイドでしか実行できないという制約の元、サーバ実装をコンポーネントの中で実行できるようになっている機能だ。
ただ、最初にデータを取得して表示するだけの画面であれば、クライアントサイドで画面を動かしながら、データをfetchするということ自体が不要になる機能でもある。クライアントサイドで画面を動かすのではなく、単にサーバにHTMLを取りに行けば十分パフォーマンスが出るという機能だからだ。
そういった意味で、ここではRSCを挙げた。

今回、筆者は無限スクロールを実装したので、クライアントサイドで画面を動かしながらデータをfetchする必要があったが、そういったことにはRSCは使えない。Path設計をした上で、Client Componentを利用する必要がある。
Remixのloaderは、まさにクライアントサイドから、サーバの機能を透過的に使うための機能なので、無限スクロールのような画面に利用できる。
https://remix-docs-ja.techtalk.jp/route/loader

もう一点言及しておくべきこととして、画面に表示するデータとコンポーネントの関係がある。
特定のPathの画面に表示するデータのパターンが、多様な状態はあるだろうか。基本的に単一のパターンだろうし、したがってできる限り一つのEndpointからデータを取得したいはずだ。

これは、Remixのloaderでも抑えて置くべき考え方だ。Remixのloaderは、特定のPathにある画面においては、一つしか実装できない。その画面からPath設計なしで透過的にloadできる実装は一つだけなのだ。
これはRSCでも多様なパターンのデータを一つのコンポーネントで表現したくないはずなので、取得するデータは単一のパターンだろう。

#### POST
ここまではGET側に言及してきたが、POST側はまた違ったものがある。

Remixのaction関数は、loaderと同様の制約、つまり特定のPathに対して一つしか実装できない。
データ取得と違って、特定のデータに対する操作は、複数パターン考えられるので、これは少し不自由かもしれない。CRUDだって、loadする処理はRのみだが、actionする処理はCUDで3つも定義されている。

App RouterのServer Actionsはそういった制約がない。
Formと紐づけることもできるし、コード上は副作用を起こすただの関数のように呼び出すこともできる。特定の画面で複数のServer Actionsを実装、呼び出すことも可能だ。

#### Path設計を不要にする技術の総評
サーバサイドの機能に焦点を絞ると、上記のような細かい違いが存在する。
実質的にAPI Routesを切ってしまえばどうとでもなるので、結局のところ実現したい画面はどちらでも実装可能だ。ただ、使用感にはそれぞれ特徴があり、好みが分かれるかもしれない。

これは補足だが、Remixのloaderとactionは、同じ名前空間でexportするコンポーネントがあれば、そのコンポーネント用のGET処理、POST処理としてPath設計不要で利用できる。
逆にexportするコンポーネントがなければ、ただただGET、POST用のAPI Routesとしての実装になる。
一つの関数を使い方で機能が変わるのは、ちょっとややこしいかもしれないが、実際の実装としてはほぼ同じになりそうで、逆にわかりやすくもある。

### Static Export
静的出力は、Remixには当初機能がなかったが、SPAモードが追加され、一つのファイルであれば出力が可能になった。
Next.jsは、静的出力が可能だが、SPAモードのように一つのファイルにまとめるのではなく、path毎にhtmlを出力してくれる。

前提として静的出力するということは、サーバサイドで動くRuntimeを想定しないということだ。つまり動的pathには基本的に対応できない。
クライアントサイドで、画面遷移した場合には動的にpathを書き換えて表示できるが、初期表示時にhtmlファイルを取得することはできない。という意味だ。

Remixは動的pathに対応できないばかりか、そもそも1ファイルなのでルートにアクセスしないとhtmlは取得できない。

Next.jsの場合は、ファイルを展開してくれるので、実際に存在するページのpathにアクセスすれば、htmlが取得できる。
また、コンポーネントが動的pathを想定している場合、その動的pathのパターンをすべて網羅して静的出力する機能がある。
実装としては動的なのだが、すべてのパターンをビルド時に解決できるなら、この方法は最適だろう。
https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes#generating-static-params

ただ、ビルド時に解決できない動的pathはどうしようもないというのは、どちらも共通だ。

ビルド時に解決が難しい動的pathの場合どうするかというと、NginxなどのWebサーバであれば設定で存在しないルートに対する処理を定義して、対応することになる。
Github Pagesは404.htmlを配置して、そちらでハンドリングするようだ。
動的pathを使わないのであれば、Next.jsは何もケアがいらないが、動的pathを使うならどちらも同じである。

ちなみに今回は静的出力の機能は利用しない。
なので比較軸としては採用しないものだが、特徴的な機能なので言及しておく。

### AuthN/Z
AuthN/ZはAutheNtication（認証）、AuthoriZation（認可）のこと。
Auth.jsはApp RouterでもRemixでも利用できるが、App Routerの場合はデファクトスタンダードであり、他の選択肢は検討に上がらなかった。反面、Remixはremix-authがある。
Auth.jsとremix-authは特徴がだいぶ違うので、App RouterとRemixの比較要素としてあげたほうがよいと判断した。

Auth.jsを見て最初にギョッとしたのは、DBスキーマの強制だ。
以下はPrismaの例だが、指定されたスキーマを作らなくてはならない。
https://authjs.dev/getting-started/adapters/prisma

remix-authならばスキーマを指定しない。
反面、DBに認可の連携情報を記録するのは、自分で実装しなければならないようだった。
https://remix.run/resources/remix-auth

基本的にはスキーマ設計というのはアプリケーションエンジニアのコントロール領域だという考えから、remix-authのほうがポジティブに感じる。
ただ、IDプロバイダーからの情報や、セッション管理の情報などは、基本的なデータモデルがあるはずで、たしかに理想的なスキーマは指定の形に落ち着くかもしれない。また、スキーマを改修する際も、拡張情報ならば別テーブルを用意したほうが理想的になる場面も多そうだ。
つまりAuth.jsの指定するスキーマを改修することは基本的に想定しなくていい。

最初はギョッとしたが、Auth.jsでも実害はなさそうだという印象を持った。ただ、このような特徴的なライブラリは、できる限りアプリケーションコードからは隔離し、できる限りVersion Upや、ライブラリ変更の影響を小さくしたい。
基本的な機能としては、remix-authのほうが必要十分な印象だ。

### Next.js App Router vs Remixの結論
Remixのほうができることが少なくわかりやすい。Web標準というポリシーも掲げて安心できそうという印象を持った。
反面、Reactの最新機能である、RSCを使うのであればApp Router一択となる。今回は、好きなReactの最新機能にキャッチアップしたいという点が勝り、App Routerを採用した。散々いろいろ比較してきて総合してではなく、RSCという結論になってしまった。

もし仕事で採用するなら、もっと他の細かい機能要件を比較し、不足がないことを確認する必要がある。以下のRemix乗り換えの記事の内容までは、この記事では言及できていない。
https://user-first.ikyu.co.jp/entry/2023/12/15/093427

### 感想
この記事は実装が終わったあとに書いているので、結論で終わりではなく、実際どうだったかの感想もある。
当初の目的としてはRSCへの理解と、設計イメージをつけることだったが、それは学びとしては大きく、その点に関してはApp Routerを使ってみて良かった。

jsを書いているという感覚こそReactの良さだと考えているが、App Routerはサーバサイドで実行されるコードと、クライアントサイドで実行されるコードを分けてしまう。
分けてしまうと、変数の参照が切れてしまったりと、ケアが必要になる部分があるはずで、その部分が少し気持ち悪いと感じていた。

だが、クライアントで実行されるコンポーネントは`use client`ディレクティブで、Server Actionsは`use server`ディレクティブで明示的になるようになっているし、ファイル単位で分割もできる。
こういった機能によって、独立した名前空間ごとにサーバ/クライアントが分けやすいように設計されている。
独立した名前空間なので、変数の参照で特に意識することはなく、jsを書いているという感覚でいられるわけだ。

また、RSCからはClient Componentを呼べるが、その逆はできないという制約も、最初は混乱したが、書いていくと比較的自然にそういう風にかけることが多かった。RSCとClient Componentの使い分けも、それほど迷わずに考えられる。
すごいプログラマが設計したものというのは、いい出来になっているものだと感動した。

RSCには満足したし、どういうメリットがあるかわかったので、どういうアプリケーションに向いているか今後は考えられると思う。
次はRemixを使ってみたい。というより、どちらかというとNext.jsには満足したので、今後作るものにはRemixを使いたい。

## 不足の機能
App RouterとRemixを比較してきた。ここからは、App Routerを使う上で、補っておかなければならない機能について言及する。
また、App Router + Auth.jsを想定しているが、以下に記載していく内容は、Remix + remix-authでも同様のケアをすべきものだ。
ちなみに、versionは`next: 14.2.15`、`next-auth: 5.0.0-beta.22`で実装した。next-authのver5以降がAuth.js。

Next.jsはWebフロントエンドのためのフレームワークであり、サーバサイドの機能をあまり持たない。
API Routesでリクエストを受け付けることができるが、受け付けるだけでその後のことは何もしてくれない。

基本的にはリレーショナルデータベースを利用するので、PrismaなりのDBアクセスライブラリは利用するとして、そこに至るまでのアプリケーションの記載を支援する仕組みが必要になる。
以下のようなものがあるだろう。
- Session管理
- リクエストハンドリング
- DI
- トランザクション管理
- ロギング

上記にたいしてどう用意すべきか検討していく。

### Session管理
Session管理は、Auth.jsに任せることになる。
Auth.jsはサーバサイドでsession情報を取り出す役目だが、クライアントサイドでも透過的にサーバからSession情報を取得できるAPIが用意されている。

ただし、基本的にはクライアントサイドでAuth.jsのAPIは用いいないことにした。
すでに言及したが、Auth.jsはDBスキーマまで口出してくるライブラリで、なるべく変更の影響を小さくしたい。
これはサーバサイドでも同様で、Sessionから取り出すのはユーザの特定に必要なuser.idのみで、あとの処理では、Auth.jsで指定されたスキーマは参照、更新を一切しないこととした。

クライアントサイドでSession情報を得る際には、App Routerのlayout.tsxでsession情報を取得し、クライアントサイドでReact Contextに登録して子コンポーネントから利用する仕掛けにした。
以下がlayout.tsxの実装例だ。

```tsx
import type { Metadata } from "next";
import "./globals.css";
import { SessionProvider } from "./SessionProvider";
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

### リクエストハンドリング
ここでリクエストハンドリングとは、受け取ったHTTP Requestを解釈、バリデーションし、処理に引き渡すこと。
また処理から受け取ったデータを整形してHTTP Responseを返すこととして、定義する。

よくあるパターンだが、Zodを使って入力値のバリデーションを行い、型の制約を表現した。
また、例外発生時にはHTTP Status Codeをコントロールできる実装もしている。

上記の機能を果たすヘルパー関数を用意した。
リクエストハンドリングではなくSession管理だが、Auth.jsの利用もここに実装している。
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
本来は必須なのか不要なのかちゃんと理解する必要があるが、上記のようなリクエストハンドリングを行うヘルパーはServer Actionsのためのものもいくつか用意した。

### DI
DIは依存性の解決であり、特にIOを生じさせるような処理をメインのロジックから切り離して、テストしやすくプラガブルにするために行う。
ただ、DIはclassベースのプログラミングパラダイムから生まれてきた単語で、今回はclassはかなり限定的に使っているので、厳密な意味でのDIは実装しない。

IOの処理を分離し、テストをしやすくするというのは、classを用いなくても重要なテクニックになるので、それを実現する機能は実装している。
解説の分量が多いので、別パートで解説する。

### トランザクション管理
トランザクションの処理は、フレームワーク的な機能としては用意しなかった。
コンテナ環境へのデプロイであり、オブジェクトストレージへのアクセス、外部サービスへのアクセスなど、DB以外のIOも多様にある状況においては、一辺倒なトランザクション管理は適切でないと考えたため。
代わりに、アプリケーションコードの中でトランザクションをコントロールできる関数を用意した。

そうではなく、特定のコードをmiddlewareのような機能で囲い、そのmiddleware上でトランザクションが管理されるような形でも実装できる。
が、そうなっていないので、フレームワーク的な機能ではないということだ。
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

// 注入する依存の実態。直接記載するので依存があるように見えるが、bindはされていない
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

DIコンテナは、大きく設定ファイルを記載するタイプと、対象classにアノテーションで依存先を表現するタイプがある。
今回はアノテーションをつける形に近く、上記のコードで言えば`getUser`関数がdbに依存しているため、依存先のdbを今回のContextの仕組みを用いて、切替可能にしている。
これは実行されるコードの設定が別ファイルだと、両方参照しないと全容が掴みづらいと考えたためだ。

上記はテストのためと述べた通り、ファイル名前空間上は依存が明確にある。
ただ、テスト自体は`db`変数をモックすればよいので副作用を除外して書ける。

### 再帰的な解決
実装が重くなるという点もそうだが、必要性からも再帰的な解決ができるようには実装しなかった。
以下のコード例では、serviceとdbをbindしているが、`service.getUser`に`db.getUser`を渡している。
基本的なDIにおいては、serviceの中にdbをbindした状態で、依存が解決されているはずだが、それはしていない。

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
  // db.getUserはcallbackとして実行時に与えているが、DIなら事前にbindする実装になるはず
  return service.getUser(db.getUser, userId);
};

setContext(getUser, getUserContext);
```

上記の名前空間が、直接`db.getUser`を使っていることがわかるので、何に依存しているかが明確だ。
これを再帰的に解決しようと、serviceの中にbindしてしまうと、serviceの中で、どのようなIOが行われているか、コードを追う必要が出てくる。
この名前空間上で、何が行われているのか表現できたほうが、筆者としてはわかりやすく感じたため、このような実装になっている。

また、上記で依存を解決するものについては、特別な準備が不要だ。
上記の`service`という変数にbindしたオブジェクトは、その場で即時的に作っている。`userService#getUser`は、ただの関数なのだ。
DIにおいては、classを用意し、constructorでインスタンス変数を用意するなどのDI用の実装がいるが、ここでは何でもDIできる。

### 実装してみて
Contextの実装においては、厳密には違うかもしれないが、初めていわゆる型レベルプログラミングというものを実装した。
入力の型から、出力の型を導出する技術で、関数のロジックの表現ができてしまう。
今回はそういったことは実装していないが、四則演算や、任意の文字列の整形なども、型として表現できるようだ。
かなり強力な機能なので、何らかの制限が必要だろうなと感じた。

制限としては、任意の数字や文字列ではなく、たとえば1-100までの整数や、特定の文字列だけ受け取れる。というような型の制限の上で、実装するべきだと考えている。
ただ、ちょろっと触っただけなので、どういう場面で、どう効果的なのか、今後も着目していきたい。

## Outro
App RouterとRemixを比較して、様々な対立軸で特徴があることを示した。
また、それらのメタフレームワークを利用する際に、補うべき機能の実装について示した。
DI（厳密には違う）については、詳しく解説した。

上記は、筆者自身が検討したことなので、的はずれなこともあるかもしれないが、筆者としては学びになってよかった。

また、Remixについて今回比較対象としたが、最終的に採用しておらず、細かい機能の認識が間違っているかもしれない。
気になった方にはご指摘いただきたい。

