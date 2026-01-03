# Lev 29 ー CSVデータ入力・出力と標準入力・出力
第19回「ファイル入出力とデータ記録、第20回「数の書式設定」、第25回「数値積分と微分」でも触れたファイルへの出力について、もう一度整理する。

https://gforth.org/manual/Formatted-numeric-output.html

## 1. ファイル操作の基礎

### 1.1 操作

FORTHでファイルを扱うには、以下のワードを使う。

| ワード | 機能 | スタック効果 |
|--------|------|--------------|
| `CREATE-FILE` | ファイル作成 | ( c-addr u fam -- fileid ior ) |
| `OPEN-FILE` | ファイルオープン | ( c-addr u fam -- fileid ior ) |
| `CLOSE-FILE` | ファイルクローズ | ( fileid -- ior ) |
| `WRITE-LINE` | 1行書き込み | ( c-addr u fileid -- ior ) |
| `READ-LINE` | 1行読み込み | ( c-addr u fileid -- u2 flag ior ) |

### 1.2. ファイルアクセスモード

| モード | 意味 | 使用例 |
|--------|------|--------|
| `R/O` | Read Only（読み取り専用） | データ読み込み |
| `W/O` | Write Only（書き込み専用） | データ出力 |
| `R/W` | Read/Write（読み書き） | データ更新 |

## 2. ファイル入力

### 2.1.  文字列を1行読みだすワード

```
: get-line ( fileid -- addr u flag )
	PAD 256 ROT READ-LINE THROW
	PAD -ROT
;
```

```:例
: show-file ( -- )
	S" test.txt" R/O OPEN-FILE THROW
	BEGIN
		DUP get-line
		-ROT TYPE CR
		0=
	UNTIL
	CLOSE-FILE THROW
;
```

### 2.2. CSVファイルを読み込む

サンプルファイルとして、以下のデータの入った`score.csv`というファイルを使用する。

```:score.csv
名前,国語,数学,英語,理科,社会
田中,78,68,40,90,72
鈴木,90,55,60,75,85
佐藤,60,80,70,80,65
加藤,72,65,55,60,70
中村,85,72,68,88,75
小林,55,45,50,60,58
吉田,88,78,82,90,80
山本,70,60,52,65,68
斎藤,92,70,75,85,88
高橋,65,50,48,55,60
```

読み込んだデータを`,`で切り分ける。

```
\ 1行読み込み
: get-line ( fileid -- addr u flag )
	PAD 256 ROT READ-LINE THROW
	PAD -ROT
;

\ , で区切りスタックへ
: split-part ( addr u -- addr1 u1 addr2 u2 )
	DUP 0> IF
		',' $SPLIT RECURSE
	THEN
;

\ 1行分出力
: type-part ( addr1 u1 ... addr6 u6 -- )
	DEPTH 2 / 0 DO
		TYPE .\" \t"  \ 1つ出力してタブ
	LOOP
	.\" \n"  \ レコード終わりに改行
;

\ 反転
: reverse
	DEPTH 2 / 1 DO
		I 2* 1+ ROLL
		I 2* 1+ ROLL
	LOOP
;

: show-file ( -- )
	S" test.csv" R/O OPEN-FILE THROW { fileid }
	BEGIN
		fileid get-line 0= { EOF }
		split-part
		DROP DROP
		EOF FALSE = IF 
			reverse
		 	type-part 
		THEN
		EOF
	UNTIL
	fileid CLOSE-FILE THROW
;

show-file
```

```:出力例
名前        国語    数学    英語    理科    社会
田中    78      68      40      90      72
鈴木    90      55      60      75      85
佐藤    60      80      70      80      65
加藤    72      65      55      60      70
中村    85      72      68      88      75
小林    55      45      50      60      58
吉田    88      78      82      90      80
山本    70      60      52      65      68
斎藤    92      70      75      85      88
高橋    65      50      48      55      60
```

## 3. ファイル出力

### 3.1. 文字列を1行書き出すワード

```forth
: write-line ( addr u fileid -- )
    WRITE-LINE THROW ;
```

