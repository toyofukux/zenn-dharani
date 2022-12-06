---
title: "Safariでも指定したエレメントまでスムーズスクロールする"
emoji: "📜"
type: "tech"
topics: ["React", "Typescript", "Javascript"]
published: true
---

記事の最後にサンプル実装リポジトリへのリンク置いてます。

# 背景

回遊性向上のために、コンテンツ表示量を増やしたい場面がありました。
この場合、ページ表示時にヘッダとかをすっ飛ばしてメインコンテンツまでスクロールする方法があります。
`Element.scrollIntoView`を使うと指定した要素にスムーズにスクロールができるんですが、Safari だとオプションに対応してなくてスムーズにスクロールができません。  
よって Polyfill をあてる必要がある。

![Browser Support Status](https://storage.googleapis.com/zenn-user-upload/367cad230d2c-20221206.png)
https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView

# 利用する Polyfill ライブラリ

Typescript 版を使いたいので、直接は以下のライブラリを利用します。
https://github.com/magic-akari/seamless-scroll-polyfill

元のライブラリはこれ。ドキュメントとかはこっちがわかりやすい。
https://github.com/iamdustan/smoothscroll
http://iamdustan.com/smoothscroll/

# Code

```ts:useScroll.ts
import { useEffect, useRef } from "react";
import { elementScrollIntoViewPolyfill } from "seamless-scroll-polyfill";

const useMountEffect = (fn: () => void) => useEffect(fn, []);

export const useScroll = <T extends HTMLElement>(key: string) => {
  const ref = useRef<T>(null);
  const scrollTo = () => {
    // add scroll-margin style on target element
    ref.current?.scrollIntoView({
      behavior: "smooth",
    });
  };

  useMountEffect(() => elementScrollIntoViewPolyfill());

  useEffect(() => {
    scrollTo();
  }, [key]);

  return {
    ref,
  };
};
```

ポイントは、対象 Element に`margin-top`がついている場合はスクロール後の見た目が美しくないので(ケースバイケース)、Element 側に`scroll-margin-top`とかをつけることです。
`Window.getComputedStyle`とかも一応使えそう。

# 〆

Github に置いてます。
https://github.com/t104fk/polyfill-scroll
