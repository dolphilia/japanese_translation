[原文](https://duktape.org/api.html)

# Duktape API 日本語訳


## はじめに

バージョン: 2.6.0 (2020-10-13)


### ドキュメントスコープ

Duktape API（duktape.hで定義）は定数とAPIコールのセットで、C/C++プログラムがECMAScriptコードとインターフェースできるようにし、値の表現などの内部詳細から保護するものです。

この文書では、Duktape APIとそのコア・コンセプトに関する簡潔なリファレンスを提供します。Duktapeに初めて触れる方は、まず「Duktapeプログラマーズ・ガイド」をお読みください。


### ブラウザ検索を使った関数の検索

ブラウザ検索（通常はCTRL-F）を使って、検索語の前にドットを付けることで関数定義を検索することができます。例えば、duk_example_func()を探すには、".duk_example_func "を使用します。ほとんどのブラウザでは、関数を定義している実際のセクションのみが検索されるはずです。

### API の安全性

一般的なルールとして、APIコールはすべてのパラメータをチェックし、安全でない動作（クラッシュ）なしにNULL引数や無効な値のスタック・インデックスを許容しています。

一つの大きな例外は、Duktape コンテキストの初期引数である ctx です。特に断りのない限り、これはチェックされず、NULLでないことが要求され、さもなければ安全でない動作が発生する可能性があります。これは、この引数に対する明示的なチェックは、コードのフットプリントを増加させ、実用的な利益をほとんどもたらさないからです。

また、エラー処理のベストプラクティス、特に診断が困難な致命的なエラーにつながるキャッチされないエラーを回避するためにを参照してください。

### APIコールはマクロであってもよい

Duktapeの全てのAPIコールはマクロである可能性があります。あるAPIコールの実装は、互換性のあるリリース間であっても、マクロと実際の関数の間で変更される可能性があります。詳しくは Duktape API を参照してください。

### 最小限のDuktapeプログラム

```c
#include "duktape.h"

int main(int argc, const char *argv[]) {
    duk_context *ctx = duk_create_heap_default();
    if (ctx) {
        duk_eval_string(ctx, "print('Hello world from Javascript!');");
        duk_destroy_heap(ctx);
    }
    return 0;
}
```


## 表記法

### バリュースタック

この文書では、以下のスタック表記を使用します。スタックは、右に行くほど大きくなるように視覚的に表現されます。

例：

```c
duk_push_number(ctx, 123);
duk_push_string(ctx, "foo");
duk_push_true(ctx);
```


スタックは次のようになります。

```
| 123 | "foo" | true |
```


スタック上の要素のうち、操作に影響を与えないものは、1つの要素に省略記号（"..."）を付けて表記しています。

```
| ... |
```


読み取り、書き込み、挿入、削除などアクティブに操作される要素は、背景が白になります。

```
| ... | <obj> | <key> | 
```


APIコールの明示的なインデックスで識別される要素は、スタックのどこにあってもよいことを強調するために、周囲を省略記号で囲んで表示されます。

```
| ... | <obj> | ... | <key> | <value> | 
```


場合によっては、要素のインデックスを括弧内の数値またはシンボル値で強調することがあります。

```
| ... | <val(index)> | <val(index+1)> | ... |
```


スタック変換は矢印と2つのスタックで表現します。

```
| ... | <obj> | ... | <key> | <value> |  ->  | ... | <obj> | ... |
```


## 概念

### ヒープ

ガベージコレクションのための単一の領域です。1つ以上のコンテキストで共有されます。


### コンテキスト

Duktape APIで使用されるハンドルで、Duktapeスレッド（コルーチン）およびそのコールスタックとバリュースタックに関連付けられます。


### コールスタック

コンテキストのアクティブな関数呼び出しチェーンのブックキーピングです。ECMAScriptと Duktape/Cの両方の関数呼び出しが含まれます。

### バリュースタック

コンテキストのコールスタックで、現在のアクティブ化に属するタグ付けされた値のためのストレージです。

バリュースタックに保持される値は、伝統的なタグ付き型です。スタック・エントリーは、最も最近の関数呼び出しの下 (>= 0) または上 (< 0) からインデックスが付けられます。

### バリュースタックのインデックス

非負 (>= 0) のインデックスは、現在のスタックフレームのスタックエントリを、フレームの底を基準にして指します。

```
| 0 | 1 | 2 | 3 | 4 | <5> | 
```


負（< 0）のインデックスは、フレームトップからの相対的なスタックエントリを指します。

```
| -6 | -5 | -4 | -3 | -2 | <-1> | 
```


特殊定数 DUK_INVALID_INDEX は、無効なスタックインデックスを示す負の整数ですこれは API 呼び出しから返されることがあり、また「値がない」ことを示すためにいくつかの API 呼び出しに与えられることがあります。

バリュースタックトップ(または単に「トップ」)は、最高使用インデックスのすぐ上の虚数要素の非負のインデックスです。例えば、最も使用されているインデックスの上は 5 であるので、スタックトップは 6 です。 トップは現在のスタックサイズを示し、スタックにプッシュされる次の要素のインデックスでもあります。

```
| 0 | 1 | 2 | 3 | 4 | <5> | (6) |
```


> API のスタック操作は、常に現在のスタックフレームに限定されます。現在のフレームより下のスタックエントリを参照する方法はない。これは、コールスタック内の関数が互いの値に影響を与えないようにするためで、意図的なものです。


### スタックタイプ

| Type      | Type constant      | Type mask constant      | Description | Heap alloc |
| --------- | ------------------ | ----------------------- | ---- | ---- |
| (none)    | DUK_TYPE_NONE      | DUK_TYPE_MASK_NONE      | no type (missing value, invalid index, etc) | no |
| undefined	| DUK_TYPE_UNDEFINED | DUK_TYPE_MASK_UNDEFINED | undefined | no |
| null      | DUK_TYPE_NULL	     | DUK_TYPE_MASK_NULL      | null | no |
| boolean   | DUK_TYPE_BOOLEAN   | DUK_TYPE_MASK_BOOLEAN   | true and false | no |
| number    | DUK_TYPE_NUMBER    | DUK_TYPE_MASK_NUMBER    | IEEE double | no |
| string    | DUK_TYPE_STRING    | DUK_TYPE_MASK_STRING    | immutable string | yes |
| object    | DUK_TYPE_OBJECT    | DUK_TYPE_MASK_OBJECT    | object with properties | yes |
| buffer    | DUK_TYPE_BUFFER    | DUK_TYPE_MASK_BUFFER    | mutable byte buffer, fixed/dynamic | yes |
| pointer   | DUK_TYPE_POINTER   | DUK_TYPE_MASK_POINTER   | opaque pointer (void *) | no |
| lightfunc | DUK_TYPE_LIGHTFUNC | DUK_TYPE_MASK_LIGHTFUNC | plain Duktape/C pointer (non-object) | no |


Heap alloc カラムは、タグ付けされた値がヒープで割り当てられたオブジェクトを指しているかどうかを示し、（オブジェクトのプロパティテーブルのように）追加で割り当てられる可能性があることを示す。

### スタックタイプマスク

各スタックタイプにはビットインデックスがあり、スタックタイプのセットをビットマスクとして表現することができます。呼び出し側のコードはこのようなビットセットを使って、例えばあるスタック値があるタイプのセットに属するかどうかをチェックすることができます。

例：

```c
if (duk_get_type_mask(ctx, -3) & (DUK_TYPE_MASK_NUMBER |
                                  DUK_TYPE_MASK_STRING |
                                  DUK_TYPE_MASK_OBJECT)) {
    printf("type is number, string, or object\n");
}
```

さらに便利にタイプのセットをマッチングさせるための特定のAPIコールがあります。

```c
if (duk_check_type_mask(ctx, -3, DUK_TYPE_MASK_NUMBER |
                                 DUK_TYPE_MASK_STRING |
                                 DUK_TYPE_MASK_OBJECT)) {
    printf("type is number, string, or object\n");
}
```


### 配列のインデックス

ECMAScript のオブジェクトと配列のキーは文字列のみです。配列のインデックス (0, 1, 2, ...) も文字列で表現され、これは [0, 2**32-2] 範囲のすべての整数に対するそれぞれの数値の標準文字列表現 ("0", "1", "2", ...) として表現されます。概念的には、プロパティ・ルックアップで使われるキーは、まず文字列に強制されます。つまり、obj[123]は実際にはobj["123"]の略語に過ぎないのです。

Duktapeは、可能な限り、配列へのアクセスにおいて明示的な数値から文字列への変換を避けようとします。そのため、ECMAScriptコードとDuktape APIを使用するCコードの両方で、数値配列インデックスを使用することが望ましいと言えます。文字列と数値のインデックスを受け付けるAPIコールのバリエーションが一般的に存在します。

### Duktape/C関数

Duktape/C API 署名のある C 関数は、ECMAScript 関数オブジェクトまたは lightfunc 値と関連付けることができ、関連付けられた値が ECMAScript コードから呼び出されたときに呼び出されます。

例：

```c
duk_ret_t my_func(duk_context *ctx) {
    duk_push_int(ctx, 123);
    return 1;
}
```

バリュースタックで与えられる引数の数は、対応する ECMAScript 関数オブジェクトが作成されるときに指定されます: 引数を "そのまま" 取得するための DUK_VARARGS の固定数の引数のどちらかです。

- 固定数の引数を使用する場合、余分な引数は削除され、不足する引数は必要に応じて undefined で埋められます。
- DUK_VARARGS を使用する場合、実際の引数の数を決定するために duk_get_top() を使用します。

関数の戻り値。

- 戻り値1：スタックトップが戻り値を含む。
- 戻り値0：戻り値は未定義です。
- DUK_ERR_xxx の否定形である DUK_RET_xxx 定数を使用します。
- 戻り値 >1: 予約済み、現在は無効。
 
エラーの省略記法を使用した場合、エラーメッセージは表示されません。

```c
duk_ret_t my_func(duk_context *ctx) {
    if (duk_get_top(ctx) == 0) {
        /* throw TypeError if no arguments given */
        return DUK_RET_TYPE_ERROR;
    }
    /* ... */
}
```


### スタッシュ

スタッシュとは、Duktape/C API を使用して C コードから到達可能で、ECMAScript コードから到達不可能なオブジェクトのことです。スタッシュは、C コードが ECMAScript コードから安全に分離された内部状態を保存することを可能にします。スタッシュは3つあります。

- ヒープ・スタッシュ：同じヒープ内のすべてのスレッドで共有されます。
- グローバルスタッシュ: グローバルオブジェクトを共有するすべてのスレッドで共有されます。
- スレッドキャッシュ: 特定のスレッドに固有のもの。


## ヘッダ定義

このセクションでは、duktape.h で一般的に必要とされるいくつかのヘッダ 定義を要約します。これは完全なものではなく、抜粋は読みやすさのために再 編成されています。特定の定義値には頼らず、定義名だけを頼りにしてください。疑問があれば、ヘッダを直接参照してください。

### Duktapeバージョン

Duktape のバージョンは、DUK_VERSION 定義によって利用可能です。

- DUK_VERSION: 数値（メジャー * 10000 + マイナー * 100 + パッチ）。

例えば、バージョン2.3.4は、値20304となります。同じ値が Duktape.version を通して ECMAScript コードで利用できます。プレリリースの場合、DUK_VERSION は実際のリリースより 1 つ少ない値です。例えば、2.4.0 プレリリースは 20399 となります。バージョニングを参照してください。

### Git情報

以下のGit識別子が利用可能です（すべてDuktapeのGitHubリポジトリを参照）。

- DUK_GIT_DESCRIBE: DuktapeビルドのGit記述文字列です。公式リリースの場合は、"v1.0.0 "などですが、スナップショット・ビルドの場合は、"v1.0.0-155-g5b7ef1f-dirty "などの有用なバージョン情報を提供します。
- DUK_GIT_COMMIT: 配布ファイルのビルド元となる正確なコミットハッシュ。
- DUK_GIT_BRANCH: 配布ファイルのビルド元となったブランチ。これは、開発ブランチからビルドされたプロトタイプを識別するのに便利です。

ECMAScript 環境では、これと同等の定義がありません。

### デバッグプロトコルバージョン

デバッグ・プロトコルのバージョン番号です。

- DUK_DEBUG_PROTOCOL_VERSION デバッグプロトコルのバージョン番号 (1つの整数)。

### 構造体・型定義

```c
typedef struct duk_hthread duk_context;

typedef duk_ret_t (*duk_c_function)(duk_context *ctx);
typedef void *(*duk_alloc_function) (void *udata, duk_size_t size);
typedef void *(*duk_realloc_function) (void *udata, void *ptr, duk_size_t size);
typedef void (*duk_free_function) (void *udata, void *ptr);
typedef void (*duk_fatal_function) (void *udata, const char *msg);
typedef void (*duk_decode_char_function) (void *udata, duk_codepoint_t codepoint);
typedef duk_codepoint_t (*duk_map_char_function) (void *udata, duk_codepoint_t codepoint);
typedef duk_ret_t (*duk_safe_call_function) (duk_context *ctx, void *udata);
typedef duk_size_t (*duk_debug_read_function) (void *udata, char *buffer, duk_size_t length);
typedef duk_size_t (*duk_debug_write_function) (void *udata, const char *buffer, duk_size_t length);
typedef duk_size_t (*duk_debug_peek_function) (void *udata);
typedef void (*duk_debug_read_flush_function) (void *udata);
typedef void (*duk_debug_write_flush_function) (void *udata);
typedef void (*duk_debug_detached_function) (duk_context *ctx, void *udata);

struct duk_memory_functions {
        duk_alloc_function alloc_func;
        duk_realloc_function realloc_func;
        duk_free_function free_func;
        void *udata;
};
typedef struct duk_memory_functions duk_memory_functions;

struct duk_function_list_entry {
	const char *key;
	duk_c_function value;
	duk_int_t nargs;
};
typedef struct duk_function_list_entry duk_function_list_entry;

struct duk_number_list_entry {
	const char *key;
	duk_double_t value;
};
typedef struct duk_number_list_entry duk_number_list_entry;
```


### エラーコード

duk_error() などで使用されるエラーコードです。

- duk_err_none:エラーなし。例えば duk_get_error_code() から。
- DUK_ERR_ERROR: エラー。
- DUK_ERR_EVAL_ERROR: EvalError。
- DUK_ERR_RANGE_ERROR:RangeError：レンジエラー
- DUK_ERR_REFERENCE_ERROR:リファレンス・エラー
- DUK_ERR_SYNTAX_ERROR: 構文エラー
- DUK_ERR_TYPE_ERROR: TypeError (タイプエラー)
- DUK_ERR_URI_ERROR: URIエラー


### Duktape/C関数のリターンコード

例えば return DUK_RET_TYPE_ERROR は duk_type_error() を呼び出すのと似ていますが、より短いものです。

- DUK_RET_ERROR: DUK_ERR_ERROR と一緒に投げます。のと似ています。
- DUK_RET_EVAL_ERROR: DUK_ERR_EVAL_ERROR での投擲と同様です。
- DUK_RET_RANGE_ERROR: DUK_ERR_RANGE_ERROR でのスローイングに似ている
- DUK_RET_REFERENCE_ERROR: DUK_ERR_REFERENCE_ERROR と一緒に投げます。のと似ている
- DUK_RET_SYNTAX_ERROR: DUK_ERR_SYNTAX_ERROR とスローするのと似ています。
- DUK_RET_TYPE_ERROR: DUK_ERR_TYPE_ERROR とスローするのと似ている
- DUK_RET_URI_ERROR: DUK_ERR_URI_ERROR とスローするのと似ている


### 保護された呼び出しのためのリターンコード

保護された呼び出し（例：duk_safe_call()、duk_pcall()）に対するリターンコード。

- duk_exec_success:呼び出しはエラーなしで終了
- duk_exec_error:呼び出しに失敗、エラーは捕捉された


### duk_compileのためのコンパイルフラグ

duk_compile() や duk_eval() などのためのコンパイルフラグです。

- DUK_COMPILE_EVAL: (プログラムではなく)評価コードをコンパイルする
- DUK_COMPILE_FUNCTION: (プログラムの代わりに)関数コードをコンパイルする
- DUK_COMPILE_STRICT:プログラム、eval、または関数に対して、厳密な（外側の）コンテキストを使用する


### duk_def_propのフラグについて

duk_def_prop() とその派生型のためのフラグです。

- DUK_DEFPROP_WRITABLE: 書き込み可能に設定する (DUK_DEFPROP_HAVE_WRITABLE が設定されている場合のみ有効)。
- DUK_DEFPROP_ENUMERABLE: 列挙可能に設定する（DUK_DEFPROP_HAVE_ENUMERABLEが設定されている場合に有効）。
- DUK_DEFPROP_CONFIGURABLE: 設定可能 (DUK_DEFPROP_HAVE_CONFIGURABLE が設定されている場合のみ有効)
- DUK_DEFPROP_HAVE_WRITABLE: 書き込み可能かどうかを設定/解除する
- DUK_DEFPROP_HAVE_ENUMERABLE: 列挙可能を設定または解除します。
- DUK_DEFPROP_HAVE_CONFIGURABLE: コンフィギュラブルを設定/解除します。
- DUK_DEFPROP_HAVE_VALUE: 値を設定します (バリュースタックで指定されます)。
- DUK_DEFPROP_HAVE_GETTER: ゲッターを設定します (バリュースタックに保存されます)
- DUK_DEFPROP_HAVE_SETTER: (バリュースタックにある)セッターを設定する
- DUK_DEFPROP_FORCE:可能であれば変更を強制。仮想プロパティなどではまだ失敗する可能性があります
- DUK_DEFPROP_SET_WRITABLE: (DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_WRITABLE)
- DUK_DEFPROP_CLEAR_WRITABLE: DUK_DEFPROP_HAVE_WRITABLE
- DUK_DEFPROP_SET_ENUMERABLE: (DUK_DEFPROP_HAVE_ENUMERABLE | DUK_DEFPROP_ENUMERABLE)
- DUK_DEFPROP_CLEAR_ENUMERABLE: DUK_DEFPROP_HAVE_ENUMERABLE
- DUK_DEFPROP_SET_CONFIGURABLE: (DUK_DEFPROP_HAVE_CONFIGURABLE | DUK_DEFPROP_CONFIGURABLE)
- DUK_DEFPROP_CLEAR_CONFIGURABLE: DUK_DEFPROP_HAVE_CONFIGURABLE

いくつかの便利なバリアントは省略され、 duk_def_prop() を参照。


### duk_enumの列挙型フラグ

duk_enum() の列挙フラグ。

- DUK_ENUM_INCLUDE_NONENUMERABLE: 列挙可能なプロパティに加え、列挙不可能なプロパティも列挙します。
- DUK_ENUM_INCLUDE_HIDDEN: 隠されたシンボルも列挙する (Duktape 1.x では内部プロパティと呼ばれる)
- DUK_ENUM_INCLUDE_SYMBOLS: シンボルを列挙します。Symbolのキーを列挙する (デフォルトは列挙しない)
- DUK_ENUM_EXCLUDE_STRINGS: 文字列のキーを列挙しない（デフォルトは列挙する）
- DUK_ENUM_OWN_PROPERTIES_ONLY: プロトタイプチェーンは歩かず、自身のプロパティのみをチェックする
- DUK_ENUM_ARRAY_INDICES_ONLY: 配列のインデックスのみを列挙する
- DUK_ENUM_SORT_ARRAY_INDICES: 配列インデックスのソート(継承された配列インデックスを含む全列挙結果に適用)
- DUK_ENUM_NO_PROXY_BEHAVIOR: プロキシ動作を起動せず、プロキシオブジェクト自体を列挙します。


### duk_gcのガーベッジコレクションフラグ

duk_gc() のフラグ。

- DUK_GC_COMPACT: ヒープオブジェクトをコンパクトに


### 強制力のヒント

強制力のヒント

- DUK_HINT_NONE: 強制入力が Date でない場合は、String を優先する（E5 8.12.8 セクション）。
- DUK_HINT_STRING: 文字列を優先する
- DUK_HINT_NUMBER: 数値を優先する


### シンボルリテラルマクロ

以下のマクロは、C リテラルとして内部 Symbol 表現を作成するために定義されています。すべての引数は文字列リテラルでなければならず、計算値であってはなりません。

- DUK_HIDDEN_SYMBOL(x): Duktape固有の隠しシンボルのCリテラル。
- DUK_GLOBAL_SYMBOL(x): グローバルシンボルのCリテラル。Symbol.for(x)と同等。
- DUK_LOCAL_SYMBOL(x,uniq): ローカルシンボルのCリテラル、Symbol(x)と同等、'uniq'で提供されるユニークパートはDuktape内部フォーマットと衝突してはならない、推奨はユニークパートの前に"!"をつけること。
- DUK_WELLKNOWN_SYMBOL(x): Symbol.iteratorのようなよく知られたシンボルを表すCリテラル
- DUK_INTERNAL_SYMBOL(x): Duktape内部シンボルのCリテラル。アプリケーションは通常このマクロを全く使うべきでなく、Duktape内部シンボルのみに予約されています（バージョン保証なし）。


### その他の定義

- DUK_INVALID_INDEX: スタックインデックスが無効、なし、またはn/aです。
- DUK_VARARGS: 関数が変数の引数を取る
- DUK_API_ENTRY_STACK: 関数エントリ時に確保が保証されているバリュースタックエントリ数


### タグ別APIコール


- 1.0.0
  - [duk_alloc](#duk_alloc)
  - [duk_alloc_raw](#duk_alloc_raw)
  - [duk_base64_decode](#duk_base64_decode)
  - [duk_base64_encode](#duk_base64_encode)
  - [duk_call](#duk_call)
  - [duk_call_method](#duk_call_method)
  - [duk_call_prop](#duk_call_prop)
  - [duk_char_code_at](#duk_char_code_at)
  - [duk_check_stack](#duk_check_stack)
  - [duk_check_stack_top](#duk_check_stack_top)
  - [duk_check_type](#duk_check_type)
  - [duk_check_type_mask](#duk_check_type_mask)
  - [duk_compact](#duk_compact)
  - [duk_compile](#duk_compile)
  - [duk_compile_lstring](#duk_compile_lstring)
  - [duk_compile_lstring_filename](#duk_compile_lstring_filename)
  - [duk_compile_string](#duk_compile_string)
  - [duk_compile_string_filename](#duk_compile_string_filename)
  - [duk_concat](#duk_concat)
  - [duk_copy](#duk_copy)
  - [duk_create_heap](#duk_create_heap)
  - [duk_create_heap_default](#duk_create_heap_default)
  - [duk_decode_string](#duk_decode_string)
  - [duk_del_prop](#duk_del_prop)
  - [duk_del_prop_index](#duk_del_prop_index)
  - [duk_del_prop_string](#duk_del_prop_string)
  - [duk_destroy_heap](#duk_destroy_heap)
  - [duk_dup](#duk_dup)
  - [duk_dup_top](#duk_dup_top)
  - [duk_enum](#duk_enum)
  - [duk_equals](#duk_equals)
  - [duk_error](#duk_error)
  - [duk_eval](#duk_eval)
  - [duk_eval_lstring](#duk_eval_lstring)
  - [duk_eval_lstring_noresult](#duk_eval_lstring_noresult)
  - [duk_eval_noresult](#duk_eval_noresult)
  - [duk_eval_string](#duk_eval_string)
  - [duk_eval_string_noresult](#duk_eval_string_noresult)
  - [duk_fatal](#duk_fatal)
  - [duk_free](#duk_free)
  - [duk_free_raw](#duk_free_raw)
  - [duk_gc](#duk_gc)
  - [duk_get_boolean](#duk_get_boolean)
  - [duk_get_buffer](#duk_get_buffer)
  - [duk_get_c_function](#duk_get_c_function)
  - [duk_get_context](#duk_get_context)
  - [duk_get_current_magic](#duk_get_current_magic)
  - [duk_get_finalizer](#duk_get_finalizer)
  - [duk_get_global_string](#duk_get_global_string)
  - [duk_get_int](#duk_get_int)
  - [duk_get_length](#duk_get_length)
  - [duk_get_lstring](#duk_get_lstring)
  - [duk_get_magic](#duk_get_magic)
  - [duk_get_memory_functions](#duk_get_memory_functions)
  - [duk_get_number](#duk_get_number)
  - [duk_get_pointer](#duk_get_pointer)
  - [duk_get_prop](#duk_get_prop)
  - [duk_get_prop_index](#duk_get_prop_index)
  - [duk_get_prop_string](#duk_get_prop_string)
  - [duk_get_prototype](#duk_get_prototype)
  - [duk_get_string](#duk_get_string)
  - [duk_get_top](#duk_get_top)
  - [duk_get_top_index](#duk_get_top_index)
  - [duk_get_type](#duk_get_type)
  - [duk_get_type_mask](#duk_get_type_mask)
  - [duk_get_uint](#duk_get_uint)
  - [duk_has_prop](#duk_has_prop)
  - [duk_has_prop_index](#duk_has_prop_index)
  - [duk_has_prop_string](#duk_has_prop_string)
  - [duk_hex_decode](#duk_hex_decode)
  - [duk_hex_encode](#duk_hex_encode)
  - [duk_insert](#duk_insert)
  - [duk_is_array](#duk_is_array)
  - [duk_is_boolean](#duk_is_boolean)
  - [duk_is_bound_function](#duk_is_bound_function)
  - [duk_is_buffer](#duk_is_buffer)
  - [duk_is_c_function](#duk_is_c_function)
  - [duk_is_callable](#duk_is_callable)
  - [duk_is_constructor_call](#duk_is_constructor_call)
  - [duk_is_dynamic_buffer](#duk_is_dynamic_buffer)
  - [duk_is_ecmascript_function](#duk_is_ecmascript_function)
  - [duk_is_fixed_buffer](#duk_is_fixed_buffer)
  - [duk_is_function](#duk_is_function)
  - [duk_is_nan](#duk_is_nan)
  - [duk_is_null](#duk_is_null)
  - [duk_is_null_or_undefined](#duk_is_null_or_undefined)
  - [duk_is_number](#duk_is_number)
  - [duk_is_object](#duk_is_object)
  - [duk_is_object_coercible](#duk_is_object_coercible)
  - [duk_is_pointer](#duk_is_pointer)
  - [duk_is_primitive](#duk_is_primitive)
  - [duk_is_strict_call](#duk_is_strict_call)
  - [duk_is_string](#duk_is_string)
  - [duk_is_thread](#duk_is_thread)
  - [duk_is_undefined](#duk_is_undefined)
  - [duk_is_valid_index](#duk_is_valid_index)
  - [duk_join](#duk_join)
  - [duk_json_decode](#duk_json_decode)
  - [duk_json_encode](#duk_json_encode)
  - [duk_map_string](#duk_map_string)
  - [duk_new](#duk_new)
  - [duk_next](#duk_next)
  - [duk_normalize_index](#duk_normalize_index)
  - [duk_pcall](#duk_pcall)
  - [duk_pcall_method](#duk_pcall_method)
  - [duk_pcall_prop](#duk_pcall_prop)
  - [duk_pcompile](#duk_pcompile)
  - [duk_pcompile_lstring](#duk_pcompile_lstring)
  - [duk_pcompile_lstring_filename](#duk_pcompile_lstring_filename)
  - [duk_pcompile_string](#duk_pcompile_string)
  - [duk_pcompile_string_filename](#duk_pcompile_string_filename)
  - [duk_peval](#duk_peval)
  - [duk_peval_lstring](#duk_peval_lstring)
  - [duk_peval_lstring_noresult](#duk_peval_lstring_noresult)
  - [duk_peval_noresult](#duk_peval_noresult)
  - [duk_peval_string](#duk_peval_string)
  - [duk_peval_string_noresult](#duk_peval_string_noresult)
  - [duk_pop](#duk_pop)
  - [duk_pop_2](#duk_pop_2)
  - [duk_pop_3](#duk_pop_3)
  - [duk_pop_n](#duk_pop_n)
  - [duk_push_array](#duk_push_array)
  - [duk_push_boolean](#duk_push_boolean)
  - [duk_push_buffer](#duk_push_buffer)
  - [duk_push_c_function](#duk_push_c_function)
  - [duk_push_context_dump](#duk_push_context_dump)
  - [duk_push_current_function](#duk_push_current_function)
  - [duk_push_current_thread](#duk_push_current_thread)
  - [duk_push_dynamic_buffer](#duk_push_dynamic_buffer)
  - [duk_push_error_object](#duk_push_error_object)
  - [duk_push_false](#duk_push_false)
  - [duk_push_fixed_buffer](#duk_push_fixed_buffer)
  - [duk_push_global_object](#duk_push_global_object)
  - [duk_push_global_stash](#duk_push_global_stash)
  - [duk_push_heap_stash](#duk_push_heap_stash)
  - [duk_push_int](#duk_push_int)
  - [duk_push_lstring](#duk_push_lstring)
  - [duk_push_nan](#duk_push_nan)
  - [duk_push_null](#duk_push_null)
  - [duk_push_number](#duk_push_number)
  - [duk_push_object](#duk_push_object)
  - [duk_push_pointer](#duk_push_pointer)
  - [duk_push_sprintf](#duk_push_sprintf)
  - [duk_push_string](#duk_push_string)
  - [duk_push_this](#duk_push_this)
  - [duk_push_thread](#duk_push_thread)
  - [duk_push_thread_new_globalenv](#duk_push_thread_new_globalenv)
  - [duk_push_thread_stash](#duk_push_thread_stash)
  - [duk_push_true](#duk_push_true)
  - [duk_push_uint](#duk_push_uint)
  - [duk_push_undefined](#duk_push_undefined)
  - [duk_push_vsprintf](#duk_push_vsprintf)
  - [duk_put_function_list](#duk_put_function_list)
  - [duk_put_global_string](#duk_put_global_string)
  - [duk_put_number_list](#duk_put_number_list)
  - [duk_put_prop](#duk_put_prop)
  - [duk_put_prop_index](#duk_put_prop_index)
  - [duk_put_prop_string](#duk_put_prop_string)
  - [duk_realloc](#duk_realloc)
  - [duk_realloc_raw](#duk_realloc_raw)
  - [duk_remove](#duk_remove)
  - [duk_replace](#duk_replace)
  - [duk_require_boolean](#duk_require_boolean)
  - [duk_require_buffer](#duk_require_buffer)
  - [duk_require_c_function](#duk_require_c_function)
  - [duk_require_context](#duk_require_context)
  - [duk_require_int](#duk_require_int)
  - [duk_require_lstring](#duk_require_lstring)
  - [duk_require_normalize_index](#duk_require_normalize_index)
  - [duk_require_null](#duk_require_null)
  - [duk_require_number](#duk_require_number)
  - [duk_require_object_coercible](#duk_require_object_coercible)
  - [duk_require_pointer](#duk_require_pointer)
  - [duk_require_stack](#duk_require_stack)
  - [duk_require_stack_top](#duk_require_stack_top)
  - [duk_require_string](#duk_require_string)
  - [duk_require_top_index](#duk_require_top_index)
  - [duk_require_type_mask](#duk_require_type_mask)
  - [duk_require_uint](#duk_require_uint)
  - [duk_require_undefined](#duk_require_undefined)
  - [duk_require_valid_index](#duk_require_valid_index)
  - [duk_resize_buffer](#duk_resize_buffer)
  - [duk_safe_call](#duk_safe_call)
  - [duk_safe_to_lstring](#duk_safe_to_lstring)
  - [duk_safe_to_string](#duk_safe_to_string)
  - [duk_set_finalizer](#duk_set_finalizer)
  - [duk_set_global_object](#duk_set_global_object)
  - [duk_set_magic](#duk_set_magic)
  - [duk_set_prototype](#duk_set_prototype)
  - [duk_set_top](#duk_set_top)
  - [duk_strict_equals](#duk_strict_equals)
  - [duk_substring](#duk_substring)
  - [duk_swap](#duk_swap)
  - [duk_swap_top](#duk_swap_top)
  - [duk_throw](#duk_throw)
  - [duk_to_boolean](#duk_to_boolean)
  - [duk_to_buffer](#duk_to_buffer)
  - [duk_to_dynamic_buffer](#duk_to_dynamic_buffer)
  - [duk_to_fixed_buffer](#duk_to_fixed_buffer)
  - [duk_to_int](#duk_to_int)
  - [duk_to_int32](#duk_to_int32)
  - [duk_to_lstring](#duk_to_lstring)
  - [duk_to_null](#duk_to_null)
  - [duk_to_number](#duk_to_number)
  - [duk_to_object](#duk_to_object)
  - [duk_to_pointer](#duk_to_pointer)
  - [duk_to_primitive](#duk_to_primitive)
  - [duk_to_string](#duk_to_string)
  - [duk_to_uint](#duk_to_uint)
  - [duk_to_uint16](#duk_to_uint16)
  - [duk_to_uint32](#duk_to_uint32)
  - [duk_to_undefined](#duk_to_undefined)
  - [duk_trim](#duk_trim)
  - [duk_xcopy_top](#duk_xcopy_top)
  - [duk_xmove_top](#duk_xmove_top)
- 1.1.0
  - [duk_def_prop](#duk_def_prop)
  - [duk_error_va](#duk_error_va)
  - [duk_get_error_code](#duk_get_error_code)
  - [duk_get_heapptr](#duk_get_heapptr)
  - [duk_is_error](#duk_is_error)
  - [duk_push_error_object_va](#duk_push_error_object_va)
  - [duk_push_heapptr](#duk_push_heapptr)
  - [duk_require_heapptr](#duk_require_heapptr)
- 1.2.0
  - [duk_debugger_attach](#duk_debugger_attach)
  - [duk_debugger_cooperate](#duk_debugger_cooperate)
  - [duk_debugger_detach](#duk_debugger_detach)
- 1.3.0
  - [duk_config_buffer](#duk_config_buffer)
  - [duk_dump_function](#duk_dump_function)
  - [duk_get_buffer_data](#duk_get_buffer_data)
  - [duk_instanceof](#duk_instanceof)
  - [duk_load_function](#duk_load_function)
  - [duk_pnew](#duk_pnew)
  - [duk_push_buffer_object](#duk_push_buffer_object)
  - [duk_push_external_buffer](#duk_push_external_buffer)
  - [duk_require_buffer_data](#duk_require_buffer_data)
  - [duk_steal_buffer](#duk_steal_buffer)
- 1.4.0
  - [duk_is_eval_error](#duk_is_eval_error)
  - [duk_is_range_error](#duk_is_range_error)
  - [duk_is_reference_error](#duk_is_reference_error)
  - [duk_is_syntax_error](#duk_is_syntax_error)
  - [duk_is_type_error](#duk_is_type_error)
  - [duk_is_uri_error](#duk_is_uri_error)
  - [duk_require_callable](#duk_require_callable)
  - [duk_require_function](#duk_require_function)
- 1.5.0
  - [duk_debugger_notify](#duk_debugger_notify)
  - [duk_debugger_pause](#duk_debugger_pause)
- 1.6.0
  - [duk_resume](#duk_resume)
  - [duk_suspend](#duk_suspend)
- 2.0.0
  - [duk_buffer_to_string](#duk_buffer_to_string)
  - [duk_components_to_time](#duk_components_to_time)
  - [duk_del_prop_lstring](#duk_del_prop_lstring)
  - [duk_eval_error](#duk_eval_error)
  - [duk_eval_error_va](#duk_eval_error_va)
  - [duk_generic_error](#duk_generic_error)
  - [duk_generic_error_va](#duk_generic_error_va)
  - [duk_get_global_lstring](#duk_get_global_lstring)
  - [duk_get_now](#duk_get_now)
  - [duk_get_prop_desc](#duk_get_prop_desc)
  - [duk_get_prop_lstring](#duk_get_prop_lstring)
  - [duk_has_prop_lstring](#duk_has_prop_lstring)
  - [duk_inspect_callstack_entry](#duk_inspect_callstack_entry)
  - [duk_inspect_value](#duk_inspect_value)
  - [duk_is_buffer_data](#duk_is_buffer_data)
  - [duk_is_symbol](#duk_is_symbol)
  - [duk_push_bare_object](#duk_push_bare_object)
  - [duk_put_global_lstring](#duk_put_global_lstring)
  - [duk_put_prop_lstring](#duk_put_prop_lstring)
  - [duk_range_error](#duk_range_error)
  - [duk_range_error_va](#duk_range_error_va)
  - [duk_reference_error](#duk_reference_error)
  - [duk_reference_error_va](#duk_reference_error_va)
  - [duk_samevalue](#duk_samevalue)
  - [duk_set_length](#duk_set_length)
  - [duk_syntax_error](#duk_syntax_error)
  - [duk_syntax_error_va](#duk_syntax_error_va)
  - [duk_time_to_components](#duk_time_to_components)
  - [duk_type_error](#duk_type_error)
  - [duk_type_error_va](#duk_type_error_va)
  - [duk_uri_error](#duk_uri_error)
  - [duk_uri_error_va](#duk_uri_error_va)
- 2.1.0
  - [duk_get_boolean_default](#duk_get_boolean_default)
  - [duk_get_buffer_data_default](#duk_get_buffer_data_default)
  - [duk_get_buffer_default](#duk_get_buffer_default)
  - [duk_get_c_function_default](#duk_get_c_function_default)
  - [duk_get_context_default](#duk_get_context_default)
  - [duk_get_heapptr_default](#duk_get_heapptr_default)
  - [duk_get_int_default](#duk_get_int_default)
  - [duk_get_lstring_default](#duk_get_lstring_default)
  - [duk_get_number_default](#duk_get_number_default)
  - [duk_get_pointer_default](#duk_get_pointer_default)
  - [duk_get_string_default](#duk_get_string_default)
  - [duk_get_uint_default](#duk_get_uint_default)
  - [duk_opt_boolean](#duk_opt_boolean)
  - [duk_opt_buffer](#duk_opt_buffer)
  - [duk_opt_buffer_data](#duk_opt_buffer_data)
  - [duk_opt_c_function](#duk_opt_c_function)
  - [duk_opt_context](#duk_opt_context)
  - [duk_opt_heapptr](#duk_opt_heapptr)
  - [duk_opt_int](#duk_opt_int)
  - [duk_opt_lstring](#duk_opt_lstring)
  - [duk_opt_number](#duk_opt_number)
  - [duk_opt_pointer](#duk_opt_pointer)
  - [duk_opt_string](#duk_opt_string)
  - [duk_opt_uint](#duk_opt_uint)
- 2.2.0
  - [duk_del_prop_heapptr](#duk_del_prop_heapptr)
  - [duk_freeze](#duk_freeze)
  - [duk_get_prop_heapptr](#duk_get_prop_heapptr)
  - [duk_has_prop_heapptr](#duk_has_prop_heapptr)
  - [duk_is_constructable](#duk_is_constructable)
  - [duk_push_proxy](#duk_push_proxy)
  - [duk_put_prop_heapptr](#duk_put_prop_heapptr)
  - [duk_require_object](#duk_require_object)
  - [duk_seal](#duk_seal)
- 2.3.0
  - [duk_del_prop_literal](#duk_del_prop_literal)
  - [duk_get_global_heapptr](#duk_get_global_heapptr)
  - [duk_get_global_literal](#duk_get_global_literal)
  - [duk_get_prop_literal](#duk_get_prop_literal)
  - [duk_has_prop_literal](#duk_has_prop_literal)
  - [duk_push_literal](#duk_push_literal)
  - [duk_push_new_target](#duk_push_new_target)
  - [duk_put_global_heapptr](#duk_put_global_heapptr)
  - [duk_put_global_literal](#duk_put_global_literal)
  - [duk_put_prop_literal](#duk_put_prop_literal)
  - [duk_random](#duk_random)
- 2.4.0
  - [duk_push_bare_array](#duk_push_bare_array)
  - [duk_require_constructable](#duk_require_constructable)
  - [duk_require_constructor_call](#duk_require_constructor_call)
  - [duk_safe_to_stacktrace](#duk_safe_to_stacktrace)
  - [duk_to_stacktrace](#duk_to_stacktrace)
- 2.5.0
  - [duk_cbor_decode](#duk_cbor_decode)
  - [duk_cbor_encode](#duk_cbor_encode)
  - [duk_pull](#duk_pull)
- base64
  - [duk_base64_decode](#duk_base64_decode)
  - [duk_base64_encode](#duk_base64_encode)
- borrowed
  - [duk_del_prop_heapptr](#duk_del_prop_heapptr)
  - [duk_get_context](#duk_get_context)
  - [duk_get_context_default](#duk_get_context_default)
  - [duk_get_global_heapptr](#duk_get_global_heapptr)
  - [duk_get_heapptr](#duk_get_heapptr)
  - [duk_get_heapptr_default](#duk_get_heapptr_default)
  - [duk_get_prop_heapptr](#duk_get_prop_heapptr)
  - [duk_has_prop_heapptr](#duk_has_prop_heapptr)
  - [duk_opt_context](#duk_opt_context)
  - [duk_opt_heapptr](#duk_opt_heapptr)
  - [duk_push_current_thread](#duk_push_current_thread)
  - [duk_push_heapptr](#duk_push_heapptr)
  - [duk_push_thread](#duk_push_thread)
  - [duk_push_thread_new_globalenv](#duk_push_thread_new_globalenv)
  - [duk_put_global_heapptr](#duk_put_global_heapptr)
  - [duk_put_prop_heapptr](#duk_put_prop_heapptr)
  - [duk_require_context](#duk_require_context)
  - [duk_require_heapptr](#duk_require_heapptr)
- buffer
  - [duk_buffer_to_string](#duk_buffer_to_string)
  - [duk_config_buffer](#duk_config_buffer)
  - [duk_get_buffer](#duk_get_buffer)
  - [duk_get_buffer_data](#duk_get_buffer_data)
  - [duk_get_buffer_data_default](#duk_get_buffer_data_default)
  - [duk_get_buffer_default](#duk_get_buffer_default)
  - [duk_is_buffer](#duk_is_buffer)
  - [duk_is_buffer_data](#duk_is_buffer_data)
  - [duk_is_dynamic_buffer](#duk_is_dynamic_buffer)
  - [duk_is_fixed_buffer](#duk_is_fixed_buffer)
  - [duk_opt_buffer](#duk_opt_buffer)
  - [duk_opt_buffer_data](#duk_opt_buffer_data)
  - [duk_push_buffer](#duk_push_buffer)
  - [duk_push_dynamic_buffer](#duk_push_dynamic_buffer)
  - [duk_push_external_buffer](#duk_push_external_buffer)
  - [duk_push_fixed_buffer](#duk_push_fixed_buffer)
  - [duk_require_buffer](#duk_require_buffer)
  - [duk_require_buffer_data](#duk_require_buffer_data)
  - [duk_resize_buffer](#duk_resize_buffer)
  - [duk_steal_buffer](#duk_steal_buffer)
  - [duk_to_buffer](#duk_to_buffer)
  - [duk_to_dynamic_buffer](#duk_to_dynamic_buffer)
  - [duk_to_fixed_buffer](#duk_to_fixed_buffer)
- bufferobject
  - [duk_get_buffer_data](#duk_get_buffer_data)
  - [duk_is_buffer_data](#duk_is_buffer_data)
  - [duk_opt_buffer_data](#duk_opt_buffer_data)
  - [duk_push_buffer_object](#duk_push_buffer_object)
  - [duk_require_buffer_data](#duk_require_buffer_data)
- bytecode
  - [duk_dump_function](#duk_dump_function)
  - [duk_load_function](#duk_load_function)
- call
  - [duk_call](#duk_call)
  - [duk_call_method](#duk_call_method)
  - [duk_call_prop](#duk_call_prop)
  - [duk_new](#duk_new)
  - [duk_pcall](#duk_pcall)
  - [duk_pcall_method](#duk_pcall_method)
  - [duk_pcall_prop](#duk_pcall_prop)
  - [duk_pnew](#duk_pnew)
  - [duk_safe_call](#duk_safe_call)
- cbor
  - [duk_cbor_decode](#duk_cbor_decode)
  - [duk_cbor_encode](#duk_cbor_encode)
- codec
  - [duk_base64_decode](#duk_base64_decode)
  - [duk_base64_encode](#duk_base64_encode)
  - [duk_cbor_decode](#duk_cbor_decode)
  - [duk_cbor_encode](#duk_cbor_encode)
  - [duk_hex_decode](#duk_hex_decode)
  - [duk_hex_encode](#duk_hex_encode)
  - [duk_json_decode](#duk_json_decode)
  - [duk_json_encode](#duk_json_encode)
- compare
  - [duk_equals](#duk_equals)
  - [duk_instanceof](#duk_instanceof)
  - [duk_samevalue](#duk_samevalue)
  - [duk_strict_equals](#duk_strict_equals)
- compile
  - [duk_compile](#duk_compile)
  - [duk_compile_lstring](#duk_compile_lstring)
  - [duk_compile_lstring_filename](#duk_compile_lstring_filename)
  - [duk_compile_string](#duk_compile_string)
  - [duk_compile_string_filename](#duk_compile_string_filename)
  - [duk_eval](#duk_eval)
  - [duk_eval_lstring](#duk_eval_lstring)
  - [duk_eval_lstring_noresult](#duk_eval_lstring_noresult)
  - [duk_eval_noresult](#duk_eval_noresult)
  - [duk_eval_string](#duk_eval_string)
  - [duk_eval_string_noresult](#duk_eval_string_noresult)
  - [duk_pcompile](#duk_pcompile)
  - [duk_pcompile_lstring](#duk_pcompile_lstring)
  - [duk_pcompile_lstring_filename](#duk_pcompile_lstring_filename)
  - [duk_pcompile_string](#duk_pcompile_string)
  - [duk_pcompile_string_filename](#duk_pcompile_string_filename)
  - [duk_peval](#duk_peval)
  - [duk_peval_lstring](#duk_peval_lstring)
  - [duk_peval_lstring_noresult](#duk_peval_lstring_noresult)
  - [duk_peval_noresult](#duk_peval_noresult)
  - [duk_peval_string](#duk_peval_string)
  - [duk_peval_string_noresult](#duk_peval_string_noresult)
- debug
  - [duk_push_context_dump](#duk_push_context_dump)
- debugger
  - [duk_debugger_attach](#duk_debugger_attach)
  - [duk_debugger_cooperate](#duk_debugger_cooperate)
  - [duk_debugger_detach](#duk_debugger_detach)
  - [duk_debugger_notify](#duk_debugger_notify)
  - [duk_debugger_pause](#duk_debugger_pause)
- error
  - [duk_error](#duk_error)
  - [duk_error_va](#duk_error_va)
  - [duk_eval_error](#duk_eval_error)
  - [duk_eval_error_va](#duk_eval_error_va)
  - [duk_fatal](#duk_fatal)
  - [duk_generic_error](#duk_generic_error)
  - [duk_generic_error_va](#duk_generic_error_va)
  - [duk_get_error_code](#duk_get_error_code)
  - [duk_is_error](#duk_is_error)
  - [duk_is_eval_error](#duk_is_eval_error)
  - [duk_is_range_error](#duk_is_range_error)
  - [duk_is_reference_error](#duk_is_reference_error)
  - [duk_is_syntax_error](#duk_is_syntax_error)
  - [duk_is_type_error](#duk_is_type_error)
  - [duk_is_uri_error](#duk_is_uri_error)
  - [duk_push_error_object](#duk_push_error_object)
  - [duk_push_error_object_va](#duk_push_error_object_va)
  - [duk_range_error](#duk_range_error)
  - [duk_range_error_va](#duk_range_error_va)
  - [duk_reference_error](#duk_reference_error)
  - [duk_reference_error_va](#duk_reference_error_va)
  - [duk_syntax_error](#duk_syntax_error)
  - [duk_syntax_error_va](#duk_syntax_error_va)
  - [duk_throw](#duk_throw)
  - [duk_type_error](#duk_type_error)
  - [duk_type_error_va](#duk_type_error_va)
  - [duk_uri_error](#duk_uri_error)
  - [duk_uri_error_va](#duk_uri_error_va)
- experimental
  - [duk_cbor_decode](#duk_cbor_decode)
  - [duk_cbor_encode](#duk_cbor_encode)
- finalizer
  - [duk_get_finalizer](#duk_get_finalizer)
  - [duk_set_finalizer](#duk_set_finalizer)
- function
  - [duk_get_c_function](#duk_get_c_function)
  - [duk_get_c_function_default](#duk_get_c_function_default)
  - [duk_get_current_magic](#duk_get_current_magic)
  - [duk_get_magic](#duk_get_magic)
  - [duk_is_bound_function](#duk_is_bound_function)
  - [duk_is_c_function](#duk_is_c_function)
  - [duk_is_ecmascript_function](#duk_is_ecmascript_function)
  - [duk_is_function](#duk_is_function)
  - [duk_is_strict_call](#duk_is_strict_call)
  - [duk_opt_c_function](#duk_opt_c_function)
  - [duk_push_c_function](#duk_push_c_function)
  - [duk_push_c_lightfunc](#duk_push_c_lightfunc)
  - [duk_push_current_function](#duk_push_current_function)
  - [duk_push_current_thread](#duk_push_current_thread)
  - [duk_push_new_target](#duk_push_new_target)
  - [duk_push_this](#duk_push_this)
  - [duk_require_c_function](#duk_require_c_function)
  - [duk_set_magic](#duk_set_magic)
- heap
  - [duk_create_heap](#duk_create_heap)
  - [duk_create_heap_default](#duk_create_heap_default)
  - [duk_destroy_heap](#duk_destroy_heap)
  - [duk_gc](#duk_gc)
  - [duk_get_memory_functions](#duk_get_memory_functions)
- heapptr
  - [duk_del_prop_heapptr](#duk_del_prop_heapptr)
  - [duk_get_global_heapptr](#duk_get_global_heapptr)
  - [duk_get_heapptr](#duk_get_heapptr)
  - [duk_get_heapptr_default](#duk_get_heapptr_default)
  - [duk_get_prop_heapptr](#duk_get_prop_heapptr)
  - [duk_has_prop_heapptr](#duk_has_prop_heapptr)
  - [duk_opt_heapptr](#duk_opt_heapptr)
  - [duk_push_heapptr](#duk_push_heapptr)
  - [duk_put_global_heapptr](#duk_put_global_heapptr)
  - [duk_put_prop_heapptr](#duk_put_prop_heapptr)
  - [duk_require_heapptr](#duk_require_heapptr)
- hex
  - [duk_hex_decode](#duk_hex_decode)
  - [duk_hex_encode](#duk_hex_encode)
- inspect
  - [duk_inspect_callstack_entry](#duk_inspect_callstack_entry)
  - [duk_inspect_value](#duk_inspect_value)
- json
  - [duk_json_decode](#duk_json_decode)
  - [duk_json_encode](#duk_json_encode)
- lightfunc
  - [duk_push_c_lightfunc](#duk_push_c_lightfunc)
- literal
  - [duk_del_prop_literal](#duk_del_prop_literal)
  - [duk_get_global_literal](#duk_get_global_literal)
  - [duk_get_prop_literal](#duk_get_prop_literal)
  - [duk_has_prop_literal](#duk_has_prop_literal)
  - [duk_push_literal](#duk_push_literal)
  - [duk_put_global_literal](#duk_put_global_literal)
  - [duk_put_prop_literal](#duk_put_prop_literal)
- magic
  - [duk_get_current_magic](#duk_get_current_magic)
  - [duk_get_magic](#duk_get_magic)
  - [duk_set_magic](#duk_set_magic)
- memory
  - [duk_alloc](#duk_alloc)
  - [duk_alloc_raw](#duk_alloc_raw)
  - [duk_compact](#duk_compact)
  - [duk_free](#duk_free)
  - [duk_free_raw](#duk_free_raw)
  - [duk_gc](#duk_gc)
  - [duk_get_memory_functions](#duk_get_memory_functions)
  - [duk_realloc](#duk_realloc)
  - [duk_realloc_raw](#duk_realloc_raw)
- module
  - [duk_push_global_stash](#duk_push_global_stash)
  - [duk_push_heap_stash](#duk_push_heap_stash)
  - [duk_push_thread_stash](#duk_push_thread_stash)
  - [duk_put_function_list](#duk_put_function_list)
  - [duk_put_number_list](#duk_put_number_list)
- object
  - [duk_compact](#duk_compact)
  - [duk_enum](#duk_enum)
  - [duk_freeze](#duk_freeze)
  - [duk_get_finalizer](#duk_get_finalizer)
  - [duk_get_prototype](#duk_get_prototype)
  - [duk_is_object](#duk_is_object)
  - [duk_is_object_coercible](#duk_is_object_coercible)
  - [duk_new](#duk_new)
  - [duk_next](#duk_next)
  - [duk_pnew](#duk_pnew)
  - [duk_push_array](#duk_push_array)
  - [duk_push_bare_array](#duk_push_bare_array)
  - [duk_push_bare_object](#duk_push_bare_object)
  - [duk_push_error_object](#duk_push_error_object)
  - [duk_push_error_object_va](#duk_push_error_object_va)
  - [duk_push_global_object](#duk_push_global_object)
  - [duk_push_global_stash](#duk_push_global_stash)
  - [duk_push_heap_stash](#duk_push_heap_stash)
  - [duk_push_heapptr](#duk_push_heapptr)
  - [duk_push_object](#duk_push_object)
  - [duk_push_proxy](#duk_push_proxy)
  - [duk_push_thread_stash](#duk_push_thread_stash)
  - [duk_require_object_coercible](#duk_require_object_coercible)
  - [duk_seal](#duk_seal)
  - [duk_set_finalizer](#duk_set_finalizer)
  - [duk_set_prototype](#duk_set_prototype)
  - [duk_to_object](#duk_to_object)
- property
  - [duk_call_prop](#duk_call_prop)
  - [duk_compact](#duk_compact)
  - [duk_def_prop](#duk_def_prop)
  - [duk_del_prop](#duk_del_prop)
  - [duk_del_prop_heapptr](#duk_del_prop_heapptr)
  - [duk_del_prop_index](#duk_del_prop_index)
  - [duk_del_prop_literal](#duk_del_prop_literal)
  - [duk_del_prop_lstring](#duk_del_prop_lstring)
  - [duk_del_prop_string](#duk_del_prop_string)
  - [duk_enum](#duk_enum)
  - [duk_freeze](#duk_freeze)
  - [duk_get_global_heapptr](#duk_get_global_heapptr)
  - [duk_get_global_literal](#duk_get_global_literal)
  - [duk_get_global_lstring](#duk_get_global_lstring)
  - [duk_get_global_string](#duk_get_global_string)
  - [duk_get_prop](#duk_get_prop)
  - [duk_get_prop_desc](#duk_get_prop_desc)
  - [duk_get_prop_heapptr](#duk_get_prop_heapptr)
  - [duk_get_prop_index](#duk_get_prop_index)
  - [duk_get_prop_literal](#duk_get_prop_literal)
  - [duk_get_prop_lstring](#duk_get_prop_lstring)
  - [duk_get_prop_string](#duk_get_prop_string)
  - [duk_has_prop](#duk_has_prop)
  - [duk_has_prop_heapptr](#duk_has_prop_heapptr)
  - [duk_has_prop_index](#duk_has_prop_index)
  - [duk_has_prop_literal](#duk_has_prop_literal)
  - [duk_has_prop_lstring](#duk_has_prop_lstring)
  - [duk_has_prop_string](#duk_has_prop_string)
  - [duk_next](#duk_next)
  - [duk_pcall_prop](#duk_pcall_prop)
  - [duk_put_function_list](#duk_put_function_list)
  - [duk_put_global_heapptr](#duk_put_global_heapptr)
  - [duk_put_global_literal](#duk_put_global_literal)
  - [duk_put_global_lstring](#duk_put_global_lstring)
  - [duk_put_global_string](#duk_put_global_string)
  - [duk_put_number_list](#duk_put_number_list)
  - [duk_put_prop](#duk_put_prop)
  - [duk_put_prop_heapptr](#duk_put_prop_heapptr)
  - [duk_put_prop_index](#duk_put_prop_index)
  - [duk_put_prop_literal](#duk_put_prop_literal)
  - [duk_put_prop_lstring](#duk_put_prop_lstring)
  - [duk_put_prop_string](#duk_put_prop_string)
  - [duk_seal](#duk_seal)
- protected
  - [duk_pcall](#duk_pcall)
  - [duk_pcall_method](#duk_pcall_method)
  - [duk_pcall_prop](#duk_pcall_prop)
  - [duk_pcompile](#duk_pcompile)
  - [duk_pcompile_lstring](#duk_pcompile_lstring)
  - [duk_pcompile_lstring_filename](#duk_pcompile_lstring_filename)
  - [duk_pcompile_string](#duk_pcompile_string)
  - [duk_pcompile_string_filename](#duk_pcompile_string_filename)
  - [duk_peval](#duk_peval)
  - [duk_peval_lstring](#duk_peval_lstring)
  - [duk_peval_lstring_noresult](#duk_peval_lstring_noresult)
  - [duk_peval_noresult](#duk_peval_noresult)
  - [duk_peval_string](#duk_peval_string)
  - [duk_peval_string_noresult](#duk_peval_string_noresult)
  - [duk_pnew](#duk_pnew)
  - [duk_safe_call](#duk_safe_call)
  - [duk_safe_to_stacktrace](#duk_safe_to_stacktrace)
  - [duk_safe_to_string](#duk_safe_to_string)
- prototype
  - [duk_get_prototype](#duk_get_prototype)
  - [duk_set_prototype](#duk_set_prototype)
- random
  - [duk_random](#duk_random)
- sandbox
  - [duk_def_prop](#duk_def_prop)
  - [duk_get_prop_desc](#duk_get_prop_desc)
  - [duk_push_global_stash](#duk_push_global_stash)
  - [duk_push_heap_stash](#duk_push_heap_stash)
  - [duk_push_thread_stash](#duk_push_thread_stash)
  - [duk_set_global_object](#duk_set_global_object)
- slice
  - [duk_xcopy_top](#duk_xcopy_top)
  - [duk_xmove_top](#duk_xmove_top)
- stack
  - [duk_buffer_to_string](#duk_buffer_to_string)
  - [duk_check_stack](#duk_check_stack)
  - [duk_check_stack_top](#duk_check_stack_top)
  - [duk_check_type](#duk_check_type)
  - [duk_check_type_mask](#duk_check_type_mask)
  - [duk_config_buffer](#duk_config_buffer)
  - [duk_copy](#duk_copy)
  - [duk_dump_function](#duk_dump_function)
  - [duk_dup](#duk_dup)
  - [duk_dup_top](#duk_dup_top)
  - [duk_get_boolean](#duk_get_boolean)
  - [duk_get_boolean_default](#duk_get_boolean_default)
  - [duk_get_buffer](#duk_get_buffer)
  - [duk_get_buffer_data](#duk_get_buffer_data)
  - [duk_get_buffer_data_default](#duk_get_buffer_data_default)
  - [duk_get_buffer_default](#duk_get_buffer_default)
  - [duk_get_c_function](#duk_get_c_function)
  - [duk_get_c_function_default](#duk_get_c_function_default)
  - [duk_get_context](#duk_get_context)
  - [duk_get_context_default](#duk_get_context_default)
  - [duk_get_error_code](#duk_get_error_code)
  - [duk_get_heapptr](#duk_get_heapptr)
  - [duk_get_heapptr_default](#duk_get_heapptr_default)
  - [duk_get_int](#duk_get_int)
  - [duk_get_int_default](#duk_get_int_default)
  - [duk_get_length](#duk_get_length)
  - [duk_get_lstring](#duk_get_lstring)
  - [duk_get_lstring_default](#duk_get_lstring_default)
  - [duk_get_number](#duk_get_number)
  - [duk_get_number_default](#duk_get_number_default)
  - [duk_get_pointer](#duk_get_pointer)
  - [duk_get_pointer_default](#duk_get_pointer_default)
  - [duk_get_string](#duk_get_string)
  - [duk_get_string_default](#duk_get_string_default)
  - [duk_get_top](#duk_get_top)
  - [duk_get_top_index](#duk_get_top_index)
  - [duk_get_type](#duk_get_type)
  - [duk_get_type_mask](#duk_get_type_mask)
  - [duk_get_uint](#duk_get_uint)
  - [duk_get_uint_default](#duk_get_uint_default)
  - [duk_insert](#duk_insert)
  - [duk_inspect_callstack_entry](#duk_inspect_callstack_entry)
  - [duk_inspect_value](#duk_inspect_value)
  - [duk_is_array](#duk_is_array)
  - [duk_is_boolean](#duk_is_boolean)
  - [duk_is_bound_function](#duk_is_bound_function)
  - [duk_is_buffer](#duk_is_buffer)
  - [duk_is_buffer_data](#duk_is_buffer_data)
  - [duk_is_c_function](#duk_is_c_function)
  - [duk_is_callable](#duk_is_callable)
  - [duk_is_constructable](#duk_is_constructable)
  - [duk_is_constructor_call](#duk_is_constructor_call)
  - [duk_is_dynamic_buffer](#duk_is_dynamic_buffer)
  - [duk_is_ecmascript_function](#duk_is_ecmascript_function)
  - [duk_is_error](#duk_is_error)
  - [duk_is_eval_error](#duk_is_eval_error)
  - [duk_is_fixed_buffer](#duk_is_fixed_buffer)
  - [duk_is_function](#duk_is_function)
  - [duk_is_lightfunc](#duk_is_lightfunc)
  - [duk_is_nan](#duk_is_nan)
  - [duk_is_null](#duk_is_null)
  - [duk_is_null_or_undefined](#duk_is_null_or_undefined)
  - [duk_is_number](#duk_is_number)
  - [duk_is_object](#duk_is_object)
  - [duk_is_object_coercible](#duk_is_object_coercible)
  - [duk_is_pointer](#duk_is_pointer)
  - [duk_is_primitive](#duk_is_primitive)
  - [duk_is_range_error](#duk_is_range_error)
  - [duk_is_reference_error](#duk_is_reference_error)
  - [duk_is_string](#duk_is_string)
  - [duk_is_symbol](#duk_is_symbol)
  - [duk_is_syntax_error](#duk_is_syntax_error)
  - [duk_is_thread](#duk_is_thread)
  - [duk_is_type_error](#duk_is_type_error)
  - [duk_is_undefined](#duk_is_undefined)
  - [duk_is_uri_error](#duk_is_uri_error)
  - [duk_is_valid_index](#duk_is_valid_index)
  - [duk_load_function](#duk_load_function)
  - [duk_normalize_index](#duk_normalize_index)
  - [duk_opt_boolean](#duk_opt_boolean)
  - [duk_opt_buffer](#duk_opt_buffer)
  - [duk_opt_buffer_data](#duk_opt_buffer_data)
  - [duk_opt_c_function](#duk_opt_c_function)
  - [duk_opt_context](#duk_opt_context)
  - [duk_opt_heapptr](#duk_opt_heapptr)
  - [duk_opt_int](#duk_opt_int)
  - [duk_opt_lstring](#duk_opt_lstring)
  - [duk_opt_number](#duk_opt_number)
  - [duk_opt_pointer](#duk_opt_pointer)
  - [duk_opt_string](#duk_opt_string)
  - [duk_opt_uint](#duk_opt_uint)
  - [duk_pop](#duk_pop)
  - [duk_pop_2](#duk_pop_2)
  - [duk_pop_3](#duk_pop_3)
  - [duk_pop_n](#duk_pop_n)
  - [duk_pull](#duk_pull)
  - [duk_push_array](#duk_push_array)
  - [duk_push_bare_array](#duk_push_bare_array)
  - [duk_push_bare_object](#duk_push_bare_object)
  - [duk_push_boolean](#duk_push_boolean)
  - [duk_push_buffer](#duk_push_buffer)
  - [duk_push_buffer_object](#duk_push_buffer_object)
  - [duk_push_c_function](#duk_push_c_function)
  - [duk_push_c_lightfunc](#duk_push_c_lightfunc)
  - [duk_push_context_dump](#duk_push_context_dump)
  - [duk_push_current_function](#duk_push_current_function)
  - [duk_push_current_thread](#duk_push_current_thread)
  - [duk_push_dynamic_buffer](#duk_push_dynamic_buffer)
  - [duk_push_error_object](#duk_push_error_object)
  - [duk_push_error_object_va](#duk_push_error_object_va)
  - [duk_push_external_buffer](#duk_push_external_buffer)
  - [duk_push_false](#duk_push_false)
  - [duk_push_fixed_buffer](#duk_push_fixed_buffer)
  - [duk_push_global_object](#duk_push_global_object)
  - [duk_push_global_stash](#duk_push_global_stash)
  - [duk_push_heap_stash](#duk_push_heap_stash)
  - [duk_push_heapptr](#duk_push_heapptr)
  - [duk_push_int](#duk_push_int)
  - [duk_push_literal](#duk_push_literal)
  - [duk_push_lstring](#duk_push_lstring)
  - [duk_push_nan](#duk_push_nan)
  - [duk_push_new_target](#duk_push_new_target)
  - [duk_push_null](#duk_push_null)
  - [duk_push_number](#duk_push_number)
  - [duk_push_object](#duk_push_object)
  - [duk_push_pointer](#duk_push_pointer)
  - [duk_push_proxy](#duk_push_proxy)
  - [duk_push_sprintf](#duk_push_sprintf)
  - [duk_push_string](#duk_push_string)
  - [duk_push_this](#duk_push_this)
  - [duk_push_thread](#duk_push_thread)
  - [duk_push_thread_new_globalenv](#duk_push_thread_new_globalenv)
  - [duk_push_thread_stash](#duk_push_thread_stash)
  - [duk_push_true](#duk_push_true)
  - [duk_push_uint](#duk_push_uint)
  - [duk_push_undefined](#duk_push_undefined)
  - [duk_push_vsprintf](#duk_push_vsprintf)
  - [duk_remove](#duk_remove)
  - [duk_replace](#duk_replace)
  - [duk_require_boolean](#duk_require_boolean)
  - [duk_require_buffer](#duk_require_buffer)
  - [duk_require_buffer_data](#duk_require_buffer_data)
  - [duk_require_c_function](#duk_require_c_function)
  - [duk_require_callable](#duk_require_callable)
  - [duk_require_constructable](#duk_require_constructable)
  - [duk_require_constructor_call](#duk_require_constructor_call)
  - [duk_require_context](#duk_require_context)
  - [duk_require_function](#duk_require_function)
  - [duk_require_heapptr](#duk_require_heapptr)
  - [duk_require_int](#duk_require_int)
  - [duk_require_lstring](#duk_require_lstring)
  - [duk_require_normalize_index](#duk_require_normalize_index)
  - [duk_require_null](#duk_require_null)
  - [duk_require_number](#duk_require_number)
  - [duk_require_object](#duk_require_object)
  - [duk_require_object_coercible](#duk_require_object_coercible)
  - [duk_require_pointer](#duk_require_pointer)
  - [duk_require_stack](#duk_require_stack)
  - [duk_require_stack_top](#duk_require_stack_top)
  - [duk_require_string](#duk_require_string)
  - [duk_require_top_index](#duk_require_top_index)
  - [duk_require_type_mask](#duk_require_type_mask)
  - [duk_require_uint](#duk_require_uint)
  - [duk_require_undefined](#duk_require_undefined)
  - [duk_require_valid_index](#duk_require_valid_index)
  - [duk_resize_buffer](#duk_resize_buffer)
  - [duk_safe_to_lstring](#duk_safe_to_lstring)
  - [duk_safe_to_stacktrace](#duk_safe_to_stacktrace)
  - [duk_safe_to_string](#duk_safe_to_string)
  - [duk_set_global_object](#duk_set_global_object)
  - [duk_set_length](#duk_set_length)
  - [duk_set_top](#duk_set_top)
  - [duk_steal_buffer](#duk_steal_buffer)
  - [duk_swap](#duk_swap)
  - [duk_swap_top](#duk_swap_top)
  - [duk_to_boolean](#duk_to_boolean)
  - [duk_to_buffer](#duk_to_buffer)
  - [duk_to_dynamic_buffer](#duk_to_dynamic_buffer)
  - [duk_to_fixed_buffer](#duk_to_fixed_buffer)
  - [duk_to_int](#duk_to_int)
  - [duk_to_int32](#duk_to_int32)
  - [duk_to_lstring](#duk_to_lstring)
  - [duk_to_null](#duk_to_null)
  - [duk_to_number](#duk_to_number)
  - [duk_to_object](#duk_to_object)
  - [duk_to_pointer](#duk_to_pointer)
  - [duk_to_primitive](#duk_to_primitive)
  - [duk_to_stacktrace](#duk_to_stacktrace)
  - [duk_to_string](#duk_to_string)
  - [duk_to_uint](#duk_to_uint)
  - [duk_to_uint16](#duk_to_uint16)
  - [duk_to_uint32](#duk_to_uint32)
  - [duk_to_undefined](#duk_to_undefined)
  - [duk_trim](#duk_trim)
  - [duk_xcopy_top](#duk_xcopy_top)
  - [duk_xmove_top](#duk_xmove_top)
- stash
  - [duk_push_global_stash](#duk_push_global_stash)
  - [duk_push_heap_stash](#duk_push_heap_stash)
  - [duk_push_thread_stash](#duk_push_thread_stash)
- string
  - [duk_buffer_to_string](#duk_buffer_to_string)
  - [duk_char_code_at](#duk_char_code_at)
  - [duk_compile_lstring](#duk_compile_lstring)
  - [duk_compile_lstring_filename](#duk_compile_lstring_filename)
  - [duk_compile_string](#duk_compile_string)
  - [duk_compile_string_filename](#duk_compile_string_filename)
  - [duk_concat](#duk_concat)
  - [duk_decode_string](#duk_decode_string)
  - [duk_del_prop_lstring](#duk_del_prop_lstring)
  - [duk_del_prop_string](#duk_del_prop_string)
  - [duk_eval_lstring](#duk_eval_lstring)
  - [duk_eval_lstring_noresult](#duk_eval_lstring_noresult)
  - [duk_eval_string](#duk_eval_string)
  - [duk_eval_string_noresult](#duk_eval_string_noresult)
  - [duk_get_global_lstring](#duk_get_global_lstring)
  - [duk_get_global_string](#duk_get_global_string)
  - [duk_get_lstring](#duk_get_lstring)
  - [duk_get_lstring_default](#duk_get_lstring_default)
  - [duk_get_prop_lstring](#duk_get_prop_lstring)
  - [duk_get_prop_string](#duk_get_prop_string)
  - [duk_get_string](#duk_get_string)
  - [duk_get_string_default](#duk_get_string_default)
  - [duk_has_prop_lstring](#duk_has_prop_lstring)
  - [duk_has_prop_string](#duk_has_prop_string)
  - [duk_is_string](#duk_is_string)
  - [duk_is_symbol](#duk_is_symbol)
  - [duk_join](#duk_join)
  - [duk_map_string](#duk_map_string)
  - [duk_opt_lstring](#duk_opt_lstring)
  - [duk_opt_string](#duk_opt_string)
  - [duk_pcompile_lstring](#duk_pcompile_lstring)
  - [duk_pcompile_lstring_filename](#duk_pcompile_lstring_filename)
  - [duk_pcompile_string](#duk_pcompile_string)
  - [duk_pcompile_string_filename](#duk_pcompile_string_filename)
  - [duk_peval_lstring](#duk_peval_lstring)
  - [duk_peval_lstring_noresult](#duk_peval_lstring_noresult)
  - [duk_peval_string](#duk_peval_string)
  - [duk_peval_string_noresult](#duk_peval_string_noresult)
  - [duk_push_lstring](#duk_push_lstring)
  - [duk_push_sprintf](#duk_push_sprintf)
  - [duk_push_string](#duk_push_string)
  - [duk_push_vsprintf](#duk_push_vsprintf)
  - [duk_put_global_lstring](#duk_put_global_lstring)
  - [duk_put_global_string](#duk_put_global_string)
  - [duk_put_prop_lstring](#duk_put_prop_lstring)
  - [duk_put_prop_string](#duk_put_prop_string)
  - [duk_require_lstring](#duk_require_lstring)
  - [duk_require_string](#duk_require_string)
  - [duk_safe_to_lstring](#duk_safe_to_lstring)
  - [duk_safe_to_stacktrace](#duk_safe_to_stacktrace)
  - [duk_safe_to_string](#duk_safe_to_string)
  - [duk_substring](#duk_substring)
  - [duk_to_lstring](#duk_to_lstring)
  - [duk_to_stacktrace](#duk_to_stacktrace)
  - [duk_to_string](#duk_to_string)
  - [duk_trim](#duk_trim)
- symbol
  - [duk_is_symbol](#duk_is_symbol)
- thread
  - [duk_is_thread](#duk_is_thread)
  - [duk_push_current_thread](#duk_push_current_thread)
  - [duk_push_thread](#duk_push_thread)
  - [duk_push_thread_new_globalenv](#duk_push_thread_new_globalenv)
  - [duk_push_thread_stash](#duk_push_thread_stash)
  - [duk_resume](#duk_resume)
  - [duk_set_global_object](#duk_set_global_object)
  - [duk_suspend](#duk_suspend)
- time
  - [duk_components_to_time](#duk_components_to_time)
  - [duk_get_now](#duk_get_now)
  - [duk_time_to_components](#duk_time_to_components)
- vararg
  - [duk_error_va](#duk_error_va)
  - [duk_eval_error_va](#duk_eval_error_va)
  - [duk_generic_error_va](#duk_generic_error_va)
  - [duk_push_error_object_va](#duk_push_error_object_va)
  - [duk_push_vsprintf](#duk_push_vsprintf)
  - [duk_range_error_va](#duk_range_error_va)
  - [duk_reference_error_va](#duk_reference_error_va)
  - [duk_syntax_error_va](#duk_syntax_error_va)
  - [duk_type_error_va](#duk_type_error_va)
  - [duk_uri_error_va](#duk_uri_error_va)


## duk_alloc() 

1.0.0  memory

### プロトタイプ

```c
void *duk_alloc(duk_context *ctx, duk_size_t size);
```

### スタック

(バリュースタックに影響なし)

### 概要

duk_alloc_raw() と同様ですが、要求を満たすためにガベージコレクションを起動させることがあります。しかし、割り当てられたメモリ自体は自動的にガベージコレクションされません。ガベージコレクションの後でも、割り当て要求は失敗することがあり、その場合は NULL が返されます。割り当てられたメモリは自動的にゼロにされないので、任意のゴミを含む可能性があります。

duk_alloc() で割り当てられたメモリは、duk_free() または duk_free_raw() で解放することができます。

### 例

```c
/* Although duk_alloc() triggers a GC if necessary, it can still fail to
 * allocate the desired amount of memory.  Caller must check for NULL
 * (however, if allocation size is 0, a NULL may be returned even in
 * a success case).
 */
void *buf = duk_alloc(ctx, 1024);
if (buf) {
    printf("allocation successful: %p\n", buf);
} else {
    printf("allocation failed\n");
}
```

### 参照

- duk_alloc_raw


## duk_alloc_raw() 

1.0.0  memory


### プロトタイプ

```c
void *duk_alloc_raw(duk_context *ctx, duk_size_t size);
```

### スタック

(バリュースタックに影響なし)

### 要約

コンテキストに登録された生の割り当て関数を使用して、サイズバイトを割り当てます。割り当てに失敗した場合、NULL を返す。size が 0 の場合、この呼び出しは NULL を返すか、あるいは duk_free_raw() などに安全に渡すことができる非 NULL 値を返すかもしれない。また、割り当てられたメモリは自動的にガベージコレクションされません。割り当てられたメモリは自動的にゼロにされず、任意のゴミを含む可能性があります。

duk_alloc_raw() で割り当てられたメモリは、 duk_free() か duk_free_raw() で解放することができます。

### 例

```c
void *buf = duk_alloc_raw(ctx, 1024);
if (buf) {
    printf("allocation successful: %p\n", buf);
} else {
    printf("allocation failed\n");
}
```

### 参照

- duk_alloc


## duk_base64_decode() 

1.0.0 codec base64

### プロトタイプ

```c
void duk_base64_decode(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | base64_val | ... | -> | ... | val | ... |

### 要約

base-64でエンコードされた値をインプレース操作でバッファにデコードします。入力が無効な場合、エラーを投げます。。

### 例

```c
duk_push_string(ctx, "Zm9v");
duk_base64_decode(ctx, -1);  /* buffer */
printf("base-64 decoded: %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);

/* Output:
 * base-64 decoded: foo
 */
```

### 参照

duk_base64_encode


## duk_base64_encode() 

1.0.0 codec base64

### プロトタイプ

```c
const char *duk_base64_encode(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | base64_val | ... |

### 要約

任意の値をバッファにコピーし、その結果をインプレース操作でbase-64にエンコードします。便宜上、結果の文字列へのポインタを返します。

バッファへの強制は，まずバッファ以外の値を文字列に強制し，次にその文字列をバッファに強制します。結果として得られるバッファは、CESU-8エンコーディングの文字列を含む。

### 例

```c
duk_push_string(ctx, "foo");
printf("base-64 encoded: %s\n", duk_base64_encode(ctx, -1));

/* Output:
 * base-64 encoded: Zm9v
 */
```

### 参照

duk_base64_decode


## duk_buffer_to_string() 

2.0.0 string stack buffer

### プロトタイプ

```c
const char *duk_buffer_to_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | buffer_to_string(val) | ... |

### 要約

idx のバッファ値 (プレーンバッファまたは任意のバッファオブジェクト) を、内部バイト表現がバッファから 1:1 で取得される文字列に置き換えます (バッファデータが duk_push_lstring() でプッシュされた場合と同じです)。読み取り専用でNUL終端の文字列データへの非NULLポインタを返します。(1) 引数がバッファ値でない場合、(2) 引数がバッファオブジェクトで、そのバッキングバッファが見かけのバッファサイズをカバーしていない場合、(3) インデックスが無効な場合、TypeError がスローされます。

バッファデータはそのまま文字列の生成に使用されるため、本APIコールでは、バッファデータのサニタイズ処理を行わない限り、Symbol値の生成は可能です。

カスタムの型強制については、型変換とテストに記載されています。

Duktape 2.0以降、バッファ値に対するduk_to_string()の強制は、通常"[object Uint8Array]" のようになりますが、これは通常意図されたものではありません。

### 例

```c
unsigned char *ptr;
ptr = (unsigned char *) duk_push_fixed_buffer(ctx, 4);
ptr[0] = 0x61;
ptr[1] = 0x62;
ptr[2] = 0x63;
ptr[3] = 0x64;  /* 61 62 63 64 */
printf("coerced string: %s\n", duk_buffer_to_string(ctx, -1));
duk_pop(ctx);
```



## duk_call() 

1.0.0 call

### プロトタイプ

```c
void duk_call(duk_context *ctx, duk_idx_t nargs);
```

### スタック

| ... | func | arg1 | ... | argN | -> | ... | retval |

### 要約

ターゲット関数funcをnargsの引数で呼び出す（関数自体を除く）。関数とその引数は、単一の戻り値に置き換えられます。関数呼び出し中に投げられたエラーは、自動的に捕捉されない。

このバインディングのターゲット関数は、初期状態では未定義に設定されています。ターゲット関数が厳密でない場合、バインディングは、関数が呼び出される前にグローバルオブジェクトに置き換えられます。「関数コードの入力」を参照してください。このバインディングを制御したい場合は、代わりに duk_call_method() または duk_call_prop() を使用することができます。

このAPIコールは、以下と同等です。

```javascript
var retval = func(arg1, ..., argN);
```

または

```javascript
var retval = func.call(undefined, arg1, ..., argN);
```

### 例

```c
/* Assume target function is already on stack at func_idx; the target
 * function adds arguments and returns the result.
 */
duk_idx_t func_idx = /* ... */;

duk_dup(ctx, func_idx);
duk_push_int(ctx, 2);
duk_push_int(ctx, 3);
duk_call(ctx, 2);  /* [ ... func 2 3 ] -> [ ... 5 ] */
printf("2+3=%ld\n", (long) duk_get_int(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_call_method
duk_call_prop


## duk_call_method() 

1.0.0 call

### プロトタイプ

```c
void duk_call_method(duk_context *ctx, duk_idx_t nargs);
```

### スタック

| ... | func | this | arg | ... | argN | -> | ... | retval |

### 要約

ターゲット関数funcを明示的なthisバインディングとnargs引数（関数やthisバインディングの値はカウントしない）で呼び出します。関数オブジェクト、thisバインディング値、関数引数は、1つの戻り値に置き換えられます。関数呼び出し中に投げられたエラーは、自動的に捕捉されない。

ターゲット関数が厳密でない場合、ターゲット関数が見るバインディング値は、関数コードの入力で指定された処理によって変更されることがあります。

本APIコールは、以下と同等です。

```javascript
var retval = func.call(this_binding, arg1, ..., argN);
```

### 例

```c
/* The target function here prints:
 *
 *    this: 123
 *    2 3
 *
 * and returns 5.
 */

duk_push_string(ctx, "(function(x,y) { print('this:', this); "
                     "print(x,y); return x+y; })");
duk_eval(ctx);  /* -> [ ... func ] */
duk_push_int(ctx, 123);
duk_push_int(ctx, 2);
duk_push_int(ctx, 3);
duk_call_method(ctx, 2);  /* [ ... func 123 2 3 ] -> [ ... 5 ] */
printf("2+3=%ld\n", (long) duk_get_int(ctx, -1));  /* prints 5 */
duk_pop(ctx);
```



## duk_call_prop() 

1.0.0 property call

### プロトタイプ

```c
void duk_call_prop(duk_context *ctx, duk_idx_t obj_idx, duk_idx_t nargs);
```

### スタック

| ... | obj | ... | key arg1 | ... | argN | -> | ... | obj | ... | retval |

### 要約

このバインディングを obj に設定し、nargs 引数で obj.key を呼び出します。プロパティ名と関数引数は単一の戻り値に置き換えられ、ターゲットオブジェクトはタッチされません。関数呼び出し中のエラーは自動的に捕捉されない。

ターゲット関数が厳密でない場合、ターゲット関数が見るバインディングの値は、関数コードの入力で指定された処理によって変更されることがあります。

本APIコールは、以下と同等です。

```javascript
var retval = obj[key](arg1, ..., argN);
```

or:

```javascript
var func = obj[key];
var retval = func.call(obj, arg1, ..., argN);
```

プロパティアクセスの基本値は通常オブジェクトですが、技術的には任意の値を使用することができます。文字列やバッファの値には仮想的なインデックスプロパティがあり、例えば "foo"[2]にアクセスすることができます。また、ほとんどのプリミティブ値は何らかのプロトタイプオブジェクトを継承しているので、例えば (12345).toString(16) のようにメソッドを呼び出すことができます。

### 例

```c
/* obj.myAdderMethod(2,3) -> 5 */
duk_idx_t obj_idx = /* ... */;

duk_push_string(ctx, "myAdderMethod");
duk_push_int(ctx, 2);
duk_push_int(ctx, 3);
duk_call_prop(ctx, obj_idx, 2);  /* [ ... "myAdderMethod" 2 3 ] -> [ ... 5 ] */
printf("2+3=%ld\n", (long) duk_get_int(ctx, -1));
duk_pop(ctx);
```



## duk_cbor_decode() 

2.5.0 experimental codec cbor

### プロトタイプ

```c
void duk_cbor_decode(duk_context *ctx, duk_idx_t idx, duk_uint_t decode_flags);
```

### スタック

| ... | cbor_val | ... | -> | ... | val | ... |

### 要約

CBORでエンコードされた値（任意のバッファタイプで与えられる）をインプレース操作でデコードします。結果の値は任意のECMAScriptの値になります。現在、フラグは定義されていませんので、フラグに0を渡してください。

CBOR から ECMAScript の値へのマッピングは実験的なもので、デコード結果は時間の経過とともに変化する可能性があります。例えば、ECMAScript 値のエンコードとデコードのラウンドトリッピングを改善するために、カスタム CBOR タグが追加されるでしょう。

### 例

```c
duk_cbor_decode(ctx, -1, 0);
```

### 参照

duk_cbor_encode


## duk_cbor_encode() 

2.5.0 experimental codec cbor

### プロトタイプ

```c
void duk_cbor_encode(duk_context *ctx, duk_idx_t idx, duk_uint_t encode_flags);
```

### スタック

| ... | val | ... | -> | ... | cbor_val | ... |

### 要約

インプレース操作で任意の値をCBOR表現にエンコードします。結果として得られる値は、（今のところ）ArrayBufferです。現時点ではフラグは定義されていないので、フラグには0を渡します。

ECMAScript の値から CBOR へのマッピングは実験的なもので、エンコーディングの結果は時間の経過とともに変化する可能性があります。例えば、ECMAScript の値のエンコードとデコードのラウンドトリッピングを改善するために、カスタム CBOR タグが追加される予定です。また、結果の型（ArrayBuffer）も後で変更されるかもしれません。

### 例

```c
unsigned char *buf;
duk_size_t len;

duk_push_object(ctx);
duk_push_int(ctx, 42);
duk_put_prop_string(ctx, -2, "meaningOfLife");
duk_cbor_encode(ctx, -1, 0);
buf = duk_require_buffer_data(ctx, -1, &len);
/* CBOR data is in buf[0] ... buf[len - 1]. */
duk_pop(ctx);
```

### 参照

duk_cbor_decode


## duk_char_code_at() 

1.0.0 string

### プロトタイプ

```c
duk_codepoint_t duk_char_code_at(duk_context *ctx, duk_idx_t idx, duk_size_t char_offset);
```

### スタック

| ... | str | ... |

### 要約

文字列の文字オフセット char_offset にある文字のコードポイントを idx で取得します。idxの値が文字列でない場合は、エラーがスローされます。文字列が（拡張）UTF-8デコードできない場合、結果はU+FFFD（Unicode置換文字）です。char_offset が無効な場合（文字列の外側）には、0 が返されます。


### 例

```c
printf("char code at char index 12: %ld\n", (long) duk_char_code_at(ctx, -3, 12));
```



## duk_check_stack() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_check_stack(duk_context *ctx, duk_idx_t extra);
```

### スタック

(バリュースタックに影響なし。)


### 要約

バリュースタックに、呼び出し元が使用するために、現在のスタックの先頭から相対的に、少なくとも余分な予約（割り当て）要素があることを確認します。成功すれば1を、そうでなければ0を返す。呼び出しが成功した場合、呼び出し元は、バリュースタック関連のエラーなしに、追加の要素をバリュースタックにプッシュできることが保証される（メモリ不足などの他のエラーはまだ発生する可能性がある）。呼び出し元は、余分な値をプッシュできることに依存してはならない（MUST NOT）。

Duktape/C関数に入る時、そして呼び出しの外では、バリュースタックの呼び出し引数に加え、呼び出し側のために（DUK_API_ENTRY_STACK要素の）自動的な予備が確保されています。より多くのバリュースタックスペースが必要な場合、呼び出し側は関数の最初（例えば、必要な要素数が既知であるか、引数に基づいて計算できる場合）または動的（例えば、ループ内）に明示的に多くのスペースを予約しなければなりません。現在割り当てられているバリュースタックを越えて値をプッシュしようとするとエラーになることに注意してください．これは，内部実装を簡略化するためである．

Duktapeは、ユーザー予約要素に加えて、全てのAPIコールが更なる割り当てをせずに動作するのに十分なバリュースタック空間を確保するために、自動的に内部バリュースタック予備を保持します。また、メモリ再割り当ての動作を最小限に抑えるため、バリュースタックはある程度大きなステップで拡張されます。その結果、呼び出し元が指定した余分な値を超えて利用可能なバリュースタック要素の内部数は、かなり変化します。呼び出し元はこれを考慮する必要はなく、利用可能な追加要素に依存するべきでは決してない。

一般的なルールとして、より多くのスタックスペースを確保するために、この関数の代わりに duk_require_stack() が使用されるべきです。バリュースタックを拡張できない場合、エラーを投げて巻き戻す以外に、有用な回復方法はほとんどありません。


### 例

```c
duk_idx_t nargs = duk_get_top(ctx);  /* number or arguments */

/* reserve space for one temporary for each input argument */
if (!duk_check_stack(ctx, nargs)) {
    /* return 'undefined' if cannot allocate space */
    printf("failed to reserve enough stack space\n");
    return 0;
}

/* ... */
```

### 参照

duk_require_stack


## duk_check_stack_top() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_check_stack_top(duk_context *ctx, duk_idx_t top);
```

### スタック

(バリュースタックに影響なし。)


### 要約

duk_check_stack() と同様であるが、呼び出し側が使用するために、 トップまでのスペースがあることを保証します。これは、(現在のスタックの先頭からの相対的な) 追加要素の数を指定して予約するよりも便利な場合があります。

呼び出し側は、呼び出し側の使用のために確保された最大のスタックトップを指定し、呼び出し側のために予約された最高値のスタックインデックスを指定しない。例えば、top が 500 の場合、呼び出し元のために予約された最高値のスタックインデックス は 499 です。

一般論として、より多くのスタックスペースを確保するために、この関数の代わりに duk_require_stack_top() を使用するべきです。バリュースタックを拡張できない場合、エラーを投げて巻き戻す以外に有用な回復策はほとんどない。


### 例

```c
if (duk_check_stack_top(ctx, 100)) {
    printf("value stack guaranteed up to index 99\n");
} else {
    printf("cannot get value stack space\n");
}
```

### 参照

duk_require_stack_top


## duk_check_type() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_check_type(duk_context *ctx, duk_idx_t idx, duk_int_t type);
```

### スタック

| ... | val | ... |

### 要約

idxにある値の型とtypeをマッチさせます。型が一致する場合は1を，そうでない場合は0を返す。


### 例

```c
if (duk_check_type(ctx, -3, DUK_TYPE_NUMBER)) {
    printf("value is a number\n");
}
```


## duk_check_type_mask() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_check_type_mask(duk_context *ctx, duk_idx_t idx, duk_uint_t mask);
```

### スタック

| ... | val | ... |

### 要約

idxにある値の型とmaskで与えられた型マスクビットをマッチさせます。値の型が型マスクビットの1つに一致する場合は1を，そうでない場合は0を返す。


### 例

```c
if (duk_check_type_mask(ctx, -3, DUK_TYPE_MASK_STRING |
                                 DUK_TYPE_MASK_NUMBER)) {
    printf("value is a string or a number\n");
}
```


## duk_compact() 

1.0.0 property object memory

### プロトタイプ

```c
void duk_compact(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

(バリュースタックに影響なし。)


### 要約

オブジェクトの内部メモリ割り当てをリサイズし、メモリ使用量を最小にします。obj_idx の値がオブジェクトでない場合は、何もしない。コンパクションは通常安全であるが、内部エラー（メモリ不足エラーなど）により失敗することがあります。コンパクションは、オブジェクトの「自身のプロパティ」にのみ影響し、継承されたプロパティには影響しません。

コンパクションは最終的な操作ではなく、オブジェクトのセマンティクスに影響を与えることはありません。オブジェクトに新しいプロパティを追加することは可能ですが（オブジェクトが拡張可能であることが前提）、オブジェクトのサイズ変更が発生します。既存のプロパティ値は、オブジェクトの内部メモリ割り当てに影響を与えることなく変更することができます（プロパティが書き込み可能であると仮定して）。オブジェクトは複数回コンパクト化できます。例えば、以前にコンパクト化したオブジェクトに新しいプロパティを追加する場合、プロパティ追加後のメモリフットプリントを最小にするために、オブジェクトを再度コンパクト化することができます。

オブジェクトをコンパクトにすると、オブジェクトごとにわずかな量のメモリを節約することができます。一般的には、(1) メモリフットプリントが非常に重要な場合、(2) オブジェクトに新しいプロパティが追加される可能性が低い場合、(3) オブジェクトが比較的長寿命の場合、そして (4) 大きな違いを生むほど多くのオブジェクトに圧縮を適用できる場合に有効です。

Object.seal()、Object.freeze()、Object.preventExtensions()が呼ばれた場合、オブジェクトは自動的に圧縮されます。


### 例

```c
duk_push_object(ctx);                           /* [ ... obj ] */
duk_push_int(ctx, 42);                          /* [ ... obj 42 ] */
duk_put_prop_string(ctx, -2, "meaningOfLife");  /* [ ... obj ] */
duk_compact(ctx, -1);                           /* [ ... obj ] */
```



## duk_compile() 

1.0.0 compile

### プロトタイプ

```c
void duk_compile(duk_context *ctx, duk_uint_t flags);
```

### スタック

| ... | source | filename -> | ... | function |

### 要約

ECMAScript のソースコードをコンパイルし、コンパイルされた関数オブジェクトに置き換えます（コードは実行されません）。filename 引数は結果の関数の fileName プロパティとして保存され、例えばトレースバックで関数を識別するために使用される名前となります。コンパイル時のエラー（メモリ不足、内部制限エラーなどの通常の内部エラーに加えて）に対しては、SyntaxError を投げます。ことがあります。

以下のフラグを指定することができます。

- DUK_COMPILE_EVAL 入力を ECMAScript プログラムとしてではなく、eval コードとしてコンパイルします。
- DUK_COMPILE_FUNCTION 入力を ECMAScript プログラムとしてではなく、関数としてコンパイルします。
- DUK_COMPILE_STRICT 入力を強制的にストリクトモードでコンパイルする
- DUK_COMPILE_SHEBANG 入力の最初の行で、非標準の shebang コメント (#! ...) を許可する

コンパイルされるソースコードは、以下のようなものであってもよい。

- グローバルコード: 引数ゼロの関数にコンパイルされ、トップレベルのECMAScriptプログラムのように実行されます。
- 評価コード：ECMAScript の eval コールのように実行される、引数ゼロの関数にコンパイルされます (DUK_COMPILE_EVAL フラグ)
- 関数コード：0 個以上の引数を持つ関数にコンパイルされます (フラグ DUK_COMPILE_FUNCTION)

これらは全て、ECMAScript では若干異なるセマンティクスを持っています。詳細な議論については、実行コンテキストの確立を参照してください。一つの大きな違いは、グローバルコンテキストと eval コンテキストには暗黙の戻り値があることです：最後の空でない文の値がプログラムや eval コードの自動的な戻り値であるのに対し、関数には自動的な戻り値がないのです。

グローバルコードと eval コードには、明示的な関数構文がありません。例えば、次のようなものはグローバル式としても eval 式としてもコンパイルできます。

```javascript
print("Hello world!");
123;  // implicit return value
```

関数コードはECMAScriptの関数構文に従う（関数名は省略可能）。

```javascript
function adder(x,y) {
    return x+y;
}
```

関数をコンパイルすることは，関数式を含む eval コードをコンパイルすることと同じです．そうしないと、evalコードは関数値を返す代わりに、adderという名前のグローバル関数を登録することになります。

```javascript
(function adder(x,y) {
    return x+y;
})
```

グローバルコードと eval コードで生成されるバイトコードは、現在、関数で生成されるバイトコードよりも遅いです。プログラムコードと eval コードでは、すべての変数アクセスに「スローパス」が使用され、プログラムコードと eval コードの暗黙の戻り値処理により、不要なバイトコードがいくつか生成されます。性能の観点（メモリ性能と実行性能）から言えば、できるだけ多くのコードを関数の中に置くことが望ましい。

eval やグローバル式をコンパイルする際には、ECMAScript の通常のゴッチャを避けるように注意してください。

```javascript
/* Function at top level is a function declaration which registers a global
 * function, and is different from a function expression.  Use parentheses
 * around the top level expression.
 */

eval("function adder(x,y) { return x+y; }");   /* registers 'adder' to global */
eval("function (x,y) { return x+y; }");        /* not allowed */
eval("(function (x,y) { return x+y; })");      /* function expression (anonymous) */
eval("(function adder(x,y) { return x+y; })"); /* function expression (named) */

/* Opening curly brace at top level is interpreted as start of a block
 * expression, not an object literal.  Use parentheses around the top
 * level expression.
 */

eval("{ myfunc: 1 }");   /* block with -label- "myfunc" and statement "1" (!) */
eval("({ myfunc: 1 })";  /* object literal { myfunc: 1 } */
```

### 例

```c
/* Global code.  Note that the hello() function is a function
 * declaration which gets registered into the global object when
 * executed.  Implicit return value is 123.
 */

duk_push_string(ctx, "print('global');\n"
                     "function hello() { print('Hello world!'); }\n"
                     "123;");
duk_push_string(ctx, "hello");
duk_compile(ctx, 0);   /* [ source filename ] -> [ func ] */
duk_call(ctx, 0);      /* [ func ] -> [ result ] */
printf("program result: %lf\n", (double) duk_get_number(ctx, -1));
duk_pop(ctx);

/* Eval code */

duk_push_string(ctx, "2+3");
duk_push_string(ctx, "eval");
duk_compile(ctx, DUK_COMPILE_EVAL);
duk_call(ctx, 0);      /* [ func ] -> [ result ] */
printf("eval result: %lf\n", (double) duk_get_number(ctx, -1));
duk_pop(ctx);

/* Function code */

duk_push_string(ctx, "function (x,y) { return x+y; }");
duk_push_string(ctx, "function");
duk_compile(ctx, DUK_COMPILE_FUNCTION);
duk_push_int(ctx, 5);
duk_push_int(ctx, 6);
duk_call(ctx, 2);      /* [ func 5 6 ] -> [ result ] */
printf("function result: %lf\n", (double) duk_get_number(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_pcompile
duk_compile_string
duk_compile_string_filename
duk_compile_lstring
duk_compile_lstring_filename


## duk_compile_lstring() 

1.0.0 string compile

### プロトタイプ

```c
void duk_compile_lstring(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

| ... | -> | ... | function |

### 要約

duk_compile() と同様であるが、コンパイル入力は長さを明示した C 文字列として与えられます。この関数に関連するファイル名は "input" です。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では便利です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_compile_lstring(ctx, 0, src, len);
```


## duk_compile_lstring_filename() 

1.0.0 string compile

### プロトタイプ

```c
void duk_compile_lstring_filename(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

| ... | filename -> | ... | function |

### 要約

duk_compile() と同様ですが、コンパイル入力は長さを明示した C 文字列として与えられます。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では便利です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_push_string(ctx, "myFile.js");
duk_compile_lstring_filename(ctx, 0, src, len);
```


## duk_compile_string() 

1.0.0 string compile

### プロトタイプ

```c
void duk_compile_string(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

| ... | -> | ... | function |

### 要約

duk_compile() と同様であるが、コンパイル入力は C の文字列として与えられますこの関数に関連するファイル名は "input" です。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では便利です。

### 例

```c
duk_compile_string(ctx, 0, "print('program code');");
```


## duk_compile_string_filename() 

1.0.0 string compile

### プロトタイプ

```c
void duk_compile_string_filename(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

| ... | filename -> | ... | function |

### 要約

duk_compile() と同様であるが、コンパイル入力は C の文字列として与えられます。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
duk_push_string(ctx, "myFile.js");
duk_compile_string_filename(ctx, 0, "print('program code');");
```


## duk_components_to_time() 

2.0.0 time

### プロトタイプ

```c
duk_double_t duk_components_to_time(duk_context *ctx, duk_time_components *comp);
```

### スタック

(バリュースタックに影響なし。)


### 要約

UTCで解釈される構成要素（年、月、日、など）を時間値に変換します。comp->weekday 引数は変換の際に無視されます。構成要素の値が無効な場合、エラーが投げられます。

ECMAScriptのDate.UTC()組み込みと若干の違いがあります。

2桁の年号を特別に扱うことはありません。例えば、Date.UTC(99, 0, 1)は1999-01-01として解釈されます。comp->time が 99 の場合、99 年と解釈されます。
ミリ秒成分は分数（サブミリ秒の分解能）が許され、結果として得られる時刻の値に分数を持たせることができます。
ECMAScript のプリミティブと同様に、成分はその自然な範囲を超えることができ、正規化されます。例えば、comp->minutesを120と指定すると、時間値に2時間追加されると解釈されます。成分はIEEE doublesで表現され、大きな値や負の値を使用できるようにします。


### 例

```c
duk_time_components comp;
duk_double_t time;

memset((void *) &comp, 0, sizeof(comp));  /* good practice even if fully initialized */
comp.year = 2016;
comp.month = 0;  /* 0-based, 1=January */
comp.day = 2;    /* 1-based: second of January */
comp.hours = 3;
comp.minutes = 4;
comp.seconds = 5;
comp.milliseconds = 6.0;  /* allows sub-millisecond resolution */
comp.weekday = 0;  /* ignored */

time = duk_components_to_time(ctx, &comp);
printf("2016-01-02 03:04:05.006Z -> %lf\n", (double) time);
```

### 参照

duk_time_to_components


## duk_concat() 

1.0.0 string

### プロトタイプ

```c
void duk_concat(duk_context *ctx, duk_idx_t count);
```

### スタック

| ... | val1 | ... | valN -> | ... | result |

### 要約

ゼロ個以上の値を結果の文字列に連結します。入力値は、ToString()で自動的に強制されます。

このプリミティブは、文字列の中間的な操作の回数を最小限に抑え、手動で文字列を連結するよりも優れています。

Array.prototype.concat() とは異なり、この API コールは配列引数の平坦化や Symbol.isConcatSpreadable のサポートは行いません。

### 例

```c
duk_push_string(ctx, "foo");
duk_push_int(ctx, 123);
duk_push_true(ctx);
duk_concat(ctx, 3);

printf("result: %s\n", duk_get_string(ctx, -1));  /* "foo123true" */
duk_pop(ctx);
```

### 参照

duk_join


## duk_config_buffer() 

1.3.0 stack buffer

### プロトタイプ

```c
void duk_config_buffer(duk_context *ctx, duk_idx_t idx, void *ptr, duk_size_t len);
```

### スタック

| ... | buf | ... | -> | ... | buf | ... |

### 要約

外部バッファポインタと外部バッファ値の長さを設定します。


### 例

```c
/* Create an external buffer and point it to my_buffer containing
 * my_length bytes.
 */
duk_push_external_buffer(ctx);
duk_config_buffer(ctx, -1, my_buffer, my_length);
```

### 参照

duk_push_external_buffer


## duk_copy() 

1.0.0 stack

### プロトタイプ

```c
void duk_copy(duk_context *ctx, duk_idx_t from_idx, duk_idx_t to_idx);
```

### スタック

| ... | old(to_idx) | ... | val(from_idx) | ... | -> | ... | val(to_idx) | ... | val(from_idx) | ... |

### 要約

from_idxからto_idxに値をコピーし、前の値を上書きします。どちらかのインデックスが無効な場合、エラーを投げます。。

これは，以下の略記法です。

```c
/* to_index must be normalized in case it is negative and would change its
 * meaning after duk_dup().
 */
to_idx = duk_normalize_idx(ctx, to_idx);
duk_dup(ctx, from_idx);
duk_replace(ctx, to_idx);
```

### 例

```c
duk_copy(ctx, -3, 1);
```

### 参照

duk_insert
duk_replace


## duk_create_heap() 

1.0.0 heap

### プロトタイプ

```c
duk_context *duk_create_heap(duk_alloc_function alloc_func,
                             duk_realloc_function realloc_func,
                             duk_free_function free_func,
                             void *heap_udata,
                             duk_fatal_function fatal_handler);
```

### スタック

(バリュースタックに影響なし。)


### 要約

新しいDuktapeヒープを作成し、初期コンテキスト(スレッド)を返す。ヒープの初期化に失敗した場合、NULLが返されます。現在のところ、より詳細なエラー情報を得る方法はない。

呼び出し側は、alloc_func、realloc_func、free_func でカスタムメモリ管理関数を提供することができます。ポインタはすべてNULLでなければなりませんし、すべて非NULLでなければなりません。ポインタが NULL の場合，デフォルトのメモリ管理関数 (ANSI C の malloc(), realloc() および free()) が使用される．メモリ管理関数は、同じ不透明なユーザデータポインタである heap_udata を共有します。このユーザーデータポインターは、致命的なエラー処理や低メモリーポインター圧縮マクロなど、他のDuktape機能にも使用されています。

致命的なエラー・ハンドラはfatal_handlerで提供されます。このハンドラは、キャッチできないエラー、ガベージコレクションで解決できないメモリ不足のエラー、セルフテストエラーなどの回復不能なエラー状況で呼び出されます。呼び出し側は、ほとんどのアプリケーションで致命的なエラーハンドラを実装すべきです(SHOULD)。もし指定がなければ、Duktapeに組み込まれたデフォルトの致命的なエラーハンドラが代わりに使われます。デフォルトの致命的エラーハンドラは、(duk_config.h で上書きされない限り) abort() を呼び出し、stdout や stderr にエラーメッセージを表示しない。より詳細な情報と例については、致命的なエラーの処理方法を参照してください。

デフォルトの設定で Duktape ヒープを作成するには、 duk_create_heap_default() を使用して下さい。

同じヒープにリンクされた新しいコンテキストは、 duk_push_thread() と duk_push_thread_new_globalenv() で作成できます。


### 例

```c
/*
 *  Simple case: use default allocation functions and fatal error handler
 */

duk_context *ctx = duk_create_heap(NULL, NULL, NULL, NULL, NULL);
if (ctx) {
    /* success */
} else {
    /* error */
}

/*
 *  Customized handlers
 */

duk_context *ctx = duk_create_heap(my_alloc, my_realloc, my_free,
                                   (void *) 0xdeadbeef, my_fatal);
if (ctx) {
    /* success */

    /* ... after heap is no longer needed: */
    duk_destroy_heap(ctx);
} else {
    /* error */
}
```

### 参照

duk_create_heap_default
duk_destroy_heap


## duk_create_heap_default() 

1.0.0 heap

### プロトタイプ

```c
duk_context *duk_create_heap_default(void);
```

### スタック

(バリュースタックに影響なし。)


### 要約

新しいDuktapeヒープを作成し、初期コンテキスト(スレッド)を返す。ヒープの初期化に失敗した場合は、NULLを返します。現在のところ、より詳細なエラー情報を得る方法はない。

作成されたヒープには、デフォルトのメモリ管理関数と致命的なエラーハンドラ関数が使用されます。このAPIコールは、以下と同等です。

```c
ctx = duk_create_heap(NULL, NULL, NULL, NULL, NULL);
```

### 例

```c
duk_context *ctx = duk_create_heap_default();
if (ctx) {
    /* success */

    /* ... after heap is no longer needed: */
    duk_destroy_heap(ctx);
} else {
    /* error */
}
```


## duk_debugger_attach() 

1.2.0 debugger

### プロトタイプ

```c
void duk_debugger_attach(duk_context *ctx,
                         duk_debug_read_function read_cb,
                         duk_debug_write_function write_cb,
                         duk_debug_peek_function peek_cb,
                         duk_debug_read_flush_function read_flush_cb,
                         duk_debug_write_flush_function write_flush_cb,
                         duk_debug_request_function request_cb,
                         duk_debug_detached_function detached_cb,
                         void *udata);
```

### スタック

(バリュースタックに影響なし。)


### 要約

Duktapeのヒープにデバッガーをアタッチします。アタッチが完了すると、デバッガは自動的にポーズ状態になります。デバッガのサポートがDuktapeにコンパイルされていない場合、エラーを投げます。。デバッガー統合の詳細については、debugger.rstを参照してください。コールバックは以下のとおりです。オプションのコールバックはNULLでもかまいません。

コールバック 必須 説明
read_cb はい デバッグトランスポート読み込みコールバック
write_cb はい デバッグトランスポートの書き込みコールバック
peek_cb No デバッグトランスポートのピークコールバック、強く推奨するがオプション
read_flush_cb No デバッグトランスポートリードフラッシュコールバック
write_flush_cb No デバッグトランスポートライトフラッシュコールバック
request_cb No アプリケーション固有のコマンド (AppRequest) コールバック、NULL は AppRequest のサポートがないことを示します。
detached_cb No デバッガーを切り離すコールバック
重要：コールバックは、以下を除き、Duktape API関数を呼び出すことはできません（そうすることで、メモリが安全でない動作が発生する可能性があります）。

request_cb AppRequestコールバックは、debugger.rstに記載されているガイドラインの範囲内で、バリュースタックを使用することができます。
Duktape 1.4.0以降では、デバッガー離脱コールバックはduk_debugger_attach()を呼び出して、すぐにデバッガーを再接続することが許可されるようになりました。Duktape 1.4.0以前では、即時再接続は潜在的にいくつかの問題を引き起こしていました。
Duk_debugger_attach_custom() と duk_debugger_attach() APIコールは、Duktape 2.xでマージされました。

### 例

```c
duk_debugger_attach(ctx,
                    my_read_cb,
                    my_write_cb,
                    my_peek_cb,
                    NULL,
                    NULL,
                    NULL,
                    my_detached_cb,
                    my_udata);
```

### 参照

duk_debugger_detach
duk_debugger_cooperate
duk_debugger_notify


## duk_debugger_cooperate() 

1.2.0 debugger

### プロトタイプ

```c
void duk_debugger_cooperate(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし。)


### 要約

受信したデバッグコマンドをブロックせずにチェックし、実行します。デバッグ・コマンドは、引数ctxのコンテキストで実行されます。Duktapeへの呼び出しが（どのコンテキストからも）進行中でない場合にのみ呼び出すことができます。詳細については、debugger.rst を参照してください。


### 例

```c
duk_debugger_cooperate(ctx);
```

### 参照

duk_debugger_attach
duk_debugger_detach


## duk_debugger_detach() 

1.2.0 debugger

### プロトタイプ

```c
void duk_debugger_detach(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし。)


### 要約

Duktapeヒープからデバッガを切り離す。デバッガ・サポートがDuktapeにコンパイルされていない場合、エラーを投げます。。詳細は debugger.rst を参照のこと。


### 例

```c
duk_debugger_detach(ctx);
```

### 参照

duk_debugger_attach
duk_debugger_cooperate


## duk_debugger_notify() 

1.5.0 debugger

### プロトタイプ

```c
duk_bool_t duk_debugger_notify(duk_context *ctx, duk_idx_t nvalues);
```

### スタック

| ... | val1 | ... | valN | -> | ... |

### 要約

デバッグプロトコルのdvaluesにマッピングされたバリュースタックトップのnvalues値を含むアプリケーション固有のデバッガ通知（AppNotify）を送信します。戻り値は、notifyが正常に送信されたか(0以外)、送信されなかったか(0)を示します。引数nvaluesは常にスタックトップからポップオフされます。デバッガサポートがコンパイルされていない場合、またはデバッガが接続されていない場合、このコールはno-opである; いずれの場合も、コールはnotifyが送信されなかったことを示す0を返す。

アプリケーション固有の通知の使用方法に関する詳細や例については、デバッガのドキュメントを参照してください。


### 例

```c
/* Causes the following notify to be sent over the debugger protocol:
 *
 *   NFY AppNotify "BatteryStatus" 740 1000 true EOM
 */
int battery_current = 740;
int battery_limit = 1000;
int battery_charging = 1;

duk_push_string(ctx, "BatteryStatus");
duk_push_int(ctx, battery_current);
duk_push_int(ctx, battery_limit);
duk_push_boolean(ctx, battery_charging);
if (duk_debugger_notify(ctx, 4 /*nvalues*/)) {
    printf("battery status notification sent\n");
} else {
    printf("battery status notification not sent\n");
}
```


## duk_debugger_pause() 

1.5.0 debugger

### プロトタイプ

```c
void duk_debugger_pause(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし。)


### 要約

デバッガの一時停止をできるだけ早く要求し、ブロックせずに返します。一時停止は ECMAScript バイトコードが次に実行されたときにトリガーされ、通常はほぼ即座に実行されます。しかし、Duktape/C 関数のようなネイティブコールが進行中であったり、ECMAScript コードが現在実行されていない場合、一時停止が有効になるまでに時間がかかることがあります。

この呼び出しは、（1）デバッガ・サポートがコンパイルされていない、（2）デバッガが接続されていない、（3）Duktapeがすでに一時停止している、などの場合には実行できません。これは、ECMAScriptのデバッガ・ステートメントのセマンティクスを模倣しています。

他のDuktape APIコールと同様、このコールはスレッドセーフではありません。そのコンテキストのECMAScriptコードを実行しているスレッドとは別のスレッドから一時停止をトリガーしたくなるかもしれませんが、これは安全ではなく、現在サポートされていません。

### 例

```c
/* In your event loop: */
if (key_pressed == KEY_F12) {
    duk_debugger_pause(ctx);
}
```


## duk_decode_string() 

1.0.0 string

### プロトタイプ

```c
void duk_decode_string(duk_context *ctx, duk_idx_t idx, duk_decode_char_function callback, void *udata);
```

### スタック

| ... | val | ... |

### 要約

idxの文字列を処理し、文字列の各コードポイントに対してコールバックを呼び出す。コールバックには、引数udataとコードポイントが渡されます。入力文字列は変更されない。値が文字列でない場合、またはインデックスが無効な場合は、エラーを投げます。。


### 例

```c
static void decode_char(void *udata, duk_codepoint_t codepoint) {
    printf("codepoint: %ld\n", (long) codepoint);
}

duk_push_string(ctx, "test_string");
duk_decode_string(ctx, -1, decode_char, NULL);
```

### 参照

duk_map_string


## duk_def_prop() 

1.1.0 sandbox property

### プロトタイプ

```c
void duk_def_prop(duk_context *ctx, duk_idx_t obj_idx, duk_uint_t flags);
```

### スタック

| ... | obj | ... | key | -> | ... | obj | ... | (if have no value, setter, getter) |
| ... | obj | ... | key | value | -> | ... | obj | ... | (if have value) |
| ... | obj | ... | key | getter | -> | ... | obj | ... | (if have getter, but no setter) |
| ... | obj | ... | key | setter | -> | ... | obj | ... | (if have setter, but no getter) |
| ... | obj | ... | key | getter | setter | -> | ... | obj | ... | (if have both getter and setter) |

### 要約

Object.defineProperty() のようなセマンティクスで、 obj_idx にあるオブジェクトの property key の属性を作成または変更します。要求された変更が許可されない場合（例えば、プロパティが設定できない場合）、TypeErrorがスローされます。ターゲットがオブジェクトでない場合（またはインデックスが無効な場合）、エラーがスローされます。

flags フィールドは、どのプロパティの属性が変更されるかを示す "have" フラグを提供し、これは Object.defineProperty() で許可される部分プロパティ記述子をモデル化したものです。書き込み可能、設定可能、および列挙可能な属性の値は flags フィールドで指定し、プロパティ値、ゲッター、およびセッターはバリュースタック引数として指定します。新しいプロパティを作成するとき、属性値が見つからないと、ECMAScript のデフォルトが使用されます（すべての boolean 属性に対して false、value、getter、setter に対して undefined）；既存のプロパティを修正するとき、属性値が見つからないと、既存の属性値が変更されないことを意味します。

具体的な例として

```javascript
// Set value, set writable, clear enumerable, leave configurable untouched.
Object.defineProperty(obj, { value: 123, writable: true, enumerable: false });
```

このようにマッピングします。

```c
duk_push_uint(ctx, 123);  /* value is taken from stack */
duk_def_prop(ctx, obj_idx,
             DUK_DEFPROP_HAVE_VALUE |  /* <=> value: 123 */
             DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_WRITABLE |  /* <=> writable: true */
             DUK_DEFPROP_HAVE_ENUMERABLE);  /* <=> enumerable: false */
```

DUK_DEFPROP_FORCE フラグを使用すると、拡張不可能なターゲットオブジェクトや設定不可能なプロパティのために操作が通常失敗する場合でも、プロパティの変更を強制することができます。これは ECMAScript コードから Object.defineProperty() で行うことはできず、例えばサンドボックスのセットアップに有用です。場合によっては、強制的な変更さえも不可能で、エラーが投げられることになります。例えば、内部で仮想プロパティとして実装されているプロパティは、変更不可能であったり（string .length や index プロパティなど）、制限がある場合があります（array .length プロパティは内部制限により設定や列挙ができないなど）。

以下の基本フラグが定義されています。

定義 説明
DUK_DEFPROP_WRITABLE DUK_DEFPROP_HAVE_WRITABLE が設定されている場合のみ有効です。
DUK_DEFPROP_ENUMERABLE DUK_DEFPROP_HAVE_ENUMERABLEが設定されている場合のみ有効です。
DUK_DEFPROP_CONFIGURABLE 設定可能な属性を設定します。DUK_DEFPROP_HAVE_CONFIGURABLE が設定されている場合のみ有効です。
DUK_DEFPROP_HAVE_WRITABLE 書き込み可能な属性を設定または解除します （未設定の場合は変更されません）。
DUK_DEFPROP_HAVE_ENUMERABLE 列挙可能な属性を設定または解除します (未設定の場合は変更されません)。
DUK_DEFPROP_HAVE_CONFIGURABLE コンフィギュラブル属性を設定または解除します (未設定の場合、変更されません)。
DUK_DEFPROP_HAVE_VALUE 値属性を設定します。値はバリュースタックで指定されます。
DUK_DEFPROP_HAVE_GETTER ゲッター属性を設定し、値はバリュースタックに保存されます。
DUK_DEFPROP_HAVE_SETTER セッター属性の設定、値はバリュースタックで指定されます。
DUK_DEFPROP_FORCE 属性が設定不可能な場合でも、可能であれば属性を強制的に変更します。

また、一般的なフラグの組み合わせに対応する便利な定義もあります。例えば、DUK_DEFPROP_SET_WRITABLE は DUK_DEFPROP_HAVE_WRITABLE と DUK_DEFPROP_WRITABLE に相当します。現在、以下のものが定義されています。

定義 説明
DUK_DEFPROP_{SET,CLEAR}_WRITABLE 'writable' 属性を設定またはクリアします。
DUK_DEFPROP_{SET,CLEAR}_ENUMERABLE 'enumerable' 属性を設定または解除します。
DUK_DEFPROP_{SET,CLEAR}_CONFIGURABLE '設定可能' 属性を設定または解除します。
DUK_DEFPROP_SET_{W,E,C,WE,WC,EC,WEC} 1つまたは複数の属性を設定し、他の属性には触れないようにします。
DUK_DEFPROP_CLEAR_{W,E,C,WE,WC,EC,WEC} 1つまたは複数の属性をクリアします。
DUK_DEFPROP_ATTR_{W,E,C,WE,WC,EC,WEC} 書き込み可能、 列挙可能、 設定可能な属性を設定ま たは ク リ ア し ます。例えば、DUK_DEFPROP_ATTR_WCはwritableとconfigurableを設定し、enumerableをクリアします。
DUK_DEFPROP_HAVE_{W,E,C,WE,WC,EC,WEC} 1つ以上の属性が設定／クリアされることを示します。例えば、DUK_DEFPROP_HAVE_WC は DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_HAVE_CONFIGURABLE と同等です。
DUK_DEFPROP_{W,E,C,WE,WC,EC,WEC} 属性値を与えます（一致する "have" フラグが設定されている場合に有効）。例えば DUK_DEFPROP_WE は DUK_DEFPROP_WRITABLE | DUK_DEFPROP_ENUMERABLE と同じ意味です。

いくつかの例（以下にもっと例を示します）。

値を設定し、書き込み可能を設定し、列挙可能をクリアし、設定可能を設定するには、値をバリュースタックにプッシュして、フラグを設定します。
基本フラグ。DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_WRITABLE | DUK_DEFPROP_HAVE_ENUMERABLE | DUK_DEFPROP_HAVE_CONFIGURABLE; or DUK_DEFPROP_CONFIGURABLE。
便利です。DUK_defprop_have_value | DUK_defprop_attr_WC.
書き込み可能な属性を消去する（他の属性はそのままにしておく）には、 flags を設定します。
基本フラグ。DUK_DEFPROP_HAVE_WRITABLE; または
便利です。DUK_DEFPROP_CLEAR_WRITABLE、または
便宜上DUK_DEFPROP_CLEAR_W.
値を設定し、書き込み可能を消去し、列挙可能を設定するには（他の属性はそのままで）、値をバリュースタックにプッシュし、フラグを設定します。
基本フラグ。DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_HAVE_ENUMERABLE | DUK_DEFPROP_ENUMERABLE; または
便利です。DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_CLEAR_W | DUK_DEFPROP_SET_E、または
便利です。DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_HAVE_WE | DUK_DEFPROP_E.

このAPIコールはいろいろと便利です。

Cコードから直接、非デフォルトの属性を持つプロパティを作成します。
Cコードから直接、アクセッサ（ゲッター／セッター）プロパティを作成します。
既存のプロパティの属性を C コードから直接変更します。
より多くの例については、APIテストケースtest-def-prop.cを参照してください。

ターゲットがdefinePropertyトラップを実装しているProxyオブジェクトの場合、トラップが起動するはずです。しかし、Duktapeは現在definePropertyトラップをサポートしておらず、defineProperty()オペレーションは現在ターゲットに転送されません。サポートが追加された場合、このAPIコールはトラップを呼び出したり、操作をターゲット・オブジェクトに転送したりするようになります。

### 例

```c
duk_idx_t obj_idx = /* ... */;

/* Create an ordinary property which is writable and configurable, but
 * not enumerable.  Equivalent to:
 *
 *   Object.defineProperty(obj, 'my_prop_1', {
 *      value: 123, writable: true, enumerable: false, configurable: true
 *   });
 */

duk_push_string(ctx, "my_prop_1");
duk_push_int(ctx, 123);  /* prop value */
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_VALUE |
             DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_WRITABLE |
             DUK_DEFPROP_HAVE_ENUMERABLE | 0 |
             DUK_DEFPROP_HAVE_CONFIGURABLE | DUK_DEFPROP_CONFIGURABLE);

/* The same but more readable using convenience flags. */

duk_push_string(ctx, "my_prop_1");
duk_push_int(ctx, 123);  /* prop value */
duk_def_prop(ctx, obj_idx, DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_ATTR_WC);

/* Change the property value and make it non-writable.  Don't touch other
 * attributes.  Equivalent to:
 *
 *   Object.defineProperty(obj, 'my_prop_1', {
 *      value: 321, writable: false
 *   });
 */

duk_push_string(ctx, "my_prop_1");
duk_push_int(ctx, 321);
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_VALUE |
             DUK_DEFPROP_HAVE_WRITABLE | 0);

/* The same but more readable using convenience flags. */

duk_push_string(ctx, "my_prop_1");
duk_push_int(ctx, 321);
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_CLEAR_WRITABLE);

/* Make the property non-configurable, don't touch value or other attributes.
 * Equivalent to:
 *
 *   Object.defineProperty(obj, 'my_prop_1', {
 *      configurable: false
 *   });
 */

duk_push_string(ctx, "my_prop_1");
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_CONFIGURABLE | 0);

/* The same but more readable using convenience flags. */

duk_push_string(ctx, "my_prop_1");
duk_def_prop(ctx, obj_idx, DUK_DEFPROP_CLEAR_CONFIGURABLE);

/* Create an accessor property which is non-configurable and non-enumerable.
 * Attribute flags are not given so they default to ECMAScript defaults
 * (false) automatically.  Equivalent to:
 *
 *   object.defineproperty(obj, 'my_accessor_1', {
 *      get: my_getter, set: my_setter
 *   });
 */

duk_push_string(ctx, "my_accessor_1");
duk_push_c_function(ctx, my_getter, 0 /*nargs*/);
duk_push_c_function(ctx, my_setter, 1 /*nargs*/);
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_GETTER |
             DUK_DEFPROP_HAVE_SETTER);

/* Create an accessor property which is non-configurable but enumerable.
 * Attribute flags are given explicitly which is easier to read without
 * knowing about ECMAScript attribute default values.  Equivalent to:
 *
 *   Object.defineProperty(obj, 'my_accessor_2', {
 *      get: my_getter, set: my_setter, enumerable: true, configurable: false
 *   });
 */

duk_push_string(ctx, "my_accessor_2");
duk_push_c_function(ctx, my_getter, 0 /*nargs*/);
duk_push_c_function(ctx, my_setter, 1 /*nargs*/);
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_GETTER |
             DUK_DEFPROP_HAVE_SETTER |
             DUK_DEFPROP_HAVE_CONFIGURABLE |  /* clear */
             DUK_DEFPROP_HAVE_ENUMERABLE | DUK_DEFPROP_ENUMERABLE);  /* set */

/* The same but more readable with convenience flags. */

duk_push_string(ctx, "my_accessor_2");
duk_push_c_function(ctx, my_getter, 0 /*nargs*/);
duk_push_c_function(ctx, my_setter, 1 /*nargs*/);
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_GETTER |
             DUK_DEFPROP_HAVE_SETTER |
             DUK_DEFPROP_HAVE_CLEAR_CONFIGURABLE |
             DUK_DEFPROP_HAVE_SET_ENUMERABLE);

/* Change the value of a non-configurable property by force.
 * No ECMAScript equivalent.
 */

duk_push_string(ctx, "my_nonconfigurable_prop");
duk_push_value(ctx, 321);
duk_def_prop(ctx,
             obj_idx,
             DUK_DEFPROP_HAVE_VALUE |
             DUK_DEFPROP_FORCE);
```

### 参照

duk_get_prop_desc


## duk_del_prop() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_del_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

| ... | obj | ... | key | -> | ... | obj | ... |

### 要約

obj_idx にある値のプロパティ key を削除します。 key はスタックから削除されます。リターンコードとエラースローの動作。

property が存在し、かつ設定可能な場合（削除可能）、property を削除し、1 を返す。
property が存在し、設定可能でない場合、エラーを投げます。（strict mode semantics）。
property が存在しない場合、1 を返す（0 ではない）。
obj_idx の値がオブジェクト互換でない場合、エラーを投げます。。
obj_idx が無効な場合、エラーを投げます。。
プロパティの削除は、ECMAScript の式 res = delete obj[key] と等価です。正確な意味は、Property Accessors, The delete operator and [[Delete]] (P, Throw) を参照してください。戻り値とエラースローの動作は、ECMAScriptのdelete演算子の動作を反映しています。対象値とキーは共に強制されます。

ターゲット値は、自動的にオブジェクトに強制されます。しかし、このオブジェクトは一時的なものであり、そのプロパティを削除することはあまり有益ではありません。
key引数は内部的にToPropertyKey()強制を使って強制され、結果は文字列かSymbolになります。配列と数値インデックスには、明示的な文字列強制を避ける内部高速パスがあるので、該当する場合は数値キーを使用します。
ターゲットが deleteProperty トラップを実装する Proxy オブジェクトの場合、トラップが呼び出され、API 呼び出しの戻り値はトラップの戻り値に一致します。

このAPIコールは、ターゲットプロパティが存在しない場合、1を返します。これはあまり直感的ではありませんが、ECMAScript のセマンティクスに従っています: delete obj.nonexistent も true と評価されます。
もしキーが固定文字列であれば、1 つの API 呼び出しを回避して、 duk_del_prop_string() 変数を使用することができます。同様に、もしキーが配列のインデックスであれば、 duk_del_prop_index() 変数を使うことができます。

プロパティアクセスのベースとなる値は通常オブジェクトですが、技術的には任意の値でありえます。普通の文字列やバッファの値には仮想的なインデックスプロパティがあり、例えば "foo"[2] にアクセスすることができます。また、ほとんどのプリミティブな値は何らかのプロトタイプオブジェクトを継承しているので、例えば (12345).toString(16) のようにメソッドを呼び出すことができます。

### 例

```c
duk_bool_t rc;

duk_push_string(ctx, "myProperty");
rc = duk_del_prop(ctx, -3);
printf("delete obj.myProperty -> rc=%d\n", (int) rc);
```

### 参照

duk_del_prop_index
duk_del_prop_string
duk_del_prop_lstring
duk_del_prop_literal
duk_del_prop_heapptr


## duk_del_prop_heapptr() 

2.2.0 property heap ptrborrowed

### プロトタイプ

```c
duk_bool_t duk_del_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_del_prop() と同様ですが、プロパティ名は、例えば duk_get_heapptr() を使って取得した Duktape ヒープポインタとして与えられます。ptr が NULL の場合、undefined がキーとして使用されます。


### 例

```c
duk_bool_t rc;
void *ptr;

duk_push_string(ctx, "propertyName");
ptr = duk_get_heapptr(ctx, -1);
/* String behind 'ptr' must remain reachable! */

rc = duk_del_prop_heapptr(ctx, -3, ptr);
printf("delete obj.propertyName -> rc=%d\n", (int) rc);
```

### 参照

duk_del_prop
duk_del_prop_index
duk_del_prop_string
duk_del_prop_lstring
duk_del_prop_literal


## duk_del_prop_index() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_del_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_del_prop() と同様ですが、プロパティ名は符号なし整数 arr_idx として与えられます。これは特に配列の要素を削除するのに便利です (しかし、それに限定されるものではありません)。

概念的には、プロパティ削除のために数値は文字列に強制されます。例えば、123はプロパティ名 "123 "と同等です。Duktapeは可能な限り、明示的な強制を避けています。


### 例

```c
duk_bool_t rc;

rc = duk_del_prop_index(ctx, -3, 123);
printf("delete obj[123] -> rc=%d\n", (int) rc);
```

### 参照

duk_del_prop
duk_del_prop_string
duk_del_prop_lstring
duk_del_prop_literal
duk_del_prop_heapptr


## duk_del_prop_literal() 

2.3.0 property literal

### プロトタイプ

```c
duk_bool_t duk_del_prop_literal(duk_context *ctx, duk_idx_t obj_idx, const char *key_literal);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_del_prop() と同様ですが、プロパティ名は文字列リテラルとして与えられます (duk_push_literal() を参照してください)。


### 例

```c
duk_bool_t rc;

rc = duk_del_prop_literal(ctx, -3, "myPropertyKey");
printf("delete obj.myPropertyKey -> rc=%d\n", (int) rc);
```

### 参照

duk_del_prop
duk_del_prop_index
duk_del_prop_string
duk_del_prop_lstring
duk_del_prop_heapptr


## duk_del_prop_lstring() 

2.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_del_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_del_prop() と同様ですが、プロパティ名は長さを明示的に指定した文字列として与えられます。


### 例

```c
duk_bool_t rc;

rc = duk_del_prop_lstring(ctx, -3, "internal" "\x00" "nul", 12);
printf("delete obj.internal[00]nul -> rc=%d\n", (int) rc);
```

### 参照

duk_del_prop
duk_del_prop_index
duk_del_prop_string
duk_del_prop_literal
duk_del_prop_heapptr


## duk_del_prop_string() 

1.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_del_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

Like duk_del_prop(), but the property name is given as a NUL-terminated C string key.


### 例

```c
duk_bool_t rc;

rc = duk_del_prop_string(ctx, -3, "propertyName");
printf("delete obj.propertyName -> rc=%d\n", (int) rc);
```

### 参照

duk_del_prop
duk_del_prop_index
duk_del_prop_lstring
duk_del_prop_literal
duk_del_prop_heapptr


## duk_destroy_heap() 

1.0.0 heap

### プロトタイプ

```c
void duk_destroy_heap(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

Duktapeヒープを破棄します。引数contextには、ヒープにリンクしている任意のcontextを指定することができます。ヒープに関連する全てのリソースは解放され、この呼び出しが完了した後は参照してはならない。これらのリソースには、ヒープにリンクされた全てのコンテキストと、ヒープ内の全ての文字列とバッファのポインタが含まれます。

ctxがNULLの場合、この呼び出しは失敗です。


### 例

```c
duk_destroy_heap(ctx);
```


## duk_dump_function() 

1.3.0 stack bytecode

### プロトタイプ

```c
void duk_dump_function(duk_context *ctx);
```

### スタック

| ... | function | -> | ... | bytecode |

### 要約

スタックトップにある ECMAScript 関数をバイトコードにダンプし、関数をバイトコードデータを含むバッファに置き換えます。バイトコードは duk_load_function() を使ってロードバックすることができます。

Duktapeバイトコードダンプ/ロード、サポートされる機能、および既知の制限に関するより多くの情報については、bytecode.rstを参照してください。Duktapeバイトコード・フォーマットは難読化を意図したものではありませんので、難読化に関する注記を参照してください。

### 例

```c
duk_eval_string(ctx, "(function helloWorld() { print('hello world'); })");
duk_dump_function(ctx);
/* stack top now contains a buffer containing helloWorld bytecode */
```

### 参照

duk_load_function


## duk_dup() 

1.0.0 stack

### プロトタイプ

```c
void duk_dup(duk_context *ctx, duk_idx_t from_idx);
```

### スタック

| ... | val | ... | -> | ... | val | ... | val |

### 要約

Push a duplicate of value at from_idx to the stack. If from_idx is invalid, throws an error.


### 例

```c
duk_push_int(ctx, 123);  /* -> [ ... 123 ] */
duk_push_int(ctx, 234);  /* -> [ ... 123 234 ] */
duk_dup(ctx, -2);        /* -> [ ... 123 234 123 ] */
```

### 参照

duk_dup_top


## duk_dup_top() 

1.0.0 stack

### プロトタイプ

```c
void duk_dup_top(duk_context *ctx);
```

### スタック

| ... | val | -> | ... | val | val |

### 要約

スタックトップにある値の複製をスタックにプッシュします。バリュースタックが空の場合、エラーを投げます。。

duk_dup(ctx, -1) を呼び出すのと同等です。


### 例

```c
duk_push_int(ctx, 123);  /* -> [ ... 123 ] */
duk_push_int(ctx, 234);  /* -> [ ... 123 234 ] */
duk_dup_top(ctx);        /* -> [ ... 123 234 234 ] */
```


## duk_enum() 

1.0.0 property object

### プロトタイプ

```c
void duk_enum(duk_context *ctx, duk_idx_t obj_idx, duk_uint_t enum_flags);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... | enum |

### 要約

obj_idx にあるオブジェクトの列挙子を作成します。列挙の詳細はenum_flagsで制御することができます。対象値がオブジェクトでない場合は、エラーを投げます。。

列挙のフラグ。

DUK_ENUM_INCLUDE_NONENUMERABLE 列挙不可能なプロパティも列挙します。デフォルトでは、列挙可能なプロパティのみが列挙されます。
DUK_ENUM_INCLUDE_HIDDEN 非表示のシンボルも列挙します。デフォルトでは、非表示のシンボルは列挙されません。DUK_ENUM_INCLUDE_SYMBOLS と一緒に使用します。Duktape 1.x では、このフラグは DUK_ENUM_INCLUDE_INTERNAL と呼ばれていた。
DUK_ENUM_INCLUDE_SYMBOLS シンボルを列挙の結果に含めます。DUK_ENUM_INCLUDE_HIDDEN が指定されていない限り、隠しシンボルは含まれません。
DUK_ENUM_EXCLUDE_STRINGS 列挙結果から文字列を除外します。デフォルトでは文字列が含まれます。
DUK_ENUM_OWN_PROPERTIES_ONLY オブジェクトの「自身の」プロパティのみを列挙します。デフォルトでは、継承されたプロパティも列挙されます。
DUK_ENUM_ARRAY_INDICES_ONLY 配列のインデックス、すなわち "0"、"1"、"2" などの形式のプロパティ名のみを列挙します。
DUK_ENUM_SORT_ARRAY_INDICES ES2015 [[OwnPropertyKeys]] の列挙順序を継承レベルごとではなく、列挙結果全体に適用します。また、シンボルは通常の文字列キーの後にソートされます (どちらも挿入順です)。
DUK_ENUM_NO_PROXY_BEHAVIOR プロキシ動作を呼び出さずにプロキシオブジェクト自身を列挙します。
フラグがない場合、列挙はfor-inのように動作します。自己および継承された列挙可能なプロパティが含まれ、列挙順はECMAScript ES2015 [[OwnPropertyKeys]] 列挙順（各相続レベルに対して適用）に従っています。

列挙子を作成したら、duk_next() を使用して列挙子からキー (またはキーと値のペア) を抽出します。

ES2015 [[OwnPropertyKeys]] の列挙順は、 (1) 配列インデックスの昇順、 (2) 配列インデックス以外のキーの挿入順、 (3) シンボルの挿入順となります。このルールは継承レベルごとに別々に適用されるため、配列インデックスのキーが継承された場合、結果的に順番が狂って表示されることになります。配列のインデックスが継承されることはほとんどないため、ほとんどの実用的なコードでは、これは問題ではありません。DUK_ENUM_SORT_ARRAY_INDICES を使用すると、列挙シーケンス全体を強制的にソートすることができます。

### 例

```c
duk_enum(ctx, -3, DUK_ENUM_INCLUDE_NONENUMERABLE);

while (duk_next(ctx, -1 /*enum_idx*/, 0 /*get_value*/)) {
    /* [ ... enum key ] */
    printf("-> key %s\n", duk_get_string(ctx, -1));
    duk_pop(ctx);  /* pop_key */
}

duk_pop(ctx);  /* pop enum object */
```

### 参照

duk_next


## duk_equals() 

1.0.0 compare

### プロトタイプ

```c
duk_bool_t duk_equals(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

| ... | val1 | ... | val2 | ... |

### 要約

idx1 と idx2 の値が等しいかどうかを比較します。ECMAScript の Equals 演算子 (==) のセマンティクスを使用して値が等しいと見なされる場合は 1 を、そうでない場合は 0 を返す。また、どちらかのインデックスが無効な場合は0を返す。

Equals 演算子が使用する Abstract Equality Comparison Algorithm は値の強制を行うため (ToNumber() および ToPrimitive()) 、この比較には副作用があり、エラーをスローする可能性があります。duk_strict_equals() で使用できる厳密な等式比較には副作用がありません。

Duktapeカスタム型の比較アルゴリズムは、Equality (non-strict) で説明されています。


### 例

```c
if (duk_equals(ctx, -3, -7)) {
    printf("values at indices -3 and -7 are equal\n");
}
```

### 参照

duk_strict_equals


## duk_error() 

1.0.0 error

### プロトタイプ

```c
duk_ret_t duk_error(duk_context *ctx, duk_errcode_t err_code, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

新しいエラーオブジェクトをスタックにプッシュし、それをスローします。この呼び出しは決して戻りません。

エラーオブジェクトのメッセージプロパティには、fmt と残りの引数を用いて sprintf フォーマットの文字列が設定されます。作成されたエラーオブジェクトの内部プロトタイプは err_code に基づいて選択されます。例えば、DUK_ERR_RANGE_ERROR は、組み込みの RangeError プロトタイプが使用されるようにします。ユーザーエラーコードの有効範囲は [1,16777215] です。

Error オブジェクトを投げずにスタックにプッシュするには、 duk_push_error_object() を使用します。

この関数は決して戻ってきませんが、プロトタイプは、以下のようなコードを可能にする戻り値を記述しています。

```c
if (argvalue < 0) {
    return duk_error(ctx, DUK_ERR_TYPE_ERROR, "invalid argument value: %d", (int) argvalue);
}
```

戻り値が無視される場合、コンパイル時の警告を避けるため、voidにキャストします。

```c
if (argvalue < 0) {
    (void) duk_error(ctx, DUK_ERR_TYPE_ERROR, "invalid argument value: %d", (int) argvalue);
}
```

### 例

```c
(void) duk_error(ctx, DUK_ERR_RANGE_ERROR, "argument out of range: %d", (int) argval);
```

### 参照

duk_error_va
duk_throw
duk_push_error_object
duk_generic_error
duk_eval_error
duk_range_error
duk_reference_error
duk_syntax_error
duk_type_error
duk_uri_error


## duk_error_va() 

1.1.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_error_va(duk_context *ctx, duk_errcode_t err_code, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

duk_error() の Vararg 変形です。

この API コールは、呼び出し側のコードの va_end() マクロが (例えばエラースローのために) 到達しないかもしれないので、完全には移植性がない。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存しています; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_range_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_error_va(ctx, DUK_ERR_RANGE_ERROR, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error
duk_throw
duk_push_error_object
duk_generic_error_va
duk_eval_error_va
duk_range_error_va
duk_reference_error_va
duk_syntax_error_va
duk_type_error_va
duk_uri_error_va


## duk_eval() 

1.0.0 compile

### プロトタイプ

```c
void duk_eval(duk_context *ctx);
```

### スタック

| ... | source | -> | ... | result |

### 要約

ECMAScript のソースコードをスタックの一番上で評価し、スタックの一番上に単一の戻り値を残す。このようなエラーは自動的には捕捉されません。一時的なeval関数に関連するファイル名は "eval "です。

このAPIコールは、基本的に次のようなショートカットになっています。

```c
duk_push_string(ctx, "eval");
duk_compile(ctx, DUK_COMPILE_EVAL);  /* [ ... source filename ] -> [ function ] */
duk_call(ctx, 0);
```

ソースコードは、明示的な "use strict" 指令を含まない限り、非厳密モードで評価されます。特に、現在のコンテキストの厳密性は eval のコードには反映されません。これにより、Duktape/Cの関数呼び出しの内部でevalを使うか（strictモード）、外部で使うか（non-strictモード）で、evalの厳密性が変わってしまうような混乱が避けられます。

eval の入力が固定文字列の場合、duk_eval_string() を使うこともできます。


### 例

```c
/* Eval result in ECMAScript is the last non-empty statement; here, the
 * result of the eval is the number 123.
 */

duk_push_string(ctx, "print('Hello world!'); 123;");
duk_eval(ctx);
printf("return value is: %lf\n", (double) duk_get_number(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_peval
duk_eval_noresult
duk_eval_string
duk_eval_string_noresult
duk_eval_lstring
duk_eval_lstring_noresult


## duk_eval_error() 

2.0.0 error

### プロトタイプ

```c
duk_ret_t duk_eval_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_EVAL_ERROR を持つ duk_error() と同等の便宜的な API 呼び出し。


### 例

```c
return duk_eval_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_eval_error_va() 

2.0.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_eval_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_EVAL_ERROR を持つ duk_error_va() と同等の便宜的な API 呼び出し。

このAPIコールは完全には移植性がない。なぜなら、呼び出し側のコードの va_end() マクロは(例えばエラースローにより)決して到達できないかもしれないからです。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存します; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_eval_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_eval_error_va(ctx, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error_va


## duk_eval_lstring() 

1.0.0 string compile

### プロトタイプ

```c
void duk_eval_lstring(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

| ... | -> | ... | result |

### 要約

duk_eval() と同様ですが、eval の入力は、長さを明示した C 文字列として与えられます。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では便利です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_eval_lstring(ctx, src, len);
printf("result is: %s\n", duk_get_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_eval_lstring_noresult


## duk_eval_lstring_noresult() 

1.0.0 string compile

### プロトタイプ

```c
void duk_eval_lstring_noresult(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

| ... | -> | ... |

### 要約

duk_eval_lstring() と同様だが、バリュースタックに結果を残さない。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_eval_lstring_noresult(ctx, src, len);
```


## duk_eval_noresult() 

1.0.0 compile

### プロトタイプ

```c
void duk_eval_noresult(duk_context *ctx);
```

### スタック

| ... | source | -> | ... |

### 要約

duk_eval() と同様であるが、バリュースタックに結果を残さない。


### 例

```c
duk_push_string(ctx, "print('Hello world!');");
duk_eval_noresult(ctx);
```

### 参照

duk_eval_string_noresult
duk_eval_lstring_noresult


## duk_eval_string() 

1.0.0 string compile

### プロトタイプ

```c
void duk_eval_string(duk_context *ctx, const char *src);
```

### スタック

| ... | -> | ... | result |

### 要約

duk_eval() と同様ですが、eval の入力は C の文字列として与えられます。

この変種では、入力ソースコードは Duktape によってインター ネットされないので、低メモリ環境では有用です。

### 例

```c
duk_eval_string(ctx, "'testString'.toUpperCase()");
printf("result is: %s\n", duk_get_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_eval_string_noresult


## duk_eval_string_noresult() 

1.0.0 string compile

### プロトタイプ

```c
void duk_eval_string_noresult(duk_context *ctx, const char *src);
```

### スタック

| ... | -> | ... |

### 要約

duk_eval_string() と同様であるが、バリュースタックに結果を残さない。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
duk_eval_string_noresult(ctx, "print('testString'.toUpperCase())");
```


## duk_fatal() 

1.0.0 error

### プロトタイプ

```c
duk_ret_t duk_fatal(duk_context *ctx, const char *err_msg);
```

### スタック

(バリュースタックに影響ナシ)


### 要約

致命的なエラーハンドラをオプションのメッセージ付きで呼び出します (err_msg は NULL かもしれません)。Duktape のすべての文字列と同様に、エラーメッセージは UTF-8 文字列であるべきですが、純粋な ASCII が強く推奨されます。

致命的なエラー・ハンドラは決して戻らず、例えば現在のプロセスを終了させるかもしれません。エラー捕捉ポイント (try-catch 文やエラー捕捉 API 呼び出しなど) は回避され、ファイナライザは実行されない。この関数は、本当に致命的な、回復不可能なエラーが発生したときにのみ呼び出すべきです。

この関数は決して戻りませんが、プロトタイプは以下のようなコードを可能にする戻り値を記述しています。

```c
if (argvalue < 0) {
    return duk_fatal(ctx, "argvalue invalid, cannot continue execution");
}
```

### 例

```c
duk_fatal(ctx, "assumption failed");
```


## duk_free() 

1.0.0 memory

### プロトタイプ

```c
void duk_free(duk_context *ctx, void *ptr);
```

### スタック

(バリュースタックに影響なし)


### 要約

duk_free_raw() と同様ですが、ガベージコレクションのステップを含むかもしれません。ガベージコレクションとの相互作用は、操作を失敗させる原因にはなりません。

duk_alloc() や duk_alloc_raw() で割り当てられたメモリや、それらの再割り当ての派生型のメモリを解放するために、 duk_free() を使用することが可能です。

現在のところ、duk_free() は決してガベージコレクションをパスさせません。

### 例

```c
void *buf = duk_alloc(ctx, 1024);
/* ... */

duk_free(ctx, buf);  /* safe even if 'buf' is NULL */
```

### 参照

duk_free_raw


## duk_free_raw() 

1.0.0 memory

### プロトタイプ

```c
void duk_free_raw(duk_context *ctx, void *ptr);
```

### スタック

(バリュースタックに影響なし)


### 要約

コンテキストに登録されたアロケーション機能で割り当てられたメモリを解放します。この操作は失敗しない．ptr が NULL の場合，この呼び出しは失敗です．

duk_free_raw() は、 duk_alloc() や duk_alloc_raw() およびそれらの再割り当て関数で割り当てられたメモリを解放するために使用できます。


### 例

```c
void *buf = duk_alloc_raw(ctx, 1024);
/* ... */

duk_free_raw(ctx, buf);  /* safe even if 'buf' is NULL */
```

### 参照

duk_free


## duk_freeze() 

2.2.0 property object

### プロトタイプ

```c
void duk_freeze(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

(バリュースタックに影響なし)


### 要約

obj_idx にあるオブジェクトの Object.freeze() に相当します。凍結されると、オブジェクトは自動的にコンパクト化されます。凍結された場合、オブジェクトは自動的に圧縮されます。インデックスが無効な場合、エラーがスローされます。


### 例

```c
duk_freeze(ctx, -3);
```


## duk_gc() 

1.0.0 memory heap

### プロトタイプ

```c
void duk_gc(duk_context *ctx, duk_uint_t flags);
```

### スタック

(バリュースタックに影響なし)


### 要約

マークアンドスイープガベージコレクションラウンドを強制的に実行します。

以下のフラグが定義されています。

定義 説明
DUK_GC_COMPACT オブジェクトプロパティテーブルの圧縮を強制します。

ファイナライザーを持つオブジェクトも確実に収集するために、この関数を2回呼び出したい場合があります。現在、そのようなオブジェクトを収集するためには、2回のマーク＆スイープラウンドが必要です。最初のラウンドでは、オブジェクトをファイナライズ可能なものとしてマークし、ファイナライザを実行します。2回目のラウンドでは、ファイナライズの後でもオブジェクトが到達できないことを確認し、オブジェクトを解放します。


### 例

```c
duk_gc(ctx, 0);
```


## duk_generic_error() 

2.0.0 error

### プロトタイプ

```c
duk_ret_t duk_generic_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_ERROR を持つ duk_error() と同等の便宜的な API 呼び出し。


### 例

```c
return duk_generic_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_generic_error_va() 

2.0.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_generic_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_ERROR を持つ duk_error_va() と同等の便宜的な API 呼び出し。

このAPIコールは完全には移植性がない。なぜなら、呼び出し側のコードの va_end() マクロは(例えばエラースローにより)到達しないかもしれないからです。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存します; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_generic_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_generic_error_va(ctx, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error_va


## duk_get_boolean() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_get_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある boolean 値を、値を変更したり強制したりすることなく取得します。値が真であれば1を、値が偽であるか、booleanでないか、またはインデックスが無効であれば0を返す。

値は強制されないので、ECMAScript の「真実の」値（空でない文字列など）であっても、偽として扱われることに注意してください。値を強制したい場合は、duk_to_boolean()を使用してください。


### 例

```c
if (duk_get_boolean(ctx, -3)) {
    printf("value is true\n");
}
```

### 参照

duk_get_boolean_default


## duk_get_boolean_default() 

2.1.0 stack

### プロトタイプ

```c
duk_bool_t duk_get_boolean_default(duk_context *ctx, duk_idx_t idx, duk_bool_t def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_boolean() と同様ですが、デフォルト値が明示されており、値が boolean でない場合、あるいはインデックスが無効な場合に返されます。


### 例

```c
duk_bool_t flag_xyz = duk_get_boolean_default(ctx, 2, 1);  /* default: true */
```

### 参照

duk_get_boolean


## duk_get_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_get_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... |

### 要約

idxにある（プレーン）バッファ値のデータポインタを、値を変更したり強制したりすることなく取得します。値が0以外のサイズの有効なバッファである場合、非NULLポインタを返します。サイズが0のバッファの場合、NULLまたはNULLでないポインタを返すことがあります。値がバッファでない場合、またはインデックスが無効な場合は NULL を返す。out_size が非NULLの場合、バッファのサイズが *out_size に書き込まれます。

対象が固定バッファの場合、返されるポインタは安定であり、バッファの寿命が尽きるまで変化することはない。動的および外部バッファの場合、ポインタはバッファのサイズ変更または再構成に伴って変更される可能性が あります。呼び出し側は、バッファのサイズ変更/再構成後に本APIコールから返されるポインタ値が使用され ないようにする責任を負う。

サイズゼロのバッファと非バッファを返り値だけで区別する信頼できる方法はありません: 非バッファにはサイズゼロのNULLが返されます。非バッファの場合、サイズ0のNULLが返されます。サイズ0のバッファの場合、同じ値が返されるかもしれません（NULLでないポインタが返される可能性もありますが）。duk_is_buffer() や duk_is_buffer_data() 、あるいはバッファやバッファ・オブジェクトの型チェックを行う際に使用してください。

### 例

```c
void *ptr;
duk_size_t sz;

ptr = duk_get_buffer(ctx, -3, &sz);
printf("buf=%p, size=%lu\n", ptr, (unsigned long) sz);
```

### 参照

duk_require_buffer
duk_get_buffer_data
duk_require_buffer_data


## duk_get_buffer_data() 

1.3.0 stack buffer object buffer

### プロトタイプ

```c
void *duk_get_buffer_data(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... |

### 要約

idxにあるプレーンバッファまたはバッファオブジェクト(ArrayBuffer, Node.js Buffer, DataView, TypedArray view)の値のデータポインタを、値を変更したり強制したりすることなく取得します。値がゼロでないサイズの有効なバッファである場合、非NULLポインタを返します。サイズが0のバッファの場合、NULLまたは非NULLポインタを返すことがあります。値がバッファでない場合、値がバッファオブジェクトで、そのバッファオブジェクトの見かけのサイズを完全にカバーしない場合、またはインデックスが無効な場合、NULLを返します。out_size が非 NULL の場合、バッファのサイズが *out_size に書き込まれます。

戻り値のポインタと length が示すデータ領域は、プレーンバッファの場合はバッファ全体であり、バッファオブジェクトの場合はアクティブな「スライス」です。返される長さは(要素数ではなく)バイト数で表現され、常に ptr[0] から ptr[len - 1] にアクセスできるようにします。例えば

例えば、new Uint8Array(16) の場合、戻り値のポインタは配列の先頭を指し、長さは 16 になります。
new Uint32Array(16) の場合、リターンポインタは配列の先頭を指し、長さは64になります。
new Uint32Array(16).subarray(2, 6) の場合、リターンポインタはサブ配列の先頭（Uint32Array の先頭から 2 x 4 = 8 バイト）を指し、長さは 16 (= (6-2) x 4) になります。
バッファオブジェクトのターゲット値または基礎となるバッファ値が固定バッファの場合、返されるポインタは安定で、バッファの寿命が尽きるまで変化することはありません。動的および外部バッファの場合、ポインタはバッファのサイズ変更または再構成に伴って変更される可能性があります。呼び出し側は、この API 呼び出しから返されたポインタ値がバッファのサイズ変更/再構成後に使用されないようにする責任を負います。ECMAScript コードから new Buffer(8) などで直接作成されたバッファは、自動的に固定バッファにバックアップされるため、そのポインタは常に安定したものとなります。

特殊なケースとして、バッファオブジェクトの見かけのサイズより小さいプレーンバッファがバッファオブジェクトのバックグランドになることがあります。このような場合、NULL が返されます。これにより、返されたポインタとサイズが常に安全に使用できることが保証されます。(この動作はマイナーバージョンでも変更される可能性があります)。

サイズがゼロのバッファとそうでないバッファを、返り値だけから区別する信頼できる方法はありません。非バッファの場合、サイズ0のNULLが返されます。サイズ0のバッファの場合、同じ値が返されるかもしれません（NULLでないポインタが返される可能性もありますが）。duk_is_buffer() や duk_is_buffer_data() 、あるいはバッファやバッファ・オブジェクトの型チェックを行う際に使用してください。

### 例

```c
void *ptr;
duk_size_t sz;

duk_eval_string(ctx, "new Uint16Array(16)");
ptr = duk_get_buffer_data(ctx, -1, &sz);

/* Here 'ptr' would be non-NULL and sz would be 32 (bytes). */
```

### 参照

duk_require_buffer_data
duk_get_buffer
duk_require_buffer


## duk_get_buffer_data_default() 

2.1.0 stack buffer

### プロトタイプ

```c
void *duk_get_buffer_data_default(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

| ... | val | ... |

### 要約

duk_get_buffer_data() と同様ですが、明示的にデフォルト値が設定されており、値がプレーンバッファ、バッファオブジェクトでない場合、またはインデックスが無効な場合に返されます。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルト・ポインタ値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
void *ptr;
duk_size_t sz;
char buf[256];

/* Use a buffer or buffer object given at index 2, or default to 'buf'. */
ptr = duk_get_buffer_data_default(ctx, 2, &sz, (void *) buf, sizeof(buf));
```

### 参照

duk_get_buffer_data
duk_get_buffer_default


## duk_get_buffer_default() 

2.1.0 stack buffer

### プロトタイプ

```c
void *duk_get_buffer_default(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

| ... | val | ... |

### 要約

duk_get_buffer() と同様ですが、デフォルト値が明示されており、値がバッファでない場合、あるいはインデックスが無効な場合に返されます。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトポインタの値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
void *ptr;
duk_size_t sz;
char buf[256];

/* Use a buffer given at index 2, or default to 'buf'. */
ptr = duk_get_buffer_data_default(ctx, 2, &sz, (void *) buf, sizeof(buf));
```

### 参照

duk_get_buffer
duk_get_buffer_data_default


## duk_get_c_function() 

1.0.0 stack function

### プロトタイプ

```c
duk_c_function duk_get_c_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

Duktape/C関数に関連付けられたECMAScript関数オブジェクトからDuktape/C関数ポインタ（duk_c_function）を取得します。値がそのような関数でない場合、または idx が無効な場合、NULL を返します。

もし、無効な値やインデックスに対してエラーが投げられることを望むなら、 duk_require_c_function() を使用してください。


### 例

```c
duk_c_function funcptr;

funcptr = duk_get_c_function(ctx, -3);
```

### 参照

duk_get_c_function_default


## duk_get_c_function_default() 

2.1.0 stack function

### プロトタイプ

```c
duk_c_function duk_get_c_function_default(duk_context *ctx, duk_idx_t idx, duk_c_function def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_c_function() と同様ですが、デフォルト値が明示されており、値が Duktape/C関数でない場合、あるいはインデックスが無効な場合に返されます。


### 例

```c
duk_c_function funcptr;

/* Native callback, default to nop_callback. */
funcptr = duk_get_c_function_default(ctx, -3, nop_callback);
```

### 参照

duk_get_c_function


## duk_get_context() 

1.0.0 stack borrowed

### プロトタイプ

```c
duk_context *duk_get_context(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxにあるDuktapeスレッドのコンテキスト・ポインタを取得します。idxの値がDuktapeスレッドでない場合、またはインデックスが無効な場合、 NULLを返します。

返されたコンテキスト・ポインタは、ガベージコレクションの観点から Duktapeスレッドに到達可能な間だけ有効です。

無効な値やインデックスに対してエラーを投げたい場合は、 duk_require_context() を使ってください。


### 例

```c
duk_context *new_ctx;

/* Create a new thread and get a context pointer. */

(void) duk_push_thread(ctx);
new_ctx = duk_get_context(ctx, -1);

/* You can use new_ctx as long as the related thread is reachable
 * from a garbage collection point of view.
 */

duk_push_string(new_ctx, "foo");

/* This duk_pop() makes the new thread unreachable (assuming there
 * is no other reference to it), so new_ctx is no longer valid
 * afterwards.
 */

duk_pop(ctx);

/* Using new_ctx here may cause a crash. */
```

### 参照

duk_get_context_default


## duk_get_context_default() 

2.1.0 stack borrowed

### プロトタイプ

```c
duk_context *duk_get_context_default(duk_context *ctx, duk_idx_t idx, duk_context *def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_context() と同様ですが、デフォルト値が明示されており、値が Duktapeスレッドでない場合、またはインデックスが無効な場合に返されます。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルト・ポインタの値は、Duktape によって追跡されない。例えば、 duk_opt_string() はデフォルト文字列引数のコピーを作らない。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
duk_context *target_ctx;

target_ctx = duk_get_context_default(ctx, 2, default_ctx);
```

### 参照

duk_get_context


## duk_get_current_magic() 

1.0.0 magic function

### プロトタイプ

```c
duk_int_t duk_get_current_magic(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

実行中のDuktape/C関数に関連する16ビット符号付き「マジック」値を取得します。現在起動しているものがない場合は、0が返されます。

magic関数は、同じDuktape/C関数を少し異なる動作で使用することを可能にします； 動作フラグや他のパラメータをmagicフィールドに渡すことができます。これは、動作に関連するフラグやプロパティを通常のプロパティとして関数オブジェクトに持たせるよりも低コストです。

符号なし16ビット値が必要な場合は、単純にビット単位のANDで結果を得ます。

```c
unsigned int my_flags = ((unsigned int) duk_get_current_magic(ctx)) & 0xffffU;
```

### 例

```c
duk_int_t my_flags = duk_get_current_magic(ctx);
if (my_flags & 0x01) {
    printf("flag set\n");
} else {
    printf("flag not set\n");
}
```

### 参照

duk_get_magic
duk_set_magic


## duk_get_error_code() 

1.1.0 stack error

### プロトタイプ

```c
duk_errcode_t duk_get_error_code(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値を、その値がどの Error サブクラスを継承しているかに基づいてエラーコード DUK_ERR_xxx にマップします。たとえば、スタックトップの値が ReferenceError を継承したユーザー定義エラーである場合、返り値は DUK_ERR_REFERENCE_ERROR となります。値が Error を継承しているが、標準のサブクラス (EvalError, RangeError, ReferenceError, SyntaxError, TypeError, URIError) のいずれかを継承していない場合、DUK_ERR_ERROR が返されます。値がオブジェクトでない場合、Error を継承しない場合、または idx が無効な場合、0 (= DUK_ERR_NONE) を返します。


### 例

```c
if (duk_get_error_code(ctx, -3) == DUK_ERR_URI_ERROR) {
    printf("Invalid URI\n");
}
```


## duk_get_finalizer() 

1.0.0 object finalizer

### プロトタイプ

```c
void duk_get_finalizer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | val | ... | finalizer |

### 要約

idxの値に関連するファイナライザを取得します。値がオブジェクトでない場合、またはファイナライザを持たないオブジェクトである場合、代わりにundefinedがスタックにプッシュされます。


### 例

```c
/* Get the finalizer of an object at index -3. */
duk_get_finalizer(ctx, -3);
```

### 参照

duk_set_finalizer


## duk_get_global_heapptr() 

2.3.0 property heap ptr borrowed

### プロトタイプ

```c
duk_bool_t duk_get_global_heapptr(duk_context *ctx, void *ptr);
```

### スタック

| ... | -> | ... | val | (if key exists)
| ... | -> | ... | undefined | (if key doesn't exist)

### 要約

duk_get_global_string() と同様ですが、プロパティ名は、例えば duk_get_heapptr() を使って取得した Duktape ヒープポインタとして与えられます。ptr が NULL の場合、undefined がキーとして使用されます。


### 例

```c
(void) duk_get_global_heapptr(ctx, my_ptr_ref);
```

### 参照

duk_get_global_string
duk_get_global_lstring
duk_get_global_literal


## duk_get_global_literal() 

2.3.0 property literal

### プロトタイプ

```c
duk_bool_t duk_get_global_literal(duk_context *ctx, const char *key_literal);
```

### スタック

| ... | -> | ... | val | (if key exists)
| ... | -> | ... | undefined | (if key doesn't exist)

### 要約

duk_get_global_string() と同様ですが、プロパティ名は文字列リテラルとして与えられます (duk_push_literal() を参照)。


### 例

```c
(void) duk_get_global_literal(ctx, "encodeURIComponent");
duk_push_string(ctx, "foo bar");
duk_call(ctx, 1);  /* [ ... encodeURIComponent "foo bar" ] -> [ "foo%20bar" ] */
printf("encoded: %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_get_global_string
duk_get_global_lstring
duk_get_global_heapptr


## duk_get_global_lstring() 

2.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_get_global_lstring(duk_context *ctx, const char *key, duk_size_t key_len);
```

### スタック

| ... | -> | ... | val | (if key exists)
| ... | -> | ... | undefined | (if key doesn't exist)

### 要約

duk_get_global_string() と同様ですが、キーは長さを明示した文字列として与えられます。


### 例

```c
(void) duk_get_global_lstring(ctx, "internal" "\x00" "nul", 12);
```

### 参照

duk_get_global_string
duk_get_global_literal
duk_get_global_heapptr


## duk_get_global_string() 

1.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_get_global_string(duk_context *ctx, const char *key);
```

### スタック

| ... | -> | ... | val | (if key exists)
| ... | -> | ... | undefined | (if key doesn't exist)

### 要約

グローバルオブジェクトから key という名前のプロパティを取得します。そのプロパティが存在する場合は非ゼロを、そうでない場合はゼロを返します。これは、以下の関数と同等のことを行う便利な関数です。

```c
duk_bool_t ret;

duk_push_global_object(ctx);
ret = duk_get_prop_string(ctx, -1, key);
duk_remove(ctx, -2);
/* 'ret' would be the return value from duk_get_global_string() */
```

### 例

```c
(void) duk_get_global_string(ctx, "encodeURIComponent");
duk_push_string(ctx, "foo bar");
duk_call(ctx, 1);  /* [ ... encodeURIComponent "foo bar" ] -> [ "foo%20bar" ] */
printf("encoded: %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_get_global_lstring
duk_get_global_literal
duk_get_global_heapptr


## duk_get_heapptr() 

1.1.0 stack heap ptr borrowed

### プロトタイプ

```c
void *duk_get_heapptr(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx で Duktape ヒープに割り当てられた値（オブジェクト、バッファ、文字列）への参照 を借用した void * を取得します。インデックスが無効であるか、対象の値がヒープに割り当てられていない場合は NULL を返します。返されたポインタは解釈したり再参照したりしてはならないが、 duk_push_heapptr() は、後で元の値をバリュースタックにプッシュするために使うことができます。

返された void ポインタは、ガベージコレクションの観点から見て、元の値に到達可能な間だけ有効です。そうでない場合、duk_push_heapptr() を使用することはメモリ上安全ではありません。


### 例

```c
duk_context *new_ctx;
void *ptr;

duk_eval_string(ctx, "({ foo: 'bar' })");
ptr = duk_get_heapptr(ctx, -1);

/* The original value must remain reachable for Duktape up to a future
 * duk_push_heapptr().  Here we just write it to the global object, but
 * it could also be a value stack somewhere, a stash object, etc.
 */
duk_put_global_string(ctx, "ref");

/* Later, assuming the original value has been reachable all the way
 * to here:
 */

duk_push_heapptr(ctx, ptr);
duk_get_prop_string(ctx, -1, "foo");
printf("obj.foo: %s\n", duk_safe_to_string(ctx, -1));  /* prints 'bar' */
```

### 参照

duk_require_heapptr
duk_push_heapptr
duk_get_heapptr_default


## duk_get_heapptr_default() 

2.1.0 stack heap ptr borrowed

### プロトタイプ

```c
void *duk_get_heapptr_default(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_heapptr() と同様ですが、デフォルト値が明示されており、値が boolean でない場合、あるいはインデックスが無効な場合に返されます。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトポインタの値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
void *ptr;

ptr = duk_get_heapptr_default(ctx, 2, default_ptr);
```

### 参照

duk_get_heapptr


## duk_get_int() 

1.0.0 stack

### プロトタイプ

```c
duk_int_t duk_get_int(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx に数値を取得し、まず [DUK_INT_MIN, DUK_INT_MAX] の間で値をクランプし、次にゼロに向かって切り捨てることで C の duk_int_t に変換します。スタック上の値は変更されません。値がNaNであるか、数値でないか、またはインデックスが無効である場合、0を返す。

変換の例

Input	Output
-Infinity	DUK_INT_MIN
DUK_INT_MIN - 1	DUK_INT_MIN
-3.9	-3
3.9	3 
DUK_INT_MAX + 1	DUK_INT_MAX
+Infinity	DUK_INT_MAX
NaN	0
"123"	0 (non-number)

この強制は、NaN を DUK_INT_MIN に強制するような直感的でない（移植性のない）動作をする可能性のある、double から integer への基本的な C のキャストとは異なります。また、ECMAScript の ToInt32() による強制とは異なり、ネイティブな duk_int_t の全範囲が許可されます (32ビットより多くなる可能性があります)。

### 例

```c
printf("int value: %ld\n", (long) duk_get_int(ctx, -3));
```

### 参照

duk_get_int_default


## duk_get_int_default() 

2.1.0 stack

### プロトタイプ

```c
duk_int_t duk_get_int_default(duk_context *ctx, duk_idx_t idx, duk_int_t def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_int() と同様ですが、デフォルト値が明示されており、値が数値でない場合、あるいはインデックスが無効な場合に返されます。


### 例

```c
int port = (int) duk_get_int_default(ctx, 1, 80);  /* default: 80 */
```

### 参照

duk_get_int


## duk_get_length() 

1.0.0 stack

### プロトタイプ

```c
duk_size_t duk_get_length(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxにある値の型固有の "長さ "を取得します。

String: 文字列の文字数 (バイト数ではない)
オブジェクト。Math.floor(ToNumber(obj.length)): 結果が duk_size_t unsigned の範囲内にあれば、それ以外は 0 (配列の場合、結果は Array .length になる)
バッファ: バッファのバイト長
その他の型、または無効なスタックインデックス。0
文字列のバイト長を得るには、duk_get_lstring() を使用します。


### 例

```c
if (duk_is_string(ctx, -3)) {
    printf("string char len is %lu\n", (unsigned long) duk_get_length(ctx, -3));
}
```

### 参照

duk_set_length


## duk_get_lstring() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_get_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

| ... | val | ... |

### 要約

idx にある文字列の文字データポインタと長さを、値を変更したり強制したりすることなく取得します。読み取り専用で NUL 終端の文字列データへの非 NULL ポインタを返し、文字列のバイト長を *out_len に書き込む (out_len が非 NULL の場合)。値が文字列でない場合、またはインデックスが無効な場合は、NULL を返し、*out_len に 0 を書き込む (out_len が NULL でない場合)。

文字列の文字数（バイト数ではなく）を取得するには、 duk_get_length() を使用します。

これは、バッファデータポインタの扱い方とは異なります(技術的な理由による)。

### 例

```c
const char *str;
duk_size_t len;

str = duk_get_lstring(ctx, -3, &len);
if (str) {
    printf("value is a string, %lu bytes\n", (unsigned long) len);
}
```

### 参照

duk_get_string
duk_get_lstring_default
duk_get_string_default


## duk_get_lstring_default() 

2.1.0 string stack

### プロトタイプ

```c
const char *duk_get_lstring_default(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len, const char *def_ptr, duk_size_t def_len);
```

### スタック

| ... | val | ... |

### 要約

duk_get_lstring() と同様ですが、値が文字列でないかインデックスが無効な場合に返される、 明示的なデフォルト値があります。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトポインタの値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
const char *str;
duk_size_t len;

str = duk_get_lstring_default(ctx, -3, &len, "foo" "\x00" "bar", 7);
```

### 参照

duk_get_string_default


## duk_get_magic() 

1.0.0 magic function

### プロトタイプ

```c
duk_int_t duk_get_magic(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxにあるDuktape/C関数に関連する16ビット符号付き「マジック」値を取得します。もしその値がDuktape/C関数でない場合は、エラーが投げられます。

軽量関数は、符号付き整数（-128から127）として解釈される8ビットのマジック・バリューのためのスペースしか持っていません。

### 例

```c
duk_int_t my_flags = duk_get_magic(ctx, -3);
```

### 参照

duk_get_current_magic
duk_set_magic


## duk_get_memory_functions() 

1.0.0 memory heap

### プロトタイプ

```c
void duk_get_memory_functions(duk_context *ctx, duk_memory_functions *out_funcs);
```

### スタック

(バリュースタックに影響なし)


### 要約

コンテキストが使用するメモリ管理関数を取得します。

通常、この関数を呼び出す理由はない。 duk_alloc(), duk_realloc(), duk_free() のようなラップされたメモリ管理関数を通して、メモリ管理プリミティブを使用することができるからです。


### 例

```c
duk_memory_functions funcs;

duk_get_memory_functions(ctx, &funcs);
```


## duk_get_now() 

2.0.0 time

### プロトタイプ

```c
duk_double_t duk_get_now(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

ECMAScript 環境から見た POSIX ミリ秒単位で現在の時刻を取得します。戻り値は Date.now() と一致しますが、サブミリ秒の分解能が利用できるかもしれないという留保がつきます。


### 例

```c
duk_double_t now;

now = duk_get_now(ctx);
print("timestamp: %lf\n", (double) now);
```


## duk_get_number() 

1.0.0 stack

### プロトタイプ

```c
duk_double_t duk_get_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある数値の値を、値を変更したり強制したりすることなく取得します。値が数値でない場合、またはインデックスが無効な場合は NaN を返す。


### 例

```c
printf("value: %lf\n", (double) duk_get_number(ctx, -3));
```

### 参照

duk_get_number_default


## duk_get_number_default() 

2.1.0 stack

### プロトタイプ

```c
duk_double_t duk_get_number_default(duk_context *ctx, duk_idx_t idx, duk_double_t def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_number() と同様ですが、デフォルト値が明示されており、 値が数字でない場合、あるいはインデックスが無効な場合に返されます。


### 例

```c
duk_double_t backoff_multiplier = duk_get_number_default(ctx, 2, 1.5);  /* default: 1.5 */
```

### 参照

duk_get_number


## duk_get_pointer() 

1.0.0 stack

### プロトタイプ

```c
void *duk_get_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にあるポインタの値を、値を変更したり強制したりすることなく、void * として取得します。値がポインタでない場合、またはインデックスが無効な場合は NULL を返す。


### 例

```c
void *ptr;

ptr = duk_get_pointer(ctx, -3);
printf("my pointer is: %p\n", ptr);
```


## duk_get_pointer_default() 

2.1.0 stack

### プロトタイプ

```c
void *duk_get_pointer_default(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_pointer() と同様ですが、明示的にデフォルト値が設定されており、値がポインタでない場合、あるいはインデックスが無効な場合に返されます。


### 例

```c
void *ptr;

ptr = duk_get_pointer_default(ctx, -3, (void *) 0x12345678);
printf("my pointer is: %p\n", ptr);
```

### 参照

duk_get_pointer


## duk_get_prop() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_get_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

| ... | obj | ... | key -> | ... | obj | ... | val | (if key exists)
| ... | obj | ... | key -> | ... | obj | ... | undefined | (if key doesn't exist)

### 要約

obj_idx に指定された値のプロパティキーを取得します。リターンコードとエラースローの動作。

プロパティが存在する場合、1 が返され、key はバリュースタック上のプロパティ値で置き換えられます。プロパティがアクセサである場合、getter 関数はエラーを投げます。かもしれない。
プロパティが存在しない場合、0 が返され、key はバリュースタック上の undefined に置き換えられます。
obj_idx の値がオブジェクト互換でない場合、エラーを投げます。。
obj_idx が無効な場合、エラーがスローされます。
プロパティの読み取りは ECMAScript 式 res = obj[key] と同等であるが、プロパティの有無は呼び出しの戻り値で示されるという例外があります。正確なセマンティクスについては、プロパティアクセサ、GetValue（V）、および [[Get]] (P) を参照してください。ターゲット値及びキーは両方とも強制されます。

ターゲット値は自動的にオブジェクトにコーセーされます。例えば、文字列はStringに変換され、その "length "プロパティにアクセスすることができます。
key 引数は内部的に ToPropertyKey() 強制変換で文字列か Symbol に変換されます。配列や数値インデックスに対しては、明示的な文字列強制を回避する内部的な高速パスが存在するため、該当する場合は数値キーを使用します。
ターゲットが get トラップを実装する Proxy オブジェクトである場合、トラップが呼び出され、API コールは常に 1 (すなわち、プロパティが存在する) を返します: プロパティの不在/存在は、get Proxy トラップで示されません。このように、ターゲットオブジェクトが潜在的にProxyである場合、APIコールの戻り値は限定的にしか利用できない場合があります。

もしキーが固定文字列であれば、1回のAPIコールを回避して、 duk_get_prop_string() variant を使用することができます。同様に、キーが配列のインデックスである場合、 duk_get_prop_index() を使用することができます。

プロパティアクセスの基本値は通常オブジェクトですが、技術的には任意の値にすることができます。普通の文字列やバッファの値には仮想的なインデックスプロパティがあり、例えば "foo"[2] にアクセスすることができます。また、ほとんどのプリミティブな値は何らかのプロトタイプオブジェクトを継承しているので、例えば (12345).toString(16) のようにメソッドを呼び出すことができます。

### 例

```c
/* reading [global object].Math.PI */
duk_push_global_object(ctx);    /* -> [ global ] */
duk_push_string(ctx, "Math");   /* -> [ global "Math" ] */
duk_get_prop(ctx, -2);          /* -> [ global Math ] */
duk_push_string(ctx, "PI");     /* -> [ global Math "PI" ] */
duk_get_prop(ctx, -2);          /* -> [ global Math PI ] */
printf("Math.PI is %lf\n", (double) duk_get_number(ctx, -1));
duk_pop_n(ctx, 3);

/* reading a configuration value, cfg_idx is normalized
 * index of a configuration object.
 */
duk_push_string(ctx, "mySetting");
if (duk_get_prop(ctx, cfg_idx)) {
    const char *str_value = duk_to_string(ctx, -1);
    printf("configuration setting present, value: %s\n", str_value);
} else {
    printf("configuration setting missing\n");
}
duk_pop(ctx);  /* remember to pop, regardless of whether or not present */
```

### 参照

duk_get_prop_index
duk_get_prop_string
duk_get_prop_lstring
duk_get_prop_literal
duk_get_prop_heapptr


## duk_get_prop_desc() 

2.0.0 sandbox property

### プロトタイプ

```c
void duk_get_prop_desc(duk_context *ctx, duk_idx_t obj_idx, duk_uint_t flags);
```

### スタック

| ... | obj | ... | key -> | ... | obj | ... | desc | (if property exists)
| ... | obj | ... | key -> | ... | obj | ... | undefined | (if property doesn't exist)

### 要約

C API の Object.getOwnPropertyDescriptor() と同等： obj_idx にあるオブジェクトの名前付きプロパティに対するプロパティ記述子オブジェクトをプッシュします。ターゲットがオブジェクトでない場合 (またはインデックスが無効な場合) は、エラーがスローされます。

フラグはまだ定義されていないので、フラグには 0 を使用します。


### 例

```c
duk_idx_t obj_idx = /* ... */;

/* Check if property "my_prop" is a getter. */

duk_push_string(ctx, "my_prop");
duk_get_prop_desc(ctx, obj_idx, 0 /*flags*/);
if (duk_is_object(ctx, -1)) {
  /* Property found. */
  if (duk_has_prop_string(ctx, -1, "get")) {
    printf("my_prop is a getter\n");
  } else {
    printf("my_prop is not a getter\n");
  }
} else {
  printf("my_prop not found\n");
}
```

### 参照

duk_def_prop


## duk_get_prop_heapptr() 

2.2.0 property heap ptr borrowed

### プロトタイプ

```c
duk_bool_t duk_get_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... | val | (if key exists)
| ... | obj | ... | -> | ... | obj | ... | undefined | (if key doesn't exist)

### 要約

duk_get_prop() と同様ですが、プロパティ名は、例えば duk_get_heapptr() を使って取得した Duktape ヒープポインタとして与えられます。ptr が NULL の場合、undefined がキーとして使用されます。


### 例

```c
void *ptr;

duk_push_string(ctx, "propertyName");
ptr = duk_get_heapptr(ctx, -1);
/* String behind 'ptr' must remain reachable! */

(void) duk_get_prop_heapptr(ctx, -3, ptr);
printf("obj.propertyName = %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_get_prop
duk_get_prop_index
duk_get_prop_string
duk_get_prop_lstring
duk_get_prop_literal


## duk_get_prop_index() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_get_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... | val | (if key exists)
| ... | obj | ... | -> | ... | obj | ... | undefined | (if key doesn't exist)

### 要約

duk_get_prop() と同様ですが、プロパティ名は符号なし整数の arr_idx として与えられます。これは特に配列の要素にアクセスするのに便利です (しかし、それに限定されるものではありません)。

概念的には、数値はプロパティを読み込むための文字列に強制されます。例えば、123 はプロパティ名 "123" に相当します。Duktapeは可能な限り、明示的な強制を避けています。


### 例

```c
(void) duk_get_prop_index(ctx, -3, 123);
printf("obj[123] = %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_get_prop
duk_get_prop_string
duk_get_prop_lstring
duk_get_prop_literal
duk_get_prop_heapptr


## duk_get_prop_literal() 

2.3.0 property literal

### プロトタイプ

```c
duk_bool_t duk_get_prop_literal(duk_context *ctx, const char *key_literal);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... | val | (if key exists)
| ... | obj | ... | -> | ... | obj | ... | undefined | (if key doesn't exist)

### 要約

duk_get_prop() と同様ですが、プロパティ名は文字列リテラルとして与えられます (duk_push_literal() を参照してください)。


### 例

```c
(void) duk_get_prop_literal(ctx, -3, "myPropertyName");
printf("obj.myPropertyName = %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_get_prop
duk_get_prop_index
duk_get_prop_string
duk_get_prop_lstring
duk_get_prop_heapptr


## duk_get_prop_lstring() 

2.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_get_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... | val | (if key exists)
| ... | obj | ... | -> | ... | obj | ... | undefined | (if key doesn't exist)

### 要約

duk_get_prop() と同様ですが、プロパティ名は、長さを明示した文字列として与えられます。


### 例

```c
(void) duk_get_prop_lstring(ctx, -3, "internal" "\x00" "nul", 12);
printf("obj.internal[00]nul = %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_get_prop
duk_get_prop_index
duk_get_prop_string
duk_get_prop_literal
duk_get_prop_heapptr


## duk_get_prop_string() 

1.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_get_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... | val | (if key exists)
| ... | obj | ... | -> | ... | obj | ... | undefined | (if key doesn't exist)

### 要約

duk_get_prop() と同様ですが、プロパティ名は、NUL で終端する C 文字列キーとして与えられます。


### 例

```c
(void) duk_get_prop_string(ctx, -3, "propertyName");
printf("obj.propertyName = %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_get_prop
duk_get_prop_index
duk_get_prop_lstring
duk_get_prop_literal
duk_get_prop_heapptr


## duk_get_prototype() 

1.0.0 prototy pe object

### プロトタイプ

```c
void duk_get_prototype(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | val | ... | proto |

### 要約

idx にある値の内部プロトタイプを取得します。値がオブジェクトでない場合、エラーがスローされます。オブジェクトがプロトタイプを持たない場合（これは「裸のオブジェクト」に対して可能である）、代わりにundefinedがスタックにプッシュされます。


### 例

```c
/* Get the internal prototype of an object at index -3. */
duk_get_prototype(ctx, -3);
```

### 参照

duk_set_prototype


## duk_get_string() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_get_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある文字列の文字データポインタを、値を変更したり強制したりすることなく取得します。読み取り専用、NUL終端の文字列データへの非NULLポインタを返す。値が文字列でない場合、またはインデックスが無効な場合はNULLを返す。

文字列のバイト長を明示的に取得するには（文字列にNUL文字が埋め込まれている場合に有効）、 duk_get_lstring() を使用します。

これは、バッファデータポインタの扱い方とは異なります(技術的な理由による)。
シンボル値は C API では文字列として見えるので、 duk_is_symbol() と duk_is_string() の両方が真となります。この動作は、Duktape 1.xの内部文字列と同様です。duk_is_symbol()を使えば、シンボルを普通の文字列と区別することができます。内部表現については、symbols.rstを参照してください。

### 例

```c
const char *str;

str = duk_get_string(ctx, -3);
if (str) {
    printf("value is a string: %s\n", str);
}
```

### 参照

duk_get_lstring
duk_get_string_default
duk_get_lstring_default


## duk_get_string_default() 

2.1.0 string stack

### プロトタイプ

```c
const char *duk_get_string_default(duk_context *ctx, duk_idx_t idx, const char *def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_string() と同様ですが、デフォルト値が明示されており、値が文字列でない場合、あるいはインデックスが無効な場合に返されます。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトポインタの値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
const char *host = duk_get_string_default(ctx, 3, "localhost");
```

### 参照

duk_get_lstring_default


## duk_get_top() 

1.0.0 stack

### プロトタイプ

```c
duk_idx_t duk_get_top(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

現在のスタックトップ（>= 0）を取得し、（現在の活性化の）バリュースタック上の現在の値の数を示す。


### 例

```c
printf("stack top is %ld\n", (long) duk_get_top(ctx));
```


## duk_get_top_index() 

1.0.0 stack

### プロトタイプ

```c
duk_idx_t duk_get_top_index(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

スタック上の最上位値の絶対インデックス(>= 0)を取得します。スタックが空の場合、DUK_INVALID_INDEX を返します。


### 例

```c
duk_idx_t idx_top;

idx_top = duk_get_top_index(ctx);
if (idx_top == DUK_INVALID_INDEX) {
    printf("stack is empty\n");
} else {
    printf("index of top element: %ld\n", (long) idx_top);
}
```

### 参照

duk_require_top_index


## duk_get_type() 

1.0.0 stack

### プロトタイプ

```c
duk_int_t duk_get_type(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある値の型を返します。戻り値は DUK_TYPE_xxx のいずれか、あるいは idx が無効な場合は DUK_TYPE_NONE となります。

シンボル値は、C API では文字列として見えるので、 duk_is_symbol() と duk_is_string() の両方が真となります。この動作は、Duktape 1.xの内部文字列と同様です。duk_is_symbol() を使えば、シンボルを普通の文字列と区別することができます。内部表現については、symbols.rstを参照してください。

### 例

```c
if (duk_get_type(ctx, -3) == DUK_TYPE_NUMBER) {
    printf("value is a number\n");
}
```

### 参照

duk_check_type
duk_get_type_mask
duk_check_type_mask


## duk_get_type_mask() 

1.0.0 stack

### プロトタイプ

```c
duk_uint_t duk_get_type_mask(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxにある値の型マスクを返します。戻り値は DUK_TYPE_MASK_xxx のいずれかであり、 idx が無効な場合は DUK_TYPE_MASK_NONE です。

タイプマスクは、例えば、複数の型を一度に比較するのに便利です (この目的のためには、 duk_check_type_mask() 呼び出しがより便利です)。

シンボル値は C API では文字列として見えるので、 duk_is_symbol() と duk_is_string() の両方が真となります。この動作は、Duktape 1.xの内部文字列に類似しています。duk_is_symbol() を使えば、シンボルを普通の文字列と区別することができます。内部表現については、symbols.rstを参照してください。

### 例

```c
if (duk_get_type_mask(ctx, -3) & (DUK_TYPE_MASK_STRING |
                                  DUK_TYPE_MASK_NUMBER)) {
    printf("value is a string or a number\n");
}
```

### 参照

duk_get_type
duk_check_type_mask


## duk_get_uint() 

1.0.0 stack

### プロトタイプ

```c
duk_uint_t duk_get_uint(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx で数値を取得し、まず [0, DUK_UINT_MAX] の間で値をクランプし、次に 0 に向かって切り捨てることで C の duk_uint_t に変換します。スタック上の値は変更されない。値がNaNであるか、数値でないか、またはインデックスが無効である場合、0を返す。

変換の例

Input	Output
-Infinity	0
-1	0
-3.9	0
3.9	3 
DUK_UINT_MAX + 1	DUK_UINT_MAX
+Infinity	DUK_UINT_MAX
NaN	0
"123"	0 (non-number)

この強制は、例えばNaN値に対して直感的でない（そして移植性のない）動作をする可能性のある、doubleから符号なし整数への基本的なCキャストとは異なります。また、ECMAScript の ToUint32() による強制とは異なり、ネイティブな duk_uint_t の全範囲が許可されます (32ビットより多くなる可能性があります)。

### 例

```c
printf("unsigned int value: %lu\n", (unsigned long) duk_get_uint(ctx, -3));
```

### 参照

duk_get_uint_default


## duk_get_uint_default() 

2.1.0 stack

### プロトタイプ

```c
duk_uint_t duk_get_uint_default(duk_context *ctx, duk_idx_t idx, duk_uint_t def_value);
```

### スタック

| ... | val | ... |

### 要約

duk_get_uint() と同様ですが、値が数値でないか、インデックスが無効な場合に返される、明示的なデフォルト値があります。


### 例

```c
unsigned int count = (unsigned int) duk_get_uint_default(ctx, 1, 3);  /* default: 3 */
```

### 参照

duk_get_uint


## duk_has_prop() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_has_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

| ... | obj | ... | key | -> | ... | obj | ... |

### 要約

obj_idx の値が property key を持つかどうかを調べますす。 key はスタックから取り除かれます。リターンコードとエラースローの動作

プロパティが存在する場合、1 が返されます。
プロパティが存在しない場合、0 が返されます。
obj_idx の値がオブジェクトでない場合、エラーがスローされます。
obj_idx が無効の場合、エラーがスローされます。
プロパティの存在チェックは、ECMAScript の式 res = key in obj と同等です。セマンティクスについては、Property Accessors, The in operator, および [[HasProperty]] (P) を参照してください。キーは強制されます。

key 引数は、文字列または Symbol になる ToPropertyKey() 強制適用を使用して、内部で強制適用されます。配列と数値インデックスには、明示的な文字列強制を回避する内部高速パスが存在するため、該当する場合は数値キーを使用します。
ターゲットがhasトラップを実装するProxyオブジェクトである場合、トラップが呼び出され、APIコールの戻り値はトラップの戻り値に一致します。

ほとんどのプロパティ関連APIコールのように）任意のオブジェクト強制値を受け入れる代わりに、このコールはそのターゲット値としてオブジェクトのみを受け入れます。これは、ECMAScriptの演算子セマンティクスに従うため、意図的なものです。
もしキーが固定文字列なら、一つの API コールを避けて、 duk_has_prop_string() 変数を使用することができます。同様に、もしキーが配列のインデックスなら、 duk_has_prop_index() variant を使うことができます。


### 例

```c
duk_push_string(ctx, "myProperty");
if (duk_has_prop(ctx, -3)) {
    printf("obj has 'myProperty'\n");
} else {
    printf("obj does not have 'myProperty'\n");
}
```

### 参照

duk_has_prop_index
duk_has_prop_string
duk_has_prop_lstring
duk_has_prop_literal
duk_has_prop_heapptr


## duk_has_prop_heapptr() 

2.2.0 property heap ptr borrowed

### プロトタイプ

```c
duk_bool_t duk_has_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_has_prop() と同様ですが、プロパティ名は、例えば duk_get_heapptr() を使って得られた Duktape ヒープ・ポインタとして与えられます。ptr が NULL ならば、undefined がキーとして使用されます。


### 例

```c
void *ptr;

duk_push_string(ctx, "propertyName");
ptr = duk_get_heapptr(ctx, -1);
/* String behind 'ptr' must remain reachable! */

if (duk_has_prop_heapptr(ctx, -3, ptr)) {
    printf("obj has 'myProperty'\n");
} else {
    printf("obj does not have 'myProperty'\n");
}
```

### 参照

duk_has_prop
duk_has_prop_index
duk_has_prop_string
duk_has_prop_lstring
duk_has_prop_literal


## duk_has_prop_index() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_has_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_has_prop() と同様ですが、プロパティ名は符号なし整数 arr_idx として与えられます。これは特に配列要素の存在をチェックするのに便利です（ただし、これに限定されるものではありません）。

概念的には、数値はプロパティの存在をチェックするために文字列に強制され、例えば123はプロパティ名 "123 "と同じになります。Duktapeは可能な限り、明示的な強制を避けています。


### 例

```c
if (duk_has_prop_index(ctx, -3, 123)) {
    printf("obj has index 123\n");
} else {
    printf("obj does not have index 123\n");
}
```

### 参照

duk_has_prop
duk_has_prop_string
duk_has_prop_lstring
duk_has_prop_literal
duk_has_prop_heapptr


## duk_has_prop_literal() 

2.3.0 property literal

### プロトタイプ

```c
duk_bool_t duk_has_prop_literal(duk_context *ctx, duk_idx_t obj_idx, const char *key_literal);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_has_prop() と同様ですが、プロパティ名は文字列リテラルとして与えられます (duk_push_literal() を参照して下さい)。


### 例

```c
if (duk_has_prop_literal(ctx, -3, "myPropertyKey");
    printf("obj has 'myPropertyKey'\n");
} else {
    printf("obj does not have 'myPropertyKey'\n");
}
```

### 参照

duk_has_prop
duk_has_prop_index
duk_has_prop_string
duk_has_prop_lstring
duk_has_prop_heapptr


## duk_has_prop_lstring() 

2.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_has_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_has_prop() と同様ですが、プロパティ名は長さを明示した文字列として与えられます。


### 例

```c
if (duk_has_prop_lstring(ctx, -3, "internal" "\x00" "nul", 12)) {
    printf("obj has 'internal[00]nul'\n");
} else {
    printf("obj does not have 'internal[00]nul'\n");
}
```

### 参照

duk_has_prop
duk_has_prop_index
duk_has_prop_string
duk_has_prop_literal
duk_has_prop_heapptr


## duk_has_prop_string() 

1.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_has_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

duk_has_prop() のように、プロパティ名は NUL で終端する C 文字列キーとして与えられます。


### 例

```c
if (duk_has_prop_string(ctx, -3, "myProperty")) {
    printf("obj has 'myProperty'\n");
} else {
    printf("obj does not have 'myProperty'\n");
}
```

### 参照

duk_has_prop
duk_has_prop_index
duk_has_prop_lstring
duk_has_prop_literal
duk_has_prop_heapptr


## duk_hex_decode() 

1.0.0 hex codec

### プロトタイプ

```c
void duk_hex_decode(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | hex_val | ... | -> | ... | val | ... |

### 要約

16進エンコードされた値をインプレース操作でバッファにデコードします。入力が無効な場合、エラーを投げます。。


### 例

```c
duk_push_string(ctx, "7465737420737472696e67");
duk_hex_decode(ctx, -1);  /* buffer */
printf("hex decoded: %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);

/* Output:
 * hex decoded: test string
 */
```

### 参照

duk_hex_encode


## duk_hex_encode() 

1.0.0 hex codec

### プロトタイプ

```c
const char *duk_hex_encode(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | hex_val | ... |

### 要約

任意の値をバッファに取り込み，その結果をインプレース操作で16進数にエンコードします。便宜上、結果の文字列へのポインタを返します。

バッファへの強制は，まずバッファにない値を文字列に強制し，次にその文字列をバッファに強制します。結果として得られるバッファには、CESU-8エンコーディングの文字列が格納されます。

### 例

```c
duk_push_string(ctx, "foo");
printf("hex encoded: %s\n", duk_hex_encode(ctx, -1));

/* Output:
 * hex encoded: 666f6f
 */
```

### 参照

duk_hex_decode


## duk_insert() 

1.0.0 stack

### プロトタイプ

```c
void duk_insert(duk_context *ctx, duk_idx_t to_idx);
```

### スタック

| ... | old(to_idx) | ... | val | -> | ... | val(to_idx) | old | ... |

### 要約

スタックトップからポップした値でto_idxに値を挿入します。to_idxにある前の値とその上の値は、スタックの上に1ステップ移動します。to_idxが無効なインデックスの場合、エラーを投げます。。

負のインデックスは、スタックトップの値をポップする前に評価されます。これは、例でも説明されています。

### 例

```c
duk_push_int(ctx, 123);
duk_push_int(ctx, 234);
duk_push_int(ctx, 345);       /* -> [ 123 234 345 ] */
duk_push_string(ctx, "foo");  /* -> [ 123 234 345 "foo" ] */
duk_insert(ctx, -3);          /* -> [ 123 "foo" 234 345 ] */
```

### 参照

duk_pull
duk_replace


## duk_inspect_callstack_entry() 

2.0.0 stack inspect

### プロトタイプ

```c
void duk_inspect_callstack_entry(duk_context *ctx, duk_int_t level);
```

### スタック

| ... | -> | ... | info |

### 要約

コールスタック・エントリーをレベルで検査し、そのエントリーに関する Duktape 固有の内部情報を含むオブジェクトをプッシュします。level 引数は負でなければならず、バリュースタックのインデックス規則を模倣している： -1 は最も新しい（最も内側の）呼び出し、-2 はその呼び出し元、といった具合です。level 引数が無効な場合 (たとえば、現在のコールスタックの外側)、代わりに undefined がプッシュされます。

結果オブジェクトはバージョン保証の対象外なので、そのプロパティはマイナーリリースでも変更される可能性があります(パッチリリースではありません)。これは現実的な妥協点です。内部はかなり頻繁に変更されるので、バージョニングを保証しないか、内部を全く公開しないかのどちらかを選択することになります。そのため、呼び出し側のコードは、特定のフィールドのセットが利用可能であることに依存してはいけませんし、結果フィールドを解釈する際にDUK_VERSIONをチェックする必要があるかもしれません。
次の表は、現在のプロパティをまとめたものです。

プロパティ 説明
function 実行中の関数。信頼されないコードにさらされた場合、サンドボックス化の懸念があることに注意。
pc ECMAScript 関数用のプログラムカウンタ。ネイティブ関数を実行する場合は 0 です。
lineNumber ECMAScript 関数用の行番号。ネイティブ関数を実行している場合、または pc から行への変換データが利用できない場合は 0 となります。

### 例

```c
duk_inspect_callstack_entry(ctx, -1);
duk_get_prop_string(ctx, -1, "lineNumber");
printf("immediate caller is executing on line %ld\n", (long) duk_to_int(ctx, -1));
duk_pop_2(ctx);
```


## duk_inspect_value() 

2.0.0 stack inspect

### プロトタイプ

```c
void duk_inspect_value(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | val | ... | info |

### 要約

idxの値を検査し、そのオブジェクトに関するDuktape固有の内部情報を含む オブジェクトをプッシュします。バリュースタック・インデックスが無効な場合、"none "値を記述したオブジェクトをプッシュします。

結果オブジェクトはバージョン保証の対象外なので、そのプロパティはマイナー・リリースでも変更される可能性があります（パッチ・リリースは不可）。これは現実的な妥協点です。内部はかなり頻繁に変更されるので、バージョニングを保証しないか、内部を全く公開しないかのどちらかを選択することになります。そのため、呼び出し側のコードは、特定のフィールドのセットが利用可能であることに依存してはいけませんし、結果フィールドを解釈する際にDUK_VERSIONをチェックする必要があるかもしれません。
次の表は、現在のプロパティをまとめたものです。メモリのバイトサイズにはヒープオーバーヘッドは含まれておらず、使用するアロケーション関数によって0〜16バイト(またはそれ以上)の間で変化する可能性があります。

プロパティ 説明
type duktape.hのDUK_TYPE_xxxにマッチするタイプ番号。
itag 内部 DUK_TAG_xxx の定義にマッチする内部型タグ。値はバージョン間で変更される可能性があり、設定オプションと、内部でタグ付けされた値に使用されるメモリレイアウトに依存します。
hptr ヒープで割り当てられた値のためのヒープ・ポインタ。値のタイプに関連するDuktape内部ヘッダー構造体を指す。同じ値が duk_get_heapptr() から返されます。
refc 参照カウント。参照カウントは一切調整されず、duk_inspect_value()コールによる値への参照も含まれます。
class オブジェクトの場合、内部クラス番号、内部 DUK_HOBJECT_CLASS_xxx の定義に一致します。
hbytes メインヒープオブジェクトの割り当てのバイトサイズ。いくつかの値では、これが唯一の割り当てであり、他の値では追加の割り当てがあります。
pbytes オブジェクトのプロパティテーブルのバイトサイズ。プロパティテーブルには、配列部分、ハッシュ部分、およびキー/値エントリ部分が含まれる可能性があります。
bcbytes ECMAScript の関数バイトコード（命令、定数）のバイトサイズ。ある関数テンプレートの全インスタンス（クロージャ）間で共有されます。
dbytes ダイナミックバッファまたは外部バッファの現在の割り当てのバイトサイズ。外部バッファーの割り当ては、Duktapeのヒープの一部ではないことに注意してください。
esize オブジェクト・エントリ部分のサイズ（要素数）。
enext オブジェクト・エントリ部の最初の空きインデックス（＝次に使用されるプロパ ティ・スロットのインデックス）。実際には、圧縮されていない削除されたキーを持たないオブジェクトのための独自のプロパティの数に一致します。
asize オブジェクトの配列部分のサイズを要素数で表し、配列部分がない場合や配列部分が放棄されている場合（疎な配列）には 0 となります。見かけ上の配列の長さよりも大きくても小さくてもかまいません。
hsize 要素数で表したオブジェクトハッシュ部のサイズ。
tstate 内部スレッド状態、内部の DUK_HTHREAD_STATE_xxx の定義にマッチします。
variant 特定の型に対する型バリアントを識別します。文字列の場合、variant 0 は通常のヒープに割り当てられた文字列で、variant 1 は外部文字列です。バッファの場合、variant 0 は固定バッファ、1 は動的バッファ、2 は外部バッファです。

### 例

```c
duk_inspect_value(ctx, -3);
duk_get_prop_string(ctx, -1, "refc");
printf("refcount of value at -3: %ld\n", (long) duk_to_int(ctx, -1));
duk_pop_2(ctx);
```


## duk_instanceof() 

1.3.0 compare

### プロトタイプ

```c
duk_bool_t duk_instanceof(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

| ... | val1 | ... | val2 | ... |

### 要約

ECMAScript の instanceof 演算子を使用して idx1 と idx2 の値を比較します。val1がval2をインスタンス化した場合は1を、そうでない場合は0を返します。どちらかのインデックスが無効な場合はエラーを投げます。。instanceof自体も引数の型が無効な場合はエラーを投げます。。

無効なインデックスに対してエラーを投げます。動作は、例えば duk_equals() の動作とは異なり、instanceof の厳密さと一致します。例えば、rval (idx2) が呼び出し可能なオブジェクトでない場合、instanceof は TypeError を投げます。。インデックスが無効な場合にエラーを投げます。のは、instanceofの厳密性と一致します。

### 例

```c
duk_idx_t idx_val;

duk_get_global_string(ctx, "Error");

if (duk_instanceof(ctx, idx_val, -1)) {
    printf("value at idx_val is an instanceof Error\n");
}
```


## duk_is_array() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_array(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値がオブジェクトであり、ECMAScript の配列（内部クラス Array を持つ）であれば 1 を返し、そうでなければ 0 を返す。また、value が Array をラップした Proxy である場合にも 1 を返す。idx が無効な場合は、0 を返す。

この関数は、以下の ECMAScript 式が真であるとき 1 を返す。

```javascript
Object.prototype.toString.call(val) === '[object Array]'
```

### 例

```c
if (duk_is_array(ctx, -3)) {
    /* ... */
}
```


## duk_is_boolean() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値がブール値であれば 1 を、そうでなければ 0 を返す。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_boolean(ctx, -3)) {
    /* ... */
}
```


## duk_is_bound_function() 

1.0.0 stack function

### プロトタイプ

```c
duk_bool_t duk_is_bound_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値が ECMAScript の Function.prototype.bind() により生成された Function オブジェクトである場合には 1 を返し、そうでない場合には 0 を返す。

バインドされた関数は ECMAScript の概念であり、ターゲット関数を指し示し、this バインディングと 0 個以上の引数バインディングを提供します。Function.prototype.bind (thisArg [, arg1 [, arg2, ...]]) を参照のこと。


### 例

```c
if (duk_is_bound_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
duk_bool_t duk_is_buffer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がプレーンバッファである場合は1を、そうでない場合は0を返します。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_buffer(ctx, -3)) {
    /* ... */
}
```

### 参照

duk_is_buffer_data


## duk_is_buffer_data() 

2.0.0 stack buffer object buffer

### プロトタイプ

```c
duk_bool_t duk_is_buffer_data(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がプレーンバッファまたはバッファオブジェクト型であれば1を、そうでなければ0を返す。


### 例

```c
if (duk_is_buffer_data(ctx, -3)) {
    /* ... */
}
```

### 参照

duk_is_buffer


## duk_is_c_function() 

1.0.0 stack function

### プロトタイプ

```c
duk_bool_t duk_is_c_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がFunctionオブジェクトであり、C関数（Duktape/C関数）と関連付けられている場合、1を返し、そうでない場合、0を返します。


### 例

```c
if (duk_is_c_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_callable() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_callable(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が呼び出し可能であれば1を返し、そうでなければ0を返す。また、idxが無効な場合は0を返す。

現在、これは duk_is_function() と同じです。


### 例

```c
if (duk_is_callable(ctx, -3)) {
    /* ... */
}
```


## duk_is_constructable() 

2.2.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_constructable(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が構築可能であれば1を、そうでなければ0を返す。また、idxが無効な場合は0を返す。


### 例

```c
if (duk_is_constructable(ctx, -3)) {
    /* ... */
}
```


## duk_is_constructor_call() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_constructor_call(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

現在の関数がコンストラクタとして呼び出された場合（Foo() の代わりに new Foo()）、0 以外を返します。

この呼び出しにより、C関数が通常の呼び出しとコンストラクタ呼び出しで異なる動作をすることができます（多くの組み込み関数がそうです）。


### 例

```c
if (duk_is_constructor_call(ctx)) {
    printf("called as a constructor\n");
} else {
    printf("called as a normal function\n");
}
```


## duk_is_dynamic_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
duk_bool_t duk_is_dynamic_buffer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がダイナミックバッファの場合は1を、それ以外の場合は0を返す。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_dynamic_buffer(ctx, -3)) {
    /* ... */
}
```


## duk_is_ecmascript_function() 

1.0.0 stack function

### プロトタイプ

```c
duk_bool_t duk_is_ecmascript_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値が ECMAScript のソースコードからコンパイルされた Function オブジェクトであれば 1 を返し、そうでなければ 0 を返す。 idx が無効な場合も 0 を返す。

内部的にはECMAScriptの関数はバイトコードにコンパイルされ、ECMAScriptバイトコードインタプリタによって実行されます。


### 例

```c
if (duk_is_ecmascript_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_error() 

1.1.0 stack error

### プロトタイプ

```c
duk_bool_t duk_is_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある値が Error を継承している場合は 1 を、そうでない場合は 0 を返す。idx が無効な場合も、以下を返す。0.


### 例

```c
if (duk_is_error(ctx, -3)) {
    /* Inherits from Error, attempt to print stack trace */
    duk_get_prop_string(ctx, -3, "stack");
    printf("%s\n", duk_safe_to_string(ctx, -1));
    duk_pop(ctx);
}
```


## duk_is_eval_error() 

1.4.0 stack error

### プロトタイプ

```c
duk_bool_t duk_is_eval_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある値が EvalError を継承している場合は 1 を、そうでない場合は 0 を返します。 idx が無効な場合も 0 を返す。 これは、 duk_get_error_code() == DUK_ERR_EVAL_ERROR を使用するための便宜的な呼び出しです。


### 例

```c
if (duk_is_eval_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_fixed_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
duk_bool_t duk_is_fixed_buffer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が固定バッファの場合は1を、それ以外の場合は0を返す。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_fixed_buffer(ctx, -3)) {
    /* ... */
}
```


## duk_is_function() 

1.0.0 stack function

### プロトタイプ

```c
duk_bool_t duk_is_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値がオブジェクトであり、かつ関数である場合（内部クラス Function を持つ）には 1 を返し、そうでない場合には 0 を返す。 idx が無効な場合も 0 を返す。

この関数は、以下の ECMAScript 式が真であるとき、1を返す。

```javascript
Object.prototype.toString.call(val) === '[object Function]'
```

機能の具体的な種類を判断するには、以下を使用します。

- duk_is_c_function()
- duk_is_ecmascript_function()
- duk_is_bound_function()

### 例

```c
if (duk_is_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_lightfunc() 

stack

### プロトタイプ

```c
duk_bool_t duk_is_lightfunc(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある値が lightfunc であれば 1 を、そうでなければ 0 を返します。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_lightfunc(ctx, -3)) {
    /* ... */
}
```


## duk_is_nan() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_nan(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値が NaN（特殊な数値）である場合は 1 を、そうでない場合は 0 を返す。idx が無効な場合も 0 を返す。

IEEのdoublesは、多数の異なるNaN値を持っています。Duktapeは内部でNaN値を正規化することがあります。この関数は、どのような種類のNaNであっても1を返す。


### 例

```c
if (duk_is_nan(ctx, -3)) {
    /* ... */
}
```


## duk_is_null() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_null(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がNULLの場合は1を、それ以外の場合は0を返す。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_null(ctx, -3)) {
    /* ... */
}
```


## duk_is_null_or_undefined() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_null_or_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がNULLまたは未定義の場合は1を、それ以外の場合は0を返す。. idx が無効な場合も 0 を返す。

このAPIコールはECMAScriptの(x == null)比較と似ており、nullとundefinedの両方に対して真です。


### 例

```c
if (duk_is_null_or_undefined(ctx, -3)) {
    /* ... */
}
```


## duk_is_number() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が数値の場合は1を、それ以外の場合は0を返す。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_number(ctx, -3)) {
    /* ... */
}
```


## duk_is_object() 

1.0.0 stack object

### プロトタイプ

```c
duk_bool_t duk_is_object(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値がオブジェクトである場合は 1 を、そうでない場合は 0 を返す。 idx が無効な場合も 0 を返す。

多くの値がオブジェクトとみなされることに注意してください。

ECMAScript object
ECMAScript array
ECMAScript function
Duktape thread (coroutine)
Duktape internal objects

特定のオブジェクトタイプは、例えば duk_is_array() のような別の API コールでチェックすることができます。


### 例

```c
if (duk_is_object(ctx, -3)) {
    /* ... */
}
```


## duk_is_object_coercible() 

1.0.0 stack object

### プロトタイプ

```c
duk_bool_t duk_is_object_coercible(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

CheckObjectCoercible で定義されるように、idx の値がオブジェクトコアーシブルであれば 1 を返し、そうでなければ 0 を返す。 idx が無効な場合も 0 を返す。

ECMAScript のすべての型は、undefined と null 以外はオブジェクトと互換性があります。カスタムバッファとポインタ型はオブジェクト保 持可能です。


### 例

```c
if (duk_is_object_coercible(ctx, -3)) {
    /* ... */
}
```


## duk_is_pointer() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がポインタの場合は1を、それ以外の場合は0を返す。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_pointer(ctx, -3)) {
    /* ... */
}
```


## duk_is_primitive() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_primitive(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

Returns 1 if value at idx is a primitive type, as defined in ToPrimitive, otherwise returns 0. idx が無効な場合も 0 を返す。

Any standard type other than an object is a primitive type. The custom plain pointer type is also considered a primitive type. However, the custom plain buffer type (which behaves like an Uint8Array object in most situations) and lightfunc type (which behaves like a Function object in most situations) are not considered a primitive type. This matches the behavior of duk_to_primitive() which (usually) coerces e.g. a plain buffer to the string [object Uint8Array].


### 例

```c
if (duk_is_primitive(ctx, -3)) {
    /* ... */
}
```


## duk_is_range_error() 

1.4.0 stack error

### プロトタイプ

```c
duk_bool_t duk_is_range_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

Returns 1 if value at idx inherits from RangeError, otherwise returns 0. idx が無効な場合も 0 を返す。 This is a convenience call for using duk_get_error_code() == DUK_ERR_RANGE_ERROR.


### 例

```c
if (duk_is_range_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_reference_error() 

1.4.0 stack error

### プロトタイプ

```c
duk_bool_t duk_is_reference_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある値が ReferenceError を継承している場合は 1 を、そうでない場合は 0 を返します。 idx が無効な場合も 0 を返す。 これは、 duk_get_error_code() == DUK_ERR_REFERENCE_ERROR を使用するための便宜的な呼び出しです。


### 例

```c
if (duk_is_reference_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_strict_call() 

1.0.0 function

### プロトタイプ

```c
duk_bool_t duk_is_strict_call(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

現在の（Duktape/C）関数呼び出しが厳格であるか否かをチェックします。現在の関数コールがストリクトであれば1を、そうでなければ0を返します。Duktape 0.12.0以降では、この関数はユーザーコードから呼ばれると常に1を返します（コールスタックが空でも）。


### 例

```c
if (duk_is_strict_call(ctx)) {
    printf("strict call\n");
} else {
    printf("non-strict call\n");
}
```


## duk_is_string() 

1.0.0 string stack

### プロトタイプ

```c
duk_bool_t duk_is_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が文字列であれば1を、そうでなければ0を返す。

シンボル値はC APIでは文字列として見えるので、 duk_is_symbol() と duk_is_string() の両方が真となります。この動作は、Duktape 1.xの内部文字列に類似しています。duk_is_symbol() を使えば、シンボルを普通の文字列と区別することができます。内部表現については、symbols.rstを参照してください。

### 例

```c
if (duk_is_string(ctx, -3)) {
    /* ... */
}
```


## duk_is_symbol() 

2.0.0 symbol string stack

### プロトタイプ

```c
duk_bool_t duk_is_symbol(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がシンボルであれば1を、そうでなければ0を返す。

シンボル値はC APIでは文字列として見えるので、 duk_is_symbol() と duk_is_string() の両方が真となります。この動作は、Duktape 1.xの内部文字列に類似しています。duk_is_symbol() を使えば、シンボルを普通の文字列と区別することができます。内部表現については、symbols.rstを参照してください。

### 例

```c
if (duk_is_symbol(ctx, -3)) {
    /* ... */
}
```


## duk_is_syntax_error() 

1.4.0 stack error

### プロトタイプ

```c
duk_bool_t duk_is_syntax_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある値が SyntaxError を継承している場合は 1 を、そうでない場合は 0 を返します。 idx が無効な場合も 0 を返す。 これは、duk_get_error_code() == DUK_ERR_SYNTAX_ERROR を使用するための便利なコールです。


### 例

```c
if (duk_is_syntax_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_thread() 

1.0.0 thread stack

### プロトタイプ

```c
duk_bool_t duk_is_thread(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値がオブジェクトでDuktapeスレッド（コルーチン）であれば1を返し、そうでなければ0を返します。idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_thread(ctx, -3)) {
    /* ... */
}
```


## duk_is_type_error() 

1.4.0 stack error

### プロトタイプ

```c
duk_bool_t duk_is_type_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx にある値が TypeError を継承している場合は 1 を、そうでない場合は 0 を返す。 idx が無効な場合も 0 を返す。 これは、 duk_get_error_code() == DUK_ERR_TYPE_ERROR を使用する際の便宜的な呼び出しです。


### 例

```c
if (duk_is_type_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_undefined() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が未定義の場合は1を、それ以外の場合は0を返す。 idx が無効な場合も 0 を返す。


### 例

```c
if (duk_is_undefined(ctx, -3)) {
    /* ... */
}
```


## duk_is_uri_error() 

1.4.0 stack error

### プロトタイプ

```c
duk_bool_t duk_is_uri_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値が URIError を継承している場合は 1 を、そうでない場合は 0 を返す。 idx が無効な場合も 0 を返す。 これは、 duk_get_error_code() == DUK_ERR_URI_ERROR を使用する際の便宜的な呼び出しです。


### 例

```c
if (duk_is_uri_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_valid_index() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_is_valid_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

引数インデックスを検証し、インデックスが有効であれば1、そうでなければ0を返す。


### 例

```c
if (duk_is_valid_index(ctx, -3)) {
    printf("index -3 is valid\n");
} else {
    printf("index -3 is not valid\n");
}
```

### 参照

duk_require_valid_index


## duk_join() 

1.0.0 string

### プロトタイプ

```c
void duk_join(duk_context *ctx, duk_idx_t count);
```

### スタック

| ... | sep | val1 | ... | valN | -> | ... | result |

### 要約

ゼロ個以上の値を、各値の間にセパレータを付けて結果文字列に結合します。セパレータと入力値は、ToString() で自動的に強制されます。

このプリミティブは、文字列の中間的な操作の回数を最小限に抑え、手動で文字列を結合するよりも優れています。


### 例

```c
duk_push_string(ctx, "; ");
duk_push_string(ctx, "foo");
duk_push_int(ctx, 123);
duk_push_true(ctx);
duk_join(ctx, 3);

printf("result: %s\n", duk_get_string(ctx, -1));  /* "foo; 123; true" */
duk_pop(ctx);
```

### 参照

duk_concat


## duk_json_decode() 

1.0.0 json codec

### プロトタイプ

```c
void duk_json_decode(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | json_val | ... | -> | ... | val | ... |

### 要約

任意の JSON 値をインプレース操作でデコードします。入力が無効な場合、エラーを投げます。。


### 例

```c
duk_push_string(ctx, "{\"meaningOfLife\":42}");
duk_json_decode(ctx, -1);
duk_get_prop_string(ctx, -1, "meaningOfLife");
printf("JSON decoded meaningOfLife is: %s\n", duk_to_string(ctx, -1));
duk_pop_2(ctx);

/* Output:
 * JSON decoded meaningOfLife is: 42
 */
```

### 参照

duk_json_encode


## duk_json_encode() 

1.0.0 json codec

### プロトタイプ

```c
const char *duk_json_encode(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | json_val | ... |

### 要約

任意の値をインプレース操作でJSON表現にエンコードします。便宜上、結果の文字列へのポインタを返します。


### 例

```c
duk_push_object(ctx);
duk_push_int(ctx, 42);
duk_put_prop_string(ctx, -2, "meaningOfLife");
printf("JSON encoded: %s\n", duk_json_encode(ctx, -1));
duk_pop(ctx);

/* Output:
 * JSON encoded: {"meaningOfLife":42}
 */
```

### 参照

duk_json_decode


## duk_load_function() 

1.3.0 stack byte code

### プロトタイプ

```c
void duk_load_function(duk_context *ctx);
```

### スタック

| ... | bytecode | -> | ... | function |

### 要約

バイトコードを含むバッファをロードし、オリジナルのECMAScript関数を再現します（いくつかの制限付き）。バイトコードが互換性のあるDuktapeのバージョンでダンプされ、それ以降にバイトコードが変更されていないことを確認する必要があります。信頼できないソースからバイトコードをロードすると、メモリが安全でなくなり、悪用可能な脆弱性につながる可能性があります。

Duktapeのバイトコードダンプ/ロード、サポートされている機能、既知の制限に関するより詳細な情報は、bytecode.rstを参照してください。Duktapeバイトコード・フォーマットは難読化を意図したものではないので、難読化についての注意を参照してください。

### 例

```c
duk_eval_string(ctx, "(function helloWorld() { print('hello world'); })");
duk_dump_function(ctx);
/* stack top now contains a buffer containing helloWorld bytecode */

duk_load_function(ctx);  /* [ ... bytecode ] -> [ ... function ] */
duk_call(ctx, 0);
duk_pop(ctx);
```

### 参照

duk_dump_function


## duk_map_string() 

1.0.0 string

### プロトタイプ

```c
void duk_map_string(duk_context *ctx, duk_idx_t idx, duk_map_char_function callback, void *udata);
```

### スタック

| ... | val | ... | -> | ... | val | ... |

### 要約

idxの文字列を処理し、文字列の各コードポイントに対してコールバックを呼び出す。コールバックは、引数 udata とコードポイントを与えられ、置き換えられたコードポイントを返します。成功した場合、置換されたコードポイントからなる新しい文字列が元の文字列を置き換えます。値が文字列でない場合、またはインデックスが無効な場合は、エラーを投げます。


### 例

```c
static duk_codepoint_t map_char(void *udata, duk_codepoint_t codepoint) {
    /* Convert ASCII to uppercase. */
    if (codepoint >= (duk_codepoint_t) 'a' && codepoint <= (duk_codepoint_t) 'z') {
        return codepoint - (duk_codepoint_t) 'a' + (duk_codepoint_t) 'A';
    }
    return codepoint;
}

duk_push_string(ctx, "test_string");
duk_map_string(ctx, -1, map_char, NULL);
printf("result: %s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_decode_string


## duk_new() 

1.0.0 object call

### プロトタイプ

```c
void duk_new(duk_context *ctx, duk_idx_t nargs);
```

### スタック

| ... | constructor | arg1 | ... | argN | -> | ... | retval |

### 要約

コンストラクタ関数をnargsの引数で呼び出します（関数自体を除く）。関数とその引数は、単一の戻り値に置き換えられます。コンストラクタの呼び出し中に投げられたエラーは、自動的に捕捉されません。

このバインディングのターゲット関数は、新しく作成された空のオブジェクトに設定されます。constructor.prototype がオブジェクトの場合は、新しいオブジェクトの内部プロトタイプがその値に設定され、そうでない場合は標準の組み込みオブジェクトプロトタイプが内部プロトタイプとして使用されます。ターゲット関数の戻り値は、コンストラクタ呼び出しの結果が何であるかを決定します。コンストラクタがオブジェクトを返す場合は、新しい空のオブジェクトに置き換えられます。そうでない場合は、新しい空のオブジェクト（コンストラクタによって変更される可能性があります）が返されます。[コンストラクタ]()を参照してください。


### 例

```c
/* Assume target function is already on stack at func_idx.
 * Equivalent to ECMAScript 'new func("foo", 123)'.
 */
duk_idx_t func_idx = /* ... */;

duk_dup(ctx, func_idx);
duk_push_string(ctx, "foo");
duk_push_int(ctx, 123);
duk_new(ctx, 2);  /* [ ... func "foo" 123 ] -> [ ... res ] */
printf("result is object: %d\n", (int) duk_is_object(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_pnew


## duk_next() 

1.0.0 property object

### プロトタイプ

```c
duk_bool_t duk_next(duk_context *ctx, duk_idx_t enum_idx, duk_bool_t get_value);
```

### スタック

| ... | enum | ... | -> | ... | enum | ... | (if enum empty; function returns zero)
| ... | enum | ... | -> | ... | enum | ... | key | (if enum not empty and get_value == 0; function returns non-zero)
| ... | enum | ... | -> | ... | enum | ... | key | value | (if enum not empty and get_value != 0; function returns non-zero)

### 要約

duk_enum() で作成した列挙体から、次のキー（とオプションで値）を取得します。列挙が使い果たされた場合、スタックには何もプッシュされず、この関数は 0 を返します。そうでなければ、キーがスタックに押され、 get_value が0でなければ値がそれに続き、この関数は0以外の値を返す。

値を取得すると，ゲッターを呼び出したり，Proxyトラップを発生させたりして，任意の副作用が発生する可能性があることに注意してください（エラーを投げます。可能性もあります）．


### 例

```c
while (duk_next(ctx, enum_idx, 1)) {
    printf("key=%s, value=%s\n", duk_to_string(ctx, -2), duk_to_string(ctx, -1));
    duk_pop_2(ctx);
}
```

### 参照

duk_enum


## duk_normalize_index() 

1.0.0 stack

### プロトタイプ

```c
duk_idx_t duk_normalize_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

現在のフレームの底を基準として、引数のインデックスを正規化します。結果のインデックスは 0 以上となり、後のスタックの変更に影響されません。入力インデックスが無効な場合、DUK_INVALID_INDEX を返します。無効なインデックスに対してエラーを投げます。ことを望む場合は、 duk_require_normalize_index() を使用してください。


### 例

```c
duk_idx_t idx = duk_normalize_index(ctx, -3);
```

### 参照

duk_require_normalize_index


## duk_opt_boolean() 

2.1.0 stack

### プロトタイプ

```c
duk_bool_t duk_opt_boolean(duk_context *ctx, duk_idx_t idx, duk_bool_t def_value);
```

### スタック

| ... | val | ... |

### 要約

idx にある boolean 値を、値を変更したり強制したりすることなく取得します。 値が真のとき1、偽のとき0を返す。値が未定義の場合，あるいはインデックスが無効な場合，def_value デフォルト値が返されます。その他の場合（ヌルまたは非マッチ型）は，エラーを投げます。。


### 例

```c
duk_bool_t flag_xyz = duk_opt_boolean(ctx, 2, 1);  /* default: true */
```


## duk_opt_buffer() 

2.1.0 stack buffer

### プロトタイプ

```c
void *duk_opt_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

| ... | val | ... |

### 要約

idxにある(プレーン)バッファ値のデータポインタを，値を変更したり強制したりすることなく取得します。値が0以外のサイズの有効なバッファである場合、非NULLポインタを返します。サイズが0のバッファの場合、NULLまたは非NULLポインタを返すことがあります。out_size が非NULLの場合、バッファのサイズは *out_size に書き込まれます。値が未定義またはインデックスが無効な場合，def_ptr のデフォルト値が返され， def_len のデフォルト長が *out_size に書き込まれる (out_size が nonNULL の場合)．その他の場合(NULLおよび型が一致しない場合)は，エラーを投げます。。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトポインタ値は、Duktape によって追跡されない。例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作らない。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc で割り当てられた文字列の場合、呼び出し側は duk_opt_string() から返されたポインタが libc で割り当てられた文字列の寿命以上に使用されないようにしなければなりません。
サイズゼロのバッファと非バッファを、戻り値だけから区別する信頼できる方法はありません。非バッファの場合、サイズ0のNULLが返されます。サイズ0のバッファの場合、同じ値が返されるかもしれません（NULLでないポインタが返される可能性もありますが）。duk_is_buffer() や duk_is_buffer_data() 、あるいはバッファやバッファ・オブジェクトの型チェックを行う際に使用してください。

### 例

```c
void *ptr;
duk_size_t sz;
char buf[256];

/* Use a buffer given at index 2, or default to 'buf'. */
ptr = duk_opt_buffer(ctx, 2, &sz, (void *) buf, sizeof(buf));
printf("buf=%p, size=%lu\n", ptr, (unsigned long) sz);
```

### 参照

duk_opt_buffer_data


## duk_opt_buffer_data() 

2.1.0 stack buffer object buffer

### プロトタイプ

```c
void *duk_opt_buffer_data(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

| ... | val | ... |

### 要約

idxにあるプレーンバッファまたはバッファオブジェクト(ArrayBuffer, Node.js Buffer, DataView, TypedArray view)の値のデータポインタを、値を変更したり強制したりすることなく取得します。値がゼロでないサイズの有効なバッファである場合、非NULLポインタを返します。サイズが0のバッファの場合、NULLまたは非NULLポインタを返すことがあります。out_size が非NULLの場合、バッファのサイズは *out_size に書き込まれます。値が未定義またはインデックスが無効な場合，def_ptr のデフォルト値が返され， def_len のデフォルト長が *out_size に書き込まれる (out_size が nonNULL の場合)．それ以外の場合(NULLや型が一致しない場合)はエラーを投げます。。また、値がバッファオブジェクトで、その「バッキングバッファ」がバッファオブジェクトの見かけのサイズを完全にカバーしていない場合にもスローされます。

戻り値のポインタと長さで示されるデータ領域は、プレーンバッファの場合は完全なバッファであり、バッファオブジェクトの場合はアクティブな「スライス」です。返される長さは (要素数ではなく) バイト数で表現され、常に ptr[0] から ptr[len - 1] にアクセスできるようになります。例については duk_get_buffer_data() を参照してください。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトのポインタ値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
void *ptr;
duk_size_t sz;
char buf[256];

/* Use a buffer or buffer object given at index 2, or default to 'buf'. */
ptr = duk_opt_buffer_data(ctx, 2, &sz, (void *) buf, sizeof(buf));
```

### 参照

duk_opt_buffer


## duk_opt_c_function() 

2.1.0 stack function

### プロトタイプ

```c
duk_c_function duk_opt_c_function(duk_context *ctx, duk_idx_t idx, duk_c_function def_value);
```

### スタック

| ... | val | ... |

### 要約

Duktape/C関数に関連付けられたECMAScript関数オブジェクトから、Duktape/C関数ポインタ（duk_c_function）を取得します。もし値が未定義であるか、インデックスが無効であれば、def_value デフォルト値が返されます。その他の場合（ヌルまたは非マッチ型）は、エラーを投げます。。


### 例

```c
duk_c_function funcptr;

/* Native callback, default to nop_callback. */
funcptr = duk_opt_c_function(ctx, -3, nop_callback);
```


## duk_opt_context() 

2.1.0 stack borrowed

### プロトタイプ

```c
duk_context *duk_opt_context(duk_context *ctx, duk_idx_t idx, duk_context *def_value);
```

### スタック

| ... | val | ... |

### 要約

idxにあるDuktapeスレッドのコンテキスト・ポインタを取得します。値が未定義であるか、インデックスが無効な場合、def_value デフォルト値が返されます。その他の場合（NULLまたは型が一致しない）には、エラーを投げます。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルト・ポインタ値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
duk_context *target_ctx;

target_ctx = duk_opt_context(ctx, 2, default_ctx);
```


## duk_opt_heapptr() 

2.1.0 stack heap ptr borrowed

### プロトタイプ

```c
void *duk_opt_heapptr(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

| ... | val | ... |

### 要約

idxにあるDuktapeヒープに割り当てられた値(オブジェクト、バッファ、文字列)への借用された void * 参照を取得します。値が未定義であるか、インデックスが無効である場合、def_value デフォルト値が返されます。その他の場合(NULLまたは型が一致しない)はエラーを投げます。。

duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルト・ポインタ値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
void *ptr;

ptr = duk_opt_heapptr(ctx, 2, default_ptr);
```


## duk_opt_int() 

2.1.0 stack

### プロトタイプ

```c
duk_int_t duk_opt_int(duk_context *ctx, duk_idx_t idx, duk_int_t def_value);
```

### スタック

| ... | val | ... |

### 要約

idx に数値を取得し、まず [DUK_INT_MIN, DUK_INT_MAX] の間で値をクランプし、次にゼロに向かって切り捨てることで C の duk_int_t に変換します。スタック上の値は変更されません。値が未定義であるかインデックスが無効である場合、def_value デフォルト値が返されます。その他の場合（ヌルまたは非マッチ型）は，エラーを投げます。。


### 例

```c
int port = (int) duk_opt_int(ctx, 1, 80);  /* default: 80 */
```


## duk_opt_lstring() 

2.1.0 string stack

### プロトタイプ

```c
const char *duk_opt_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len, const char *def_ptr, duk_size_t def_len);
```

### スタック

| ... | val | ... |

### 要約

idx にある文字列の文字データポインタと長さを、値を変更したり強制したりすることなく取得します。読み取り専用でNUL終端の文字列データへの非NULLポインタを返し、文字列のバイト長を *out_len に書き込む (out_len が非NULLの場合)。値が未定義またはインデックスが無効な場合、def_ptr のデフォルト値が返され、 def_len のデフォルト長が *out_len に書き込まれる (out_len が NULL でない場合)。その他の場合(NULLまたは型が一致しない場合)はエラーを投げます。。

これは、バッファデータポインタの扱い方とは異なります(技術的な理由による)。
duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトポインタの値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc に割り当てられた文字列である場合、呼び出し側は、 duk_opt_string() が返すポインタが libc に割り当てられた文字列の寿命を越えて使用されないようにしなければなりません。

### 例

```c
const char *str;
duk_size_t len;

str = duk_opt_lstring(ctx, -3, &len, "foo" "\x00" "bar", 7);
```

### 参照

duk_opt_string


## duk_opt_number() 

2.1.0 stack

### プロトタイプ

```c
duk_double_t duk_opt_number(duk_context *ctx, duk_idx_t idx, duk_double_t def_value);
```

### スタック

| ... | val | ... |

### 要約

idxにある数値の値を、値を変更したり強制したりすることなく取得します。値が未定義であるか、インデックスが無効である場合、def_value デフォルト値が返されます。その他の場合（ヌルまたは非マッチ型）は、エラーを投げます。。


### 例

```c
double backoff_multiplier = (double) duk_opt_number(ctx, 2, 1.5);  /* default: 1.5 */
```


## duk_opt_pointer() 

2.1.0 stack

### プロトタイプ

```c
void *duk_opt_pointer(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

| ... | val | ... |

### 要約

idxのポインタ値を、値を変更したり強制したりすることなく、void * として取得します。値が未定義であるか、インデックスが無効である場合、def_value デフォルト値が返されます。その他の場合（ヌルまたは非マッチ型）は，エラーを投げます。。


### 例

```c
void *ptr;

ptr = duk_opt_pointer(ctx, -3, (void *) 0x12345678);
printf("my pointer is: %p\n", ptr);
```


## duk_opt_string() 

2.1.0 string stack

### プロトタイプ

```c
const char *duk_opt_string(duk_context *ctx, duk_idx_t idx, const char *def_ptr);
```

### スタック

| ... | val | ... |

### 要約

idx にある文字列の文字データポインタを、値を変更したり強制したりすることなく取得します。読み取り専用でNUL終端の文字列データへのNULLでないポインタを返す。値が未定義であるか、インデックスが無効である場合、def_ptrのデフォルト値が返されます。その他の場合（NULLまたは型が一致しない場合）は，エラーを投げます。。

文字列のバイト長を明示的に得るには（文字列に NUL 文字が埋め込まれている場合に有効）、 duk_opt_lstring() を使用します。

これは、バッファデータポインタの扱い方とは異なります(技術的な理由による)。
duk_opt_xxx() と duk_get_xxx_default() に与えられたデフォルトポインタの値は、Duktape によって追跡されません、例えば、 duk_opt_string() は、デフォルト文字列引数のコピーを作成しません。呼び出し側は、デフォルト・ポインタがその意図された用途に有効であり続けることを保証する責任があります。例えば、 duk_opt_string(ctx, 3, "localhost") は、文字列定数が常に有効であるため、問題なく動作しますが、引数が libc で割り当てられた文字列の場合、呼び出し側は duk_opt_string() から返されたポインタが libc で割り当てられた文字列の寿命以上に使われないようにする必要があります。
シンボル値は C API では文字列として見えるので、 duk_is_symbol() と duk_is_string() の両方が真となります。この動作は、Duktape 1.xの内部文字列と同様です。duk_is_symbol()を使えば、シンボルを普通の文字列と区別することができます。内部表現については、symbols.rstを参照してください。

### 例

```c
const char *host = duk_opt_string(ctx, 3, "localhost");
```

### 参照

duk_opt_lstring


## duk_opt_uint() 

2.1.0 stack

### プロトタイプ

```c
duk_uint_t duk_opt_uint(duk_context *ctx, duk_idx_t idx, duk_uint_t def_value);
```

### スタック

| ... | val | ... |

### 要約

idx で数値を取得し、まず [0, DUK_UINT_MAX] の間で値をクランプし、次に 0 に向かって切り捨てることで C の duk_uint_t に変換します。スタック上の値は変更されません。値が未定義であるかインデックスが無効である場合、def_value デフォルト値が返されます。その他の場合（ヌルまたは非マッチ型）は，エラーを投げます。。


### 例

```c
unsigned int count = (unsigned int) duk_opt_uint(ctx, 1, 3);  /* default: 3 */
```


## duk_pcall() 

1.0.0 protected call

### プロトタイプ

```c
duk_int_t duk_pcall(duk_context *ctx, duk_idx_t nargs);
```

### スタック

| ... | func | arg1 | ... | argN | -> | ... | retval | (if success, return value == 0)
| ... | func | arg1 | ... | argN | -> | ... | err | (if failure, return value != 0)

### 要約

ターゲット関数funcをnargsの引数で呼び出す（関数自体を除く）。関数とその引数は、単一の戻り値または単一のエラー値で置き換えられます。関数呼び出し中に投げられたエラーはキャッチされます。

戻り値はduk_exec_success (0)です。呼び出しが成功した場合、nargs 引数は単一の戻り値で置き換えられます。(この戻り値コード定数はゼロであることが保証されているので、「ゼロかゼロでないか」のチェックで成功を確認することができる)。
DUK_EXEC_ERROR: 呼び出しに失敗、nargs 引数は単一のエラー値で置き換えられます。(例外的に、例えばバリュースタック上の引数が少なすぎる場合、呼び出しは投げます。かもしれない)。
ほとんどのDuktape APIコールとは異なり、このコールは成功時にゼロを返します。これにより、複数のエラー・コードを後で定義することができます。
捕捉されたエラー・オブジェクトは通常Errorのインスタンスで、.stack、.fileName、.lineNumberなどの有用なプロパティを持ちます。これらは通常のプロパティメソッドでアクセスすることができます。しかし、任意の値が投げられる可能性があるので、常にそうであると仮定することは避けるべきです。

このバインディングの対象関数は、初期状態では未定義に設定されています。ターゲット関数が厳密でない場合、関数が呼び出される前にバインディングはグローバルオブジェクトに置き換えられます。関数コードの入力を参照してください。このバインディングを制御したい場合は、代わりに duk_pcall_method() または duk_pcall_prop() を使用することができます。


### 例

```c
/* Assume target function is already on stack at func_idx; the target
 * function adds arguments and returns the result.
 */

duk_idx_t func_idx;
duk_int_t rc;

/* Basic example: function call and duk_safe_to_string() error print. */

duk_dup(ctx, func_idx);
duk_push_int(ctx, 2);
duk_push_int(ctx, 3);
rc = duk_pcall(ctx, 2);  /* [ ... func 2 3 ] -> [ 5 ] */
if (rc == DUK_EXEC_SUCCESS) {
    printf("2+3=%ld\n", (long) duk_get_int(ctx, -1));
} else {
    /* Coercing with duk_safe_to_string() is a useful default, but you can
     * also look up e.g. the .stack property of the error.
     */
    printf("error: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);

/* Accessing .stack to print a stack trace if the value caught is an
 * Error instance.
 */

duk_dup(ctx, func_idx);
duk_push_int(ctx, 2);
duk_push_int(ctx, 3);
rc = duk_pcall(ctx, 2);  /* [ ... func 2 3 ] -> [ 5 ] */
if (rc == DUK_EXEC_SUCCESS) {
    printf("2+3=%ld\n", (long) duk_get_int(ctx, -1));
} else {
    if (duk_is_error(ctx, -1)) {
        /* Accessing .stack might cause an error to be thrown, so wrap this
         * access in a duk_safe_call() if it matters.
         */
        duk_get_prop_string(ctx, -1, "stack");
        printf("error: %s\n", duk_safe_to_string(ctx, -1));
        duk_pop(ctx);
    } else {
        /* Non-Error value, coerce safely to string. */
        printf("error: %s\n", duk_safe_to_string(ctx, -1));
    }
}
duk_pop(ctx);
```

### 参照

duk_pcall_method
duk_pcall_prop


## duk_pcall_method() 

1.0.0 protected call

### プロトタイプ

```c
duk_int_t duk_pcall_method(duk_context *ctx, duk_idx_t nargs);
```

### スタック

| ... | func | this | arg | ... | argN | -> | ... | retval | (if success, return value == 0)
| ... | func | this | arg | ... | argN | -> | ... | err | (if failure, return value != 0)

### 要約

duk_pcall() と同様であるが、ターゲット関数 func は、バリュースタック上で明示的に与えられた this バインディングで呼び出されます。


### 例

```c
/* The target function here prints:
 *
 *    this: 123
 *    2 3
 *
 * and returns 5.
 */

duk_int_t rc;

duk_push_string(ctx, "(function(x,y) { print('this:', this); "
                     "print(x,y); return x+y; })");
duk_eval(ctx);  /* -> [ ... func ] */
duk_push_int(ctx, 123);
duk_push_int(ctx, 2);
duk_push_int(ctx, 3);
rc = duk_pcall_method(ctx, 2);  /* [ ... func 123 2 3 ] -> [ 5 ] */
if (rc == DUK_EXEC_SUCCESS) {
    printf("2+3=%ld\n", (long) duk_get_int(ctx, -1));  /* prints 5 */
} else {
    printf("error: %s\n", duk_to_string(ctx, -1));
}
duk_pop(ctx);
```


## duk_pcall_prop() 

1.0.0 protected property call

### プロトタイプ

```c
duk_int_t duk_pcall_prop(duk_context *ctx, duk_idx_t obj_idx, duk_idx_t nargs);
```

### スタック

| ... | obj | ... | key arg1 | ... | argN | -> | ... | obj | ... | retval | (if success, return value == 0)
| ... | obj | ... | key arg1 | ... | argN | -> | ... | obj | ... | err | (if failure, return value != 0)

### 要約

duk_pcall() と同様ですが、ターゲット関数は obj.key から検索され、obj は関数の this バインディングとして使用されます。


### 例

```c
/* obj.myAdderMethod(2,3) -> 5 */
duk_idx_t obj_idx;
duk_int_t rc;

duk_push_string(ctx, "myAdderMethod");
duk_push_int(ctx, 2);
duk_push_int(ctx, 3);
rc = duk_pcall_prop(ctx, obj_idx, 2);  /* [ ... "myAdderMethod" 2 3 ] -> [ ... 5 ] */
if (rc == DUK_EXEC_SUCCESS) {
    printf("2+3=%ld\n", (long) duk_get_int(ctx, -1));
} else {
    printf("error: %s\n", duk_to_string(ctx, -1));
}
duk_pop(ctx);
```


## duk_pcompile() 

1.0.0 protected compile

### プロトタイプ

```c
duk_int_t duk_pcompile(duk_context *ctx, duk_uint_t flags);
```

### スタック

| ... | source | filename | -> | ... | function | (if success, return value == 0)
| ... | source | filename | -> | ... | err | (if failure, return value != 0)

### 要約

duk_compile() と同様であるが、コンパイルに関連するエラー (ソースのシンタックスエラーなど) を捕捉します。0 の返り値は成功を示し、コンパイルされた関数はスタックトップに残されます。0 以外の返り値はエラーを示し、そのエラーはスタックトップに残されます。

スタックトップが低すぎる (2 より小さい) 場合は、エラーがスローされます。


### 例

```c
duk_push_string(ctx, "print('program'); syntax error here=");
duk_push_string(ctx, "hello-with-syntax-error");
if (duk_pcompile(ctx, 0) != 0) {
    printf("compile failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    duk_call(ctx, 0);      /* [ func ] -> [ result ] */
    printf("program result: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);
```

### 参照

duk_compile
duk_pcompile_string
duk_pcompile_string_filename
duk_pcompile_lstring
duk_pcompile_lstring_filename


## duk_pcompile_lstring() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_pcompile_lstring(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

| ... | -> | ... | function | (if success, return value == 0)
| ... | -> | ... | err | (if failure, return value != 0)

### 要約

duk_pcompile() と同様であるが、コンパイル入力は長さを明示した C 文字列として与えられます。この関数に関連するファイル名は "input" です。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

if (duk_pcompile_lstring(ctx, 0, src, len) != 0) {
    printf("compile failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    duk_call(ctx, 0);      /* [ func ] -> [ result ] */
    printf("program result: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);
```


## duk_pcompile_lstring_filename() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_pcompile_lstring_filename(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

| ... | filename | -> | ... | function | (if success, return value == 0)
| ... | filename | -> | ... | err | (if failure, return value != 0)

### 要約

duk_pcompile() と同様ですが、コンパイル入力は、長さを明示した C 文字列として与えられます。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_push_string(ctx, "myFile.js");
if (duk_pcompile_lstring_filename(ctx, 0, src, len) != 0) {
    printf("compile failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    duk_call(ctx, 0);      /* [ func ] -> [ result ] */
    printf("program result: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);
```


## duk_pcompile_string() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_pcompile_string(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

| ... | -> | ... | function | (if success, return value == 0)
| ... | -> | ... | err | (if failure, return value != 0)

### 要約

duk_pcompile() と同様であるが、コンパイル時の入力は C の文字列として与えられます。この関数に関連するファイル名は "input" です。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
if (duk_pcompile_string(ctx, 0, "print('program code'); syntax error here=") != 0) {
    printf("compile failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    duk_call(ctx, 0);      /* [ func ] -> [ result ] */
    printf("program result: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);
```


## duk_pcompile_string_filename() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_pcompile_string_filename(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

| ... | filename | -> | ... | function | (if success, return value == 0)
| ... | filename | -> | ... | err | (if failure, return value != 0)

### 要約

duk_pcompile() と同様ですが、コンパイル入力は C の文字列として与えられます。

この変種では、入力ソースコードは Duktape によって インターンされないので、低メモリ環境では便利です。

### 例

```c
duk_push_string(ctx, "myFile.js");
if (duk_pcompile_string_filename(ctx, 0, "print('program code'); syntax error here=") != 0) {
    printf("compile failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    duk_call(ctx, 0);      /* [ func ] -> [ result ] */
    printf("program result: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);
```


## duk_peval() 

1.0.0 protected compile

### プロトタイプ

```c
duk_int_t duk_peval(duk_context *ctx);
```

### スタック

| ... | source | -> | ... | result | (if success, return value == 0)
| ... | source | -> | ... | err | (if failure, return value != 0)

### 要約

duk_eval() と同様ですが、コンパイルに関連するエラー (ソースのシンタックスエラーなど) を捕捉します。0 の返り値は成功を表し、eval の結果はスタックトップに残されます。0 以外の返り値はエラーを示し、そのエラーはスタックトップに残されます。

スタックトップが低すぎる (1より小さい) 場合は、エラーがスローされます。


### 例

```c
duk_push_string(ctx, "print('Hello world!'); syntax error here=");
if (duk_peval(ctx) != 0) {
    printf("eval failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    printf("result is: %s\n", duk_safe_to_string(ctx, -1));
}
duk_pop(ctx);
```

### 参照

duk_eval
duk_peval_noresult
duk_peval_string
duk_peval_string_noresult
duk_peval_lstring
duk_peval_lstring_noresult


## duk_peval_lstring() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_peval_lstring(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

| ... | -> | ... | result | (if success, return value == 0)
| ... | -> | ... | err | (if failure, return value != 0)

### 要約

duk_peval() と同様ですが、eval の入力は、長さを明示した C 文字列として与えられます。一時的なeval関数に関連するファイル名は "eval "です。

このバリエーションでは、入力ソースコードは Duktape によって インターナルされないので、低メモリ環境では便利です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

if (duk_peval_lstring(ctx, src, len) != 0) {
    printf("eval failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    printf("result is: %s\n", duk_get_string(ctx, -1));
}
duk_pop(ctx);
```

### 参照

duk_peval_lstring_noresult


## duk_peval_lstring_noresult() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_peval_lstring_noresult(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

| ... | -> | ... |

### 要約

duk_peval_lstring() と同様ですが、(成功/エラーの結果に関わらず) バリュースタックに結果を残しません。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

if (duk_peval_lstring_noresult(ctx, src, len) != 0) {
    printf("eval failed\n");
} else {
    printf("eval successful\n");
}
```


## duk_peval_noresult() 

1.0.0 protected compile

### プロトタイプ

```c
duk_int_t duk_peval_noresult(duk_context *ctx);
```

### スタック

| ... | source | -> | ... |

### 要約

Like duk_peval() but leaves no result on the value stack (regardless of success/error result).


### 例

```c
duk_push_string(ctx, "print('Hello world!'); syntax error here=");
if (duk_peval_noresult(ctx) != 0) {
    printf("eval failed\n");
} else {
    printf("eval successful\n");
}
```

### 参照

duk_peval_string_noresult
duk_peval_lstring_noresult


## duk_peval_string() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_peval_string(duk_context *ctx, const char *src);
```

### スタック

| ... | -> | ... | result | (if success, return value == 0)
| ... | -> | ... | err | (if failure, return value != 0)

### 要約

duk_peval() と同様ですが、eval の入力は C の文字列として与えられます。一時的なeval関数に関連するファイル名は "eval "です。

このバリエーションでは、入力ソースコードは Duktape によって インターンされないので、低メモリ環境では便利です。

### 例

```c
if (duk_peval_string(ctx, "'testString'.toUpperCase()") != 0) {
    printf("eval failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    printf("result is: %s\n", duk_get_string(ctx, -1));
}
duk_pop(ctx);
```

### 参照

duk_peval_string_noresult


## duk_peval_string_noresult() 

1.0.0 string protected compile

### プロトタイプ

```c
duk_int_t duk_peval_string_noresult(duk_context *ctx, const char *src);
```

### スタック

| ... | -> | ... |

### 要約

duk_peval_string() と同様であるが、(成功/エラー結果に関わらず) バリュースタックに結果を残さない。

この変種では、入力ソースコードは Duktape によってインターンされないので、 低メモリ環境では有用です。

### 例

```c
if (duk_peval_string_noresult(ctx, "print('testString'.toUpperCase());") != 0) {
    printf("eval failed\n");
} else {
    printf("eval successful\n");
}
```


## duk_pnew() 

1.3.0 protected object call

### プロトタイプ

```c
duk_ret_t duk_pnew(duk_context *ctx, duk_idx_t nargs);
```

### スタック

| ... | constructor | arg1 | ... | argN | -> | ... | retval | (if success, return value == 0)
| ... | constructor | arg1 | ... | argN | -> | ... | err | (if failure, return value != 0)

### 要約

duk_new() と同様ですが、エラーを捕捉します。0 の返り値は成功を示し、コンストラクタの結果はスタックトップに残されます。0 ではない返り値はエラーを示し、そのエラーはスタックトップに残されます。

スタックトップが低すぎる (nargs + 1 より小さい) 場合、または nargs が負である場合、エラーがスローされます。


### 例

```c
duk_ret_t rc;

/* Protected call to: new MyConstructor("foo", 123) */
duk_eval_string(ctx, "MyConstructor");
duk_push_string(ctx, "foo");
duk_push_int(ctx, 123);
rc = duk_pnew(ctx, 2);  /* [ ... func "foo" 123 ] -> [ ... res ] */
if (rc != 0) {
    printf("failed: %s\n", duk_safe_to_string(ctx, -1));
} else {
    printf("success\n");
}
duk_pop(ctx);
```

### 参照

duk_new


## duk_pop() 

1.0.0 stack

### プロトタイプ

```c
void duk_pop(duk_context *ctx);
```

### スタック

| ... | val | -> | ... |

### 要約

スタックから要素を1つ取り出します。スタックが空の場合、エラーを投げます。。

複数の要素をポップするには、duk_pop_n() または、よくある場合のショートカットである duk_pop_2() と duk_pop_3() を使用します。


### 例

```c
duk_pop(ctx);
```

### 参照

duk_pop_2
duk_pop_3
duk_pop_n


## duk_pop_2() 

1.0.0 stack

### プロトタイプ

```c
void duk_pop_2(duk_context *ctx);
```

### スタック

| ... | val1 | val2 | -> | ... |

### 要約

スタックから2つの要素を取り出します。スタックの要素が2つより少ない場合は、エラーを投げます。。


### 例

```c
duk_pop_2(ctx);
```


## duk_pop_3() 

1.0.0 stack

### プロトタイプ

```c
void duk_pop_3(duk_context *ctx);
```

### スタック

| ... | val1 | val2 | val3 | -> | ... |

### 要約

スタックから3つの要素を取り出します。スタックの要素が3つより少ない場合は、エラーを投げます。。


### 例

```c
duk_pop_3(ctx);
```


## duk_pop_n() 

1.0.0 stack

### プロトタイプ

```c
void duk_pop_n(duk_context *ctx, duk_idx_t count);
```

### スタック

| ... | val1 | ... | valN | -> | ... |

### 要約

スタックから count 個の要素を取り出します。スタックの要素が count より少ない場合、エラーを投げます。。count が 0 の場合、この呼び出しは失敗です。負のカウントは、エラーをスローします。


### 例

```c
duk_pop_n(ctx, 10);
```


## duk_pull() 

2.5.0 stack

### プロトタイプ

```c
void duk_pull(duk_context *ctx, duk_idx_t from_idx);
```

### スタック

| ... | val | ... | -> | ... | ... | val |

### 要約

from_idxの値を削除し、バリュースタックの先頭にプッシュします。

from_idxが無効なインデックスの場合、エラーを投げます。。

### 例

```c
duk_push_int(ctx, 123);
duk_push_int(ctx, 234);
duk_push_int(ctx, 345);       /* -> [ 123 234 345 ] */
duk_pull(ctx, -2);            /* [ 123 234 345 ] -> [ 123 345 234 ] */
```

### 参照

duk_insert
duk_remove


## duk_push_array() 

1.0.0 stack object

### プロトタイプ

```c
duk_idx_t duk_push_array(duk_context *ctx);
```

### スタック

| ... | -> | ... | arr |

### 要約

空の配列をスタックにプッシュします。押された配列の非負のインデックス（スタックの底からの相対値）を返します。

生成されたオブジェクトの内部プロトタイプは Array.prototype です。これを変更するには、 duk_set_prototype() を使用します。


### 例

```c
duk_idx_t arr_idx;

arr_idx = duk_push_array(ctx);
duk_push_string(ctx, "foo");
duk_put_prop_index(ctx, arr_idx, 0);
duk_push_string(ctx, "bar");
duk_put_prop_index(ctx, arr_idx, 1);

/* array is now: [ "foo", "bar" ], and array.length is 2 (automatically
 * updated for ECMAScript arrays).
 */

duk_pop(ctx);  /* pop array */
```


## duk_push_bare_array() 

2.4.0 stack object

### プロトタイプ

```c
duk_idx_t duk_push_bare_array(duk_context *ctx);
```

### スタック

| ... | -> | ... | arr |

### 要約

duk_push_array() と似ていますが、押された配列は他のオブジェクトを継承していません、つまり、その内部プロトタイプは null です。押された配列の非負のインデックス（スタックの底からの相対位置）を返します。


### 例

```c
duk_idx_t arr_idx;

arr_idx = duk_push_bare_array(ctx);
```

### 参照

duk_push_array
duk_push_bare_object


## duk_push_bare_object() 

2.0.0 stack object

### プロトタイプ

```c
duk_idx_t duk_push_bare_object(duk_context *ctx);
```

### スタック

| ... | -> | ... | obj |

### 要約

duk_push_object() と似ていますが、プッシュされたオブジェクトは他のオブジェクトを継承していません、つまり、その内部プロトタイプはヌルです。この呼び出しは、Object.create(null) と同じです。プッシュされたオブジェクトの非負のインデックス（スタックの底からの相対値）を返します。


### 例

```c
duk_idx_t obj_idx;

obj_idx = duk_push_bare_object(ctx);
```

### 参照

duk_push_object
duk_push_bare_array


## duk_push_boolean() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_boolean(duk_context *ctx, duk_bool_t val);
```

### スタック

| ... | -> | ... | true | (if val != 0)
| ... | -> | ... | false | (if val == 0)

### 要約

真（val != 0の場合）または偽（val == 0の場合）をスタックにプッシュします。


### 例

```c
duk_push_boolean(ctx, 0);  /* -> [ ... false ] */
duk_push_boolean(ctx, 1);  /* -> [ ... false true ] */
duk_push_boolean(ctx, 123);  /* -> [ ... false true true ] */
```


## duk_push_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_push_buffer(duk_context *ctx, duk_size_t size, duk_bool_t dynamic);
```

### スタック

| ... | -> | ... | buf |

### 要約

サイズbyteの新しいバッファを割り当て、それをバリュースタックにプッシュします。ゼロサイズのバッファの場合、NULLまたは非NULLを返します。バッファデータ領域は自動的にゼロになります。dynamicが0でない場合，バッファはサイズ変更可能であり，そうでなければバッファは固定サイズになります。割り当てに失敗した場合は，エラーをスローします。

duk_push_fixed_buffer() と duk_push_dynamic_buffer() というショートカットも存在します。

動的バッファは、内部的に 2 つのメモリ割り当てを必要とします: 1 つはバッファヘッダ用、もう 1 つは現在割り当てられているデータエリア用です。固定バッファでは、バッファヘッダの後にデータ領域が続くため、1つのメモリ割り当てで済みます。
NULLデータ・ポインタはエラーではありませんし、呼び出し元のコードを混乱させることもありません。
Duktape は、割り当てられたバッファ・データの自動ゼロ化機能を無効にする設定オプショ ンを付けてコンパイルすることができます（ゼロ化機能はデフォルトです）。この場合、必要に応じて手動でバッファをゼロにする必要があります。

### 例

```c
/* Allocate a fixed buffer of 1024 bytes.  There is no need to check for
 * the return value: an error is thrown if allocation fails.
 */

void *p;

p = duk_push_buffer(ctx, 1024, 0);
printf("allocated buffer, data area: %p\n", p);
```

### 参照

duk_push_fixed_buffer
duk_push_dynamic_buffer
duk_push_external_buffer


## duk_push_buffer_object() 

1.3.0 stack buffer object

### プロトタイプ

```c
void duk_push_buffer_object(duk_context *ctx, duk_idx_t idx_buffer, duk_size_t byte_offset, duk_size_t byte_length, duk_uint_t flags);
```

### スタック

| ... | buffer | ... | -> | ... | buffer | ... | bufobj | (when creating an ArrayBuffer or a view)
| ... | ArrayBuffer | ... | -> | ... | ArrayBuffer | ... | bufobj | (when creating a view)

### 要約

新しいバッファオブジェクトまたはバッファビューオブジェクトをプッシュします。基本となるプレーンなバッファまたは ArrayBuffer (ビューを作成するときに受け取ります) が、インデックス idx_buffer で提供されます。バッファまたはビューのタイプは flags で与えられます (例: DUK_BUFOBJ_UINT16ARRAY)。バッファから使用されるアクティブな範囲または「スライス」は byte_offset と byte_length によって示されます。

利用可能なバッファの種類は以下の通りです。

定義 バッファ/ビュータイプ

DUK_BUFOBJ_NODEJS_BUFFER	Buffer (Node.js), a Uint8Array inheriting from Buffer.prototype
DUK_BUFOBJ_ARRAYBUFFER	ArrayBuffer
DUK_BUFOBJ_DATAVIEW	DataView
DUK_BUFOBJ_INT8ARRAY	Int8Array
DUK_BUFOBJ_UINT8ARRAY	Uint8Array
DUK_BUFOBJ_UINT8CLAMPEDARRAY	Uint8ClampedArray
DUK_BUFOBJ_INT16ARRAY	Int16Array
DUK_BUFOBJ_UINT16ARRAY	Uint16Array
DUK_BUFOBJ_INT32ARRAY	Int32Array
DUK_BUFOBJ_UINT32ARRAY	Uint32Array
DUK_BUFOBJ_FLOAT32ARRAY	Float32Array
DUK_BUFOBJ_FLOAT64ARRAY	Float64Array

ArrayBuffer 以外のものを作成し、引数として与えられたバッキングバッファがプレーンバッファである場合、ビューをバッキングする ArrayBuffer が自動的に作成されます。これは、ビューオブジェクトのbufferプロパティからアクセス可能です。ArrayBufferの内部byteOffsetは0になり、ArrayBufferのインデックスbyteOffsetはビューのインデックス0に一致します。ArrayBufferのbyteLengthは、ビューの範囲がArrayBufferに対して有効であるように、byte_offset + byte_lengthになるでしょう。

ArrayBufferを作成するとき、byte_offset引数は0にすることを強くお勧めします。さもなければ、ArrayBuffer上で構築されたビューの外部.byteOffsetプロパティは、誤解を招くでしょう：その値は、ArrayBufferに対する相対値ではなく、ArrayBufferの下にあるプレーンバッファに対する相対値となります。ゼロのbyte_offsetでは、2つのオフセットの間に違いはありません。

基礎となるプレーンバッファーは通常 byte_offset と byte_length 引数で示される範囲をカバーすべきですが、そうでない場合でもメモリ安全性は保証されます。例えば、基礎となるバッファの外側の値を読み込もうとすると、0が返されます。バッファオブジェクトの作成時に、意図的に基礎となるバッファのサイズをチェックしません: 作成時にバッファがバイト範囲を完全にカバーしていたとしても、後でサイズが変更されるかもしれません。


### 例

```c
/* Map byte range [100,150[ of plain buffer at idx_plain_buf into a
 * Uint16Array object which will have the following properties:
 *
 *   - length: 25             (length in Uint16 elements)
 *   - byteLength: 50         (length in bytes)
 *   - byteOffset: 100        (byte offset to start of slice)
 *   - BYTES_PER_ELEMENT: 2   (Uint16)
 *
 * The Uint16Array's .buffer property will be an ArrayBuffer with the
 * following properties:
 *
 *   - byteLength: 200
 *   - internal byteOffset: 0
 */

duk_push_buffer_object(ctx, idx_plain_buf, 100, 50, DUK_BUFOBJ_UINT16ARRAY);
```

### 参照

duk_push_buffer
duk_push_fixed_buffer
duk_push_dynamic_buffer
duk_push_external_buffer


## duk_push_c_function() 

1.0.0 stack function

### プロトタイプ

```c
duk_idx_t duk_push_c_function(duk_context *ctx, duk_c_function func, duk_idx_t nargs);
```

### スタック

| ... | -> | ... | func |

### 要約

C関数に関連付けられた新しい関数オブジェクトをスタックにプッシュします。関数オブジェクトは ECMAScript の関数オブジェクトで、呼び出されると、func は Duktape/C の関数インターフェイスを使って呼び出されます。プッシュされた関数の負でないインデックス（スタックの底からの相対値）を返します。

nargs 引数は、func が入力されたときにバリュースタックがどのように見えるかを制御します。

nargs が >= 0 であれば、それは関数が期待する引数の正確な数を示します。余分な引数は捨てられ、足りない引数は未定義値で埋められます。関数に入ると、バリュースタックの先頭は常にnargsと一致します。
nargs が DUK_VARARGS に設定されている場合、バリュースタックは実際の（可変）呼び出し引数を含み、関数は duk_get_top() で実際の引数カウントをチェックする必要があります。
作成された関数は、通常の関数 (func()) としても、コンストラクタ (new func()) としても呼び出すことが可能です。この 2 つの呼び出し方は、 duk_is_constructor_call() を使って区別することができます。この関数はコンストラクタとして使用できますが、ECMAScript 関数のような自動的なプロトタイププロパティを持ちません。

押された関数をコンストラクタとして使用するつもりであれば、通常、プロトタイプ・オブジェクトを作成し、関数のプロトタイプ・プロパティを手動で設定する必要があります。

### 例

```c
duk_ret_t my_addtwo(duk_context *ctx) {
    double a, b;

    /* Here one can expect that duk_get_top(ctx) == 2, because nargs
     * for duk_push_c_function() is 2.
     */

    a = duk_get_number(ctx, 0);
    b = duk_get_number(ctx, 1);
    duk_push_number(ctx, a + b);
    return 1;   /*  1 = return value at top
                 *  0 = return 'undefined'
                 * <0 = throw error (use DUK_RET_xxx constants)
                 */
}

void test(void) {
    duk_idx_t func_idx;

    func_idx = duk_push_c_function(ctx, my_addtwo, 2);
    duk_push_int(ctx, 2);
    duk_push_int(ctx, 3);  /* -> [ ... func 2 3 ] */
    duk_call(ctx, 2);      /* -> [ ... res ] */
    printf("2+3 is %ld\n", (long) duk_get_int(ctx, -1));
    duk_pop(ctx);
}
```

### 参照

duk_push_c_lightfunc


## duk_push_c_lightfunc() 

 stack light func function

### プロトタイプ

```c
duk_idx_t duk_push_c_lightfunc(duk_context *ctx, duk_c_function func, duk_idx_t nargs, duk_idx_t length, duk_int_t magic);
```

### スタック

| ... | -> | ... | func |

### 要約

C 関数に関連付けられた新しい lightfunc 値をスタックにプッシュします。プッシュされた lightfunc の非負のインデックス (スタックの底に相対的) を返します。

lightfunc は、Duktape/C 関数ポインタと、関連するヒープ割り当てのない小さな内部制御フラグのセットを含むタグ付き値です。内部制御フラグは、nargs、length、および magic 値をエンコードし、それゆえ、重要な制限を持ちます。

nargsは[0,14]またはDUK_VARARGSでなければなりません。
length は [0,15] でなければならず、lightfunc の仮想長プロパティにマップされます。
magic は、[-128,127] でなければなりません。
lightfunc は、独自のプロパティを保持できず、仮想の名前と長さのプロパティのみを持ち、その他のプロパティは Function.prototype から継承されます。

nargs 引数は、func が入力されたときにバリュースタックがどのように見えるかを制御し、通常の Duktape/C 関数のように動作します（ duk_push_c_function() を参照してください）。

作成された関数は、通常の関数 (func()) としても、コンストラクタ (new func()) としても呼び出すことが可能です。この 2 つの呼び出し方は duk_is_constructor_call() を使って区別することができます。この関数はコンストラクタとして使用できますが、通常の Function オブジェクトのようにプロトタイププロパティを持つことはできません。

プッシュされた lightfunc をコンストラクタとして使用するつもりで、（Object.prototype の代わりに）カスタム プロトタイプ オブジェクトを使用したい場合、lightfunc はオブジェクト値を返さなければなりません。このオブジェクトは、コンストラクタのために自動的に作成されたデフォルトのインスタンス（これにバインドされている）を置き換え、新しい MyLightFunc() 式の値となります。

### 例

```c
duk_idx_t func_idx;

func_idx = duk_push_c_lightfunc(ctx, my_addtwo, 2 /*nargs*/, 2 /*length*/, 0 /*magic*/);
```

### 参照

duk_push_c_function


## duk_push_context_dump() 

1.0.0 stack debug

### プロトタイプ

```c
void duk_push_context_dump(duk_context *ctx);
```

### スタック

| ... | -> | ... | str |

### 要約

コンテキスト ctx の現在のアクティブ化の状態を要約した一行文字列をプッシュします。これは Duktape/C コードのデバッグに有用であり、実稼働環境での使用は意図していない。

正確なダンプ内容はバージョンに依存します。現在のフォーマットでは、スタック・トップ（スタック上の要素数）を含み、現在の要素をJXフォーマット（Duktapeのカスタム拡張JSONフォーマット）の値の配列としてプリントアウトします。以下の例では、次のようなものがプリントされます。

```sh
ctx: top=2, stack=[123,"foo"]
```

プロダクションコードにダンプコールを残しておくべきではありません。

### 例

```c
duk_push_int(ctx, 123);
duk_push_string(ctx, "foo");
duk_push_context_dump(ctx);
printf("%s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```


## duk_push_current_function() 

1.0.0 stack function

### プロトタイプ

```c
void duk_push_current_function(duk_context *ctx);
```

### スタック

| ... | -> | ... | func | (if current function exists)
| ... | -> | ... | undefined | (if no current function)

### 要約

現在実行中の関数をスタックにプッシュします。プッシュされる値は ECMAScript Function オブジェクトです。現在実行中の関数がない場合、代わりに undefined がプッシュされます。

現在の関数が 1 つ以上のバインド関数や Proxy オブジェクトを介して呼び出されていた場合、この呼び出しから返される関数は最終的に解決された関数です（バインド関数やProxy ではありません）。

この関数により、C関数はその関数オブジェクトにアクセスすることができます。複数の関数オブジェクトは内部的に同じC関数を指すことができるので、関数オブジェクトは関数のパラメータ化のための便利な場所であり、内部の状態の隠し場所として機能することもできます。


### 例

```c
duk_push_current_function(ctx);
```


## duk_push_current_thread() 

1.0.0 thread stack function borrowed

### プロトタイプ

```c
void duk_push_current_thread(duk_context *ctx);
```

### スタック

| ... | -> | ... | thread | (if current thread exists)
| ... | -> | ... | undefined | (if no current thread)

### 要約

現在実行中の Duktape スレッドをスタックにプッシュします。プッシュされる値はスレッドオブジェクトであり、ECMAScript オブジェクトでもあります。現在のスレッドがない場合、代わりに undefined がプッシュされます。

現在のスレッドは (ほとんど常に) ctx ポインタによって表されるスレッドです。

スレッドに関連付けられた duk_context * を取得するには、 duk_get_context() を使用します。


### 例

```c
duk_push_thread(ctx);
```


## duk_push_dynamic_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_push_dynamic_buffer(duk_context *ctx, duk_size_t size);
```

### スタック

| ... | -> | ... | buf |

### 要約

動的な (サイズ変更可能な) バッファを割り当て、それをバリュースタックにプッシュします。dynamic = 1 での duk_push_buffer() のショートカットです。


### 例

```c
void *p;

p = duk_push_dynamic_buffer(ctx, 1024);
printf("allocated buffer, data area: %p\n", p);
```


## duk_push_error_object() 

1.0.0 stack object error

### プロトタイプ

```c
duk_idx_t duk_push_error_object(duk_context *ctx, duk_errcode_t err_code, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

新しいエラーオブジェクトを作成し、それをバリュースタックにプッシュします（エラーはスローされません）。プッシュされたエラーオブジェクトの非負のインデックス（スタックの底からの相対値）を返します。

エラーオブジェクトの message プロパティには、fmt と残りの引数を使用して sprintf フォーマットの文字列が設定されます。作成されたエラーオブジェクトの内部プロトタイプは err_code に基づいて選択されます。例えば、DUK_ERR_RANGE_ERROR は、組み込みの RangeError プロトタイプが使用されるようにします。ユーザーエラーコードの有効範囲は [1,16777215] です。


### 例

```c
duk_idx_t err_idx;

err_idx = duk_push_error_object(ctx, DUK_ERR_TYPE_ERROR, "invalid argument value: %d", arg_value);
```

### 参照

duk_push_error_object_va


## duk_push_error_object_va() 

1.1.0 vararg stack object error

### プロトタイプ

```c
duk_idx_t duk_push_error_object_va(duk_context *ctx, duk_errcode_t err_code, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

duk_push_error_object() の Vararg 変形です。

この API 呼び出しは、呼び出し側のコードの va_end() マクロが (例えばエラースローのために) 到達しないかもしれないので、完全に移植性があるわけではありません。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存しています; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
duk_idx_t my_type_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;
    duk_idx_t err_idx;

    va_start(ap, fmt);
    err_idx = duk_push_error_object_va(ctx, DUK_ERR_TYPE_ERROR, fmt, ap);
    va_end(ap);

    return err_idx;
}
```

### 参照

duk_push_error_object


## duk_push_external_buffer() 

1.3.0 stack buffer

### プロトタイプ

```c
void duk_push_external_buffer(duk_context *ctx);
```

### スタック

| ... | -> | ... | buf |

### 要約

外部バッファを割り当て、それをバリュースタックにプッシュします。外部バッファとは、Duktape がメモリ管理していない、ユーザが割り当てた外部バッファを指します。最初の外部バッファ・ポインタは NULL で、サイズは 0 です。duk_config_buffer() を使って、外部バッファのポインタとサイズを更新することができます。

外部バッファは ECMAScript コードが外部で割り当てられたデータ構造にアクセスできるようにするのに便利です。例えば、外部で割り当てられたフレームバッファにバイトを書き込むために使用することができます。外部バッファにはアライメント要件がありません（Duktapeはアクセスする際にアライメントの仮定を行いません）。


### 例

```c
void *p;
duk_size_t len;

/* Allocate a frame buffer from a hypothetical graphics library.
 * The frame buffer is allocated outside of Duktape.
 */
p = allocate_frame_buffer(1920 /*width*/, 1080 /*height*/);
len = 1920 * 1080 * 4;

/* Create an external buffer pointing to the frame buffer. */
duk_push_external_buffer(ctx);
duk_config_buffer(ctx, -1, p, len);

/* ECMAScript code with access to the buffer object could now do
 * something like:
 *
 *    buf[100] = 123;
 *
 * Calling code must make sure ECMAScript never reads/writes an
 * external buffer whose backing buffer is no longer valid.
 */
```

### 参照

duk_config_buffer


## duk_push_false() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_false(duk_context *ctx);
```

### スタック

| ... | -> | ... | false |

### 要約

false をスタックにプッシュします。duk_push_boolean(ctx, 0)を呼ぶのと同じです。


### 例

```c
duk_push_false(ctx);
```


## duk_push_fixed_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_push_fixed_buffer(duk_context *ctx, duk_size_t size);
```

### スタック

| ... | -> | ... | buf |

### 要約

固定サイズのバッファを割り当てて、それをバリュースタックにプッシュします。dynamic = 0 での duk_push_buffer() のショートカットです。


### 例

```c
void *p;

p = duk_push_fixed_buffer(ctx, 1024);
printf("allocated buffer, data area: %p\n", p);
```


## duk_push_global_object() 

1.0.0 stack object

### プロトタイプ

```c
void duk_push_global_object(duk_context *ctx);
```

### スタック

| ... | -> | ... | global |

### 要約

グローバルオブジェクトをスタックにプッシュします。


### 例

```c
duk_push_global_object(ctx);
```


## duk_push_global_stash() 

1.0.0 stash stack sandbox object module

### プロトタイプ

```c
void duk_push_global_stash(duk_context *ctx);
```

### スタック

| ... | -> | ... | stash |

### 要約

グローバルスタッシュオブジェクトをスタックにプッシュします。グローバルスタッシュは内部オブジェクトで、C コードからキー/値のペアを保存してガベージコレクションに到達できるようにするために使用できますが、ECMAScript コードからアクセスすることはできません。スタッシュは、同じグローバルオブジェクトに関連付けられた ctx 引数を持つ C コードからしかアクセスできません。


### 例

```c
duk_ret_t set_timer_callback(duk_context *ctx) {
    duk_push_global_stash(ctx);
    duk_dup(ctx, 0);  /* timer callback */
    duk_put_prop_string(ctx, -2, "timerCallback");
    return 0;
}
```

### 参照

duk_push_heap_stash
duk_push_thread_stash


## duk_push_heap_stash() 

1.0.0 stash stack sandbox object module

### プロトタイプ

```c
void duk_push_heap_stash(duk_context *ctx);
```

### スタック

| ... | -> | ... | stash |

### 要約

ヒープスタッシュオブジェクトをスタックにプッシュします。ヒープ・スタッシュは内部オブジェクトで、Cコードからキー/値のペアを保存してガベージコレクションに到達できるようにするために使用できますが、ECMAScriptコードからアクセスすることはできません。スタッシュはCコードからのみアクセス可能で、同じDuktapeヒープを共有する全てのコードで同じスタッシュオブジェクトが使用されます（同じグローバルオブジェクトを共有しない場合でも）。


### 例

```c
duk_push_heap_stash(ctx);
```

### 参照

duk_push_global_stash
duk_push_thread_stash


## duk_push_heapptr() 

1.1.0 stack object heap ptr borrowed

### プロトタイプ

```c
duk_idx_t duk_push_heapptr(duk_context *ctx, void *ptr);
```

### スタック

| ... | -> | ... | obj | (if ptr != NULL)
| ... | -> | ... | undefined | (if ptr == NULL)

### 要約

Duk_get_heapptr() またはその亜種から借用したポインタ参照を使って、 Duktape ヒープオブジェクトをバリュースタックにプッシュします。ptr が NULL の場合、undefined がプッシュされます。

呼び出し側は、引数 ptr がプッシュされたときにまだ有効である (Duktape ガベージコレクションによって解放されていない) ことを確認する責任があります。これを怠ると、予測不可能でメモリが安全でない動作になります。

これを確実にするために、2つの基本的な方法があります。

強力な後方参照：関連するヒープ・オブジェクトが、duk_get_heapptr() と duk_push_heapptr() の間で Duktape ガベージコレクションのために常に到達可能であることを保証します。例えば、借用した void ポインタが使用されている間に、関連するオブジェクトがスタッ シュに書き込まれたことを確認します。要するに、アプリケーションは、強く参照される値によってバックされる借用された参照を保持しているのです。これは、Duktape 2.1以前でサポートされていた唯一の方法です。
弱い参照＋ファイナライザー：関連するヒープ・オブジェクトのために（できればネイティブの）ファイナライザーを追加し、遅くともファイナライザーが呼ばれた時点で（オブジェクトがファイナライザーによって救出されないと仮定して）voidポインターの使用を停止します。Duktape 2.1以降では、到達不可能なオブジェクトに対して、ファイナライザーを呼び出すまで、duk_push_heapptr()が許可されるようになりました。この方法によって、アプリケーションは関連するオブジェクトへの弱い参照を保持することができます。ただし、以下の制限事項を参照してください。
ファイナライザーの呼び出しは、メモリ不足などで静かに失敗することがあります。ポインタの有効期間の終了を示すためにファイナライザコールに依存している場合、 ファイナライザコールを逃すと duk_push_heapptr() にぶら下がったポインタが与えられる可能性があります。この状況に対する回避策は今のところありません。したがって、もしメモリ不足の状態が予想されるなら、ファイナライザに基づくアプローチは確実に機能しないかもしれません。今後の課題は、少なくともネイティブファイナライザ関数へのエントリーが成功しないとオブジェクトが解放されないようにすることです。https://github.com/svaarala/duktape/issues/1456 を参照してください。

### 例

```c
void *ptr;
duk_idx_t idx;

/* 'ptr' originally obtained using duk_get_heapptr() earlier: */

idx = duk_push_heapptr(ctx, ptr);
```

### 参照

duk_get_heapptr
duk_require_heapptr


## duk_push_int() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_int(duk_context *ctx, duk_int_t val);
```

### スタック

| ... | -> | ... | val |

### 要約

valをIEEEダブルに変換し、スタックにプッシュします。

これは duk_push_number(ctx, (duk_double_t) val) を呼び出すための省略記法です。


### 例

```c
duk_push_int(ctx, 123);
```


## duk_push_literal() 

2.3.0 stack literal

### プロトタイプ

```c
const char *duk_push_literal(duk_context *ctx, const char *str_literal);
```

### スタック

| ... | -> | ... | str |

### 要約

C 言語のリテラルをスタックにプッシュし、インターンした文字列データ領域へのポインタを返す（引数リテラルと同じであってもなくてもよい）。引数str_literal。

は、例えばsizeof(str)を使って操作できる、 "foo "のような非NULLのCリテラルでなければなりません。sizeof()は、自動的なNULターミネータを含む文字列長（例えば、 sizeof("foo") は4）を生成します。
データ内容が不変であること。すなわち、読み取り専用メモリ、または書き込み可能なメモリであるが、変更されないことが保証されていること。
内部のNUL文字を含んではならず、Cの文字列と同様に終端NULを持たなければならない。
引数str_literalはAPIマクロにより複数回評価される場合があります。str_literalの引数に対して、実行時のNULLポインタチェックは行われませんので、NULLを渡すとメモリセーフでない動作になります。

このコールは概念的には duk_push_string() と同じです。呼び出し側のコードは、フットプリントや速度のわずかな違いが問題となる場合にのみ、これを使用する必要があります。不変のCリテラルが持つ特性により、Duktape内部でちょっとした最適化が可能です。

文字列の長さは、コンパイル時に sizeof(str_literal) - 1 を使って、呼び出し側で計算することができます。
デフォルトでは、Cリテラル（duk_push_literal()またはduk_get_prop_literal()のようなリテラル便利コール）を介してアクセスされるヒープ文字列は、次のマーク＆スイープラウンドまで自動的に固定され、Cリテラルアドレスを固定された内部ヒープ文字列にマップするためのルックアップキャッシュが存在します。この最適化では、文字列の重複排除（一般的ですが、保証されていません）を想定していません。
文字列データは不変であると仮定しているので、内部文字列表現はコピーを作成する代わりにデータを指すだけでよいのです。(Duktape 2.5では、この最適化は行われていません）。
入力文字列が内部に NUL 文字を含む可能性がある場合、代わりに duk_push_lstring() を使ってください。duk_push_literal() での埋め込み NUL の扱いは設定オプションに依存し、呼び出し側のコードは 決してその挙動に依存してはいけません。


### 例

```c
/* Basic case. */
duk_push_literal(ctx, "foo");

/* Argument may involve compile time concatenation and parentheses. */
duk_push_literal(ctx, ("foo" "bar"));

/* Argument may also be e.g. DUK_HIDDEN_SYMBOL() which produces a literal. */
duk_push_literal(ctx, DUK_HIDDEN_SYMBOL("mySymbol"));
```


## duk_push_lstring() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_push_lstring(duk_context *ctx, const char *str, duk_size_t len);
```

### スタック

| ... | -> | ... | str |

### 要約

明示的な長さの文字列をスタックにプッシュします。文字列は、内部の NUL 文字を含む任意のデータを含むことができます。内部文字列データへのポインタが返されます。操作に失敗した場合は，エラーを投げます。。

str が NULL の場合、len に関係なく空の文字列がスタックに押され、空の文字列への非 NULL ポインタが返されます。返されたポインタは再参照可能であり、NUL終端文字が保証されます。この動作は、意図的に duk_push_string と異なっています。

Cコードは通常、有効なCESU-8文字列のみをスタックにプッシュすべきです。


### 例

```c
const char tmp1[5] = { 'f', '\0', '\0', 'x', 'y' };
const char tmp2[1] = { '\0' };

duk_push_lstring(ctx, tmp1, 5);   /* push the string "f\x00\x00xy" */
duk_push_lstring(ctx, tmp2, 1);   /* push the string "\x00" */
duk_push_lstring(ctx, tmp2, 0);   /* push empty string */
duk_push_lstring(ctx, NULL, 0);   /* push empty string */
duk_push_lstring(ctx, NULL, 10);  /* push empty string */
```


## duk_push_nan() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_nan(duk_context *ctx);
```

### スタック

| ... | -> | ... | NaN |

### 要約

NaN (not-a-number) をスタックにプッシュします。


### 例

```c
duk_push_nan(ctx);

printf("NaN is number: %d\n", (int) duk_is_number(ctx, -1));
```


## duk_push_new_target() 

2.3.0 stack function

### プロトタイプ

```c
void duk_push_new_target(duk_context *ctx);
```

### スタック

| ... | -> | ... | undefined | (if no current function or not a constructor call)
| ... | -> | ... | func | (if current function call is a constructor call)

### 要約

現在実行中の関数の new.target に相当する値をスタックにプッシュします。現在の呼び出しがコンストラクタ呼び出しでない場合、またはコールスタックが空の場合、プッシュされる値は未定義です。


### 例

```c
duk_push_new_target(ctx);
```


## duk_push_null() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_null(duk_context *ctx);
```

### スタック

| ... | -> | ... | null |

### 要約

nullをスタックにプッシュします。


### 例

```c
duk_push_null(ctx);
```


## duk_push_number() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_number(duk_context *ctx, duk_double_t val);
```

### スタック

| ... | -> | ... | val |

### 要約

数値（IEEE double）valをスタックにプッシュします。

val が NaN の場合、他の NaN 形式に正規化される場合があります。


### 例

```c
duk_push_number(ctx, 123.0);
```


## duk_push_object() 

1.0.0 stack object

### プロトタイプ

```c
duk_idx_t duk_push_object(duk_context *ctx);
```

### スタック

| ... | -> | ... | obj |

### 要約

空のオブジェクトをスタックにプッシュします。押されたオブジェクトの非負のインデックス（スタックの底からの相対値）を返します。

作成されたオブジェクトの内部プロトタイプは Object.prototype です。これを変更するには、 duk_set_prototype() を使用します。


### 例

```c
duk_idx_t obj_idx;

obj_idx = duk_push_object(ctx);
duk_push_int(ctx, 42);
duk_put_prop_string(ctx, obj_idx, "meaningOfLife");

/* object is now: { "meaningOfLife": 42 } */

duk_pop(ctx);  /* pop object */
```

### 参照

duk_push_bare_object


## duk_push_pointer() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_pointer(duk_context *ctx, void *p);
```

### スタック

| ... | -> | ... | ptr |

### 要約

pをポインタ値としてスタックに押し込んでください。Duktapeはこのポインタを何ら解釈しません。


### 例

```c
struct mystruct *p = /* ... */;

duk_push_pointer(ctx, (void *) p);
```


## duk_push_proxy() 

2.2.0 stack object

### プロトタイプ

```c
duk_idx_t duk_push_proxy(duk_context *ctx, duk_uint_t proxy_flags);
```

### スタック

| ... | target | handler | -> | ... | proxy |

### 要約

new Proxy(target, handler)に相当する、ターゲットとハンドラのテーブルに対する 新しいProxyオブジェクトをバリュースタックにプッシュします。proxy_flagsは現在(Duktape 2.5まで)未使用です。


### 例

```c
duk_idx_t proxy_idx;

duk_push_object(ctx);  /* target */
duk_push_object(ctx);  /* handler */
duk_push_c_function(ctx, my_get, 3);  /* 'get' trap */
duk_put_prop_string(ctx, -2, "get");
proxy_idx = duk_push_proxy(ctx, 0);  /* [ target handler ] -> [ proxy ] */
```


## duk_push_sprintf() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_push_sprintf(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | str |

### 要約

sprintf()のように（ただし安全に）文字列をフォーマットし、その結果をバリュースタックにプッシュします。結果の文字列へのNULLでないポインタを返します。

fmt が NULL の場合、空文字列がスタックにプッシュされ、空文字列への非 NULL ポインタが返される (この動作は、少なくとも Linux では、NULL フォーマット文字列に対する sprintf() の動作に類似している)。返されたポインタは再参照可能で、NULターミネータ文字が保証されます。

sprintf()とは異なり、文字列のフォーマットは結果の長さに関して安全です。具体的には、この実装は、一時的にフォーマットされた値に対して十分な大きさのバッファが見つかるまで、一時的なバッファのサイズを大きくしてみます。

いくつかの書式指定子にはプラットフォーム特有の動作があるかもしれません。例えば、引数がNULLの場合の%sの動作は公式には未定義であり、実際の動作はプラットフォームによって異なり、メモリ的に安全でない可能性さえあります。

### 例

```c
duk_push_sprintf(ctx, "meaning of life: %d, name: %s", 42, "Zaphod");
```


## duk_push_string() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_push_string(duk_context *ctx, const char *str);
```

### スタック

| ... | -> | ... | str | (if str != NULL)
| ... | -> | ... | null | (if str == NULL)

### 要約

C 言語の文字列をスタックに格納します。文字列の長さは、strlen()に相当するもので自動的に検出される（つまり、最初の NUL 文字を探す）。インターナルされた文字列データへのポインタが返されます。操作に失敗した場合は、エラーを投げます。。

strがNULLの場合、ECMAScriptのNULLがスタックにプッシュされ、NULLが返されます。この動作は、意図的にduk_push_lstringと異なっています。

Cコードは通常、有効なCESU-8文字列のみをスタックにプッシュすべきです。無効な CESU-8/UTF-8 バイト列の中には、Symbol 値を表すような特別な用途のために予約されているものがあります。このような無効なバイト列をプッシュすると、バリュースタック上の値は C コードでは文字列のように振る舞いますが、ECMAScript コードでは Symbol として表示されます。詳しくは、シンボルを参照してください。
入力文字列が内部 NUL 文字を含む可能性がある場合、代わりに duk_push_lstring() を使用します。


### 例

```c
duk_push_string(ctx, "foo");
duk_push_string(ctx, "foo\0bar");  /* push "foo", not "foo\0bar" */
duk_push_string(ctx, "");          /* push empty string */
duk_push_string(ctx, NULL);        /* push 'null' */
```


## duk_push_this() 

1.0.0 stack function

### プロトタイプ

```c
void duk_push_this(duk_context *ctx);
```

### スタック

| ... | -> | ... | this |

### 要約

現在実行中の C 関数の this バインディングをスタックにプッシュします。


### 例

```c
duk_push_this(ctx);
```


## duk_push_thread() 

1.0.0 thread stack borrowed

### プロトタイプ

```c
duk_idx_t duk_push_thread(duk_context *ctx);
```

### スタック

| ... | -> | ... | thr |

### 要約

新しい Duktape スレッド (コンテキスト、コルーチン) をスタックにプッシュします。プッシュされたスレッドの負でないインデックス(スタックの底からの相対値)を返します。新しいスレッドは、引数 ctx と同じ Duktape ヒープに関連付けられ、同じグローバルオブジェクト環境を共有します。

Duktape API で新しいスレッドと対話するには、API 呼び出し用のコンテキスト・ポインタを取得するために duk_get_context() を使用します。


### 例

```c
duk_idx_t thr_idx;
duk_context *new_ctx;

thr_idx = duk_push_thread(ctx);
new_ctx = duk_get_context(ctx, thr_idx);
```

### 参照

duk_push_thread_new_globalenv


## duk_push_thread_new_globalenv() 

1.0.0 thread stack borrowed

### プロトタイプ

```c
duk_idx_t duk_push_thread_new_globalenv(duk_context *ctx);
```

### スタック

| ... | -> | ... | thr |

### 要約

新しい Duktape スレッド (コンテキスト、コルーチン) をスタックにプッシュします。プッシュされたスレッドの負でないインデックス(スタックの底からの相対値)を返します。新しいスレッドは、引数 ctx と同じ Duktape ヒープに関連付けられますが、新しいグローバルオブジェクト環境 (ctx が使用する環境とは別) を持つことになります。

Duktape API で新しいスレッドと対話するには、duk_get_context() を使って API 呼び出しのためのコンテキスト・ポインタを取得します。


### 例

```c
duk_idx_t thr_idx;
duk_context *new_ctx;

thr_idx = duk_push_thread_new_globalenv(ctx);
new_ctx = duk_get_context(ctx, thr_idx);
```

### 参照

duk_push_thread


## duk_push_thread_stash() 

1.0.0 thread stash stack sandbox object module

### プロトタイプ

```c
void duk_push_thread_stash(duk_context *ctx, duk_context *target_ctx);
```

### スタック

| ... | -> | ... | stash |

### 要約

target_ctx に関連する stash オブジェクトをスタックにプッシュします (ctx と target_ctx 引数は同じスレッドを参照することができます)。スレッドスタッシュは内部オブジェクトで、C コードからキー/値ペアを保存するために使用され、ガベージコレクションのために到達可能ですが、ECMAScript コードからアクセスすることができないようにします。スタッシュは、マッチする target_ctx 引数を持つ C コードからしかアクセスできません。

target_ctx が NULL の場合、エラーを投げます。


### 例

```c
duk_push_thread_stash(ctx, ctx2);
```

### 参照

duk_push_heap_stash
duk_push_global_stash


## duk_push_true() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_true(duk_context *ctx);
```

### スタック

| ... | -> | ... | true |

### 要約

スタックにtrueをプッシュします。duk_push_boolean(ctx, 1)を呼ぶのと同等です。


### 例

```c
duk_push_true(ctx);
```


## duk_push_uint() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_uint(duk_context *ctx, duk_uint_t val);
```

### スタック

| ... | -> | ... | val |

### 要約

valをIEEEダブルに変換し、スタックにプッシュします。

これは duk_push_number(ctx, (duk_double_t) val) を呼び出すための省略記法です。


### 例

```c
duk_push_uint(ctx, 123);
```


## duk_push_undefined() 

1.0.0 stack

### プロトタイプ

```c
void duk_push_undefined(duk_context *ctx);
```

### スタック

| ... | -> | ... | undefined |

### 要約

未定義をスタックに積む。


### 例

```c
duk_push_undefined(ctx);
```


## duk_push_vsprintf() 

1.0.0 vararg string stack

### プロトタイプ

```c
const char *duk_push_vsprintf(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | str |

### 要約

vsprintf()のように（ただし安全に）文字列をフォーマットし、結果の値をバリュースタックにプッシュします。結果の文字列へのNULLでないポインタを返します。

fmtがNULLの場合、空文字列がスタックにプッシュされ、空文字列へのNULLでないポインタが返される (この動作は、少なくともLinuxでは、NULLフォーマット文字列に対するvsprintf()の動作に類似しています。)。返されたポインタは再参照可能であり、NULターミネータ文字が保証されます。

vsprintf()とは異なり、文字列のフォーマットは安全です。具体的には、この実装は、一時的にフォーマットされた値に対して十分な大きさのバッファが見つかるまで、一時的なバッファのサイズを大きくしてみます。

ap 引数は、複数回の呼び出しに対して安全に再利用することはできません。これは、vararg メカニズムの制限事項です。

このAPIコールは、呼び出し側のコードのva_end()マクロが(例えばエラースローにより)到達しないかもしれないので、完全に移植性があるわけではありません。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存しています; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void test_vsprintf(duk_context *ctx, ...) {
    va_list ap;

    va_start(ap, ctx);
    duk_push_vsprintf(ctx, "test: %d+%d=%d", ap);
    va_end(ap);
}

void test(duk_context *ctx) {
    test_vsprintf(ctx, 2, 3, 5);
}
```


## duk_put_function_list() 

1.0.0 property module

### プロトタイプ

```c
void duk_put_function_list(duk_context *ctx, duk_idx_t obj_idx, const duk_function_list_entry *funcs);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

複数の関数プロパティを obj_idx にあるターゲットオブジェクトに設定します。関数は、nameがNULLのトリプル（正しさのためにfunctionもNULLであることが望ましい）で終わる、トリプル（name、function、nargs）のリストとして指定されます。

これは、例えば、Duktape/C関数のセットとして実装されたモジュールやクラスを定義する際に便利です。


### 例

```c
const duk_function_list_entry my_module_funcs[] = {
    { "tweak", do_tweak, 0 /* no args */ },
    { "adjust", do_adjust, 3 /* 3 args */ },
    { "frobnicate", do_frobnicate, DUK_VARARGS /* variable args */ },
    { NULL, NULL, 0 }
};

/* Initialize an object with a set of function properties, and set it to
 * global object 'MyModule'.
 */

duk_push_global_object(ctx);
duk_push_object(ctx);  /* -> [ ... global obj ] */
duk_put_function_list(ctx, -1, my_module_funcs);
duk_put_prop_string(ctx, -2, "MyModule");  /* -> [ ... global ] */
duk_pop(ctx);
```

### 参照

duk_put_number_list


## duk_put_global_heapptr() 

2.3.0 property heap ptr borrowed

### プロトタイプ

```c
duk_bool_t duk_put_global_heapptr(duk_context *ctx, void *ptr);
```

### スタック

| ... | val | -> | ... |

### 要約

duk_put_global_string() と同様ですが、プロパティ名は、例えば duk_get_heapptr() を用いて取得した Duktape ヒープポインタとして与えられます。ptr が NULL の場合、undefined がキーとして使用されます。


### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_heapptr(ctx, my_ptr_ref);
```

### 参照

duk_put_global_string
duk_put_global_lstring
duk_put_global_literal


## duk_put_global_literal() 

2.3.0 property literal

### プロトタイプ

```c
duk_bool_t duk_put_global_literal(duk_context *ctx, const char *key_literal);
```

### スタック

| ... | val | -> | ... |

### 要約

duk_put_global_string() と同様ですが、プロパティ名は文字列リテラルとして与えられます (duk_push_literal() を参照してください)。


### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_literal(ctx, "my_app_version");
```

### 参照

duk_put_global_string
duk_put_global_lstring
duk_put_global_heapptr


## duk_put_global_lstring() 

2.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_put_global_lstring(duk_context *ctx, const char *key, duk_size_t key_len);
```

### スタック

| ... | val | -> | ... |

### 要約

duk_put_global_string() と同様ですが、キーは長さを明示した文字列として与えられます。


### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_lstring(ctx, "internal" "\x00" "nul", 12);
```

### 参照

duk_put_global_string
duk_put_global_literal
duk_put_global_heapptr


## duk_put_global_string() 

1.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_put_global_string(duk_context *ctx, const char *key);
```

### スタック

| ... | val | -> | ... |

### 要約

key という名前のプロパティをグローバルオブジェクトに配置します。戻り値は duk_put_prop() と同様に動作します。これは、それと同等のことをする便宜的な関数です。

```c
duk_bool_t ret;

duk_push_global_object(ctx);
duk_insert(ctx, -2);
ret = duk_put_prop_string(ctx, -2, key);
duk_pop(ctx);
/* 'ret' would be the return value from duk_put_global_string() */
```

### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_string(ctx, "my_app_version");
```

### 参照

duk_put_global_lstring
duk_put_global_literal
duk_put_global_heapptr


## duk_put_number_list() 

1.0.0 property module

### プロトタイプ

```c
void duk_put_number_list(duk_context *ctx, duk_idx_t obj_idx, const duk_number_list_entry *numbers);
```

### スタック

| ... | obj | ... | -> | ... | obj | ... |

### 要約

複数の数値（double）プロパティを obj_idx にあるターゲットオブジェクトに設定します。数値リストは、(name, number) のペアのリストとして与えられ、name が NULL であるペアで終了します。

これは、C言語で実装されたモジュールやクラスで数値定数を定義する場合などに有用です。


### 例

```c
const duk_number_list_entry my_module_consts[] = {
    { "FLAG_FOO", (double) (1 << 0) },
    { "FLAG_BAR", (double) (1 << 1) },
    { "FLAG_QUUX", (double) (1 << 2) },
    { "DELAY", 300.0 },
    { NULL, 0.0 }
};

duk_put_number_list(ctx, -3, my_module_consts);
```

### 参照

duk_put_function_list


## duk_put_prop() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_put_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

| ... | obj | ... | key | val | -> | ... | obj | ... |

### 要約

obj_idx にある値のプロパティキーに val を書き込む。 key と val はスタックから削除されます。リターンコードとエラースローの動作

プロパティの書き込みに成功した場合、1を返す。
書き込みに失敗した場合、エラーを投げます。（ストリクトモードセマンティクス）。また、アクセッサプロパティのセッタ関数によってもエラーがスローされる場合があります。
obj_idx の値がオブジェクト互換でない場合、エラーを投げます。。
obj_idx が無効な場合、エラーを投げます。。
プロパティの書き込みは、ECMAScript の式 obj[key] = val と等価です。プロパティ書き込みが成功または失敗するときの正確なルールは、同等の代入を行う ECMAScript コードと同じです。正確なセマンティクスは、プロパティアクセサ、PutValue (V, W)、および [[Put]] (P, V, Throw) を参照してください。ターゲット値とキーは共に強制されます。

ターゲット値は自動的にオブジェクトに強制されます。しかし、オブジェクトは一過性のものなので、そのプロパティを書いてもあまり意味がない。さらに、ECMAScript のセマンティクスはそのような一時的なオブジェクトのために新しいプロパティを作成することを防ぎます（特別な [[Put]] バリアントのステップ 7、PutValue (V, W) を参照）。
キー引数は内部的に文字列または Symbol になる ToPropertyKey() 強制適用を使用して強制適用されます。配列と数値インデックスには、明示的な文字列強制を避ける内部高速パスがあるので、該当する場合は数値キーを使用してください。
ターゲットがセットトラップを実装するProxyオブジェクトである場合、トラップが呼び出され、APIコールの戻り値はトラップの戻り値に一致します。

ECMAScript では、代入式は、代入の成功の有無にかかわらず、右辺の式の値を持つ。この API コールの戻り値は、ECMAScript によって指定されておらず、ECMAScript コードで利用可能でもありません。API コールは、割り当てが成功したかどうかに応じて 0 または 1 を返します (厳格なコードでは 0 の戻り値がエラーに昇格します)。
キーが固定文字列である場合、1 つの API 呼び出しを回避して、 duk_put_prop_string() 変数を使用することができます。同様に、キーが配列のインデックスである場合、 duk_put_prop_index() 変数を使用することができます。

プロパティアクセスの基本値は通常オブジェクトですが、技術的には任意の値にすることができます。普通の文字列やバッファの値には仮想的なインデックスプロパティがあり、例えば "foo"[2] にアクセスすることができます。また、ほとんどのプリミティブな値は何らかのプロトタイプオブジェクトを継承しているので、例えば (12345).toString(16) のようにメソッドを呼び出すことができます。

### 例

```c
duk_bool_t rc;

duk_push_string(ctx, "key");
duk_push_string(ctx, "value");
rc = duk_put_prop(ctx, -3);
printf("rc=%d\n", (int) rc);
```

### 参照

duk_put_prop_index
duk_put_prop_string
duk_put_prop_lstring
duk_put_prop_literal
duk_put_prop_heapptr


## duk_put_prop_heapptr() 

2.2.0 property heap ptr borrowed

### プロトタイプ

```c
duk_bool_t duk_put_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

| ... | obj | ... | val | -> | ... | obj | ... |

### 要約

duk_put_prop() と同様ですが、プロパティ名は、例えば duk_get_heapptr() を使って取得した Duktape ヒープポインタとして与えられます。ptr が NULL の場合、undefined がキーとして使用されます。


### 例

```c
duk_bool_t rc;
void *ptr;

duk_push_string(ctx, "propertyName");
ptr = duk_get_heapptr(ctx, -1);
/* String behind 'ptr' must remain reachable! */

duk_push_string(ctx, "value");
rc = duk_put_prop_heapptr(ctx, -3, ptr);
printf("rc=%d\n", (int) rc);
```

### 参照

duk_put_prop
duk_put_prop_index
duk_put_prop_string
duk_put_prop_lstring
duk_put_prop_literal


## duk_put_prop_index() 

1.0.0 property

### プロトタイプ

```c
duk_bool_t duk_put_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

| ... | obj | ... | val | -> | ... | obj | ... |

### 要約

duk_put_prop() と同様ですが、プロパティ名は符号なし整数の arr_idx として与えられます。これは特に配列要素への書き込みに便利です（ただし、これに限定されるものではありません）。

概念的には、プロパティ書き込みのために数値は文字列に強制されます。例えば、123 はプロパティ名 "123" と同等です。Duktapeは可能な限り、明示的な強制を避けています。


### 例

```c
duk_push_string(ctx, "value");
rc = duk_put_prop_index(ctx, -3, 123);  /* write to obj[123] */
printf("rc=%d\n", (int) rc);
```

### 参照

duk_put_prop
duk_put_prop_string
duk_put_prop_lstring
duk_put_prop_literal
duk_put_prop_heapptr


## duk_put_prop_literal() 

2.3.0 property literal

### プロトタイプ

```c
duk_bool_t duk_put_prop_literal(duk_context *ctx, duk_idx_t obj_idx, const char *key_literal);
```

### スタック

| ... | obj | ... | val | -> | ... | obj | ... |

### 要約

duk_put_prop() と同様ですが、プロパティ名は文字列リテラルとして与えられます (duk_push_literal() を参照してください)。


### 例

```c
duk_bool_t rc;

duk_push_string(ctx, "value");
rc = duk_put_prop_literal(ctx, -3, "myPropertyName");
printf("rc=%d\n", (int) rc);
```

### 参照

duk_put_prop
duk_put_prop_index
duk_put_prop_string
duk_put_prop_lstring
duk_put_prop_heapptr


## duk_put_prop_lstring() 

2.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_put_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

| ... | obj | ... | val | -> | ... | obj | ... |

### 要約

duk_put_prop() と同様ですが、プロパティ名は、長さを明示した文字列として与えられます。


### 例

```c
duk_bool_t rc;

duk_push_string(ctx, "value");
rc = duk_put_prop_lstring(ctx, -3, "internal" "\x00" "nul", 12);
printf("rc=%d\n", (int) rc);
```

### 参照

duk_put_prop
duk_put_prop_index
duk_put_prop_string
duk_put_prop_literal
duk_put_prop_heapptr


## duk_put_prop_string() 

1.0.0 string property

### プロトタイプ

```c
duk_bool_t duk_put_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

| ... | obj | ... | val | -> | ... | obj | ... |

### 要約

duk_put_prop() のように、プロパティ名は、NUL で終端する C 文字列キーとして与えられます。


### 例

```c
duk_bool_t rc;

duk_push_string(ctx, "value");
rc = duk_put_prop_string(ctx, -3, "key");
printf("rc=%d\n", (int) rc);
```

### 参照

duk_put_prop
duk_put_prop_index
duk_put_prop_lstring
duk_put_prop_literal
duk_put_prop_heapptr


## duk_random() 

2.3.0 random

### プロトタイプ

```c
duk_double_t duk_random(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし。)


### 要約

Math.random()に対応するC APIで、範囲 [0,1] のランダムなIEEEダブル値を返します。


### 例

```c
if (duk_random(ctx) <= 0.01) {
    printf("surprise!\n");
}
```


## duk_range_error() 

2.0.0 error

### プロトタイプ

```c
duk_ret_t duk_range_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_RANGE_ERROR を持つ duk_error() に相当する便利な API コールです。


### 例

```c
return duk_range_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_range_error_va() 

2.0.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_range_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_RANGE_ERROR を持つ duk_error_va() と同等の便宜的な API 呼び出し。

このAPIコールは完全には移植性がない。なぜなら、呼び出し側のコードの va_end() マクロは(例えばエラースローにより)決して到達できないかもしれないからです。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存します; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_range_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_range_error_va(ctx, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error_va


## duk_realloc() 

1.0.0 memory

### プロトタイプ

```c
void *duk_realloc(duk_context *ctx, void *ptr, duk_size_t size);
```

### スタック

(バリュースタックに影響なし)


### 要約

duk_realloc_raw() と同様ですが、要求を満たすためにガベージコレクションを起動させることがあります。しかし、割り当てられたメモリ自体は、自動的にガベージコレクションされません。割り当てられたサイズが前回の割り当てから拡張された場合、新たに割り当てられたバイトは自動的にゼロにされず、任意のゴミを含む可能性があります。

duk_realloc() で再割り当てされたメモリは、duk_free() または duk_free_raw() で解放することができます。


### 例

```c
void *buf = duk_alloc(ctx, 1024);
if (buf) {
    void *buf2 = duk_realloc(ctx, 2048);
    if (!buf2) {
        printf("failed to reallocate, 'buf' still valid\n");
    } else {
        printf("reallocate successful, 'buf2' now valid\n");
    }
}
```

### 参照

duk_realloc_raw


## duk_realloc_raw() 

1.0.0 memory

### プロトタイプ

```c
void *duk_realloc_raw(duk_context *ctx, void *ptr, duk_size_t size);
```

### スタック

(バリュースタックに影響なし。)


### 要約

コンテキストに登録されたアロケーション関数で作成された以前のアロケーションのサイズを変更します。ptr 引数は以前の割り当てを指し、size は新しい割り当てのサイズです。この呼び出しは、以前のものと異なるポインタを持つ可能性のある、新しい割り当てへのポインタを返します。再割り当てに失敗した場合、NULL が返され、以前の割り当てはまだ有効です。再配置の失敗は、新しいサイズが以前のサイズよりも大きい場合にのみ起こり得ます (つまり、呼び出し側が割り当てを大きくしようとした場合)。再割り当ての試みはガベージコレクションを引き起こすことはできず、割り当てられたメモリは自動的にガベージコレクションされることはありません。割り当てられたサイズが以前の割り当てから拡張された場合、新しく割り当てられたバイトは自動的にゼロにされず、任意のゴミを含む可能性があります。

正確な動作は、以下のように ptr と size の引数に依存します。

ptr が NULL でなく、size が 0 より大きい場合、以前の割り当てはサイズ変更されます。
ptr が非 NULL で size が 0 の場合、この呼び出しは duk_free_raw() と等価です。
ptr が NULL で size が 0 より大きい場合、この呼び出しは duk_alloc_raw() と等価です。
ptr が NULL で size が 0 の場合、この呼び出しは size 引数が 0 の duk_alloc_raw() と等価です。
duk_realloc_raw() で再割り当てされたメモリは、 duk_free() または duk_free_raw() で解放することができます。


### 例

```c
void *buf = duk_alloc_raw(ctx, 1024);
if (buf) {
    void *buf2 = duk_realloc_raw(ctx, 2048);
    if (!buf2) {
        printf("failed to reallocate, 'buf' still valid\n");
    } else {
        printf("reallocate successful, 'buf2' now valid\n");
    }
}
```

### 参照

duk_realloc


## duk_reference_error() 

2.0.0 error

### プロトタイプ

```c
duk_ret_t duk_reference_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_REFERENCE_ERROR を持つ duk_error() に相当する便利な API コールです。


### 例

```c
return duk_reference_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_reference_error_va() 

2.0.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_reference_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_REFERENCE_ERROR を持つ duk_error_va() と同等の便宜的な API 呼び出し。

このAPIコールは完全には移植性がない。なぜなら、呼び出し側のコードの va_end() マクロは(例えばエラースローにより)決して到達できないかもしれないからです。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存します; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_reference_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_reference_error_va(ctx, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error_va


## duk_remove() 

1.0.0 stack

### プロトタイプ

```c
void duk_remove(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val(idx) | ... | -> | ... | ... |

### 要約

idxの値を削除します。idxより上の要素は、スタックの下に一段階シフトされます。idxが無効なインデックスの場合、エラーを投げます。。


### 例

```c
duk_push_int(ctx, 123);
duk_push_int(ctx, 234);
duk_push_int(ctx, 345);       /* -> [ 123 234 345 ] */
duk_remove(ctx, -2);          /* -> [ 123 345 ] */
```


## duk_replace() 

1.0.0 stack

### プロトタイプ

```c
void duk_replace(duk_context *ctx, duk_idx_t to_idx);
```

### スタック

| ... | old(to_idx) | ... | val | -> | ... | val(to_idx) | ... |

### 要約

to_idxにある値をスタックトップからポップした値で置き換えます。to_idxが無効なインデックスの場合、エラーを投げます。。

負のインデックスは、スタックトップにある値をポップする前に評価されます。これは、例でも説明されています。

### 例

```c
duk_push_int(ctx, 123);
duk_push_int(ctx, 234);
duk_push_int(ctx, 345);       /* -> [ 123 234 345 ] */
duk_push_string(ctx, "foo");  /* -> [ 123 234 345 "foo" ] */
duk_replace(ctx, -3);         /* -> [ 123 "foo" 345 ] */
```

### 参照

duk_insert


## duk_require_boolean() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_require_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_boolean() と同様ですが、idx の値が boolean でない場合、あるいはインデックスが無効な場合にエラーを発生します。


### 例

```c
if (duk_require_boolean(ctx, -3)) {
    printf("value is true\n");
}
```


## duk_require_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_require_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... |

### 要約

duk_get_buffer() と同様ですが、idx の値が (plain) buffer でない場合、またはインデックスが無効な場合にエラーを発生させます。


### 例

```c
void *ptr;
duk_size_t sz;

ptr = duk_require_buffer(ctx, -3, &sz);
printf("buf=%p, size=%lu\n", ptr, (unsigned long) sz);
```

### 参照

duk_get_buffer
duk_require_buffer_data
duk_get_buffer_data


## duk_require_buffer_data() 

1.3.0 stack buffer object buffer

### プロトタイプ

```c
void *duk_require_buffer_data(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... |

### 要約

duk_get_buffer_data() と同様ですが、idx の値がプレーンバッファまたはバッファオブジェクトでない場合、値がバッファオブジェクトでその「バッキングバッファ」がバッファオブジェクトの見た目のサイズを完全にカバーしていない場合、またはインデックスが無効な場合にエラーを送出します。


### 例

```c
void *ptr;
duk_size_t sz;

ptr = duk_require_buffer_data(ctx, -3, &sz);
printf("buf=%p, size=%lu\n", ptr, (unsigned long) sz);
```

### 参照

duk_get_buffer_data
duk_require_buffer
duk_get_buffer


## duk_require_c_function() 

1.0.0 stack function

### プロトタイプ

```c
duk_c_function duk_require_c_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_c_function() と同様ですが、idx の値が Duktape/C 関数に関連付けられた ECMAScript 関数でない場合、またはインデックスが無効な場合にエラーをスローします。


### 例

```c
duk_c_function funcptr;

funcptr = duk_require_c_function(ctx, -3);
```


## duk_require_callable() 

1.4.0 stack

### プロトタイプ

```c
void duk_require_callable(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値が呼び出し可能でない場合、またはインデックスが無効な場合にエラーを投げます。。

任意の callable 値に対する有用な C の戻り値がないので、"get" プリミティブ (duk_get_callable()) は存在しません。現時点では、この呼び出しはduk_require_function()の呼び出しと等価です。

### 例

```c
duk_require_callable(ctx, -3);
```


## duk_require_constructable() 

2.4.0 stack

### プロトタイプ

```c
void duk_require_constructable(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が構成可能な関数でない場合、またはidxが無効な場合、エラーを投げます。。それ以外の場合は返す。


### 例

```c
duk_require_constructable(ctx, 3);
```


## duk_require_constructor_call() 

2.4.0 stack

### プロトタイプ

```c
void duk_require_constructor_call(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

現在の関数がコンストラクタ呼び出し (new Foo()) ではなく、通常の関数呼び出し (Foo()) として呼び出された場合にエラーをスローします。また、現在の呼び出しが進行中でない場合にもスローします。それ以外の場合は、次のようになります。


### 例

```c
duk_require_constructor_call(ctx);
```


## duk_require_context() 

1.0.0 stack borrowed

### プロトタイプ

```c
duk_context *duk_require_context(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_context() と同様であるが、idx の値が Duktape スレッドでない場合、またはインデックスが無効な場合にエラーを投げます。。


### 例

```c
duk_context *new_ctx;

(void) duk_push_thread(ctx);
new_ctx = duk_require_context(ctx, -1);
```


## duk_require_function() 

1.4.0 stack

### プロトタイプ

```c
void duk_require_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値が関数でない場合、またはインデックスが無効な場合にエラーを投げます。。

任意の関数に対する有用なCの戻り値がないため、getプリミティブ(duk_get_function())は存在しない。

### 例

```c
duk_require_function(ctx, -3);
```


## duk_require_heapptr() 

1.1.0 stack heap ptr borrowed

### プロトタイプ

```c
void *duk_require_heapptr(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_heapptr() と同様ですが、idx の値が Duktape ヒープに割り当てられた値 （オブジェクト、バッファ、文字列）でない場合、またはインデックスが無効な場合にエラーを 投げます。


### 例

```c
void *ptr;

ptr = duk_require_heapptr(ctx, -3);
```

### 参照

duk_get_heapptr
duk_push_heapptr


## duk_require_int() 

1.0.0 stack

### プロトタイプ

```c
duk_int_t duk_require_int(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_int() と同様であるが、idx の値が数値でない場合、またはインデックスが無効な場合にエラーを投げます。。


### 例

```c
printf("int value: %ld\n", (long) duk_require_int(ctx, -3));
```


## duk_require_lstring() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_require_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

| ... | val | ... |

### 要約

duk_get_lstring() と同様であるが、idx の値が文字列でない場合、またはインデックスが無効な場合にエラーを投げます。。

これは、バッファデータポインタの扱い方とは (技術的な理由により) 異なります。

### 例

```c
const char *buf;
duk_size_t len;

buf = duk_require_lstring(ctx, -3, &len);
printf("value is a string, %lu bytes\n", (unsigned long) len);
```

### 参照

duk_require_string


## duk_require_normalize_index() 

1.0.0 stack

### プロトタイプ

```c
duk_idx_t duk_require_normalize_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

現在のフレームの底を基準として、引数のインデックスを正規化します。結果として得られるインデックスは0以上になり、後のスタックの変更に依存しなくなります。入力インデックスが無効な場合は、エラーを投げます。。


### 例

```c
duk_idx_t idx = duk_require_normalize_index(-3);
```

### 参照

duk_normalize_index


## duk_require_null() 

1.0.0 stack

### プロトタイプ

```c
void duk_require_null(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値が NULL でない場合、またはインデックスが無効な場合にエラーを投げます。。

get" プリミティブ (duk_get_null()) は存在しない。なぜなら、そのような関数は無意味だからです。

### 例

```c
duk_require_null(ctx, -3);
```


## duk_require_number() 

1.0.0 stack

### プロトタイプ

```c
duk_double_t duk_require_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_number() と同様であるが、idx の値が数値でない場合、またはインデックスが無効な場合にエラーを投げます。。


### 例

```c
printf("value: %lf\n", (double) duk_require_number(ctx, -3));
```


## duk_require_object() 

2.2.0 stack

### プロトタイプ

```c
void duk_require_object(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idx の値がオブジェクトでない場合、またはインデックスが無効な場合、エラーを投げます。。

get" プリミティブ (duk_get_object()) は存在しません。なぜなら、そのような関数は無用だからです。

### 例

```c
duk_require_object(ctx, -3);
```


## duk_require_object_coercible() 

1.0.0 stack object

### プロトタイプ

```c
void duk_require_object_coercible(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_is_object_coercible() と同様ですが、val がオブジェクト保 持可能でない場合、TypeError を投げます。


### 例

```c
duk_require_object_coercible(ctx, -3);
```


## duk_require_pointer() 

1.0.0 stack

### プロトタイプ

```c
void *duk_require_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_pointer() と同様であるが、idx の値がポインタでない場合、またはインデックスが無効な場合にエラーを投げます。。


### 例

```c
void *ptr;

ptr = duk_require_pointer(ctx, -3);
printf("my pointer is: %p\n", ptr);
```


## duk_require_stack() 

1.0.0 stack

### プロトタイプ

```c
void duk_require_stack(duk_context *ctx, duk_idx_t extra);
```

### スタック

(バリュースタックに影響なし。)


### 要約

duk_check_stack() と同様ですが、バリュースタックを再割り当てする必要があり、再割り当てに失敗した場合にエラーがスローされます。

一般的なルールとして、呼び出し側はより多くのスタック空間を確保するためにこの関数を使用する必要があります。バリュースタックを拡張できない場合、エラーを投げてアンワインドする以外、有用な回復策はほとんどない。


### 例

```c
duk_idx_t nargs;

nargs = duk_get_top(ctx);  /* number or arguments */

/* reserve space for one temporary for each input argument */
duk_require_stack(ctx, nargs);
```

### 参照

duk_check_stack


## duk_require_stack_top() 

1.0.0 stack

### プロトタイプ

```c
void duk_require_stack_top(duk_context *ctx, duk_idx_t top);
```

### スタック

(バリュースタックに影響なし。)


### 要約

duk_check_stack_top() と同様ですが、バリュースタックを再割り当てする必要があり、再割り当てに失敗した場合にエラーがスローされます。

一般的なルールとして、呼び出し側はより多くのスタック空間を確保するためにこの関数を使用すべきです。バリュースタックを拡張できない場合、エラーを投げて巻き戻す以外に有用な回復策はほとんどない。


### 例

```c
duk_require_stack_top(ctx, 100);
printf("value stack guaranteed up to index 99\n");
```

### 参照

duk_check_stack_top


## duk_require_string() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_require_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_string() と同様であるが、idx の値が文字列でない場合、またはインデックスが無効な場合にエラーを投げます。。

これは、バッファデータポインタの扱い方とは (技術的な理由により) 異なります。
シンボル値は C API では文字列として見えるので、 duk_is_symbol() と duk_is_string() の両方が真となります。この動作は、Duktape 1.xの内部文字列と同様です。duk_is_symbol() を使えば、シンボルを普通の文字列と区別することができます。内部表現については、symbols.rstを参照してください。

### 例

```c
const char *buf;

buf = duk_require_string(ctx, -3);
printf("value is a string: %s\n", buf);
```

### 参照

duk_require_lstring


## duk_require_top_index() 

1.0.0 stack

### プロトタイプ

```c
duk_idx_t duk_require_top_index(duk_context *ctx);
```

### スタック

(バリュースタックに影響なし)


### 要約

スタック上の最上位の値の絶対インデックス(>= 0)を取得します。スタックが空の場合、エラーを投げます。。


### 例

```c
duk_idx_t idx_top;

/* throws error if stack is empty */
idx_top = duk_require_top_index(ctx);
printf("index of top element: %ld\n", (long) idx_top);
```

### 参照

duk_get_top_index


## duk_require_type_mask() 

1.0.0 stack

### プロトタイプ

```c
void duk_require_type_mask(duk_context *ctx, duk_idx_t idx, duk_uint_t mask);
```

### スタック

| ... | val | ... |

### 要約

duk_check_type_mask() と同様ですが、val の型がどのマスクビットにも一致しない場合、 TypeError を投げます。


### 例

```c
duk_require_type_mask(ctx, -3, DUK_TYPE_MASK_STRING |
                               DUK_TYPE_MASK_NUMBER);
```


## duk_require_uint() 

1.0.0 stack

### プロトタイプ

```c
duk_uint_t duk_require_uint(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

duk_get_uint() と同様であるが、idx の値が数値でない場合、あるいはインデックスが無効な場合にエラーを発生させます。


### 例

```c
printf("unsigned int value: %lu\n", (unsigned long) duk_require_uint(ctx, -3));
```


## duk_require_undefined() 

1.0.0 stack

### プロトタイプ

```c
void duk_require_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

idxの値が未定義でない場合、またはインデックスが無効な場合にエラーを投げます。。

get" プリミティブ (duk_get_undefined()) は存在しません。

### 例

```c
duk_require_undefined(ctx, -3);
```


## duk_require_valid_index() 

1.0.0 stack

### プロトタイプ

```c
void duk_require_valid_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... |

### 要約

引数indexの妥当性を確認します。indexが有効であればエラーをスローし、そうでなければ（戻り値なしで）リターンします。


### 例

```c
duk_require_valid_index(ctx, -3);
printf("index -3 is valid\n");
```

### 参照

duk_is_valid_index


## duk_resize_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_resize_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t new_size);
```

### スタック

| ... | val | ... |

### 要約

idx のダイナミックバッファを new_size バイトにリサイズします。new_size が現在のサイズより大きい場合、新しく割り当てられたバイト（古いサイズより大きい）は自動的にゼロにされます。新しいバッファのデータ領域へのポインタを返す。new_size が 0 の場合、NULL または NULL 以外の値を返す。サイズ変更に失敗した場合、idxの値がダイナミックバッファでない場合、 idxが無効な場合、エラーを投げます。。


### 例

```c
void *ptr;

ptr = duk_resize_buffer(ctx, -3, 4096);
```


## duk_resume() 

1.6.0 thread

### プロトタイプ

```c
void duk_resume(duk_context *ctx, const duk_thread_state *state);
```

### スタック

| ... | state(N) | -> | ... | (number of popped stack entries may vary)

### 要約

duk_suspend() で一時停止した Duktape の実行を再開します。state 引数は NULL であってはならない。Value スタックと state は、 duk_suspend() によって残された状態でなければならない； そうでない場合、メモリ不安全な動作が起こります。


### 例

```c
/* See example for duk_suspend(). */
```

### 参照

duk_suspend


## duk_safe_call() 

1.0.0 protected call

### プロトタイプ

```c
duk_int_t duk_safe_call(duk_context *ctx, duk_safe_call_function func, void *udata, duk_idx_t nargs, duk_idx_t nrets);
```

### スタック

| ... | arg1 | ... | argN | -> | ... | ret1 | ... | retN |

### 要約

現在のバリュースタックフレーム内で保護された純粋な C 関数呼び出しを実行します (呼び出しはコールスタック上で見えません)。現在のバリュースタックフレーム内の nargs 最上位値は呼び出し引数として識別され、nrets 戻り値は呼び出しが戻った後に提供されます。呼び出し元のコードは、例えば duk_check_stack() を使用して、バリュースタックが nrets のために予約されていること、および予約エラーを処理することを確認する必要があります。

udata (userdata) ポインタは、そのまま func に渡され、バリュースタックを使用せずに、1つまたは複数の C 値をターゲット関数に簡単に渡すことができます。複数の値を渡すには、スタックに確保されたC構造体に値を詰め、その構造体へのポインタをuserdataとして渡します。userdata引数はDuktapeでは解釈されません。もしそれが必要でなければ、単にNULLを渡して、セーフコールターゲットのudata引数を無視してください。

戻り値は

DUK_EXEC_SUCCESS (0) です。call が成功した場合、nargs 引数は nrets の戻り値に置き換えられます。(この戻り値コード定数はゼロであることが保証されているので、「ゼロかゼロ以外か」のチェックで成功を確認することができる)
DUK_EXEC_ERROR: 呼び出しに失敗、nargs 引数は nrets 値に置き換えられ、そのうちの最初の値はエラー値で、残りは未定義です。(例外的なケースとして、例えばバリュースタック上の引数が少なすぎる場合、呼び出しは投げます。かもしれない)。
ほとんどのDuktape APIコールとは異なり、このコールは成功時にゼロを返します。これにより、複数のエラーコードを後で定義することができます。
このコールは現在のバリュースタック・フレーム上で動作するため、スタックの動作とリターン・コードの処理は、他のコール・タイプと少し異なっています。

スタックtopのnargs要素は、funcへの引数として識別され、戻りスタックのための「ベース・インデックス」を確立するように。

```c
(duk_get_top() - nargs)
```

func が戻るとき、その戻り値で、スタックの一番上にプッシュした戻り値の数を示す；複数またはゼロの戻り値も可能です。スタックは、呼び出し前に設定された「基本インデックス」から始まるちょうどnrets個の値が存在するように操作されます。

func はバリュースタックへのフルアクセスを持っているので、意図した引数の下のスタックを変更し、スタックから「ベースインデックス」の下の要素をポップすることさえできることに注意してください。このような要素は、リターン時にスタックが常に一貫した状態になるように、リターン前に未定義の値でリストアされます。

エラーが発生した場合、スタックは「ベースインデックス」にnretsの値を持つことになります。nrets が 0 の場合、スタック上にエラーは存在しない (戻り値のスタックトップは "base index" に等しい) ので、 nrets を 0 としてこの関数を呼び出すと、起こりうるエラーの原因がわからなくなり、あまり役に立たないかもしれない。

nargs = 3, nrets = 2, func returns 4 でのバリュースタックの動作例。パイプ文字は論理的なバリュースタックの境界を示す。

```sh
      .--- frame bottom
      |
      |     .--- "base index"
      v     v
[ ... | ... | a b c ]            stack before calling 'func'

[ ... | ... | a b | x y z w ]    stack after calling 'func', which has
                                 popped one argument and written four
                                 return values (and returned 4)

[ ... | ... | x y ]              stack after duk_safe_call() returns,
                                 2 (= nrets) first 'func' return values
                                 are left at "base index"
```

func は呼び出し元のスタックフレームを使用するので、呼び出し元のコンテキストがわからない限り、 'func' 内でのボトムベースの参照は危険であることに注意してください。
userdata引数はDuktape 2.xで追加されました。

### 例

```c
typedef struct {
  int floor;
} my_safe_args;

duk_ret_t my_safe_func(duk_context *ctx, void *udata) {
    my_safe_args *args = (my_safe_args *) udata;
    double a, b, c, t;

    a = duk_get_number(ctx, -3);
    b = duk_get_number(ctx, -2);
    c = duk_get_number(ctx, -1);  /* ignored on purpose */
    t = a + b;
    if (args->floor) {
        t = floor(t);
    }
    duk_push_number(ctx, t);

    /* Indicates that there is only one return value.  Because the caller
     * requested two (nrets == 2), Duktape will automatically add an
     * additional "undefined" result value.
     */
    return 1;
}

my_safe_args args;  /* C struct whose pointer is passed as userdata */
duk_int_t rc;

args.floor = 1;
duk_push_int(ctx, 10);
duk_push_int(ctx, 11);
duk_push_int(ctx, 12);
rc = duk_safe_call(ctx, my_func, (void *) &args, 3 /*nargs*/, 2 /*nrets*/);
if (rc == DUK_EXEC_SUCCESS) {
    printf("1st return value: %s\n", duk_to_string(ctx, -2));  /* 21 */
    printf("2nd return value: %s\n", duk_to_string(ctx, -1));  /* undefined */
} else {
    printf("error value: %s\n", duk_to_string(ctx, -2));
}
duk_pop_2(ctx);
```


## duk_safe_to_lstring() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_safe_to_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

| ... | val | ... | -> | ... | ToString(val) | ... |

### 要約

duk_to_lstring() と同様であるが、最初の文字列強制に失敗した場合、エラー値を文字列に強制します。それも失敗した場合、固定されたエラー文字列が返されます。

呼び出し側は、この関数を使って安全に値を文字列に強制することができ、 Cコードでは、printf()で安全に戻り値をプリントアウトするのに便利です。捕捉されないエラーは、メモリ不足とその他の内部エラーだけです。


### 例

```c
const char *ptr;
duk_size_t sz;

/* Coercion causes error. */
duk_eval_string(ctx, "({ toString: function () { throw new Error('toString error'); } })");
ptr = duk_safe_to_lstring(ctx, -1, &sz);
printf("coerced string: %s (length %lu)\n", ptr, (unsigned long) sz);  /* -> "Error: toString error" */
duk_pop(ctx);

/* Coercion causes error, and the error itself cannot be string coerced. */
duk_eval_string(ctx, "({ toString: function () { var e = new Error('cannot string coerce me');"
                     "                           e.toString = function () { throw new Error('coercion error'); };"
                     "                           throw e; } })");
ptr = duk_safe_to_lstring(ctx, -1, &sz);
printf("coerced string: %s (length %lu)\n", ptr, (unsigned long) sz);  /* -> "Error" */
duk_pop(ctx);
```

### 参照

duk_to_lstring


## duk_safe_to_stacktrace() 

2.4.0 string stack protected

### プロトタイプ

```c
const char *duk_safe_to_stacktrace(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | val.stack | ... |

### 要約

duk_to_stacktrace() と同様であるが、もし coercion が失敗した場合、 duk_to_stacktrace() が coercion エラーに適用されます。これも失敗した場合、代わりにあらかじめ確保された固定エラー文字列 "Error" が使用されます (この文字列はあらかじめ確保されているので、メモリ不足で失敗することはありません)。

ECMAScript に相当します。

```javascript
function duk_safe_to_stacktrace(val) {
    try {
        return duk_to_stacktrace(val);
    } catch (e) {
        try {
            return duk_to_stacktrace(e);
        } catch (e2) {
            return 'Error';
        }
    }
}
```

### 例

```c
if (duk_peval_string(ctx, "1 + 2 +") != 0) {  /* => SyntaxError */
    printf("failed: %s\n", duk_safe_to_stacktrace(ctx, -1));
} else {
    printf("success\n");
}
duk_pop(ctx);
```

### 参照

duk_safe_to_string
duk_to_stacktrace


## duk_safe_to_string() 

1.0.0 string stack protected

### プロトタイプ

```c
const char *duk_safe_to_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToString(val) | ... |

### 要約

duk_to_string() と同様であるが、最初の文字列強制に失敗した場合、エラー値を文字列に強制します。これも失敗した場合、あらかじめ確保された固定エラー文字列 "Error" が代わりに使用される (文字列はあらかじめ確保されているので、メモリ不足で失敗することはない)。

呼び出し側は、この関数を使って安全に値を文字列に変換することができます。これは、Cコードでprintf()を使って安全に戻り値をプリントアウトするのに便利な機能です。捕捉されないエラーは、メモリ不足とその他の内部エラーのみです。

ECMAScriptに相当します。

```javascript
function duk_safe_to_string(val) {
    function tostr(x) {
        try {
            return String(x);
        } catch (e) {
            return e;
        }
    }

    // Initial coercion attempt.
    var t = tostr(val);
    if (typeof t === 'string') {
        // ToString() coercion succeeded with a string result, or
        // coercion failed with a plain string error value; return as is.
        return t;
    }

    // Result was an Error or a non-string, try to coerce again.
    t = tostr(t);
    if (typeof t === 'string') {
        return t;
    }

    // Still an Error or a non-string, return fixed value.
    return 'Error';
}
```

文字列の強制は、エラースローから安全ですが、現在の実装では副作用があるかもしれません。特に、文字列の強制は無限ループに入り、決して戻ってこないかもしれない。

### 例

```c
/* Coercion causes error. */
duk_eval_string(ctx, "({ toString: function () { throw new Error('toString error'); } })");
printf("coerced string: %s\n", duk_safe_to_string(ctx, -1));  /* -> "Error: toString error" */
duk_pop(ctx);

/* Coercion causes error, and the error itself cannot be string coerced. */
duk_eval_string(ctx, "({ toString: function () { var e = new Error('cannot string coerce me');"
                     "                           e.toString = function () { throw new Error('coercion error'); };"
                     "                           throw e; } })");
printf("coerced string: %s\n", duk_safe_to_string(ctx, -1));  /* -> "Error" */
duk_pop(ctx);
```

### 参照

duk_to_string


## duk_samevalue() 

2.0.0 compare

### プロトタイプ

```c
duk_bool_t duk_samevalue(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

| ... | val1 | ... | val2 | ... |

### 要約

idx1 と idx2 の値が等しいかどうかを比較します。ECMAScript の SameValue アルゴリズム (ES2015 では Object.is()) のセマンティクスを使用して値が等しいと見なされる場合は 1 を返し、そうでない場合は 0 を返します。また、どちらかのインデックスが無効な場合は 0 を返す。


### 例

```c
if (duk_samevalue(ctx, -3, -7)) {
    printf("values at indices -3 and -7 are SameValue() equal\n");
}
```


## duk_seal() 

2.2.0 property object

### プロトタイプ

```c
void duk_seal(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

(バリュースタックには影響しません。)


### 要約

obj_idx にあるオブジェクトの Object.seal() に相当します。seal されると、オブジェクトは自動的にコンパクト化されます。封印された場合、オブジェクトは自動的に圧縮されます。インデックスが無効な場合、エラーがスローされます。


### 例

```c
duk_seal(ctx, -3);
```


## duk_set_finalizer() 

1.0.0 object finalizer

### プロトタイプ

```c
void duk_set_finalizer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | finalizer | -> | ... | val | ... |

### 要約

idx の値のファイナライザをスタックトップにある値に設定します。対象がオブジェクトでない場合は、エラーを投げます。。ファイナライザの値は任意であり、関数以外の値は、ファイナライザが設定されていないものとして扱われます。オブジェクトからファイナライザを削除するには、undefinedを設定します。

Proxyオブジェクトのファイナライザは現在未対応です。オブジェクトが拡張不可能な場合、ファイナライザを設定することはできませんので、オブジェクトを封印/凍結する前にファイナライザを設定してください。

### 例

```c
duk_ret_t my_finalizer(duk_context *ctx) {
    /* Object being finalized is at stack index 0. */
    printf("object being finalized\n");
    return 0;
}

/* Create an object whose finalizer is my_finalizer(). */
duk_push_object(ctx);
duk_push_c_function(ctx, my_finalizer, 1 /*nargs*/);
duk_set_finalizer(ctx, -2);
```

### 参照

duk_get_finalizer


## duk_set_global_object() 

1.0.0 thread stack sandbox

### プロトタイプ

```c
void duk_set_global_object(duk_context *ctx);
```

### スタック

| ... | new_global | -> | ... |

### 要約

現在のコンテキストのグローバルオブジェクトを、バリュースタックの一番上にあるオブジェクトで置き換えます。値がオブジェクトでない場合は、エラーがスローされます。

この操作は、他のコンテキストのグローバルオブジェクトには影響しないことに注意してください。たとえ、この時点まで同じグローバル環境を共有していたとしてもです。他のコンテキストに変更を継承するには、duk_push_thread() を呼び出す前に、まずグローバルオブジェクトを置き換えます。

変更後の詳細な動作については、テストケース test-set-global-object.c を参照してください。


### 例

```c
/* Build a global object with a subset of bindings. */
duk_eval_string(ctx,
    "({\n"
    "    print: this.print,\n"
    "    JSON: this.JSON\n"
    "})\n");

/* Replace global object. */
duk_set_global_object(ctx);
```


## duk_set_length() 

2.0.0 stack

### プロトタイプ

```c
void duk_set_length(duk_context *ctx, duk_idx_t idx, duk_size_t len);
```

### スタック

| ... | val | ... |

### 要約

idxの値に対して "length "を設定します。ECMAScriptのobj.length = len;と等価です。


### 例

```c
/* Set array length to zero, deleting elements as a side effect. */
duk_set_length(ctx, -3, 0);
```

### 参照

duk_get_length


## duk_set_magic() 

1.0.0 magic function

### プロトタイプ

```c
void duk_set_magic(duk_context *ctx, duk_idx_t idx, duk_int_t magic);
```

### スタック

| ... | val | ... |

### 要約

idxにあるDuktape/C関数に関連する16ビット符号付き「マジック」値を設定します。もしその値がDuktape/C関数でない場合は、エラーが投げられます。


### 例

```c
duk_set_magic(ctx, -3, 0x1234);
```

### 参照

duk_get_current_magic
duk_get_magic


## duk_set_prototype() 

1.0.0 prototype object

### プロトタイプ

```c
void duk_set_prototype(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | proto | -> | ... | val | ... |

### 要約

idx の値の内部プロトタイプをスタックの先頭の値（オブジェクトまたは未定義でなければならない）に設定します。対象値がオブジェクトでない場合、またはプロトタイプ値がオブジェクトまたは未定義 でない場合は、エラーをスローします。

ECMAScript のプロトタイプ操作プリミティブと異なり、本 API コールはプロトタイプループを作成することができます。Duktapeは長いプロトタイプ・チェーンやループしたプロトタイプ・チェーンを検出してエラーを投げますが、そのような動作は無限ループを避けるための最後の手段としてのみ意図されています。

### 例

```c
/* Create new object, internal prototype is Object.prototype by default. */
duk_push_object(ctx);

/* Change the object's internal prototype to object at index -3. */
duk_dup(ctx, -3);
duk_set_prototype(ctx, -2);  /* [ obj proto ] -> [ obj ] */
```

### 参照

duk_get_prototype


## duk_set_top() 

1.0.0 stack

### プロトタイプ

```c
void duk_set_top(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | -> | ... |

### 要約

スタックトップ（スタックサイズ）を引数idxに一致するように設定し、負のインデックス値を正規化します。バリュースタックが縮小する場合、新しいスタックトップより上の値は削除されます。バリュースタックが拡大する場合、未定義の値は新しいスタックスロットにプッシュされます。

負のインデックス値は、他のAPIコールと同様に、現在のスタックトップを基準として解釈されます。例えば、インデックス -1 は、スタックトップを 1 つ減らすことになります。


### 例

```c
/* Assume stack is empty initially. */

duk_push_int(ctx, 123);  /* -> top=1, stack: [ 123 ] */
duk_set_top(ctx, 3);     /* -> top=3, stack: [ 123 undefined undefined ] */
duk_set_top(ctx, -1);    /* -> top=2, stack: [ 123 undefined ] */
duk_set_top(ctx, 0);     /* -> top=0, stack: [ ] */
```


## duk_steal_buffer() 

1.3.0 stack buffer

### プロトタイプ

```c
void *duk_steal_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... |

### 要約

idx にあるダイナミックバッファの現在のアロケーションを盗用します。具体的には、Duktape は以前の割り当てを忘れ、バッファをゼロサイズ (および NULL データポインタ) にリセットします。以前の割り当てへのポインタが返され、以前の割り当ての長さは、非NULLの場合、out_sizeに書き込まれます。ダイナミックバッファ自体はバリュースタック上に残り、再利用することができます。呼び出し元は、duk_free() を使用して前の割り当てを解放する責任があります。Duktapeは、ガベージコレクションやDuktapeヒープが破壊されたときでさえ、以前の割り当てを解放しません。

このAPIコールは、動的バッファがバッファ操作アルゴリズムにおいて安全な一時的バッファと して使用される場合に便利です（エラーが発生した場合に備えて、自動的にメモリ管 理されます）。このようなアルゴリズムの最後に、呼び出し元はバッファを盗んで、ダイナミック・バッファ自体が解放されても、Duktapeガベージ・コレクションによって解放されないようにしたい場合があります。


### 例

```c
duk_size_t sz;
void *ptr;

ptr = duk_push_dynamic_buffer(ctx, 256);  /* initial size */
for (;;) {
    /* Error prone algorithm, resizes and appends to buffer.  If an error
     * occurs, the dynamic buffer is automatically freed.
     */

    /* ... */
    duk_resize_buffer(ctx, -1, new_size);
}

/* Algorithm is done, we want to own the buffer.  The duk_steal_buffer()
 * API call returns the final data pointer (in case it has been changed
 * by resizes etc).
 */
ptr = duk_steal_buffer(ctx, -1, &sz);

/* At this point the dynamic buffer is still on the value stack.
 * Its size is zero and the current allocation is empty.  Here we
 * just pop it as unneeded.
 */
duk_pop(ctx);

/* ... */

/* Eventually, when done with the buffer, you must free it yourself,
 * otherwise memory will be leaked.  Duktape won't free the allocation
 * automatically, even at Duktape heap destruction.
 */
duk_free(ctx, ptr);
```


## duk_strict_equals() 

1.0.0 compare

### プロトタイプ

```c
duk_bool_t duk_strict_equals(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

| ... | val1 | ... | val2 | ... |

### 要約

idx1 と idx2 の値が等しいかどうかを比較します。ECMAScript Strict Equals operator (===)のセマンティクスにより、値が等しいとみなされた場合は1を、そうでない場合は0を返す。また、どちらかのインデックスが無効な場合は0を返す。

Strict Equals 演算子が使用する Strict Equality Comparison Algorithm は値の強制を行わないので、比較に副作用はなく、エラーを発生させることもありません。

Duktapeカスタム型の比較アルゴリズムについては、厳密な等価性で説明されています。


### 例

```c
if (duk_strict_equals(ctx, -3, -7)) {
    printf("values at indices -3 and -7 are strictly equal\n");
}
```

### 参照

duk_equals


## duk_substring() 

1.0.0 string

### プロトタイプ

```c
void duk_substring(duk_context *ctx, duk_idx_t idx, duk_size_t start_char_offset, duk_size_t end_char_offset);
```

### スタック

| ... | str | ... | -> | ... | substr | ... |

### 要約

idx にある文字列を、文字列自身の部分文字列 [start_char_offset, end_char_offset[ ]で置換します。idxの値が文字列でない場合、またはインデックスが無効な場合は、エラーを投げます。。

部分文字列操作は、バイトではなく文字で動作し、オフセットは文字オフセットです。この呼び出しのセマンティクスは String.prototype.substring (start, end) に類似しています。オフセット値は文字列長にクランプされ（size_t は符号なしなので負の値をクランプする必要はない）、開始オフセットが終了オフセットより大きい場合、結果は空文字列です。

バイト単位の部分文字列を得るには、 duk_get_lstring() で文字列データポインタと長さを取得し、 duk_push_lstring() でバイトのスライスをプッシュします。

### 例

```c
/* String at index -3 is 'foobar'.  Substring [2,5[ is "oba". */

duk_substring(ctx, -3, 2, 5);
printf("substring: %s\n", duk_get_string(ctx, -3));
```


## duk_suspend() 

1.6.0 thread

### プロトタイプ

```c
void duk_suspend(duk_context *ctx, duk_thread_state *state);
```

### スタック

| ... | -> | ... | state(N) | (number of pushed stack entries may vary)

### 要約

別のネイティブ・スレッドが同じDuktapeヒープ上で操作できるように、現在のコール・スタックを一時停止します。必要な内部状態は、バリュースタックや提供された状態割り当てに格納されます。状態ポインタはNULLであってはならず、さもなければメモリ不安全な動作が発生します。実行は、後で duk_resume() を使って再開されなければなりません。後で実行が再開されない場合、いくつかの内部帳簿が矛盾した状態で残されます。Duktapeの実行が中断されている間、（duk_suspend()を呼び出した）現在のネイティブ・スレッドのネイティブ・コール・スタックは巻き戻されてはいけません。

このAPIコールは、直接または間接的に、以下の場所から使用してはならない。

ファイナライザー・コール
Duktape.errCreate()エラー補強コール
Duktapeは、一度に特定のDuktapeヒープにアクセスするネイティブ・スレッドだけを確保するためのロッキングを提供しません。アプリケーション・コードでそのようなメカニズムを提供する必要があります。
スレッディングを参照してください。


### 例

```c
/* Example of a blocking connect which suspends Duktape execution while the
 * connect blocks.  The example assumes an external locking mechanism for
 * ensuring only one native thread accesses the Duktape heap at a time.
 * When my_blocking_connect() is entered, the currently executing native
 * thread is assumed to have already obtained the lock.
 */

duk_ret_t my_blocking_connect(duk_context *ctx) {
    duk_thread_state st;
    const char *host;
    int port;
    int success;

    host = duk_require_string(ctx, 0);
    port = (int) duk_require_int(ctx, 1);

    /* Suspend the Duktape thread.  Once duk_suspend() returns you must
     * not call into the Duktape API before doing a duk_resume().
     */
    duk_suspend(ctx, &st);
    my_release_lock();

    /* Blocks until connect attempt is finished.  Another native thread
     * may call into Duktape while we're blocked provided that application
     * guarantees only thread does so at a time.
     */
    success = blocking_connect(host, port);

    /* When we want to resume execution we must ensure no other thread is
     * active for the Duktape heap.  We then call duk_resume() and proceed
     * normally.
     */
    my_acquire_lock();
    duk_resume(ctx, &st);

    if (!success) {
        duk_type_error(ctx, "failed to connect");
    }
    return 0;
}
```

### 参照

duk_resume


## duk_swap() 

1.0.0 stack

### プロトタイプ

```c
void duk_swap(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

| ... | val1 | ... | val2 | ... | -> | ... | val2 | ... | val1 | ... |

### 要約

インデックス idx1 と idx2 の値を入れ替えます。インデックスが同じであれば、この呼び出しは失敗です。どちらかのインデックスが無効な場合、エラーを投げます。。


### 例

```c
duk_swap(ctx, -5, -1);
```

### 参照

duk_swap_top


## duk_swap_top() 

1.0.0 stack

### プロトタイプ

```c
void duk_swap_top(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val1 | ... | val2 | -> | ... | val2 | ... | val1 |

### 要約

スタックの先頭とidxの値を入れ替えます。idxがスタックの先頭を参照している場合、この呼び出しはノー・オペレーションです。バリュースタックが空であるか、idxが無効である場合、エラーを投げます。。


### 例

```c
duk_swap_top(ctx, -3);
```


## duk_syntax_error() 

2.0.0 error

### プロトタイプ

```c
duk_ret_t duk_syntax_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_SYNTAX_ERROR を持つ duk_error() に相当する便利な API コールです。


### 例

```c
return duk_syntax_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_syntax_error_va() 

2.0.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_syntax_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_SYNTAX_ERROR を持つ duk_error_va() と同等の便宜的な API 呼び出し。

このAPIコールは完全には移植性がない。なぜなら、呼び出し側のコードの va_end() マクロは(例えばエラースローにより)決して到達できないかもしれないからです。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存します; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_syntax_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_syntax_error_va(ctx, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error_va


## duk_throw() 

1.0.0 error

### プロトタイプ

```c
duk_ret_t duk_throw(duk_context *ctx);
```

### スタック

| ... | val |

### 要約

スタックの一番上にある値を投げます。。この呼び出しは決して戻りません。

この関数は決して戻りませんが、プロトタイプは戻り値を記述しており、次のようなコードを可能にします。

```c
if (argvalue < 0) {
    duk_push_error_object(ctx, DUK_ERR_TYPE_ERROR, "invalid argument: %d", (int) argvalue);
    return duk_throw(ctx);
}
```

戻り値が無視される場合、コンパイル時の警告を避けるため、voidにキャストします。

```c
if (argvalue < 0) {
    duk_push_error_object(ctx, DUK_ERR_TYPE_ERROR, "invalid argument: %d", (int) argvalue);
    (void) duk_throw(ctx);
}
```

### 例

```c
/* Throw a string value; equivalent to the ECMAScript code:
 *
 *   throw "this string is thrown";
 */

duk_push_string(ctx, "this string is thrown");
(void) duk_throw(ctx);
```

### 参照

duk_error
duk_push_error_object


## duk_time_to_components() 

2.0.0 time

### プロトタイプ

```c
void duk_time_to_components(duk_context *ctx, duk_double_t time, duk_time_components *comp);
```

### スタック

(バリュースタックに影響なし。)


### 要約

UTCで解釈されるコンポーネント（年、月、日など）に時間値を変換します。時間値が無効な場合、例えば ECMAScript の有効な時間範囲を超えている場合、エラーがスローされます。

Date.prototype.getUTCMinutes() のような ECMAScript の Date UTC アクセッサといくつかの相違点があります。

時間値は、ミリ秒コンポーネントが分数を持つことができるように、分数（サブミリ秒の解像度）を持つことが許可されています。

### 例

```c
duk_double_t time = 1451703845006.0;  /* 2016-01-02 03:04:05.006Z */
duk_time_components comp;
const char *weekdays[7] = {
  "Sunday", "Monday", "Tuesday", "Wednesday",
  "Thursday", "Friday", "Saturday"
};

/* Note that month is zero-based to match the ECMAScript API, so for
 * human readable printing add 1 to the month.  Time components are
 * IEEE doubles to match ECMAScript Date behavior.
 */
duk_time_to_components(ctx, time, &comp);
printf("Datetime: %04d-%02d-%02d %02d:%02d:%02d.%03dZ\n",
       (int) comp.year, (int) comp.month + 1, (int) comp.day,
       (int) comp.hours, (int) comp.minutes, (int) comp.seconds,
       (int) comp.milliseconds);

printf("Weekday is: %s\n", weekdays[(int) comp.weekday]);  /* 0=Sunday */
```

### 参照

duk_components_to_time


## duk_to_boolean() 

1.0.0 stack

### プロトタイプ

```c
duk_bool_t duk_to_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToBoolean(val) | ... |

### 要約

idx の値を ECMAScript の ToBoolean() で強制された値で置き換えます。強制の結果が真であれば1を、そうでなければ0を返します。idx が無効な場合、エラーを投げます。。

カスタム型強制は型変換とテストに記載されています。


### 例

```c
if (duk_to_boolean(ctx, -3)) {
    printf("coerced value is true\n");
}
```


## duk_to_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_to_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... | -> | ... | buffer(val) | ... |

### 要約

idxの値をバッファで強制された値で置き換えます。バッファデータへのポインタ (サイズゼロのバッファの場合はNULLかもしれない) を返し、バッファのサイズを *out_size に書き込む (out_size が非NULLの場合)。idx が無効な場合，エラーを投げます。。

強制適用ルール。

バッファ: 変更なし、動的/固定/外部の性質は変更なし
文字列: 固定サイズのバッファに強制され、バイト-バイトになります。
その他の型: 最初に ECMAScript ToString() を適用し、その後、バイト単位の固定サイズバッファに強制変換する

### 例

```c
void *ptr;
duk_size_t sz;

ptr = duk_to_buffer(ctx, -3, &sz);
printf("coerced data at %p, size %lu\n", ptr, (unsigned long) sz);
```

### 参照

duk_to_fixed_buffer
duk_to_dynamic_buffer


## duk_to_dynamic_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_to_dynamic_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... |

### 要約

duk_to_buffer() と同様ですが、結果は常に動的バッファです (エラーがスローされない限り)。値が固定バッファまたは外部バッファである場合、それをダイナミックバッファに変換します。


### 例

```c
duk_size_t sz;
void *buf = duk_to_dynamic_buffer(ctx, -3, &sz);
```


## duk_to_fixed_buffer() 

1.0.0 stack buffer

### プロトタイプ

```c
void *duk_to_fixed_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

| ... | val | ... |

### 要約

duk_to_buffer() と同様ですが、値がダイナミックバッファまたは外部バッファである場合、それを固定バッファに変換します。したがって、結果は常に（エラーが投げられない限り）固定バッファになります。


### 例

```c
duk_size_t sz;
void *buf = duk_to_fixed_buffer(ctx, -3, &sz);
```


## duk_to_int() 

1.0.0 stack

### プロトタイプ

```c
duk_int_t duk_to_int(duk_context *ctx, duk_int_t index);
```

### スタック

| ... | val | ... | -> | ... | ToInteger(val) | ... |

### 要約

index の値を ECMAScript の ToInteger() で強制された値で置き換えます。duk_get_int() で説明したアルゴリズムを用いて、 ToInteger() の結果からさらに強制された duk_int_t を返します (この 2 番目の強制は、新しいスタック値には反映されません)。index が無効な場合、エラーを投げます。。

カスタムの型強制は、型変換とテストに記載されています。

もし、ToInteger() の強制された値の double バージョンが欲しいなら、これを使います。

```c
double d;

(void) duk_to_int(ctx, -3);
d = (double) duk_get_number(ctx, -3);
```

duk_get_int() の int coercion は戻り値のみに適用され、バリュースタックには反映されない。例えば、バリュースタックに文字列 "Infinity" がある場合、スタック上の値は数値 Infinity に強制され、DUK_INT_MAX が API コールから返されます。

### 例

```c
printf("ToInteger() + int coercion: %ld\n", (long) duk_to_int(ctx, -3));
printf("ToInteger() coercion: %lf\n", (double) duk_get_number(ctx, -3));
```


## duk_to_int32() 

1.0.0 stack

### プロトタイプ

```c
duk_int32_t duk_to_int32(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToInt32(val) | ... |

### 要約

idx の値を ECMAScript の ToInt32() で強制された値で置き換えます。強制された値を返す。idx が無効な場合、エラーを投げます。。

カスタム型強制は型変換とテストに記載されています。


### 例

```c
printf("ToInt32(): %ld\n", (long) duk_to_int32(ctx, -3));
```


## duk_to_lstring() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_to_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

| ... | val | ... | -> | ... | ToString(val) | ... |

### 要約

idx の値を ECMAScript の ToString() で強制された値で置き換えます。読み取り専用で NUL 終端の文字列データへの非 NULL ポインタを返し、文字列のバイト長を *out_len に書き込む (out_len が非 NULL の場合)。idx が無効な場合、エラーを投げます。。

カスタムの型強制については、型変換とテストに記載されています。


### 例

```c
const char *ptr;
duk_size_t sz;

ptr = duk_to_lstring(ctx, -3, &sz);
printf("coerced string: %s (length %lu)\n", ptr, (unsigned long) sz);
```

### 参照

duk_safe_to_lstring


## duk_to_null() 

1.0.0 stack

### プロトタイプ

```c
void duk_to_null(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | null | ... |

### 要約

前の値に関係なく、idxの値をnullに置き換えます。indexが無効な場合はエラーを投げます。。

duk_push_null() に続いて、ターゲット・インデックスに duk_replace() を実行するのと同等です。

### 例

```c
duk_to_null(ctx, -3);
```


## duk_to_number() 

1.0.0 stack

### プロトタイプ

```c
duk_double_t duk_to_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToNumber(val) | ... |

### 要約

idx の値を ECMAScript の ToNumber() で強制された値で置き換えます。強制された値を返す。idx が無効な場合、エラーを投げます。。

カスタム型強制は型変換とテストに記載されています。


### 例

```c
printf("coerced number: %lf\n", (double) duk_to_number(ctx, -3));
```


## duk_to_object() 

1.0.0 stack object

### プロトタイプ

```c
void duk_to_object(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToObject(val) | ... |

### 要約

idx の値を ECMAScript の ToObject() で強制された値で置き換えます。戻り値はない。idx の値がオブジェクト強制でない場合、または idx が無効な場合、エラーを投げます。。

以下の型はオブジェクト互換性がない。

- undefined
- null

カスタムタイプの動作: タイプアルゴリズム参照。特に、プレーンなバッファの値は、同等のUint8Arrayオブジェクトに変換されることに注意してください。


### 例

```c
duk_to_object(ctx, -3);
```


## duk_to_pointer() 

1.0.0 stack

### プロトタイプ

```c
void *duk_to_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | pointer(val) | ... |

### 要約

idxの値をポインタに強制された値で置き換えます。結果として得られる void * 値を返す。idxが無効な場合、エラーを投げます。。

強制適用ルール。

ポインタ: それ自身に強制され、変更はない。
すべてのヒープ割り当てオブジェクト（文字列、オブジェクト、バッファ）： Duktape内部ヒープ・ヘッダを指すポインタに強制されます（デバッグにのみ使用、読み書きは不可）。
その他の型：NULLに変換
このAPIコールは、実際にはデバッグにのみ有用です。特に、内部ヒープヘッダを指しているため、返されたポインタにアクセスしてはならないことに注意。これは、文字列/バッファ値の場合でも同様です。返されるポインタは、 duk_get_string() や duk_get_buffer() が返すものとは異なります。

### 例

```c
/* Don't dereference the pointer. */
printf("coerced pointer: %p\n", duk_to_pointer(ctx, -3));
```


## duk_to_primitive() 

1.0.0 stack

### プロトタイプ

```c
void duk_to_primitive(duk_context *ctx, duk_idx_t idx, duk_int_t hint);
```

### スタック

| ... | val | ... | -> | ... | ToPrimitive(val) |

### 要約

idx のオブジェクトを ECMAScript の ToPrimitive() で強制された値で置き換えます。hint 引数はオブジェクトのプリミティブ型への強制に影響し、文字列 (DUK_HINT_STRING)、数値 (DUK_HINT_NUMBER)、またはその両方 (DUK_HINT_NONE) を優先することを表します。DUK_HINT_NONE は数値を優先しますが、入力値が Date インスタンスの場合は文字列を優先します (正確な強制動作は ECMAScript 仕様に記載されています)。idx が無効な場合は、エラーを投げます。

カスタム型強制。

バッファ値 (プレーン): Uint8Array のように扱われ、通常は文字列 [object Uint8Array] に強制されます。
ポインタ値(プレーン): 既存の値を保持します。
ポインタオブジェクト: その下のプレーンなポインタ値に強制されます。
Lightfunc 値 (プレーン): 関数のように扱われ、通常、文字列 [object Function] に強制されます。
カスタムの型強制については、型変換とテストに記載されています。


### 例

```c
duk_to_primitive(ctx, -3, DUK_HINT_NUMBER);
```


## duk_to_stacktrace() 

2.4.0 string stack

### プロトタイプ

```c
const char *duk_to_stacktrace(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | val.stack | ... |

### 要約

任意の値をスタックトレースに出力します。idxの値はErrorインスタンスであることが期待されますが、必須ではありません。引数がオブジェクトの場合、この呼び出しは引数の .stack プロパティを検索し、プロパティ値が文字列であればその結果になります。そうでなければ、引数は文字列の結果を確実にするために duk_to_string(val) で置き換えられます。副作用やメモリ不足のために強制が失敗した場合、エラーが投げられるかもしれません。

この API 呼び出しの安全なバージョンとして、 duk_safe_to_stacktrace() があり、これは、エラーを処理する際に、より有用かもしれません。

ECMAScript の値強制に相当します。

```javascript
function duk_to_stacktrace(val) {
    if (typeof val === 'object' && val !== null) {  // Require val to be an object.
        var t = val.stack;  // Side effects, may throw.
        if (typeof t === 'string') {  // Require .stack to be a string.
            return t;
        }
    }
    return String(val);  // Side effects, may throw.
}
```

この強制は、別のグローバル環境（Realm）で作成された外部Errorオブジェクトでも動作するように、（継承に基づく）instanceofチェックを意図的に回避しています。


### 例

```c
if (duk_peval_string(ctx, "1 + 2 +") != 0) {  /* => SyntaxError */
    printf("failed: %s\n", duk_to_stacktrace(ctx, -1));  /* Note: may throw */
} else {
    printf("success\n");
}
duk_pop(ctx);
```

### 参照

duk_to_string
duk_safe_to_stacktrace


## duk_to_string() 

1.0.0 string stack

### プロトタイプ

```c
const char *duk_to_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToString(val) | ... |

### 要約

idx の値を ECMAScript の ToString() で強制された値で置き換えます。読み取り専用で NUL 終端の文字列データへの非 NULL ポインタを返します。idx が無効な場合、エラーを投げます。。

カスタムの型強制については、型変換とテストに記載されています。

Symbol値に対するToString()強制はTypeErrorを発生させます。
Duktape 2.xでは、プレーン・バッファはUint8Arrayオブジェクトを模倣しており、通常ToString()は "[object Uint8Array]" に強制されます。バッファやバッファ・オブジェクトの内容を文字列に変換するには、 duk_buffer_to_string() (Duktape 2.0 の新機能) を使ってください。

### 例

```c
printf("coerced string: %s\n", duk_to_string(ctx, -3));
```

### 参照

duk_safe_to_string
duk_buffer_to_string


## duk_to_uint() 

1.0.0 stack

### プロトタイプ

```c
duk_uint_t duk_to_uint(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToInteger(val) | ... |

### 要約

duk_to_int() と同様だが、戻り値の強制は duk_get_uint() と同様です。

duk_get_uint() の int強制は戻り値のみに適用され、バリュースタックには反映されない。例えば、バリュースタックに文字列 "Infinity" があった場合、スタック上の値は数値 Infinity に強制され、APIコールから DUK_UINT_MAX が返されることになります。

### 例

```c
printf("ToInteger() + uint coercion: %lu\n", (unsigned long) duk_to_uint(ctx, -3));
printf("ToInteger() coercion: %lf\n", (double) duk_get_number(ctx, -3));
```


## duk_to_uint16() 

1.0.0 stack

### プロトタイプ

```c
duk_uint16_t duk_to_uint16(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToUint16(val) | ... |

### 要約

idxの値をECMAScriptのToUint16()で強制された値で置き換えます。強制された値を返します。idxが無効な場合、エラーを投げます。。

カスタム型強制は型変換とテストに記載されています。


### 例

```c
printf("ToUint16(): %u\n", (unsigned int) duk_to_uint16(ctx, -3));
```


## duk_to_uint32() 

1.0.0 stack

### プロトタイプ

```c
duk_uint32_t duk_to_uint32(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | ToUint32(val) | ... |

### 要約

idxの値をECMAScriptのToUint32()で強制された値で置き換えます。強制された値を返します。idxが無効な場合、エラーを投げます。。

カスタム型強制は型変換とテストに記載されています。


### 例

```c
printf("ToUint32(): %lu\n", (unsigned long) duk_to_uint32(ctx, -3));
```


## duk_to_undefined() 

1.0.0 stack

### プロトタイプ

```c
void duk_to_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | val | ... | -> | ... | undefined | ... |

### 要約

前の値に関係なく、idxの値をundefinedに置き換えます。indexが無効な場合はエラーを投げます。。

duk_push_undefined() の後に duk_replace() を実行し、対象インデックスを指定するのと 同等。

### 例

```c
duk_to_undefined(ctx, -3);
```


## duk_trim() 

1.0.0 string stack

### プロトタイプ

```c
void duk_trim(duk_context *ctx, duk_idx_t idx);
```

### スタック

| ... | str | ... | -> | ... | trimmed_str | ... |

### 要約

idxの文字列をトリミングします。idx の値が文字列でない場合、またはインデックスが無効な場合は、エラーを投げます。。

トリミングは、文字列の先頭と末尾の空白文字を削除します。文字列がすべて空白文字で構成されている場合、結果は空文字列となります。ホワイトスペースとみなされる文字のセットは、WhiteSpaceとLineTerminatorの両方を含むStrWhiteSpaceプロダクションによって定義されます。トリミングの動作は String.prototype.trim(), parseInt(), parseFloat() の動作と一致します。


### 例

```c
duk_trim(ctx, -3);
```


## duk_type_error() 

2.0.0 error

### プロトタイプ

```c
duk_ret_t duk_type_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_TYPE_ERROR を持つ duk_error() に相当する便利な API コールです。


### 例

```c
return duk_type_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_type_error_va() 

2.0.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_type_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_TYPE_ERROR を持つ duk_error_va() と同等の便宜的な API 呼び出し。

このAPIコールは完全には移植性がない。なぜなら、呼び出し側のコードの va_end() マクロは(例えばエラースローにより)到達しないかもしれないからです。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存します; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_type_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_type_error_va(ctx, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error_va


## duk_uri_error() 

2.0.0 error

### プロトタイプ

```c
duk_ret_t duk_uri_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_URI_ERROR を持つ duk_error() に相当する便利な API コールです。


### 例

```c
return duk_uri_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_uri_error_va() 

2.0.0 vararg error

### プロトタイプ

```c
duk_ret_t duk_uri_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

| ... | -> | ... | err |

### 要約

エラーコード DUK_ERR_URI_ERROR を持つ duk_error_va() と同等の便宜的な API 呼び出し。

このAPIコールは完全には移植性がない。なぜなら、呼び出し側のコードの va_end() マクロは(例えばエラースローにより)決して到達できないかもしれないからです。いくつかの実装は、例えば va_start() によって割り当てられたメモリを解放するために va_end() に依存します; https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown を参照してください。しかし、そのような実装はまれであり、これは通常実用的な懸念ではありません。

### 例

```c
void my_uri_error(duk_context *ctx, const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    duk_uri_error_va(ctx, fmt, ap);
    va_end(ap);
}
```

### 参照

duk_error_va


## duk_xcopy_top() 

1.0.0 stack slice

### プロトタイプ

```c
void duk_xcopy_top(duk_context *to_ctx, duk_context *from_ctx, duk_idx_t count);
```

### スタック

| ... | val1 | ... | valN | -> | ... | val1 | ... | valN | (on source stack, from_ctx)
| ... | -> | ... | val1 | ... | valN | (on target stack, to_ctx)

### 要約

duk_xmove_top() と同様ですが、コピーされる要素はコピー元スタックからポップアップされませ ん。コピー元とコピー先の両方のスタックは、同じ Duktape ヒープに存在しなければなりません。

Lua の lua_xmove() と比較して、スタックからの順序とスタックへの順序が逆になっています。

### 例

```c
duk_xcopy_top(new_ctx, ctx, 7);
```

### 参照

duk_xmove_top


## duk_xmove_top() 

1.0.0 stack slice

### プロトタイプ

```c
void duk_xmove_top(duk_context *to_ctx, duk_context *from_ctx, duk_idx_t count);
```

### スタック

| ... | val1 | ... | valN | -> | ... | (on source stack, from_ctx)
| ... | -> | ... | val1 | ... | valN | (on target stack, to_ctx)

### 要約

ソーススタックの最上位から count 引数を取り除き、ターゲットスタックにプッシュします。呼び出し側は、例えば duk_require_stack() を使って、ターゲット・スタックに十分な割り当て領域があることを確認しなければなりません。ソースとターゲットの両方のスタックは、同じ Duktape ヒープに存在しなければなりません。

もしソースとターゲットのスタックが同じであれば、現在エラーが投げられています。

Lua の lua_xmove() と比較して、スタックからスタックへの順序が逆になっています。

### 例

```c
/* A Duktape/C function which executes a given function in a new thread.
 */
static duk_ret_t call_in_thread(duk_context *ctx) {
    duk_idx_t nargs;
    duk_context *new_ctx;

    /* Arguments: func, arg1, ... argN. */
    nargs = duk_get_top(ctx);
    if (nargs < 1) {
        return DUK_RET_TYPE_ERROR;  /* missing func | argument */
    }

    /* Create a new context. */
    duk_push_thread();
    new_ctx = duk_require_context(ctx, -1);

    /* Move arguments to the new context.  Note that we need to extend
     * the target stack allocation explicitly.
     */
    duk_require_stack(new_ctx, nargs);
    duk_xmove_top(new_ctx, ctx, nargs);

    /* Call the function; new_ctx is now: [ func | arg1 ... argN ]. */
    duk_call(new_ctx, nargs - 1);

    /* Return the function call result by copying it to the original stack. */
    duk_xmove_top(ctx, new_ctx, 1);
    return 1;
}
```

### 参照

duk_xcopy_top