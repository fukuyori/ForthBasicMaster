# Lev 30 ー gforthイメージファイルとファイル管理
## 1 イメージファイルの基礎概念

### 1.1 イメージファイルとは

Gforthを開始するには通常は`gforth`とするだけで、デフォルトのイメージファイル`gforth.fi`が自動的にロードされます。

イメージファイルは、Forth辞書のスナップショットです。プログラムの実行可能な状態を丸ごと保存したもので、通常`.fi`という拡張子を持ちます。

なぜイメージファイルが必要なのか

Forthには特殊な事情があります。それはForthコンパイラ自体がForthで書かれているという自己参照的な性質です。これは以下のような問題を引き起こします。

1. Forthコードをコンパイルするにはコンパイラが必要
2. しかし、コンパイラもForthコード
3. エンジン（C言語の実行ファイル）だけでは起動できない

この「鶏と卵」の問題を解決するのがイメージファイルです。イメージファイルには、すでにコンパイルされたコンパイラと基本的なシステムが含まれています。

比喩で理解する

- 仮想マシンのスナップショット：コンピュータの状態を丸ごと保存して、後で復元する
- 冷凍食品：調理済みの料理を凍結保存し、温めるだけで食べられる状態にする
- インストール済みOS：OSをセットアップ済みの状態で保存し、すぐに使えるようにする

### 1.2 Gforthの二層構造

Gforthシステムは2つの部分から構成されます。

第1層：エンジン（Engine）
- C言語で実装された実行環境
- プリミティブ命令を直接実行
- ファイル名：`gforth`、`gforth-fast`、`gforth-itc`など
- 役割：低レベルの処理と高速実行

第2層：イメージファイル（Image File）
- Forthで書かれた高レベル定義群
- コンパイラ、テキストインタプリタ、標準ライブラリ
- デフォルト：`gforth.fi`
- 役割：Forthシステムの本体

この二層構造により、以下の利点があります。

- エンジンの最適化：C言語で高速化
- 柔軟なカスタマイズ：イメージを差し替えるだけで異なる環境を構築
- 開発とデプロイの分離：開発用と本番用で異なるイメージを使用

### 1.3 起動プロセスの詳細

Gforthが起動する際、以下のステップを経ます。


##### 【フェーズ1：初期化】
1. エンジン（Cプログラム）が起動
2. メモリ領域の確保
3. イメージファイルの検索と読み込み

##### 【フェーズ2：再配置】
4. メモリアドレスの調整（relocation）
5. コードトークンの解決
6. スタック領域の設定

##### 【フェーズ3：初期化ファイル実行】
7. gforthrc0の実行（最初の初期化）
8. コマンドライン引数の処理
9. gforthrcの実行（ユーザー設定）

##### 【フェーズ4：実行開始】
10. ブートメッセージの表示
11. REPLまたはスクリプトの実行

この起動プロセスは、OSがプログラムを起動する際の動作に似ていますが、Forth特有の要素（辞書の再配置、初期化ファイル）が追加されています。

## 2. イメージファイルの種類

イメージファイルには3つの種類があり、それぞれ移植性（portability）と生成の容易さ（ease of creation）のトレードオフが異なります。

### 2.1 非再配置イメージ（Non-Relocatable）

#### 技術的特性

非再配置イメージは、辞書のメモリダンプそのものです。以下の特徴があります。

- アドレス固定：作成時のメモリアドレスに完全に依存
- 実行ファイル依存：作成時の`gforth`実行ファイルでのみ動作
- 最も単純：特別な処理なしに作成可能

しかし、すべての再配置コストを回避しますが、アドレス空間のランダム化があるOSでは機能しません。

現代のOSにはASLR（Address Space Layout Randomization）というセキュリティ機能があります。これは。

1. プログラムを起動するたびにメモリ配置をランダム化
2. バッファオーバーフロー攻撃を困難にする
3. 毎回異なるアドレスに配置される

