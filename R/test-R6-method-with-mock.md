---
title: R6 メソッドのテストでモックを使う
---

# はじめに

testthat は `with_mock` 関数を使ってモックを使うことができます。
以下の例では、 f という関数を呼び出す g という関数において、 f が正しく呼び出されていることを f のモックを利用して確認しています。

```r
library(testthat)
library(mockery)

f <- function(x) {
    print(x)
}
g <- function(x) {
    f(x)
}

test_that("test", {
    mock_f <- mock()
    with_mock(
        f = mock_f,
        {
            g("test")
            expect_args(mock_f, 1, "test")
        }
    )
})
```

モック化する対象が関数であれば上記のように簡単にできるのですが、モック化する対象が R6 クラスのメソッドの場合には簡単ではありません。
以下の例は失敗します。

```r
A <- R6::R6Class("A", public = list(
    f = function(x) {
        print(x)
    }
))
B <- R6::R6Class("B", public = list(
    g = function(x) {
        a <- A$new()
        a$f(x)
    }
))

test_that("test", {
    mock_f <- mock()
    with_mock(
        f = mock_f,
        {
            b <- B$new()
            b$g("test")
            expect_args(mock_f, 1, "test")
        }
    )
})
```

```
Error: Test failed: 'test'
* Function f not found in environment testthat.
Backtrace:
 1. testthat::with_mock(...)
 2. testthat:::extract_mocks(mock_funs, .env = .env)
 3. base::lapply(...)
 4. testthat:::FUN(X[[i]], ...)
```

with_mock の引数の `f = ` の部分を `"a$f" =` や `"A$public_methods$f" = ` のように変えてもダメです。
エラーメッセージの通り[^error-message]評価環境（ここでは testthat）から f なる関数見つからないことが原因です。

[^error-message]: インタラクティブ環境でコードを実行すると、既にグローバル環境に f が定義されているので別のエラーメッセージになる場合があります。
                  その場合は `rm(f)` のように f を環境から削除して再度実行してみてください。


# 解決策

やりたいことは `a$f` という関数がモックに置き換えられることです。
それには A のコンストラクター（new）で返されるインスタンスをモックにしてやる必要があるのですが、同様の理由でコンストラクターをモックに差し替えることができません。
となると、 A そのものを差し替えて new がモックを返すようにできればよさそうなのですが、そのような操作は testthat ではサポートしていません。

しかしあきらめる必要はありません。
testthat ではサポートされていなくても、 R には環境を書き換える方法が標準で用意されています。
環境を書き換えて B のインスタンスメソッドである g の関数環境で参照される A を差し替えてやれば良いのです。
具体的には次のようにします。

```r
test_that("test", {
    mock_f <- mock()
    stub_a <- list(
        new = function(...) {
            list(f = mock_f)
        }
    )

    b <- B$new()
    assign("A", stub_a, envir = environment(b$g))
    b$g("test")
    expect_args(mock_f, 1, "test")
})
```

`stub_a` は A のスタブで、 new によってインスタンスの代わりとなるリストを返します。
リストには f という名前のモックを定義しているため、 `A$new` によって作成されたインスタンスのメソッド f がモックに差し替えられるということを再現することができます。
スタブを定義しただけでは B のメソッド g から参照されないので、 B のインスタンス b のメソッド `b$g` の環境に作成した `stub_a` を `assign` 関数で代入してやります。
R では変数の参照は評価環境の定義が最優先されるので、評価環境に代入されたスタブがメソッド内で参照されるというわけです。

この方法の良いところは、インスタンスメソッドの環境のみ書き換えているということです。
書き換えたインスタンスのメソッドのみ書き換えられており、他のインスタンスのメソッドの環境は汚しません。
そのためテスト後に書き換えられた環境をもとに戻すといったこと後始末のことは考えなくても問題は起こりません。
