---
title: "Reactでpropsにスプレッド構文を使った場合にclassNameが意図通り適用されない"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "Typescript"]
published: true
---

ハマったのでメモ。
以下のような Text Component があるとする。
このコンポーネントでは div をラップしているので、Props には HTMLDivElement の attributes をユニオンしており、onClick などをいちいち定義しなくてよいようにしている。

```tsx
export type Props = React.HTMLAttributes<HTMLDivElement> & {
  size: number;
  weight?: Weight;
  align?: Align;
  className?: string;
};
const Text: React.FC<Props> = (props) => {
  const { size, weight, block = false, align, className, children } = props;

  const className2 = "hoge";
  return (
    <div className={clsx(className, className2)} {...props} block={block}>
      {children}
      {style.styles}
    </div>
  );
};
```

`{...props}`と記述して onClick などのプロパティを展開して div 適用しているが、`className`が意図した値にならない。
これはスプレッド構文により`className`が上書きされたためであり、この場合だと`className2`が適用されない。
なので記述する順番を考慮し以下のようにスプレッド構文は先頭に書く。

```tsx
const Text: React.FC<Props> = (props) => {
  const { size, weight, block = false, align, className, children } = props;

  const className2 = "hoge";
  return (
    <div {...props} className={clsx(className, className2)} block={block}>
      {children}
      {style.styles}
    </div>
  );
};
```

コンポーネント props をスプレッド展開する場合は先頭じゃないとだめ！的な eslint があると嬉しいなあ。
