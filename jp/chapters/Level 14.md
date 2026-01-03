# Lev 14 ー VARIABLE・CONSTANT・VALUE・文字列・PAD
FORTHは「すべてがワードである」という思想のもと、**データ保持の仕組みさえもワード化** している。これにより、C言語の`int x;`やPythonの`x = 10`に相当する操作も、FORTHでは辞書登録を通じて実現される。本章では、`VARIABLE`、`CONSTANT`、`VALUE`の3つの代表的な手段を比較しながら、「どう保持し、どう変更するか」という観点でFORTH的データ管理を整理する。

## VARIABLE ― 書き換え可能な記憶場所

`VARIABLE`は、最も基本的な“可変セル”を定義するワードである。宣言すると、そのワード名は**アドレス**を返す。値を代入・取得するには、`!`（ストア）と`@`（フェッチ）を用いる。

```forth
VARIABLE COUNT   \ 可変領域COUNTを作成
10 COUNT !       \ COUNTに10を書き込む
COUNT @ .        \ COUNTの内容を表示 → 10
```

ここでの動作は以下のように理解できる。

* `VARIABLE COUNT` … COUNTという名前を辞書に登録。空のセル（1セル=1整数分）を確保。
* `COUNT` … そのセルのアドレスを返す。
* `10 COUNT !` … COUNTのアドレスに10を格納。
* `COUNT @` … COUNTのアドレスから値を読み出す。

つまり、`VARIABLE`は「変数名＝アドレスを返すワード」として機能する。FORTHでは**値ではなくアドレスが流れる**点を理解することが重要である。

## CONSTANT ― 一度決めた値を固定化する

`CONSTANT`は、定義時にスタック上の値を取り、それを保持する定数を作る。

```forth
123 CONSTANT LIMIT
LIMIT .      \ → 123
```

このとき、`LIMIT`は実行されると「123」という値そのものをスタックに積む。`VARIABLE`がアドレスを返すのに対し、`CONSTANT`は**値を返す**という違いがある。

例えば次のような比較ができる。

| 種別 | 実行結果 | 動作 | 再代入 |
|--|--|--|:-:|
| `VARIABLE COUNT` | COUNT → アドレス | 可変データ領域を返す | 可 |
| `CONSTANT LIMIT` | LIMIT → 値    | 定数値を返す     | 不可 |

`CONSTANT`は再代入できない。変更したい場合は新しい定数を再定義する。
したがって、物理定数・設定固定値・配列長など、「プログラム実行中に変わらない値」を表すのに最適である。

## VALUE ― 再代入可能な定数

FORTHでは「再定義可能な定数」という概念を`VALUE`で表す。定義時にはスタックの値を初期値とし、後から`TO`ワードでその値を変更できる。

```forth
10 VALUE SPEED
SPEED .        \ → 10
20 TO SPEED
SPEED .        \ → 20
```

`VALUE`は`CONSTANT`に似ているが、内部的には書き換え可能な領域を持ち、`TO`を使うことで新しい値を格納する。`TO`の使い方はやや独特で、**左辺がワード、右辺が値**である。

```
新しい値 TO VALUE名
```

つまり「値をスタックに積んでから、`TO`によって格納先を指定する」構文である。C言語で言えば「代入演算子＝を後ろに回したような書き方」になる。

## 三者の比較

次の表に、3種のワードの性質を整理する。

| 機能        | VARIABLE     | CONSTANT | VALUE       |
| --------- | ------------ | -------- | ----------- |
| 変更可否      | 〇 (`!`で書き換え) | ×        | 〇 (`TO`で変更) |
| 実行時スタック効果 | アドレスを返す      | 値を返す     | 値を返す        |
| 初期化       | 未定義(0)       | 定義時の値    | 定義時の値       |
| 推奨用途      | 一時領域・計数器     | 固定パラメータ  | 設定値・動的閾値    |

たとえば、次のように組み合わせて使うと良い。

