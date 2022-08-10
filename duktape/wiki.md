# ((o)) Duktape Wiki

Duktapeの公式Wikiへようこそ!


## ドキュメンテーション

- http://duktape.org/guide.html - 過去のバージョン: 1.5 1.4 1.3 1.2 1.1 1.0
- http://duktape.org/api.html - 過去のバージョン: 1.5 1.4 1.3 1.2 1.1 1.0


## はじめに

- はじめに：ライン処理
- はじめに：プライマリーテスト


## How-To

- 致命的なエラーの処理方法
- 値スタック型の扱い方
- 関数呼び出しの方法
- 仮想プロパティの使い方
- ファイナライゼーションの使い方
- バッファの扱い方 (Duktape 1.x、Duktape 2.x)
- lightfuncsの使い方
- モジュールの使い方
- コルーチンの使い方
- ロギングの使い方
- ネイティブ・コードでオブジェクト参照を持続させる方法
- ネイティブのコンストラクタ関数の書き方
- 配列の反復処理
- エラー・オブジェクトを拡張する
- Duktapeバイトコードのデコード方法
- 非BMP文字を扱うには
- グローバル・オブジェクトへの参照を取得する方法
- ベアメタルプラットフォームで動作させる方法
- デバッグ・プリントを有効にする方法
- Duktape用のエディターを設定する方法


## よくある質問

- Duktapeを開発するための設定
- トラブルシューティングの基本
- 内部プロトタイプと外部プロトタイプ
- API Cタイプ
- ES5以降の機能


## コンフィグと機能オプション

- ビルドのためのDuktapeの設定
- duk_config.h で使用される設定オプション (DUK_USE_xxx)
- コンパイラのコマンドラインオプションとして使用される機能オプション (DUK_OPT_XXX) (Duktape 1.3 まで), https://github.com/svaarala/duktape/tree/master/config/feature-options を参照。


## 移植性と互換性

- 様々なコンパイラとターゲットに対する移植性の注意、コンパイルとトラブルシューティングのヒント
- プラットフォーム
- アーキテクチャー
- コンパイラ
- 標準ライブラリ: musl, uclibc
- TypeScriptとの互換性


## パフォーマンス

- http://duktape.org/benchmarks.html
- パフォーマンスを最適化する方法
- Duktape 1.3.0のパフォーマンス測定
- Duktape 1.4.0のパフォーマンス測定
- Duktape 1.5.0パフォーマンス測定
- Duktape 2.0.0パフォーマンス測定
- Duktape 2.1.0パフォーマンス測定
- Duktape 2.2.0パフォーマンス測定
- Duktape 2.3.0性能測定
- Duktape 2.4.0パフォーマンス測定
- Duktape 2.5.0パフォーマンス測定


## ロー・メモリの最適化

- ロー・メモリー環境: low-memory.rst
- ローメモリ設定オプションの提案: low-memory.yaml
- ハイブリッドプールアロケータの例: alloc-hybrid


## その他

- Duktapeを使ったプロジェクト
- デバッグ・クライアント


## 寄稿、著作権、ライセンス

- https://github.com/svaarala/duktape-wiki



## はじめに：ライン処理

### 概要

簡単なサンプルプログラムを見てみましょう。このプログラムはCのメインループを使って標準入力から行を読み込み、ECMAScriptヘルパーを呼び出して行を変換し、その結果をプリントアウトします。行処理機能は、正規表現のようなECMAScriptの良いところを利用することができ、Cプログラムを再コンパイルすることなく簡単に変更することができる。

スクリプトのコードはprocess.jsに配置される。行処理関数の例では、プレーンテキストの行をHTMLに変換し、星の間のテキストを自動的に太字にする。

- https://github.com/svaarala/duktape/blob/master/examples/guide/process.js

C言語のコードprocesslines.cは、Duktapeコンテキストを初期化し、スクリプトを評価した後、標準入力から行を処理し、各行に対してprocessLine()を呼び出します。

- https://github.com/svaarala/duktape/blob/master/examples/guide/processlines.c

### processlines.cの内訳

このサンプルコードのDuktape固有の部分を一つ一つ見ていきましょう。ここでは簡潔にするため、いくつかの詳細について説明します。詳細については、プログラミング・モデルを参照してください。

