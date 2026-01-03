# Lev 1 ー FORTHとは何か

## FORTHの生い立ち

チャールズ・H・ムーア（Charles H. Moore）は、MIT在学中からスミソニアン天体物理観測所でプログラミングを始めていた人物である。1961年に物理学の博士号を取得したのち、スタンフォード大学大学院で数学を専攻したが、1965年に中退し、フリーランスのプログラマーとして活動を始めた。

1970年頃、アリゾナ州のキット・ピーク国立天文台において電波望遠鏡の制御システムを開発する必要が生じ、ムーアは独自のプログラミング言語「FORTH」を創案した。言語名の由来は「fourth generation（第4世代）」という言葉にあるが、当時使用していたコンピュータが大文字5文字までしかファイル名を扱えなかったため、「FORTH」と短縮して綴られたものである。

FORTHは次のような革新的特徴を備えていた。

- **スタックベースのアーキテクチャ**：逆ポーランド記法（RPN）を採用し、演算の明快化を実現
- **インタプリタとコンパイラの統合**：入力即実行が可能で、対話的な開発環境を提供
- **小さなフットプリント**：限られたメモリ空間でも高速に動作
- **拡張性**：ユーザーが容易に新しい命令（ワード）を定義し、言語を自ら拡張可能

この特性により、FORTHは組み込みシステムやリアルタイム制御の分野で注目を集めた。ムーアは1973年に**FORTH, Inc.** を設立し、商用版FORTHの開発と普及に努めた。

1983年には「FORTH-83」標準が策定され、多くのパーソナルコンピュータやワークステーションにFORTH環境が実装された。また、NASAのボイジャー探査機やスペースシャトルのサブシステムなど、数多くの宇宙開発ミッションにも利用されている。FORTHの軽量性と信頼性は、過酷な環境での制御システムに最適であった。

その後も標準化は続き、1994年には米国規格協会（ANSI）による「ANS FORTH」が制定され、2012年には「Forth 2012」として改訂版が発表された。現在に至るまで、FORTHはマイコン、組込みボード、リアルタイム制御装置などの領域で現役で使われ続けている。

FORTHは決して主流言語にはならなかったが、その設計思想は多くの後続技術に影響を与えた。特にスタックベースの仮想マシン設計――たとえばJavaのJVMやPostScriptの実行モデル――にFORTHの理念が色濃く残っている。ムーアの目指した「小さくとも完全なシステム」は、今日のミニマル言語や組込みOSにも受け継がれている。


## 現在利用可能なFORTHシステム

FORTHを学ぶ第一歩は、実際に動く環境を手に入れることから始まる。現代ではいくつかの処理系が維持・発展しており、用途に応じて選択できる。以下では、デスクトップPC向けとマイコン向けに分けて紹介する。

#### デスクトップ向け（Windows / Linux / macOS）

**gforth**
GNU Projectの公式FORTH実装であり、ANS Forth標準に準拠している。オープンソースで広く普及しており、学習・教育・研究用途に最適である。ほとんどのUNIX系環境では `sudo apt install gforth` だけで導入できる。本シリーズでもgforthを使用する。
- 公式サイト: https://www.gnu.org/software/gforth/

**4tH**
Hans Bezemer氏が開発した教育向けForthコンパイラ。従来のForthエンジンではなく、バイトコードを生成する設計を採用している。エラー検出機能が充実しており、Forthを「正しい作法」で学ぶのに適している。Windows、Linux、macOSなど多くのプラットフォームで動作し、生成したバイトコードはプラットフォーム間で移植可能である。
- 公式サイト: https://thebeez.home.xs4all.nl/4tH/

**pForth**
Phil Burk氏が開発した、ANSI Cで書かれた移植性の高いANS Forth実装。特別な前処理なしにコンパイルでき、Windows、Linux、macOS、Raspberry Piなど多様な環境で動作する。組込みシステムへの移植も容易で、stdlib呼び出しなしでのコンパイルも可能である。
- GitHub: https://github.com/philburk/pforth

**SwiftForth**
FORTH, Inc.が開発する商用の統合開発環境。Windows、Linux、macOSに対応し、最適化されたネイティブコード生成と高度なデバッグ機能を備える。産業用制御やリアルタイム処理を目的とするユーザーに向く。無料評価版が提供されている。
- 公式サイト: https://www.forth.com/swiftforth/

#### マイコン向け

