# Online.sg: LLVM

2010年2月21日(日)

Tatsuhiro Ujihisa

## LLVM

> The Low Level Virtual Machine (LLVM) is a compiler infrastructure, written in C++, which is designed for compile-time, link-time, run-time, and "idle-time" optimization of programs written in arbitrary programming languages. LLVM was originally developed as a research infrastructure at the University of Illinois at Urbana-Champaign to investigate dynamic compilation techniques for static and dynamic programming languages...

<http://en.wikipedia.org/wiki/Low_Level_Virtual_Machine>

## In short,

CのためのJVM

ただし、gccでコンパイルした実行可能ファイルより速く動く (ことが多い)

最適化がスゴい (後述)

## C言語のコードの実行方法

C → アセンブラ → 機械語 → 実行

    $ vim a.c
    $ gcc a.c -S -o a.s
    $ gcc a.s -o a
    $ ./a

## LLVMの実行方法 (1)

C → LLVMアセンブラ → LLVMビットコード → インタプリタで実行

    $ vim a.c
    $ llvm-gcc a.c -S -o a.ll
    $ llvm-as a.ll -o a.bc
    $ lli a.bc

## LLVMの実行方法 (2)

C → LLVMアセンブラ → LLVMビットコード → 機械語 → 実行

    $ vim a.c
    $ llvm-gcc a.c -S -o a.ll
    $ llvm-as a.ll -o a.bc
    $ llc a.bc -o a
    $ ./a

## LLVMの実行方法 (3)

LLVMアセンブラ → LLVMビットコード → インタプリタで実行

    $ vim a.ll
    $ llvm-as a.ll -o a.bc
    $ lli a.bc

## LLVMの実行方法 (4)

LLVMアセンブラ → LLVMビットコード → 最適化 → インタプリタで実行

    $ vim a.ll
    $ llvm-as a.ll -o a.bc
    $ opt a.bc -o a2.bc
    $ lli a2.bc

## LLVMの実行方法 (5)

LLVMアセンブラ → LLVMビットコード → 最適化 → 最適化 → インタプリタで実行

    $ vim a.ll
    $ llvm-as a.ll -o a.bc
    $ opt -O3 a.bc -o a2.bc
    $ opt -O3 a2.bc -o a3.bc
    $ lli a3.bc

## LLVMの最適化の確認

逆アセンブル

    $ llvm-as a.ll -o a.bc
    $ opt -O3 a.bc -o a2.bc
    $ opt -O3 a2.bc -o a3.bc
    $ llvm-dis a3.bc -o a3.ll
    $ vim a3.ll

## 整理

* `llvm-as`: LLVMアセンブリ言語ファイル(.ll) から LLVMビットコード(.bc)に変換
* `lli`: LLVMビットコード(.bc)を実行
* `opt`: LLVMビットコード(.bc)を最適化し、別のLLVMビットコード(.bc)を生成
* `llvm-dis`: `llvm-as`の逆
* `llvm-gcc`: C言語ファイル(.c)からLLVMアセンブリ言語(.ll)に変換

## LLVMアセンブリ言語でHello, world!

Hello, world!

    @str = internal constant [14 x i8] c"Hello, world!\00"
    declare i32 @puts(i8*)
    define i32 @main()
    {
      call i32 @puts( i8* getelementptr ([14 x i8]* @str, i32 0,i32 0))
      ret i32 0
    }

## 分かりやすくCで説明してみる

    char str[14] = "Hello, world!";
    int main() {
      puts(str);
      return 0;
    }

## LLVMアセンブリ言語の特徴

* 逐次処理
* 再代入禁止 (全ては定数)
* Cの関数は大抵そのまま呼べる

## ちなみに

VimのquickrunはLLVMアセンブリ言語対応済み

.llなファイルを編集中に`<Leader>r`するだけでllvm-asとlliしてくれる

## LLVMの用途と目的

コンパイラを作る人のための道具。

* 新しいコンパイル型言語を作るなら、LLVMアセンブリ言語にさえ変換すればOK (C言語経由でもOK)
* LLVMならばMac, Linux, Windowsで確実に動く上に、かなり速い。

## LLVM化された(らしい)言語処理系

* C (llvm-gcc)
* Perl
* Python (pypy)
* Ruby (Rubinius, MacRuby, etc)
* Haskell
* Brainf\*\*k

... LLVM対応されていない言語を探す方が難しい

## LLVM前提で作られた言語

Pure

* 動的型付け
* 関数型 (項書き換え)
* ユーザ定義文法、マクロ
* Haskell風の文法
* `sudo port install pure`

![pure](http://pure-lang.googlecode.com/svn/wiki/waterdrop.png)

## ここまでのまとめ

* LLVMは速くて便利
* コンパイラを作るならLLVMを使おう
* 既にLLVMを使ったコンパイラがたくさん

## 問題点

* まだ混沌 (最適化の二度漬けなど)
* LLVMを使った処理系をビルドするのが難しいときも (MacRuby)
* 変化がすごすぎる (終わらない`svn up`)

## 実践!

Brainf\*\*k → LLVMアセンブリ言語

* BFC: Brainf\*\*k Compiler
* `git clone git://github.com/ujihisa/bfc.git`
* `vim bfc/bfc.rb`

## 変換例: +

`+`: ポインタが示すメモリ位置のデータをインクリメント

    /* Cでいうと、 */
    /* char *hがあるとして */
    ++*h;

LLVMは変数の値を書き換えれない! → 変数としてはポインタだけを使えばとりあえずOK

## コンパイラ実装例 (bfc.rbより抜粋)

    when '+'
      a = tc += 1; b = tc += 1; c = tc += 1; d = tc += 1
      "%tmp#{a} = load i32* %i, align 4\n" <<
      "%tmp#{b} = getelementptr [1024 x i8]* %h, i32 0, i32 %tmp#{a}\n" <<
      "%tmp#{c} = load i8* %tmp#{b}, align 1\n" <<
      "%tmp#{d} = add i8 1, %tmp#{c}\n" <<
      "store i8 %tmp#{d}, i8* %tmp#{b}, align 1\n"

1. ポインタが指す位置を取得
2. その位置から、実際のデータの位置を取得
3. その位置から、実際のデータを取得
4. そのデータに1を足す
5. 実際のデータの一にそのデータを上書き格納させる

## 実演

    $ cat helloworld.bf
    $ cat helloworld.bf | ruby bfc.rb --llvm > helloworld.ll
    $ llvm-as helloworld.ll > helloworld.bc
    $ opt -O3 helloworld.bc # 引数指定なしで自分自身を書き換える
    $ lli helloworld.bc
    Hello, world!

もしくは単に

    $ ruby bfc.rb --llvm helloworld.bf --run

## おわり

参考文献:

* BFC: Brainf\*\*k Compilers
    * <http://ujihisa.blogspot.com/2009/12/bfc-brainfk-compilers.html>
* LLVM For Starters
    * <http://ujihisa.blogspot.com/2009/12/llvm-for-starters.html>
* Let's Try LLVM
    * <http://ujihisa.blogspot.com/2009/12/let-try-llvm.html>

## Q&A

* JVMとの違いは?
    * クラスとかGCとか
    * LLVMの方がLow Level

## TOP NEWS

* GHCがLLVMに!
  <http://www.haskell.org/pipermail/cvs-ghc/2010-February/052606.html>
