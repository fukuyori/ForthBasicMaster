# Lev 22 ー 統計演算
本回では、浮動小数点演算とループ制御を組み合わせ、データ配列から**平均・分散・標準偏差**を求める方法を学ぶ。統計量の計算はFORTHのような低レベル言語でも十分に可能であり、データの走査・加算・除算・平方根などの基本演算の組み合わせで実現できる。

## 配列データの準備

まず、サンプルデータを格納する領域を確保する。
ここでは浮動小数点配列を定義する。

```forth
CREATE data 10 CELLS ALLOT   \ 10要素分の領域を確保
```

値を初期化する場合は、`F!`を用いて格納する。

```forth
0e data     F!
1e data 1 CELLS + F!
2e data 2 CELLS + F!
3e data 3 CELLS + F!
4e data 4 CELLS + F!
5e data 5 CELLS + F!
6e data 6 CELLS + F!
7e data 7 CELLS + F!
8e data 8 CELLS + F!
9e data 9 CELLS + F!
```

## 平均値（mean）の計算

平均値は次式で求められる。

$$
\bar{x} = \frac{1}{N} \sum_{i=1}^{N} x_i
$$

FORTHでは、`DO LOOP`を使って総和を求め、要素数で割る。

```forth
: fmean ( addr n -- f: mean )
   TUCK                     \ ( n addr n ) ループ内でaddrを使い、最後にnを残す
   0e                       \ fsum 初期化
   0 DO
      I CELLS OVER + F@     \ i番目の値を取得 (addr + i*cell)
      F+                    \ 累積和を加算
   LOOP
   DROP                     \ addrを捨てて ( n fsum )
   S>F F/ ;                 \ nをfloatに変換して除算 → 平均値
```

```forth:使用例
data 10 fmean f.  \ → 4.5
```

## 分散（variance）の計算

分散は、平均からの偏差の二乗平均である。

$$
s^2 = \frac{1}{N} \sum_{i=1}^{N} (x_i - \bar{x})^2
$$

```forth
: fvariance ( addr n -- f: var )
   TUCK                        \ ( n addr n )
   2DUP fmean                  \ ( n addr n ) ( f: mean )
   0e                          \ fsum初期化 ( n addr n ) ( f: mean 0e )
   0 DO
      I CELLS OVER + F@        \ i番目の値を取得
      2 FPICK F-               \ meanを取り出し ( x_i - mean )
      FDUP F*                  \ 偏差^2
      F+                       \ 累積
   LOOP
   DROP                        \ addr を捨てる
   FNIP                        \ meanを捨てる ( fsum のみ残す )
   S>F F/ ;                    \ nで割って分散を算出
```

```forth:使用例
data 10 fvariance f.  \ → 8.25
```

## 標準偏差（standard deviation）の計算

標準偏差は分散の平方根である。

$$
\sigma = \sqrt{s^2}
$$

```forth
: fstddev ( addr n -- f: sd )
   fvariance fsqrt ;
```

```forth:使用例
data 10 fstddev f.  \ → 2.87228132326901
```

### 5. 応用：任意データ入力と統計出力

実際のデータを読み取り、平均・標準偏差を表示する例。

```forth
: input-data ( addr n -- )
   PAD 80 ERASE
   0 DO
      CR ." data[" I . ." ] = " 
      PAD 80 ACCEPT
      PAD SWAP EVALUATE S>F
      DUP I CELLS + F!
   LOOP DROP ;

: show-stats ( addr n -- )
   CR
   2DUP fmean ." 平均 = " f.
   2DUP fvariance ." 分散 = " f.
   fstddev ." 標準偏差 = " f. ;
```

```:使用例
data 5 input-data    \ ← 1 2 3 4 5
data 5 show-stats    \ → 平均 = 3. 分散 = 2. 標準偏差 = 1.4142135623731
```

入力したデータの平均・分散・標準偏差が順に表示される。

