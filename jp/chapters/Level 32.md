# Lev 32 ー 補講６　Thinking Forth の"高速化哲学"
## 1. Thinking Forth の"高速化哲学"

Thinking Forth の高速化思想は、単にコードを速くするという表面的な意味ではなく、

> 「適切な語彙設計・抽象化・データフローによって、そもそも軽量で高速な構造を自然につくる」

という哲学である。

多くの初心者が行う"力任せの最適化"とは違う。

その基本方針は次の三つである。


### 原則1：高速化は最後に行う

* まずは 正しさ と 読みやすさ を優先する。
* 性能問題を感じていない段階での最適化は無意味である。
* 設計段階での構造化が性能問題の8割を消す。


### 原則2：ホットスポット（最頻実行部分）に限定して最適化

* 全体の速度を決めるのは"ボトルネック部分"のみ。
* 高レベル語彙の最適化はほぼ意味がない。
* 内側ループ・反復回数の大きい部分を徹底的に磨く。


### 原則3：高速化は「語彙レベル」で行う

これは Thinking Forth 特有のポイントである。

* 低レベル語彙（primitive）を高速に保つ
* 高レベル語彙はその高速な"道具"を組み合わせる
* 語彙の分割が正しく出来ていれば、高速化は局所的に行える

## 2. gforth における性能計測の基本

gforth では、性能改善の判断には明確な計測結果が必要である。


### 実行時間計測の基本

gforthでは `UTIME` を使った簡易ベンチマークが可能である。

```forth
\ 簡易ベンチマーク
: BENCH ( xt -- us )
   UTIME 2>R
   EXECUTE
   UTIME 2R> D- DROP ;

\ 使用例
: TEST 1000000 0 DO LOOP ;
' TEST BENCH . CR  \ マイクロ秒で表示
```

より詳細な測定には、繰り返し実行して平均を取る。

```forth
: BENCH-AVG ( xt n -- avg-us )
   TUCK 0 -ROT               \ ( n 0 xt n )
   0 DO
      DUP BENCH ROT + SWAP   \ 累積
   LOOP
   DROP SWAP / ;             \ 平均

' TEST 10 BENCH-AVG .  \ 10回の平均
```

この計測結果なしに、速度改善を議論してはならない。


### "高速化が必要かどうか"を判断する指標

#### 指標1：ワードが1秒間に10^6回以上呼ばれる

→ 最適化の対象として十分。

#### 指標2：ループ内で複数ワードを呼んでいる

→ 合体・インライン化を検討。

#### 指標3：メモリアクセスがボトルネック

→ 早期にキャッシュ性を考慮したデータ構造へ。


## 3. 内側ループ（inner loop）最適化の基本

最も大きな性能効果を持つのは「内側ループの最適化」である。

### スタック操作の回数削減

スタック操作は軽量だが、積み重なると大きな差になる。

例えば、

```forth
OVER SWAP ROT
```

のような"多段階入れ替え"は避ける。

#### 改善の方針

* スタックフレンドリーな語彙を設計する
* 不要な SWAP / ROT / DUP を削除
* 中間値はローカル変数に退避させる

#### 例：ローカル変数を使った高速版

```forth
\ スタック操作が多い版
: DOT-PROD-SLOW ( addr1 addr2 n -- sum )
	0 3 ROLL 3 ROLL 3 ROLL
	0 DO
		OVER I CELLS + @
		OVER I CELLS + @
		*
		3 ROLL + -ROT
	LOOP
	DROP DROP
;

\ ローカル変数版（高速）
: DOT-PROD-FAST ( addr1 addr2 n -- sum )
   LOCALS| n a2 a1 |
   0
   n 0 DO
      a1 I CELLS + @
      a2 I CELLS + @
      * +
   LOOP ;
```

ローカル変数により ROLL / OVER / -ROT がなくなり高速化されている。


### メモリアクセスの最適化（特に行列計算）

行列計算などでは、

* アクセス順
* 配列の直線性
* キャッシュの局所性

が重要。

行優先格納の行列では、行方向のアクセスがキャッシュ効率が良い。