非再配置イメージは特定のアドレスを期待しているため、ASLRによって期待と異なるアドレスに配置されると動作不能になります。


```forth:作成方法
gforth myapp.fs -e "savesystem myapp.fi bye"
```

また、gforthの対話モードで、

```
SAVESYSTEM myapp.fi
```

```:読み込み
gforth --image-file myapp.fi
gforth -i myapp.fi
```

```:エラー
gforth: Cannot load nonrelocatable image (compiled for address 0x715f8edfe040) at address 0x7dc5bcdfe040
```

:::note alert 
非再配置イメージは現代の環境では実用的ではありません。特別な場合を除いて使用を避けるべきです。
:::

### 2.2 データ再配置イメージ（Data-Relocatable）

技術的特性

データ再配置イメージは、中間的な解決策です。

- データアドレス：再配置可能（実行時に調整）
- コードアドレス：固定（作成時のまま）
- バイナリ依存：特定の実行ファイルと紐付き

再配置のメカニズム

起動時に、すべてのデータアドレスに対して以下の処理が行われます。

```
実際のアドレス = 保存されたアドレス + オフセット
```

制限事項

1. コードアドレスは固定：別のバイナリでは動作しない
2. 動的コード生成不可：ネイティブコード生成に非対応
3. バージョン依存：同じビルドの同じバイナリが必要

使用場面

起動速度が重要で、特定環境でのみ動作すればよい場合。

- 組み込みシステム
- ベンチマークプログラム
- 専用アプリケーション

作成方法

```bash
GFORTHD="/usr/local/bin/gforth-fast --no-dynamic" gforthmi myimage.fi source.fs
```

WSLとUbuntu上では、`Segmentation fault (core dumped)`となって、イメージの読み込みはできなかった。

### 2.3 完全再配置イメージ（Fully Relocatable）

技術的特性

完全再配置イメージは、最も柔軟な形式です。

- データアドレス：完全に再配置可能
- コードアドレス：トークン化され、起動時に解決
- バイナリ独立：異なる実行ファイルで動作
- アーキテクチャ依存：同じデータ形式（バイトオーダー、セルサイズ）が必要

トークン化の仕組み

コードアドレスは「トークン」という抽象的な識別子として保存されます。

```
【保存時】
ワード定義 → トークンID（例：0x0042）として保存

【起動時】
トークンID 0x0042 → 実際のコードアドレス（例：0x7f8a9c001234）に変換
```

これにより、バイナリが異なっても（デバッグ版/最適化版など）、トークンから正しいコードアドレスを解決できます。

マルチバイナリ対応

同じイメージファイルを以下の複数のエンジンで使用可能。

- `gforth`：標準版（詳細なエラー情報）
- `gforth-fast`：高速版（エラー情報は簡易）
- `gforth-itc`：間接スレッド版（後方互換性）

クロスプラットフォーム

以下が同じであれば、異なるマシンで動作。

- バイトオーダー：リトルエンディアン/ビッグエンディアン
- セルサイズ：32ビット/64ビット
- 浮動小数点形式：IEEE 754など

例：Linux x86_64で作成したイメージをLinux ARM64で使用（セルサイズとバイトオーダーが同じ場合）

作成方法

```bash
gforthmi myapp.fi myapp.fs
```

この操作で作成されたイメージファイルは、以下の様に読み込みます。

```
gforth -i myapps.fi
```

または、`myapps.fi`単独で起動することもできます。

```
./myapps.fi
```

また、`--application`オプションをつけることで、再配布可能な実行ファイルを作成することが出来ます。

```
gforth --application myapps.fi myapps.fs
```

```:実行例
./myapps.fi
```

## 3 イメージファイルの作成

:::note alert
イメージフィルの作成は期待通りの動作をしないことが多いので、この作業に時間を割くことはあまりお勧めしません。
:::

### 3.1 gforthmiツール

gforthmiは、完全再配置イメージを作成するための専用ツールです。名前は「Gforth Make Image」の略です。

