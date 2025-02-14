---
title: "Vite-PWAでQueryパラメータをつけるとトップページに遷移して困る問題について"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Vite', 'Vite-PWA', 'Workbox']
published: true
---

# Intro
最近ニコニコ漫画でなろう系の漫画ばかり読んでいるのでタイトルが長くなってしまった。
タイトルの通りの事象が発生し、解決した。ただ、技術的な詳細については迫れておらず、推測にとどまる。
対象の事例で日本語の記事が見つからなかったので、場当たり的に解決案を記述しておくだけでもいいかなということで記事を起こした。

https://vite-pwa-org.netlify.app/

## 現象
Github PagesにdeployしているVite + Reactな静的サイトに、Vite-PWAというVite pluginを導入したら、意図しない挙動が発生した。
Github Pagesは`<account>.github.io`ではなく、`<account>.github.io/<project>`がurlという環境だが、これは関係ないような気がしている。

事象は具体的には、URLにQuery Parameterをつけた場合に、トップページのHTMLが取得される。
URLはQuery Parameterをつけた通りで変わっていないのだが、HTMLだけが当該ページのものではなく、トップページのHTMLが取得されていた。
これはChome dev-toolで確認している。

また、Query ParameterをつけたURLへの遷移はhistory APIを利用したアプリケーション挙動のものではなく、Location#assignによるHTMLの取得による遷移で実現している。

## 対応
Vite-PWAは内部的にWorkboxというライブラリを利用している。
そのWorkboxの設定に[ignore_url_parameters](https://developer.chrome.com/docs/workbox/modules/workbox-precaching?hl=ja#ignore_url_parameters)というオプションがあり、Googleのサポートページにも記載方法が書いてある。

このオプションを有効にすべく、`vite.config.ts`を以下のように記述した。

```diff ts
  import { resolve } from "path";
  import { defineConfig } from "vite";
  import react from "@vitejs/plugin-react";
  import tsconfigPaths from "vite-tsconfig-paths";
  import { VitePWA } from "vite-plugin-pwa";

  // VITE_URL_PREFIXはproject pathが入る。本番と開発環境でurlが変わるためのもの
  const path = process.env.VITE_URL_PREFIX ? "/" + process.env.VITE_URL_PREFIX + "/" : "/";

  // https://vitejs.dev/config/
  export default defineConfig({
    root: "src/pages",
    plugins: [
      react(),
      tsconfigPaths(),
      VitePWA({
        registerType: "autoUpdate",
+       workbox: {
+         ignoreURLParametersMatching: [/./],
+       },
        manifest: {
          scope: path,
          // other pwa manifest setting...
        },
      }),
    ],
    publicDir: resolve(__dirname, "public"),
    build: {
      // build setting...
    },
    base: path,
  });
```

現象と対応だけで、原因とメカニズムがわからないので、原因の可能性を残すべく、なるべくいろんな設定を記載しておくが、PWAのmanifestとviteのbuild設定は流石に影響がないと思うので、省いて記載している。
これで、前述した現象が起きなくなった。

これは、Vite-PWA上の以下のissueを参考にしている。
https://github.com/vite-pwa/vite-plugin-pwa/issues/653

## 現象についての推測
今回の事象について、様々な原因が考えられた。
Github Actionsでdeploy時におかしくなる。build時にhtmlが差し替わっている。Github Pages、要はjekyllのセッティングの問題。

だが、URLは正しいのに、HTMLが違ったものが表示されるというのは、404発生時のフォールバックの挙動に思えた。
また、Github Pagesは特に変なことはしておらず、jekyllのセッティングであべこべなHTMLが返されるというのも考えづらい。

とすると、PWA時にcache firstでHTMLを探した際に、Query ParameterもCache Keyになっているとしたら、対象のHTMLが見つからず、トップページのHTMLを取得したと推測した。
だが、厳密にはcache firstな設定はしていないはず（これもVite-PWAに詳しくないので、ちゃんと意図通りかわからない）。
そうすると単にGithub Pagesから取得した正しいHTMLを使いそうなものでもある。

とりあえず、Query Paramerに関連しそうなオプションを、Vite-PWA、Workboxにまたがって調べていたら、当該の設定が見つかり、適用したら治った。というところ。
[ignore_url_parameters](https://developer.chrome.com/docs/workbox/modules/workbox-precaching?hl=ja#ignore_url_parameters)の説明にある通り、Query StringまでをURL matchingの範囲とするのは仕様なようで、オンライン時にサーバではなくcacheだけで判断しているとしても、URL matchingの仕様から考えてもbugだと断定するにも技術的な知見が足りなすぎる。
なので筆者は特に報告するつもりはなく、開発者が気づいて設定を入れるというのが、当面の対応になるのではないかと考える。

## Outro
service workerは数年前に、workboxを使わずに自前で実装を試みてうまくいかず失敗したが、当時もっと勉強していれば、技術的詳細に迫れたかもしれない。
それにしても、Vite-PWAを導入していれば、必ずぶち当たりそうなものだが、皆さんどうしているのか。根本的に筆者の技術選択や使い方が間違っている可能性も捨てきれない。

力不足が悩ましいが、同じ事例にあたった人（将来の自分を含め）のために記事を残しておく。