```c
/* For brevity assumes a maximum file length of 16kB. */
static void push_file_as_string(duk_context *ctx, const char *filename) {
    FILE *f;
    size_t len;
    char buf[16384];

    f = fopen(filename, "rb");
    if (f) {
        len = fread((void *) buf, 1, sizeof(buf), f);
        fclose(f);
        duk_push_lstring(ctx, (const char *) buf, (duk_size_t) len);
    } else {
        duk_push_undefined(ctx);
    }
}
```


Duktapeは埋め込み可能なエンジンであり、最小限の仮定を行うため、デフォルトのCまたはECMAScript APIにはファイルI/Oバインディングがありません。上記のヘルパーは、ファイルの内容を文字列としてプッシュする方法の例です。この例では簡潔にするために固定の読み込みバッファを使っていますが、より良い実装ではまずファイルのサイズをチェックして、そのバッファを確保することになるでしょう。Duktapeの配布物には「extras」が含まれており、ファイルI/Oヘルパーを含む、便利なCやECMAScriptヘルパーが提供されています。

```c
ctx = duk_create_heap_default();
if (!ctx) {
    printf("Failed to create a Duktape heap.\n");
    exit(1);
}
```


まず、Duktapeコンテキストを作成します。コンテキストは、値をスタックにプッシュしたりポッピングしたりすることで、ECMAScriptコードと値を交換できるようにします。Duktape APIのほとんどの呼び出しは、値スタックを操作し、スタック上の値をプッシュ、ポッピング、検査します。実運用コードでは、致命的なエラー・ハンドラを設定できるように duk_create_heap() を使用する必要があります。エラー処理のベストプラクティスについては、 エラー処理 を参照してください。

```c
push_file_as_string(ctx, "process.js");
if (duk_peval(ctx) != 0) {
    printf("Error: %s\n", duk_safe_to_string(ctx, -1));
    goto finished;
}

duk_pop(ctx);  /* ignore result */
```


まず、ファイルヘルパーを使って、process.js を文字列としてバリュースタックにプッシュします。それから duk_peval() を使ってスクリプトをコンパイルし、実行します。このスクリプトは、後で使用するために processLine() を ECMAScript グローバルオブジェクトに登録します。保護された呼び出しである duk_peval() はスクリプトの実行に使用され、構文エラーのようなスクリプトのエラーは致命的なエラーを引き起こすことなく捕捉され処理されるようにします。エラーが発生した場合、エラーメッセージは duk_safe_to_string() を使って安全に強制され、さらなるエラーを発生させないことが保証されます。文字列強制の結果は、読み取り専用で NUL 終端の UTF-8 エンコード文字列を指す const char * で、printf() で直接使用することができます。この文字列は、対応する文字列値が値スタック上にある限り、有効である。文字列は、値が値スタックからポップ・オフされると、自動的に解放される。

```c
duk_push_global_object(ctx);
duk_get_prop_string(ctx, -1 /*index*/, "processLine");
```

最初の呼び出しは、ECMAScript のグローバルオブジェクトを値スタックにプッシュします。2 番目の呼び出しは、グローバルオブジェクトの processLine プロパティを検索します (process.js 内のスクリプトで定義されています)。負の値はスタック要素を上から順に参照するので、-1 はスタックの最上位要素であるグローバルオブジェクトを参照します。

```c
duk_push_string(ctx, line);
```


line が指す文字列を値スタックにプッシュする。文字列の長さは、 strlen() のように NUL ターミネータをスキャンすることで自動的に決定されます。Duktape は、文字列がスタックにプッシュされたときにそのコピーを作成するので、呼び出しが返されたときにラインバッファを自由に変更することができる。

```c
if (duk_pcall(ctx, 1 /*nargs*/) != 0) {
    printf("Error: %s\n", duk_safe_to_string(ctx, -1));
} else {
    printf("%s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);  /* pop result/error */
```


このとき、値スタックに含まれる（スタックは右側に成長する）。

```
[ globalObject processLine line ]
```


duk_pcall() メソッドは、バリュースタック上に指定された数の引数で関数を呼び出し、関数と引数の値の両方を関数の戻り値に置き換えます。ここでは、nargs の数が 1 なので、processLine 関数と行が戻り値に置き換えられ、結果的にバリュースタックは次のようになります。

```
[ globalObject callResult ]
```


