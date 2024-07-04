---
theme: default
title: CI/CD に PGO を組み込んだ話
author: Shota Iwamatsu
download: true
export:
  format: pdf
  withClicks: true
  withToc: false
drawings:
  persist: false
transition: slide-left
highlighter: shiki
mdc: true
background: https://cdn.jsdelivr.net/gh/slidevjs/slidev-covers@main/static/gKmpcDQWPmY.webp
favicon: /icon.jpg
---

# CI/CD に PGO を組み込んだ話

layerx.go #1

Shota Iwamatsu (@shota8511_tech)


---
layout: two-cols
---

### 自己紹介

<br/>

## Shota Iwamatsu

<br/>

<div>
  <ul class="flex flex-col gap-2">
    <li>LayerX
      <ul>
        <li>2023/8~</li>
        <li>バクラク事業部 カード開発グループ</li>
        <li>ソフトウェアエンジニア</li>
      </ul>
    </li>
    <li>ex. Infcurion, Latona, Accenture</li>
    <li>Go 歴3年くらい</li>
  </ul>
</div>

::right::

<div class="flex flex-col justify-center items-center gap-2 mt-24">
  <img src="/icon.jpg" class="h-50 rounded-full" />
  <a href="https://twitter.com/shota8511_tech">@shota8511_tech</a>
</div>


---
layout: image
image: /layerx_intro.svg
---


---

# 今日のトピック

<br/>

## 1. PGO とは

<br/>

## 2. 実際に導入してみる

<br/>

## 3. 導入してどうなったか


---

# Profile-Guided Optimization (PGO) とは

- コンパイラ最適化手法の1つ。
- ビルド時に、コンパイラにランタイム情報 (プロファイル) を与えることで、より実際のワークロードに即した最適化を行うことができる。
- 平たく言えば、ソースコードの変更無しで、パフォーマンスを向上できるビルド方法のこと。
- Go に限らず様々な言語で提供されている。Feedback-Directed Optimization (FDO) とも呼ばれている。


---
layout: center
---

# 例えば


---

# 関数のインライン展開

- コンパイラ最適化の1つ。
- 関数呼び出しを関数本体に置き換える。

<br/>

````md magic-move
```go
func main() {
  a := 1
  b := 2
  c := sum(a, b)
  fmt.Println(c)
}

func sum(a, b int) int {
  return a + b
}
```

```go {1-6}
func main() {
  a := 1
  b := 2
  c := a + b // 関数呼び出しが関数本体に置き換えられた。
  fmt.Println(c)
}

func sum(a, b int) int {
  return a + b
}
```
````


---
layout: center
---

# 何が嬉しい？


---

# インライン展開のメリット

<br/>

## 1. 関数呼び出しのオーバーヘッドがなくなる

<br/>

- 関数呼び出しは、余分な CPU 命令 (オーバーヘッド) が発生する。
- このオーバーヘッドは通常ナノ秒レベルだが、積み重なることでパフォーマンスに影響が出る。
- インライン展開により関数呼び出しが不要になるので、このオーバーヘッドがなくなる。

<br/>
<br/>

## 2. さらなるコンパイラ最適化が可能になる

<br/>

- インライン展開することで、他の最適化を適用できるようになる。
  - 定数畳み込みやエスケープ解析など。


---

# ただしデメリットもある

- バイナリサイズが増加する。
- ビルド時間が長くなる。

<br/>

### → 通常のビルド時は、単純な関数のみに適用するよう対象を制限している。


---

# とはいえ...

- 頻繁に呼び出される関数は、多少のデメリットを許容してでも、パフォーマンスを向上させたい。
- しかしコンパイラは、実際のワークロードにおいて、どの関数が頻繁に呼び出されるのか分からない。

<br/>

### → プロファイルを与えることで、それを加味した最適化ができるようになる！


---

# Go で PGO を適用する方法

- Go 1.21 で正式リリースされた。
- main パッケージに default.pgo という名前で CPU プロファイルを配置すれば、自動的に適用される。
  - `go build -pgo=/tmp/foo.pprof` で任意のファイルを指定することも可能。
  - プロファイルは pprof のフォーマットに則っている必要がある。
