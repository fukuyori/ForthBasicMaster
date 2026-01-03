# Lev 26 ー 行列演算と線形代数
## 1. 行列とは何か ー なぜコンピュータで扱うのか

行列とは、数値を縦横に並べた二次元の数表です。一見すると単なる「表」ですが、コンピュータ科学では極めて重要な役割を果たします。

**なぜ行列が重要なのか:**
- **画像処理**: 画像は画素の行列として表現される
- **機械学習**: ニューラルネットワークの計算は行列演算で構成される
- **統計解析**: データセットは行列形式で扱われる
- **3Dグラフィックス**: 回転や拡大縮小は行列で表現される

例えば、次のような2行3列の行列を考えます:

```
    列1  列2  列3
行1 [ 1   2   3 ]
行2 [ 4   5   6 ]
```

Pythonなどの高級言語では、このような行列は `[[1,2,3],[4,5,6]]` のように簡単に書けます。しかし、コンピュータのメモリは一次元の連続した領域です。では、どうやって二次元の行列を表現するのでしょうか?

## 2. メモリ上での行列表現 ー 二次元を一次元に変換する

### 2.1 メモリの仕組み

コンピュータのメモリは、番地(アドレス)が付いた一列の箱のようなものです。FORTHでは、この「箱」を**セル(CELL)**と呼び、各セルに1つの数値を格納します。

```
アドレス: [0][1][2][3][4][5][6][7]...
         メモリは一次元に並んでいる
```

### 2.2 行列の格納形式

行列をメモリに格納して演算を行えるようにするため、以下のような構造にしました。

1. **メタデータ**: 行数と列数
2. **データ本体**: 各要素の値

例として、2行3列の行列 `[[1,2,3],[4,5,6]]` を格納してみましょう:

```
メモリ配置:
┌────────┬────────┬───┬───┬───┬───┬───┬───┐
│ ROW:2  │ COL:3  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │
└────────┴────────┴───┴───┴───┴───┴───┴───┘
 アドレス  アドレス  データ領域(行優先で格納)
   +0       +1      +2  +3  +4  +5  +6  +7
```

この格納方式を**行優先順序(row-major order)** と呼びます。最初の行を全て格納してから、次の行を格納していきます。

### 2.3 要素へのアクセス

行列の要素 `A[i][j]` (i行j列目) のアドレスを計算する公式:

```
要素のアドレス = 先頭アドレス + (2 + (i × 列数 + j)) × セルサイズ
```

**具体例**: 上記の行列で `A[1][2]` (2行目3列目、値は6) を取得する場合:
```
アドレス = 先頭 + 2 + (1 × 3 + 2) = 先頭 + 7
         = 7番目のセル → 値は 6
```

このように、二次元の座標を一次元のアドレスに変換できます。

### 2.4 FORTHでの定数定義

メモリ配置を明確にするため、オフセットを定数として定義します:

```forth
\ 行列データ構造: [行数][列数][データ...]
0 CONSTANT ROW-OFFSET      \ 行数は先頭(オフセット0)
1 CELLS CONSTANT COL-OFFSET \ 列数は1セル目
2 CELLS CONSTANT DATA-OFFSET \ データは2セル目から
```

**説明**:
- `ROW-OFFSET`: 行数が格納された位置(先頭から0セル)
- `COL-OFFSET`: 列数が格納された位置(先頭から1セル)
- `DATA-OFFSET`: 実際のデータ開始位置(先頭から2セル)

## 3. FORTHによる行列の基本操作

### 3.1 行列の作成

まず、行列を格納するメモリ領域を確保します:

```forth
\ 行列を作成(行数 × 列数の領域を確保)
: MAT-CREATE ( rows cols "name" -- )
    CREATE
        OVER ,              \ 行数を格納
        DUP ,               \ 列数を格納
        * 2 + CELLS ALLOT   \ データ領域を確保
    DOES> ( -- addr )       \ 実行時は行列のアドレスを返す
;
```

**詳細説明**:
1. `CREATE`: 新しい単語(名前)を辞書に登録
2. `OVER ,`: 行数をメモリに書き込む
3. `DUP ,`: 列数をメモリに書き込む
4. `* 2 + CELLS ALLOT`: (行数×列数+2)分のメモリを確保
5. `DOES>`: この単語が実行されたときの動作を定義

**使用例**:
```forth
2 3 MAT-CREATE M1   \ 2行3列の行列M1を作成
```

