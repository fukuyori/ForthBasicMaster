# Lev 25 ー 数値積分と微分
数学の授業で習う「積分」と「微分」は、実は私たちの日常生活に深く関わっています。

**微分の例：**
- 自動車の **速度計**：位置の変化率（= 微分）を表示
- 気温の **変化速度**：「1時間あたり2度上昇」= 時間微分
- 株価の **トレンド**：上昇速度・下降速度

**積分の例：**
- 自動車の **走行距離**：速度を時間で積分
- 電気代の **総使用量**：電力（ワット）を時間で積分
- 降水量の **総計**：降雨強度を時間で積分

しかし、現実世界のデータは **離散的（とびとび）** です。

```
気温測定：
8:00 → 20℃
9:00 → 22℃
10:00 → 24℃
11:00 → 25℃

問題：8:00から11:00の間、気温はどう変化したか？
     総熱量（積分）はいくらか？
```

このように、**とびとびのデータから連続的な情報を推定する技術** が **数値積分** と **数値微分** です。

本章では、以下を学びます：

1. **数値積分の原理**：矩形・台形・シンプソン法
2. **数値微分の原理**：前進差分・中心差分
3. **FORTHによる実装**：明示的なデータフローで理解を深める
4. **誤差評価**：近似の精度を判定する方法
5. **実用応用**：外部ツールとの連携


## 1. 数値積分の基礎理論

### 1.1 積分とは何か？ ― 面積で理解する

積分は、**グラフの下の面積** として理解できます。

```
例：速度が v(t) = 2t [m/s] のとき、0秒から5秒までの移動距離は？

速度
↑
10|         ／
  |       ／
 5|     ／
  |   ／
  | ／
  +----------→ 時間
  0   2   5

面積 = ∫[0→5] 2t dt = [t²][0→5] = 25 m
```

しかし、**複雑な関数や測定データでは解析的に計算できない** ことが多々あります。

そこで、**面積を小さな図形で近似する** のが数値積分です。


### 1.2 矩形法（最もシンプルな近似）

**原理：** 区間を細かく分割し、各区間を **長方形** で近似（左リーマン和）

```
   f(x)
    ↑
 4  |  ■■
    |  ■■■■
 3  |  ■■■■■■
    |  ■■■■■■■■
 2  |  ■■■■■■■■■■
    |  ■■■■■■■■■■■■
 1  |  ■■■■■■■■■■■■■■
    +─────────────────→ x
    a              b

各長方形の面積を合計 ≈ 積分値
```

**公式：**

$$
\int_{a}^{b} f(x)\ dx \\approx\
h \\bigl[\ f(x_0) + f(x_1) + \cdots + f(x_{n-1}) \\bigr]
$$
$$
\text{ただし }\
h = \frac{b - a}{n} \quad (\text{刻み幅}) 
\qquad
x_i = a + i h \quad (\text{分割点})
$$


**長所：** 実装が簡単
**短所：** 精度が低い（誤差 O(h)）


### 1.3 台形法（矩形法の改良）

**原理：** 各区間を **台形** で近似

```
   f(x)
    ↑
 4  |  ／＼
    | ／  ＼
 3  |／    ＼／＼
    |      ／  ＼
 2  |    ／      ＼
    |  ／          ＼
    +─────────────────→ x
    a              b

台形の面積 = (上底 + 下底) × 高さ / 2
```

**公式：**

$$
\int_{a}^{b} f(x)\ dx\
\approx\
\frac{h}{2}
\left[f(a)+ 2f(x_1)+ 2f(x_2)+
  \cdots+ 2f(x_{n-1})+ f(b)
\right]
$$

**長所：** 矩形法より高精度（誤差 O(h²)）
**短所：** 曲線の近似がまだ粗い

### 1.4 シンプソン法（放物線近似）

**原理：** 3点ずつを **放物線（2次曲線）** で近似

```
   f(x)
    ↑
    |    ○
    |   ／＼
    |  ／  ＼
    | ／    ＼○
    |／      ＼
    ○────────○──→ x
    a   m    b

3点を通る放物線で近似
```

**公式（シンプソンの1/3公式）：**