```forth
10 CONSTANT MAX-SPEED
5 VALUE CUR-SPEED
VARIABLE STEP

: ACCEL  ( -- )
   STEP @ CUR-SPEED + TO CUR-SPEED
   CUR-SPEED MAX-SPEED > IF
      MAX-SPEED TO CUR-SPEED
   THEN ;
```

```:実行例
3 STEP !
ACCEL CUR-SPEED . \ → 8
ACCEL CUR-SPEED . \ → 10
```

ここでは、`MAX-SPEED`は変わらない定数、`CUR-SPEED`は再設定可能なパラメータ、`STEP`は逐次変化を管理する変数として役割分担している。

## VALUEとTOの詳細動作

`VALUE`を定義すると、辞書に「実行時に値を積む動作」と「書き換え用のTOリンク」が登録される。`TO`はコンパイルモードでは特別な挙動を取り、直後に続く`VALUE`名を解決して値を変更する。以下のように、即時実行時でもTOは正しく動作する。

```forth
0 VALUE SPEED
: TEST ( n -- ) TO SPEED ;
30 TEST
SPEED .   \ → 30
```

FORTHの「実行／コンパイルモード」切替機構により、`TO`は**文脈に応じて意味が変化する即時ワード（IMMEDIATE）** として実装されている。これが、FORTHの柔軟さと低レベル制御力の一端である。

| モード          | 動作                       |
| ------------ | ------------------------ |
| **実行モード**    | 即座に指定 VALUE に書き込む        |
| **コンパイルモード** | 書き込みコードを後で実行するようにコンパイルする |

## VALUEとVARIABLEの実装上の違い

|観点|VARIABLE|VALUE|
|----------------------|------------------------------------|------------------------------------------------------|
|定義時の本質|`CREATE`で1セル確保し、初期値0を格納|`CREATE`で1セル確保し、初期値を格納し、さらに`DOES>`で実行時意味を付与|
|実行時意味（interpretで名前を実行）|PFA（データ部のアドレス）をスタックへ積む（=「場所」を返す）|格納セルの中身（値）をスタックへ積む（=「値」を返す）|
|読み出し|`name(--addr)`→`@`で読む|`name(--n)`（即値が積まれる）|
|書き込み|`name(addr)`に対して`!`を使う|`TOname`で書き込む（`TO`が「次の語＝VALUE名」を読んでそのPFAに`!`する）|
|内部構造の典型|`CREATE name 0 ,`|`CREATE name init , DOES>( -- n ) @`|
|`>BODY`の意味|`>BODY`はそのまま書き込み先（=`!`する先）|`>BODY`はVALUEの“実体セル”のアドレス（=`TO`が`!`する先）|

`COUNT`（VARIABLE）は単純にメモリアドレスを返す構造を持つが、`SPEED`（VALUE）は`DOES>`構文で作られた「読み出し＋書き換え可能な振る舞い」を持つ。つまり、`VALUE`は`CREATE … DOES>`を内部的に利用している。FORTHでは、**構文の違い＝辞書登録時の振る舞いの違い**に過ぎない。

## 定数と変数を使い分ける設計思考

FORTHにおけるデータ保持は、型よりも**意味と用途**で区別される。言い換えれば、「どんな意図で変わる（変わらない）のか」を考えることが設計である。

| 状況                       | 適切な手段      |
| ------------------------ | ---------- |
| 環境依存・変更不可の値（π, e, 配列サイズ） | `CONSTANT` |
| 状態やカウンタなど、頻繁に更新される値      | `VARIABLE` |
| 実行中に動的調整する設定値（閾値・係数）     | `VALUE`    |

こうした設計指針を意識することで、FORTHプログラム全体が「制御と構造を分離した柔軟なシステム」に変化する。

## 応用例：温度センサ制御モデル

以下は、`VALUE`と`VARIABLE`を組み合わせた制御例である。

```forth
20 VALUE TEMP          \ 現在温度
25 CONSTANT TARGET     \ 目標温度
VARIABLE STEP          \ 温度変化量

: READ-SENSOR ( -- f )  \ 仮のセンサ入力
   TEMP ;

: CONTROL ( -- )
   READ-SENSOR TARGET < IF
      STEP @ TEMP + TO TEMP
   ELSE
      STEP @ NEGATE TEMP + TO TEMP
   THEN
   ." TEMP=" TEMP . CR ;
```

