---
title: "CloudFront DistributionのカスタムドメインをVercelで管理する"
emoji: "🕸️"
type: "tech"
topics: ["Vercel", "AWS", "CloudFront", "DNS"]
published: true
---

# 前提

- Vercel でドメインを管理していて、ワイルドカードドメインを持っている
- CloudFront にカスタムドメインを適用したい。
- 以前書いた[APIGateway のカスタムドメインを Vercel で管理する](https://zenn.dev/takasing/articles/vercel-apigateway-domain)を読んだ

https://zenn.dev/takasing/articles/vercel-apigateway-domain

# 結論

ACM(Amazon Certificate Manager)の証明書を米国東部 (バージニア北部) (us-east-1) リージョンで取得し、赤枠の部分を埋める。
![Preview of preferences of CloudFront](https://storage.googleapis.com/zenn-user-upload/4a3fa58f6ff9-20221224.png)

Tokyo とかでやっちゃうとエラーでちゃんとアセットが配信できなかった。
よく見ると以下のように記載してある。

> Associate a certificate from AWS Certificate Manager. The certificate must be in the US East (N. Virginia) Region (us-east-1).

# 参考記事

- [代替ドメイン名 (CNAME) を追加することによるカスタム URL の使用](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html)