$$
\int_{a}^{b} f(x)\ dx\
\approx\
\frac{h}{3}
\left[
f(x_0)+ 4f(x_1)+ 2f(x_2)+ 4f(x_3)+ \cdots+ 2f(x_{n-2})+ 4f(x_{n-1})+ f(x_n)
\right]
$$

係数パターン：1, 4, 2, 4, 2, ..., 4, 1

**重要な制約：** 分割数 n は **偶数** でなければならない

**長所：** 高精度（誤差 O(h⁴)）
**短所：** 偶数分割が必要、計算量がやや多い


### 1.5 精度比較の実例

関数 `f(x) = x²` を区間 `[0, 1]` で積分（理論値 = 1/3 ≈ 0.333333）

| 手法      | n=10    | n=100     | n=1000      | 誤差の次数 |
| ------- | ------- | --------- | ----------- | ----- |
| 矩形法     | 0.285   | 0.32835   | 0.3328335   | O(h)  |
| 台形法     | 0.335   | 0.33335   | 0.333333    | O(h²) |
| シンプソン法  | 0.33333 | 0.3333333 | 0.333333333 | O(h⁴) |

**観察：**
- 台形法：n を 10倍 → 誤差が 1/100（h² に比例）
- シンプソン法：n が小さくても高精度

## 2. 数値微分の基礎理論

### 2.1 微分とは何か？ ― 傾きで理解する

微分は、**グラフの傾き（変化率）** を表します。

```
例：位置 x(t) = t² のとき、t = 2 での速度は？

位置
 ↑
16|           ● ← x(4) = 16
  |
 9|      ● ← x(3) = 9
  |
 4|   ● ← x(2) = 4
  |
 1| ● ← x(1) = 1
  +──────────→ 時間
  0  1  2  3  4

t = 2 での傾き（速度）= dx/dt = 2t = 4 m/s
```

しかし、測定データは **離散点** しかありません。そこで、**近くの点から傾きを推定** します。


### 2.2 前進差分法（最もシンプルな近似）

**原理：** 現在の点と次の点から傾きを計算

```
   f(x)
    ↑
    |      ●
    |     ／
    |    ／ ← この傾きを計算
    |   ／
    |  ●
    +──────→ x
    x  x+h

傾き ≈ [f(x+h) - f(x)] / h
```

**公式：**

$$
f'(x) \approx \frac{f(x+h) - f(x)}{h}
$$

**長所：** 実装が簡単
**短所：** 精度が低い（誤差 O(h)）


### 2.3 中心差分法（対称性を利用した高精度法）

**原理：** 前後の点から傾きを計算（対称性で誤差キャンセル）

```
   f(x)
    ↑
    |    ●
    |   ／＼
    |  ／  ＼ ← 中心の傾き
    | ／    ＼
    |●      ●
    +──────────→ x
   x-h  x  x+h

傾き ≈ [f(x+h) - f(x-h)] / (2h)
```

**公式：**

$$
f'(x) \approx \frac{f(x+h) - f(x-h)}{2h}
$$

**長所：** 前進差分より高精度（誤差 O(h²)）
**短所：** 端点（境界）では使えない


### 2.4 精度比較の実例

関数 `f(x) = sin(x)` を `x = π/4` で微分（理論値 = cos(π/4) ≈ 0.707107）

| 手法    | h=0.01   | h=0.001    | h=0.0001     | 誤差の次数  |
| ----- | -------- | ---------- | ------------ | ------ |
| 前進差分法 | 0.702081 | 0.706605   | 0.707057     | O(h)   |
| 中心差分法 | 0.707106 | 0.7071067  | 0.707106781  | O(h²)  |

**観察：**
- 中心差分法は同じ h で 10〜100倍 高精度
- 前進差分：h を 1/10 → 誤差が 1/10
- 中心差分：h を 1/10 → 誤差が 1/100


## 3. FORTHによる実装

### 3.1 Phase 1：基本的な台形法

まず、最もシンプルな台形法から実装します。

```forth
: f-square ( f: x -- f: x^2 )
   FDUP F* ;

: trapezoid 
   { XT: xt F: a F: b n -- F: integral }

   b a F- n S>F F/      \ 刻み幅 
   a xt                 \ 最初の点 f(a)

   \ 中間点 i = 1 ～ n-1 の和（係数 2）
   n 0 DO
      FOVER I S>F F* a F+ xt 2.0e F*
      F+
   LOOP

   b xt F+              \ 終点 f(b) を加える
   FSWAP 2.0e F/ F*     \ 結果を返す
;
```