```:実行例
2 STEP !
CONTROL \ → TEMP=22
CONTROL \ → TEMP=24
CONTROL \ → TEMP=26
CONTROL \ → TEMP=24
```

ここでは、`TEMP`がVALUEとして動的に更新され、`STEP`が調整単位、`TARGET`が固定パラメータとして機能している。FORTHでは、このように小さな構成要素を自在に組み合わせて「プロセス制御」や「反復シミュレーション」を構築できる。


## 文字列の扱い

FORTHは数値計算や制御処理を中心に設計された言語だが、実際のプログラムでは文字列の入力・出力・加工も欠かせない。他の言語のような「文字列型」は存在せず、FORTHでは アドレスと長さの組（addr len）で文字列を表現する。

もっとも基本的な文字列ワードは `S"`（エス・ダブルクォート）である。
次のように書くと：

```forth:入力
S" Hello, Forth!" TYPE
```

```:出力
Hello, Forth!
```

`S"` はコンパイル時に、引用符で囲まれた文字列を辞書内に格納する。実行時にはその文字列の**アドレスと長さ**をスタックに積む：

```
( -- addr len )
```

`TYPE` はこの `(addr len)` を受け取り、指定された長さの文字列を出力する。

このように、FORTHの文字列はポインタ操作に近く、「どこから」「何文字分」を扱うかを明示的に管理する。

`MOVE` は、FORTHにおける**メモリ転送の基本ワード**であり、C言語における `memmove()` に相当する。与えられたアドレス範囲のデータを、指定した別の領域に安全にコピーするためのものである。

```:構文
MOVE  ( addr_src addr_dst u -- )
```

ここで、

* `addr_src` はコピー元のアドレス、
* `addr_dst` はコピー先のアドレス、
* `u` は転送するバイト数である。

この命令を実行すると、`addr_src` から `addr_dst` に向けて `u` バイト分のデータが転送される。実行前のスタック状態は `( addr_src addr_dst u )` であり、実行後はスタックが空になる。コピーの方向は処理系が自動的に判断するため、コピー領域が部分的に重なっていても正しく動作する。つまり、`MOVE` は `memcpy()` ではなく `memmove()` と同等の動きをする。

```forth:入力
CREATE BUF 20 ALLOT
S" Forth" BUF SWAP MOVE
BUF 5 TYPE
```

```:出力
Forth
```

この例では、`S" Forth"` により文字列のアドレスと長さがスタックに積まれ、`BUF` を宛先として `MOVE` によりメモリが転送されている。結果として `BUF` 内に “Forth” が格納され、`TYPE` により出力される。`TUCK`を使って、長さをスタックの一番下にコピーしておけば、TYPEのために長さを数えなくていい。

```forth:入力
CREATE BUF 20 ALLOT
S" Forth" TUCK BUF SWAP MOVE
BUF SWAP TYPE
```

一方、即座に出力したい場合は `."`（ドット・ダブルクォート）を使う。

```forth:入力
." Hello World" CR
```

```:出力
Hello World
```

`."` は定義内では即時実行されるワードであり、実行時に自動的に`TYPE`を伴って文字列を出力する。そのため、`S"` のように `TYPE` を書く必要がない。ただし、`."` は**文字列を返さない**（アドレスや長さはスタックに残らない）。

`S\"`は、文字列の中にC言語のようなエスケープシーケンスを入れることが出来る。`\a` ベル、`\b` バックスペース、`\e` エスケープ、 `\f` FF、 `\n` 改行、 `\r` CR、 `\t` タブ、 `\v` 垂直タブなどがある。`S"`と`."`と同様に、`S\"`には`.\"`がある。

```
S\" Hello\tWorld!" TYPE
.\" Welcome,\nthe entire land"
```

```:出力
Hello   World!
Welcome,
the entire land
```

### String words

