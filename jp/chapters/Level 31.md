# Lev 31 ー DSL構築 ― 科学計算ミニ言語の設計
## なぜ科学計算に独自言語が必要なのか

現代の科学計算、特に数値解析・最適化・統計的推定・シミュレーション・高精度モデリングなどを行う際、研究者はしばしば「思考対象」と「実装手続き」の間に生じる 抽象度のギャップ に悩まされます。

たとえば、あなたが「気温と売上の関係」を調べたいとします。頭の中では次のように考えます。

> 「気温が20度のとき、売上はいくらだろう？25度では？30度では？」

しかし、通常のプログラミング言語では、このシンプルな疑問を実現するために。

- 変数をどこに保存するか（メモリアドレス）
- どの順番で計算するか（制御フロー）
- データをどう受け渡すか（スタック操作）

といった 「コンピュータ内部の仕組み」 を意識しなければなりません。

これは、「日本語で考えていることを、機械語に翻訳する」作業に似ています。翻訳の手間が大きいほど、本来考えたいこと（気温と売上の関係）から遠ざかってしまいます。


本章の目的は、このギャップを埋めるために、科学計算専用 DSL（Domain Specific Language：領域特化言語） を構築することです。

> 「研究者や学生が、自分の考えをそのまま記述できる言語環境を提供する」

これは単なる「プログラミング教育」ではなく、人間の思考と計算機の橋渡し という課題への挑戦です。


## 1. 科学計算 DSL の概念的位置づけと要件思想

科学計算 DSL は、一般的なプログラミング言語と異なる設計思想を持ちます。

| 観点    | 通常のプログラミング言語           | 科学計算 DSL               |
| ----- | ------------------------ | ---------------------- |
| 主要目的  | アプリケーション構築（ウェブサイト、ゲームなど） | モデル記述・実験・検証（研究目的）      |
| 使用者想定 | 開発者・エンジニア                | 研究者／学生／技術者（非情報系含む）     |
| 関心対象  | 制御・状態・データ構造              | 関数・方程式・パラメータ（数学的概念）    |
| 評価軸   | 正しく動くか・効率的か              | 現象理解に寄与するか・実験しやすいか     |
| 記述様式  | 手続き的（どうやって計算するか）         | 宣言的（何を計算したいか）          |

#### 例：同じ計算の記述比較

一般的な言語（手続き的）。
```
温度配列を作成
各温度について繰り返し開始
  売上を計算
  結果を配列に格納
繰り返し終了
結果を表示
```

DSL（宣言的）。
```
温度を 20, 25, 30 として
売上 = 温度 × 1000 + 5000
結果を表示
```

つまり、科学計算 DSL の評価基準は、プログラムの正当性 だけでなく、研究対象の理解促進・仮説検証の容易性・表現力の適合 にあります。

## 2. 要求仕様の詳細化 ― 機能面・認知面の両立

### 2.1 機能的要求（Functional Requirements）

本ミニ言語が満たすべき基本機能を、優先順位順に列挙します。

##### 必須機能（Phase 1）

1. 変数宣言機能
   - 実数変数を、わかりやすい名前で宣言できること
   - 例：`temperature`（温度）、`sales`（売上）など
   - 内部的には浮動小数点数として統一管理

2. 値の代入と参照
   - 代入：`1.5 to x` のように自然な表現
   - 参照：`show x` で現在の値を確認

3. 簡単な計算式の評価
   - `calculate x * 2 + 1` のような計算を直接実行
   - 中間結果の確認が容易

##### 基本機能（Phase 2）

4. 関数定義機能
   - 数学的な関数を定義できること
   - 例：`f(x) = x² + 1` のような関数

5. 関数評価機能
   - 特定の点での関数値を計算
   - 例：「x = 2.0 のとき f(x) は？」

6. 実験手順の記述
   - 一連の操作を「実験」として名前付け
   - 後から再実行可能

##### 発展機能（Phase 3以降）

7. パラメータ走査（複数の値を自動的に試す）
8. 結果の出力形式指定（CSV、グラフなど）
9. 微分方程式、最適化などへの拡張


### 2.2 認知的要求（Cognitive Requirements）

科学計算 DSL は、使用者の 思考様式 と一致する必要があります。

