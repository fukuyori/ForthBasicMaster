# Lev 15 ー ローカル変数とスコープ管理
前章では、`VARIABLE`、`CONSTANT`、`VALUE`という**グローバルなデータ保持機構**を学んだ。これらはすべて辞書に登録され、プログラム全体からアクセス可能である。しかし、特定のワード定義内だけで使う一時的な変数が欲しい場合、毎回グローバル変数を作るのは煩雑であり、名前の衝突リスクも高まる。

本章では、FORTHにおける**ローカル変数**を扱う。これは、ワード定義のスコープ内でのみ有効な変数であり、関数型プログラミングや構造化プログラミングにおける「局所変数」に相当する。Gforthの強力な拡張機能と、標準仕様との違いを理解することで、より保守性の高いコードが書けるようになる。

## なぜローカル変数が必要か

FORTHの伝統的なスタイルでは、すべての一時データをスタックで管理する。しかし、複雑な計算になると、スタック操作（`DUP`、`SWAP`、`ROT`など）が増え、コードの可読性が低下する。

例えば、次の二次方程式の解の公式を考えてみよう。

```forth
\ x = (-b ± sqrt(b^2 - 4ac)) / 2a
: QUADRATIC ( f: a b c -- f: x1 x2 )
	FROT 2 FPICK FDUP F*        \ ( b c a b^2 )
	FROT 2 FPICK F* 4e F*       \ ( b a b^2 4ac )
	F- FSQRT       		\ ( b a SQRT(b^2-4ac) )
	FROT FNEGATE FDUP 	\ ( a SQRT(b^2-4ac) -b -b)
	2 FPICK F+ 			\ ( a SQRT(b^2-4ac) b b+SQRT(b^2-4ac) )
	FSWAP FROT F-		\ ( a b+SQRT(b^2-4ac) b-SQRT(b^2-4ac) )
	2 FPICK 2e F* F/ 	\ ( a b+SQRT(b^2-4ac) b-SQRT(b^2-4ac)/2a )
	FSWAP FROT 2e F* F/
;
```

```:実行例
1e -5e 6e quadratic  \ → 2.0 3.0
1e -3e 2e quadratic  \ → 1.0 2.0
```

このコードはスタック操作が複雑で、何をしているか直感的に理解しにくい。ローカル変数を使うと、次のように書ける。

```forth
\ x = (-b ± sqrt(b^2 - 4ac)) / 2a
: QUADRATIC
 	{ F: a F: b F: c -- x1 x2 }

    b b F* 4e a F* c F* F- i \ 判別式 D = b^2 - 4ac
	FSQRT                	\ sqrt(D)
	b FNEGATE             	\ -b
	FDUP 2 FPICK F- 		\ -b + sqrt(D)
	a 2e F* F/            	\ x1 = (-b + sqrt(D)) / 2a
	FROT FROT F+            \ -b - sqrt(D)
	a 2e F* F/           	\ x2 = (-b - sqrt(D)) / 2a
;
```

係数`a`、`b`、`c`が名前付きで参照できるため、コードの意図が明確になる。

## ローカル変数の基本構文

Gforthでは、ローカル変数を波括弧`{}`を使って定義する。

```forth
{ local1 local2 -- comment }
```

または、コメント部分を省略して

```forth
{ local1 local2 }
```

と書く。この構文は、FORTHのスタックコメントと**意図的に**似せてある。実際、ローカル変数定義がスタックコメントを置き換える形で機能する。

ワードの最初に記述することで、スタック操作に苦労せずに受け取った値を使用することが出来る。また、ワードの途中でローカル変数を作成することもできる。

```
: test { a b -- c }
	10 { c }
	a b < IF
		a  b + c *
	ELSE
		a b -  c *
	THEN
;
```

```:実行例
1 2 test  \ → 30
2 1 test  \ → 10
```

### ローカル変数の順序

ローカル変数の順序は、**スタックコメントの順序と一致する**。つまり、スタックの一番下（最初に積まれた値）が`local1`に、一番上（最後に積まれた値）が最後のローカル変数に対応する。

