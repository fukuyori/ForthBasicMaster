# Lev 2 ー gforth開発環境構築
FORTH学習をスムーズに始めるために、Windows＋Visual Studio Code＋WSL環境でgforthを導入・設定し、基本的な動作確認を行う。説明は大雑把にするので、詳細が必要な場合は各自調べてください。

## 1. 開発環境の全体像

本講座では、以下の構成でgforthを利用する。

* **エディタ:** Visual Studio Code（VS Code）
* **実行環境:** Windows Subsystem for Linux（WSL2, Ubuntu 20.04 以降）
* **FORTH処理系:** gforth 0.7.9（公式GNU Forthディストリビューション）

この構成の利点は次のとおりである。

* Windows上でLinux版gforthを使用できるため、UNIX的ツールとの連携が容易。
* VS Codeを利用してFORTHコードの編集・実行が可能。

## 2. WSL（Ubuntu）の導入手順

1. Windows Subsystem for Linux

   * Windows Subsystem for Linuxを有効にする

2. Ubuntuのインストール

   * Microsoft Storeから「Ubuntu xx.xx LTS」を検索しインストール。
   * 初回起動時にユーザー名とパスワードを設定。

3. **基本更新**

   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

## 3. gforth のインストール方法

```bash
sudo apt install gforth -y
```

WSLでは 0.7.9_20250321 だった。（2025年11月時点）

Ubuntu標準リポジトリのgforthは0.7.3で止まっているようだ。このバージョンだと、FORTH標準のほとんどは動作するが、gforthのドキュメントで書かれている機能の一部が動作しない。それは、gforthの独自機能の場合もあるし、FORTH標準機能の場合もある。

できれば、0.7.9 の方が使い勝手がいい。

Windowsのバイナリが提供されているのは、0.7.0 と、かなり古いものになっている。DJGPPでコンパイルできるとあるのだが、情報が見当たらない。

## 4. Visual Studio Code の設定

1. **拡張機能を導入**

   * `Forth`（作者: Filippo Tortomasi など）

2. **WSL環境に接続**

   * VS Codeのターミナルを表示させ、Ubuntu（WSL）を選択。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/864075/581cfef2-2e91-431d-a708-fb204e16eeda.png)

3. **コード実行テスト**
   新規ファイル `hello.fs` を作成し、以下を記述して保存。

   ```forth
   CR ." Hello, Forth world!"
   ```

   VS Codeのターミナルで：

   ```bash:入力
   gforth hello.fs
   ```

   あるいは、ターミナルでgforthを起動した状態で

   ```bash:入力
   include hello.fs
   ```
   
   ```:出力
   Hello, Forth world!
   ```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/864075/bea12a86-73e8-46c5-9d93-fedefde31723.png)

gforthの操作を行う場合、対話モードでスタックなどを確認しながら進めると理解しやすい。しかし、ワード定義など、一連の動作をまとめて実行させたい場合は、ファイルに記述して`include`を使った方が作業が簡単になる。


## 5. gforthの終了方法

  ```
  bye
  ```

  あるいは、行の先頭で `Ctrl + D` を押す。

## 6. 注意点

ソースファイルの保存形式をUTF-8に統一すること。

gforthは低レベルなメモリアクセスを許すため、不完全な操作によって無限ループやシステム不安定化を起こすことがある。
その場合は、`Ctrl + C` による中断、または `Ctrl + D` による終了を行う。
動作が不安定なときは、一度終了して再起動すること。

:::note warn
FORTHはアドレス操作をユーザーが直接行う言語である。
範囲外アクセスや未初期化領域アクセスを防ぐ仕組みはなく、
C言語と同様にユーザー自身の責任でメモリを管理する必要がある。
:::



## GitHubから最新版を取得

Ubuntu標準リポジトリはgforthは0.7.3（2025年11月現在）なので、0.7.9 を使うにはGitHubリポジトリからクローンしてコンパイルする。Windows 11のWSL上でもコンパイルできる。


### 1. 依存パッケージのインストール

まず、gforthのビルドに必要な開発ツールとライブラリをインストールする。

```bash
sudo apt update
sudo apt install -y git build-essential autoconf automake libtool texinfo libffi-dev
```

### 2. GitHubリポジトリをクローン

GNU公式のgforthリポジトリを取得する。

```bash
git clone https://github.com/forthy42/gforth.git
cd gforth
```

最新版のソースコードは、以下のサイトで公開されている。

https://cgit.git.savannah.gnu.org/cgit/gforth.git

https://github.com/forthy42/gforth