この呼び出しは、エラーを捕捉してプリントできるように保護されている。duk_safe_to_string() API 呼び出しは、エラーを安全に印刷するために再び使用されます。一度印刷されると、結果（またはエラー）は値スタックからポップオフされ、グローバルオブジェクトはまだスタックに残ります。

```c
duk_destroy_heap(ctx);
```


最後に、Duktapeコンテキストは破棄され、コンテキストが保持するすべてのリソースが解放されます。この呼び出しによって、値スタックと値スタック上の全ての参照が解放されます。この例では、グローバル・オブジェクトを意図的に値スタックに残しています。これは問題ではありません。ヒープが破壊されたときに値スタックが空でなくても、メモリリークは発生しません。

### コンパイル

単純に次のようにコンパイルします。

```sh
# src/ contains Duktape sources from the distributable or prepared
# explicitly using tools/configure.py.

$ gcc -std=c99 -o processlines -Isrc/ src/duktape.c processlines.c -lm
```


テスト実行、process.jsがカレントディレクトリにあることを確認する。

```sh
$ echo "I like *Sam & Max*." | ./processlines
I like <b>Sam & Max</b>.
```


## はじめに：プライマリーテスト

入門：行処理の例では、ECMAScriptでは簡単だがCでは難しいことを、CコードがECMAScriptに呼び出すことができることを説明しました。

この記事の例はその逆で、ECMAScript のコードが C のコードを呼び出す方法を説明します：スクリプトは多くのことに有用ですが、低レベルのバイトや文字処理には最適ではありません。最適化された C ヘルパーを呼び出すことができれば、スクリプト・ロジックの大部分を美しい ECMAScript で書き、パフォーマンスが重要な部分については C を呼び出すことができます。ネイティブ関数を使用するもう一つの理由は、ネイティブライブラリへのアクセスを提供することです。

ネイティブ関数を実装するには、Duktape/Cバインディングという特別な呼び出し方法に従った普通のC関数を書きます。Duktape/C関数は1つの引数（Duktapeコンテキスト）を取り、エラーまたは戻り値の数を示す1つの値を返します。関数は、Duktape APIで操作されたDuktapeコンテキストの値スタックを介して、呼び出しの引数にアクセスし、戻り値を提供します。Duktape/CバインディングとDuktape APIについては、後ほど詳しく説明します。

### 簡単な例：数値を二乗する

簡単な例を挙げます。

```c
duk_ret_t my_native_func(duk_context *ctx) {
    double arg = duk_require_number(ctx, 0 /*index*/);
    duk_push_number(ctx, arg * arg);
    return 1;
}
```


これを一行ずつ見ていきましょう。

```c
double arg = duk_require_number(ctx, 0 /*index*/);
```


値スタックインデックス0（スタックの底、関数呼び出しの第1引数）の数値が数値であるかどうかをチェックし、そうでない場合はエラーを投げて決して戻りません。値が数値の場合、doubleとして返す。

```c
duk_push_number(ctx, arg * arg);
```

引数の2乗を計算し、値スタックにプッシュする。

```c
return 1;
```


関数呼び出しから戻り、値スタックの最上位に（単一の）戻り値があることを示す。複数の戻り値はまだサポートされていません。0を返して戻り値がないことを示すこともできますが、その場合DuktapeのデフォルトはECMAScriptのundefinedになります。負の戻り値は、自動的にエラーをスローします。これは、エラーをスローするための便利な省略記法です。Duktape は、関数が戻った時に自動的にその処理を行います。詳しくは、プログラミング・モデルを参照してください。

### プライマリーテスト

ECMAScriptのアルゴリズムを高速化するためにネイティブコードを使用する例として、プリマリティテストを使用することにします。具体的には、私たちのテストプログラムは1000000以下のプリムで、数字'9999'で終わるものを探します。このプログラムのECMAScriptバージョンは次の通りです。

- https://github.com/svaarala/duktape/blob/master/examples/guide/prime.js

このプログラムはネイティブヘルパーがあればそれを使い、なければECMAScriptバージョンにフォールバックすることに注意してください。これにより、ECMAScriptのコードは他の含むプログラムで使用することができます。また、プライムチェックのプログラムが、変更しないとネイティブ版がコンパイルできない別のプラットフォームに移植された場合、ヘルパーが移植されるまで、プログラムは（速度は遅いですが）機能し続けます。この場合、ネイティブヘルパーの検出はスクリプトがロードされたときに行われます。また、実際にコードが呼び出されたときに検出することもでき、より柔軟な対応が可能です。

