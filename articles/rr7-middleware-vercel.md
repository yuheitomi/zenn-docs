---
title: "React Router v7 middleware を Vercel で使う (v7.9安定化版)"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [reactrouter, vercel]
published: true
slug: rr7-middleware-vercel
---

React Router v7 のミドルウェアを Vercel で使う方法です。Vercel 環境でのコンテキスト設定方法を中心に説明します。

:::message alert
Vercel でデプロイする際、`unstable_middleware: true` を設定すると以下のエラーが発生します：

```
Error: Unable to create initial `unstable_RouterContextProvider` instance.
Please confirm you are returning an instance of `Map<unstable_routerContext, unknown>`
from your `getLoadContext` function.

TypeError: init is not iterable
```

このエラーは、`getLoadContext` 関数が `RouterContextProvider` のインスタンスを返していないことが原因です。
:::

## 要因

Vercel の React Router プリセットでは `AppLoadContext` （ミドルウェア以前のもの）がコンテキストオブジェクトとして利用されるように設定されています。一方、ミドルウェア機能を使用する場合は、`RouterContextProvider` のインスタンスを利用する形に変更されており、そのままでは互換性がありません。

## Vercel デプロイ向けのミドルウェア設定手順

以下に Vercel でミドルウェアを利用する手順を示します。なお以下は Vercel 向けの設定についてのみ記述していますので、React Router ミドルウェア全般については先にミドルウェアに関する記事（[React Router v7 ドキュメント 日本語版](https://react-router-docs-ja.techtalk.jp/how-to/middleware) など）を見ていただけると良いと思います。

### 1. ミドルウェア設定 (`react-router.config.ts`)

Vercel 固有ではなく、React Router 全般の設定です。

```diff ts
export default {
  ssr: true,
  presets: [vercelPreset()],
  future: {
-   unstable_viteEnvironmentApi: true,
+   unstable_viteEnvironmentApi: true,
+   v8_middleware: true,
  },
} satisfies Config;
```

### 2. リクエストハンドラーによるコンテキストの設定 (`server/app.ts`)

:::message
この設定が重要です。
:::

Vercel で `AppLoadContext` に変わって `RouterContextProvider` を利用するための設定です。

`server/app.ts` を作成してリクエストハンドラーを設定します（パスは任意ですが、後述する `vite.config.ts` で指定する必要があります）。ここで `RouterContextProvider` のインスタンスを作成しておく事が重要です。

```ts
import * as build from "virtual:react-router/server-build";
import { createRequestHandler, RouterContextProvider } from "react-router";

const handler = createRequestHandler(build);

export default (req: Request) => handler(req, new RouterContextProvider());
```

### 3. Vite 設定の更新 (`vite.config.ts`)

SSR ビルド時に `server/app.ts` を `input` として指定します。

```ts
import { reactRouter } from "@react-router/dev/vite";
import tailwindcss from "@tailwindcss/vite";
import { defineConfig } from "vite";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig(({ isSsrBuild }) => ({
  build: {
    rollupOptions: isSsrBuild ? { input: "./server/app.ts" } : undefined,
  },
  plugins: [tailwindcss(), reactRouter(), tsconfigPaths()],
}));
```

## 使用例 (middleware を使ったサンプルコード)

以下は middleware を使ったサンプルコードですので、Vercel 固有のものではありません。詳細は [React Router - How to Use Middleware](https://reactrouter.com/how-to/middleware) をご参照ください。

### ルートでの Middleware 使用 (`app/routes/home.tsx`)

```ts
import type { Route } from "./+types/home";
import { createContext } from "react-router";

const UserContext = createContext<{ user: { id: string } }>();

const authMiddleware: Route.MiddlewareFunction = async ({ request, context }) => {
  const user = { id: "12345" };
  context.set(UserContext, { user });
};

export const middleware: Route.MiddlewareFunction[] = [authMiddleware];

export function loader({ context }: Route.LoaderArgs) {
  const { user } = context.get(UserContext);
  console.log(`User ID: ${user.id}`);
  return { message: "Hello" };
}
```

## 補足

なおサーバーエントリーポイントとして、`app/entry.server.tsx` をカスタマイズしている場合は注意が必要です。

エントリーポイントに渡される `loadContext` は `RouterContextProvider` になっていますが、Vercel 側の `handleRequest` は引き続き `AppLoadContext` を期待しています。

`loadContext` をそのままパススルーする場合は問題ありませんが、このエントリーポイントで処理が必要な場合、`@vercel/react-router/entry.server` の内容をベースに自身でハンドラーを定義するなどの対応が必要と思います。

```ts
import { handleRequest as vercelHandleRequest } from "@vercel/react-router/entry.server";
import type { EntryContext, RouterContextProvider } from "react-router";

export const streamTimeout = 5_000;

export default function handleRequest(
  request: Request,
  responseStatusCode: number,
  responseHeaders: Headers,
  routerContext: EntryContext,
  _loadContext: RouterContextProvider // RouterContextProvider
) {
  return vercelHandleRequest(
    request,
    responseStatusCode,
    responseHeaders,
    routerContext
    // _loadContext, // Vercel の handleRequest は AppLoadContext を想定
  );
}
```

## 参考

- [React Router - How to Use Middleware](https://reactrouter.com/how-to/middleware)
- [GitHub Issue: Cannot deploy react-router with future unstable_middleware=true #13327](https://github.com/vercel/vercel/issues/13327)
- [React Router on Vercel](https://vercel.com/docs/frameworks/frontend/react-router)
- [React Router v7 middleware を Vercel で使う](https://zenn.dev/articles/3970cb405ffb9a)