https://github.com/earl/gforth-mirror

2025年11月時点で **0.7.9_20251119** が公開されている。

### 3. BUILD-FROM-SCRATCHの実行

gforthはAutotoolsを使ってビルドされるが、そのすべてを自動で処理してくれるスクリプトが `BUILD-FROM-SCRATCH` である。このスクリプトは **コンパイル・設定・インストールを一括で実行** し、Autoconf警告（例：`AC_HEADER_STDC is obsolete`）を回避できる。

:::note warn
既に`sudo apt install`でgforthをインストールしている場合は、`sudo apt autoremove gforth`で削除しておくこと。
:::

#### 実行手順：

```bash
./BUILD-FROM-SCRATCH
```

完了後、次でバージョンを確認：

```bash
gforth --version
```

```:出力例
gforth 0.7.9_20251001 amd64
```

コンパイルが終わったら、以下のコマンドでインストールを行う。

```bash:インストール
sudo make install
```

### 4. REPL

インタラクティブに起動して動作確認します：

```bash
gforth
```

gforthが起動すると、プロンプトとして ok が現れる。これがFORTHのREPL（Read-Eval-Print Loop）であり、ここでワードを直接入力し、即座に結果を確認できる。

たとえば `2 3 + .` と入力すると、スタックに積まれた「2」と「3」を加算し、結果の「5」が表示される。.S を使えば現在のスタックの状態を表示でき、: SQUARE DUP * ; と定義すれば新しい命令（ワード）が即座に環境へ追加される。

以下は、WindowsのWSL(Ubuntu)での実行画面で、赤線の部分がユーザー入力した箇所になる。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/864075/a6465ac5-82d9-498a-adde-e48df2055995.png)

このバージョンでは、`gforthrc` を自動で読み込んだり、画面下のステータスバーにスタックの状態が表示される。


プロンプトが表示されたら次を入力：

```:入力
1 2 + .
```

```:出力
3  ok
```

終了は：

```
bye
```

REPL上で定義・実行・確認を繰り返すことで、プログラマはFORTHを単なる言語ではなく、自分で拡張していく「思考の実験場」として扱うことができる。わずか数秒でコンパイルから実行まで完結し、環境の中で言語そのものが成長していく――それがFORTHの学習体験の魅力である。

## gforthrc

gforth 0.7.9 では、起動のたびに指定された一連の処理を自動で実行させることが出来る。gforthのコマンドラインで`help`と入力すると最後2行に説明がある。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/864075/0810275d-90a5-4503-a724-e7fd9bbcf04b.png)

日本語訳
>**Gforthマニュアル**は、[http://gforth.org/manual/](http://gforth.org/manual/) で利用できます。または、お好みの情報リーダー（例: シェルから `info gforth`）を使用してください。また単に `help <word>` と入力することで、該当ワードのヘルプを表示することもできます。
>Gforthを終了するには、`bye` と入力するか、行の先頭で `Ctrl + D` を押してください。
>浮動小数点数を入力する場合は、指数表記を使用します。例: `1e`。
>倍長整数（double-cell integers）を警告なしで入力するには、基数プレフィクスを付け、小数点 `.` は最後の位置のみに置いてください。例: `#1.`。
>暗い背景色に適した色表示にするには、`dark-mode` と入力します。
>警告を無効化するには `warnings off` を入力し、
>警告を最小限に抑えるには `-1 warnings !` を入力します。
>`~/.config/gforthrc0` は、OSコマンドラインを処理する前に読み込まれます。
>`~/.config/gforthrc` は、Forthコマンドラインに入る前に読み込まれます。


問題は、`~` で表示されているホームディレクトリがどこにあるかだが、以下のコマンドを使えば分かる。

```
s" HOME" GETENV TYPE
```

私は、ホームディレクトリ以下の `.config/gforthrc` に、以下のような定義を書き込み、よく使うワードを登録している。

```:gforthrc
\ スタックの内容を全て加算
: sum
  begin depth 1 > while
    +
  repeat ;

\ 浮動小数点スタックの内容を全て加算
: fsum 
  begin fdepth 1 > while
    f+
  repeat ;

\ スタックの内容をすべて消す
: clear
  begin depth 0> while
    drop
  repeat
  begin fdepth 0> while
    fdrop
  repeat ;

\ 配列の内容を表示
: .ARRAY ( addr n -- )
  0 DO DUP I CELLS + @ . LOOP DROP CR ;

\ カレントディレクトリのファイルを表示
: LS
    S" ls -l" SYSTEM ;
```

