---
title: "React Router v7 middleware を Vercel で使う (v7.9安定化版)"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [reactrouter, vercel]
published: false
slug: rr7-middleware-vercel
---

React Router v7 ミドルウェアを Vercel で使う方法についての説明です。

React Router v7 ミドルウェアへの移行については、基本的に公式サイトの記事を参照すれば問題ないのですが、Vercel へのデプロイの場合、デフォルトで作成される Context がミドルウェア以前の AppLoaderContext になっておりそのままではビルド後、ランタイムエラーが発生します。
