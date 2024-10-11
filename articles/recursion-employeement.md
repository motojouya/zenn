---
title: "再帰ック☆採用試験"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "PostgreSQL", "wget", "PHP"]
published: true
published_at: 2018-12-06
---

OPENLOGIアドベントカレンダー2018の6日目です。
今年の6月から働いています。杉山です。
今年は技術的なTRYがなく正直ネタがないのですが、OPENLOGIに入るときのコードテストが割とシンドかったので、まとめようかなぁと思います。
ちなみに自分のコードテストに時間がかかり過ぎてしまったせいで、私以降はコードテストは行ってません。
ただ、技術は大切にしている会社だと思うので、いつか復活するかもしれません(採用かかわってないのでわからん)。

## テストの内容

テストの内容は以下でした。

```
Wikipediaの任意のページにおいて、そのページの目次の前にある、テキスト中のリンクテキストのリストを取得。
そのリンクから、同じことを再帰的に行い、任意のページからどの単語が関連しているかを、表現できるようにする。
```

今思うと、自分には結構ヘビーだったなぁと思いますが、だいたい3週間くらいで作りました。
前職の仕事もしつつ、phpは触ったことない状態だったので、ちょっと大変でしたね。
[自分の回答](https://github.com/motojouya/exam-wikipedia-tree-sugiyama/tree/web-interface)

その課題を解く上で、木構造や再帰を使う必要があり、色々なところに再帰のエッセンスがあったので、それについてまとめてみようかなぁと思います。
課題の回答の中でやっている再帰は、sqlとvueとphpでの再帰ですが、それだけだとつまらないので、他のものも調べてみます。

## JavaScript
コードテストでは書かなかったのですが、まずはJavaScriptからです。自分はJavaScriptが大好きなので。
弊社では、フロントエンドフレームワークでSPAで作っているので、JavaScriptはゴリゴリ使います。

```JavaScript
const fruits = {
  "name": "フルーツ",
  "nodes": [
    {
      "name": "みかん",
      "nodes": [
        { "name": "みかん", "nodes": [], },
        { "name": "レモン", "nodes": [], },
        { "name": "伊予柑", "nodes": [], },
      ]
    },
    {
      "name": "ぶどう",
      "nodes": [
        { "name": "ぶどう", "nodes": [], },
        { "name": "巨峰", "nodes": [], },
        { "name": "マスカット", "nodes": [], },
      ]
    },
    {
      "name": "りんご",
      "nodes": [],
    },
  ]
};

const searchTree = (tree, parents) => {
  const myself = (parents ? parents + ' > ' : '') + tree.name;
  console.log(myself);
  for (const node of tree.nodes) {
    searchTree(node, myself); // ここで自分を呼び出す再帰
  }
};
searchTree(fruits);
```

```結果
フルーツ
フルーツ > みかん
フルーツ > みかん > みかん
フルーツ > みかん > レモン
フルーツ > みかん > 伊予柑
フルーツ > ぶどう
フルーツ > ぶどう > ぶどう
フルーツ > ぶどう > 巨峰
フルーツ > ぶどう > マスカット
フルーツ > りんご
```

これは簡単ですね。幅方向はループ、深さの時に再帰です。
では幅方向も再帰にできるでしょうか。

```JavaScript
const searchBreadth = (nodes, parents) => {
  if (nodes.length === 0) {
    return;
  }
  const node = nodes.shift();
  searchDepth(node, parents); // searchBreadthとsearchDepthを相互に呼び出す再帰
  searchBreadth(nodes, parents); // ここで自分を呼び出す再帰
};

const searchDepth = (tree, parents) => {
  const myself = (parents ? parents + ' > ' : '') + tree.name;
  console.log(myself);
  searchBreadth(tree.nodes, myself);
};

searchDepth(fruits);
searchBreadth(fruits.nodes);
```

簡単に書くために`Array.prototype.shift`を使ってしまっているので、`fruits`の中身が変化してしまう破壊的なコードですが、できました。
また再帰とは別の話ですが、半ば強制的に関数をわける羽目になったので、`searchBreadth(fruits.nodes)`のような呼び出し方もできるようになりました。
書き方にもいろいろできそうです。

## lodash
JavaScriptを使う上では、lodashは非常によく使うライブラリです。
もちろん弊社でも使っています。

flattenという関数がありますが、多重配列をフラットにしてくれます。
https://github.com/lodash/lodash/blob/4.17.11/lodash.js#L7327

内部ではbaseFlattenという関数で再帰をしています。
https://github.com/lodash/lodash/blob/4.17.11/lodash.js#L2939

```JavaScript
function baseFlatten(array, depth, predicate, isStrict, result) {
  var index = -1,
      length = array.length;

  predicate || (predicate = isFlattenable);
  result || (result = []);

  while (++index < length) {
    var value = array[index];
    if (depth > 0 && predicate(value)) {
      if (depth > 1) {
        // Recursively flatten arrays (susceptible to call stack limits).
        baseFlatten(value, depth - 1, predicate, isStrict, result);
      } else {
        arrayPush(result, value);
      }
    } else if (!isStrict) {
      result[result.length] = value;
    }
  }
  return result;
}
```

先程のJavaScriptの例と同様、幅探索はwhileループ、深さは再帰を使っています。
単純に考えれば、幅が配列で表現されており、配列を処理するための構文は揃っているので、幅はループで処理するほうがいいのでしょう。

## php
次はPHPです。
弊社のサーバサイドは基本的にPHPでできています。
自分はphp歴が半年なので、たぶん穴だらけなんですが、そこはあしからず。

```php
<?php

$fruits = [
  "name" => "フルーツ",
  "nodes" => [
    [
      "name" => "みかん",
      "nodes" => [
        [ "name" => "みかん", "nodes" => [], ],
        [ "name" => "レモン", "nodes" => [], ],
        [ "name" => "伊予柑", "nodes" => [], ],
      ]
    ],
    [
      "name" => "ぶどう",
      "nodes" => [
        [ "name" => "ぶどう", "nodes" => [], ],
        [ "name" => "巨峰", "nodes" => [], ],
        [ "name" => "マスカット", "nodes" => [], ],
      ]
    ],
    [
      "name" => "りんご",
      "nodes" => [],
    ],
  ]
];


function searchTree($tree, $parents = null)
{
  $myself = ($parents ? $parents . ' > ' : '') . $tree['name'];
  echo $myself."\n";
  foreach ($tree['nodes'] as $node) {
    searchTree($node, $myself);
  }
};

searchTree($fruits);
```

新しいことは何もないですね。
これだけだとつまらないので、今度は幅優先探索を書いてみようと思います。JavaScriptのものを含め、今までは深さ優先探索でした。

```php
function breadthSearch($trees)
{
  $nextTrees = [];
  foreach ($trees as $treeWithParent) {

    $tree = $treeWithParent['tree'];
    if (array_key_exists('parent', $treeWithParent)) {
      $myself = $treeWithParent['parent'].' > '.$tree['name'];
    } else {
      $myself = $tree['name'];
    }

    echo $myself."\n";

    foreach ($tree['nodes'] as $node) {
      $nextTrees[] = ['tree' => $node, 'parent' => $myself];
    }
  }

  if (!empty($nextTrees)) {
    breadthSearch($nextTrees);
  }
}

breadthSearch([['tree' => $fruits]]);
```

幅優先探索のコードはちょっとややこしいですね。
自分の子どものnodeをキャッシュしておいて、同一の深さのnodeを処理したあとに、一つ下の深さに潜る感じになります。

実際にコードテストの回答でも幅優先探索を実装しています。これは更にややこしいですね。
https://github.com/motojouya/exam-wikipedia-tree-sugiyama/blob/master/app/Services/Tree/Tree.php

## React
Reactです。
弊社のフロントエンドの大半はReactでできています。

```HTML
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>React Recursive</title>
  </head>
  <body>
    <div id="content"></div>
    <script src="https://unpkg.com/react@16/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@16/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/babel-standalone@6.26.0/babel.min.js"></script>
    <script type="text/babel">

      const fruits = {
        "name": "フルーツ",
        "nodes": [
          {
            "name": "みかん",
            "nodes": [
              { "name": "みかん", "nodes": [], },
              { "name": "レモン", "nodes": [], },
              { "name": "伊予柑", "nodes": [], },
            ]
          },
          {
            "name": "ぶどう",
            "nodes": [
              { "name": "ぶどう", "nodes": [], },
              { "name": "巨峰", "nodes": [], },
              { "name": "マスカット", "nodes": [], },
            ]
          },
          {
            "name": "りんご",
            "nodes": [],
          },
        ]
      };

      const searchTree = trees => {
        return (
          <dl>
            {
              trees.map(tree =>
                <React.Fragment>
                  <dt>{ tree.name }</dt>
                  <dd>{ searchTree(tree.nodes) }</dd>
                </React.Fragment>
              )
            }
          </dl>
        );
      };

      ReactDOM.render(searchTree([fruits]), document.getElementById('content'));
    </script>
  </body>
</html>
```

正直Reactは簡単でしょう。
Vueに比べてJavaScriptでプログラミングする感覚で書けるので、実現方法はJavaScriptで例示したものと基本的に同じです。

## Vue
Vueです。
今後、弊社では新規のプロダクトを作るときはVueで作っていこうという流れです。
Reactも使っているので、そのうちマイクロフロントエンドみたいな話もでてくるかもしれませんね。

プロダクトだとsfcで書くと思っていて、sfcでブラウザコンパイルする方法がわからなかったので、適当にdockerで作りました。
https://github.com/motojouya/vue-recursive-example

ちなみに弊社は基本的に、thinkpadにubuntuをインストールし、docker-composeで開発環境を整えています。ただdocker使わずにやってたりmacで開発している人もいたりで、決まりはありません。

再帰の部分は以下です。

```vue
<template>
  <dl v-if="exist">
    <dt>
      <span>{{ tree.name }}</span>
    </dt>
    <dd v-if="hasChildren">
      <recursive v-for="node in tree.nodes" :tree="node"></recursive>
    </dd>
  </dl>
</template>

<script>
export default {
  name: 'recursive',
  props: {
    tree: Object,
  },
  computed: {
    hasChildren: function () {
      return this.tree
          && this.tree.nodes
          && this.tree.nodes.length
    },
    exist: function () {
      return !!this.tree;
    },
  },
}
</script>
```

ポイントは`name`で名前をつけて呼び出す点です。
VueについてはReactのようにJavaScriptっぽくは書けませんが、HTMLとして直感的に書けますね。

## PostgreSQL
弊社ではMySQLを使っているので、MySQLで紹介すべきところですが、自分がPostgreSQLが好きなので、そちらで例示します。
PostgreSQLはSQL標準準拠なので、まぁMySQLも似たようなもんでしょう(調べてない)。

dockerで適当に環境を作ります。

```shell
docker run --name postgres -e POSTGRES_USER=develop -e POSTGRES_PASSWORD=password -d postgres:9.6
docker exec -it postgres bash
psql -U develop
```

あとはこんな感じ

```sql
create database recursive;
alter database recursive;

CREATE TABLE fruits (
  id INTEGER PRIMARY KEY,
  parent INTEGER,
  name VARCHAR(64) NOT NULL
);

INSERT INTO fruits (id,parent,name        )
            VALUES ( 1,  null,'フルーツ'  )
                 , ( 2,     1,'みかん'    )
                 , ( 3,     1,'ぶどう'    )
                 , ( 4,     1,'りんご'    )
                 , ( 5,     2,'みかん'    )
                 , ( 6,     2,'レモン'    )
                 , ( 7,     2,'伊予柑'    )
                 , ( 8,     3,'ぶどう'    )
                 , ( 9,     3,'巨峰'      )
                 , (10,     3,'マスカット')
                 ;

WITH RECURSIVE recursive_fruits (id, parent, name) AS (
  SELECT fruits.id
       , fruits.parent
       , fruits.name
    FROM fruits
   WHERE fruits.name = 'フルーツ'
UNION
  SELECT fruits.id
       , fruits.parent
       , fruits.name
    FROM fruits
   INNER JOIN recursive_fruits
      ON fruits.parent = recursive_fruits.id
)
SELECT recursive_fruits.id
     , recursive_fruits.parent
     , recursive_fruits.name
  FROM recursive_fruits
 ORDER BY recursive_fruits.id
     ;
```

`with recursive`句で再帰sqlが書けます。上記のSQLでinsertで入れた全てのデータを取得できます。
慣れないとなにがなんだかですが、recursive句の中で自分自身を呼び出し、それをunionでまとめてる感じです。

## wget
最後にwgetです。
コードテストでは使わなかった。というか、自分が知らなかっただけで、おそらく出題した人はwgetを想定していたでしょう。
wikipediaの[フルーツ](https://ja.wikipedia.org/wiki/%E6%9E%9C%E7%89%A9)のページを取得してみようと思います。

```shell
wget -r -l3 -w5 -e robots=off https://ja.wikipedia.org/wiki/%E6%9E%9C%E7%89%A9
```

`r`(recursive)で再帰的に辿れ、`l`(level)でデフォルト5階層まで辿ってくれます。
基本的にサイトは変にクロールしてほしくないので、`robots.txt`で禁止されています。
なので`-e robots=off`でそれを無視する必要があります。
あまり一気にアクセスすると迷惑なので、`w`(wait)オプションで数秒待つほうがいいでしょう。単位は秒です。

## まとめ
再帰はプログラミングにおいて、基本的な概念です。for文などに比べると、なかなか使う場面も少ないですが、覚えておくと強力な手段となると思います。