```forth
: example ( n1 n2 n3 -- )
  { a b c }
  \ スタック状態: n1 n2 n3
  \ a ← n1 (一番下)
  \ b ← n2 (真ん中)
  \ c ← n3 (一番上)
  a . b . c . ;

10 20 30 example  \ → 10 20 30
```

## 基本的な使用例

### 最大値を求める

```forth
: max 
  { n1 n2 -- n3 }
  n1 n2 > if
    n1
  else
    n2
  endif ;

10 5 max .    \ → 10
3 8 max .     \ → 8
```

この定義では：
1. スタックから`n1`と`n2`を取得
2. ローカル変数として名前付きで参照可能
3. ワードの実行が終了すると自動的に破棄される

### 値の変更

ローカル変数は`VALUE`と同様に、`TO`で値を変更できる。

```forth
: countdown { n -- }
  begin
    n .
    n 1 - TO n
    n 0=
  until ;

5 countdown  \ → 5 4 3 2 1
```

## Gforth拡張：型指定子

Gforthでは、Standard Forthの仕様を拡張し、ローカル変数に**型を指定**することができる。これはGforth独自の機能であり、標準的なForthシステムでは使えない点に注意が必要である。

### 型指定子一覧

| 型指定子 | 意味 | サイズ | 値/アドレス |
|---------|------|--------|------------|
| `W:` または無指定 | セル（ワード） | 1 cell | 値 |
| `W^` | セルの変数 | 1 cell | アドレス |
| `D:` | ダブルワード | 2 cells | 値 |
| `D^` | ダブルワードの変数 | 2 cells | アドレス |
| `F:` | 浮動小数点数 | 1 float | 値 |
| `F^` | 浮動小数点変数 | 1 float | アドレス |
| `C:` | 文字 | 1 char | 値 |
| `C^` | 文字の変数 | 1 char | アドレス |
| `XT:` | 実行トークン | 1 char | アドレス |
| `XT^` | 実行トークン変数 | 1 char | アドレス |

### 値フレーバーと変数フレーバー

#### 値フレーバー（`W:`、`D:`、`F:`、`C:`）:
- ローカル変数名を参照すると、その**値**がスタックに積まれる
- `TO`を使って値を変更できる
- `VALUE`と同じ振る舞い

```:値フレーバー
\ ==================== W: セル（値フレーバー） ====================
: price-with-tax { W: price W: qty W: tax -- total }
  price qty * dup tax * 100 / + ;

1000 3 10 price-with-tax .     \ → 3300

\ ==================== F: 浮動小数点数（値フレーバー） ====================
3.14159e fconstant mypi

: calc-area { F: radius -- F: area }
  mypi radius radius f* f* ;

5e calc-area f.       \ → 78.53975

\ ==================== D: ダブルワード ====================
: add-thousand { D: num -- D: result }
  num 1000. d+ ;

5000. add-thousand d.   \ → 6000

\ ==================== C: 文字（値フレーバー） ====================
: char-upper { C: ch -- C: result }
  ch [char] a >= ch [char] z <= and if
    ch 32 -
  else
    ch
  then ;

: test-char-upper
  [char] a char-upper emit space
  [char] z char-upper emit space
  ." (should be A Z)" cr ;

test-char-upper     \ → A Z
```


#### 変数フレーバー（`W^`、`D^`、`F^`、`C^`）:
- ローカル変数名を参照すると、その**アドレス**がスタックに積まれる
- `!`や`@`を使って値を操作する
- `VARIABLE`と同じ振る舞い
- スコープを抜けるとアドレスは無効になる

```
\ ==================== W^ セル（変数フレーバー） ====================
\ スタックから値を取り、そのアドレスで計算
: times-two-and-swap { W^ a W^ b } 
  a @ 2 * b @ 2 * \ a*2 b*2
  a ! b !         \ aのアドレスにb*2、bのアドレスにa*2を格納
  a @ b @
; 

3 7 times-two . .  \ → 6 14

\ ==================== F^ 浮動小数点数（変数フレーバー） ====================
\ 2つの浮動小数点数を入れ替える
: float-swap { F^ a F^ b -- fa fb }
  a F@ b F@
  a F! b F!
  a F@ b F@ ;

1.5e 2.5e float-swap F. F.    \ → 1.5 2.5

\ ==================== C^ 文字（変数フレーバー） ====================
: print-one-char { C^ ch-ptr -- }
  ch-ptr 1 type ;

: test-print-chars
  72 print-one-char
  69 print-one-char 
  76 print-one-char 
  76 print-one-char 
  79 print-one-char 
  ." (should be HELLO)" cr ;

test-print-chars    \ → HELLO
```