FORTHは組込みシステムの分野で長い歴史を持ち、現在も多くのマイコンボードで利用できる。対話的な開発スタイルは、ハードウェアの動作確認やプロトタイピングに特に有効である。

**Raspberry Pi Pico / RP2040**

- **Mecrisp-Stellaris Forth**: Matthias Koch氏が開発したARM Cortex-M向けForth。RP2040に対応しており、UF2ファイルをドラッグ＆ドロップするだけでインストールできる。フラッシュメモリへの書き込みやGPIO制御が可能。
  - 配布元: https://sourceforge.net/projects/mecrisp/

- **zeptoforth**: RP2040、RP2350、STM32など複数のCortex-Mボードに対応した本格的なForthシステム。プリエンプティブなマルチタスカー、FAT32ファイルシステム、IPv4/IPv6スタック（Pico W対応）など豊富な機能を備える。
  - GitHub: https://github.com/tabemann/zeptoforth

**Arduino / AVR**

- **FlashForth**: Mikael Nordman氏が開発した、PICおよびAVR ATmega向けのForth。Arduino Uno、Nano、Mega 2560などで動作する。フラッシュメモリへのワード保存、マルチタスク機能を備え、シリアル接続で対話的に開発できる。ISPプログラマを使用してインストールする。
  - 公式サイト: https://flashforth.com/
  - 解説サイト: https://arduino-forth.com/

- **AmForth**: AVR ATmegaおよびTI MSP430向けのForth。Forth 2012標準に近い仕様で、Arduino Leonardoなど多くのボードに対応している。
  - 公式サイト: http://amforth.sf.net/

**ESP32 / M5Stack**

- **ESP32forth**: Dr. C.H. Ting氏のeForthをベースに、Brad Nelson氏が開発したESP32向けForth。Arduino IDEからスケッチとしてコンパイル・書き込みが可能で、WiFi経由のWebインターフェースも備える。シリアル接続またはブラウザから対話的に操作できる。
  - 公式サイト: https://esp32forth.appspot.com/
  - コミュニティ: https://esp32forth.forth2020.org/

- **M5Stack向け拡張**: ESP32forthをM5Stackで使用するための拡張プロジェクトが存在する。ボタン、IMU、LCDなどM5Stack固有のハードウェアにForthからアクセスできる。QWERTYキーボード付きのM5Stack Facesと組み合わせることで、ポータブルなForth開発環境を構築可能。
  - GitHub: https://github.com/Esp32forth-org/jasontay

### 本シリーズでの環境

このシリーズでは、入手の容易さと豊富なドキュメントを考慮し、**gforth**を使用する。

gforthはGNU Projectの一部として開発されており、オープンソースで広く普及している。学習・教育・研究用途に最適で、ほとんどのUNIX系環境では `sudo apt install gforth` だけで導入できる。もう一方のVFX Forthは、イギリスのMPE社が開発する商用・高性能版であり、最適化されたコンパイル結果や高速浮動小数点演算を特徴とする。産業用制御やリアルタイム処理を目的とするユーザーに向く。

動作を確認しているのは、**0.7.9**のバージョンなので、最新版を入手して欲しい。

https://gforth.org/


## FORTHの学習に有効な書籍・サイト

### 書籍（無料でオンライン公開されているもの）

#### 入門書（最初に読むべき）

**Starting FORTH** - Leo Brodie著
FORTHの古典的入門書で、1981年に初版が出版された。手描きのイラストを多用した親しみやすい解説で、プログラミング未経験者でも理解できるよう書かれている。現代のANS Forth標準に合わせて更新されたバージョンがオンラインで公開されている。**最初に読むべき一冊**として多くのForthプログラマが推奨している。
- オンライン版: https://www.forth.com/starting-forth/0-starting-forth/
- PDF版: https://www.forth.com/wp-content/uploads/2018/01/Starting-FORTH.pdf

#### 中級・設計思想

**Thinking FORTH** - Leo Brodie著
FORTHを使った問題解決の哲学とプログラミングスタイルを解説した本。単なる言語解説ではなく、ソフトウェア設計の考え方を扱っている。ファクタリング、モジュール性、ボトムアップ設計など、現代のエクストリームプログラミングで再発見された原則が多く含まれている。FORTH開発者Charles H. Mooreへのインタビューも収録。Creative Commonsライセンスで公開されている。
- オンライン版: https://thinking-forth.sourceforge.net/
- **日本語訳**: https://thinking-forth-ja.readthedocs.io/ja/latest/

