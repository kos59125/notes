---
title: Bolero で 404 対応を⾏う
date: 2019-12-15
---

<div class="notification">
本記事は以前 Qiita で公開していたものを移植したものです。
</div>

# はじめに

F# 上で Blazor アプリを Elimish で書けるライブラリーの Bolero というものがあります。
雑に⾔うと F# でシングルページアプリケーション（SPA）が簡単に作ることができます。
Bolero についてはあまり⽇本語で情報を⾒かけないのですが、⼊⾨記事を書いて需要があるかわからないので、⾃分が実際にプロジェクトで Bolero を利⽤するにあたって出会った問題（の⼀つ）について扱いたいと思います。
需要︖ 知らない⾔葉ですね。

SPA の問題の⼀つに 404 の扱いがあります。
SPA は⾮同期にコンテンツを読み込んで表⽰を切り替えるため、リクエストされたコンテンツが存在しない場合があります。
しかしリクエストは既に読み込まれたページ上で実施されるため、ページそのものの HTTP ステータスコードとしては 200 となっています。
ウェブページを読み込むときに存在しないときに 404 を返すことをハード 404 と⾔います。
SPA ではハード 404 はできないので、ボットにコンテンツが存在しないことを⽰すためには代替の⽅法が必要になります。
主に以下の⽅法があります。

1. コンテンツが存在しない旨を表⽰する（ソフト 404）
2. 404 エラーページにリダイレクト
3. 実在する別のページにリダイレクト

