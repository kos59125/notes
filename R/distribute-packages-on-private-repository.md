---
title: プライベートリポジトリ―でパッケージを配布する
---

# はじめに

R パッケージの配布方法は公式の CRAN リポジトリーで配布する以外にも、 Bioconductor のような著名なサードパーティーのリポジトリーで配布したり、 GitHub で公開して remotes パッケージでインストールさせるというような方法もあります。
GitHub は今やメジャーなプラットフォームであり、比較的導入に対する抵抗感も低いのですが、環境によっては使いづらいケースもあります。
ここではプライベートなリポジトリーを作成し、そこでパッケージを配布するという方法について示します。

# リポジトリー構造

リポジトリーは単なる HTTP サーバーとして公開することができます[^repository-server]。
リポジトリーのベースとなる URL の直下に以下のようなディレクトリー構造を作ります。

- src/contrib
- bin/windows/contrib/x.y
- bin/macosx/contrib/4.y
- bin/macosx/el-capitan/contrib/3.y

これらのディレクトリーはそれぞれ `install.packages` 関数の `type` 引数に与える値によってどのフォルダーを見に行くかを変えます。
`type = "source"` の場合は src/contrib、 `type = "win.binary"` の場合は `bin/windows/contrib/x.y` といった具合です。

[^repository-server]: HTTP サーバーのほかに FTP サーバーも可能ですが、現代において特別な理由がなければ選択することはないでしょう。

ここからは話を単純にするためにソース配布のケース（src/contrib）の場合について記述します。

src/contrib の下には、ソースパッケージファイルとインデックスファイルの 2 種類の配布を配置します。
ソースパッケージファイルは `R CMD build` コマンドで作成される .tar.gz 形式のファイルです。
インデックスファイルは、ソースファイルを配置したディレクトリーで `tools::write_PACKAGES` を実行すれば PACKAGES、 PACKAGES.gz、 PACKAGES.rds という 3 つのファイルを吐き出します。

PKG というパッケージのソースパッケージファイルを作成し、インデックスファイルを作成する場合は以下のようなイメージです。

```bash
$ R CMD build PKG
$ Rscript -e "tools::write_PACKAGES()"
```

インデックスファイルは以下のような内容が含まれます。

```
Package: PKG
Version: 1.0
Depends: anotherpkg
MD5sum: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh
NeedsCompilation: no
```

# インストール

src/contrib 内にソースパッケージファイルとインデックスファイルを配置して HTTP サーバーで公開すると、 `install.packages` 関数でインストールできるようになります。

```r
install.packages("PKG", repos = my_repos_url)
```

配布パッケージが他のパッケージに依存していない場合は良いのですが、多くの場合は他のパッケージに依存しています。
その場合、 repos 引数に公式の CRAN も含めてやることで、依存パッケージが CRAN で配布されていれば依存解決することができます[^minicran]。

```r
install.packages("PKG", repos = c("https://cloud.r-project.org", my_repos_url))
```

[^minicran]: ちなみにインターネットに疎通できないような閉じた環境では、 CRAN にアクセスできないため、依存パッケージも含めてリポジトリーを構築することになります。
             そのようなリポジトリーを作る場合は、依存関係から必要なパッケージを抽出して配置してくれる miniCRAN パッケージが便利です。

# 複数のバージョンを配布する

開発中のテスト版と、現行の安定版を同時に配布したい場合があります。
複数のバージョンのソースパッケージファイルがディレクトリー内にあるとき、 `tools::write_PACKAGES()` は最新バージョンのインデックスのみ作成します。
ディレクトリー内のすべてのバージョンをインデックスしたいときは `tools::write_PACKAGES(latestOnly = FALSE)` とすることで、過去バージョンも含めてインデックスを作成することができます[^official-cran]。

```
Package: PKG
Version: 1.0
Depends: anotherpkg
MD5sum: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh
NeedsCompilation: no

Package: PKG
Version: 1.1
Depends: anotherpkg
MD5sum: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh
NeedsCompilation: no

Package: PKG
Version: 2.0
Depends: anotherpkg
MD5sum: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh
NeedsCompilation: no
```