- [公式ドキュメント](https://go.dev/doc/pgo)曰く 2~14% のパフォーマンス向上が見込める。


---
layout: center
---

# 実際に導入してみる


---

# どのようにプロファイルを収集するか

- PGO で良い結果を得るには、実際のワークロードに近いプロファイルを収集できるかどうかが重要。
- 基本的には本番環境のプロファイルを取得することが推奨されている。
- `net/http/pprof` を使えば取得できるが、考えないといけないことが多い。
  - どこからリクエストを送る？
  - いつ・どのくらいの期間で取得する？
  - 取得時に例外的に負荷が少ない状態だったら？


---

# 継続的プロファイラーを使う

- 常時プロファイルを収集し、定期的に管理用のサーバーに送信してくれる。
- 必要な時に任意のプロファイルを DL することができる。
- 各社が提供しているサービスや OSS が存在する。
  - **Datadog: Continuous Profiler**
    - LayerX ではオブザーバビリティに Datadog を使用しているため、今回はこちらを採用。
  - Google Cloud: Cloud Profiler
  - Grafana: Grafana Pyroscope


---

# [Datadog Continuous Profiler](https://docs.datadoghq.com/profiler/)

- プロファイルの一覧から、任意のプロファイルを DL できる。
- フレームグラフを表示したり、プロファイル同士を比較したりもできる。

<br/>

<img src="/profile_list.png" />


---

# [Datadog Continuous Profiler](https://docs.datadoghq.com/profiler/)

- Datadog がライブラリを提供している。
- 既に Datadog を利用している場合は簡単に導入できる。

```go
import (
  "gopkg.in/DataDog/dd-trace-go.v1/profiler"
)

func main() {
  err := profiler.Start(
    profiler.WithService("<SERVICE_NAME>"),
    profiler.WithEnv("<ENVIRONMENT>"),
    profiler.WithVersion("<APPLICATION_VERSION>"),
    // ...
  )
  if err != nil {
    log.Fatal(err)
  }
  defer profiler.Stop()
  // ...
}
```


---

# どのように CI/CD に組み込むか

- PGO において、プロファイルは一度収集して終わりではない。
  - ソースコードは常に変化するので、プロファイルも直近のものを使うのが望ましい。
- ビルドの度に毎回手動でプロファイルを DL してコミットするのは面倒。
- 自動化して CI/CD に組み込みたい。


---

# [datadog-pgo](https://github.com/DataDog/datadog-pgo) を使う

- Datadog が提供している、プロファイルを DL する CLI ツール。
- 直近72時間で CPU 使用量が大きいプロファイルを5個選出し、マージしてくれる。
  - 時間と個数はコマンドライン引数で調整可能。
- `go build` の実行前に datadog-pgo を実行するだけで、CI/CD に PGO を組み込める。

<br/>

```sh
export DD_API_KEY=xxx
export DD_APP_KEY=xxx
go run github.com/DataDog/datadog-pgo@latest 'service:foo env:prod' ./cmd/foo/default.pgo

# datadog-pgo により default.pgo が作成されているので、ビルド時に PGO が適用される。
go build ./cmd/foo/...
```


---
layout: center
---

# 導入してどうなったか


---

# CPU 使用率が平均13.5%減少した！

- メトリクスの計算式: `((CPU 使用率 / 1週間前の CPU 使用率) -1) * 100`
- PGO 適用後3日間について、上記メトリクスの平均値が-13.5%だった。

<img src="/cpu_metrics.png" />


---

# メモリ使用率も平均9.5%減少した！

- メトリクスの計算式: `((メモリ使用率 / 1週間前のメモリ使用率) -1) * 100`
- PGO 適用後3日間について、上記メトリクスの平均値が-9.5%だった。

<img src="/memory_metrics.png" />


---

# ただし...

- パフォーマンス向上の大部分は `gopkg.in/DataDog/dd-trace-go.v1/profiler` だった。
  - PGO によってプロファイラーが最適化されるという、マッチポンプ的な結果になってしまった...。
  - とはいえプロファイラーはパフォーマンス分析に欠かせないので、パフォーマンス向上は嬉しい。
- 実際のエンドポイントに限定すると、約5%のパフォーマンス向上。
  - 1分あたりの CPU 使用時間が約4ミリ秒短縮された。
  - 劇的な改善ではないが、導入の手軽さを考慮すると十分な結果。


---

# その他の留意点

- バイナリサイズが増加する。
  - 今回は0.1%以下の増加だった。
- ビルド時間が長くなる。
  - 今回は誤差の範疇だったため正確な値は不明。


---

# まとめ

- PGO により、ソースコードの変更無しで 2~14% のパフォーマンス向上が見込める。
- プロファイルの収集は、継続的プロファイラーを導入すると便利。
- datadog-pgo を使うと、簡単に PGO を CI/CD に組み込める。
