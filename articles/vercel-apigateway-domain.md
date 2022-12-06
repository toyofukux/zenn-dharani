---
title: "APIGatewayのカスタムドメインをVercelで管理する"
emoji: "🕸️"
type: "tech"
topics: ["Vercel", "AWS", "APIGateway", "DNS"]
published: true
---

# 前提

- Vercel でドメインを管理していて、ワイルドカードドメインを持っている
- APIGateway にサブドメインを適用したい

例えば、
「Next.js アプリケーションを Vercel でドメインごと管理しており、Backend が AWS で APIGateway を使用していて、 `api.example.com` みたいに API 用のサブドメインを割り当てたい」みたいなシチュエーション。

# 手順

1. AWS Certificate Manager(以下 ACM)で証明書を発行
2. 証明書の所有権確認
3. APIGateway にカスタムドメインを適用
4. CNAME レコードを追加
5. 動作確認

## ACM で証明書を発行

割り当てたいサブドメインを決めたら、ACM で証明書を発行します。
![](https://storage.googleapis.com/zenn-user-upload/18cd7e3536ae-20221206.png)

赤枠の値を使って所有権を確認します。(次ステップへ)
![](https://storage.googleapis.com/zenn-user-upload/6b8afc0a2dd7-20221206.png)

## 証明書の所有権確認

確認用の CNAME を登録するだけでは失敗してしまうので、まず CAA レコードを追加。
[(オプション) CAA レコードの設定](https://docs.aws.amazon.com/ja_jp/acm/latest/userguide/setup-caa.html)

割り当てたいサブドメインに対して以下の CAA レコードを追加する。

```
0 issue "amazon.com"
```

CAA レコードと CNAME レコードの合計 2 レコードを追加するとこのようになります。(letsencrypt.org のものは勝手に追加される)

![](https://storage.googleapis.com/zenn-user-upload/22267ee3d7de-20221206.png)

追加後、少し待って ACM 上で Status が Success になったら成功。

## APIGateway にカスタムドメインを適用

証明書が発行できたら早速 APIGateway の Endpoint に対してカスタムドメインを割り当てる変更を行う。

![](https://storage.googleapis.com/zenn-user-upload/d1df1346fcfe-20221206.png)

証明書が正しく発行できてたら ACM Certificate が選択できる状態になっていて、 "Create domain name" を押して作成が成功する。  
証明書ステータスが Success になっててもここで失敗する場合は少し待つ。

![](https://storage.googleapis.com/zenn-user-upload/f64c6ea066aa-20221206.png)

カスタムドメイン設定できると以下のようになる。
赤い部分は後で CNAME として追加するドメイン。
![](https://storage.googleapis.com/zenn-user-upload/48f355a0e13a-20221206.png)

カスタムドメイン設定するだけじゃなくて、カスタムドメインをどのステージに向けるかのマッピングを追加する。
パスも設定できる。バージョニングとかに使うと便利そう。
![](https://storage.googleapis.com/zenn-user-upload/8d8f6de30fcd-20221206.png)

## CNAME レコードを追加

設定作業の〆として、CNAME レコードを追加する。
これまで設定している状態のまま追加しようとすると失敗する。理由は CAA レコードが残っているため。
Vercel のサポートに問い合わせたら以下のような回答が返ってきた。

「CAA レコードは CNAME レコードに従うため、CNAME を使用している場合は必要ないはず。」とのことで、バリデーションを設けてある。

```
You're getting this message because you can't set a CNAME record and a CAA record on the same subdomain.

To fix this, you need to remove the CAA records for api.xxxxxxxx.com and you will then be able to use the CNAME record. CAA records follow the CNAME record, so they shouldn't be needed when using a CNAME but if you still want to set the CAA records, you can do this at root level by adding the Amazon CAA record to the apex/root domain.
```

なのでまず CAA レコードを消す。

![](https://storage.googleapis.com/zenn-user-upload/8d070c5fac89-20221206.png)

消したら APIGateway に追加したカスタムドメインの Endpoint を CNAME として登録。
最終的に以下のようになっていれば OK。

![](https://storage.googleapis.com/zenn-user-upload/21251c767175-20221206.png)

## 動作確認

最後動作確認して終わり。
もしうまく疎通できてなかったら `dig` とかでトラブルシュートする。

# まとめ

個人的に Vercel で CAA レコードを消すという所がハマリポイントでした。
それにしても Vercel のサポートはレスが早くてよい。

# 参考記事

- [API Gateway でカスタムドメインを利用する（SSL 証明書のみ AWS 機能で用意するパターン）](https://dev.classmethod.jp/articles/api-gateway-custom-domain/)