本記事では、 Bolero で上記の⽅法について実装パターンを⽰します。
なお、 Bolero のベースである Blazor にはクライアントサイド Blazor（Blazor WebAssembly）とサーバーサイド Blazor（Blazor Server）のホスティングモデルが⽤意されていますが、本記事で扱うのはクライアントサイドでの話です。
ホスティングモデルのメリット・デメリットについては「[ASP.NET Core Blazor hosting models](https://docs.microsoft.com/ja-jp/aspnet/core/blazor/hosting-models?view=aspnetcore-3.1)」を参照してください。

# アプリケーションのベース

F# Advent Calendar のデータベースアプリのようなものを作ります。

## ルーティング

```fsharp
type Route =
   | [<EndPoint("/")>] Home
   | [<EndPoint("/{lang}/{year}/{day}")>] AdventItemInfo of lang:string * year:int * day:int
```

http://localhost:5000/ja/2017/1 にアクセスすると、 2017 年の⽇本語版の Advent Calendar の 1 ⽇⽬の記事情報を表⽰するようなイメージです。

## モデル

アドベントカレンダーの記事ごとの情報をモデルに詰め込みます。

```fsharp
type AdventItem = {
   Year : int
   Day : int
   BlogUri : string
   Author : string
   Title : string
   Language : string
}

type Model = {
   Page : Route
   AdventItem : AdventItem option
   Error : string option
}
```

## サービス

⾔語と年と⽇を指定するとそのアドベントカレンダー記事の情報を返すサービスです。

```fsharp
type AdventService =
   {
      GetItem : string * int * int -> Async<AdventItem option>
   } 
   interface IRemoteService with
      member _.BasePath = "/-/advent"
```

GetItem で AdventItem option 型が結果となりますが、これはつまり Some は記事が存在し、 None は記事が存在しないという状態を指します。
本記事の⽬的はこの結果の取り扱いということになります。

## メッセージ

```fsharp
type Message =
   | SetPage of Route
   | GetItem of lang:string * year:int * day:int
   | GotItem of AdventItem option
   | Error of exn
   | ClearError
```

## アプリケーション

```fsharp
let router = Router.infer SetPage (fun model -> model.Page) 

let init _ = { Page = Home; AdventItem = None; Error = None }, Cmd.none 

let update (program:ProgramComponent<_, _>) message model =
   match message with
   | SetPage(page) ->
      let cmd =
         match page with
         | Home -> Cmd.none
         | AdventItemInfo(lang, year, day) -> Cmd.ofMsg <| GetItem(lang, year, day)
      { model with Page = page }, cmd
   | GetItem(lang, year, day) ->
      let remote = program.Remote<AdventService>()
      let cmd = Cmd.ofAsync remote.GetItem (lang, year, day) GotItem Error
      { model with AdventItem = None }, cmd
   | GotItem(item) ->
      { model with AdventItem = item }, Cmd.none
   | Error(RemoteUnauthorizedException) ->
      { model with Error = Some("unauthorized") }, Cmd.none
   | Error(ex) ->
      { model with Error = Some(ex.Message) }, Cmd.none
   | ClearError ->
      { model with Error = None }, Cmd.none 
 
let view model dispatch =
   match model.Page with
   | Home -> homeView dispatch  // ホームページの表示
   | AdventItemInfo(_, _, _) ->
      cond model.AdventItem <| function
         | None -> adventItemLoadingView dispatch  // 読み込み中状態の表示
         | Some(adventItem) -> adventItemView adventItem dispatch  // 記事情報の表示 
 
type AdventApp() =
   inherit ProgramComponent<Model, Message>()
   override this.Program =
      Program.mkProgram init (update this) view
      |> Program.withRouter router
```


## 動作概要

ホーム画⾯でリンク（Day #x）をクリックすると、 SetPage メッセージが送信され、ビューが更新されます。 SetPage メッセージは継続して GetItem、 GotItem とメッセージが続きます（それぞれのメッセージに対してビューの更新を伴う）。
GotItem はサーバーからのリクエストを（正常に）受け取った際に送信されるメッセージです。したがって、 GotItem で None を受信した場合にクライアントサイドで 404 ページを表⽰するようにします。

# ソフト 404

GotItem はサーバーからのリクエストを（正常に）受け取った際に送信されるメッセージです。
したがって、 GotItem で None を受信した場合にクライアントサイドで 404 ページを表⽰するようにします。

```fsharp
type Model = {
   (* omit *)
   IsNotFound : bool
}

type Message =
   (* omit *)
   | NotFound
 
let update program message model =
   (* omit *)
   | GotItem(item) ->
      let cmd =
         match item with
         | Some(_) -> Cmd.none
         | None -> Cmd.ofMsg NotFound
      { model with AdventItem = item }, cmd
   | NotFound ->
      { model with IsNotFound = true }, Cmd.none
 
let view model dispatch =
   match model.IsNotFound with
   | true -> notFoundView dispatch  // 404 ページの表示
   | false -> (* omit *)
```

# 404 エラーページにリダイレクト

Bolero のサービスはあくまで⾮同期リクエストで呼び出すため、 301 等でリダイレクト先を受け取ったとしてもブラウザーはリダイレクトしてくれません。
したがって、ソフト 404 と同じようなやり⽅で、 GotItem で None を受信した場合にクライアントサイドで⾃発的に 404 ページにリダイレクトする必要があります。

```fsharp
type Message =
   (* omit *)
   | NotFound

let update program message model =
   (* omit *)
   | GotItem(item) ->
      let cmd =
         match item with
         | Some(_) -> Cmd.none
         | None -> Cmd.ofMsg NotFound
      { model with AdventItem = item }, cmd
   | NotFound ->
      program.NavigationManager.NavigateTo("/Error/404", true)
      model, Cmd.none

// Startup.Configure
app
   (* omit *)
   .UseEndpoints(fun endpoints ->
      // define 404 page
      endpoints.MapGet("/Error/404", fun req ->
         async {
            do req.Response.StatusCode <- StatusCodes.Status404NotFound
            // build your 404 page contents
         }
         |> Async.StartImmediateAsTask
         :> System.Threading.Tasks.Task
      ) |> ignore
      ... 
   )
```

404 ページは SPA ではなくサーバー側で別のハンドラーとして提供しているため、 URL リダイレクトで強制リロードをかける必要があります（NavigationManager の forceLoad パラメーターを true にする）。


# 実在する別のページにリダイレクト

404 ページにリダイレクトするのとほぼ同じですが、こちらはあくまで SPA の枠組みで完結させるため、 Bolero のルーティングの⽅法に従ってリダイレクトします。

```fsharp
type Message =
   (* omit *)
   | NotFound

let update program message model =
   (* omit *)
   | GotItem(item) ->
      let cmd =
         match item with
         | Some(_) -> Cmd.none
         | None -> Cmd.ofMsg NotFound
      { model with AdventItem = item }, cmd
   | NotFound -> 
      // redirect to Home
      { model with Error = Some("The requested resource is not found.") }, Cmd.of (SetPage(Home))
```

この⽅法の場合、ユーザーは期待するページと異なるページに移動するため、なぜそのようになったかをユーザー側に通知してあげるべきでしょう。

# まとめ

SPA では 404 の扱いが難しいのですが、それに対する⽅法はいくつか提案されています。
特にどれが良いということは本記事では議論しませんが、いずれの選択肢をとるとしても、 Bolero で簡単に実装することができます。