```forth:例
S" test.txt"  W/O CREATE-FILE  THROW        \ ファイルを開く。スタックにファイルIDが入る
DUP S" Hello, World!" ROT WRITE-LINE THROW  \ 文字を書き込む
CLOSE-FILE                                  \ ファイルを閉じる
```

解説。
- `addr u` : 書き込む文字列のアドレスと長さ
- `fileid` : ファイルハンドル（識別番号）
- `THROW` : エラーがあれば例外を発生させる

### 3.2. シングル整数の出力

```
\ 符号なしシングル整数（最後に改行）
: write-unsigned-single ( fileid n -- )
	0 <<# #S #> #>> ROT WRITE-LINE THROW
;

\ 符号ありシングル整数（最後に改行）
: write-single ( filedid n -- )
	S>D     	\ 符号ありシングルを符号ありダブルに (-123)→(-123 -1)
	SWAP OVER  	\ (-1 -123 -1)
	DABS      	\ 符号ありダブルから符号なしダブルへ (-1 123 0)
	<<# #S ROT SIGN #> #>>
	ROT WRITE-LINE THROW
;

\ 符号ありシングル整数（最後にカンマ）
: write-single-with-comma ( filedid n -- )
	S>D     	\ 符号ありシングルを符号ありダブルに (-123)→(-123 -1)
	SWAP OVER  	\ (-1 -123 -1)
	DABS      	\ 符号ありダブルから符号なしダブルへ (-1 123 0)
	<<# [CHAR] , HOLD #S ROT SIGN #> #>>  \ カンマ付き
	ROT WRITE-FILE THROW
;

: test 
	S" test-single.txt"  W/O CREATE-FILE  THROW
    DUP S" value1, value2, value3" ROT WRITE-LINE THROW
    
	DUP 1234 write-unsigned-single 
	DUP -5678 write-single
    
    \ CSV形式
	DUP 1234 write-single-with-comma
	DUP -5678 write-single-with-comma
	DUP 90 write-single
	CLOSE-FILE 
;
```

```:出力例:test-single.txt
value1, value2, value3
1234
-5678
1234,-5678,90
```

### 3.3. ダブル整数の出力

`<#` '#>` 書式は、符号なしダブル整数を対象としているので、シングルより手順は少なくなる。

```
\ 符号なしダブル整数（最後に改行）
: write-unsigned-double ( fileid d -- )
	 <<# #S #> #>> ROT WRITE-LINE THROW
;

\ 符号ありダブル整数（最後に改行）
: write-double ( filedid d -- )
	SWAP OVER  	\ (-123 -1)→(-1 -123 -1)
	DABS      	\ 符号ありダブルから符号なしダブルへ (-1 123 0)
	<<# #S ROT SIGN #> #>>
	ROT WRITE-LINE THROW
;

\ 符号ありダブル整数（最後にカンマ）
: write-double-with-comma ( filedid d -- )
	SWAP OVER  	\ (-1 -123 -1)
	DABS      	\ 符号ありダブルから符号なしダブルへ (-1 123 0)
	<<# [CHAR] , HOLD #S ROT SIGN #> #>>  \ カンマ付き
	ROT WRITE-FILE THROW
;

: test 
	S" test-double.txt"  W/O CREATE-FILE  THROW
    DUP S" value1, value2, value3" ROT WRITE-LINE THROW
    
	DUP #1234. write-unsigned-double
	DUP #-5678. write-double
    
    \ CSV形式
	DUP #1234. write-double-with-comma
	DUP #-5678. write-double-with-comma
	DUP #90. write-double
	CLOSE-FILE 
;
```

```:出力例:test-double.txt
value1, value2, value3
1234
-5678
1234,-5678,90
```

### 3.4. 浮動小数点数の出力

Gforthでは、浮動小数点数の変換には、`F.RDP` を使う。このワードは、ANS Forth標準には存在せず、Gforth固有ワードとなる。

f.rdp, f>str-rdp, f>buf-rdp の3ワードは、整数スタック（S:）上で共通の3つのパラメータ（+nr +nd +np）を使って書式を指定する。

