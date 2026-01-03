# Lev 24 ー ニュートン法による根の探索
本章では、**ニュートン法（Newton-Raphson法）**をFORTHで実装し、非線形方程式の解（ゼロ点）を反復計算で求める方法を学ぶ。

## ニュートン法の原理

関数 ( f(x) = 0 ) の解を求めたいとき、初期値 ( x_0 ) から以下の式で反復する：

$$
x_{n+1} = x_n - \frac{f(x_n)}{f'(x_n)}
$$

この式を繰り返すことで、解 ( x ) に収束する。
例えば ( f(x) = x^2 - 2 ) の場合、解は ( \sqrt{2} ) である。

## FORTHでの実装方針

FORTHでは、関数やその導関数を別ワードとして定義し、`DO LOOP` による反復で更新を行う。

```forth:コード例：平方根を求めるニュートン法
\ ニュートン法による平方根の近似
\ f(x) = x^2 - a の根を求める

FVARIABLE a        \ 目標値 a
FVARIABLE x        \ 現在の近似値 x
VARIABLE iter      \ 反復回数

: f(x) ( f: x -- f: fx )  \ f(x) = x^2 - a
   FDUP F*                \ x^2
   a F@ F- ;              \ x^2 - a

: f'(x) ( f: x -- f: f'x ) \ f'(x) = 2x
   2e F* ;

: newton-step ( f: x -- f: x' )
   FDUP                   \ x x
   FDUP f(x)              \ x x f(x)
   FSWAP f'(x)            \ x f(x) f'(x)
   F/                     \ x f(x)/f'(x)
   F- ;                   \ x - f(x)/f'(x)

: newton ( f: x0 -- f: root )
   0 iter !               \ カウンタ初期化
   BEGIN
      FDUP x F!           \ 現在の値を保存
      newton-step         \ 次の値を計算
      iter @ 1+ iter !    \ 反復回数を更新
      FDUP x F@ F- FABS   \ |x_new - x_old|
      1e-10 F<            \ 収束判定
   UNTIL
   CR ." Root = " FDUP F.
   CR ." Iterations = " iter @ . ;
```

```:実行例
2e a F!     \ a = 2
1e newton   \ 初期値 x0 = 1
```

```:出力例
Root = 1.4142135623731
Iterations = 5
```

#### コード解説

| セクション         | 内容                     |
| ------------- | ---------------------- |
| `f(x)`        | 目的関数を定義。ここでは `x^2 - a` |
| `f'(x)`       | 導関数を定義。ニュートン法ではこれが必須   |
| `newton-step` | 1回分の反復を行うステップ          |
| `newton`      | ループで繰り返し、差分が閾値以下で停止    |
| `iter`        | 収束に要したステップ数を確認するための変数  |

#### 改良課題

**f’(x)が0に近い場合**にエラーを防ぐため、除算前にチェックを追加する。

```
: newton-step ( f: x -- f: x' )
   FDUP                   \ x x          ← 現在値を複製
   FDUP f(x)              \ x x f(x)     ← f(x) を計算
   FSWAP f'(x)            \ x f(x) f'(x) ← f'(x) を計算（スタック順整列）

   FOVER FABS 1E-12 F< IF \ f'(x) が極小なら発散を防ぐ
      ." Derivative too small!" CR
      FDROP FDROP FDROP
      0E EXIT
   THEN

   F/                     \ x f(x)/f'(x)
   F- ;                   \ x - f(x)/f'(x)
```

**最大反復回数**を設定し、収束しない場合に警告を出す。

```
1000 MAX-ITER !    \ 例：1000回まで
...
: newton ( f: x0 -- f: root )
     ...
      FDUP x F@ F- FABS   \ |x_new - x_old|
      1E-10 F<            \ 収束判定
      iter @ MAX-ITER @ >= OR  \ または回数上限
   UNTIL
   ...
```


**複数初期値**を試すように拡張し、局所解の違いを観察する。