基本文法

```bash
gforthmi [オプション] イメージファイル名 [ソースファイル...]
```

動作原理

gforthmiは巧妙な方法でイメージを作成します。

```
【ステップ1】異なるアドレスで2つの非再配置イメージを生成
  イメージA：辞書開始 = アドレスX
  イメージB：辞書開始 = アドレスY

【ステップ2】2つのイメージを比較
  各セルごとに値を比較
  
【ステップ3】差分の分析
  同じ値 → アドレス非依存データ（そのまま保存）
  差がオフセット → データアドレス（オフセット記録）
  その他の差 → コードアドレス（トークン化）

【ステップ4】完全再配置イメージを生成
  再配置情報を含む最終イメージを出力
```

簡単な例

```bash
## 単一ファイル
gforthmi calculator.fi calculator.fs

## 複数ファイル
gforthmi myapp.fi utils.fs main.fs ui.fs
```

出力の解読

```
$ gforthmi myapp.fi myapp.fs
...（Gforth起動メッセージ）...
Data offset: 0x12340000
Code offset: 0x56780000
78DC         BFFFFA50         BFFFFA40
Done.
```

最後の行の意味。
- `78DC`：辞書内のオフセット位置
- `BFFFFA50`/`BFFFFA40`：イメージA/Bでの値の違い
- この差異は再配置できない（デッドセルであるべき）

#### gf3.2 アプリケーションイメージ

通常のイメージとの違い

```bash
## 通常のイメージ
gforthmi myapp.fi myapp.fs
./myapp.fi arg1 arg2
## → arg1, arg2はエンジンオプションとして解釈

## アプリケーションイメージ
gforthmi --application myapp.fi myapp.fs  
./myapp.fi arg1 arg2
## → arg1, arg2はForthプログラムに渡される
```

実装上の違い

イメージファイルの先頭行（shebang）が異なります。

```bash
## 通常のイメージ
##! /usr/local/bin/gforth -i

## アプリケーションイメージ  
##! /usr/local/bin/gforth --appl-image
```

`--appl-image`オプションは、すべてのコマンドライン引数をイメージに渡し、エンジンオプションとして解釈しません。

ターンキーアプリケーション

アプリケーションイメージを使うと、Forthを知らないユーザーにも配布可能なスタンドアロンプログラムを作成できます。

### 3.3 メモリサイズの設定

イメージファイルにはデフォルトのメモリサイズが埋め込まれます。

デフォルト値

```
辞書：256KB
データスタック：16KB
リターンスタック：15KB
ローカルスタック：14.5KB
```

カスタムサイズの指定

```bash
## イメージ作成時に指定
gforthmi -m 4M -d 64k -r 32k myapp.fi myapp.fs

## 起動時に上書き
gforth -i myapp.fi -m 10M
```

サイズ単位の詳細

- `b`：バイト（bytes）
- `e`：要素＝セル（elements、32ビットシステムで1セル=4バイト）
- `k`：キロバイト（1024バイト）
- `M`：メガバイト（1024KB）
- `G`：ギガバイト（1024MB）
- `T`：テラバイト（1024GB）

メモリ戦略

```
小規模スクリプト。
  辞書 256KB、スタック 16KB （デフォルト）

中規模アプリケーション。
  辞書 1-4MB、スタック 64KB

大規模システム。
  辞書 10MB以上、スタック 128KB以上

データ処理。
  辞書は小さく、スタックを大きく（例：辞書1MB、スタック1MB）
```

## 4 イメージファイルの制限事項

イメージファイルには構造上の制限があります。これらを理解することで、予期しない動作を避けられます。

### 4.1 単一セグメント制限

技術的背景

イメージファイルは連続した単一のメモリ領域（セグメント）として保存されます。

##### 保存不可能なもの

1. 動的に確保されたメモリ（`ALLOCATE`で確保）
2. 動的メモリへのポインタ
3. スタックの内容