- +nr (Right): 出力の総フィールド幅。変換後の文字列はこの幅で右寄せされる
- +nd (Digits): 固定小数点表記が選択された場合の、小数点以下の桁数
- +np (Precision/Minimum): 最低有効桁数

```
\ 浮動小数点数をファイルに書き込み（改行付き）
: write-float ( fileid F: n -- )
	10 { width }
	PAD width 5 0 F>BUF-RDP         \ PADに浮動小数点を文字列化
	PAD width ROT WRITE-LINE THROW  \ ファイルに書き込み+改行
;

\ 浮動小数点数をファイルに書き込み（カンマ付き、改行なし）
: write-float-with-comma ( fileid F: n -- )
	10 { width }
	PAD width 5 0 F>BUF-RDP         \ PADに浮動小数点を文字列化
	s" ," PAD width + SWAP CMOVE    \ 末尾にカンマを追加
	PAD width 1+ ROT WRITE-FILE THROW  \ ファイルに書き込み（改行なし）
;

: test
	S" test-float.txt" W/O CREATE-FILE THROW  \ ファイルを書き込みモードで作成
	DUP S" value1, value2, value3" ROT WRITE-LINE THROW  \ ヘッダ行
	DUP 123.4567890123e write-float           \ 単独の値（改行付き）
	DUP -123.456e write-float                 \ 単独の値（改行付き）
	\ CSV形式の行
	DUP 1.2e write-float-with-comma           \ 1列目
	DUP -3.4e write-float-with-comma          \ 2列目
	DUP 567.890123e write-float               \ 3列目（改行付き）
	CLOSE-FILE THROW                          \ ファイルを閉じる
;
```

```:test-float.txt
value1, value2, value3
 123.45679
-123.45600
   1.20000,  -3.40000, 567.89012
```

### 3.5.  正弦波データを CSV に出力


```forth
\ 浮動小数点数をカンマ付きで書き込み（改行なし）
: write-float-with-comma ( fileid F: n -- )
	10 { width }
	PAD width 5 0 F>BUF-RDP         \ PADに浮動小数点を文字列化
	s" ," PAD width + SWAP CMOVE    \ 末尾にカンマを追加
	PAD width 1+ ROT WRITE-FILE THROW  \ ファイルに書き込み（改行なし）
;

\ 浮動小数点数をファイルに書き込み（改行付き）
: write-float ( fileid F: n -- )
	10 { width }
	PAD width 5 0 F>BUF-RDP         \ PADに浮動小数点を文字列化
	PAD width ROT WRITE-LINE THROW  \ ファイルに書き込み+改行
;

\ x, y をカンマ区切りで書き込むワード
: write-csv ( fileid F: x y -- )
	{ fid }                         \ fileidをローカル変数に保存
	FSWAP                           \ F: y x → x y
	fid write-float-with-comma      \ x をカンマ付きで書き込み（xを消費）
	fid write-float                 \ y を改行付きで書き込み（yを消費）
;

\ 正弦波データを生成
: generate-sine ( -- )
	\ ファイルを作成してハンドルを保存
	S" sine.csv" W/O CREATE-FILE THROW { csv-fid }
	csv-fid S" x,sin(x)" ROT WRITE-LINE THROW  \ ヘッダー行を書き込み
	0e                      \ x = 0.0 からスタート
	BEGIN
		FDUP 10e F<=        \ x <= 10.0 の間ループ
	WHILE
		FDUP FDUP           \ x x x
		FSIN                \ x x sin(x)
		csv-fid write-csv   \ x を残して x, sin(x) を書き込み
		0.1e F+             \ x = x + 0.1
	REPEAT
	FDROP                   \ 最後の x を破棄
	csv-fid CLOSE-FILE THROW   \ ファイルを閉じる
;

generate-sine
```


```csv:sine.csv
x,sin(x)
   0.00000,   0.00000
   0.10000,   0.09983
   0.20000,   0.19867
...
   9.80000,  -0.36648
   9.90000,  -0.45754
  10.00000,  -0.54402
```

## 4. 文字列を数値に変換する

### 4.1. 整数変換

`S>NUMBER?` は、文字列をダブル整数に変換し、最後に変換結果をスタックします。