変数フレーバー（アドレスを返す）を使うと、より柔軟な操作が可能になる。

### 実行トークン

実行トークンは、ワードのアドレスで、データスタックに入る。アドレスの状態で受け取った場合は、`EXECUTE`を使って実行することが出来る。

```
: test { W: xt -- n }
    xt EXECUTE ;
1 2 ' + test  \ → 3
```

`'`（シングルクォーテーション） の後にスペースを入れてワードを記述すると、ワードのアドレスがスタックに積まれる。gforthでは、\`（バッククオート）に続けてワードを記述することが出来る。

```
: test { W: xt -- n }
    xt EXECUTE ;
1 2 `+ test  \ → 3
```

実行トークンを`xt:`を使ってローカル変数として受け取った場合、`execute`を使わずにワードを実行することが出来る。

```
: test { XT: x -- n }
    x ;
1 2 `+ test  \ → 3
```

### emit の再実装

標準ワード`emit`は、`type`を使って次のように定義できる。

```forth
: emit { C^ char* -- }
  char* 1 type ;
```

```forth:実行例
65 emit   \ → A  （ASCII 65は'A'）
72 emit 69 emit 76 emit 76 emit 79 emit  \ → HELLO
```

この実装では、`char*`はスタックから取得した文字値（1バイト）を格納するローカル変数の**アドレス**を保持する。`TYPE`はアドレスと長さを受け取るため、`char* 1`で「このアドレスから1文字分」を出力している。

従来の実装では、以下のようになる。

```forth:標準的な実装（ローカル変数なし）
: emit ( c -- )
  PAD C!      \ PADに文字を格納
  PAD 1 TYPE  \ PADから1文字出力
;
```

ローカル変数版では、明示的に`PAD`を使う必要がなく、自動的に一時領域が確保される点が利点である。

## Standard Forth localsとの違い

Gforthのローカル変数は、ANS Forth標準の仕様を拡張したものである。標準的なForthシステムに移植する場合は、以下の制限に注意する必要がある。

### Standard Forth localsの制限事項

| 項目 | Standard Forth | Gforth |
|------|----------------|--------|
| 型指定子 | **不可**（セルサイズのみ） | 可（W:, D:, F:, C: 等） |
| 定義位置 | **制御構造の外側のみ** | 定義内のどこでも可 |
| 1行制限 | **必須** | 不要 |
| リターンスタック | 干渉の可能性あり | 同様 |
| 動作 | VALUEと同じ | VALUEと同じ（値フレーバー） |

#### 1. 型指定子が使えない

Standard Forthでは、ローカル変数はセルサイズ（通常の整数）の値のみをサポートする。`F:`や`D:`などの型指定子は使えない。

```forth
\ Gforth専用（他では動作しない）
: area { F: width F: height -- F: result }
  width height f* ;

\ Standard Forth互換
: area { width height -- result }
  width height * ;
```

#### 2. 制御構造の外側でのみ定義可能

Standard Forthでは、ローカル変数は定義の**先頭**で宣言しなければならない。Gforthでは定義内のどこでも宣言できる。

```forth
\ Gforthでは可能だが、Standard Forthでは不可
: example ( n -- )
  dup . cr
  { n -- }   \ ← 制御構造の途中での定義
  n 2 * . ;

\ Standard Forth互換の書き方
: example { n -- }
  n dup . cr
  n 2 * . ;
```

#### 3. リターンスタックとの干渉

ローカル変数の実装は、多くの場合**リターンスタック**を使用する。そのため、`>R`、`R>`、`R@`などのリターンスタック操作ワードと併用する場合は注意が必要である。

```forth
\ 危険：ローカル変数とリターンスタック操作の混在
: risky { a b -- }
  a >R          \ リターンスタックに積む
  b 2 *         \ 計算
  R> + ;        \ 取り出して加算