#### その他の書籍

**And So Forth** - Hans Bezemer著
4tHコンパイラの作者によるForth入門書。
- http://win32forth.sourceforge.net/doc/Forth_Primer.pdf

**A Beginner's Guide to Forth** - J.V. Noble著（バージニア大学）
コンパクトながら実用的なForth入門。
- https://galileo.phys.virginia.edu/classes/551.jvn.fall01/primer.htm

**Programming Forth** - Stephen Pelc著
ANS Forthシステムへの包括的な入門書です。プログラミング経験はあるがForth未経験の読者向けに、基本原則からマルチタスク、組み込みシステム向けクロスコンパイルなどの高度な概念まで網羅しています。
- https://vfxforth.com/arena/ProgramForth.pdf

**Forth Application Techniques** - FORTH, Inc.
基礎から上級まで、実践的なForthプログラミングを学べます。Forth基礎を紹介し、ソフトウェアソリューション作成のための一般的知識を養うビルディングブロック演習を課します。 
- https://amzn.asia/d/6DDZlkN

**Forth Programmer's Handbook** - FORTH, Inc.
どのForth実装を使用していても、事実上のANS Forthリファレンスマニュアルとしての評価を得ています。広範なForth言語用語集と、一般的・高度なプログラミング主題について丁寧で明快な解説を含みます。
- https://amzn.asia/d/gcIwVYb



### 公式ドキュメント・リファレンス

**Gforth Manual**
Gforthの公式マニュアル。リファレンスとしてだけでなく、Forthの入門とチュートリアルも含まれている。
- https://gforth.org/manual/
- チュートリアル: https://gforth.org/manual/Tutorial.html

**Forth 2012 Standard**
最新のForth標準仕様書です。困ったときはまずこれを参照。
- オンライン: https://forth-standard.org/
- PDF: https://archive.org/download/forth_2012_rc2/forth_2012_rc2.pdf

### 日本語リソース

**Thinking FORTH 日本語訳**
- https://thinking-forth-ja.readthedocs.io/ja/latest/

**Wikipedia日本語版「Forth」**
概要を把握するのに有用。
- https://ja.wikipedia.org/wiki/Forth

**日本語解説サイト・ブログ記事**
kuma35氏によるGforth 0.7.9_20240418 のドキュメント日本語訳
- https://kuma35.github.io/gforth-docs-ja/docs-ja-0/index.html
- ページ分割版: https://kuma35.github.io/gforth-docs-ja/docs-ja-0/gforth/index.html
- 一括ページ版: https://kuma35.github.io/gforth-docs-ja/docs-ja-0/gforth-no-split/gforth.html

**絶版書籍**
- **「標準FORTH」** 
    - 井上外志雄 著  共立出版, 1985.8
- **「はじめて学ぶFORTHプログラミング入門」** 
    - オーウェン・ビショップ 著 荒 実 / 玄 光男 共訳 啓学出版, 1987.3
- **「パソコン・ユーザのためのFORTH入門」** 
    - A.ウィンフィールド 著 寺島 元章 訳 近代科学社, 1989.3
- **「FORTH入門 (I/O books)」** 
    - レオ・ブロディー 著 原 道宏 訳 工学社, 1984.4

### コミュニティ

**Forth2020**
ESP32forthを中心としたForthコミュニティ。月1回程度ZOOMミーティングを開催しており、FORTH開発者のCharles H. Moore氏も参加することがある。Facebookのプライベートグループで活動。
- https://www.forth2020.org/
- Facebookグループ: https://www.facebook.com/groups/forth2020/

**comp.lang.forth**
Forthに関するUsenetニュースグループ（Google Groupsからアクセス可能）。


### 動画チュートリアル

**Gforth解説動画（YouTube）**
英語だがGforthの使い方を解説した動画シリーズがある。
- https://www.youtube.com/playlist?list=PLGY0au-Sczllm2NcYC0Ypk2CYcFAadYZa

---

## 推奨学習順序

1. **Starting FORTH** を読みながらgforthで実際に試す
2. **Gforth Manual**のチュートリアルセクションで補完
3. **Thinking FORTH** で設計思想を学ぶ
4. **Forth 2012 Standard** をリファレンスとして活用
4. **Forth Programmer's Handbook** - リファレンスとして常備
4. **Programming Forth** - 実践的なANS Forth技法
4. **Moving Forth / JonesForth** - 処理系実装に興味がある場合