これで、M1という名前で2×3の行列が使えるようになります。

### 3.2 行数・列数の取得

行列のサイズ情報を読み取る基本ワードを定義します:

```forth
\ 行数を取得
: MAT-ROWS ( addr -- rows )
    @ ;  \ 先頭セルの値を読む

\ 列数を取得
: MAT-COLS ( addr -- cols )
    CELL+ @ ;  \ 1セル進んでから読む

\ 総要素数を取得
: MAT-SIZE ( addr -- size )
    DUP MAT-ROWS SWAP MAT-COLS * ;
```

**動作確認**:
```forth
M1 MAT-ROWS .   \ 2 と表示される
M1 MAT-COLS .   \ 3 と表示される
M1 MAT-SIZE .   \ 6 と表示される
```

### 3.3 行列への値の設定

行列に実際の数値を格納する処理です。これが最も重要な操作の一つです:

```forth
\ 行列に値を設定(スタック上の値を逆順に格納)
: MAT-SET ( a11 a12 ... ann addr -- )
	DUP MAT-SIZE DUP                   \ 要素数
	ROT DATA-OFFSET + SWAP 1- CELLS +  \ データ領域の最後
	SWAP 0 DO
		SWAP OVER !                    \ 値を書き込む
		1 CELLS -					   \ アドレスを1つ減らす
	LOOP
    DROP
;
```

**スタック操作の詳細**:

初期状態: `1 2 3 4 5 6 M1`のとき
```
スタック: [ 1 2 3 4 5 6 M1 ]
          ↑要素         ↑アドレス
```

ループ内での処理(i=0の場合):
```
1. DUP DATA-OFFSET +  → データ領域の先頭アドレス
2. I CELLS +          → 0番目の要素位置
3. ROT ROT            → 値(1)を取り出す
4. SWAP !             → その位置に1を書き込む
```

**使用例**:
```forth
2 3 MAT-CREATE M1
1 2 3 4 5 6 M1 MAT-SET  \ M1に値を設定

実行後のM1の内容:
[ 1 2 3 ]
[ 4 5 6 ]
```

### 3.4 行列の表示

行列を見やすい形式で表示します:

```forth
\ 行列を表示
: MAT-SHOW ( addr -- )
    DUP MAT-COLS        \ 列数
    OVER DATA-OFFSET +  \ データ領域
    ROT MAT-ROWS        \ 行数
	0 DO
        CR ." [ "       \ 行の開始
        OVER
		0 DO
			DUP @ .     \ 値を表示
			CELL+       \ 次のデータ領域へ
		LOOP
        ." ]"           \ 行の終了
	LOOP
    DROP DROP
;
```

**要素アドレスの計算**:
```
A[i][j]のアドレス = データ開始 + (i × 列数 + j) × セルサイズ
```

ここで、`J`はFORTHの外側ループカウンタ(行番号)、`I`は内側ループカウンタ(列番号)です。

**出力例**:
```forth
M1 MAT-SHOW

[ 1 2 3 ]
[ 4 5 6 ]
```

## 4. 行列の加算 ー 要素ごとの演算

### 4.1 行列加算の数学的定義

2つの行列A、Bの加算は、対応する要素同士を足し合わせます:

```
C[i][j] = A[i][j] + B[i][j]
```

**条件**: AとBは同じサイズでなければなりません。

**例**:
```
[ 1 2 ]   [ 5 6 ]   [ 6  8 ]
[ 3 4 ] + [ 7 8 ] = [ 10 12 ]
```

### 4.2 実装戦略

行列加算を段階的に実装します:

1. サイズチェック
2. 各要素を取得
3. 対応する要素同士を加算
4. 結果を新しい行列に格納

```forth
\ 2つの行列のサイズが一致するかチェック
: MAT-SIZE-MATCH? ( addr1 addr2 -- flag )
    OVER MAT-ROWS OVER MAT-ROWS = 
    ROT MAT-COLS ROT MAT-COLS = 
    AND ;

\ 行列の加算
: MAT-ADD ( addrA addrB -- )
    2DUP MAT-SIZE-MATCH? 0= IF
        2DROP
            CR ." Error: Matrix sizes do not match"
        EXIT
    THEN

    DUP MAT-SIZE        \ ループ回数
    ROT DATA-OFFSET +   \ 配列1のアドレス
    ROT DATA-OFFSET +   \ 配列2のアドレス
    ROT 0 DO
        OVER @          \ 配列1の値
        OVER @ +        \ 配列2の値
        ROT CELL+       \ 配列1のアドレスを次に進める
        ROT CELL+       \ 配列2のアドレスを次に進める
    LOOP
    DROP DROP
;
```

