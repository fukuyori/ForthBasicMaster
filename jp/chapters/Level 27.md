# Lev 27 ー ソートと探索アルゴリズム
ソート（sort）とは、データを一定の順序に並べ替える操作のことである。たとえば、学生の得点を高い順に並べたり、文字列を辞書順に整列させたりする。この「順序づけ」は、あらゆる情報処理の基盤であり、後に学ぶ**探索（search）や統計処理**の前提となる。

FORTHにおけるソート学習の目的は単なる並べ替えではない。FORTHでは、すべての操作を**スタック（stack）という一時的な記憶領域**を通して表現する。したがって、ソートを通じて「データの流れ」と「制御構造」の両方を理解することができる。この章では、FORTHの哲学「データが通る道筋がプログラムである」を体現する題材としてソートを扱う。

## ソートの考え方と分類

ソートアルゴリズムは大別すると次のような群に分けられる。

| 種類     | 代表例                  | 特徴           |
| ------ | -------------------- | ------------ |
| 比較型    | バブルソート、挿入ソート、クイックソート | 要素同士を比較して並べる |
| 非比較型   | カウントソート、基数ソート        | 値そのものの性質を利用  |
| 安定ソート  | 挿入ソート、マージソート         | 同値要素の順序を保持   |
| 不安定ソート | クイックソート、ヒープソート       | 元の順序を保持しない   |

FORTHは低レベル制御に強いため、ここでは**比較型の基本二種（バブルソートと挿入ソート）**を扱う。これらは単純でありながら、「比較」「入れ替え」「繰り返し」の3構造をすべて含むため、FORTHの制御文とスタック操作を理解する格好の題材である。

## FORTHで配列を扱う ― メモリアクセスの準備

CやPythonでは `array[i]` のように書くが、FORTHでは「アドレス＋オフセット」で計算する。
以下の3つのワードを定義しておくことで、FORTHでも直感的に配列を扱えるようになる。

```forth
\ 配列要素アクセスユーティリティ
: CELL-INDEX ( base i -- addr )  CELLS + ;
: A@ ( base i -- n )             CELL-INDEX @ ;
: A! ( n base i -- )             CELL-INDEX ! ;
```

* `A@` は配列の要素取得（Array get）
* `A!` は要素代入（Array put）
* `CELL-INDEX` は基底アドレスから i 番目のセルを求める

これらを整えておくと、FORTHのコードが格段に可読になる。

## バブルソート ― 隣接交換のアルゴリズム

### 1. 原理

バブルソート（bubble sort）は、隣り合う要素を比較し、大小を入れ替えることで整列させる方法である。「大きい値が泡のように上に浮かぶ」ことからこの名がついた。

処理の流れは以下の通りである。

1. 配列の先頭から隣り合う要素を比較
2. 順序が逆であれば交換する
3. 最後まで到達したら、最大の要素が末尾にくる
4. 残りの部分に対して同じ操作を繰り返す

複雑なスタック操作を行わず、ローカル変数を使用し、すっきり記述した。

```forth:バブルソート
\ 要素の交換
: SWAP-ELEMENTS ( base i j -- )
	{ base i j }
	base i A@                       \ 値1 = base[i]
	base j A@                       \ 値2 = base[j]  スタック: (値1 値2)
	base i A!                       \ base[i] = 値2
	base j A!                       \ base[j] = 値1
;

\ 2要素の比較と交換（昇順）
: COMPARE-AND-SWAP ( base i -- )
	{ base i }
	base i A@                       \ 値1 = base[i]
	base i 1+ A@                    \ 値2 = base[i+1]  スタック: (値1 値2)
	> IF                            \ 値1 > 値2 なら交換
		base i i 1+ SWAP-ELEMENTS
	THEN
;

\ バブルソート本体
: BUBBLE-SORT ( base n -- )
	{ base n }
	n 2 < IF EXIT THEN              \ 要素数が2未満なら終了
	n 1- 0 DO                       \ 外側ループ: n-1 パス
		n I - 1- 0 DO                 \ 内側ループ: 未確定部分を走査
			base I COMPARE-AND-SWAP     \ 隣接要素を比較・交換
		LOOP                          \ 各パスで最大値が末尾に確定
	LOOP
;
```

1. **SWAP-ELEMENTS** ( base i j -- )
   - 配列の2要素を交換する低レベル操作
   - `PICK`と`-ROT`でスタックのみで実現