\ より安全：ローカル変数内で完結
: safe { a b -- }
  a b 2 * + ;
```

**原則**: ローカル変数を使う定義内でリターンスタック操作を直接使わなければ、問題は起きない。

#### 4. 定義全体が1行でなければならない

Standard Forthの仕様では、ローカル変数を使う定義は**1行**に収める必要がある（Gforthではこの制限はない）。

```forth
\ Standard Forthでは1行で書く必要がある
: area { w h -- } w h * ;

\ Gforthでは複数行でも可
: area { w h -- }
  w h 
  * ;
```

### LOCALS| 構文について（非推奨）

ANS Forth標準には`LOCALS|`という別の構文も定義されているが、**ローカル変数の順序がスタックコメントと逆になる**ため、Gforthマニュアルでは使用を強く非推奨としている。

```forth
\ LOCALS| 構文（非推奨）
: max
  LOCALS| n2 n1 |  \ ← 順序が逆！
  n1 n2 > if n1 else n2 endif ;

\ 推奨される {} 構文
: max { n1 n2 -- }
  n1 n2 > if n1 else n2 endif ;
```

`LOCALS|`は読み間違いや書き間違いを招きやすく、可読性が低下するため、使用すべきではない。

## ローカル変数とグローバル変数の比較

| 特性 | VARIABLE/VALUE | ローカル変数 |
|------|---------------|------------|
| スコープ | グローバル（辞書全体） | ワード定義内のみ |
| 生存期間 | 永続的 | ワード実行中のみ |
| 初期化 | 明示的（スタックまたは固定値） | スタックから自動取得 |
| 変更方法 | `!`または`TO` | `TO`（値）、`!`（変数） |
| 型サポート | 明示的な別定義が必要 | Gforthでは型指定子 |
| 名前の衝突 | 可能性高い | ワード内で完結 |
| 移植性 | 高い | Standard準拠なら高い |
| メモリ効率 | 常に確保 | 必要時のみ確保 |
| 推奨用途 | 複数ワードで共有 | ワード内の一時計算 |

### いつローカル変数を使うべきか

| 状況 | 推奨 | 理由 |
|------|------|------|
| 複数のワードで共有する状態 | グローバル（VALUE/VARIABLE） | スコープが広い |
| ワード内だけで使う一時変数 | ローカル変数 | 名前の衝突を防ぐ |
| 複雑なスタック操作が必要 | ローカル変数 | 可読性向上 |
| 単純な計算（1〜2個のスタック項目） | スタック操作 | シンプル |
| 再帰的な処理 | ローカル変数 | 各呼び出しで独立 |

## 使用上の注意点

### 1. スタックコメントとの混同を避ける

ローカル変数の構文はスタックコメントと似ているため、混同しないよう注意が必要。

```forth
\ これはローカル変数定義
: example { a b -- c }
  a b + ;

\ これは単なるスタックコメント
: example ( a b -- c )
  + ;
```

**推奨**: 同一ワード内では、ローカル変数定義とスタックコメントの両方を書かない。ローカル変数定義がスタックコメントを兼ねる。

### 2. スコープと生存期間の理解

ローカル変数は、定義されたワード内でのみ有効。ワードの実行が終了すると、変数フレーバーのアドレスも無効になる。

```forth
\ 危険：スコープ外でアドレスを使用
: get-address { W^ x -- addr }
  x ;  \ ← このアドレスはワード終了後に無効！

: dangerous
  10 get-address  \ アドレスを取得
  @ . ;           \ ← 未定義動作！
```

### 3. 移植性の考慮

他のForthシステムへの移植を考える場合、Gforth拡張機能の使用は避ける。

```forth
\ 移植性低（Gforth専用）
: hypotenuse { F: a F: b -- F: c }
  a a f* b b f* f+ fsqrt ;

\ 移植性高（Standard Forth互換）
: hypotenuse { a b -- c }
  a a * b b * + ;  \ 整数演算のみ