primeCheckECMAScript() と同等の機能を持つネイティブヘルパーは、 非常に簡単に実装することができます。プログラムmainを追加し、ECMAScriptグローバルオブジェクトに単純なprint()バインディングを追加すると、primecheck.cが得られます。

- https://github.com/svaarala/duktape/blob/master/examples/guide/primecheck.c

Getting started: line processingと比較した新しいコールは、一行一行です。

```c
int val = duk_require_int(ctx, 0);
int lim = duk_require_int(ctx, 1);
```

これらの 2 つのコールは、ネイティブヘルパーに与えられた 2 つの引数値をチェックします。もし値が ECMAScript の数値型でない場合はエラーがスローされます。数値であれば、その値は整数に変換され、val と lim ロケールに代入されます。インデックス 0 は最初の関数引数を、インデックス 1 は 2 番目の関数引数を指します。

技術的には、duk_require_int() は duk_int_t を返します。この間接型は、int が 16 ビット幅しかない稀なプラットフォームを除いて、常に int にマップされます。通常のアプリケーションコードでは、このことを気にする必要はありません。

```c
duk_push_false(ctx);
return 1;
```

ECMAScript の false を値スタックにプッシュします。C の戻り値 1 は、ECMAScript の呼び出し元に false 値が返されることを示す。

```c
duk_push_global_object(ctx);
duk_push_c_function(ctx, native_prime_check, 2 /*nargs*/);
duk_put_prop_string(ctx, -2, "primeCheckNative");
```


最初の呼び出しは、前と同様に、ECMAScript グローバルオブジェクトを値スタックにプッシュします。2 番目の呼び出しは ECMAScript Function オブジェクトを作成し、それを値スタックにプッシュします。この Function オブジェクトは Duktape/C の関数 native_prime_check() に束縛されています：ここで作成された ECMAScript 関数が ECMAScript から呼び出されると、C 関数が呼び出されます。第2呼び出し引数(2)は、C関数が値スタック上にいくつの引数を取得するかを示す。呼び出し側が与える引数が少なければ、不足する引数は undefined で埋められ、呼び出し側が与える引数が多ければ、余分な引数は自動的に削除される。最後に、3回目の呼び出しで、関数オブジェクトを primeCheckNative という名前でグローバルオブジェクトに登録し、関数値をスタックからポップアウトします。

```c
duk_get_prop_string(ctx, -1, "primeTest");
if (duk_pcall(ctx, 0) != 0) {
    printf("Error: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);  /* ignore result */
```


ここに来たとき、値スタックはすでにスタックの一番上にグローバルオブジェクトを含んでいます。1 行目は、グローバルオブジェクト（ロードされたスクリプトによって定義された）から primeTest 関数を検索しています。2-4 行目は、primeTest 関数を引数ゼロで呼び出し、エラーが発生した場合は安全にプリントアウトします。5 行目では、呼び出しの結果をスタックから取り出しています。

### コンパイルと実行

前回と同様にコンパイルします。

```sh
# src/ contains Duktape sources from the distributable or prepared
# explicitly using tools/configure.py.

$ gcc -std=c99 -o primecheck -Isrc/ src/duktape.c primecheck.c -lm
```


テスト実行、prime.jsがカレントディレクトリにあることを確認します。

```sh
$ time ./primecheck
Have native helper: true
49999 59999 79999 139999 179999 199999 239999 289999 329999 379999 389999
409999 419999 529999 599999 619999 659999 679999 769999 799999 839999 989999

real    0m2.985s
user    0m2.976s
sys 0m0.000s
```


実行時間のほとんどはプライムチェックに費やされるため、プレーンなECMAScriptと比較して大幅なスピードアップを実現しています。prime.jsを編集して、ネイティブヘルパーの使用を無効にすることで確認できます。

```javascript
// Select available helper at load time
var primeCheckHelper = primeCheckECMAScript;
```


再コンパイルして、テストを再実行する。

```sh
$ time ./primecheck
Have native helper: false
49999 59999 79999 139999 179999 199999 239999 289999 329999 379999 389999
409999 419999 529999 599999 619999 659999 679999 769999 799999 839999 989999

real    0m23.609s
user    0m23.573s
sys 0m0.000s
```