```
variable mystring
S" Hello, World!" mystring $! \ $!    : 文字列格納
mystring $@ type              \ $@    : 文字列取得 → Hello, World!
mystring $.                   \ $.    : 文字列表示 → Hello, World!
mystring $@len .              \ $@len : 文字列の長さ → 13
7 mystring $!len              \ $!len : 長さ操作 → Hello,
S" land" mystring $+!         \ $+!   : 文字列追加 → Hello, land
'!' mystring C$+!             \ C$+!  : 1文字追加 → Hello, land!
S" new " mystring 7 $ins      \ $ins  : 文字列挿入 → Hello, new land!
mystring 7 4 $del             \ $del  : 文字列削除 → Hello, land!
S" world" mystring 7 $over    \ $over : 上書き → Hello, world
```

String wordsで使われる変数には、2セル変数へのアドレスが入っており、2セル変数には長さが1セル目に、文字列が格納されたアドレスが2セル目に入る。

```
VARIABLE mystr                      \ 文字列構造体へのポインタを格納する変数
CREATE hab 2 CELLS ALLOT            \ 2セル分の領域確保(文字列長+アドレス用)
CREATE buf 10 CELLS ALLOT           \ 10セル分のバッファ領域確保

S" test" buf SWAP CMOVE             \ "test"をbufにコピー
buf 4 TYPE                          \ bufから4文字表示 → test

4 hab !                             \ 文字列長(4)をhabの最初のセルに格納
buf @ hab 1 CELLS + !               \ bufのアドレスをhabの2番目のセルに格納
hab mystr !                         \ hab(文字列構造体)のアドレスをmystrに格納

mystr $.                            \ mystrが指す文字列を表示 → test
```

ローカル変数でString wordsを使用するには、`W^`でvariableを使う。

```
: greet
	0 { W^ mystr }
	s" Hello, " mystr $!
	s" World!" mystr $+!
	mystr $.
;

greet  \ → Hello, World!
```

### 実行トークンから一時文字列を生成

```:
: current-dir s" dir" SYSTEM ;
' generate-text $tmp type     \ →　現在のディレクトリのファイル
```

### 文字列の分割

```:文字列を区切り文字で2つに分割（$split）
S" name=value" '=' $split
\ スタック: S" name" S" value"
```

```:文字列を区切り文字で分割（$iter）
: print-part ( addr u -- ) type cr ;
variable mystring
S" 1234,567,890" mystring $!
mystring ',' ' print-part $iter
```

```:出力例
1234
567
890
```

### 文字列配列操作

```
variable names
s" Alice" 0 names $[]!     \ names[0]に"Alice"を格納
s" Bob" 1 names $[]!       \ names[1]に"Bob"を格納
s" Charlie" 2 names $[]!   \ names[2]に"Charlie"を格納
1 names $[]@ type          \ names[1]の値 → Bob
names $[]#                 \ 要素数 → 3
```

文字列配列のすべての要素に対して実行トークンを実行します。

```:
: print-name ( addr u -- ) type cr ;
names ' print-name $[]map
```

```:出力例
Alice
Bob
Charlie
```

全ての配列エントリを表示します。

```
names $[].
```

```:出力例
Alice
Bob
Charlie
```

配列内のすべての文字列を解放し、配列自体も開放します。

```
names $[]free
```

### ファイルから配列への読み込み

ファイルfidを行ごとに文字列配列addrに読み込みます。

```
VARIABLE lines
S" textfile.txt" R/O OPEN-FILE THROW
DUP lines $[]slurp
CLOSE-FILE THROW
lines $[].           \ ファイルの内容表示
```

指定された名前のファイルを行ごとに文字列配列に読み込みます。

```
VARIABLE lines
S" textfile.txt" lines $[]slurp-file
lines $[].           \ ファイルの内容表示
```


https://gforth.org/manual/String-words.html#String-words

## 特別な記憶領域 ― HERE と PAD

FORTHでは、ユーザー定義の`VARIABLE`や`CONSTANT`とは別に、システムが内部的に管理する“特別な記憶領域”がいくつか存在する。その中でも特に重要なのが **`HERE`** と **`PAD`** である。これらは辞書（dictionary）と作業領域（scratchpad）の境界を示し、FORTHシステムのメモリ構造を理解する鍵となる。