##### 理由

```
イメージファイル = 辞書のスナップショット

辞書外のメモリ。
  - OS管理のヒープ領域
  - スタック領域（別のメモリ空間）
  → イメージファイルの範囲外
```

#### 回避策

```forth
\  保存できない
variable my-buffer
100 cells allocate throw my-buffer !
savesystem bad.fi

\  起動時に確保
: init-buffers
  100 cells allocate throw my-buffer !
;
' init-buffers is 'cold
```

### 4.2 サポートされる再配置の種類

イメージローダーは2種類の再配置のみをサポートします。

タイプ1：単純なオフセット加算

すべてのデータアドレスに同じオフセットを加算。

```
新アドレス = 旧アドレス + オフセット
```

これは以下のような単純なポインタに適用されます。

```forth
variable counter
variable next-item
create buffer 100 allot
```

タイプ2：トークンの置換

特殊なトークン値を実際のコードアドレスまたは機械語コードで置換。

```forth
: my-word ... ;
' my-word  \ 実行トークン → コードアドレスに変換
```

### 4.3 複雑な計算の禁止

アドレスに対して複雑な演算を行った結果は保存できません。

#### 問題例1：ハッシュテーブル

```forth
\ アドレスベースのハッシュ関数
: hash-addr ( addr -- hash )
  #65536 mod  \ アドレスをハッシュ化
;

\ 問題：起動時にアドレスが変わる
\ → ハッシュ値も変わる
\ → テーブルが壊れる
```

解決策

```forth
\ Gforthの組み込み機能を使用
\ （起動時に自動再構築される）
wordlist constant my-dict

\ または、起動時に再構築
: rebuild-hash
  \ ハッシュテーブルを再計算
;
' rebuild-hash is 'cold
```

#### 問題例2：XORリンクリスト

XORによる双方向リンクリスト（メモリ効率的だが複雑）。

```forth
\ next = prev XOR next
\ アドレスをXORで保存

\ 問題：XOR演算の結果は再配置できない
```

解決策

```forth
\ イメージでは単方向リンクとして保存
\ 起動時に双方向リンクを復元
```

#### 問題例3：機械語コード内の絶対アドレス

```forth
\ CODE定義で絶対アドレスを埋め込む
code my-primitive
  ... 
  $12345678 call  \ 絶対アドレス
  ...
end-code

\ 問題：このアドレスは再配置不可
```

解決策

```forth
\ 相対アドレスを使用
\ または、アドレスを別の場所に保存し、
\ 実行時にロード
```

### 4.4 制限事項のまとめ

```
【保存可能】
✓ 辞書内のワード定義
✓ 変数とその値
✓ 定数
✓ 単純なポインタ

【保存不可能】
✗ ALLOCATE/FREEで管理されるメモリ
✗ スタックの状態
✗ 開いているファイルハンドル
✗ アドレスを計算した結果
✗ 機械語コード内の絶対アドレス
```

## 5 ファイルの読み込み（include/require）

### 5.1 基本概念

Forthプログラムは通常、複数のソースファイルに分割されます。ファイルを読み込む方法は2つあります。

#### include：無条件に読み込む

```forth
include myfile.fs
```

特徴
- 常にファイルを読み込む
- 同じファイルを複数回読み込み可能
- シンプルだが重複のリスク

#### require：一度だけ読み込む（推奨）

```forth
require myfile.fs
```

特徴
- 既に読み込まれていればスキップ
- 重複を自動的に防止
- モジュール化に最適

### 5.2 includeの詳細

動作メカニズム

```
1. ファイル名を解析
2. ファイルを開く
3. 内容をテキストインタプリタに渡す
4. ファイルを閉じる
```

文字列版

```forth
s" filename.fs" included
```

これは`include`と等価ですが、ファイル名を動的に生成できます。

使用場面

- テストスクリプト（毎回実行したい）
- データファイル（内容が変わる）
- 明示的な再読み込み

注意点