**使用例**:
```forth
2 2 MAT-CREATE M1
2 2 MAT-CREATE M2
2 2 MAT-CREATE MR   \ 結果用の行列

1 2 3 4 M1 MAT-SET
5 6 7 8 M2 MAT-SET

M1 MAT-SHOW CR      \ [ 1 2 ] [ 3 4 ]
M2 MAT-SHOW CR      \ [ 5 6 ] [ 7 8 ]

M1 M2 MR MAT-ADD
MR MAT-SHOW         \ [ 6 8 ] [ 10 12 ]
```

### 4.3 エラー処理

サイズが一致しない場合の動作:

```forth
3 3 MAT-CREATE M3
1 2 3 4 5 6 7 8 9 M3 MAT-SET

M1 M3 MR MAT-ADD
\ 出力: Error: Matrix sizes do not match
```

## 5. 行列の乗算 ー より複雑な演算

### 5.1 行列乗算の数学的定義

行列の乗算は加算より複雑です。行列A(m×n)と行列B(n×p)の積C(m×p)は:

```
C[i][j] = Σ(k=0 to n-1) A[i][k] × B[k][j]
```

**重要な制約**: AとBが乗算可能であるためには、**Aの列数 = Bの行数**でなければなりません。

**例** (2×3行列 × 3×2行列 = 2×2行列):
```
[ 1 2 3 ]   [ 1 2 ]   [ 1×1+2×3+3×5  1×2+2×4+3×6 ]
[ 4 5 6 ] × [ 3 4 ] = [ 4×1+5×3+6×5  4×2+5×4+6×6 ]
            [ 5 6 ]
          
          = [ 22 28 ]
            [ 49 64 ]
```

### 5.2 実装に必要な部品

行列乗算を実装するには、以下のヘルパーワードが必要です:

```forth
\ 指定行の先頭アドレスを取得
: MAT-ROW-ADDR ( addr n -- addr' )
	OVER MAT-COLS *             \ 列数を掛けてオフセット計算
	CELLS +                     \ バイトオフセットに変換
	DATA-OFFSET +                      \ データ開始位置を加算
;

\ 指定位置のセルの値を取得
: MAT-GET-CELL ( addr m n -- value )
	ROT ROT MAT-ROW-ADDR        \ 行のアドレスを取得
	SWAP CELLS + @              \ 列オフセットを加算して値を取得
;
```

### 5.3 行列乗算の実装

```forth
\ 行列の乗算 A × B
: MAT-MUL ( addr-A addr-B -- a11*b11+a12*b21 ... )
    { addr-A addr-B }
    addr-A MAT-COLS addr-B MAT-ROWS =
    0= IF
        CR ." Error: Matrix dimensions incompatible"
        EXIT
    THEN
    addr-A MAT-ROWS 0 DO                    \ C[I][J]の計算
        addr-B MAT-COLS 0 DO
            0                               \ 累積値の初期化
            addr-A MAT-COLS 0 DO            \ 内積計算
                addr-A K I MAT-GET-CELL     \ A[I][K]
                addr-B I J MAT-GET-CELL     \ B[K][J]
                CR .S
                * +                         \ 乗算して累積加算
            LOOP
        LOOP
    LOOP
;
```

**使用例**:
```forth
2 3 MAT-CREATE MA
3 2 MAT-CREATE MB
2 2 MAT-CREATE MC    \ 結果用

1 2 3 4 5 6 MA MAT-SET
1 2 3 4 5 6 MB MAT-SET

MA MAT-SHOW CR
\ [ 1 2 3 ]
\ [ 4 5 6 ]

MB MAT-SHOW CR
\ [ 1 2 ]
\ [ 3 4 ]
\ [ 5 6 ]

MA MB MAT-MUL
MC MAT-SET
MC MAT-SHOW
\ [ 22 28 ]
\ [ 49 64 ]
```

## 6. 転置行列 ー 行と列の入れ替え

### 6.1 転置の定義

行列の転置(transpose)とは、行と列を入れ替える操作です。数学的には:

```
Aᵀ[i][j] = A[j][i]
```

**例**:
```
元の行列:      転置行列:
[ 1 2 3 ]     [ 1 4 ]
[ 4 5 6 ]     [ 2 5 ]
              [ 3 6 ]
```

