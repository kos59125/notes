---
title: アクティブパターンで末尾スラッシュを取り除く
---

# はじめに

ファイルパスや URL 操作で、パスの末尾につくスラッシュを除去したい場合があります。
具体的には以下のように関数を定義して呼び出すだけで良いのですが、関数呼び出しの行数が増えてしまうため、見た目上すっきりしません。

```fsharp
let removeTrailingSlash (path:string) =
    path.TrimEnd('/')

let doSomething (path:string) =
    let path = removeTrailingSlash path
    (* メイン処理 *)
```

そこでアクティブパターンを利用してすっきりと書く方法を紹介します。

# アクティブパターンを利用する

アクティブパターンを利用した末尾スラッシュの除去は以下のような形式になります。

```fsharp
let (|NoTrailingSlash|) (path:string) =
    path.TrimEnd('/')

let doSomething (NoTrailingSlash path) =
    (* メイン処理 *)
```

アクティブパターンのパターン数が一つの場合、データを別のデータ形式に変換することができます。
今回のケースでは、末尾にスラッシュがついているかついていなパスを、スラッシュがついていないパスに変換しています。

アクティブパターンは型ではないので、本来のシグネチャに影響しません。
上記の例では doSomething は `string -> *` であって、利用側に特別な型変換を要求することがありません。

アクティブパターンは match の中でも利用できるので、以下のように何らかの処理の結果を受け取って利用することもできます（どちらかというとこちらの使い方が本来の使い方）。

```fsharp
match getPath(x) with
| NoTrailingSlash path -> (* 処理 *)
```

# 参考文献

- [アクティブ パターン - F# | Microsoft Docs](https://docs.microsoft.com/ja-jp/dotnet/fsharp/language-reference/active-patterns)