```forth:使用例
\ f(x) = x² を [0, 1] で積分（理論値 = 1/3）
' f-square 0e 1e 10 trapezoid F.      \ 出力: 0.335
' f-square 0e 1e 100 trapezoid F.     \ 出力: 0.33335
' f-square 0e 1e 1000 trapezoid F.    \ 出力: 0.3333335
```

参考として、Pythonでのプログラムは以下の様になります。

```Python
def f(x):
    return x * x  # 例: f(x) = x^2

def trapezoid(a, b, n):
    h = (b - a) / n            # 刻み幅 h を計算
    total = f(a)               # 最初の点 f(a) を加える
    # 中間点 i = 1 ～ n-1 の和（係数 2）
    for i in range(1, n):
        total += 2 * f(a + i * h)
    total += f(b)              # 終点 f(b) を加える
    return (h / 2) * total     # 台形法の公式で結果を返す

## 使用例
result = trapezoid(0.0, 1.0, 1000)
print("result =", result)
```

#### 実行トークンとラムダ式

関数の実行トークン(xt)を使うことで、本体を書き換えることなく、使用する関数を切り替えることができます。独自の関数を作らずに、SINなどの関数を直接呼び出すこともできます。

```
' FSIN 0e 1e 10 trapezoid F.  \ → 0.459314548857976
```

また、`:NONAME` を使うと、関数をラムダ式（無名関数）として書くこともできます。

```
:NONAME FDUP F* ; 0e 1e 10 trapezoid F.  \ → 0.335
```

### 3.2 Phase 2：シンプソン法（高精度版）

```forth
\ ========================================
\ シンプソン法による数値積分
\ ========================================
: f-square ( f: x -- f: x^2 )
   FDUP F* ;

: even? ( n -- flag )
	2 MOD 0= ;

: simpson
	{ XT: xt F: a F: b n -- F: integral }

	n even? 0= IF 
		CR ." ERROR: Simpson's rule requires even n" 
		EXIT
	THEN


	b a F- n S>F F/					\ 刻み幅(h)
	a xt b xt F+					\ 両端の係数は１

	n 1 DO
		FOVER I S>F F* a F+ xt 4.0e F*	\ f(a + i * h) * 4
		F+ 								\ スタックの値と合算
	2 +LOOP

	n 2 DO
		FOVER I S>F F* a F+ xt 2.0e F*	\ f(a + i * h) * 2
		F+ 								\ スタックの値と合算
	2 +LOOP

   FSWAP 3.0e F/ F*     \ 結果を返す (h / 2) * 合算値	
;
```

```forth:使用例
' f-square 0e 1e 10 simpson F.     \ 出力: 0.333333
' f-square 0e 1e 100 simpson F.    \ 出力: 0.3333333333
```

**エラー処理の例：**
```forth
' f-square 0e 1e 9 simpson    \ ERROR: Simpson's rule requires even n
```

Pythonでは以下の通り。

```python
def f(x):
    return x * x

def simpson(a, b, n):
    """
    シンプソン法による数値積分
    a: 開始点
    b: 終点
    n: 分割数（偶数である必要がある）
    """
    if n % 2 != 0:
        raise ValueError("n は偶数である必要があります (n must be even).")

    h = (b - a) / n           # 刻み幅
    total = f(a) + f(b)       # 両端の係数は 1

    # 奇数番目の項（係数 4）
    for i in range(1, n, 2):
        total += 4 * f(a + i * h)

    # 偶数番目の項（係数 2）
    for i in range(2, n, 2):
        total += 2 * f(a + i * h)

    return (h / 3) * total    # シンプソン法の公式

## 使用例
result = simpson(0.0, 1.0, 1000)
print("result =", result)
```


### 3.3 Phase 3：数値微分（前進差分法）

```forth
: f-sin ( F: x -- F: x' )
	FSIN
;

: f-square ( F: x -- F: x' )
	FDUP F*
;

: forward-diff
	{ XT: xt F: x F: h -- F: derivative }

	\ sin(x + h)
	x h F+ xt

	\ sin(x)
	x xt

	\ (f(x + h) - f(x)) / h
	F- h F/ 
;
```

