# Lev 19 ー ファイル入出力とデータ記録
FORTHは対話的な実行環境として知られていますが、実用プログラムを作るには「データを保存・読み出す」ことが欠かせません。gforthでは標準規格に準拠した**ファイル入出力ワード群**が用意されており、テキストファイルやログの操作を簡単に行えます。

## ファイル入出力の基本構造

ファイルを扱う際は、次のような流れになります。

```
OPEN-FILE    （ファイルを開く）
READ-LINE    （読み出し）
WRITE-LINE   （書き込み）
CLOSE-FILE   （閉じる）
```

これらの操作では、**ファイルID（ファイルハンドル）**と呼ばれる識別子を使って、どのファイルに対して操作を行うかを指定します。


## ファイルを開く ― `OPEN-FILE`

次の例は、`test.txt`というファイルを新規作成（または上書き）して書き込む準備をするものです。

```forth
S" test.txt"  W/O CREATE-FILE  THROW  CONSTANT log-fid
```

* `S" test.txt"` は文字列をスタックに積む。
* `W/O` は**書き込み専用（Write Only）**を意味するファイルモード。
* `CREATE-FILE` は指定されたモードで新規作成。成功すればファイルIDを返す。
* `THROW` はエラー処理を行う。
* `CONSTANT log-fid` により、ファイルIDを定数として登録。

これで、以降 `log-fid` を使って書き込みが可能になります。

## ファイルへ書き込む ― `WRITE-LINE`

ファイルIDを指定して1行書き込むには次のようにします。

```forth
S" Hello, Forth world!" log-fid WRITE-LINE THROW
```

1. `S" ... "` が書き込む文字列を積む。
2. `WRITE-LINE` が改行付きでファイルに出力する。
3. `THROW` は例外処理（エラーがなければ何もしない）。

```forth:複数行を出力する例
: WRITE-SAMPLE ( -- )
	S" data.txt" W/O CREATE-FILE THROW >R   \ ファイルIDをリターンスタックへ
	S" line 1" R@ WRITE-LINE THROW
	S" line 2" R@ WRITE-LINE THROW
	R> CLOSE-FILE THROW ;

S" DATA.TXT" WRITE-SAMPLE
```

実行すると、カレントディレクトリに `data.txt` が作成されます。

## ファイルを読む ― `READ-LINE`

読み込みの基本は次の通りです。

```forth:ファイルの読み込み
: SHOW-FILE ( c-addr u -- )
  R/O OPEN-FILE THROW                \ ファイルIDはスタックに入れる
  PAD 80 ERASE
  BEGIN
     PAD 80 2 PICK READ-LINE THROW   \ 2 PICでファイルIDをコピー
  WHILE                              \ flagで継続判定、uが残る
     CR PAD SWAP TYPE                \ buffer u -> 表示
  REPEAT
  DROP CLOSE-FILE THROW              \ u を削除して ファイルIDを使う
;

S" DATA.TXT" SHOW-FILE
```

* `R/O` は読み取り専用モード。
* `READ-LINE` は「バッファ／サイズ／ファイルID」を引数にとり、読み込んだ文字数とEOFフラグを返します。
* `WHILE … REPEAT` により、ファイル末尾（EOF）まで繰り返します。
* `TYPE` は文字列を表示。

## ファイルモードの種類

| モード           | 意味          | ワード           |
| :------------ | :---------- | :------------ |
| `R/O`         | 読み取り専用      | `OPEN-FILE`   |
| `W/O`         | 書き込み専用（上書き） | `CREATE-FILE` |
| `R/W`         | 読み書き両用      | `OPEN-FILE`   |

追記する場合は、`FILE-SIZE`を取得して`REPOSITION-FILE`で位置を移動する。

```forth:追記の例
: APPEND ( c-addr u c-addr-file u-file -- ior )
	R/W OPEN-FILE THROW >R   \ ファイルIDをリターンスタックへ
	R@ FILE-SIZE THROW
	R@ REPOSITION-FILE THROW \ ファイルの最後の位置に移動
	R@ WRITE-LINE THROW      \ 書き込み
	R> CLOSE-FILE THROW ;

S" Hello!" S" DATA.TXT" APPEND
```

## INCLUDEによるスクリプト分割

大きなプログラムは、複数のソースファイルに分割して管理できます。gforthでは `INCLUDE` ワードを使い、外部ファイルを読み込んで実行します。

```forth
INCLUDE mylib.fs
```

これにより、ファイル内で定義されたワードが辞書に登録され、再利用が容易になります。特に科学計算やシミュレーションでは、**ロジック部分とデータ部分を分離**するのに有効です。

## 応用例：ログ出力プログラム

センサー値などを定期的に記録する簡単なログプログラムを作ってみましょう。

```forth
: log-data ( f: value -- )
	S" sensor.log" R/W OPEN-FILE THROW >R
	R@ FILE-SIZE THROW
	R@ REPOSITION-FILE THROW \ ファイルの最後の位置に移動
    F>S 0 <# #S #> R@ WRITE-LINE THROW
	R> CLOSE-FILE THROW ;
```

* 浮動小数点値を整数化し、文字列に変換して保存。
* `REPOSITION-FILE` により既存ファイルの末尾に移動。
* センサー値の蓄積やデバッグログの記録に使える。


### 発展課題

1. 配列内の10個の数値をファイルに書き出すワード `SAVE-DATA` を定義せよ。
2. 書き出したデータを `READ-DATA` で読み込み、合計値を表示せよ。
3. `INCLUDE` を使って、これらを別ファイル `io-tools.fs` に分割・管理せよ。

