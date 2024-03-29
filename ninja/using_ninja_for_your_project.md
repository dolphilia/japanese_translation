# プロジェクトにNinjaを使用する

Ninjaは現在、Unix系システムおよびWindowsで動作します。Linuxでの動作確認が最も多かったが、Mac OS XやFreeBSDでも問題なく動作する。

プロジェクトが小規模な場合、Ninjaの速度への影響はあまり感じられないと思います。(ただし、小規模なプロジェクトであっても、Ninjaの限られたシンタックスによって、ビルドルールを簡略化し、結果としてビルドを高速化することができる場合があります。)つまり、編集とコンパイルのサイクルタイムに満足しているプロジェクトでは、Ninjaは役に立たないと言うことです。

Ninjaよりも使いやすく、機能的なビルドシステムは他にもたくさんあります。おすすめポイント：Ninjaの作者は、Ninjaのデザインに影響を与えた[tupのビルドシステム](http://gittup.org/tup/)を見つけ、[redo](https://github.com/apenwarr/redo)のデザインは非常に巧妙だと考えています。

Ninjaの利点は、よりスマートなメタビルドシステムと組み合わせて使用することです。

[gn](https://gn.googlesource.com/gn/)

Google Chromeや関連プロジェクト（v8、node.js）、Google Fuchsiaのビルドファイルを生成するためのメタビルドシステム。 gnはChromeがサポートするすべてのプラットフォーム用のNinjaファイルを生成することができる。

[CMake](https://cmake.org/)

広く使われているメタビルドシステムで、CMakeバージョン2.8.8現在、Linux上でNinjaファイルを生成することができる。CMakeの新しいバージョンでは、WindowsとMac OS XでもNinjaファイルを生成できるようになりました。

[others](https://github.com/ninja-build/ninja/wiki/List-of-generators-producing-ninja-build-files)

Ninjaは、premakeのような他のメタビルドソフトウェアに完璧にフィットするはずです。もし、この作業を行った場合は、是非ともご一報ください。


## Ninjaを実行する

ninjaを実行します。デフォルトでは、カレントディレクトリにあるbuild.ninjaというファイルを探し、古くなったターゲットをすべてビルドします。コマンドライン引数で、ビルドするターゲット（ファイル）を指定することができます。

また、コマンドラインに入力したソースを含むルールの最初の出力として、ターゲットを指定するための特別な構文 target^ があります (存在する場合)。たとえば、target を foo.c^ と指定すると、foo.o がビルドされます (ビルドファイルにこれらのターゲットがあると仮定しています)。

ninja -h はヘルプを表示します。Ninjaのフラグの多くはMakeのフラグと意図的に一致しています。例えば、ninja -C build -j 20はbuildディレクトリに移動し、buildコマンドを20個並列に実行します。(なお、Ninjaはデフォルトでコマンドを並列に実行するので、通常-jを渡す必要はありません)。


## 環境変数

Ninjaは、その動作を制御するために1つの環境変数をサポートしています。NINJA_STATUS は、ルールが実行される前に表示される進捗状況である。

いくつかのプレースホルダーが利用可能です。

- %s: 開始したエッジの数。
- %t: ビルドを完了するために実行しなければならないエッジの総数です。
- %p: 開始されたエッジの割合です。
- %r: 現在実行中のエッジの数。
- %u: 開始する残りのエッジの数。
- %f: 完成したエッジの数。
- %o: 1秒間に仕上げるエッジの総合的なレート
- %c: 1 秒あたりのエッジの完了率 (-j またはそのデフォルト値で指定されたビルドの平均)
- %e: 経過時間(秒)。(Ninja1.2以降で利用可能)。
- %%: プレーンな%文字。

デフォルトの進行状況は、"\[%f/%t\]" です (ビルドルールから分離するための末尾のスペースに注意)。他の進行状況の例としては、"\[%u/%r/%f\]" が考えられます。


## 追加ツール

Ninjaのコマンドラインにある-tフラグは、Ninjaの開発中に便利だと思われるツールを実行します。現在のツールは


### query

指定されたターゲットの入出力をダンプします。

### browse

ウェブブラウザで依存関係グラフをブラウズします。ファイルをクリックすると、そのファイルにフォーカスが当たり、入力と出力が表示されます。この機能を使用するには、Pythonのインストールが必要です。デフォルトでは、ポート8000が使用され、Webブラウザが開かれます。これは以下のように変更することができます。

```sh
ninja -t browse --port=8000 --no-browser mytarget
```

### graph

グラフの自動レイアウトツールであるgraphvizが使用する構文でファイルを出力します。という感じで使ってください。

```sh
ninja -t graph mytarget | dot -Tpng -ograph.png
```

Ninjaのソースツリーにおいて、ninja graph.pngはNinja自身のイメージを生成する。ターゲットが与えられていない場合は、すべてのルートターゲットのグラフを生成する。

### targets

ルール別または深さ別のターゲットのリストを出力する。ninja -t targets rule name のように使用すると、与えられたルールを使って構築されたターゲットのリストが表示されます。ルールが与えられない場合、ソースファイル (グラフの葉) が表示される。ninja -t targets depth digit のように使用すると、ルートターゲット (出力のないもの) から始まる深さ優先の方法でターゲットのリストを表示します。インデントは依存関係をマークするために使われます。深さが0であれば、すべてのターゲットを表示する。引数が提供されない場合、ninja -t targets depth 1が仮定されます。このモードではターゲットは何度もリストアップされるかもしれません。このように使用すると、 ninja -t targets all はインデントなしで利用可能なすべてのターゲットを表示し、 depth モードよりも高速になります。

### commands

ターゲットのリストが与えられたとき、順番に実行すれば、すべての出力ファイルが古いと仮定して、これらのターゲットを再構築するために使用することができるコマンドのリストを表示します。

### inputs

ターゲットのリストが与えられたとき、それらのターゲットを再構築するために使用されたすべての入力のリストを表示する。Ninja 1.11以降で利用可能。

### clean

ビルドされたファイルを削除します。デフォルトでは、ジェネレーターによって作成されたものを除く、すべてのビルドされたファイルを削除します。gフラグを追加すると、ジェネレーターによって作成されたビルドファイルも削除されます ([ジェネレーター属性のルールリファレンス](https://ninja-build.org/manual.html#ref_rule)を参照してください)。追加の引数はtargetsで、与えられたターゲットとそのターゲット用にビルドされたすべてのファイルを再帰的に削除します。

ninja -t clean -r rules のように使用すると、与えられたルールでビルドされたすべてのファイルを削除します。

作成されたものの、グラフで参照されていないファイルは削除されません。このツールは-vと-nオプションを考慮します(-nは-vを意味することに注意してください)。

### cleandead

以前のビルドによって生成された、ビルドファイルに含まれなくなったファイルを削除する。Ninja 1.10以降で利用可能。

### compdb

ソースファイル名を第一入力とするC言語のコンパイラルールのリストを与え、 Clangツールインタフェースが期待する[JSON形式](https://clang.llvm.org/docs/JSONCompilationDatabase.html)のコンパイルデータベースを 標準出力に表示させる。Ninja 1.2以降で利用可能。

### deps

.ninja_deps ファイルに格納されているすべての依存関係を表示します。ターゲットを指定すると、そのターゲットの依存関係のみを表示する。Ninja 1.4 以降で利用可能。

### missingdeps

ターゲットのリストが与えられたとき、生成されたファイルに依存しているが、ジェネレータに適切な（おそらく推移的な）依存性を持っていないターゲットを探します。そのようなターゲットは、クリーンビルドでビルドの脆弱性を引き起こすかもしれません。

壊れたターゲットは、depsログ/depfile依存情報が正しいことを仮定して見つけることができます。生成されたファイル（ジェネレータターゲットの出力）に暗黙的に依存するが、ジェネレータターゲットへの明示的または順序的な依存パスを持っていないターゲットは、壊れていると見なされます。

このツールの発見は、他のターゲットをビルドすることなく、クリーンなアウトディレクトリでリストされたターゲットをビルドしようとすることによって検証することができます。ビルドは、生成されたファイルを指し示すインクルードエラーか同等のエラーが出て、それぞれ失敗するはずである。Ninja 1.11以降で利用可能。

### recompact

.ninja_deps ファイルを再コンパクトする。Ninja 1.4 以降で利用可能。

### restat

.ninja_log ファイルに記録された全てのファイル変更タイムスタンプを更新する。Ninja 1.10 以降で利用可能。

### rules

全ルールの一覧を出力する。ninja -t targets ルール名やninja -t compdbにどのルール名を渡せばよいかを知るために利用できる。dフラグを追加すると、ルールの説明も出力される。

### msvc

Windowsホストでのみ利用可能です。のように、あらかじめ設定された環境変数のセットでcl.exeコンパイラーを起動するためのヘルパーツール。

```sh
ninja -t msvc -e ENVFILE -- cl.exe <arguments>
```

ここで、ENVFILEは、WindowsのCreateProcessA()に適した環境ブロック(すなわち、NAME=VALUEのように見えるゼロ終端の文字列とそれに続く余分なゼロ終端を持つ一連の文字列)を含むバイナリファイルである。これは、ローカルのコードページ・エンコードを使用していることに注意してください。

また、このツールは、/showIncludesフラグが使用されているときに、コンパイラの出力を解析し、そこからGCC互換のdepfileを生成する非推奨の方法をサポートしています。

```
+ --- ninja -t msvc -o DEPFILE [-p STRING] — cl.exe /showIncludes <arguments> ---

+
```

このオプションを使用する場合、-p STRINGを使用して、cl.exeが依存情報を出力するために使用するローカライズされた行頭語を渡すことができます。英語圏の場合、これは二重引用符なしで "Note: including file:「という二重引用符のないものになりますが、他の地域では異なります。

Ninjaは現在、Ninjaファイル中のdeps = msvcとmsvc_deps_prefixを使用することで、これをネイティブにサポートしていることに注意してください。ネイティブサポートは、コンパイラが呼ばれるたびに余分なツールプロセスを起動することを避け、Windowsでのビルドを著しく高速化することができる。

### wincodepage

Windowsホストで利用可能(Ninja 1.11以降)。ビルドファイルのエンコーディングに対応するWindowsコードページを表示する。出力は次のような形式である。

```
Build file encoding: <codepage>
```

今後のNinjaのバージョンアップでセリフが追加される可能性があります。

<codepage>のいずれかである。

- UTF-8: UTF-8でエンコードする。
- ANSI: システム全体のANSIコードページにエンコードする。