---
title: "jsでオブジェクトのdeep diffを取得する"
emoji: "🐾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "i18n", "nextjs", "AppRouter"]
published: true
---

js のオブジェクトの deep な差分において、どのような差分がオブジェクト内のどこにあるのかを取得するライブラリを使用したいと思ったが、自分が利用したいユースケースに対応したものがなく ts のサポートも強めにしたかったので、自作しました。

## ユースケース

- オブジェクトの deep な差分を取得したい
- 差分のある項目に対して、その項目名と前後の値を取得したい

たとえば、以下のようなオブジェクトがあるとします。

```typescript
type Article = {
  id: number;
  title: string;
  description?: string;
  body: string;
  tags: string[];
  option: ?{
    some: string;
  };
};

const before: Article = {
  id: 1,
  title: "タイトル変更前",
  body: "body",
  tags: ["tag1", "tag2"],
};
const after: Article = {
  id: 1,
  title: "タイトル変更後",
  description: "詳細",
  body: "body",
  tags: ["tag1", "tag3"],
  option: {
    some: "何かしらのオプション",
  },
};
```

この差分に対して、以下のようなイメージで値を出力したい。

- 変更された項目
  - title が変更されました
    - 変更前: タイトル変更前
    - 変更後: タイトル変更後
  - 以下の tags が削除されました
    - 値: tag2
  - 以下の tags が追加されました
    - 値: tag3
  - description が追加されました
    - 追加された値: 詳細
  - optoin.some が追加されました
    - 追加された値: 何かしらのオプション

なので、diff の構造としては以下のような形になっていて欲しい。

```typescript
type Diff = {
  {
    key: string;
    operation: "update" | "add" | "delete";
    before?: any;
    after?: any;
  };
};
type Result = Diff[];
```

## 作成したライブラリの使い方

インストール

```bash
npm install @cbmrham/recursive-diff
```

以下のように使うことができます。

@[codesandbox](https://codesandbox.io/embed/3lljlw?view=Editor+%2B+Preview&module=%2Fsrc%2FApp.tsx)

リストに対しては差分が以下のようにでます（ i18n との組み合わせは工夫が必要なので改善の余地ありそう）

```typescript
const x: X = {
  nested: {
    foo: 1,
    bar: "before",
    list: [{ foo: "a" }],
  },
};
const y: Y = {
  nested: {
    foo: 2,
    baz: "after",
    list: [{ foo: "b" }],
  },
};
const result = diff(x, y);
// result = [
//   {
//     operation: "update",
//     path: "nested.foo",
//     before: 1,
//     after: 2,
//   },
//   {
//     operation: "delete",
//     path: "nested.bar",
//     before: "before",
//     after: undefined,
//   },
//   {
//     operation: "update",
//     path: "nested.list.0.foo",
//     before: "a",
//     after: "b",
//   },
//   {
//     operation: "add",
//     path: "nested.baz",
//     before: undefined,
//     after: "after",
//   },
// ];
```

## 作成したライブラリの使い方　 w/TypeScript

ユースケースに記述した型より高度にしているので、パラメータに応じた result の型が利用できます。

```typescript
type X = {
  nested: {
    foo: string;
    bar: string;
    list: Array<{ foo: string }>;
  };
};
type Y = {
  nested: {
    foo: string;
    baz: string;
    list: Array<{ foo: string }>;
  };
};
const x: X = {
  nested: {
    foo: "1",
    bar: "before",
    list: [{ foo: "test" }],
  },
};
const y: Y = {
  nested: {
    foo: "1",
    baz: "after",
    list: [{ foo: "test" }],
  },
};
// X,Yはパラメータから型推論されるので必須ではない
const result = diff<X, Y>(x, y);
result.forEach((r) => {
  if (r.path === "nested.foo") {
    console.log(r.before); // r.before type becomes string | undefined
    console.log(r.after); // r.after type becomes string | undefined
  } else if (r.path === "nested.baz" && r.operation === "add") {
    console.log(r.before); // r.before type becomes undefined
    console.log(r.after); // r.after type becomes string
  } else if (r.path === "nested.list.0.foo" && r.operation === "delete") {
    console.log(r.before); // r.before type becomes string
    console.log(r.after); // r.after type becomes undefined
  }
});
```

- 型の仕様
  - path の型はパラメータのオブジェクトの key をドット区切りで連結したもの
  - before と after の型は parameter のオブジェクトの key に対応する型
  - operation の型は"update" | "add" | "delete"
    - add の場合、before は undefined になる
    - delete の場合、after は undefined になる

## 実装

github に公開しているので、興味があればご覧ください！

[@cbmrham/recursive-diff](https://github.com/cbmrham/recursive-diff)