###  1. HERE ― 辞書の「次の空き場所」を返す

`HERE` は、**辞書（dictionary）内で次に定義されるワードの書き込み位置**を返す。

```forth
HERE .
100 ALLOT
HERE .
```

```:実行例
8041234
8041334
```

2回目の値は100バイト進んでおり、`ALLOT` が辞書ポインタを前進させたことがわかる。

gforthでは `HERE` はシステムワードであり、`HERE` を使うと、どこに新しいデータが格納されるかを確認できる。

例えば、あるソースファイルを読み込んだときにどのくらい辞書が拡張されたかを調べたい場合：

```forth
HERE
INCLUDE sample.fs \ ワード定義が書かれたサンプルファイル
HERE SWAP - .     \ → 例えば 1024
```

この結果は「`sample.fs`を読み込んだ結果、辞書が1024バイト拡張された」ことを意味する。`HERE` は、**“今どこまで定義が進んだか”を示すマーカー**として有用である。

### 2. PAD ― 一時的な作業領域（スクラッチパッド）

`PAD` は、**文字列や一時データを格納するための短期バッファ**を指す。数値の文字列化やファイル入出力の際に、内部的に頻繁に使われている。

```forth
PAD 80 ERASE    \ PAD初期化
S" Hello, PAD!" TUCK PAD SWAP MOVE \ PADに書き込み（長さをスタックに残す）
PAD SWAP TYPE   \ PAD出力 → Hello, PAD!
```

### 3. メモリ空間における位置関係（概念図）

```
+----------------------------+
| Return Stack               |
+----------------------------+
| Parameter Stack            |
|  ↑ grows downward          |
|                            |
|  ← ← ← High Memory         |
+----------------------------+
| Terminal Input Buffer (TIB)|
|      ↑ grows upward        |
+----------------------------+
| PAD  (scratch area)        | ← 一時領域
+----------------------------+
| HERE → Dictionary          | ← 新しい定義が追加される
+----------------------------+
| User Variables / System Vars|
+----------------------------+
| Low Memory (core)          |
+----------------------------+
```

このように、`PAD` は辞書とスタックの中間にあり、**文字列や一時データを保持する“作業机”のような領域**になっている。辞書が拡張すると、`PAD` の位置も自動的に上に移動する。

:::note warn
#### 使用上の注意

* `PAD` は「一時的な」領域であり、次に続く操作で警告もなく簡単に上書きされる。
  データを保持したい場合は `CREATE ... ALLOT` で独自に確保する。
* コンパイルや`FORGET`などで辞書構造が変わると、`PAD` の実際のアドレスも変動する。
* 文字列操作（`S"`、`TYPE`、`COUNT` など）でしばしば内部的に使われているため、**長時間データを保持してはいけない**。
:::

### 4. 例：数値を文字列に変換して出力する

`PAD` を使った典型的な例は、数値→文字列変換（`<#`〜`#>`）である。

```forth
12345 S>D <# #S #> TUCK PAD PLACE   \ <#（変換開始）#S（全桁変換）#>（変換終了）
PAD SWAP TYPE                            \ → 12345
```

この変換過程では、数値をASCII文字に変換しながら`PAD`上に積み上げ、最終的に `PAD COUNT TYPE` で出力している。`PAD` は FORTH における “printf の一時文字列バッファ” のような存在だ。`#S` の部分を変えると、出力フォーマットを操作することが出来る。


## 練習課題

1. `SCORE`という変数を作り、10ずつ加算するワード`ADD10`を定義せよ。
2. `PI`を3.14159として定義し、半径を入力して円の面積を求めるワードを作成せよ。
3. `THRESHOLD VALUE`を用い、入力値が閾値より大きいときだけメッセージを表示するプログラムを作成せよ。
4. `CONSTANT`と`VALUE`の違いを確認するため、両者を`TO`で再代入して挙動を比較せよ。