同じファイルを複数回includeすると、ワードが再定義されます。

```forth
\ library.fs
: helper ." Version 1" ;

\ main.fs
include library.fs  \ helper定義
helper              \ "Version 1"
include library.fs  \ helper再定義（警告が出る）
helper              \ "Version 1"（同じ動作）
```

### 5.3 requireの詳細

#### 動作メカニズム

```
1. ファイル名を完全パスに解決
2. 読み込み済みリストを確認
3. リストになければ。
   a. ファイルを読み込む
   b. リストに追加
4. リストにあればスキップ
```

#### ファイル同一性の判定

requireはフルパスで同一性を判定します。

```forth
require ./lib/utils.fs     \ 読み込まれる
require lib/utils.fs       \ 異なるパスとみなされ、再度読み込まれる可能性
```

文字列版

```forth
s" filename.fs" required
```

別名

```forth
needs filename.fs  \ requireと同じ
```

Win32Forthなど他のシステムとの互換性のために存在します。

### 5.4 パラメータ付き読み込み

ファイルにパラメータを渡すこともできますが、スタック効果に注意が必要です。

原則

ファイルは読み込み前後でスタックを変更すべきではありません。

良い例

```forth
\ config.fs - スタック効果 ( n -- n )
dup constant buffer-size  \ 受け取った値を定数として保存

\ main.fs
1024 require config.fs drop  \ パラメータを渡す
```

注意点

最初の`require`で渡されたパラメータが、すべての用途で有効である必要があります。

```forth
1024 require config.fs drop  \ buffer-size = 1024
2048 require config.fs drop  \ スキップされる、buffer-sizeは1024のまま
```

### 5.5 読み込み状態の確認

included?：ファイルが読み込まれているか確認

```forth
s" myfile.fs" included?  \ ( c-addr u -- flag )
```

sourcefilename：現在のファイル名

```forth
sourcefilename  \ ( -- c-addr u )
```

実行例

```forth
\ debug.fs
: show-location
  ." Loading from: " sourcefilename type cr
;
show-location
```

sourceline#：現在の行番号

```forth
sourceline#  \ ( -- u )
```

実行例

```forth
: error-msg
  ." Error at line " sourceline# . cr
;
```

### 5.6 includeとrequireの選択基準

```
【require を使用】（推奨）
✓ ライブラリファイル
✓ モジュール
✓ 共通ユーティリティ
✓ 依存関係のあるファイル

【include を使用】
✓ テストスクリプト
✓ データファイル
✓ 明示的に再実行したいコード
✓ 開発中の頻繁に変更するファイル
```

## 6 初期化ファイル（gforthrc）

### 6.1 初期化ファイルの役割

gforthrcは、Gforthの個人設定ファイルです。エディタの`.vimrc`や`.emacs`と同じ役割を果たします。

用途

- 個人的なワード定義
- カスタムプロンプト
- パス設定
- エイリアス
- デバッグヘルパー
- 環境のカスタマイズ

重要な原則

gforthrcは個人的な設定のみを記述すべきです。プロジェクト固有の設定は別のファイルに分離します。

### 6.2 初期化ファイルの種類

Gforthには2種類の初期化ファイルがあります。

#### gforthrc0：早期初期化

場所：`~/.config/gforthrc0`（または環境変数`GFORTH_ENV`で指定）

読み込みタイミング
```
起動 → イメージ読み込み → ★gforthrc0★ → 引数処理 → gforthrc → REPL
```

用途
- パス設定
- 環境変数
- 最も基本的な設定

#### gforthrc：標準初期化

場所：`~/.config/gforthrc`

読み込みタイミング
```
起動 → イメージ読み込み → gforthrc0 → 引数処理 → ★gforthrc★ → REPL
```

用途
- ユーザー定義ワード
- エイリアス
- カスタムプロンプト
- ヘルパー関数

### 6.3 gforthrcの作成

最小限の例

