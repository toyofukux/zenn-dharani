---
title: "Shifter HeadlessでWordpressブロックエディタのカスタムブロックを追加する"
emoji: "👻"
type: "tech"
topics: ["Wordpress", "Shifter"]
published: true
---

[Beyond Magazine](https://beyondmag.jp?utm_source=zenn) では Shifter Headless をプロダクト環境で使用しています。  
Headless CMS では、読者が閲覧する記事ページはモダンなフロントエンド技術で開発できる一方、Shifter Headless で利用できる Wordpress ではテーマを作成しないのでカスタマイズに一工夫必要です。  
ここでは Shifter Headless で利用できる Code Snippets プラグインを利用して Wordpress の最新エディタである Gutenberg でカスタムブロックを追加する方法を紹介します。

## 作戦会議

まず、経緯として
**Wordpress の記事をフロントエンドでリッチに表現したい**
というゴールがありました。
一方でプレーンな状態の Wordpress, Gutenberg でできることを整理しておきます。

### Gutenberg と WPGraphQL

Gutenberg で書かれた記事は、WPGraphQL プラグインによりフロントエンドでデータを取得できるので、フロントエンドは GraphQL で取得した内容を描画します。
その際、blocks フィールド 内には生の(Gutenberg が生成した)DOM と attributes という内部の値がそれぞれ取得できます。  
フロントエンドを Next.js, React で実装する際に主に使うのは後者の attributes です。  
よって Gutenberg でカスタムブロックを作って、それをエディタ上で入力し、任意の attributes を取り出す必要があります。

これが記事を取得するクエリ

```graphql
query MyQuery {
  post(id: "hello-world", idType: SLUG) {
    id
    blocks {
      ... on CoreParagraphBlock {
        originalContent
        attributes {
          ... on CoreParagraphBlockAttributes {
            content
          }
        }
      }
    }
  }
}
```

クエリ結果

```json
{
  "data": {
    "post": {
      "id": "cG9zdDox",
      "title": "Hello world!",
      "blocks": [
        {
          "originalContent": "<p>WordPress へようこそ。こちらは最初の投稿です。編集または削除し、コンテンツ作成を始めてください。</p>",
          "attributes": {
            "content": "WordPress へようこそ。こちらは最初の投稿です。編集または削除し、コンテンツ作成を始めてください。"
          }
        }
      ]
    }
  }
}
```

### Shifter Headless の制約

普通に Wordpress をフルフルで使うのであればプラグインなど作れば OK ですが、Shifter Headless では任意のプラグインを追加することができません。  
そこでどうするかと言うと、Code Snippets プラグインを経由して `functions.php` をカスタマイズします。  
とはいえ我々はフロントエンドエンジニアで、(基本的に)React と Typescript だけ書きたい！
よって、Code Snippets 経由で任意の js を読み込ませて `functions.php`をカスタマイズし、カスタムブロックを追加するということを行います。

## Gutenberg project で利用されているライブラリを利用してカスタムブロックを実装

今回記事で紹介しているカスタムブロックはリポジトリにおいてあります => [takasing/wordpress-custom-block](https://github.com/takasing/wordpress-custom-block/tree/1.0.0)

### packages

Gutenberg プロジェクトがサポートしているパッケージはかなりの種類あります。
[Package Reference](https://developer.wordpress.org/block-editor/reference-guides/packages/)
今回使うのは

- [`@wordpress/scripts`](https://developer.wordpress.org/block-editor/reference-guides/packages/packages-scripts/)  
  Wordpress 用の開発をする時に有用なツールセット。今回はビルドするのに使います。
- [`@wordpress/blocks`](https://developer.wordpress.org/block-editor/reference-guides/packages/packages-blocks/)
  Gutenberg のブロックを操作するときのライブラリ。今回はブロックを登録するのに使います。
- [`@wordpress/block-editor`](https://developer.wordpress.org/block-editor/reference-guides/packages/packages-block-editor/)
  Gutenberg をカスタマイズする際に使います。

### カスタムブロックを実装

実装では、ブロック自体の実装と、そのブロックの登録を行います。
今回はかんたんなパラグラフを作っただけですが、`attributes`にはいろんな型のフィールドを追加できますし、Toolbar や Control をカスタマイズすることでいろんな入力を行うことが可能になりますので別の記事で紹介できたらいいなと。

```ts:index.tsx
import { registerBlockType } from "@wordpress/blocks";
import { TestBlockConfig } from "./TestBlock";

registerBlockType("takasing/basic-paragraph-custom-block", TestBlockConfig);
```

```ts:TestBlock.tsx
import { BlockConfiguration, BlockEditProps } from "@wordpress/blocks";
import { RichText } from "@wordpress/block-editor";

import React = require("react");

type TestBlockAttributes = {
  text: string;
};
export const TestBlockConfig: BlockConfiguration<TestBlockAttributes> = {
  attributes: {
    text: {
      type: "string",
      source: "html",
      selector: "p",
    },
  },
  title: "Basic Example",
  icon: "editor-paragraph",
  category: "layout",
  keywords: ["test", "てす"],
  styles: [
    {
      // .is-style-defaultが付与される
      name: "default",
      label: "角丸",
      isDefault: true,
    },
    {
      // .is-style-shadow
      name: "shadow",
      label: "影付き",
    },
  ],
  edit: ({
    attributes,
    setAttributes,
  }: BlockEditProps<TestBlockAttributes>) => {
    const handleChange = (s: string) => {
      setAttributes({ text: s });
    };
    return (
      <RichText tagName="p" value={attributes.text} onChange={handleChange} />
    );
  },
  save: ({ attributes }) => <p>{attributes.text}</p>,
};
```

## serverless framework を使って AWS にバケットを作成し、js をデプロイ

とはいいつつ、とにかくサーバーで js を一つ配信できればよいです。
ローカル環境ではテキトーなサーバーを立てましょう。

```yml
service:
  name: wordpress-custom-blocks

plugins:
  - serverless-s3-sync

custom:
  bucketName: wordpress-custom-blocks-${opt:stage}-hosting
  s3Sync:
    - bucketName: ${self:custom.bucketName}
      localDir: build

resources:
  Resources:
    StaticSite:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead
        BucketName: ${self:custom.bucketName}
        WebsiteConfiguration:
          IndexDocument: index.html
          ErrorDocument: index.html
    StaticSiteS3BucketPolicy:
      Type: AWS::S3::BucketPolicy
      Properties:
        Bucket:
          Ref: StaticSite
        PolicyDocument:
          Statement:
            - Sid: PublicReadGetObject
              Effect: Allow
              Principal: "*"
              Action:
                - s3:GetObject
              Resource:
                Fn::Join: ["", ["arn:aws:s3:::", { "Ref": "StaticSite" }, "/*"]]

provider:
  name: aws
  runtime: nodejs12.x
  region: ap-northeast-1
  stage: ${opt:stage}
  environment:
    STAGE: ${self:provider.stage}
```

あとは S3 にデプロイを行います

```sh
sls deploy --stage beta --aws-profile テキトーなプロファイル名
```

## Code Snippets でバケットから js を読み込む

以下の Code Snippets を登録します。
Code Snippets は読み込み場所を指定できます。今回はエディタでのみ読み込めればよいので`Only run in administration area`を選択しておきます。

```php
function register_block() {
wp_enqueue_script( 'custom-blocks', 'https://bucketName.s3.ap-northeast-1.amazonaws.com/index.js', array( 'wp-blocks', 'wp-element' ), '1.0.0', true );
}
add_action( 'enqueue_block_editor_assets', 'register_block' );
```

## エディタでカスタムブロックが使用できることを確認し、記事を保存

後はエディタでカスタムブロックを使用し、記事を保存します。
![](https://storage.googleapis.com/zenn-user-upload/42e7ce9f20389c0853481f6e.png)

確認のため`GraphiQL IDE`で実際に GraphQL を通してカスタムブロックを利用した記事のデータを取得できるか確認しましょう。
blocks 内の inline fragment の type は`registerBlockType`で指定した名前のキャメルケースのものが Explorer で指定できるようになっているはずです。

クエリ

```graphql
query MyQuery {
  post(id: "hello-world", idType: SLUG) {
    id
    title
    blocks {
      ... on CoreParagraphBlock {
        originalContent
        attributes {
          ... on CoreParagraphBlockAttributes {
            content
          }
        }
        name
      }
      ... on TakasingBasicParagraphCustomBlock {
        originalContent
        attributes {
          text
        }
        name
      }
    }
  }
}
```

結果

```json
{
  "data": {
    "post": {
      "id": "cG9zdDox",
      "title": "Hello world!",
      "blocks": [
        {
          "originalContent": "<p>WordPress へようこそ。こちらは最初の投稿です。編集または削除し、コンテンツ作成を始めてください。</p>",
          "attributes": {
            "content": "WordPress へようこそ。こちらは最初の投稿です。編集または削除し、コンテンツ作成を始めてください。"
          },
          "name": "core/paragraph"
        },
        {
          "originalContent": "<p class=\"wp-block-takasing-basic-paragraph-custom-block\">パラグラフのカスタムブロック</p>",
          "attributes": {
            "text": "パラグラフのカスタムブロック"
          },
          "name": "takasing/basic-paragraph-custom-block"
        }
      ]
    }
  }
}
```

## 各種の罠

- Wordpress のライブラリは各パッケージのバージョンが古いことが多いので注意
  - webpack4 なので ts-loader 9 が使えない
  - React16 なので 17 前提だと困る事がある
- Code Snippets を登録した後、一度エディタをロードしないと Wordpress にブロックが追加されたことにならないらしく、graphql-codegen する際に死にます

## まとめ

Wordpress を Headless CMS にしてフロントエンドを構築する場合、この方法でカスタムブロックを増やしていけば基本的には php を書かずに開発をすることができます。
僕が調査する範囲では Headless CMS についての知見はあまりネット上になく、カスタムブロックを追加するのに非常に手間取りました。
これを気に個人ブログやかんたんな情報発信系のメディアだけではなく、よりリッチな Headless CMS を使ったプロダクトが増えていくといいなと思います。

なお、Beyond Magazine では開発メンバーを募集しているのでご興味ある方は Twitter の DM 下さい！
![](https://storage.googleapis.com/zenn-user-upload/umo6g8r6dcoj4pz2poy8srgbq8xn)
