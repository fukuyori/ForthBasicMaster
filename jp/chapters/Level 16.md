# Lev 16 ー スコープと辞書構造
FORTHは、見た目こそシンプルだが、その内部には「辞書（dictionary）」という強力な構造を持っている。ここには定義したすべてのワード（命令や変数、定数など）が登録され、この辞書の探索順序と構築状態が、FORTHにおける「スコープ（有効範囲）」を決定する。

本章では、gforthにおけるスコープの考え方を整理し、(1) 定義スコープ、(2) ローカル変数スコープ、(3) 名前空間スコープ、の三層構造として理解する。


## スコープとは何か ― FORTHの辞書構造

FORTHでは、C言語やPythonのようなブロック単位のスコープは存在しない。代わりに、**「定義が辞書に登録されているかどうか」** がワードの可視性を決める。

辞書はLIFO（後入れ先出し）のスタックのように動作し、新しいワードが定義されると上に積まれ、古い定義は下に隠れる。この仕組みが、FORTHにおけるスコープの本質である。

```forth
: A ." first" ;
: A ." second" ;
A   \ → "second"
```

上の例では、2つ目の `A` が辞書の上に積まれたため、古い定義は見えなくなる。このように、新しい定義は既存の定義を「シャドウ（shadow）」する。

## 定義スコープとMARKER

辞書の内容は`FORGET`あるいは`MARKER`を使って巻き戻すことができる。これにより、定義スコープ（定義の有効範囲）を人工的に管理できる。

```forth
MARKER SAFE
: TEMP ." Hello" ;
SAFE        \ ここでTEMPが定義される前の状態に戻る
TEMP        \ → エラー（辞書から削除されている）
```

`MARKER`は「この時点の辞書状態を保存しておく」ためのワードである。以後の定義はすべて、そのマーカー以前の状態に戻せる。実験的なワード定義や、一時的に試すコードを書くときに極めて便利だ。

## gforthのローカル変数スコープ ― { } 構文

古典的なFORTHでは、スタック操作によってすべての値を渡す。だがgforthには、可読性を高めるために**ローカル変数構文 `{ ... }`** が導入されている。この中で宣言された変数は、そのワード定義の内部でのみ有効である。すなわち、これは**静的スコープ（lexical scope）** を持つ仕組みである。

```forth
: SUM3 { a b c -- sum }
   a b + c + ;
5 6 7 SUM3 .   \ → 18
```

ここで `a`, `b`, `c` は `SUM3` の外からはアクセスできない。FORTHにしては珍しく、ブロック的なスコープを形成している。

ローカル変数の初期化は、通常の変数と同じく実引数で行われ、宣言の右側にある「-- sum」はスタック効果のドキュメントを兼ねている。この記法により、複雑なスタック操作を避けつつ、計算ロジックを明快に記述できる。

```forth:
: FIB { n -- f }
    n 2 < IF n EXIT THEN
    0 1
    n 1- 0 DO \ F(n) を計算するために (n-1) 回のループ
        SWAP OVER +
    LOOP
    SWAP DROP ;
```

このように、`{}`を使うことで、スタック上の煩雑なデータ操作を局所化し、安全な「一時領域（スコープ付き変数）」を確保できる。

```python:Python版
def fib(n: int) -> int:
    a, b = 1, 0
    for _ in range(n):
        a, b = b, a+b
    return b
```

## 名前空間スコープ ― VOCABULARY

FORTHのもう一つのスコープ機構は「ワードリスト（wordlist）」である。これは、C++でいう「名前空間(namespace)」に近い考え方である。

#### VOCABULARYによる分離

```forth
VOCABULARY MY-WORDS      \ MY-WORDSという名前のボキャブラリを作成
ONLY FORTH ALSO MY-WORDS
DEFINITIONS              \ MY-WORDSにワードを定義する
: HELLO ( -- ) ." Hello, My Words!" ;
```

この時、`ORDER`で状態を表示させると、

```
ORDER  \ → MY-WORDS Forth Root     MY-WORDS
```

となっている。`[MY-WORDS Forth Root]` は検索順序スタックで、最後の`[My-WORDS]`はコンテキスト（現在の定義場所）を示している。この状態でワードを定義すると、MY-WORDSのボキャブラリに定義が行われる。

検索順序の操作には、`ONLY` `ALSO` を使う。`Forth`だけにするには、`ONLY FORTH`　または `FORTH` と入力する。

```
ONLY FORTH   \ 標準（FORTH）のみ
HELLO        \ → エラー Undefined word
ORDER        \ → Forth Root      MY-WORDS
```

検索順序から`MY-WORDS`は外れているが、コンテキストは`MY-WORDS`のままなので、このままワード定義を行うと`MY-WORDS`のボキャブラリに追加されてしまう。この状態で`DEFINITIONS`（または`Forth DEFINITIONS`）
を実行すれば、コンテキストは`Forth`に設定される。`DEFINITIONS`は以降のワード登録先を切り替えることが出来る。