**使用例：**
```forth
' f-square 1.0e 1e-5 forward-diff F.          \ → 2.00001000001393 
' f-sin 0.7853981634e 0.01e forward-diff F.   \ → 0.703559491687389 
' f-sin 0.7853981634e 0.001e forward-diff F.  \ → 0.70675310997248
```

### 3.4 Phase 4：中心差分法（高精度版）

```forth
: f-sin ( F: x -- F: x' )
	FSIN
;

: central-diff
   { XT: xt F: x F: h -- F: derivative }

   \ f(x + h)
   x h F+ xt
   
   \ f(x - h)
   x h F- xt
   
   \ [f(x+h) - f(x-h)] / (2h)
   F- 2.0e h F* F/
;
```

**使用例：**
```forth
' f-sin 0.7853981634e 0.01e central-diff F.   \ → 0.707094996130653
' f-sin 0.7853981634e 0.001e central-diff F.  \ →  0.707106663333623 
```

## 4. 誤差評価と収束判定

### 4.1 Richardson の外挿法（理論）

分割数を 2倍にしたとき、誤差がどう変化するかで精度を推定できます。

**台形法の場合（誤差 O(h²)）：**

$$
I_{\text{true}}
\\approx\
I(h)
\+\
\frac{ I(h) - I(2h) }{3}
$$


ここで：
- `I_true`：真の積分値の推定
- `I(h)`：刻み幅 h での計算値
- `I(2h)`：刻み幅 2h での計算値

### 4.2 収束判定の実装

```forth
: f-square ( f: x -- f: x^2 )
   FDUP F* ;

: trapezoid 
    { XT: xt F: a F: b n -- F: integral }
    b a F- n S>F F/         	\ h = (b-a)/n 刻み幅
    a xt                    	\ sum = f(a) 初項
    
    n 1 > IF                	\ n > 1 のときのみ中間点を計算
        n 1 DO              	\ i = 1 to n-1（中間点 n-1 個）
            FOVER I S>F F* a F+ xt          \ f(a + i*h)
			2.0e F* F+                      \ sum += 2*f(xi)
        LOOP
    THEN
    
    b xt F+                 \ sum += f(b) 終項
    FSWAP 2.0e F/ F*        \ 結果 = (h/2) * sum
;

: adaptive-trapezoid
	{ xt F: a F: b F: tol -- F: integral F: error }
    10                          		\ 初期分割数 n=10
    xt a b OVER trapezoid       		\ T(n) を計算
    
    20 0 DO                     		\ 最大20回反復
        2*                      		\ n = n * 2（分割数を2倍に）
        xt a b OVER trapezoid   		\ (n) F:(T_prev T_current)
        
        \ Richardson外挿: R = T_current + (T_current - T_prev) / 3
        FOVER FOVER F- 3.0e F/  		\ (n) F:(T_prev T_current diff/3)
        FOVER F+                		\ (n) F:(T_prev T_current R)
        
        \ 誤差推定: err = |T_current - T_prev| / 3
        FROT FOVER F- FABS 3.0e F/  	\ (n) F:(T_current R err)
        
        FDUP tol F< IF          		\ err < tol なら収束
            CR ." Converged after " I 1+ . ." iterations" CR
            DROP FSWAP FDROP    		\ () F:(R err)
            UNLOOP EXIT
        THEN
        
        FDROP FSWAP FDROP       		\ (n) F:(T_current) 次の反復へ
    LOOP
    
    CR ." Warning: Did not converge" CR
;
```

```FORTH:実行例
\ f(x)=x², 0〜1, tol=1e-6
' f-square 0e 1e 1e-6 adaptive-trapezoid F. F.
\ 出力
\ Converged after 8 iterations
\ 誤差: 0.000000457190004058787
\ 積分値: 0.333333358764648

\ sin(x), 0〜π, tol=1e-6
' FSIN 0e PI 1e-6 adaptive-trapezoid F. F.
\ 出力
\ Converged after 5 iterations
\ 誤差： 0.000000526120657854771
\ 積分地： 1.99999998431269
```

## 5. 実用応用：外部ツールとの連携

### 5.1 CSV出力による可視化準備