```forth
: str>num? ( addr u -- )
    S>NUMBER? IF
        D.
    ELSE
        CR ." ERROR: Unable to convert."
    THEN
;
```

```forth:例
S" 123" str>num .  \ → 123
S" -123" str>num .  \ → -123
```

### 4.2. 浮動小数点変換

```forth
: str>float ( addr u -- )
    >FLOAT IF
        F.
    ELSE
        CR ." ERROR: Unable to convert."
    THEN
;        
```

```forth:例
S" 123.456" str>num .  \ → 123.456
S" -123.456" str>num .  \ → -123.456
```

## 5. CSVデータ変換処理

以下は、試験結果のCSVファイルを読み込み、個人毎の合計点と平均点を追加する処理。

```
\ ファイルから1行読み込み、PADに格納
: GetLine ( fileid -- addr u flag )
	PAD 256 ROT READ-LINE THROW  \ PADに最大256文字読み込み
	PAD -ROT                     \ アドレス、長さ、成功フラグを返す
;

\ カンマ区切りで文字列を分割し、再帰的にスタックに積む
: split-part ( addr u -- addr1 u1 addr2 u2 ... )
	DUP 0> IF
		',' $SPLIT RECURSE   \ カンマで分割し、残りを再帰処理
	THEN
;

\ 数値に変換可能な文字列を合計し、スタック順序を反転
: reverse 
	0 { num-count }          \ 数値カウンタを初期化
	0e                       \ 合計値を0.0で初期化
	DEPTH 2 / 0 DO
		I 2* 1+ ROLL         \ 文字列アドレスを取り出す
		I 2* 1+ ROLL         \ 文字列長を取り出す
		2DUP S>NUMBER? IF    \ 数値に変換可能か試行
			D>F F+           \ 浮動小数点に変換して合計に加算
			num-count 1+ TO num-count  \ カウント増加
		ELSE
			DROP DROP        \ 変換失敗時は破棄
		THEN
	LOOP
	num-count 0> IF          \ 数値が1つ以上あれば
		FDUP num-count S>F F/  \ 平均値を計算
	THEN
;

\ 1行分のデータをファイルに書き込み
: WriteLine 
	{ fileid header-flag -- }
	DEPTH 2 / 0 DO           \ スタック上の文字列ペアをすべて処理
		fileid WRITE-FILE THROW       \ データを書き込み
		S" ," fileid WRITE-FILE THROW \ カンマを追加
	LOOP
	header-flag IF           \ ヘッダー行の場合
		FDROP                \ 浮動小数点スタックをクリア
		s" 合計, 平均" fileid WRITE-LINE THROW  \ ヘッダーを追加
	ELSE                     \ データ行の場合
		FSWAP                \ 合計と平均の順序を入れ替え
		10 { width }
		PAD 25 ERASE         \ PADをクリア
		PAD width 0 0 F>BUF-RDP      \ 合計を文字列化
		PAD width fileid WRITE-FILE THROW
		S" ," fileid WRITE-FILE THROW
		PAD width 1 0 F>BUF-RDP      \ 平均を文字列化
		PAD width fileid WRITE-LINE THROW  \ 改行付きで書き込み
	THEN
;

\ CSVファイルを読み込み、各行の合計と平均を計算して新しいファイルに出力
: aggregate-csv ( -- )
	S" scores.csv" R/O OPEN-FILE THROW { r-fileid }       \ 入力ファイルを開く
	S" scores-aggrigation.csv" W/O CREATE-FILE THROW { w-fileid }  \ 出力ファイルを作成
	
	true { header-flag }     \ 最初の行はヘッダー
	BEGIN
		r-fileid GetLine 0= { EOF }  \ 1行読み込み、EOFフラグを設定
		split-part           \ カンマで分割
		DROP DROP            \ 最後の空の分割結果を削除
		EOF FALSE = IF       \ EOFでなければ
			reverse          \ 数値を合計・平均計算してスタック反転
			w-fileid header-flag WriteLine  \ ファイルに書き込み
			false TO header-flag  \ 2行目以降はヘッダーでない
		THEN
		EOF                  \ EOFまで繰り返し
	UNTIL
	r-fileid CLOSE-FILE THROW   \ 入力ファイルを閉じる
	w-fileid CLOSE-FILE THROW   \ 出力ファイルを閉じる
;

aggregate-csv
```