```
ORDER        \ → Forth Root      MY-WORDS
DEFINITIONS  \ Forthにコンテキストを変更
ORDER        \ → Forth Root      FORTH
```

`MY-WORDS`のボキャブラリを使用する場合は、`ALSO MY-WORDS` で検索順序スタックに追加する。

```
ONLY FORTH ALSO MY-WORDS \  標準（FORTH）とMY-WORDS
HELLO                    \ → Hello, My Words!
```

## SET-ORDER / SET-CURRENT

`GET-ORDER`は、検索順序をデータスタックにコピーする。現在の検索順序には n 個のエントリがあり、最後に検索されるワードリストidから最初に検索されるワードリストidがスタックされ、一番上にワードリストidの数がスタックされる。

```
ORDER                   \ → Forth Forth Root   Forth
VOCABULARY W1
W1                      \ W1を検索順序スタックへ追加
DEFINITIONS             \ コンテキストをW1に変更
: HELLO ( -- ) ." Hello, World!" ;
GET-ORDER               \ 検索順序をスタックに保存

ONLY FORTH
ORDER                   \ → Forth Forth Root   W1

SET-ORDER               \ スタックに保存された検索順序を適用
HELLO                   \ → Hello, WOrld!
ORDER                   \ → W1 Forth Forth Root   W1
```

現在のコンテキストの取得と変更をワードリストID（`wid`）を使って行うことが出来る。

```
ORDER            \ → Forth Forth Root   Forth
GET-CURRENT      \ 現在のwidをスタックへ
VOCABULARY W1
W1               \ W1を検索順序スタックへ追加
DEFINITIONS      \ コンテキストをW1に変更
ORDER            \ → W1 Forth Root       W1
SET-CURRENT      \ スタックにあるwidを使ってコンテキストを設定
ORDER            \ → W1 Forth Root       Forth
```

現在のコンテキストを使って、検索順序スタックに追加することが出来る。

```
VOCABULARY W1
W1 DEFINITIONS
ORDER             \ → W1 Forth Root       W1
GET-CURRENT       \ 現在のコンテキストのwidを取得
FORTH DEFINITIONS \ コンテキストをForthに変更
ORDER             \ → Forth Forth Root   Forth

>ORDER           \ スタック上のW1のコンテキストを追加
ORDER            \ → W1 Forth Forth Root       FORTH
```

## 現代的なスコープ制御 ー WORDLIST と SET-CURRENT

`WORDLIST` は、`wid` を定数（`CONSTANT`）にセットする。gforth では `VOCABULARY` よりも、より柔軟な `WORDLIST` を使うことが推奨されている。

```forth
WORDLIST CONSTANT UTIL    \ 空のワードリストを作成し、widをUTILにセットする
UTIL SET-CURRENT          \ コンテキストをUTILに変更
ORDER                     \ → Forth Forth Root    UTIL

UTIL >ORDER               \ 検索順序スタックにUTILを追加
ORDER                     \ → UTIL Forth Forth Root    UTIL
```

このように、`WORDLIST` を使うと、検索順序スタックやコンテキストの変更が簡単になる。

:::note alert
`WORDLIST`で定義した定数に`ONLY`や`ALSO`は使用できない。
:::

## 実践例：スコープを使ったモジュール化

```forth
WORDLIST CONSTANT MATH
MATH SET-CURRENT

: ADD2 ( n -- n' ) 2 + ;
: SQUARE ( n -- n2 ) DUP * ;

DEFINITIONS

WORDLIST CONSTANT IO
IO SET-CURRENT

: SHOW ( n -- ) ." Result: " . CR ;

DEFINITIONS

MATH >ORDER                   \ 検索順序スタックにMATHを追加
IO >ORDER                     \ 検索順序スタックにIOを追加
: MAIN { x -- } x ADD2 SQUARE SHOW ;
3 MAIN                        \ → Result: 25

ONLY FORTH                    \ 検索順序スタックをForthだけに
5 MAINord                     \ → error: Undefined word
```

ここでは、MATHとIOを別々のスコープ（ワードリスト）として管理している。`MAIN` は両方のスコープを同時に参照している。こうした設計により、FORTHでも「モジュール指向プログラミング」が実現できる。

## スコープと再定義（シャドウイング）

FORTHでは同名ワードを再定義すると、新しい定義が上位スコープを覆う。

```forth
WORDLIST CONSTANT W1
WORDLIST CONSTANT W2

W1 SET-CURRENT
: PRINT ." W1" ;

W2 SET-CURRENT
: PRINT ." W2" ;

W1 >ORDER W2 >ORDER
PRINT               \ → W2

ONLY FORTH
W2 >ORDER W1 >ORDER
PRINT               \ → W1
```

つまり、辞書順序がスコープの優先順位を決める。探索順序を入れ替えると、同名ワードの解釈結果も変わる。