```forth
\ ========================================
\ 積分結果のCSV出力
\ ========================================
\ ========================================
\ CSVファイル出力用ワード
\ ========================================

\ CSVヘッダー行をファイルに書き込む
: output-csv-header ( fileid -- )
   S" n,integral,error" ROT WRITE-LINE DROP ;

\ CSV形式で1行のデータをファイルに書き込む
\ 形式: "n,integral,error"
: output-csv-line
    { F: integral F: error n fileid -- }
    
    \ n（整数）を文字列に変換してファイルに書き込む
    n 0 <<# [CHAR] , HOLD #S #> #>> fileid WRITE-FILE DROP
    
    \ integral（浮動小数点）を文字列に変換してファイルに書き込む
    integral 10 10 0 F>STR-RDP fileid WRITE-FILE DROP
    s" ," fileid WRITE-FILE DROP
    
    \ error（浮動小数点）を文字列に変換してファイルに書き込む
    error 10 10 0 F>STR-RDP fileid WRITE-LINE DROP
;

\ 台形則を使った数値積分の結果をCSVファイルに出力
: adaptive-trapezoid-csv
    { xt F: a F: b -- }
    
    \ CSVファイルを作成（書き込みモード）
    s" integration_results.csv" W/O CREATE-FILE THROW
    
    \ ヘッダー行を書き込む
    DUP output-csv-header
    
    \ 10回ループ（分割数: 10, 20, 30, ..., 100）
    10 0 DO
        \ 台形則で積分を計算
        \ 分割数 n = (I+1) * 10
        xt a b I 1+ 10 * trapezoid      \ ( fileid ) F:( integral )
        
        \ 誤差を計算（厳密解 1/3 との差の絶対値）
        FDUP 0.333333e F- FABS          \ ( fileid ) F:( integral error )
        
        \ CSV行を出力
        \ n = (I+1)*10, integral, error
        I 1+ 10 * OVER output-csv-line  \ ( fileid ) F:( integral )
    LOOP
    
    \ ファイルを閉じる
    CLOSE-FILE DROP
;
```

```forth:実行例
' f-square 0.0e 1.0e adaptive-trapezoid-csv
```

**出力例（CSV）：**
```csv:integration_results.csv
n,integral,error
10,3.35000E-1,1.66700E-3
20,3.33750E-1,4.17000E-4
30,3.33519E-1,1.85519E-4
40,3.33438E-1,1.04500E-4
50,3.33400E-1,6.70000E-5
60,3.33380E-1,4.66296E-5
70,3.33367E-1,3.43469E-5
80,3.33359E-1,2.63750E-5
90,3.33354E-1,2.09095E-5
100,3.33350E-1,1.70000E-5
```

#### 参考：数値の文字列への変換
https://qiita.com/spumoni/private/589c3ac4fd4687c27707#%E6%B5%AE%E5%8B%95%E5%B0%8F%E6%95%B0%E7%82%B9%E6%95%B0%E3%81%AE%E6%9B%B8%E5%BC%8F%E8%A8%AD%E5%AE%9A


### 5.2 Pythonによる可視化

```python
import pandas as pd
import matplotlib.pyplot as plt

## CSVを読み込み
data = pd.read_csv('integration_results.csv')

## n 列の文字列から改行・空白を除去して整数化
data['n'] = data['n'].astype(str).str.strip().astype(int)

## error 列も安全のため float 化
data['error'] = data['error'].astype(str).str.strip().astype(float)

## 誤差の収束をプロット
plt.figure(figsize=(10, 6))
plt.loglog(data['n'], data['error'], 'o-', label='Trapezoidal error')

## 参照線 O(h^2)
plt.loglog(data['n'], 1.0 / (data['n']**2), '--', label='O(h²) reference')

plt.xlabel('Number of divisions (n)')
plt.ylabel('Error')
plt.title('Convergence of Trapezoidal Rule')
plt.legend()
plt.grid(True)
plt.savefig('convergence.png')
plt.show()
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/864075/27b581fd-86a4-47d7-ba91-93d8f5403e54.png)

このpythonのスクリプトを view.py として保存し、gforthのスクリプトに `s" python3 view.py" SYSTEM` として実行させると、データ出力後にグラフが表示されるようになる。しかし、pythonが仮想環境を使わないとpip installでライブラリが追加できなかったり、WindowsだとX serverが必要だったり、設定が面倒かもしれません。