```forth:遅い：列方向アクセス
n 0 DO                          \ I: 0→n-1（行インデックス）
   m 0 DO                       \ J: 0→m-1（列インデックス）
      matrix I m * J + CELLS + @  \ matrix[I][J]にアクセス
   LOOP
LOOP


アクセス順序:
[0] → [4] → [8]    ← 列0を縦に読む
[1] → [5] → [9]    ← 列1を縦に読む
[2] → [6] → [A]    ← 列2を縦に読む
[3] → [7] → [B]    ← 列3を縦に読む

メモリ上のアクセス:
[0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [A] [B]
 ↓           ↓           ↓
 1           2           3    ← 1列目
     ↓           ↓           ↓
     4           5           6 ← 2列目
         ↓           ↓           ↓
         7           8           9 ← 3列目
```

```:速い：行方向アクセス
m 0 DO                          \ I: 0→m-1（行インデックス）
   n 0 DO                       \ J: 0→n-1（列インデックス）
      matrix I n * J + CELLS + @  \ matrix[I][J]にアクセス
   LOOP
LOOP

アクセス順序:
[0] → [1] → [2] → [3]    ← 行0を横に読む
[4] → [5] → [6] → [7]    ← 行1を横に読む
[8] → [9] → [A] → [B]    ← 行2を横に読む

メモリ上のアクセス:
[0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [A] [B]
 ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓
 1   2   3   4   5   6   7   8   9  10  11  12
```

これは Forth に限らず科学計算全般で重要。


###  内側ループから関数呼び出しを排除

内側ループが数百万回回る場合、

```forth
addr I J MAT-GET-CELL
```

のようなワード呼び出しが致命的に遅くなる。

#### → "インライン化"が有効

メモリアクセスを直接書く。

```forth
\ 遅い
: PROCESS-MATRIX ( addr rows cols -- )
   0 DO
      0 DO
         OVER I J MAT-GET  \ ワード呼び出し
      LOOP
   LOOP DROP ;

\ 速い（インライン化）
: PROCESS-MATRIX-FAST ( addr rows cols -- )
   LOCALS| cols rows addr |
   rows 0 DO
      cols 0 DO
         addr I cols * J + CELLS + @  \ 直接アクセス
      LOOP
   LOOP ;
```

適切に意味ベース語彙を分けておけば、
必要なときだけ下位レベル化できる。

## 4. データ構造の高速化（Forth特有のポイント）

gforth は C 言語に近い低レベルアクセスが可能であり、
性能改善を最大にするには データ構造の選択 が重要である。

### 配列は"単純な連続メモリ"が最速

Forth では構造体を複雑化するとキャッシュ効率が悪くなる。
特に科学計算では、

```
N × M の2次元配列 → 一次元メモリに flatten
```

することが最速。

```forth
\ 2次元配列を1次元配列として確保
: MAKE-MATRIX ( rows cols -- addr )
   * CELLS ALLOCATE THROW ;

\ アクセス
: MAT@ ( addr row col cols -- n )
   ROT * + CELLS + @ ;
```


### データは「計算しやすい形」に変換しておく

前処理（precompute）が高速化の鍵。

例：行列の列数・行数を毎回取得するのではなく、
ループ前にローカル変数へ格納する。

```forth
\ 遅い
: SLOW ( matrix-struct -- )
   100 0 DO
      DUP MAT-COLS  \ 毎回取得
      DUP MAT-ROWS  \ 毎回取得
      ...
   LOOP ;

\ 速い
: FAST ( matrix-struct -- )
   DUP MAT-COLS LOCALS| cols |
   DUP MAT-ROWS LOCALS| rows |
   100 0 DO
      rows cols ...  \ 定数として使用
   LOOP ;
```

### 文字列処理ではバイト列をそのまま扱え

* `C@` を使う
* 中間バッファを作らない
* 1文字ずつ調べるより「区切り位置を一気に探す」方が高速

```forth
\ 遅い：1文字ずつ
: COUNT-SPACES-SLOW ( addr u -- n )
   0e
   0 DO
      DUP I + C@ BL = IF 1e F+ THEN
   LOOP DROP F>S ;

\ 速い：ポインタ操作
: COUNT-SPACES-FAST ( addr u -- n )
   0e
   OVER + SWAP DO
      I C@ BL = IF 1e F+ THEN
   LOOP F>S ;
```

