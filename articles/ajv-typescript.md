---
title: "Ajvでスキーマチェックしたいが、型を書きたくない話"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "Ajv", "jsonschema"]
published: true
published_at: 2024-01-26
---

もう長いもので、5,6年オープンロジにいるのですが、採用がうまく言っているのか、皆さんが優秀なのか、TypeScriptが盛り上がり気味です。
まぁでもそれとは関係なく、私は我が道を行くのですが、私自信もプライベートでTypeScriptを使ったときに迷ったことを記事にして、今年のオープンロジアドベントカレンダーとします。

今回はAjvというツールを使ったので、それについて記述します。

以下の記事を参考にさせてもらいました。
https://blog.ojisan.io/ajv-to-type/

## スキーマバリデータについて
TypeScriptというかJavaScriptではJSONを扱うことが多く、構造体の形でパラメータを受け取るケースが多いと思います。
ただ、構造体をチェックするというのは、ネストが深いとやりづらく、専用のライブラリを使うことが求められます。
そこで活躍するのがスキーマバリデータ、あるいはスキーマ定義ライブラリです。

オープンロジだと[Zod](https://github.com/colinhacks/zod)を使っているようですが、最近はフロントに触る機会が減ってしまっているので伝聞情報です。
スキーマバリデータといいつつ、Zodは、HTML formに入ってきた値を構造体に起こし、更にバリデーションを行うという役割で使われることが多いようです。

またスキーマを定義するという意味では、JSON Schemaというものもあります。
こちらはライブラリではなく、規格としてあり、様々な言語で実装されているものです。
今回、僕はJSON Schemaに沿った開発のほうがふさわしいと感じ、JSON Schemaのバリデーション実装である[Ajv](https://ajv.js.org/)を利用しました。

## Ajvについて
AjvはJSON schemaのバリデーションライブラリです。なので、基本的にJSON Schemaを記載し、それを使ってバリデーションなどを実装していきます。
Ajvでバリデーションを行うと、type guardが効くので、その後は値を定義型として扱うことができて非常に便利です。

私は、データベースに保存した値を取得する際に、型がついていない状態にバリデーションをかけてtype guardのおかげで、定義型として扱える。というような使い方をしていました。
実装する上では、スキーマこそ先にあり、それをなぞって型を定義すべきだと考えました。
実装を組み上げた上で、自然にできたインタフェースの定義を使うというのは、未知のものに対しては有効な方法です。
ですが、職業プログラマとしては先にインタフェースを定義し、双方の実装を並列で進められるよう計らうのが責任かなと思うためです。

Ajvにおいては、JSON schemaと型を同時に定義して使います。
だいたいこんな感じになります。

```ts
import Ajv, { JSONSchemaType } from 'ajv';

type Address = {
  zip: integer;
  address: string;
  name: string;
};

// 型を定義して、JSON schemaを定義するという順序
const addressSchema: JSONSchemaType<Address> = {
  type: 'object',
  properties: {
    zip: {
      type: 'string',
      minLength: 7,
      maxLength: 7,
    },
    address: { type: 'string' },
    name: { type: 'string' },
  },
  required: ['zip', 'address', 'name'],
} as const;

function getAddress(obj: any): Address {
  const ajv = new Ajv();
  const validateSchema = ajv.compile<Address>(addressSchema);
  if (!validateSchema(obj)) {
    // @ts-ignore
    const { errors } = validateSchema;
    console.debug(errors);
    throw new Error(errors, 'address schemaと一致しません');
  }

  console.log(`${obj.name}さんの住所`);
  return obj; // Address型
}
```

ですが、型とJSON schemaを両方とも定義するのは面倒だし、重複しているように感じます。
JSON schemaから型を導けるほうが、メンテナンス性が上がっていいような気がします。

そこで[json-schema-to-ts](https://github.com/ThomasAribart/json-schema-to-ts)というライブラリを使いました。
こんな感じに変化します。


```ts
import Ajv, { JSONSchemaType } from 'ajv';
import addFormats from 'ajv-formats';
import { FromSchema, $Compiler, wrapCompilerAsTypeGuard } from 'json-schema-to-ts';

const ajv = new Ajv();
addFormats(ajv);
const $compile: $Compiler = schema => ajv.compile(schema);
const createValidationCompiler = () => wrapCompilerAsTypeGuard($compile);

const addressSchema = {
  type: 'object',
  properties: {
    zip: {
      type: 'string',
      minLength: 7,
      maxLength: 7,
    },
    address: { type: 'string' },
    name: { type: 'string' },
  },
  required: ['zip', 'address', 'name'],
} as const;

// JSON schemaから導けるようになった！
export type Address = FromSchema<typeof addressSchema>;

function getAddress(obj: any): Address {
  const ajv = new Ajv();
  const compile = createValidationCompiler();
  const validateSchema = compile(addressSchema);
  if (!validateSchema(obj)) {
    // @ts-ignore
    const { errors } = validateSchema;
    console.debug(errors);
    throw new Error(errors, 'address schemaと一致しません');
  }

  console.log(`${obj.name}さんの住所`);
  return obj; // Address型
}
```

これでめでたしめでたしです。

## やり残し
ここまではスムーズに実装していたのですが、パラメータをHTML formから受け取りたいという追加の要望がでてきました。
そうなってくると、ajv + json-schema-to-tsを使いたくなります。
ただ、json-schema-to-tsは型が合いませんでした。

```ts
import type { FC } from 'react';

import { ajvResolver } from '@hookform/resolvers/ajv';
import { useForm } from 'react-hook-form';

import { addressSchema, Address, getAddress } from './address';

const AddressForm: FC<{}> = () => {
  const { handleSubmit, register } = useForm<Address>({
    resolver: ajvResolver<Address>(addressSchema),
  });

  const fromForm = async (address: any) => {
    try {
      const address = getAddress(address);
      console.log(address);
    } catch (e) {
      console.log(e);
    }
  };

  return (
    <form onSubmit={handleSubmit(fromForm)}>
      <input id="name" placeholder="name" {...register('name')} />
      <input id="zip" placeholder="name" {...register('zip')} />
      <input id="address" placeholder="name" {...register('address')} />
      <button type="submit">{'submit'}</button>
    </form>
  );
};
```

ajvResolverに渡すパラメータのAddress型を、addressSchemaから導出しているので、そこが循環参照みたいになっているのか、ちょっと追いきれませんでした。

## まとめ
Ajv + json-schema-to-tsという構成を取ることで、型を再定義しなくても良くなりました。
ただ、今回はJSON schemaを先に定義するということにこだわった結果の実装です。

たとえば、HTML formではわざわざJSON schemaで実装する必要はなく、スキーマバリデータの実装に沿う形で構わないはずです。
その場合は、型の導出など、もっと便利なツールがありそうなので、次回はそういったものを試してみたいです。