```forth
\ ~/.config/gforthrc

\ クリア画面のエイリアス
: cls page ;

\ 起動メッセージ
." Custom Forth Environment Loaded" cr
```

実用的な例

```forth
\ ~/.config/gforthrc

\ ========== パス設定 ==========
s" ~/forth-libs" fpath also>string fpath 1 !

\ ========== エイリアス ==========
: cls page ;
: ? @ . ;

\ ========== デバッグ ==========
variable debug-mode
: debug-on debug-mode on ;
: debug-off debug-mode off ;

\ ========== 定数 ==========
1024 constant KB
1024 KB * constant MB

\ ========== 起動メッセージ ==========
cr ." My Forth Ready" cr
```

### 6.4 環境変数による制御

#### GFORTH_ENV

早期初期化ファイルを指定。

```bash
export GFORTH_ENV=~/my-init.fs
gforth
```

無効化。

```bash
export GFORTH_ENV=off
gforth
```

--no-rcオプション

すべての初期化ファイルをスキップ。

```bash
gforth --no-rc
```

### 6.5 プロジェクト固有の初期化

ディレクトリ構造

```
myproject/
├── .gforthrc0          # プロジェクト専用初期化
├── src/
└── lib/
```

プロジェクト専用の.gforthrc0

```forth
\ myproject/.gforthrc0

\ プロジェクトのlibをパスに追加
s" ./lib" fpath also>string fpath 1 !

\ プロジェクトモード表示
." [MyProject Mode]" cr
```

使用方法

```bash
cd myproject
export GFORTH_ENV=./.gforthrc0
gforth
```

## 7 検索パス（GFORTHPATH）

### 7.1 検索パスの概念

GFORTHPATHは、Forthがファイルを検索するディレクトリのリストです。環境変数`PATH`のForth版と考えてください。

デフォルト

```
.:~/share/gforth/site-forth:/usr/local/share/gforth/site-forth:/usr/local/share/gforth/VERSION
```

構成
- `.`：カレントディレクトリ
- `~/share/gforth/site-forth`：ユーザーライブラリ
- `/usr/local/share/gforth/site-forth`：システム共有ライブラリ
- `/usr/local/share/gforth/VERSION`：Gforth標準ライブラリ

### 7.2 検索の仕組み

ファイルを`require`すると、以下の順序で検索されます。

```
require mylib.fs を実行

1. ./mylib.fs を確認
   ↓ 見つからない
2. ~/share/gforth/site-forth/mylib.fs を確認
   ↓ 見つからない
3. /usr/local/share/gforth/site-forth/mylib.fs を確認
   ↓ 見つかった
   読み込み完了
```

相対パス vs 絶対パス

```forth
\ 相対パス（GFORTHPATHで検索）
require utils.fs

\ 絶対パス（直接指定）
require /home/user/myproject/lib/utils.fs
```

### 7.3 パスの設定方法

環境変数で設定（永続的）

```bash
## .bashrcに追加
export GFORTHPATH=".:~/mylibs:$GFORTHPATH"
```

コマンドラインで設定（一時的）

```bash
gforth --path ".:~/mylibs:/usr/local/lib/forth"
```

Forthコード内で設定

```forth
s" ~/mylibs" fpath also>string fpath 1 !
```

### 7.4 パス管理のベストプラクティス

#### 原則1：カレントディレクトリを常に含める

```bash
export GFORTHPATH=".:その他のパス"
```

#### 原則2：プロジェクトごとにパスを分離

```bash
cd project1
export GFORTHPATH=".:./lib:$GFORTHPATH"
gforth
```

#### 原則3：ハードコードを避ける

```forth
\  悪い例
require /home/user/myproject/lib/utils.fs

\  良い例
require lib/utils.fs  \ GFORTHPATHでプロジェクトルートを含める
```

## 8 起動シーケンスの全体像

### 8.1 完全な起動フロー

Gforthの起動は9つのフェーズから構成されます。

