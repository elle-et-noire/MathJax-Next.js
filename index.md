---
marp: true
---

# Next.js で MathJax を使おうとして苦労した話

# Next.js とは

- Reactベースのフレームワーク
- SSRやSSG（やISR）ができる

Reactは基本的にブラウザ（クライアント）側でレンダリングを行う。

SSRやSSGは事前にレンダリングを行う。表示速度やSEOで有利。

Linkタグを使うことでページ遷移時に必要な要素だけ書き換えるという最適化が行われるらしい。

# MathJax とは

HTML中に書かれた数式を、埋め込みJavaScriptでSVGなどに展開する。

DOM操作という黒魔術...!

対抗馬のKaTeXとは、デフォルトで\eqrefが使えることや、configを書けばphysicsパッケージなどのLaTeXの便利どころを使えるところで一線を画す。

# MathJax in Next.js ① 素朴に

dangerouslyInnerHTMLという関数でJavaScriptを埋め込む部分をHTML的に書いてみる。

→ リロードしないと数式として展開されない。

プリレンダリングの時点では埋め込まれたスクリプトが動作しないためと思われる。

ページ遷移に合わせて描画できれば良いが（実際MathJaxにはtypeset()という描画関数も用意されているが）、どうReactから扱えばよいか分からなかった。

Linkタグを使わないなどの回避策はあるがあまりきれいではない（Next.js の利点を殺してしまうので）

# MathJax in Next.js ② rehype-mathjax

もともとMarkdownファイルに書かれた記事を表示したいのでもともと途中でremark/rehypeプラグインを挟んでいる。

rehype-mathjaxがMathJax v2しか対応していなかった...!
physicsパッケージはMathJax v3でしか使えない。

# MathJax in Next.js ③ better-react-mathjax

解決篇です。

名にし負う有能ぶり。

# MathJax in Next.js ④ texReset()

ただ、ページ遷移するごとに式番号が増えていく...!あるいは異なるページ間で\labelが重複するとエラーになる...!

Next.jsの、必要な個所以外は書き換えられないという特性が裏目に出ていると思われる。

これはやはりMathJaxにtexReset()という関数が用意されており、better-react-mathjaxはMathJaxObjectも提供してくれるので今度はReactからページ遷移ごとにtexReset()を実行できた。

# 安心も束の間

2022年3月。MathJaxに何やら異変が...

[大きな括弧の表示がおかしい](https://atatat.hatenablog.com/entry/2020/06/21/003000)

Edge、PC版Chromeでは表示がおかしいがAndroid版Chrome、FireFoxでは問題なく表示される。ブラウザの問題だと思われる。

CHTMLではなくSVGを使うようにしたら解消されたが...

# DOM操作

SVGの\refが青色固定らしい。どうしても変えたい場合はDOM操作を強いられることになる。

# 数式プレビュー機能をつけたい① CSSセレクタさん...

CSSセレクタで"~"を使うことによって離れた位置にある要素も選択できるらしい。これを使ってプレビューウィンドウを`position:fixed`でかつ`hidden`で置いておき、\refにホバーしたら`visible`に変わるようにしようとした...が

~ は同じ階層にしか使えないことを知った。\refは`<MathJax~>`の中に`<a>`を生成するのだ。

# 数式プレビュー機能をつけたい① `has`っていうのがあるらしい

子要素をCSSセレクタで指定できないかと思って検索すると`has`という疑似クラスがあるらしいが

[:has()](https://developer.mozilla.org/ja/docs/Web/CSS/:has)

対応ブラウザが壊滅状態...

# 数式プレビュー機能をつけたい① DOM操作しか勝たん

`a[href=数式のURL]`でquerySelectorAllしてマウスが乗ったときに対応する数式ウィンドウを`visible`にするようにした

# XyJax with Tailwindcss

Tailwind を読み込むとなぜか図式の文字と矢印の位置関係がおかしくなる

[tailwind.css を読み込むと XyJax の表示が崩れる](https://ja.stackoverflow.com/questions/86681/tailwind-css-%e3%82%92%e8%aa%ad%e3%81%bf%e8%be%bc%e3%82%80%e3%81%a8-xyjax-%e3%81%ae%e8%a1%a8%e7%a4%ba%e3%81%8c%e5%b4%a9%e3%82%8c%e3%82%8b)

一部のベーススタイルが変更されるためらしい。

# XyJax の図式の色が変わらない

```
\begin{xy}*[white]
\xymatrix{G \ar[d]_\pi \ar[r]^\phi & H  \\G/\operatorname{Ker}\,\phi \ar@{.>}[ur]_\psi}
\end{xy}
```
のように書けば白に変えられる。