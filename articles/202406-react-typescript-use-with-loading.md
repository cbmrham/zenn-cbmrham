---
title: "react + typescriptでloadingをいい感じに扱うhooksを作った"
emoji: "🥐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript"]
published: true
---

react + typescript において data fetch や reload を行う際に、loading 中は placeholder などを表示し、loading が終わったらデータを表示する、というようなことはよくやると思いますが、その際に loading の true/false と、fetch したデータが取得できているか（undefined でないかどうか）を両方チェックしていると、コードが冗長になりがちです。

ということで、これをいい感じに扱える hooks を作成しました。

なお Suspence は利用しない時代の話です。

## 作成した hooks

```tsx
import { useState, useCallback } from "react";

export const useWithLoading = <P extends unknown[], R>(
  fn: (...args: P) => Promise<R>
) => {
  const [state, setState] = useState<
    { loading: true } | { loading: false; data: R }
  >({ loading: true });
  const call = useCallback(
    async (...args: P) => {
      setState({ loading: true });
      const data = await fn(...args);
      setState({ loading: false, data });
    },
    [fn]
  );
  return { state, call };
};
```

- パラメータとして非同期関数を受け取ります。
- `state` は `{ loading: true }` か `{ loading: false; data: R }` のどちらかの型を持ちます。そのため、ローディング中なら data が undefined であることが保証され、data は戻り値の型になります。
- `call` は非同期関数の呼び出しを行う関数になっています。引数の非同期関数をこの call によって呼び出すことでローディング状態の管理を簡素にすることができます。

## 利用方法 example

@[codesandbox](https://codesandbox.io/embed/dcz3m9?view=editor+%2B+preview&module=%2Fsrc%2FuseWithLoading.ts)

こんな感じで、state.data によって型安全にアクセス可能で、call を使用することで初期ローディングやその後の再 fetch にも簡単に対応できます。

## 所感

使い心地がよく、react + typescript のプロジェクトではだいたい使っちゃってます。ぜひ。
