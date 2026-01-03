# Lev 21 ー 構造体とレコード設計
C言語での構造体（struct）に相当するものをFORTHで作るには、メモリ上のアドレス管理とオフセット計算を自分で設計する必要がある。

FORTHでは、すべてのデータは線形なメモリ空間に配置される。`CREATE` でその基点（ベースアドレス）を定義し、`ALLOT` で必要なバイト数を確保することで「構造体に相当する領域」を手作りできる。

## メモリとオフセットの基本

メモリ上の位置を相対的に示すため、FORTHでは「オフセット」を明示的に扱う。
例えば、整数を3つ格納するレコードを考える。

```forth
CREATE RECORD 3 CELLS ALLOT
```

この場合、`RECORD` は3つのセル（gforthでは通常 各8バイト）を占有するメモリ領域の先頭アドレスを返すワードとなる。

値を代入する場合は `!` を使う。

```forth
100 RECORD 0 CELLS + !
200 RECORD 1 CELLS + !
300 RECORD 2 CELLS + !
```

各項目へのアクセスはアドレス計算で行う。

```forth
RECORD 0 CELLS + @ . 
RECORD 1 CELLS + @ . 
RECORD 2 CELLS + @ .
```

このように、オフセットを自分で管理するのがFORTH流の「構造体」設計である。

## オフセット定義を抽象化する

オフセットを都度書くのは煩雑である。
定義済みの「フィールド名」を通してアクセスできるようにしよう。

```forth
0 CONSTANT ID
ID  CELL+ CONSTANT AGE
AGE CELL+ CONSTANT SCORE
```

これで、

```forth
RECORD ID    + @ .
RECORD AGE   + @ .
RECORD SCORE + @ .
```

のように、フィールド名で可読性を確保できる。

## CREATE … DOES\> で「構造体型」を作る

`CREATE` と `DOES>` を組み合わせると、新しい構造体型（テンプレート）を定義できる。

```forth:例：座標データ構造体 (x, y)
\ x y を受け取り、内部に格納する動作を持つ型
: COORDINATE
   CREATE 2 CELLS ALLOT
   DOES> ( x y -- ) \ 実行時、暗黙的にaddrが積まれる ( x y addr )
      ROT          ( y addr x )
      OVER         ( y addr x addr )
      !            ( y addr )  \ addr (オフセット0) に x を格納
      CELL+        ( y addr+cell )
      !            ( )         \ addr+cell (オフセット1) に y を格納
;
```

これで、以下のように使える。

```forth
COORDINATE pointA
3 4 pointA   ( x=3, y=4 を格納 )

\ 取り出し
' pointA @ .       \ x (3 が表示される)
' pointA CELL+ @ . \ y (4 が表示される)
```

この `DOES>` 部が「インスタンスを呼び出したときの動作」を定義している。この場合は「スタックからxとyを受け取り、インスタンスのメモリ領域に格納する」動作である。これにより、オブジェクト指向的な設計を可能にしている。

ここで重要なのは、pointAはワードであって変数ではないという点である。pointAを実行すると格納動作が走るため、単に値を取り出したいときは「そのワードのデータ領域アドレス」を明示的に取得する必要がある。`' pointA` は、「pointAのデータ領域の先頭アドレスをスタックに積む」操作である。このように、構造体のデータを直接読み取ることができる。

### 応用：学生レコードの定義

先に使用した複数のフィールドを持つレコードを例にする。

```forth
\ 学生データ構造体：ID, 年齢, 成績
CREATE A-STUDENT 3 CELLS ALLOT

0 CONSTANT ID
ID CELL+ CONSTANT AGE
AGE CELL+ CONSTANT SCORE

: .STUDENT ( addr -- )
   CR ." ID= " DUP ID + @  .  ." : "
   DUP AGE + @ . ." 歳 : 評価= "
   SCORE + @ . CR ;
```

使用例：
`A-STUDENT` は実行されると確保した領域の先頭アドレスをスタックに積む。

```forth:入力
123 A-STUDENT ID + !
20  A-STUDENT AGE + !
88  A-STUDENT SCORE + !

A-STUDENT .STUDENT
```

```:出力例
ID= 123 : 20 歳 : 評価= 88
```

さらに `DOES>` を組み合わせると、「フィールドアクセスワード」を自動生成することもできる。

```forth
\ n(オフセット) を受け取り、フィールド定義ワードを生成する
: FIELD: ( n -- )
   CREATE ,         \ コンパイル時: オフセットnを辞書に格納
   DOES> ( addr -- addr+offset ) \ 実行時:
      @ +             \ 格納されたオフセットnを読み出し、addrに加算する
;

\ 絶対オフセットでフィールドを定義する
0       FIELD: ID
CELL    FIELD: AGE     ( 1セル分のオフセット )
2 CELLS FIELD: SCORE   ( 2セル分のオフセット )

CREATE STUDENT 3 CELLS ALLOT
```

これにより、

```forth
123 STUDENT ID !
20  STUDENT AGE !
88  STUDENT SCORE !

STUDENT ID @ .    ( 123 )
STUDENT SCORE @ . ( 88 )
```

のように、まるで「構造体のメンバ変数」を扱うような書き方ができる。この方法は、レコード定義を複数種類用意する際に非常に有効である。

#### 7\. まとめ

  * `CREATE` と `ALLOT` でメモリ領域を確保し、構造体的に扱える。
  * `+` とオフセットを組み合わせることでフィールドアクセスを実現する。
  * `DOES>` により、構造体の“呼び出し動作”を定義できる（アドレスを返す、値を格納する、等）。
  * `FIELD:` のようなワードで汎用的な構造体テンプレートを作ることが可能である。

FORTHでは型システムが存在しない代わりに、自ら「構造」を定義し、その意味を付与する自由がある。それは、低レベルのメモリ操作と高レベルの抽象化が直結しているという、FORTHらしい強力な特徴である。