```:出力例
名前,国語,数学,英語,理科,社会,合計, 平均
田中,78,68,40,90,72,      348.,      69.6
鈴木,90,55,60,75,85,      365.,      73.0
佐藤,60,80,70,80,65,      355.,      71.0
加藤,72,65,55,60,70,      322.,      64.4
中村,85,72,68,88,75,      388.,      77.6
小林,55,45,50,60,58,      268.,      53.6
吉田,88,78,82,90,80,      418.,      83.6
山本,70,60,52,65,68,      315.,      63.0
斎藤,92,70,75,85,88,      410.,      82.0
高橋,65,50,48,55,60,      278.,      55.6
```

## 6. FORTH の標準入力・標準出力を使ったパイプ処理

### 6.1. 標準入出力とは

#### 標準入力（stdin）
- キーボードからの入力
- パイプ（`|`）からの入力
- リダイレクト（`<`）からの入力

#### 標準出力（stdout）
- 画面への出力
- パイプ（`|`）への出力
- リダイレクト（`>`）への出力

#### 標準エラー出力（stderr）
- エラーメッセージ専用の出力

### 6.2. FORTHでの標準入出力

gforthは標準入出力を完全サポートしている。

```:echo-lines.fs（標準入力から1行読み込んで標準出力に出力）
: ECHO-LINE ( -- FLAG )
  PAD 256 STDIN READ-LINE THROW  \ 標準入力から読み込み
  IF
    PAD SWAP STDOUT WRITE-LINE THROW  \ 標準出力に書き込み
    FALSE                        \ 継続
  ELSE
    DROP TRUE                    \ EOF
  THEN
;

\ すべての行をエコー
: ECHO-ALL ( -- )
  BEGIN
    ECHO-LINE
  UNTIL
;

ECHO-ALL
BYE
```

```:実行例
gforth echo-lines.fs < scores.csv
```

```:echo-chars.fs（標準入力から1文字読み込んで標準出力に出力）
: ECHO-CHAR ( -- FLAG )
  KEY?  IF                       \ 入力があるか確認
    KEY EMIT                     \ 1文字読んで出力
    FALSE
  ELSE
    TRUE
  THEN
;

: ECHO-ALL ( -- )
  BEGIN
    ECHO-CHAR
  UNTIL
;

ECHO-ALL
BYE
```

```:実行例
gforth echo-chars.fs < scores.csv
```

### 6.3. 数値を2倍にするフィルタ

```forth:double.fs
\ 文字列を数値に変換（変換失敗時は0を返す）
: str>number ( addr u -- n )
	S>NUMBER? IF        \ 変換成功
		D>S             \ ダブル数値を単精度に変換
	THEN                \ 失敗時はスタックに何も残らない（0相当）
;

\ 標準入力から1行読み込み、数値を2倍にして出力
: ECHO-LINE ( -- FLAG )
  PAD 256 STDIN READ-LINE THROW  \ 標準入力から読み込み
  IF                              \ 行が読めた場合
	PAD SWAP str>number           \ 文字列を数値に変換
	2 * DUP                       \ 2倍にしてコピー
	0 <<# #S #> #>> STDERR WRITE-LINE THROW  \ 標準エラー出力に書き込み
	. CR                          \ 標準出力に書き込み（改行付き）
    FALSE                         \ 継続フラグ
  ELSE
    DROP TRUE                     \ EOF（終了フラグ）
  THEN
;

\ 標準入力から数値を読んで2倍にするフィルター
: double-filter ( -- )
  BEGIN
    ECHO-LINE                     \ 1行処理
  UNTIL                           \ EOFまで繰り返し
;

double-filter                     \ フィルター実行
BYE                               \ gforthを終了```
```

```bash:実行例
## 1から5までの数を2倍にして、標準出力と標準エラー出力に出力
$ seq 1 5 | gforth double.fs 2> stderr.txt
2
4
6
8
10
```