| 認知的要求         | 説明                        | 具体例                      |
| ------------- | ------------------------- | ------------------------ |
| 自然な語順     | 日本語/英語の思考順序に近い            | 「1.5 を x に代入」→ `1.5 to x` |
| 意図の明示     | 「何をしたいか」が読み取れる            | `show temperature` で温度表示 |
| 段階的理解     | 少しずつ機能を追加できる              | 変数→計算→関数→実験、と段階的に学習     |
| エラーの親切さ   | 間違えたとき、何が悪いかわかる           | 「変数 xyz は定義されていません」    |
| 再現性       | 後日、自分のコードを理解できる           | 実験名を付けることで思い出しやすい      |
| 視覚的フィードバック | 計算途中の値を確認できる              | `show` で中間値を表示           |

#### 教育的重要性：なぜ「認知的要求」が必要なのか

プログラミングでつまずく主な理由

1. 概念と記法の乖離：「頭で考えること」と「コードで書くこと」が違いすぎる
2. エラーメッセージの難解さ：何が間違っているか理解できない
3. 全体像の不可視：「今、何をしているのか」がわからなくなる

DSL は、これらの問題に 言語設計 で対処します。


## 3. 科学計算ミニ言語が解決すべき「溝」の具体例

#### 事例：人口増加モデルのシミュレーション

数学的記述
```
人口 P(t) の時間変化
dP/dt = 0.1 × P

初期条件：P(0) = 100
時刻 t = 0 から 10 まで、刻み幅 0.1 で計算
```

通常のFORTH（技術的記述）
```forth
100e population F!    \ 初期人口
0.1e rate F!          \ 増加率
0.01e dt F!           \ 時間刻み

100 0 DO              \ 100ステップ繰り返し
  population F@       \ 現在の人口を取得
  rate F@             \ 増加率を取得
  F*                  \ 掛け算
  dt F@               \ 時間刻みを取得
  F*                  \ 掛け算
  population F@       \ 現在の人口を取得
  F+                  \ 足し算
  population F!       \ 新しい人口を保存
LOOP
```

DSL（意図的記述）
```forth
var population
var rate

population = 100.0
rate = 0.1

simulate from 0 to 10 step 0.1
  population = population + rate * population * step
end

show population
```

#### 違いの分析

| 項目     | 通常のFORTH            | DSL               |
| ------ | ------------------- | ----------------- |
| 可読性    | 記号的で理解困難            | 数式に近く直感的         |
| 変更容易性  | 変数追加時に多数の箇所を修正      | 変数定義と式のみ修正        |
| エラー検出  | 実行時エラー（原因不明）        | 宣言時・代入時に検出可能      |
| 学習曲線   | FORTHスタック操作の理解が前提   | 数学的背景があれば読める      |
| 保守性    | 3ヶ月後に自分でも読めない可能性あり  | 変数名と式で意図が残る       |

## 4. リファレンス設計文書 ― DSL 要件詳細仕様書

### 4.1 システム機能階層

DSLは段階的に構築します。各層は下層を基盤として機能します。

| 層  | 名称           | 説明                       | 本章での扱い |
| -- | ------------ | ------------------------ | ------ |
| L1 | 計算基盤     | FORTH の算術／型／スタック         | 前提知識   |
| L2 | 科学計算プリミティブ | 浮動小数点演算／基本関数             | 前提知識   |
| L3 | DSL基本語彙  | var/to/show/calculate    | ◎本章の中心 |
| L4 | 実験記述層    | experiment/scan/plot     | 一部実装   |
| L5 | 応用展開層    | ODE/最適化/統計/MCMC などとの連携   | 今後の課題  |

本章では L3（基本語彙） を完成させ、L4 への道筋を示します。


## 5. DSL プリミティブの仕様定義と実装

以下、各コマンドを 仕様 → 実装 → 使用例 → 注意点 の順で説明します。

#### 5.1 `var` ― 変数宣言

**仕様**
```
var <変数名>
```
- 浮動小数点変数を宣言
- 宣言後、変数名がアドレスを返すワードとして機能
- 未宣言の変数を使用するとエラー

**実装**
```forth
: var ( "name" -- )
   FVARIABLE ;
```

**使用例**
```forth
var temperature    \ 温度変数を宣言
var pressure       \ 圧力変数を宣言
var volume         \ 体積変数を宣言
```

**内部動作**

`FVARIABLE` は FORTH の標準ワードで、次の動作をします。