2×3行列が3×2行列になることに注目してください。

### 6.2 実装

```forth
\ 指定列の全要素を取得
: MAT-GET-COLS-ALL ( addr col-index -- a1n a2n a3n ... amn )
    OVER MAT-ROWS       \ 行数を取得（ループ回数）
    0 DO
        2DUP I SWAP MAT-GET-CELL    \ A[I][col-index]の要素を取得
        -ROT                        \ 取得した値をスタックの下部へ移動
    LOOP
    DROP DROP                       \ addrとcol-indexを削除
;


\ 転換行列
: MAT-TRANSPOSE ( addr -- a1 a2 ... an )
    { addr }
    addr MAT-COLS           \ 列数分ループ
    0 DO
        addr I MAT-GET-COLS-ALL  \ 列の全要素を取得
    LOOP
;
```

```forth:使用例
2 3 MAT-CREATE M1
3 2 MAT-CREATE MT    \ 転置用の行列

1 2 3 4 5 6 M1 MAT-SET

M1 MAT-SHOW CR
\ [ 1 2 3 ]
\ [ 4 5 6 ]

M1 MAT-TRANSPOSE
MT MAT-SET
MT MAT-SHOW
\ [ 1 4 ]
\ [ 2 5 ]
\ [ 3 6 ]
```

## コード

