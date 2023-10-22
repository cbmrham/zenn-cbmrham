---
title: "Next.js App router で多言語化対応 w/next-i18n-router"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "i18n", "nextjs", "next-i18n-router", "App Router"]
published: false
---

[next-i18n-router](https://github.com/i18nexus/next-i18n-router)と[i18next](https://github.com/i18next/i18next)を使用して、すっきりと i18n に対応します。加えて、[react-i18next](https://github.com/i18next/react-i18next)によって CSR に対応します。
[サンプル(GitHub)](https://github.com/cbmrham/next-i18n-app-rouer-example)を公開しています。

## next-i18n-router の設定

まずは[next-i18n-router](https://github.com/i18nexus/next-i18n-router)の README に書いてある通りです(ほぼ英訳)。

- package install

  ```bash
    npm install next-i18n-router
  ```

- config を作る

  ```tsx
  export const i18nConfig = {
    locales: ["en", "ja"],
    defaultLocale: "ja",
  };
  ```

- app 下のファイルをすべて`[locale]`下に移動する

  ```bash
  	└── app
      └── [locale]
          ├── layout.js
          └── page.js
  ```

- middleware を追加する

  ```typescript
  import { i18nRouter } from "next-i18n-router";
  import { NextRequest } from "next/server";
  import { i18nConfig } from "@/i18n/config";

  export function middleware(request: NextRequest) {
    return i18nRouter(request, i18nConfig);
  }

  // only applies this middleware to files in the app directory
  export const config = {
    matcher: "/((?!api|static|.*\\..*|_next).*)",
  };
  ```

ここまでの設定によって、クライアントサイド/サーバーサードそれぞれ以下のような形で、ブラウザの言語設定にしたがったロケールを取得できるようになります。

```tsx
'use client';

import { useCurrentLocale } from 'next-i18n-router/client';
import i18nConfig from '@/i18nConfig';

function ExampleClientComponent() {
  const locale = useCurrentLocale(i18nConfig);

  ...
}
```

```tsx
	// server component
function ExampleServerComponent({ params: { locale } }) {
  ...
}
```

そして、default locale を ja にしている場合、ブラウザの設定が日本語の場合は locale のパスが省略されます。たとえば`products` へのルーティングの場合、パスは

日本語: `/products`

英語: `/en/products`

となります。

## i18next による Sserver Component の対応

i18next を導入して SSG されるコンポーネントに対応します。

- package install

  ```bash
  npm install i18next
  ```

- i18next の言語設定を追加します。

  ```tsx
  import { InitOptions } from "i18next";
  import { en } from "./en";
  import { ja } from "./ja";

  export const i18nextInitOptions: InitOptions = {
    lng: "ja",
    fallbackLng: "ja",
    resources: {
      en,
      ja,
    },
  };
  ```

  ```tsx
  export const ja = {
    translation: {
      hello: "こんにちは！",
    },
  };
  ```

  ```tsx
  export const en = {
    translation: {
      hello: "Hello!",
    },
  };
  ```

- 設定した言語設定を使用し、`layout.tsx`で i18n インスタンスの初期化および言語設定を行います。

  ```tsx
  import i18n from "i18next";
  import { i18nextInitOptions } from "@/i18n/config";

  i18n.init(i18nextInitOptions, (err) => {
    if (err) {
      console.error("i18next failed to initialize", err);
    }
  });

  export default function RootLayout({
    children,
    params: { locale },
  }: {
    children: React.ReactNode;
    params: {
      locale: string;
    };
  }) {
    i18n.changeLanguage(locale);
    return (
      <html lang={locale}>
        <body>{children}</body>
      </html>
    );
  }
  ```

- これで、Server Component でクライアントの言語設定にしたがった t 関数が使用できます。

  ```tsx
  import { t } from "i18next";

  export default function Hello() {
    return <p>{t("hello")}</p>; // 日本語なら`こんにちは！`、 英語なら`Hello!`
  }
  ```

## react-i18next による Client Component の対応

i18next の設定ファイルはそのままに、csr も同様に t 関数をできるようにします。

- package install

  ```bash
  npm install react-i18next
  ```

- Provider を用意します。`layout.tsx`に追加しますが、`I18nextProvider`は server component なのでこちらは client component として用意する必要があります。(`use client`が必要、ということです)

  ```tsx
  "use client";
  import i18n from "i18next";
  import { i18nConfig, i18nextInitOptions } from "@/i18n/config";
  import { I18nextProvider } from "react-i18next";
  import { useCurrentLocale } from "next-i18n-router/client";
  import { ReactNode } from "react";

  i18n.init(i18nextInitOptions, (err) => {
    if (err) {
      console.error("i18next failed to initialize", err);
    }
  });

  export const I18nProvider = ({ children }: { children: ReactNode }) => {
    i18n.changeLanguage(useCurrentLocale(i18nConfig));
    return <I18nextProvider i18n={i18n}>{children}</I18nextProvider>;
  };
  ```

- `layout.tsx`に追加します。

  ```tsx
  import i18n from "i18next";
  import { i18nextInitOptions } from "@/i18n/config";
  import { I18nProvider } from "./i18nProvider";

  export const metadata = {
    title: "Next.js",
    description: "Generated by Next.js",
  };

  i18n.init(i18nextInitOptions, (err) => {
    if (err) {
      console.error("i18next failed to initialize", err);
    }
  });

  export default function RootLayout({
    children,
    params: { locale },
  }: {
    children: React.ReactNode;
    params: {
      locale: string;
    };
  }) {
    i18n.changeLanguage(locale);
    return (
      <html lang={locale}>
        <body>
          <I18nProvider>{children}</I18nProvider>
        </body>
      </html>
    );
  }
  ```

- これで、Server Component と同様に Client Component でも同じように多言語対応できます。

  ```tsx
  "use client";
  import { t } from "i18next";

  export default function Hello() {
    return <p>{t("hello")}</p>; // 日本語なら`こんにちは！`、 英語なら`Hello!`
  }
  ```

## 所感

はじめは[Next.js のドキュメント](https://nextjs.org/docs/app/building-your-application/routing/internationalization)にある形で実装しましたがちょっと煩雑になっている感が否めなかったのですが、next-i18n-router にあやかりスッキリしました。

Server Components での対応は NextRequest で i18n のインスタンス管理ができれば middleware で設定できそうな気もしますが、上手くできなかったので layout.tsx に記述してしまう形で落ち着きました。（いい方法あればぜひコメントください）
