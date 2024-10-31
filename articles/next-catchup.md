---
title: "Next.js App RouterとRemixを比べた後に使い方を検討する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Next.js', 'Auth.js']
published: false
---

## Intro
Webアプリケーションを作ったのだが、その際にNext.jsをフレームワークとして利用した。
Remixと比較して選定した理由、使い方、足りない部分をどう補ったかなどを記載していく。

## Next.js App Router vs Remix

### React
筆者はReactで開発する体験が素晴らしいと思っているので、ブラウザUIは基本的にはReactで書きたい。
ほぼほぼ以下の記事で言及されている内容と同じことを思っている。
https://qiita.com/uhyo/items/ff243a5771077aaf4b5b

Reactを選んだ時点で、メタフレームワークとしての選択肢は限られてくる。
他にもあるだろうが、メジャーどころではNext.jsとRemixだろう。
ここからは、Next.js App Router（以降App Router）とRemixを比較して行く。

### Web Standard
RemixとNext.jsを比較したときに、槍玉に上がりやすいのは、[Remix](https://remix.run/)のトップページに書かれている以下の文言だろう。
`Focused on web standards and modern web app UX, you’re simply going to build better websites`

特にNext.jsはfetchのパッチを当てていると言及される。
Web標準と章立てしたが、基本的にはfetch APIに絞って展開していく。

ただ、いろんなフレームワークを見るとfetchにはパッチを当てているケースが多い。以下ははremixのコードだ。
https://github.com/remix-run/remix/blob/main/packages/remix-node/globals.ts#L42
```ts
global.fetch = undiciFetch;
```

詳しく追っていないので解説することは避けるけども、Nodeのversionによってはサーバサイドでfetchが使えないためにパッチを当てているというのはなんとなく予想がつく。
以下の記事は参考になるかもしれない。
https://yosuke-furukawa.hatenablog.com/entry/2022/12/05/103008

クライアントサイドのfetchへのパッチが問題だ！といえばそうだが、フレームワークというのは基本的な機能を使うために何かしら標準APIを補助することが必要だ。という視点もあるのではないかと感じた。
つまり、パッチを当てるということ自体は批判すべきことではない。パッチの当て方が問題だし、不必要なパッチを当てないようにしないと行けない。

Next.jsのパッチは、以下のページで説明されているが、基本的な標準APIとしては使えつつも、別のオプションが使えるという仕様になっている。
https://nextjs.org/docs/app/api-reference/functions/fetch

つまり、通常通りのfetch関数は使えつつも、少し拡張しているということになる。
基本的な挙動は変更させていないのだから、確かにそんなに変なことはやっていないなという印象を持った。
もちろんそれならば`nextFetch`みたいな関数を切り、それを使ってもらうように誘導できなかったのか？という疑問は当然残る。
実害ではなく必要性を論点にすればネガティブな印象になる。

詳細に迫るのはこのあたりで止めておくが、筆者の意見としてはそういう視点もあるのだから、Web標準じゃないからとNext.jsを嫌いすぎる必要はないだろうという意見だ。
ただ、筆者としてもWeb標準を遵守しますというポリシーを掲げているフレームワークのほうが安心できる。将来にわたって混乱が少なそうなのはポリシーを掲げている側だろう。

### Server Side Function
Next.jsでもRemixでも、API Routeを実装し、そのpathにリクエストを送れば、サーバサイドの機能を実現できる。
ただ、どうしてもpath設計が必要になり煩雑だ。

すでに開発を終えた時点での感想だが、最初はすべてpath設計をしてクライアントサイドと、サーバサイドをなるべく疎結合にしようと考えていた。
ただ多くの場合において、サーバサイドの機能は、特定の画面でのみ利用される傾向が強い。
当然、疎結合な感じで、RESTfulな設計をした特定pathの機能を、画面が使うという構図が成り立つ画面もあるだろう。
だが、そうでないのであれば逆に画面に対するサーバ実装は専用のもの、つまり密結合な状態を目指してもいいはずだ。
そう考えた際に`画面 -> URL -> サーバ実装`というたどり方が面倒に感じ、取り払いたくなる。

この章では、path設計をせずに、画面とサーバ実装を直接つなぐ機能について、言及していく。
App RouterでもRemixでも、それに該当する機能は存在するが、データ取得するGET処理と、データ更新するPOST処理でそれぞれ機能が違う。

App Routerにおいては、以下が該当する
  GET: React Server Component(以降はRSC)
  POST: Server Actions

Remixにおいては以下だ
  GET: loader関数
  POST: action関数

App RouterでもRemixでも、path設計を取り除くというのはできるが、どちらもそれぞれ制約や癖があるため、解説していく。

#### GET
まず言及しなくてはならないのはRSCだろう。
Next.js Page Routerでは、React Componentと同じ名前空間にgetServerSidePropsという関数で実装されていたものを、React Componentに取り込んだものと捉えられる。
したがって、サーバサイドの機能としてだけではなく、コンポーネントのあり方としても評価しなくてはならない。
ただ、サーバサイドの機能は含んでいるので、その点に着目する。

RSCは、サーバサイドでしか実行できない。つまりクライアントで実行し、透過的にサーバにリクエストを送るという機能はない。
今回の実装では、無限スクロールがあり、スクロール毎にリクエストしてデータ取得する必要があるが、RSCでは実現できないので、path設計はどうしても必要になる。

Remixのloaderはそういった制約はなく、サーバサイドでSSRされるときにも呼び出され、クライアントサイドでもloaderを呼び出すことができる。
ただ、loaderの制約は、特定のpathに対して一つしか実装できないという点にある。
特定の画面のpathにコンポーネントが存在する場合、loaderも一つしかできない。
コンポーネントが、多様なデータを表現したい場合は、別のpath設計が必要になる。
ただ、コンポーネントが多様なデータを表現するのではなく、データの表現としてコンポーネントがあるほうが自然なので、これはむしろ当たり前の制約と言えそうだ。
もし、そういった様々なデータを組み合わせたければ、Nested Routeという機能で、コンポーネント（つまりloaderも）を組み合わせることもできる。

RSCもコンポーネントの中で取得するデータは固定で、その構図は同一だ。
だが、Nested Routeのような機能でなくても、単純にコンポーネントを組み合わせるというやり方の中で、複数のデータ取得ができる点が優れている。

#### POST
ここまではGET側に言及してきたが、POST側はまた違ったものがある。

App RouterのServer Actionsは実のところ、あまり制約がない。
Formと紐づけることもできるし、コード上は副作用を起こすただの関数のように呼び出すこともできる。
特定の画面で複数のServer Actionsを実装、呼び出すことも可能だ。

Remixのaction関数は、loaderと同様の制約、つまり特定のpathに対して一つしか実装できないというものがある。
データ取得と違って、特定のデータに対する操作は、複数パターン考えられるので、これは強い制約と言えそうだ。

もう一点、Remixのloaderとactionは、同じ名前空間でexportするコンポーネントがあれば、そのコンポーネント用のGET処理、POST処理としてpath設計不要で利用できる。
逆にexportするコンポーネントがなければ、ただただGET、POST用のAPI Routeとしての実装になる。
名前が同じだが、機能は違うという意味で区別が付きづらくなるかもしれないが、実際の実装としてはほぼ等価になりそうなものなので、逆にわかりやすくもある。

#### Server Side Function 総評
サーバサイドの機能に焦点を絞ると、上記のような細かい違いが存在する。
実質的にAPI Routeを切ってしまえばどうとでもなるので、結局のところ実現したい画面はどちらでも実装可能だ。
ただ、使用感にはそれぞれ特徴があり、好みが分かれるかもしれない。

Remixを利用せず、App Routerで実装した感想としては、App Router側の機能がややこしいし、RSCはコンポーネントと機能自体が密なので分けて考えづらいと感じる。
RSC自体は、別の観点でも評価するべきだが、サーバサイドの機能の実装という意味では、Remixのほうが制約や癖がわかりやすいという印象だ。
App Routerの流儀に沿った実装を習うのか、あるいはプログラマが考えてRemixのAPIを利用するのかは、考え方によるのかもしれない。

### Static Export
静的出力は、Remixには当初機能がなかったが、SPAモードが追加され、一つのファイルであれば出力が可能になった。
Next.jsは、静的出力が可能だが、SPAモードのように一つのファイルにまとめるのではなく、path毎にhtmlを出力してくれる。

一つのファイルサイズとしてはRemixのほうが重たくなるはずだが、取り回しやしやすそうだ。
反面、NginxなどのWebサーバから見て自然なのはNext.jsのほうだろう。
そうった視点もあるが、ここでは動的pathについてより詳しく言及したい。

前提として静的出力するということは、サーバサイドで動くRuntimeを想定しないということだ。つまり動的pathには基本的に対応できない。
クライアントサイドで、画面遷移した場合には動的にpathを書き換えて表示できるが、初期表示時にhtmlファイルを取得することはできない。という意味だ。

Next.jsの場合は、コンポーネントが動的pathを想定している場合、その動的pathのパターンをすべて網羅して静的出力する機能がある。
実装としては動的なのだが、すべてのパターンをビルド時に解決できるなら、この方法は最適だろう。
https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes#generating-static-params

Remixはそういう機能はなく、割り切った実装だ。
Next.jsの場合でも、ビルド時に解決が難しい動的pathの場合は、当然対応できない。
動的なpathの場合でも、どこかにそのstateが存在し、多くはサーバサイドにあるのだから、Node以外のRuntimeであっても想定し使うべきだ。
そういう意味では、

そういう意味ではRemixのほうが、ある意味わかりやすいかもしれない。

ビルド時に解決が難しい動的pathの場合どうするかというと、404.htmlを使ってそちらでハンドリングする形となる。
これは実質どちらも同様だ。

ちなみに今回は静的出力の機能は利用しない。
なので比較軸としては採用しないものだが、特徴的な機能なので言及しておく。

### AuthN/Z
AuthN/ZはAutheNtication（認証）、AuthoriZation（認可）のこと。
Auth.jsはApp RouterでもRemixでも利用できるが、App Routerの場合はデファクトスタンダードであり、他の選択肢は検討に上がらなかった。
反面、Remixはremix-authがある。
Auth.jsとremix-authは特徴がだいぶ違うので、App RouterとRemixの比較要素としてあげたほうがよいと判断した。

Auth.jsを見て最初にギョッとしたのは、DBスキーマの強制だ。
以下はPrismaの例だが、指定されたスキーマを作らなくてはならない。
https://authjs.dev/getting-started/adapters/prisma

remix-authならばスキーマを指定しない。
反面、DBに認可の連携情報を記録するのは、自分で実装しなければならないようだった。
https://remix.run/resources/remix-auth

基本的にはスキーマ設計というのはアプリケーションエンジニアのコントロール領域だという考えから、remix-authのほうがポジティブに感じる。
ただ、IDプロバイダーからの情報や、セッション管理の情報などは、基本的な考え方があるはずで、たしかに理想的なスキーマは指定の形に落ち着くかもしれない。
また、スキーマを改修する際も、拡張情報ならば別テーブルを用意したほうが理想的になる場面も多そうだ。
つまりAuth.jsの指定するスキーマを改修することは想定しなくていい。

最初はギョッとしたが、Auth.jsでも実害はなさそうだという印象を持った。
ただ、このような特徴的なライブラリは、できる限りアプリケーションコードからは隔離し、できる限りVersion Upや、ライブラリ変更の影響を小さくしたい。
基本的な機能としては、remix-authのほうが必要十分な印象だ。

### Next.js App Router vs Remixの結論
Remixのほうができることが少なく、プログラマが自分で考えて実装しやすい。Web標準というポリシーも掲げ、わかりやすい作りになっていそうという印象にある。
反面、Reactの最新機能である、RSCを使うのであればApp Router一択となる。
今回は、好きなReactの最新機能にキャッチアップしたいという点が勝り、App Routerを採用した。
散々いろいろ比較してきて総合してではなく、RSCという結論になってしまった。

もし仕事で採用するなら、もっと他の細かい機能要件を比較し、不足がないことを確認する必要がある。
以下のRemix乗り換えの記事の内容までは、この記事では言及できていない。
https://user-first.ikyu.co.jp/entry/2023/12/15/093427

ただ、App RouterもRemixも機能要件を満たしているのであれば、仕事なら安定性やカスタマイズ性を重視し、Remixを採用するだろう。

## 不足の機能
Next.jsはWebフロントエンドのためのフレームワークであり、サーバサイドの機能をあまり持たない。
API Routeでリクエストを受け付けることができるが、受け付けるだけでその後のことは何もしてくれない。

基本的にはリレーショナルデータベースを利用するので、PrismaなりのDBアクセスライブラリは利用するとして、そこに至るまでのアプリケーションの記載を支援する仕組みが必要になる。
以下のようなものがあるだろう。
- Session管理
- リクエストハンドリング
- DI
- トランザクション管理
- ロギング

上記にたいしてどう用意すべきか検討する。

### Session管理
Session管理は、Auth.jsに任せることになる。
Auth.jsはサーバサイドでsession情報を取り出す役目だが、クライアントサイドでも透過的にサーバからSession情報を取得できる。

ただし、基本的にはクライアントサイドでAuth.jsのAPIは用いいないことにした。
以下の記事でも言及したが、Auth.jsはDBスキーマまで口出してくるライブラリで、なるべく変更の影響を小さくしたい。

これはサーバサイドでも同様で、Sessionから取り出すのはユーザの特定に必要なuser.idのみで、あとの処理では、Auth.jsで指定されたスキーマは参照、更新を一切しないこととした。

クライアントサイドでSession情報を得る際には、App Routerのlayout.tsxでsession情報を取得し、クライアントサイドでハイドレーションされたらContextに登録して子コンポーネントから利用する。
サーバサイドでも、userテーブルの他に、特定のユーザを表現するテーブルを別途作り、そこにだけuser_idという紐づけのカラムを用意することで、Auth.jsのスキーマに触れないようにした。

ちなみに実装としては、後述するリクエストハンドリングを行う部分に実装を差し込む。

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
  callback: (userId: String, path: z.infer<P> | null) => Promise<R>,
): AppRouteHandlerFn {

  return auth(async (req, { params }) => {

    const pathArgs = pathSchema ? parse(pathSchema, params) : null; // path parameterをzodにかける

    try {
      const userId = getUserId(req.auth); // SessionからuserIdを取得

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

export const GET = getRouteHandler(pathSchema, (userId, p) => ({ path: val }));
```

上記はpath parameterのparseの実装だが、実際にはFormData, Request Body, Query Parameterなどのデータも処理できなくてはならない。
基本的にFormData, Request Body, Query Parameterは同時に利用される可能性は少なく、個別に実装すればよい。
以下の3パターンのためのヘルパー関数を用意した。
- Path Parameter + FormData
- Path Parameter + Request Body
- Path Parameter + Query Parameter

また、筆者のマインドマップとしては、Server Actionは透過的にサーバサイドの機能を呼び出せる機能であり、サーバサイドにあるという意識が強い。
したがって、tsの型チェックがあるからと、リクエストを受け取ったあとのスキーマバリデーションを行わないのが非常に気持ち悪く感じた。
よって、上記のようなリクエストハンドリングを行うヘルパーはServer Actionsのためのものもいくつか用意した。

### DI
DIは依存性の解決であり、特にIOを生じさせるような処理をメインのロジックから切り離して、テストしやすくプラガブルにするために行う。
ただ、DIはclassベースのプログラミングパラダイムから生まれてきた単語で、今回はclassはかなり限定的に使っているので、厳密な意味でのDIは存在しない。

IOの処理を分離し、テストをしやすくするというのは、classを用いなくても重要なテクニックになるので、それを実現する機能は実装している。
解説の分量が多いので、別パートで解説する。

### トランザクション管理
トランザクションの処理は、フレームワーク的な機能としては用意しなかった。
コンテナ環境へのデプロイであり、オブジェクトストレージへのアクセス、外部サービスへのアクセスなど、DB以外のIOも多様にある状況においては、一辺倒なトランザクション管理は適切でないと考えたため。

特定のコードをmiddlewareのような機能で囲い、そのmiddleware上でトランザクションが管理されるような形では、そのコードの中でオブジェクトストレージにアクセスした際に発生するエラーとDBのエラーを分けて管理しづらくなる。
そのように考えたため、アプリケーションのコードからトランザクションを開ける仕組みとした。これはフレームワーク側の機能ではないので、DBアクセスへの解説記事にて後日言及する。

ちなみに以下のようなイメージで使う。

```ts
const handle = (db) => (userId: string, userName: string) => {
  const user = db.getUser(userId);

  // アプリケーションコードの中で、プログラマが意識的にトランザクションを開く
  db.transact(trx => {
     trx.updateUser(userId, userName);
  });

  return db.getUser(userId);
};
```

### ロギング
ロギングは今回は実装していない。
本当は考えなくてはならないのであげたが、今回は検討していないので解説しない。

## DIモジュールについて
テストが目的で、最小限のコード

## 考えたかた
難解な機能は避けつつ、メタなコードはアプリケーションに必要。選んで機能が重複しないように
またライブラリ寄りならしかたないがアプリはより注意深く
tsの型レベルプログラミングも、自由にするのではなく、1-100までの整数や、stringのunionのような、プログラマが把握できる制約まで狭めた範囲ならよい。


## DIとは
classの上に乗せた機能だが、classの考え方を基本にするなら、自然な機能

- DI
  - classベースの考え方
- DIコンテナ
- 関数において先にbindすること

## DIの目的
- テスト
- 実装の切り替え
  - 将来的
  - 現在の実装

## ライブラリの扱い
wrapしましょう
- 付け替えを内部でできる
- ライブラリ利用のスコープを限定
- ライブラリの違いを吸収
  - 引数
  - 戻り値
  - 例外

## 実装の付け替え
実装として、アプリケーションの中で付け替えるのであれば、DIコンテナではないはず。
アプリケーションの機能として実装するのだから、DIはしてもコンテナで管理の必要はないのでは

## テスト
必要。ただし、主たる処理があり、サブな位置づけで

## 関数での実装
コードと利用例

## 特徴
上記コードの特徴とpros/cons

## つらみ
多重でDIする際には、生成知識がいる。
いずれにしろ、生成知識というやつは、アプリケーションコード上に実装する。
量が多ければ難しくなる。

けれど、多重にするにしても、2レイヤ程度ではないか
`ユースケース -> サービス -> リポジトリ`

ただし、この構造では、ユースケースがリポジトリの何にアクセスしているか、知らない。
外界に作用を引き起こす処理がなんなのか、ユースケース上で把握できるのは、よいこと。

関数なら、callbackを渡すことで、比較的かんたんにDIっぽい動きができるので、コード量もすくなくて済みやすい。

### 難しい機能
mapped typeをどこまで使うべきか。型のバリエーションに制限があるなら。任意の文字列ではなく、固定の文字列の羅列までならいいのでは

## outro

※Remixについては、最終的に採用しておらず、上記の認識がずれているかもしれない。気になった方にはご指摘いただきたい。

