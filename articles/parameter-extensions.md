---
title: "BashのParameter expansionの一部について整理"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Bash", "Nextjs"]
published: true
---

シェル芸をする際にいつも何でググればよいかわからなくなるあれのことを Parameter expansion というらしい。  
変数が存在しない際にデフォルト値設定したり、変数が存在する場合に変わりの文字を入れたりするあれ。  
~~Next.js のアプリケーションを Vercel にデプロイする際に`.env.local`をうまく拡張する際に必要になったので一部だけ整理しとく。~~ `.env.local`でうまく展開されませんでした。。。

```sh
#!/bin/bash

NO_VARIABLES=
VARIABLES=var

echo "- operator: default value"
echo ${VARIABLES:-default} # var
echo ${NO_VARIABLES:-default} # default
echo ${NO:-default} # default
echo ${VARIABLES-default} # var
echo ${NO_VARIABLES-default} # no output!
echo ${NO-default} # default
echo
echo "= operator: assign"
echo ${VARIABLES:=default} # var
echo ${NO_VARIABLES:=default} # default
echo ${NO1:=default} # default
echo ${VARIABLES=default} # var
echo ${NO_VARIABLES=default} # default <= !!!
echo ${NO2=default} # default
echo
echo "? operator: error"
echo ${VARIABLES:?default} # var
echo ${NO_VARIABLES:?default} # default
#echo ${NO3:?default} # ERROR
echo ${VARIABLES?default} # var
echo ${NO_VARIABLES?default} # default
#echo ${NO4?default} # ERROR
echo
echo "+ operator"
echo ${VARIABLES:+default} # default
echo ${NO_VARIABLES:+default} # default
echo ${NO5:+default} # no output
echo ${VARIABLES+default} # default
echo ${NO_VARIABLES+default} # default
echo ${NO6+default} # no output
```

## 参考

- [Bash Hackers Wiki-Parameter expansion](https://wiki.bash-hackers.org/syntax/pe)