```
┌──────────────────────────────────┐
│ フェーズ1：エンジン起動           │
│   C言語プログラムの実行開始       │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ2：イメージ読み込み        │
│   gforth.fi（またはパス指定）     │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ3：アドレス再配置          │
│   データ・コードの調整            │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ4：メモリ初期化            │
│   スタック、辞書の設定            │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ5：gforthrc0実行 ★        │
│   早期初期化                     │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ6：コマンドライン処理      │
│   -e、ファイル引数など            │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ7：gforthrc実行 ★         │
│   ユーザー設定                   │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ8：ブートメッセージ        │
│   バージョン情報など              │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ フェーズ9：REPL/スクリプト実行     │
│   対話モードまたは処理実行        │
└──────────────────────────────────┘
```

### 8.2 起動のカスタマイズポイント

#### 'bootmessageフック

ブートメッセージをカスタマイズ。

```forth
\ gforthrc内
: my-message
  cr ." === My Forth ===" cr
;
' my-message is bootmessage
```

#### 'coldフック

起動時の処理を追加（イメージファイル内）。

```forth
: my-startup
  init-database
  load-config
;
script? [IF]
  ' my-startup is 'cold
[THEN]
```

## 9 実践的な使用パターン

### 9.1 開発環境の構築

ディレクトリ構造

```
~/forth-dev/
├── .gforthrc0
├── .gforthrc
├── projects/
│   ├── project1/
│   └── project2/
└── libs/
    └── common/
```

.gforthrc0（パス設定）

```forth
s" ~/forth-dev/libs" fpath also>string fpath 1 !
```

.gforthrc（ツール）

```forth
: reload  s" main.fs" included ;
: test-all require tests/all.fs ;
```

### 9.2 プロジェクトテンプレート

最小構成

```
myproject/
├── main.fs
├── lib/
│   └── utils.fs
└── Makefile
```

Makefile

```makefile
IMAGE = myproject.fi

$(IMAGE): main.fs lib/utils.fs
	gforthmi $(IMAGE) main.fs

clean:
	rm -f $(IMAGE)

run: $(IMAGE)
	./$(IMAGE)
```

## 10 トラブルシューティング

### 10.1 よくあるエラー

#### 「File not found」

原因：GFORTHPATHにディレクトリが含まれていない

解決

```bash
export GFORTHPATH=".:./lib:$GFORTHPATH"
```

#### 「Invalid memory address」

原因：非再配置イメージの使用

解決

```bash
gforthmi myapp.fi myapp.fs  # 完全再配置イメージを作成
```

#### 「Redefined word」

原因：同じファイルを複数回include

解決

```forth
\ includeをrequireに変更
require mylib.fs
```

### 10.2 デバッグテクニック

パスの確認

```forth
s" GFORTHPATH" getenv type cr
```

読み込み状態の確認

```forth
s" mylib.fs" included? . 
```

現在位置の表示

```forth
: where
  sourcefilename type 
  ." :" sourceline# . cr
;
```

## 11 まとめ

### 11.1 重要ポイント

イメージファイル
- 常に完全再配置イメージ（gforthmi）を使用
- 非再配置イメージは避ける
- アプリケーション配布には--applicationを使用

ファイル読み込み
- ライブラリはrequire、テストはinclude
- スタック効果を保つ
- フルパスでの一貫性に注意

初期化ファイル
- 個人設定はgforthrcに
- プロジェクト設定は別ファイルに
- 環境変数で柔軟に制御

検索パス
- カレントディレクトリを含める
- プロジェクトごとに設定
- 相対パスを優先

### 11.2 推奨ワークフロー

```
【開発フェーズ】
1. gforthrcで個人環境を設定
2. requireでライブラリを管理
3. REPLで対話的に開発

【テストフェーズ】
1. テストスイートを作成
2. Makefileで自動化
3. CIでテスト実行

【デプロイフェーズ】
1. gforthmiでイメージ作成
2. 必要なサイズを指定
3. --applicationで配布用に
```