```

**移植用ファイル**: Gforthの`{}`構文を他のANS Forthシステムで使うには、`compat/anslocal.fs`をインクルードする。

### 4. パフォーマンスへの影響

ローカル変数の使用は、わずかなオーバーヘッドを伴う（メモリ確保と初期化）。ただし、ほとんどの場合、可読性の向上というメリットが上回る。

パフォーマンスが極めて重要な場合のみ、スタック操作による最適化を検討する。

### 5. コーディング規約

プロジェクトで一貫したスタイルを保つこと：

- **小規模プロジェクト**: スタック操作中心、複雑な部分のみローカル変数
- **大規模プロジェクト**: 積極的にローカル変数を使用し、可読性を優先
- **ライブラリコード**: Standard Forth互換性を維持

## 実用例

### 1. 二次方程式の判別式

```forth
: discriminant { F: a F: b F: c -- F: D }
  \ D = b^2 - 4ac
  b b f* 
  4e a f* c f* f- ;

1e 5e 6e discriminant f.  \ → 1.0  (b^2-4ac = 25-24 = 1)
1e 2e 1e discriminant f.  \ → 0.0  (重解)
1e 1e 1e discriminant f.  \ → -3.0 (虚数解)
```

### 2. 日付の妥当性チェック

```forth
: leap-year? { year -- flag }
  year 400 mod 0= if
    true exit  \ 400で割り切れる → うるう年
  then
  year 100 mod 0= if
    false exit  \ 100で割り切れる → 平年
  then
  year 4 mod 0= ;  \ 4で割り切れる → うるう年

: valid-date? { year month day -- flag }
  \ 月の範囲チェック
  month 1 < month 12 > or if
    false exit
  then
  \ 日の範囲チェック
  day 1 < if
    false exit
  then
  \ 各月の最大日数をチェック
  month 2 = if
    year leap-year? if
      day 29 >  \ うるう年は29日まで
    else
      day 28 >  \ 平年は28日まで
    then
    if false else true then
  else
    month 4 = month 6 = or month 9 = or month 11 = or if
      day 30 >
    else
      day 31 >
    then
    if false else true then
  then ;

2025 2 29 valid-date? .  \ → 0 (偽：2025年は平年)
2025 2 28 valid-date? .  \ → -1 (真)
```

### 3. フィボナッチ数列（再帰版）

ローカル変数を使うと、再帰的な定義が読みやすくなる。

```forth
: fib { n -- result }
  n 2 < if
    n
  else
    n 1 - recurse
    n 2 - recurse
    +
  then ;

0 fib .  \ → 0
1 fib .  \ → 1
10 fib . \ → 55
```

各再帰呼び出しで、`n`は独立したローカル変数として機能する。

### 4. 文字列の比較

```forth
: str-equal? { addr1 len1 addr2 len2 -- flag }
  len1 len2 <> if
    false exit  \ 長さが異なれば不一致
  then
  len1 0 do
    addr1 i + c@
    addr2 i + c@
    <> if
      false unloop exit
    then
  loop
  true ;

S" hello" S" hello" str-equal? .  \ → -1 (真)
S" hello" S" world" str-equal? .  \ → 0 (偽)
```

### 5. 温度変換（摂氏⇔華氏）

```forth
: c>f { F: celsius -- F: fahrenheit }
  celsius 9e f* 5e f/ 32e f+ ;

: f>c { F: fahrenheit -- F: celsius }
  fahrenheit 32e f- 5e f* 9e f/ ;

0e c>f f.    \ → 32.0
100e c>f f.  \ → 212.0
32e f>c f.   \ → 0.0
```

## デバッグとトレース

ローカル変数を使う際のデバッグテクニック：

### 値の確認

```forth
: debug-max { n1 n2 -- n3 }
  ." n1=" n1 . 
  ." n2=" n2 . cr
  n1 n2 > if
    ." returning n1" cr
    n1
  else
    ." returning n2" cr
    n2
  endif ;

10 5 debug-max .
\ → n1=10 n2=5
\ → returning n1
\ → 10
```

### 中間値の表示

```forth
: verbose-calc { a b c -- result }
  ." a=" a . cr
  ." b=" b . cr
  ." c=" c . cr
  a b + TO a
  ." a+b=" a . cr
  a c * ;

2 3 4 verbose-calc .
\ → a=2
\ → b=3
\ → c=4
\ → a+b=5
\ → 20
```

