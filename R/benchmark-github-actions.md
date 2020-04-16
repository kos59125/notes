---
title: GitHub Actions で R の環境ごとの ベンチマークをとった
date: 2019-12-08
---

<div class="notification">
本記事は以前 Qiita で公開していたものを移植したものです。
移植にあたり一部改変を加えています。
</div>

# 要旨

- GitHub Actions を使って R をサポートする各 OS（Windows、 Linux、 macOS）で複数の R のバージョンが動く環境で CSV の読み取り速度を計測した

# はじめに

CSV を読み込むときに utils::read.csv は遅いとか tidyverse 時代には readr::read_csv とか巨⼤ファイルなら data.table::fread だとか最近なら vroom::vroom が速いだとか、よくわからないので⽐較計測（ベンチマーク）してみたいと思います。

ベンチマークをとる際に注意したいのが環境の差をどう扱うかという点です。
ハードウェア環境やソフトウェア環境をできるだけそろえた状態で計測しなければ、関数のパフォーマンスを正しく⽐較することはできません。

ところで今年 GitHub Actions という機能が公開され、先⽉正式版として取り⼊れられました。 GitHub Actions はいわゆる CI/CD を実現する機能なのですが、その実⾏環境の OS は Windows、 macOS、 Ubuntu と各種取り揃えられています。しかも[いずれの実⾏環境も同⼀ハードウェア環境の仮想マシンとして提供されている](https://help.github.com/ja/actions/reference/virtual-environments-for-github-hosted-runners#supported-runners-and-hardware-resources)ため、クロスプラットフォームでベンチマークをとるのに適しています[^hardware]。

[^hardware]: 厳密に⾔えば、同⼀ホスト内の別の仮想マシンの影響は受けるため完全に同⼀の環境は保証されません。

# 注意

本記事は GitHub Actions を様々な環境で利⽤するということを主眼に置いているため、ベンチマークとしては不完全です。
ベンチマークをとる際は、⾃⾝の環境に合わせた適切な⽅法で⾏ってください。

# やること

## 比較対象

### 関数

以下の 3 関数で CSV を読み込む場合の速度を比較します[^vroom]。

- utils::read.csv
- readr::read_csv
- data.table::fread

[^vroom]: vroom::vroom も最初⽐較対象に⼊れていたのですが、 Windows の R-3.4 でインストールできなかったり、 Linux で Too many open files エラーが出たり⾯倒だったので対象から外しています。

### データ

⾏数、列数が様々なサイズのヘッダー付き CSV を読み込みます。
中⾝はいずれも数値とし、上記関数の読み込み時に (1) ⾃動型判別、 (2) 型指定、 (3) ⽂字列型指定の 3 通りで 読み込みます。

## 環境

### R

R 3.4.4 および R 3.6.1 を利⽤します。
ALTREP が導⼊された R 3.5 より前の最新バージョンと、現在の最新バージョンです。

他にも R 実⾏環境は M[icrosoft R Open](https://mran.microsoft.com/open) や [FastR](https://github.com/oracle/fastr) といった⾼速を謳ったものもあります が、本記事では扱いません。

### OS

GitHub Actions のサポートする下記 OS を利用します。

- Windows Server 2019（windows-latest）
- Ubuntu 18.04（ubuntu-latest）
- macOS X Catalina 10.15（macos-latest）

## 制限

環境、ファイルサイズによっては、メモリー不⾜などの理由により、ベンチマークが回りきらない場合がありました。
これについては、全部の関数が回りきるように調整したため、多少ベンチマークとしては物⾜りない部分があるかもしれません。

# GitHub Actions の準備

GitHub Actions は⼤まかに以下の流れで処理します。

1. 計測⽤データ⽣成
2. ベンチマーク（OS、R バージョンごと）
    1. 計測⽤データ取得
    2. R インストール
    3. パッケージインストール
    4. ベンチマーク実⾏
    5. ベンチマーク結果保存
3. 可視化

全部説明すると⻑くなるので、ベンチマークのための環境準備（R が実⾏できるようになるまで）についてのみ説明します。

## 環境準備

[r-lib/actions/setup-r](https://github.com/r-lib/actions) を利⽤するのと簡単に R がインストールできます。

```yaml
steps:
  - uses: r-lib/actions/setup-r@v1
    with:
      r-version: '3.6.1'
```

r-lib/actions/setup-r はマトリックスビルドをサポートしているため、今回のような複数バージョンでの実⾏が簡単に実現できます。

```yaml
strategy:
  matrix:
    R: ['3.4.4', '3.6.1']
  steps:
    - uses: r-lib/actions/setup-r@v1
      with:
        r-version: ${{ matrix.R }}
```

## パッケージインストール

R のパッケージをインストールする際に、管理者権限が要求されたりすると⾯倒なので、カレントディレクトリー内にパッケージをインストールするのが簡単だと思います。
.libPaths 関数でカレントディレクトリー内の lib ディレクトリーをパッケージサーチパスに追加することができます（install.packages の lib パラメーターでも OK）。

特に Linux はパッケージのビルドに時間がかかるため[^package-build]、インストールしたパッケージはキャッシュしておけば次回以降のビルド時間の節約になります。
今回は setup.R で lib デ ィレクトリーにパッケージインストールを⾏うようにしたため、 OS と R のバージョンごとにパッケージをキャッシュするためのキャッシュ⼿順は以下のようになります。

```yaml
steps:
  - id: cache-packages
    uses: actions/cache@v1
      with:
        path: lib
        key: lib-${{ runner.os }}-${{ matrix.R }}-${{ hashFiles('setup.R') }}

  - if: steps.cache-packages.outputs.cache-hit != 'true'
    run: Rscript --vanilla setup.R
```


[^package-build]: CI/CD 時代ではいかにビルド時間を削り、変更から出⼒までの時間を短縮できるかが重要です。
                  tidyverse を利⽤するのであれば tidyverse がセットアップ済みの Docker イメージを利⽤したり、パッケージがビルド済みバイナリーで配布される Windows を使うような⼯夫が⼤事です。
                  ちなみに本記事では microbenchmark、 tidyverse、 data.table、（当初使う予定だったけど使わなくなった）vroom をインストールしていますが、バイナリーでインストールする Windows と macOS が 30 秒⾜らずでインストールが完了するのに対し、 Linux は 20 分弱の時間を要しました。

## ジョブ間のデータの共有

ジョブごとに実⾏環境が異なるため、前のジョブで作成したファイルを別のジョブに利⽤させるためには upload-artifact アクションおよび download-artifact アクションを利⽤します。
前述の GitHub Actions の流れでは、⽣成したデータと、ベンチマーク結果をそれぞれ後続のジョブに共有しています。

以下は data ディレクトリーとその中⾝をアーティファクト（成果物）としてジョブ間で共有する例です。

```yaml
jobs:
  first:
    steps:
      - uses: actions/upload-artifact@v1
        with:
          name: data
          path: data
  second:
    needs: first
    steps:
      - uses: actions/download-artifact@v1
        with:
          name: data
```

複数のジョブから同⼀名のアーティファクト（ディレクトリー）に対してアップロードを⾏っても、ファイル名が異なればそれぞれのアーティファクトが保存されます。

# ベンチマーク

microbenchmark::microbenchimark を利⽤して各 CSV 読み込み関数の⽐較を⾏います。
read_type の値によって、読み込み時の型指定を制御しています。

```r
switch(
   read_type,
   "auto" = {
      c1 <- NA
      c2 <- NULL
   },
   "specify" = {
      c1 <- rep("numeric", ncol)
      c2 <- paste(rep("d", ncol), collapse = "")
   },
   "character" = {
      c1 <- rep("character", ncol)
      c2 <- paste(rep("c", ncol), collapse = "")
   }
) 
 
microbenchmark(
   "utils::read.csv" = utils::read.csv(file, colClasses = c1),
   "readr::read_csv" = readr::read_csv(file, col_types = c2),
   "data.table::fread" = data.table::fread(file, colClasses = c1),
   times = times
) %>%
   summary(unit = "s") %>%
   select(expr, lq, median, uq)
```

# 結果

（省略）

# 結論

（結果の解釈について省略）

GitHub Actions はジョブごとにランタイムを変更したり、ジョブ間のデータ共有も簡単です。
環境ごとの特性を理解しつつ GitHub Actions を利⽤することで、快適な CI/CD 生活を送りましょう。
