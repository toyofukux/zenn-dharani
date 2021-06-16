---
title: "Next.jsとemotionを使ってSSR時にCSSを適用しておく"
emoji: "🍳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Nextjs", "emotion", "SSR", "css"]
published: true
---

先日[Beyond Magazine](https://beyondmag.jp?utm_source=zenn)というメディアをオープンしました。  
Next.js, Shifter での Headless CMS という構成で開発しましたが、いくつか知見を得たのでシェアしておきます。  
(アーキテクチャなどは別途紹介する予定です)

# 構成

- Next.js 10.2.0
- Emotion 11.1.x

# 現象

いくつかの理由で SSR をしているのですが、SSR した場合に初期表示が崩れることがありました。  
`@emotion/css`を使用していましたが、この場合 Next.js の pre-render ページにスタイルが含まれない仕様のためです。Chrome の Inspector で Network のタブを開き該当リクエストを確認してみます。  
Response の HTML に emotion のスタイルが入っていないことと、Preview では表示が崩れていることを確認できます。  
(別の問題ですが svg も 404 表示されています。これは後から気づきました)。

![](https://storage.googleapis.com/zenn-user-upload/d5cs8iakc7i4r73mc2txrjtm45de)

```tsx
import { css } from "@emotion/css";
import Image from "next/image";
export const Header = () => {
  return (
    <header
      className={css`
        width: 100%;
        padding: 16px 0;
        text-align: center;
        border-bottom: 1px solid rgba(0, 118, 255, 0.9);
      `}
    >
      <Image src="/nextjs.svg" alt="Next.js Logo" height={"32"} width={"64"} />
    </header>
  );
};
```

# 解決

emotion で [SSR する際には使える API に制限があり、`@emotion/react`, `@emotion/styled`を使う必要がありました](https://emotion.sh/docs/ssr)。  
(一番上に書いてあるのに気づかず長いことハマりました)

`@emotion/react`は pragma を書いたり、書きたくない場合は runtime に細工する必要があり、更に emotion11 だとバージョン間で挙動が怪しい部分もあり、Issue が複数上がっていたことから、`@emotion/styled`を利用することにしました。  
svg は[`babel-plugin-inline-react-svg`](https://github.com/airbnb/babel-plugin-inline-react-svg)を利用すると pre-render 時に表示できるようになります。

![](https://storage.googleapis.com/zenn-user-upload/0yflv0mk52dzhff54d4ars24rd9i)

```tsx
import styled from '@emotion/styled'
import Nextjs from '../components/icons/nextjs.svg'
export const Header = () => {
  return (
    <StyledHeader>
      <Nextjs />
    </StyledHeader>
  )
}

const StyledHeader = styled.header`
  width: 100%;
  padding: 16px 0;
  text-align: center;
  border-bottom: 1px solid rgba(0, 118, 255, 0.9);
```

# まとめ

実際のプロダクトでは、TOP ページにいきなりカルーセルがあるので、ファーストビューでテキストが画面の左に寄っていたり、画像が一瞬画面いっぱいに表示されてしまう現象が起こっていて UX がとても下がる状態でしたが、わがままを言っいくつかフィーチャーを後回しにしローンチ前に解決しました(焦りましたｗ)。  
[Beyond Magazine](https://beyondmag.jp?utm_source=zenn) では雑誌を作ってきたメンバーを中心に高いデザイン性を目指しています。特に画像のクリエイティブのレベルが高いので、Web の技術者がその世界観を更に実現していけるように改善していく予定です。  
今後はメディア単体でのマネタイズだけでなく質の高いコンテンツと多角的な事業で拡大していくのでご興味のある方は Twitter で DM ください！

![](https://storage.googleapis.com/zenn-user-upload/umo6g8r6dcoj4pz2poy8srgbq8xn)

# 参考

- [In what way is JS any more maintainable than CSS? How does writing CSS in JS make it any more maintainable?](https://gist.github.com/threepointone/731b0c47e78d8350ae4e105c1a83867d)  
  CSS in JS とはなんぞや話。
- [Nextjs で svg をインラインで読み込んで react component として使う](https://naporitan.hatenablog.com/entry/2020/12/28/143545)