1. 辞書に新しいワード（ここでは `temperature`）を登録
2. 浮動小数点数1つ分のメモリ領域を確保
3. 実行時にそのアドレスをスタックに積む

したがって、`temperature` を実行すると、温度値が保存されているメモリアドレスが得られます。


### 5.2 `to` ― 値の代入（改良版）

**仕様**
```
<値> to <変数名>
```
- 浮動小数点値を指定変数に代入
- より自然な語順を実現

**実装**
```forth
: to ( f: value -- )
   BL WORD DUP >R
   FIND IF
      R> DROP EXECUTE F!
   ELSE
      DROP FDROP
      ." Error: Variable '" R> COUNT TYPE ." ' not found" CR
      ABORT
   THEN ;
```

**使用例**
```forth
var temperature
25.5e to temperature    \ 25.5度を代入

var pressure
1013.25e to pressure    \ 1013.25 hPa を代入
```

### 5.3 `show` ― 値の表示

**仕様**
```
show <変数名>
```
- 変数の現在値を人間が読める形式で表示

**実装**
```forth
: show ( -- )
   BL WORD DUP >R
   FIND IF
      R> DROP EXECUTE F@ F. CR
   ELSE
      DROP ." Error: Variable '" R> COUNT TYPE ." ' not found" CR
   THEN ;
```

**使用例**
```forth
var temperature
25.5e to temperature
show temperature
\ 出力: Value: 25.5
```

### 5.4 `define-function` ― 関数定義

**仕様**
```
define-function <関数名>
   <FORTH式>
end-function
```

**実装**
```forth
: define-function ( "name" -- )
   : ;  \ 通常のワード定義を開始

: end-function ( -- )
   POSTPONE ; ; IMMEDIATE
```

**使用例**
```forth
define-function square    \ f(x) = x²
   FDUP F*
end-function

define-function quadratic \ f(x) = x² + 1
   FDUP F* 1.0e F+
end-function

**使用方法**
```forth
2.0 square F.        \ 4.0
3.0 quadratic F.     \ 10.0
```

##### POSTPONE と IMMEDIATE

POSTPONEは、コンパイル時に動作する特殊なワード。次に来るワードを「後でコンパイル」する。ここでは、`;`を探し、それをコンパイル対象のワードに追加します。

その後の、`IMMEDIATE`によって即座に実行されます。

```
define-function square
   FDUP F*
end-function

─────────────────────────────────
ステップ1: define-function square
─────────────────────────────────

define-function の定義:
: define-function : ;

つまり、単に `:` を実行する

結果:
  → : square が実行される
  → squareという名前のワード定義が開始される