```
\ 行列データ構造: [行数][列数][データ...]
0 CONSTANT ROW-OFFSET      \ 行数は先頭(オフセット0)
1 CELLS CONSTANT COL-OFFSET \ 列数は1セル目
2 CELLS CONSTANT DATA-OFFSET \ データは2セル目から

\ 行列を作成(行数 × 列数の領域を確保)
: MAT-CREATE ( rows cols "name" -- )
    CREATE
        OVER ,              \ 行数を格納
        DUP ,               \ 列数を格納
        * 2 + CELLS ALLOT   \ データ領域を確保
    DOES> ( -- addr )       \ 実行時は行列のアドレスを返す
;

\ 行数を取得
: MAT-ROWS ( addr -- rows )
    @ ;  \ 先頭セルの値を読む

\ 列数を取得
: MAT-COLS ( addr -- cols )
    CELL+ @ ;  \ 1セル進んでから読む

\ 総要素数を取得
: MAT-SIZE ( addr -- size )
    DUP MAT-ROWS SWAP MAT-COLS * ;

\ 行列に値を設定
: MAT-SET ( a11 a12 ... ann addr -- )
	DUP MAT-SIZE DUP                   \ 要素数
	ROT DATA-OFFSET + SWAP 1- CELLS +  \ データ領域の最後
	SWAP 0 DO
		SWAP OVER !                    \ 値を書き込む
		1 CELLS -					   \ アドレスを1つ減らす
	LOOP 
    DROP
;
	
\ 行列を表示
: MAT-SHOW ( addr -- )
    DUP MAT-ROWS 0 DO           \ 各行について
        CR ." [ "               \ 行の開始
        DUP MAT-COLS 0 DO       \ 各列について
            DUP DATA-OFFSET +   \ データ領域
            J I OVER MAT-COLS * + CELLS + @  \ 要素取得
            . SPACE             \ 値を表示
        LOOP
        ." ]"                   \ 行の終了
    LOOP
    DROP ;

\ 行列を表示
: MAT-SHOW ( addr -- )
    DUP MAT-COLS        \ 列数
    OVER DATA-OFFSET +  \ データ領域
    ROT MAT-ROWS        \ 行数
	0 DO
        CR ." [ "       \ 行の開始
        OVER
		0 DO
			DUP @ .     \ 値を表示
			CELL+       \ 次のデータ領域へ
		LOOP
        ." ]"           \ 行の終了
	LOOP
    DROP DROP
;

\ ==========================
\ 2つの行列のサイズが一致するかチェック
: MAT-SIZE-MATCH? ( addr1 addr2 -- flag )
    OVER MAT-ROWS OVER MAT-ROWS = 
    ROT MAT-COLS ROT MAT-COLS = 
    AND ;

\ 行列の加算
: MAT-ADD ( addrA addrB -- )
    2DUP MAT-SIZE-MATCH? 0= IF
        2DROP
            CR ." Error: Matrix sizes do not match"
        EXIT
    THEN

    DUP MAT-SIZE        \ ループ回数
    ROT DATA-OFFSET +   \ 配列1のアドレス
    ROT DATA-OFFSET +   \ 配列2のアドレス
    ROT 0 DO
        OVER @          \ 配列1の値
        OVER @ +        \ 配列2の値
        ROT CELL+       \ 配列1のアドレスを次に進める
        ROT CELL+       \ 配列2のアドレスを次に進める
    LOOP
    DROP DROP
;

\ ==========================
\ 指定行の先頭アドレスを取得
: MAT-ROW-ADDR ( addr n -- addr' )
	OVER MAT-COLS *             \ 列数を掛けてオフセット計算
	CELLS +                     \ バイトオフセットに変換
	DATA-OFFSET +                      \ データ開始位置を加算
;

\ 指定位置のセルの値を取得
: MAT-GET-CELL ( addr m n -- value )
	ROT ROT MAT-ROW-ADDR        \ 行のアドレスを取得
	SWAP CELLS + @              \ 列オフセットを加算して値を取得
;


\ 行列の乗算 A × B
: MAT-MUL ( addr-A addr-B -- a11*b11+a12*b21 ... )
    { addr-A addr-B }
    addr-A MAT-COLS addr-B MAT-ROWS =
    0= IF
        CR ." Error: Matrix dimensions incompatible"
        EXIT
    THEN
    addr-A MAT-ROWS 0 DO                    \ C[I][J]の計算
        addr-B MAT-COLS 0 DO
            0                               \ 累積値の初期化
            addr-A MAT-COLS 0 DO            \ 内積計算
                addr-A K I MAT-GET-CELL     \ A[I][K]
                addr-B I J MAT-GET-CELL     \ B[K][J]
                * +                         \ 乗算して累積加算
            LOOP
        LOOP
    LOOP
;

\ ===================
\ 指定列の全要素を取得
: MAT-GET-COLS-ALL ( addr col-index -- a1n a2n a3n ... amn )
    OVER MAT-ROWS       \ 行数を取得（ループ回数）
    0 DO
        2DUP I SWAP MAT-GET-CELL    \ A[I][col-index]の要素を取得
        -ROT                        \ 取得した値をスタックの下部へ移動
    LOOP
    DROP DROP                       \ addrとcol-indexを削除
;


\ 転換行列
: MAT-TRANSPOSE ( addr -- a1 a2 ... an )
    { addr }
    addr MAT-COLS           \ 列数分ループ
    0 DO
        addr I MAT-GET-COLS-ALL  \ 列の全要素を取得
    LOOP
;


\ ===================

2 3 MAT-CREATE M1   \ 2行3列の行列M1を作成
CR M1 MAT-ROWS ." ROWS: " . \ 2
CR M1 MAT-COLS ." COLS: " . \ 3
CR M1 MAT-SIZE ." SIZE: " . \ 6

1 2 3 4 5 6 M1 MAT-SET
M1 MAT-SHOW CR

2 2 MAT-CREATE M1
2 2 MAT-CREATE M2
2 2 MAT-CREATE MR   \ 結果用の行列

1 2 3 4 M1 MAT-SET
5 6 7 8 M2 MAT-SET

M1 MAT-SHOW CR      \ [ 1 2 ] [ 3 4 ]
M2 MAT-SHOW CR      \ [ 5 6 ] [ 7 8 ]

M1 M2 MAT-ADD
MR MAT-SET
MR MAT-SHOW CR      \ [ 6 8 ] [ 10 12 ]

\ ====================
2 2 MAT-CREATE M1
2 3 MAT-CREATE M2
2 3 MAT-CREATE MR   \ 結果用の行列

1 2 4 -1 M1 MAT-SET
3 5 0 -1 0 1 M2 MAT-SET
M1 M2 MAT-MUL
MR MAT-SET
MR MAT-SHOW


1 3 mat-create m1
3 2 mat-create m2
1 2 mat-create ma
2 4 6 m1 mat-set
7 5 3 4 6 2 m2 mat-set
m1 m2 mat-mul
ma mat-set
ma mat-show CR

\ ======================

3 2 mat-create m1
2 3 mat-create m2
3 3 mat-create ma
1 2 3 1 3 2 m1 mat-set
1 3 5 2 4 1 m2 mat-set
m1 m2 mat-mul
ma mat-set
ma mat-show CR

\ ======================

3 2 mat-create m1
2 3 mat-create ma
1 2 3 4 5 6 m1 mat-set
m1 mat-show CR
m1 MAT-TRANSPOSE
ma mat-set
ma mat-show CR
```

