---
title: base 要素を指定してもページ内リンクを有効にする
---

# 問題点

HTML 文書内で base 要素を指定すると、 HTML 文書内の相対 URL が base 要素で指定された URL を起点とします。
これはページ内でのアンカー（`<a href="#somewhere-in-page"></a>`）に対しても適用されるため、指定方法次第ではページ内リンクのつもりが別ページへの遷移になってしまうことがあります。
例えば様々なライブラリーを用いると Markdown の脚注を HTML にレンダリングして脚注番号と脚注の相互参照のページ内アンカーが生成することができますが、生成された HTML を読み込むページに base 要素が指定されていると、このような問題が発生してしまいます。

# 解決方法

HTML レンダリング後に、アンカーのクリックによる既定の動作を止め、代わりにページ内スクロール処理を行うことで問題を解決します。

```javascript
document.querySelectorAll("a[href^='#']").forEach((anchor) => {
    anchor.addEventListener("click", (e) => {
        e.preventDefault();
        const target = document.getElementById(anchor.getAttribute("href").substring(1));
        if (target) {
            target.scrollIntoView();
        }
    })
})
```

アンカーの href 属性が \# で始まっている要素をすべて取得し、それぞれのクリック時の既定動作を `preventDefault()` で停止します。
代わりの動作として、 href 属性の \# 以降の文字列を ID に持つ要素の箇所まで [`scrollIntoView()`](https://developer.mozilla.org/ja/docs/Web/API/Element/scrollIntoView) で移動します。

# 参考文献

- https://developer.mozilla.org/ja/docs/Web/HTML/Element/base

