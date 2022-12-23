---
title: "Vercel上のnext/imageキャッシュをクリアできないときにやること"
emoji: "️🌄"
type: "tech"
topics: ["vercel", "nextjs"]
published: true
---

# 前提

- Vercel で Next.js アプリケーションをホスティングしている
- `next/image` を利用している

# 解決策

`Redeploy with existing Build Cache`にチェックを入れずに Redeploy する。

![Redeploy without Build Cache](https://storage.googleapis.com/zenn-user-upload/89f01d2262d5-20221223.png)

## なぜか

どうやら `next/image` のキャッシュは Redeploy しても残るらしい。
公式ドキュメントには以下のようには書いているが、「どうしたら `next/image` のキャッシュが消えるよ」という所までは書いてない(ように見える)。

> **Note**: Even if you redeploy, images cached in `/_next/image` or `/_vercel/image` will be preserved.

ため、変なキャッシュが残る事件が起こったときに以下を実行するとキャッシュが消えた。

1. `Redeploy with existing Build Cache`にチェックを入れて Redeploy してもキャッシュが消えない
2. `Redeploy with existing Build Cache`にチェックを入れずに Redeploy したらキャッシュが消えた

# まとめ

Redeploy するときってチェック入れる癖ついてますよね(震え声)。

# 参考記事

- [Image Optimization#Usage](https://vercel.com/docs/concepts/image-optimization#usage)