2. **COMPARE-AND-SWAP** ( base i -- )
   - i番目とi+1番目を比較し、逆順なら交換。
   - 比較ロジックをカプセル化

3. **BUBBLE-SORT-V2** ( base n -- )
   - 二重ループでアルゴリズム制御
   - 各パスで最大値が末尾へ移動
   - 外側：n-1回のパス / 内側：各パスで比較回数を減少

FORTHでは「比較」も「交換」も単なるデータフロー操作である。単一責任の原則に基づき、各ワードが1つの明確な役割を持つ。スタック操作の複雑さは各レイヤーに分散されている。


### 4.4 実行例

```forth:入力
CREATE data 5 CELLS ALLOT
10 data 0 A!
3  data 1 A!
7  data 2 A!
1  data 3 A!
9  data 4 A!

data 5 BUBBLE-SORT

: .ARRAY ( addr n -- )　\ 配列表示
  0 DO DUP I CELLS + @ . LOOP DROP CR ;

data 5 .ARRAY
```

```:出力
1 3 7 9 10
```


## 挿入ソート ― 部分的整列を拡張する方法

挿入ソート（insertion sort）は、人がトランプを並べるときのやり方に近い。左側を「すでに整列された部分」とみなし、右側から1枚ずつ取り出して適切な位置に挿入していく。

* 小規模データでは高速に動作する。
* データがほぼ整列している場合は非常に効率的。
* FORTHでは、「比較」「シフト」「挿入」の構造が明確に見える。

まず、Python版のプログラムを見る。

```python:挿入ソート（Python版）
def insertion_sort(arr):
    for i in range(1, len(arr)):
        key = arr[i]          # 挿入したい値を一時的に保持
        j = i - 1
        # 左側の整列済み部分を右にずらしながら挿入位置を探す
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key      # 見つけた位置にkeyを挿入
    return arr

 # 実行例
data = [9, 3, 5, 1, 4]
print("整列前:", data)
sorted_data = insertion_sort(data)
print("整列後:", sorted_data)
```

```
整列前: [9, 3, 5, 1, 4]
整列後: [1, 3, 4, 5, 9]
```

```forth:挿入ソート（Forth版）
: INSERT-SORT ( base n -- )
	{ base n }
	n 1 DO                              \ i = 1 to n-1
		base I A@                       \ key = base[i]  スタック: (key)
		I                               \ j = i          スタック: (key j)

        BEGIN
            DUP 0>  IF                   \ j > 0? ならば
                base OVER 1- A@ 2 PICK > \ base[j-1] > key? を評価
            ELSE
                FALSE                    \ そうでなければFALSE
            THEN
        WHILE
			base OVER 1- A@ base 2 PICK A!  \ base[j] = base[j-1]
			1-                              \ j--           スタック: (key j-1)
		REPEAT                              \ スタック: (key j)
		base SWAP A!                        \ base[j] = key
	LOOP
;
```

```:テスト
\ テストデータ
CREATE data 5 CELLS ALLOT
10 data 0 A!
3 data 1 A!
7 data 2 A!
1 data 3 A!
9 data 4 A!

clear
CR ." ソート前: " data 5 .ARRAY

data 5 insert-sort  
." ソート後: " data 5 .array
```

```:実行例
ソート前: 10 3 7 1 9
ソート後: 1 3 7 9 10
```

この実装では、**LOOP** と **BEGIN WHILE REPEAT** を組み合わせている。`BEGIN` 以下の条件部分で2つの条件を`AND`せずに`IF-ELSE-THEN`を使い、j > 0が偽の場合は配列アクセスをスキップする。これにより、j = 0の時にbase[-1]へアクセスすることを防いでいる。

Pythonでは、and 演算子は「短絡評価（short-circuit evaluation）」を行うため、 `while j >= 0 and arr[j] > key:` の式では `j >= 0` が `False` のとき、右側の `arr[j] > key` は評価されされない。

## ソートアルゴリズムの比較と応用

| アルゴリズム  | 計算量        | 特徴            |
| ------- | ---------- | ------------- |
| バブルソート  | O(n²)      | 構造が単純。教材向き。   |
| 挿入ソート   | O(n²)      | 小規模・部分整列に強い。  |
| クイックソート | O(n log n) | 実用的で高速。       |
| マージソート  | O(n log n) | 安定ソート。実装やや複雑。 |

FORTHのような低レベル言語では、アルゴリズムを理解した上で
「比較」「交換」「制御」をすべて自分で構築する。
これは、**データ構造と制御構造を一体として捉える力**を養う訓練となる。