---

## 5. 高速化のための語彙設計（Thinking Forth第二段階）

性能改善は"言語改良"と考えると理解が早い。

### ポイント1：高レベル語彙は変えずに、低レベル語彙を差し替える

これが Thinking Forth 最大の利点。
高レベル語彙は"意図"のみ書かれているため、高速化しても壊れない。

例。

```forth
\ 初期版
: @CELL ( addr i -- n )  CELLS + @ ;

\ 高速版に差し替え（64bit環境）
: @CELL ( addr i -- n )  3 LSHIFT + @ ;
```

高レベルコードはそのまま高速化される。

### ポイント2：語彙の"交換可能性"を保つ

* ロジック部分
* 入れ替え可能な低レベルアクセスコード
* 最適化された quick path
* デバッグ情報あり／なしのバージョン

複数のバージョンを持てるように設計する。

```forth
\ デバッグ版
: @CELL-DEBUG ( addr i -- n )
   2DUP ." Access: " . . CR
   CELLS + @ ;

\ 本番版
: @CELL ( addr i -- n )
   CELLS + @ ;
```


### ポイント3：高レベルでの"意味の明確化"が性能改善を助ける

Forthで高速化がやりやすい理由は、

> 高レベル語彙が意図しか書いていない

からである。

## 6. gforth特有の高速化ポイント

他言語にはない、gforth固有の最適化テクニックを紹介する。

### 6.1. ローカル変数は適切に使えば高速

スタック操作が減るため、
複雑な処理では高速化に有効。

```forth
\ スタック版（SWAP/OVERが多い）
: CALC ( a b c d -- result )
   ROT + SWAP ROT * + ;

\ ローカル変数版（明快で高速）
: CALC ( a b c d -- result )
   { a b c d }
   a b + c * d + ;
```


### 6.2. `CELLS` を明示シフトで高速化

環境依存だが、CELL サイズが確定している場合。

```forth
\ 32bit環境: 1 CELL = 4 bytes
: @CELL ( addr i -- n )  2 LSHIFT + @ ;

\ 64bit環境: 1 CELL = 8 bytes
: @CELL ( addr i -- n )  3 LSHIFT + @ ;
```

注意: 移植性を失うため、性能クリティカルな部分のみに使用。

### 6.3. `ALIGNED` の効果

メモリアラインメントを満たすことで、高速なロード・ストアが可能。

```forth
CREATE BUFFER 1000 CELLS ALLOT
BUFFER ALIGNED  \ アラインメント（境界調整）
```

### 6.4. 条件分岐の削減

内側ループでは条件分岐は避ける。

```forth
\ 遅い
: SUM-IF-POSITIVE ( addr n -- sum )
   0 -ROT
   0 DO
      OVER I CELLS + @
      DUP 0> IF + ELSE DROP THEN
   LOOP DROP ;

\ 速い
: SUM-IF-POSITIVE ( addr n -- sum )
   0 -ROT
   0 DO
      OVER I CELLS + @
      0 MAX +  \ 条件分岐なし
   LOOP DROP ;
```

### 6.5. 浮動小数点演算の最適化

科学計算では `F@` `F!` `F+` などが頻出。
ローカル変数との組み合わせで高速化。

```forth
\ ベクトルの内積を計算 (a·b = Σ(ai×bi))
: VEC-DOT ( faddr1 faddr2 n -- f: -- result )
   LOCALS| n a2 a1 |        \ ローカル変数: n=要素数, a2=vec2アドレス, a1=vec1アドレス
   0.0e                     \ 累積値を0.0で初期化
   n 0 DO                   \ n回繰り返し
      a1 I FLOATS + F@      \ vec1[i]を読み込み
      a2 I FLOATS + F@      \ vec2[i]を読み込み
      F* F+                 \ 乗算して累積値に加算
   LOOP ;                   \ 結果: Σ(a1[i]×a2[i])
```

