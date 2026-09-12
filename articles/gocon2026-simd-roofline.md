---
title: 'Recap: Go × SIMDで高速化するベクトル検索 ~ ルーフラインモデルでSIMDが効く境界を探れ！ ~'
emoji: 🔍
type: tech
topics: [go, simd, gocon, vectorsearch]
published: true
published_at: 2026-09-12 02:40
---

## はじめに

2026/09/11(金)に開催されたGo Conference 2026に参加してきました。中でも以下のショートワークショップは、その場では理解しきれない部分が多かったため、終了後に改めて手を動かしながら復習しました。本記事では、そこで得た気づきと学びをまとめます。

@[card](https://gocon.jp/2026/timetable/1264338/)

基本的な説明はスライドやリポジトリに記載されているため、そちらを参照してください。

@[card](https://speakerdeck.com/po3rin/go-x-simd-de-kousokuka-suru-bekutoru-kensaku-de-simd-ga-kiku-kyoukai-o-sagure)
@[card](https://github.com/po3rin/gocon2026-simd-search)



## 前提と計測環境

本ワークショップの目的は、10万件のベクトル検索を高速化することでした。

前提は以下の通りです:

- 探索は全探索
- クエリは 1 本ずつ
- 1 コアのみ(並列化は付録扱い)
- データはメモリ上の配列、384 次元 float32の形式
- [実行環境](https://github.com/po3rin/gocon2026-simd-search/blob/main/docs/workshop/setup.md)

つまり、1 回の検索ごとに、この 153.6 [MB](10 万[件] * 384 [次元] * 4 [byte]) を全部メモリから読むことになります。

この重い処理の高速化の手段として、Go 1.27 で追加された `simd` パッケージの機能を利用していきます。

## SIMD と Go のパッケージ

本章では、まず SIMD の概念について触れ、次に Go の `simd` / `simd/archsimd` パッケージについて触れます。

### SIMD

SIMD（Single Instruction, Multiple Data）は、1 つの命令で複数のデータをまとめて処理する CPU の機能です。詳しくは、[パタヘネ](https://link.amazon/B0j7r9chJ)の説明が演習つきでわかりやすかったのでオススメです。

具体的に、次のようなループ処理について考えてみましょう。

```go
    var sum float32
    for i := range a {
        sum += a[i] * b[i]
    }
    return sum
```

これをそのままコンパイルすると（`GOARCH=amd64 go build -gcflags=-S`）、ループ本体の演算は次の 3 命令になります。

```asm
MOVSS  (AX)(CX*4), X0    ; X0 = a[i]        1 個ロード
MULSS  (DI)(CX*4), X0    ; X0 = X0 * b[i]   1 個掛ける
ADDSS  X0, X1            ; X1 = X1 + X0     1 個足す（X1 が sum）
```

:::details ループ制御と境界チェックを含む全体

```asm
                            ; 前処理
XORPS  X1, X1               ; sum = 0（X1 が sum）
XORL   CX, CX               ; i = 0
JMP    33                   ; ループ条件へ飛ぶ

                            ; ↓ ここからがループ本体（33 → 46 → 21 → 33 で回る）
CMPQ   BX, CX               ; i < len(a) ?        ← BX = len(a)
JLE    50                   ;   終了なら 50 へ
MOVSS  (AX)(CX*4), X0       ; X0 = a[i]           ← AX = &a[0]
CMPQ   SI, CX               ; i < len(b) ?        ← SI = len(b)
JHI    21                   ;   OK なら 21 へ
JMP    55                   ;   NG なら panic
MULSS  (DI)(CX*4), X0       ; X0 *= b[i]          ← DI = &b[0]
ADDSS  X0, X1               ; sum += X0
INCQ   CX                   ; i++
                            ; → CMPQ に戻る

MOVUPS X1, X0               ; 戻り値へ
RET
CALL   runtime.panicBounds(SB)
```

`a[i]` 側の境界チェックはループ条件（`CMPQ BX, CX`）と融合して消えていますが、`b[i]` 側（`CMPQ SI, CX`）は残っています。コンパイラは `len(a)` と `len(b)` の関係を知らないためです。ループの前に `b = b[:len(a)]` と書くと消える可能性があります。

:::

注目したいのは命令の末尾の **`SS`** です。これは Scalar Single の略で「float32 を 1 個」を意味します。つまり 1 命令で float32 を 1 個しか扱っていません。これをスカラ処理と呼びます。

面白いのは、使われている `X0` / `X1` が **128 [bit] の XMM レジスタ**だという点です。float32 が 4 個入る箱を用意しておきながら、1 個分しか使っていません。対して CPU には 256 [bit] のベクトルレジスタ（`Y0` など）もあり、**float32 (32 [bit]) が 8 個入ります**。8 個詰めて掛け算命令を 1 回実行すると、8 個分の掛け算が同時に走ります[^4]。この発想が SIMD です。

[^4]: この詰めた 1 個ぶんの区画をレーンと呼びます。

つまり 384 次元の内積なら、スカラで 384 回かかる掛け算が 48 回で済むので、理論上は 8 倍になるはずです(実際はそう上手くはいかないのですが...w)。

このあと Stage 1 で SIMD 化すると、上の 3 命令が `Y` レジスタと **`PS`（Packed Single）** の命令に置き換わります。**`SS` が `PS` に変わるのが、そのまま「1 個 → 8 個」の違い**になります。

![scalar-vs-simd.png](https://raw.githubusercontent.com/po3rin/gocon2026-simd-search/refs/heads/main/docs/images/scalar-vs-simd.png)
*https://github.com/po3rin/gocon2026-simd-search/ より引用*

では、次は Go における `simd` パッケージについて見ていきましょう。

### Go の simd パッケージ

[Go 1.27](https://go.dev/doc/go1.27#simd) では `GOEXPERIMENT=simd` を付けてビルドすると SIMD のパッケージである [`simd` パッケージ](https://pkg.go.dev/simd)が使えます。simd パッケージには、アーキテクチャ固有用の (`archsimd`) とポータブル用の(`simd`) の2つが存在しています。これらが別々に存在している理由は、[#73787](https://github.com/golang/go/issues/73787) によると、Go 自身はシンプルでポータブルな思想がある一方で、ハードが持つ SIMD 演算は本質的に複雑で非ポータブルであるという特性の違いがあるためです。

| | [`simd`](https://pkg.go.dev/simd) | [`simd/archsimd`](https://pkg.go.dev/simd/archsimd) | 
| --- | --- | --- |
| Float32の型名 | `Float32s`  | `Float32x4` / `Float32x8` / `Float32x16` |
| レーン幅 | 実行時に CPU が決める (4 / 8 / 16)| 型で固定  |
| 使える命令  | 全アーキ共通に持てる演算だけ| そのアーキの命令ほぼ全部 |
| 対象アーキ | 1 ソースで全アーキ | amd64 / arm64 / wasm で API が別 |

また、関連するメインの Issue は調べた限り以下の通りです。
- [#78979](https://github.com/golang/go/issues/78979)（open）: AMD64 の archsimd を**デフォルト有効にする**提案。通れば `GOEXPERIMENT` が不要になる
- [#79781](https://github.com/golang/go/issues/79781)（open）: ARM64 の SVE 命令セット対応。SVE はレジスタ幅が実装依存なので、ポータブル API の設計とも絡みそう
- [#79413](https://github.com/golang/go/issues/79413)（open）: 標準ライブラリ `crypto` の手書きアセンブリを Go の SIMD に置き換える。これが本来の狙いだと思う
- [#80857](https://github.com/golang/go/issues/80857)（closed / completed）: min/max の畳み込みループをコンパイラが自動ベクトル化する

ただし、ワークショップではポータブルな `simd` では Stage 2 以降で必要な命令が足りないため `archsimd` で進めていきます。

### Go の simd/archsimd パッケージ

**1. 型が「データの形」を表す**

型名そのものが「何 bit 幅のレジスタに、どの型を何個詰めるか」を意味します。型を選ぶことが、使う命令幅とレーン数を選ぶことになります。

```go
var a archsimd.Float32x8    // float32 を 8 レーン  = 256 [bit] (AVX2)
var b archsimd.Float32x16   // float32 を 16 レーン = 512 [bit] (AVX-512)
var c archsimd.Uint64x4     // uint64 を 4 レーン
```

**2. メソッドが 1 つの CPU 命令に対応する**

各メソッドはベクトル命令にほぼ 1 対 1 で変換されるので、メソッド名から出てくる機械語の見当がつきます。

```go
va := archsimd.LoadFloat32x8(xs)   // スライス -> レジスタ (ロード)
va = va.MulAdd(vb, acc)            // 積和     -> VFMADD
xo := vc.Xor(vd)                   // XOR      -> VPXOR
po := xo.OnesCount()               // popcount -> VPOPCNTQ
va.Store(xs)                       // レジスタ -> スライス (ストア)
```

**3. 使う前にその CPU が対応しているか確かめる**

未対応の CPU でメソッドを呼ぶと、**不正命令 (SIGILL) でプロセスごと落ちてしまいます**。recover できる panic にもならないので、実行時に機能フラグでガードする必要があります。
例えば、x86 では `MulAdd` が使う FMA 命令が AVX2 に含まれておらず別の拡張として提供されているので、以下のように 2 つ確認する必要があります。arm64 では FMA 相当の命令が Neon 自体に含まれるため、この区別はないそうです。

```go
var hasSIMD = archsimd.X86.AVX2() && archsimd.X86.FMA()
```

それと、この機能フラグ自体のバグが複数報告されていることを確認しました。正しくチェックを書いたつもりでも検出側が間違っていれば意味がないので、x86 系での導入を検討している場合はウォッチしておくと良さそうです。

- [#78772](https://github.com/golang/go/issues/78772)（open）: `X86.AVX512()` が darwin/amd64 で常に false になる
- [#79437](https://github.com/golang/go/issues/79437)（open）: `internal/cpu` の `HasGFNI` が `HasAVX512F` の内側に誤って入れ子になっている
- [#81008](https://github.com/golang/go/issues/81008)（closed / completed）: `GODEBUG=simd=+` が CLMUL の有無を反転して報告していた

## 性能の上限確認

SIMD の概念と Go で利用するパッケージは分かりましたが、高速化のためには何が律速で SIMD によって何が改善されるかを理解する必要があります。
そこで、本章では関連する用語とルーフラインモデル、ピークの算出方法について触れます。

### FLOP (OP)

FLOP (FLoating-point OPeration) は、浮動小数点演算 1 回を意味する単位です。例えば、`x := a + b` は加算 1 回 で 1 [flop]、`z := a * b + c` なら乗算と加算 1 回ずつで 2 [flop] です[^1]。浮動小数点数ではない場合は、単に OP を用いることもあります。

[^1]: [FMA](https://ja.wikipedia.org/wiki/%E7%A9%8D%E5%92%8C%E6%BC%94%E7%AE%97)（Fused Multiply-Add、`a * b + c` を 1 命令で行う命令）は1 命令ですが、今回は 2 [flop] とカウントしています。

今回の検索 1 回だと、1要素あたりの内積の計算は積和の計算なので、 `2 [flop] * 384 [次元] * 100,000 [件] = 76.8M [flop]` です。これを実時間で除算することで、性能を示す [flop/s] を算出できます。

### 算術(演算)強度

算術強度（Arithmetic Intensity, AI）は「メモリから 1 バイト運ぶごとに何回計算するか」を意味する指標です。単位は `[flop/byte]`です。他の文献をあたると「演算強度」という単語が用いられていることが多かったですが、本記事ではワークショップ資料にあわせて算術強度という言葉を用います。

この単位系の数値を持つ理由は、CPU に計算する能力（[flop/s]）とデータを運ぶ能力（[byte/s]）という独立した 2 つの指標を加味した指標であるからだと思います。例えば、今回の浮動小数点数の内積なら「4 バイトを運んできて、計算は2 回」なので `2 [flop] / 4 [byte] = 0.5 [flop/byte]`となります。

### 演算ピークと算出方法

演算ピークとは、そのマシンが「計算だけ」に専念したときに出せる性能の上限です。演算ピークの出し方は理論値と実測値の 2 つがあります。

まず、理論値は以下の式で算出できます。例えば、EPYC 7763 なら、FMA ユニットが 2 つあるので、 `2 * 8 * 2 * 3.5GHz ≒ 110 GFLOP/s` になります。

```
演算ピークの理論値 = FMA ユニット数 [op/cycle] * レーン数 [lane/op] * FMA 1 個の flop [flop/lane] * クロック [cycle/s]
```

一方、実測値はメモリに触らずレジスタ上で FMA だけを回し続けるベンチで導出される値です。

https://github.com/po3rin/gocon2026-simd-search/blob/fe2f38aaa8d23fc43b264ddee2a7e3384ee68e0c/Makefile#L86-L90

確認には上記の `make roofline-ceiling` が利用可能で、同じ 4 コア Codespace での教材の実測値は以下の通りでした:

```
演算ピーク  25.59 GFLOP/s
メモリ帯域  20.80 GB/s（読むだけ）
→ リッジ(後述)    25.59 / 20.80 ≒ 1.23 [flop/byte]
```

ここで、実測値が理論値の 1/4 程度にしかならないことが分かります。原因はハードの限界ではなく、Go のコンパイラがアキュムレータをレジスタに置き続けられず、FMA のたびにスタックと往復させるためであり、これを register spill と呼びます。

実際に、具体的なアセンブリを見ると以下のようになっており、FMA 1 個ごとに load と store が付くので、load/store ポートが先に飽和することが分かります。

```asm
VMOVDQU 0x1b8(SP), Y2    ; スタックからレジスタへ load
VFMADD213PS Y1, Y0, Y2   ; FMA
VMOVDQU Y2, 0x1b8(SP)    ; レジスタからスタックへ store
```

この問題は既知の issue [golang/go#76969](https://github.com/golang/go/issues/76969) として報告されていますが、現状では closed as not planned 扱いでした。

@[card](https://github.com/golang/go/issues/76969)

### ルーフラインモデル

高速化の手を打つ前に **「いま何が理由で詰まっているか」と「追加でどの程度伸びる余地があるか」** を明確にするためのモデルで、以下のように屋根の形をしているのが名前の由来です。初出は[Roofline: An Insightful Visual Performance Model for Floating-Point Programs and Multicore Architectures](https://escholarship.org/content/qt5tz795vq/qt5tz795vq.pdf)（Williams et al. 2008）です。縦軸を性能 [flop/s]、横軸を算術強度 [flop/byte] に取ると、基本的にはこの線より上には行けないことを意味します。

![roofline-concept.png()](https://raw.githubusercontent.com/po3rin/gocon2026-simd-search/refs/heads/main/docs/images/roofline-concept.png)
*https://github.com/po3rin/gocon2026-simd-search/ より引用*

この上限は、

```
上限 [flop/s] = min(演算ピーク, 算術強度 [flop/byte] * メモリ帯域 [byte/s])
```

で算出されます。演算ピークは概ね定数になると思うので、算術強度が小さい場合は「算術強度 [flop/byte] * メモリ帯域 [byte/s]」によって確定し、ある点を超えると演算ピークで性能が頭打ちになります。この境目をリッジと呼びます。

```
リッジ   = 演算ピーク / メモリ帯域 = 25.59 / 20.80 ≒ 1.23 [flop/byte]
算術強度 = 0.5 [flop/byte]  <  リッジ 1.23          → メモリ律速
上限     = 算術強度 * メモリ帯域 = 0.5 * 20.8 ≒ 10 [GFLOP/s]
```

今回の演算では、リッジと上限は上記の通りで、この全探索は 10 [GFLOP/s] 前後が天井であると考えられます。

また、それぞれの演算はルーフラインモデルのどこかに点として存在します。その点の位置に応じて次の通り打つ手が変わります。

| 点の位置 | 詰まっているもの | 打つ手 | ワークショップでの該当部分 |
| --- | --- | --- | --- |
| 斜線に張り付いている | データ転送 | 運ぶ量を減らす / 運んだデータを使い回す（算術強度を上げる） | Stage 1b → 2 |
| 水平線に張り付いている | 演算 | 実装効率を上げる（SIMD、アキュムレータ分割） | Stage 2 |
| どちらからも遠い | 実装の無駄（依存連鎖など） | まず素直に実装効率を上げる | Stage 0 → 1a |

これらの対策について、早速ワークショップの内容に入っていきましょう。

## ワークショップ
本章で紹介する内容は私自身が書いたコードです。[リポジトリ](https://github.com/po3rin/gocon2026-simd-search/)には pon さんの参考コードがあります。

@[card](https://github.com/po3rin/gocon2026-simd-search)

### Stage 0: スカラ基準

最適化は基準値から始まります。以下の Search メソッドから Dot 関数が呼び出されます。

```go
func Dot(a, b []float32) float32 {
    var sum float32
    for i := range a {
        sum += a[i] * b[i]
    }
    return sum
}

func (ix *Index) Search(q []float32, k int) []Result {
    t := newTopK(k)
    for id := 0; id < ix.N; id++ {
        t.push(id, vec.Dot(q, ix.Vec(id)))
    }
    return t.results()
}
```

この実装で bench を回すと、以下の結果が得られました。これが基準値です。

```
BenchmarkSearchNaive-4    68    34877996 ns/op    4403.92 MB/s    0.5000 AI(flop/byte)    2.202 GFLOP/s    153.6 MB/query
```

そして、ここでの課題は2つ。

- a.浮動小数点演算を 1 つずつしか実行できないこと
- b.`sum` の演算が逐次処理で、前の演算が終わるまで次を実行できないこと

次の Stage でこれらの課題解決に取り組んでいきます。

### Stage 1a: AVX2 で 8 個まとめる

まず Stage 0 の課題 a（浮動小数点演算を 1 つずつしか実行できないこと）を対処します。具体的には、`Float32x8` でスライスから 8 要素をベクトルレジスタにロードし、`MulAdd`（FMA）でアキュムレータに足し込む形に置き換えます。

```go
func Dot(a, b []float32) float32 {
    var acc archsimd.Float32x8
    for len(a) >= 8 {
        va := archsimd.LoadFloat32x8(a)
        vb := archsimd.LoadFloat32x8(b)
        acc = va.MulAdd(vb, acc)
        a, b = a[8:], b[8:]
    }

    var buf [8]float32
    acc.Store(buf[:])
    archsimd.ClearAVXUpperBits()
    sum := buf[0] + buf[1] + buf[2] + buf[3] + buf[4] + buf[5] + buf[6] + buf[7]
    for i := range a {
        sum += a[i] * b[i]
    }
    return sum
}
```

この実装で bench を回すと、以下の結果が得られました。34.878 [ms] から 10.175 [ms] で **3.43 倍**、7.548 GFLOP/s まできました。

```
BenchmarkSearchNaive-4    238    10174585 ns/op    15096.44 MB/s    0.5000 AI(flop/byte)    7.548 GFLOP/s    153.6 MB/query
```

### Stage 1b: アキュムレータを 2 本にする

Stage 0 の 課題 b (`acc` の処理ブロックが逐次処理であること) に対処するため、アキュムレータを2本にして対処します。

```go
func Dot(a, b []float32) float32 {
    var acc, acc2 archsimd.Float32x8
    for len(a) >= 8*2 {
        va := archsimd.LoadFloat32x8(a)
        vb := archsimd.LoadFloat32x8(b)
        acc = va.MulAdd(vb, acc)

        va = archsimd.LoadFloat32x8(a[8:])
        vb = archsimd.LoadFloat32x8(b[8:])
        acc2 = va.MulAdd(vb, acc2)
        a, b = a[16:], b[16:]
    }

    var buf [8]float32
    acc.Add(acc2).Store(buf[:])
    archsimd.ClearAVXUpperBits()
    sum := buf[0] + buf[1] + buf[2] + buf[3] + buf[4] + buf[5] + buf[6] + buf[7]
    for i := range a {
        sum += a[i] * b[i]
    }
    return sum
}
```

この実装で bench を回すと、以下の結果が得られました。10.175 [ms] から 9.826 [ms] で、3.5 % しか速くなっていませんでした[^2]。理由として考えられるのは、全探索の粒度ではもう別の上限に当たっているからで、帯域が 15.6 [GB/s] まで来ていて DRAM 側が効き始めている可能性があると思います。ルーフラインモデルにおける斜線上に近い状態です。

[^2]: 教材では内積単体で 348 [ns] から 55.2 [ns]（6.3 倍）になると記載されており、アキュムレータ分割は効くはずなのですが🤔 シングルコア想定なので、その辺り上手く動かないのかもしれません。

```
BenchmarkSearchNaive-4    237    9825801 ns/op    15632.31 MB/s    0.5000 AI(flop/byte)    7.816 GFLOP/s    153.6 MB/query
```

そのため、次はルーフラインモデルにおける点を右に動かす方針をとりたいです。そのために算術強度を上げる必要があり、分母（運ぶ量）を小さくするか、分子（計算回数）を大きくするかになります。今回は前者で進めていきます。

### Stage 2: int8 量子化でバイトを削る

算術強度の分母である運ぶ量を減らすために、各要素を float32（4 [byte]）から int8（1 [byte]）に置き換えます。これを量子化と呼びます。

```
算術強度 = 2 [op] / 1 [byte] = 2.0 [flop/byte]   （4 倍・リッジ 1.23 の右へ）
転送量   = 153.6 MB → 38.4 MB/query              （1/4）
```

今回用いる量子化は対称スカラ量子化です。これは比率を変えずに値を潰すイメージで、ベクトルごとに絶対値の最大値 `maxAbs` を測り、`scale = maxAbs / 127` として各要素を `round(v / scale)` で -127〜127 に丸めるロジックです。以下で示す実装を見ればイメージがつきやすいと思います。

:::details スカラ量子化の関数実装

```go
func QuantizeInt8(v []float32, out []int8) (scale float32) {
	var maxAbs float32
	for _, x := range v {
		a := x
		if a < 0 {
			a = -a
		}
		if a > maxAbs {
			maxAbs = a
		}
	}
	if maxAbs == 0 {
		for i := range out[:len(v)] {
			out[i] = 0
		}
		return 1
	}
	scale = maxAbs / 127
	inv := 127 / maxAbs
	for i, x := range v {
		q := int32(math.Round(float64(x * inv)))
		if q > 127 {
			q = 127
		}
		if q < -127 {
			q = -127
		}
		out[i] = int8(q)
	}
	return scale
}
```

:::

また、こちらの量子化は非可逆圧縮なので、外れ値が 1 つあると他の要素の分解能が落ちます。しかし、ベクトル検索で知りたいのは「上位 N 件」という大小関係なので、大きな問題にはならないと考えられます。また、float から int の演算になるので、評価指標も [flop/s] から単なる [op/s] になります。

```go
func Dot(a, b []int8) int32 {
    // データの型が int8 なのにアキュムレータの型が int32x8 なのは、乗算をしたときに int8 だと値が溢れてしまうため
    var acc0, acc1 archsimd.Int32x8
    for len(a) >= 16*2 {
        acc0 = acc0.Add(archsimd.LoadInt8x16(a).ExtendToInt16().
            DotProductPairs(archsimd.LoadInt8x16(b).ExtendToInt16()))
        acc1 = acc1.Add(archsimd.LoadInt8x16(a[16:]).ExtendToInt16().
            DotProductPairs(archsimd.LoadInt8x16(b[16:]).ExtendToInt16()))
        a, b = a[32:], b[32:]
    }
    if len(a) >= 16 {
        acc0 = acc0.Add(archsimd.LoadInt8x16(a).ExtendToInt16().
            DotProductPairs(archsimd.LoadInt8x16(b).ExtendToInt16()))
        a, b = a[16:], b[16:]
    }

    var buf [8]int32
    acc0.Add(acc1).Store(buf[:])
    archsimd.ClearAVXUpperBits()
    sum := buf[0] + buf[1] + buf[2] + buf[3] + buf[4] + buf[5] + buf[6] + buf[7]
    for i := range a {
        sum += int32(a[i]) * int32(b[i])
    }
    return sum
}

func (ix *Index) Code8(id int) []int8 {
	return ix.Codes8[id*ix.Dim : (id+1)*ix.Dim]
}

func (ix *Index) SearchInt8(q []float32, k int) []Result {
	q8 := make([]int8, ix.Dim)
	qScale := QuantizeInt8(q, q8)
	t := newTopK(k)
	for id := 0; id < ix.N; id++ {
		t.push(id, qScale*ix.Scales[id]*float32(Dot(q8, ix.Code8(id))))
	}
	return t.results()
}
```

そして、このスコアの復元には scale を掛け戻せば良いです。

```go
t.push(id, qScale*ix.Scales[id]*float32(Dot(q8, ix.Code8(id))))
```

この式は量子化の定義をそのまま代入した形になっています。

$$
\sum_i d_i q_i
\approx \sum_i \bigl(\mathrm{code8}_i \cdot \mathrm{dScale}\bigr)\bigl(\mathrm{q8}_i \cdot \mathrm{qScale}\bigr)
= \mathrm{dScale} \cdot \mathrm{qScale} \cdot \sum_i \mathrm{code8}_i \cdot \mathrm{q8}_i
$$

(ここで $d_i$ は DB ベクトルの、$q_i$ はクエリの元の float32 の要素で、$\mathrm{code8}_i$ と $\mathrm{q8}_i$ がそれぞれを量子化した int8、$\mathrm{dScale}$ と $\mathrm{qScale}$ がその scale になります)

この実装で bench を回すと、以下の結果が得られました。内積単体は 371.9 [ns] から 31.38 [ns] で **11.85 倍**。float32 の SIMD 内積より高速化されています。全探索は 9.826 [ms] から 3.965 [ms] で **2.48 倍**になりました！

```
BenchmarkDotInt8Naive-4     6374116        371.9 ns/op
BenchmarkDotInt8SIMD-4     75458914         31.38 ns/op
BenchmarkSearchInt8-4           604    3965408 ns/op    9683.75 MB/s    2.000 AI(flop/byte)    19.37 Gop/s    38.40 MB/query
```

また、量子化を施したため精度も確認します。上位10件の正しさを確認するための Recall@10 を採用しており、 `Recall@10 = 0.948` でした。これは、float32 の上位 10 件を正解としたとき、平均 9.5 件が一致するという意味であり、問題なさそうです。

Stage 3 では、次なる高速化として、int8 でも 10 万件 * 384 [byte] = 38.4 [MB] で、L3 キャッシュに全て乗らないという課題に取り組みます。

### Stage 3: バイナリ量子化でキャッシュに乗せる

現在は int8 を載せているため 1 [byte] ですが、各要素を正負の符号を表す 1 [bit] だけにします。すると、int8 の 1/8 であり、4.8 [MB] 程度に抑えられるため、L3 キャッシュに全て乗せられるでしょう。

とはいえ、本当に正負の符号だけで検索できるのでしょうか？調べてみると、答えは、精度が落ちるものの検索は可能らしいです。なぜなら、検索のベクトル演算において「意味が近い = 向きが近い」でを意味し、各軸について正側か負側かを記録する符号は向きを表現する指標になっていると考えられるためです。

ここで嬉しいのが、ベクトルの内積計算の代わりにビットの一致のみを確認すれば良くなったことです。したがって、元々 SIMD で高速化していた部分はハミング距離を確認すれば良くなり、ハミング距離は XOR をして各ビットを数え上げるだけで良くなるので高速化が期待できます。この考えの起源は Charikar の [Similarity estimation techniques from rounding algorithms](https://dl.acm.org/doi/10.1145/509907.509965)（STOC 2002, pp.380-388）で、後に SimHash という通称で広まった論文とのことです。

今回の実装は以下の通りです。

```go
func Quantize(v []float32, out []uint64) {
    for i := range out {
        out[i] = 0
    }
    for i, x := range v {
        if x > 0 {
            out[i/64] |= 1 << (i % 64)
        }
    }
}

func Hamming(a, b []uint64) int {
    var d int
    for i := range a {
        d += bits.OnesCount64(a[i] ^ b[i]) // この処理はスカラの POPCNT 命令 1 個にコンパイルされ、1 命令で 64 次元分を数えることが可能
    }
    return d
}

func (ix *Index) Code(id int) []uint64 {
    return ix.Codes[id*ix.Words : (id+1)*ix.Words]
}
```

この実装で bench を回すと、以下の結果が得られました。同じ実行内の比較で 37.579 [ms] から 0.899 [ms]、**41.8 倍**です 👀

```
BenchmarkSearchNaive-4              63    37578911 ns/op    4087.40 MB/s    0.5000 AI    2.044 GFLOP/s    153.6 MB/query
BenchmarkSearchBinary-4           2318      899242 ns/op    5337.83 MB/s                                   4.800 MB/query
```

しかし、`Recall@10 = 0.180` と81%程度低下していました。そこで、次の Stage では精度を保ちつつ軽量にすませる方法にトライします。

### Stage 4: float32 SIMD で rerank して精度を戻す

精度を改善する策として、次のようなStage3 と SIMD のハイブリッドアプローチをとります。

1. 1 bit のハミング距離で 10 万件すべてを比べ、上位 100 件（返したい 10 件の 10 倍）に絞る
2. その 100 件だけ float32 のベクトルを読み、Stage 1 の SIMD 内積で採点し直して上位 10 件を返す

このアプローチによって、2. で利用するメモリを当初の 1/1000 に抑えられます。

実際に、この実装で bench を回すと以下の結果が得られました。結果は、0.899 [ms] から 0.969 [ms] で、増加は 0.07 [ms] だけでした。

```
BenchmarkSearchBinary-4              2318     899242 ns/op
BenchmarkSearchBinaryRerank-4        2530     969063 ns/op
```

また、`Recall@10 = 0.868` となっており良さそうです。ただし、この `0.868` は 2 万件で測った数字です。


## 追加で気になったポイント

### 1. `simd` と `simd/archsimd` のどちらを使うのか

結論を先に書いておくと、使いたい機能が `simd` パッケージ側にあるなら `simd` パッケージ、そうでなければ `simd/archsimd` だと思います(それはそう)。
というのも、[#78902](https://github.com/golang/go/issues/78902) が `ToArch()` という逃げ道を用意していて、式の途中で archsimd の型に落とせるんですよね。実際に、現行 API にも `func (x Float32s) ToArch() any` が存在するので、「ポータブルで書いて、足りない箇所だけ `ToArch()` で落とす」が公式に想定された使い方だと考えています。

#### コード生成と互換性の調査

一応互換性について気になったので実装を確認したのですが、ポータブル版は幅ごとに関数が複製される実装になっていました。

```
vec.DotPortable(SB)           ← 幅を見て振り分けるだけのスタブ
vec.DotPortable@simd0(SB)     ← エミュレーション版
vec.DotPortable@simd128(SB)   ← 128 bit 専用
vec.DotPortable@simd256(SB)   ← 256 bit 専用
vec.DotPortable@simd512(SB)   ← 512 bit 専用
```

スタブは `simd.maxVectorSize` を読んで該当版を `CALL` するだけなので、各複製の中では `acc.Len()` はコンパイル時定数になり、コストはグローバル 1 回読みと数回の比較と 1 回の CALL だけで、ループ本体は archsimd 版と同じ命令になるはずです。詳細実装は以下の `midway` パッケージが担っていました:

- 複製名の `@simd<N>` を組み立てている箇所: [`cmd/compile/internal/midway/rewrite.go`](https://github.com/golang/go/blob/go1.27.1/src/cmd/compile/internal/midway/rewrite.go)（`fmt.Sprintf("%s@simd%d", ...)`）。同じ関数がディスパッチ用の `switch` 文も生成していました。
- どの幅で複製するかを決めている箇所: [`cmd/compile/internal/midway/midway.go`](https://github.com/golang/go/blob/go1.27.1/src/cmd/compile/internal/midway/midway.go) の `rewriteSizes()`。**amd64 は `{0, 128, 256, 512}`、arm64 と wasm は `{0, 128}`** を返します。上記で挙げた 4 つの複製がちょうどこれに該当します。
- ディスパッチの判定に使う変数: [`src/simd/midway_common.go`](https://github.com/golang/go/blob/go1.27.1/src/simd/midway_common.go) の `maxVectorSize`

また、今回の教材で複製される側のコードは[`internal/vec/dot_portable.go`](https://github.com/po3rin/gocon2026-simd-search/blob/main/internal/vec/dot_portable.go)で、次のコマンドでシンボルが確認できます。

```sh
GOEXPERIMENT=simd GOARCH=amd64 go build -gcflags=-S ./internal/vec 2>&1 | grep 'TEXT.*DotPortable'
```

そして、ポータブル版には現在以下の機能が存在していませんでした:

- int8 → int16 の幅拡張（`Int8x16.ExtendToInt16()`）がない
- int8 のペア積和（`Int16x16.DotProductPairs()`）がない
- popcount（`Uint64x4.OnesCount()`）がない

また、今回の演算でも触れた通り、乗算において桁溢れの可能性がありコンパイルは通るのに結果が壊れる危険性があると思います。

#### これらの機能は将来追加されるのか

ポータブル版の提案スレッド [#78902](https://github.com/golang/go/issues/78902) に、何を入れるかの基準が明記されているため、それを基準に考えてみます。

> In this version, the supported vector methods are those in the **intersection of the wasm SIMD API and the current amd64 SIMD API** … The portable API will be expanded over time by various architecture-specific APIs with emulations to fill in the intersection.

「wasm と amd64 の共通部分」が基準ということなので、各アーキの `archsimd` を実際に引いて共通部分を調べてみました。

```sh
GOEXPERIMENT=simd GOARCH=arm64 go doc simd/archsimd.Int8x16
GOEXPERIMENT=simd GOOS=js GOARCH=wasm go doc simd/archsimd.Int8x16
```

| 操作 | amd64 | arm64 | wasm |
| --- | --- | --- | --- |
| 幅拡張 `ExtendLo8ToInt16`（int8→int16・同幅） | o | o | o |
| popcount `OnesCount`（バイト単位） | o | △ | o |
| ペア積和 `DotProductPairs` | o | x | x |

ペア積和はそもそも arm64 等にはないので難しそうです。幅拡張と popcount は 3 アーキすべてにありましたが、そのままだと導入は厳しいと思うので形は変えて導入されるのではないかなと思います。

- `ExtendToInt16()` は `Int8x16`（128 [bit]）から `Int16x16`（256 [bit]）を返すため、レジスタ幅が倍になる操作です。#78902 によると、この形はポータブルには持ち込めないので、入るとしたら同じ幅でレーン数が半分にするなどの制限が入ることになるのかな、と思います
- popcount も、64 [bit] 単位（`Uint64x4.OnesCount`、AVX-512 VPOPCNTDQ）は arm64 にありませんでした

### 2. ClearAVXUpperBits を自分で呼ぶ必要がある

`archsimd.ClearAVXUpperBits()` は VZEROUPPER という命令に対応する関数で、SIMD で使ったレジスタの上位 128 [bit] をゼロに掃除するためのものです。これを置かないと、SIMD からスカラの計算に戻る境界で Intel 機が大きく遅くなります。教材の付録には、これ 1 命令の有無で内積単体が 167.4 [ns] と 23.4 [ns]（7.1 倍）変わった実測が載っています。YMM レジスタの上位 128 [bit] に値が残った状態（dirty）でレガシー SSE 命令を実行すると、命令ごとに偽の依存とマージ μop が挿入されて蓄積するためです。

そして、Go 1.27 のコンパイラはこれを自動挿入しません。 [golang/go#80835](https://github.com/golang/go/issues/80835) に「archsimd の intrinsics を使う関数にレガシー SSE 符号化が出力され、AVX-SSE 遷移ペナルティを起こす」として上がっているが、執筆時点で open のままなので、自分で呼ぶ必要があります。

- [#79984](https://github.com/golang/go/issues/79984)（open）: simd の演算がループ不変式の巻き上げ（hoisting）の対象として扱われていない
- [#78138](https://github.com/golang/go/issues/78138)（open）: `VPSRLW` の定数畳み込み規則が無く、`NotEqual` のコード生成も最適でない

### 3. int8 から int16 を経由する理由

型の流れで `ExtendToInt16` を挟む部分で、int8 から直接 int32 にできないのかと思いましたが、調べると**AVX2 には int8 の積和命令が存在しませんでした**。`Int8x16` のメソッドは `Mul`（int8 → int8、溢れる）と `MulSign` だけで、`DotProduct*` が 1 つもありません。積和を持っているのは int16 のほうなので、**int16 にするのは「積和命令に到達するため」**でした。直接 int32 にする `Int8x16.ExtendToInt32()` は**存在します**が、戻り型が `Int32x16` で 512 [bit] になります。AVX-512 が必要なので、AVX2 機では使えません。

ここに気づくと納得が早い。`256 [bit] / 16 要素 = 16 bit/要素` なので、int8（128 [bit]）では積が入らず、int32（512 [bit]）では収まりません。**「1 命令で 16 要素を処理する」と決めた時点で、中間表現は int16 以外にありえません。**

なお arm64 は経路が逆でした。`Int8x16.MulWidenLo`（SMULL、int8 * int8 → int16 の幅拡張つき掛け算）が**ある**代わりにペア積和が**ない**ので、「掛けながら広げて、そのあともう一度広げる」3 段構成になります。amd64 側に `MulWiden*` は 1 つもありません。

## おわりに

本記事では、参加した Go Conference のうち、イベント中に時間が取れなかったワークショップの内容について Recap をしました。普段は触らない分野だったため、「Go Far, Go Together」の通り、Go 単体から派生して別分野についても調べるきっかけとなったので良かったと思います。

本ワークショップのオーガナイザである [ponさん](https://x.com/po3rin)、ならびに関係者のみなさま、良い機会を提供してくださりありがとうございました。

## 参考

教材とセッション:

- po3rin, [gocon2026-simd-search](https://github.com/po3rin/gocon2026-simd-search)（教材リポジトリ, 参照 2026-09-12）
- Go Conference 2026, [Go × SIMDで高速化するベクトル検索 ~ ルーフラインモデルでSIMDが効く境界を探れ！ ~](https://gocon.jp/2026/timetable/1264338/)（Workshop B, 参照 2026-09-12）

ルーフラインモデル:

- Samuel Webb Williams, Andrew Waterman, David A. Patterson, [Roofline: An Insightful Visual Performance Model for Floating-Point Programs and Multicore Architectures](https://escholarship.org/content/qt5tz795vq/qt5tz795vq.pdf), Technical Report UCB/EECS-2008-134, EECS Department, University of California, Berkeley, 2008（初出）
- Samuel Williams, Andrew Waterman, David Patterson, [Roofline: An Insightful Visual Performance Model for Multicore Architectures](https://dl.acm.org/doi/10.1145/1498765.1498785), Communications of the ACM 52(4), pp.65-76, 2009. DOI: `10.1145/1498765.1498785`（公刊版。通常はこちらを引用する）
- Aleksandar Ilic, Frederico Pratas, Leonel Sousa, [Cache-aware Roofline model: Upgrading the loft](https://doi.org/10.1109/L-CA.2013.6), IEEE Computer Architecture Letters 13(1), pp.21-24, 2014. DOI: `10.1109/L-CA.2013.6`（本記事で「キャッシュに収まると比べる相手が変わる」と書いた部分を、屋根を複数本に増やして扱うモデル）
- John D. McCalpin, Memory Bandwidth and Machine Balance in Current High Performance Computers, IEEE Computer Society Technical Committee on Computer Architecture (TCCA) Newsletter, pp.19-25, 1995（メモリ帯域の標準ベンチ。実装は [STREAM](https://www.cs.virginia.edu/stream/), 参照 2026-09-12）
- NERSC, [Roofline Performance Model](https://docs.nersc.gov/tools/performance/roofline/)（参照 2026-09-12。Empirical Roofline Toolkit を使った天井の実測手順も載っている）
- Intel, [Intel Advisor Roofline](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-advisor-roofline.html)（参照 2026-09-12。ルーフラインの自動作図）

Go の SIMD:

- The Go Authors, [Go 1.27 Release Notes: simd](https://go.dev/doc/go1.27#simd)（参照 2026-09-12）
- The Go Authors, [simd](https://pkg.go.dev/simd)（ポータブル, 参照 2026-09-12） / [simd/archsimd](https://pkg.go.dev/simd/archsimd)（アーキ固有, 参照 2026-09-12）
- golang/go#73787, [simd/archsimd: architecture-specific SIMD intrinsics under a GOEXPERIMENT](https://github.com/golang/go/issues/73787)（open, 参照 2026-09-12。導入の時系列と設計判断がここに集まっている）
- golang/go#78902, [simd: architecture and vector-size agnostic SIMD intrinsics under a GOEXPERIMENT](https://github.com/golang/go/issues/78902)（open, 参照 2026-09-12。ポータブルな `simd` パッケージ）
- golang/go#76473, [simd: rename low-level SIMD package to simd/archsimd (GOEXPERIMENT)](https://github.com/golang/go/issues/76473)（closed / completed, 参照 2026-09-12。パッケージ名の変更）
- golang/go#35307, [Proposal: Go2: Vector Basic Type - Similar to []T but with enhancements](https://github.com/golang/go/issues/35307)（closed / completed, 参照 2026-09-12。先行する提案）
- golang/go#53171, [proposal: add package for using SIMD instructions](https://github.com/golang/go/issues/53171)（closed / completed, 参照 2026-09-12。先行する提案）
- golang/go#67520, [proposal: simd: new package for intrinsics](https://github.com/golang/go/issues/67520)（closed / not planned, 参照 2026-09-12。先行する提案）
- golang/go#76175, [proposal: simd: CPU feature vet check under GOEXPERIMENT=simd](https://github.com/golang/go/issues/76175)（open, 参照 2026-09-12。CPU 機能チェック漏れを vet で検査する提案）
- golang/go#81405, [simd/archsimd: Float32x4 and Float64x2 Abs and Neg crash with SIGILL on AVX-only CPUs](https://github.com/golang/go/issues/81405)（open, 参照 2026-09-12。SIGILL の実例）
- golang/go#80835, [cmd/compile: legacy SSE encodings emitted in functions using simd/archsimd intrinsics cause AVX-SSE transition penalties](https://github.com/golang/go/issues/80835)（open, 参照 2026-09-12。AVX-SSE 遷移ペナルティ）
- golang/go#76969, [simd/archsimd: Generated code doesn't use Y15 but spills register to stack](https://github.com/golang/go/issues/76969)（closed / not planned, 参照 2026-09-12。register spill）
- golang/go#78753, [cmd/compile: AMD64 AVX-512 register allocation is limited to lower 16 ZMM registers](https://github.com/golang/go/issues/78753)（closed / completed, 参照 2026-09-12。AVX-512 の ZMM 割り当て）
- golang/go#78979, [proposal: simd/archsimd: enable AMD64 architecture-specific SIMD by default](https://github.com/golang/go/issues/78979)（open, 参照 2026-09-12。AMD64 をデフォルト有効にする提案）
- golang/go#79781, [simd/archsimd: support ARM64 SVE SIMD intrinsics under a GOEXPERIMENT](https://github.com/golang/go/issues/79781)（open, 参照 2026-09-12。ARM64 SVE）
- golang/go#79413, [crypto: replace handwritten assembly with native Go SIMD](https://github.com/golang/go/issues/79413)（open, 参照 2026-09-12。`crypto` のアセンブリ置き換え）
- Agner Fog, [The microarchitecture of Intel, AMD and VIA CPUs](https://www.agner.org/optimize/microarchitecture.pdf)（参照 2026-09-12。AVX / SSE 遷移ペナルティ）
- Agner Fog, [Instruction tables](https://www.agner.org/optimize/instruction_tables.pdf)（参照 2026-09-12。命令のレイテンシとスループット）
- [uops.info](https://uops.info/)（参照 2026-09-12。命令のレイテンシとスループットの実測データベース）

ベクトル検索と量子化:

- Moses S. Charikar, [Similarity estimation techniques from rounding algorithms](https://dl.acm.org/doi/10.1145/509907.509965), Proceedings of the 34th Annual ACM Symposium on Theory of Computing (STOC '02), pp.380-388, 2002. DOI: `10.1145/509907.509965`（符号 1 bit がコサイン類似度を保存する根拠。後に SimHash の通称で広まった）
- Jianyang Gao, Cheng Long, [RaBitQ: Quantizing High-Dimensional Vectors with a Theoretical Error Bound for Approximate Nearest Neighbor Search](https://dl.acm.org/doi/10.1145/3654970), Proceedings of the ACM on Management of Data 2(3), pp.1-27, 2024. DOI: `10.1145/3654970`（SIGMOD 2024。ランダム回転を入れた 1 bit 量子化）
- Yu. A. Malkov, D. A. Yashunin, [Efficient and Robust Approximate Nearest Neighbor Search Using Hierarchical Navigable Small World Graphs](https://doi.org/10.1109/TPAMI.2018.2889473), IEEE Transactions on Pattern Analysis and Machine Intelligence 42(4), pp.824-836, 2020. DOI: `10.1109/TPAMI.2018.2889473`（HNSW。プレプリントは [arXiv:1603.09320](https://arxiv.org/abs/1603.09320)）
- Facebook Research, [Faiss indexes](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes)（参照 2026-09-12。近似最近傍探索の索引の一覧）