現在の状態:
  コンパイル中: square
  辞書の状態: [square ... (未完成)

─────────────────────────────────
ステップ2: FDUP F*
─────────────────────────────────

通常のワードなのでコンパイルされる

現在の状態:
  コンパイル中: square
  squareの本体: FDUP F*

─────────────────────────────────
ステップ3: end-function
─────────────────────────────────

end-functionはIMMEDIATEなので、
その場で実行される！

end-functionの本体:
  POSTPONE ;

POSTPONEの動作:
  → ; をsquareの定義に追加
  → squareの定義を終了

最終結果:
  辞書に登録されたもの:
  : square FDUP F* ;
  ↑              ↑
  ステップ1      ステップ3で追加

─────────────────────────────────
完了
─────────────────────────────────

辞書の最終状態:
┌──────────────────┐
│ square           │
├──────────────────┤
│ FDUP             │
│ F*               │
│ EXIT (;の実体)   │
└──────────────────┘
```


#### 5.5 `at` ― 関数の評価と表示

**仕様**
```
<値> at <関数名>
```

**実装**
```forth
: at ( f: x -- )
   BL WORD FIND 0= IF
      FDROP ." Error: Function not found" CR
   ELSE
      EXECUTE ." Result: " F. CR
   THEN ;
```

**使用例**
```forth
define-function cube
   FDUP FDUP F* F*    \ x * x * x
end-function

2.0e at cube          \ 出力: Result: 8.0
3.0e at cube          \ 出力: Result: 27.0
```

### 5.6 `experiment` ― 実験手順の定義

**仕様**
```
experiment <実験名>
   <一連の操作>
end-experiment
```

**実装**
```forth
: experiment ( "name" -- )
   : ." --- Experiment: " ;

: end-experiment ( -- )
   POSTPONE ; ; IMMEDIATE
```

**使用例**
```forth
experiment test-quadratic
   ." Testing f(x) = x^2 + 1" CR
   1.0e at quadratic
   2.0e at quadratic
   3.0e at quadratic
end-experiment

test-quadratic  \ 実験を実行
```

上記コードは、`Floating-point stack underflow`エラーが発生します。これは、``at`の中の`BL WORD`がコンパイル時に実行されてしまうためで、`at`の動作をコンパイル時に変更するよう修正が必要です。また、`to`と`show`にも同様の問題があるため、修正が必要です。

```
\ --- 値の代入 ---
: to ( f: value "varname" -- )
   ' >BODY              \ 変数名を読み、データ領域のアドレスを取得
   STATE @ IF           \ コンパイルモード？
      POSTPONE LITERAL  \ アドレスをリテラルとしてコンパイル
      POSTPONE F!       \ F!をコンパイル
   ELSE                 \ インタプリタモード
      F!                \ 直接格納
   THEN ; IMMEDIATE

\ --- 値の表示 ---
: show ( "varname" -- )
   ' >BODY              \ 変数名を読み、データ領域のアドレスを取得
   STATE @ IF           \ コンパイルモード？
      POSTPONE LITERAL  \ アドレスをリテラルとしてコンパイル
      POSTPONE F@       \ F@をコンパイル
      POSTPONE F.       \ F.をコンパイル
      POSTPONE CR       \ CRをコンパイル
   ELSE                 \ インタプリタモード
      F@ F. CR          \ 直接表示
   THEN ; IMMEDIATE

\ --- 関数評価の実行時処理 ---
: (at) ( f: x xt -- )
   EXECUTE              \ 関数を実行
   ." Result: " F. CR ; \ 結果を表示

\ --- 関数評価 ---
: at ( "funcname" -- )
   '                    \ 関数名を読み、実行トークンを取得
   STATE @ IF           \ コンパイルモード？
      POSTPONE LITERAL  \ xtをリテラルとしてコンパイル
      POSTPONE (at)     \ (at)をコンパイル
   ELSE                 \ インタプリタモード
      (at)              \ 直接実行
   THEN ; IMMEDIATE
```

これで、先の使用例は以下のよう名結果を出すことが出来ます。

**出力例**
```
--- Experiment: test-quadratic ---
Testing f(x) = x^2 + 1
Result: 2.
Result: 5.
Result: 10.
```

#### '（ティック）とは

- 次の単語を入力から読み取る
- その単語の実行トークン（xt）を返す
- 見つからない場合はエラー

```
\ 例1：xtを取得して実行
' DUP           \ DUPのxtがスタックに
EXECUTE         \ DUPが実行される

\ 例2：確認
5 ' DUP EXECUTE .s
\ 出力: 5 5  ← DUPされた

\ 例3：xtの値を見る
' DUP .         \ 例: 140234567890（アドレス値）

\ 例4：変数のxt
FVARIABLE temperature
' temperature . \ temperatureのxtが表示される
```

```:FINDとの比較
\ FIND を使う方法（手動）
BL WORD FIND    \ ( c-addr -- xt flag ) または ( c-addr -- c-addr 0 )
                \ flagが0なら見つからない

\ ' を使う方法（簡潔）
' word-name     \ ( -- xt )
                \ 見つからない場合は自動でエラー
```

#### `>BODY` （トゥーボディ）とは

`>BODY`は実行トークン（xt）からワードのデータ領域アドレスを取得する。

```forth
>BODY ( xt -- addr )
```

FVARIABLEで作った変数は、辞書に「ヘッダ部」「コード部」「データ部」を持つ。`'`で取得するxtはコード部を指し、`>BODY`でデータ部のアドレスに変換します。

```forth
FVARIABLE x
' x           \ xtを取得
>BODY         \ データ領域のアドレスに変換
F!            \ そのアドレスに値を格納
```

```
FVARIABLE temperature

【辞書に作られる構造】

┌─────────────────────────────────────┐
│ ヘッダ部                              │
├─────────────────────────────────────┤
│ リンクフィールド（前のワードへ）         │
│ 名前フィールド: "temperature"          │
│ フラグ                                │
├─────────────────────────────────────┤
│ コード部                              │
│ 実行トークン（xt）が指す場所 ←───────── ' temperature
│ 実行時の動作:「自分のBODYアドレスを返す」│
├─────────────────────────────────────┤
│ データ部（BODY）                       │
│ 8バイトの浮動小数点値 ←───────────────── ' temperature >BODY
│ (ここに実際の値が格納される)            │
└─────────────────────────────────────┘
```

変数の場合、`' x EXECUTE`と`' x >BODY`は同じアドレスを返しますが、`>BODY`はデータ部のアドレスであることを明確に示している。

#### `[ ]` について

`[ ]`の中に書かれた処理は、コンパイル時に計算され、リテラルな値が作成される。

```:定数の事前計算
\ [ ]なし：実行時に毎回計算
: circle-area-slow ( f: r -- area )
   FDUP F*
   3.14159265358979e 2.0e F*    \ πは毎回リテラルとして読み込み
   F* 2.0e F/
;

\ [ ]あり：コンパイル時に計算
: circle-area-fast ( f: r -- area )
   FDUP F*
   [ 3.14159265358979e 2.0e F* ] FLITERAL  \ 2πを事前計算
   F* 2.0e F/              \ 結果的にπ×r²
;
```

```:条件付きコンパイル
TRUE CONSTANT DEBUG

\ DEBUGフラグに応じてコードを変える
: process ( n -- )
   [ DEBUG ] LITERAL IF
      ." Processing: " DUP . CR
   THEN
   2 *
;
```

#### `[']` について

`[']` はコンパイルモードで次のワードの実行トークン（xt）をリテラルとして埋め込む。

```
\ インタプリタモードでは ' を使う
' DUP EXECUTE

\ コンパイルモード（定義内）では ['] を使う
: test
   ['] DUP EXECUTE    \ コンパイル時にxtを埋め込む
;

\ [ ' ... ] LITERAL と同等
: test2
   [ ' DUP ] LITERAL EXECUTE
;
```

'はコンパイル時に即座実行され、xtがスタックに積まれるがコンパイルされないため、実行時にEXECUTEがxtを探してスタックアンダーフローが起きる。`[']`はコンパイル時に決まったワードを埋め込み固定される。

```
: test ' dup execute ; 
1 test .  \ → エラー
*the terminal*:57:7: error: Attempt to use zero-length string as a name