[^official-cran]: しかし公式 CRAN はこの方式をとっておらず、 src/contrib の直下には最新版のソースパッケージファイルのみ配置し、古いバージョンのパッケージは src/contrib/Archive 以下にパッケージごとにディレクトリーを作成してその中にファイルを配置しています。
                  おそらく古いシステムでディレクトリー内のファイル数上限を回避するためでしょうか。

このようなインデックス構成に対して `install.packages` 関数は、特に指定しなければ最新版をインストールすることになります。

では古いバージョンをインストールするにはどのようにすれば良いでしょうか。
実はこれは `install.packages` 関数のヘルプを見るだけでは解決せず、 `available.packages` 関数のヘルプを見る必要があります[^help-install]。

[^help-install]: `install.packages` のヘルプをよく読むと、ドット・ドット・ドット引数（`...`）が `available.packages` に渡されると書かれています。

`available.packages` 関数の引数 `filters` はデフォルトで `NULL`（標準のフィルター）ですが、ここに適切なフィルターを与えてやることで、古いバージョンのパッケージをインストールすることができます。
`filters` には文字列で定義済みのフィルターの名前を与えることもできますが、定義済みフィルターには古いバージョンのパッケージをインストールするようなものはありません。
`available.packages` 関数が返す行列を適切にフィルターする行列を定義してやることで、指定したバージョンのパッケージをインストールできるようになります。
具体的には以下のようにします。

```r
version_filter <- function(version) {
    function(available) {
        available[available[, "Version"] == version, , drop = FALSE]
    }
}
install.packages("PKG", filters = list(version_filter("1.0")), repos = ...)
```

`version_filter` は、バージョンを与えるとフィルター関数を返す関数です。
注意する点として、 `drop = FALSE` を与えないと 1 行のみヒットした場合に行列からベクトルになってしまいます。

なお、自作フィルターだけで十分であれば不要ですが、デフォルトのフィルターも適用したい場合はフィルターリストに `add = TRUE` を追加します。
しかし、次の理由から（少なくとも今回のような特定のバージョンを抽出させるケースでは） `add = TRUE` を利用せず、標準フィルターを利用する場合は明示的に文字列指定しましょう。

## 標準フィルターに関する注意

標準フィルターは `c("R_version", "OS_type", "subarch", "duplicates")` と同義で、以下のような意味を持ちます。

- `R_version`: R バージョンの要求を満たす
- `OS_type`: OS の要求を満たす
- `subarch`: 32-bit/64-bit の要求を満たす
- `duplicates`: 同名のパッケージが複数存在する場合、最新版のみを取得する

このうち `duplicates` が厄介です。

`filters` 引数に与えるリストに `add = TRUE` を指定した場合、自作フィルターの前に標準フィルターが適用されます。
そのため、バージョンを指定してインストールしたいような場合に `add = TRUE` を用いると、事前に最新版のみフィルターされた状態から自作フィルターが適用され、古いバージョンが取得できなくなります。

## その他のフィルター例

標準フィルターの `duplicates` を使うと、最新版のみを取得するという性質を利用して、バージョン 1.x の最新版を取得するようなフィルターも書けます。

```r
major_version_1_filter <- function(available) {
    available[grep("^1.", available[, "Version"]), , drop = FALSE]
}
install.packages("PKG", filters = list(major_version_1_filter, "duplicates"), repos = ...)
```

# まとめ

プライベートリポジトリーで配布する際は HTTP サーバーで特定のディレクトリー構造で特定のファイルを配置するだけです。
プライベートリポジトリーであれば古いバージョンのパッケージも一緒に配布し、ユーザーにインストールさせることも実現できます[^remotes]。

[^remotes]: remotes パッケージには `install_version` 関数という、特定のバージョンを指定してインストールすることができる関数が容易されています。
            しかし残念ながら現行バージョン（2.1.1）は公式 CRAN の仕様に合わせて Archive 下から取得するようになっており、本記事で紹介したようなインデックス中に複数バージョンが含まれているケースに対応していません。
            これは現在 remotes の[プルリクエスト](https://github.com/r-lib/remotes/pull/305)で対応作業が進められていますので、将来的には解決するはずです。

# 参考文献

- [R Installation and Administration](https://cloud.r-project.org/doc/manuals/r-release/R-admin.html)
- `help(available.packages)`