: test ['] dup execute ; 
1 test .  \ → ( 1  1 )
```

`to` `at` `show` の場合はユーザーが書いた変数名を読む必要があるため、`'`で動的に取得している。


## 6. 完全動作例：DSL による科学計算実験

以下は、すべての機能を組み合わせた実践例です。

```
\ ========================================
\ 科学計算DSL - 完成版
\ ========================================

\ --- 変数宣言 ---
: var ( "name" -- )
   FVARIABLE ;                \ 浮動小数点変数を作成

\ --- 値の代入 ---
: to ( f: value "varname" -- )
   ' >BODY                    \ 変数名を読み、データアドレスを取得
   STATE @ IF                 \ コンパイルモード？
      POSTPONE LITERAL        \   アドレスをリテラル化
      POSTPONE F!             \   F!をコンパイル
   ELSE                       \ インタプリタモード
      F!                      \   直接格納
   THEN ; IMMEDIATE

\ --- 値の表示 ---
: show ( "varname" -- )
   ' >BODY                    \ 変数名を読み、データアドレスを取得
   STATE @ IF                 \ コンパイルモード？
      POSTPONE LITERAL        \   アドレスをリテラル化
      POSTPONE F@             \   F@をコンパイル
      POSTPONE F.             \   F.をコンパイル
      POSTPONE CR             \   CRをコンパイル
   ELSE                       \ インタプリタモード
      F@ F. CR                \   直接表示
   THEN ; IMMEDIATE

\ --- 関数定義 ---
: define-function ( "name" -- )
   : ;                        \ ワード定義を開始

: end-function ( -- )
   POSTPONE ;                 \ 定義終了をコンパイル
; IMMEDIATE

\ --- 関数評価ヘルパー ---
: (at) ( f: x xt -- )
   EXECUTE                    \ 関数を実行
   ." Result: " F. CR ;       \ 結果を表示

\ --- 関数評価 ---
: at ( f: x "funcname" -- )
   '                          \ 関数名を読み、xtを取得
   STATE @ IF                 \ コンパイルモード？
      POSTPONE LITERAL        \   xtをリテラル化
      POSTPONE (at)           \   (at)をコンパイル
   ELSE                       \ インタプリタモード
      (at)                    \   直接実行
   THEN ; IMMEDIATE

\ --- 実験ブロック開始 ---
: experiment ( "name" -- )
   :                          \ 次の単語を名前としてワード定義開始
   ." ========================================" CR
   ." Experiment started" CR
   ." ========================================" CR ;

\ --- 実験ブロック終了 ---
: end-experiment ( -- )
   ." ========================================" CR
   ." [Completed]" CR
   ." ========================================" CR
   POSTPONE ;                 \ 定義終了をコンパイル
; IMMEDIATE
```

```forth
\ ========================================
\ 実験：放物運動の軌跡計算
\ ========================================

\ --- 変数宣言 ---
var initial_velocity   \ 初速度 [m/s]
var angle             \ 発射角度 [度]
var time              \ 経過時間 [s]
var height            \ 高さ [m]
var distance          \ 水平距離 [m]

\ --- 定数 ---
9.8e FCONSTANT gravity  \ 重力加速度 [m/s²]

\ --- 関数定義 ---
define-function calculate-height
   \ 入力: 時刻 t (秒)
   \ 出力: 高さ h (m)
   \ h = v₀sinθ·t - (1/2)gt²
   
   \ この実装は簡略版
   \ 実際には angle から sin を計算する必要があります
   FDUP F*              \ t²
   gravity F*           \ g·t²
   2.0e F/               \ (1/2)g·t²
   FNEGATE              \ -(1/2)g·t²
   
   initial_velocity F@
   45.0e FSIN F*         \ v₀sinθ (45度固定の例)
   time F@ F*           \ v₀sinθ·t
   
   F+                   \ 合計
end-function

\ --- 実験定義 ---
experiment projectile-motion
   ." === Projectile Motion Simulation ===" CR
   
   20.0e to initial_velocity
   ." Initial velocity: 20 m/s" CR
   
   ." Time [s]  Height [m]" CR
   
   0.0e to time
   time F@ at calculate-height
   
   0.5e to time
   time F@ at calculate-height
   
   1.0e to time
   time F@ at calculate-height
   
   1.5e to time
   time F@ at calculate-height
   
   2.0e to time
   time F@ at calculate-height
end-experiment

\ --- 実行 ---
projectile-motion
```

```:出力例
=== Projectile Motion Simulation ===
Initial velocity: 20 m/s
Time [s]  Height [m]
Result: 0.
Result: 7.28403524534118
Result: 12.1180704906824
Result: 14.5021057360236
Result: 14.4361409813647
```

#### このコードの教育的価値

1. 物理法則の可視化：数式が直接コードになっている
2. 段階的な理解：変数→定数→関数→実験、と積み上げ式
3. 再利用性：`projectile-motion` を何度でも実行可能
4. 拡張性：角度を変数化、グラフ出力など発展可能


## 7. 今後の拡張に向けた言語設計指針


| Phase | 機能               | 難易度 | 教育的価値 | 実装例                         |
| ----- | ---------------- | --- | ----- | --------------------------- |
| 1     | 配列サポート       | 中   | 高     | `array temperatures 10`     |
| 2     | ループ構文        | 低   | 高     | `for t from 0 to 10 step 1` |
| 3     | 中置記法パーサ      | 高   | 中     | `calculate x^2 + 2*x + 1`   |
| 4     | CSV出力        | 低   | 高     | `export-csv results.csv`    |
| 5     | グラフ描画連携      | 中   | 極高    | `plot x y using gnuplot`    |
| 6     | 微分方程式ソルバー    | 高   | 高     | `solve-ode dy/dt = ...`     |
| 7     | 最適化ルーチン      | 高   | 中     | `minimize f(x) from a to b` |
| 8     | 統計関数         | 中   | 高     | `mean`, `stddev`, `regress` |

#### 最終目標：統合科学計算環境

将来的には、次のような記述が可能になることを目指します。

```forth
model population-growth
   parameter r = 0.1      \ 増加率
   parameter K = 1000     \ 環境収容力
   
   variable P             \ 個体数
   
   equation dP/dt = r * P * (1 - P/K)
   
   initial P = 10
   
   simulate from 0 to 100 step 0.1
end-model

run population-growth
plot-results P vs time
export-csv population.csv
```

このレベルになると、MATLAB / R / Python (NumPy) に匹敵する科学計算環境となります。

