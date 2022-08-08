# Duktape API


## はじめに

バージョン: 2.6.0 (2020-10-13)


### ドキュメントスコープ

Duktape API（duktape.h で定義）は、C/C++ プログラムが ECMAScript コードとインターフェースすることを可能にし、値表現のような内部詳細からそれらを保護する定数と API 呼び出しのセットです。

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

この文書では、以下のスタック表記を使用します。スタックは、右に行くほど大きくなるように視覚的に表現される。例えば、次のように。

```c
duk_push_number(ctx, 123);
duk_push_string(ctx, "foo");
duk_push_true(ctx);
```


スタックは次のようになります。

```
| 123 | "foo" | true |
```


スタック上の要素のうち、操作に影響を与えないものは、1つの要素に省略記号（"..."）を付けて表記している。

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


スタック変換は，矢印と2つのスタックで表現する．

```
| ... | <obj> | ... | <key> | <value> |  ->  | ... | <obj> | ... |
```


## 概念

### ヒープ

ガベージコレクションのための単一の領域。1つ以上のコンテキストで共有される。


### コンテキスト（Context）

Duktape API で使用されるハンドルで、Duktape スレッド（コルーチン）およびそのコールスタックとバリュースタックに関連付けられます。


### (コンテキストの)コールスタック

コンテキストの）呼び出しスタック § コンテキストのアクティブな関数呼び出し連鎖の帳簿。ECMAScript と Duktape/C の両方の関数呼び出しが含まれます。

### バリュースタック（コンテキストの）§。
コンテキストの）値スタック § コンテキストの呼び出しスタックで、現在のアクティブ化に属するタグ付けされた値のためのストレージ。

値スタックに保持される値は、伝統的なタグ付き型です。スタック・エントリーは、最も最近の関数呼び出しの下 (>= 0) または上 (< 0) からインデックスが付けられます。

### 値スタックのインデックス

非負 (>= 0) のインデックスは、現在のスタックフレームのスタックエントリを、フレームの底を基準にして指します。

```
| 0 | 1 | 2 | 3 | 4 | <5> | 
```


負（< 0）のインデックスは、フレームトップからの相対的なスタックエントリを指します。

```
| -6 | -5 | -4 | -3 | -2 | <-1> | 
```


特殊定数 DUK_INVALID_INDEX は、無効なスタックインデックスを示す負の整数である。これは API 呼び出しから返されることがあり、また「値がない」ことを示すためにいくつかの API 呼び出しに与えられることがあります。

値スタックトップ(または単に「トップ」)は、最高使用インデックスのすぐ上の虚数要素の非負のインデックスである。例えば、最も使用されているインデックスの上は 5 であるので、スタックトップは 6 である。 トップは現在のスタックサイズを示し、スタックにプッシュされる次の要素のインデックスでもある。

```
| 0 | 1 | 2 | 3 | 4 | <5> | (6) |
```


> API のスタック操作は、常に現在のスタックフレームに限定される。現在のフレームより下のスタックエントリを参照する方法はない。これは、コールスタック内の関数が互いの値に影響を与えないようにするためで、意図的なものです。


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


Heap alloc" カラムは、タグ付けされた値がヒープで割り当てられたオブジェクトを指しているかどうかを示し、（オブジェクトのプロパティテーブルのように）追加で割り当てられる可能性があることを示す。

### スタックタイプマスク

各スタックタイプにはビットインデックスがあり、スタックタイプのセットをビットマスクとして表現することができます。呼び出し側のコードはこのようなビットセットを使って、例えばあるスタック値があるタイプのセットに属するかどうかをチェックすることができます。例

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

Duktape/C API 署名のある C 関数は、ECMAScript 関数オブジェクトまたは lightfunc 値と関連付けることができ、関連付けられた値が ECMAScript コードから呼び出されたときに呼び出されます。例

```c
duk_ret_t my_func(duk_context *ctx) {
    duk_push_int(ctx, 123);
    return 1;
}
```

値スタックで与えられる引数の数は、対応する ECMAScript 関数オブジェクトが作成されるときに指定されます: 引数を "そのまま" 取得するための DUK_VARARGS の固定数の引数のどちらかです。

- 固定数の引数を使用する場合、余分な引数は削除され、不足する引数は必要に応じて undefined で埋められます。
- DUK_VARARGS を使用する場合、実際の引数の数を決定するために duk_get_top() を使用します。

関数の戻り値。

- 戻り値1：スタックトップが戻り値を含む。
- 戻り値0：戻り値は未定義である。
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

- DUK_ERR_NONE: No error, e.g. from duk_get_error_code()
- DUK_ERR_ERROR: Error
- DUK_ERR_EVAL_ERROR: EvalError
- DUK_ERR_RANGE_ERROR: RangeError
- DUK_ERR_REFERENCE_ERROR: ReferenceError
- DUK_ERR_SYNTAX_ERROR: SyntaxError
- DUK_ERR_TYPE_ERROR: TypeError
- DUK_ERR_URI_ERROR: URIError


### Duktape/C関数のリターンコード

例えば return DUK_RET_TYPE_ERROR は duk_type_error() を呼び出すのと似ていますが、より短いものです。

- DUK_RET_ERROR: Similar to throwing with DUK_ERR_ERROR
- DUK_RET_EVAL_ERROR: Similar to throwing with DUK_ERR_EVAL_ERROR
- DUK_RET_RANGE_ERROR: Similar to throwing with DUK_ERR_RANGE_ERROR
- DUK_RET_REFERENCE_ERROR: Similar to throwing with DUK_ERR_REFERENCE_ERROR
- DUK_RET_SYNTAX_ERROR: Similar to throwing with DUK_ERR_SYNTAX_ERROR
- DUK_RET_TYPE_ERROR: Similar to throwing with DUK_ERR_TYPE_ERROR
- DUK_RET_URI_ERROR: Similar to throwing with DUK_ERR_URI_ERROR


### 保護された呼び出しのためのリターンコード

保護された呼び出し（例：duk_safe_call()、duk_pcall()）に対するリターンコード。

- DUK_EXEC_SUCCESS: Call finished without error
- DUK_EXEC_ERROR: Call failed, error was caught


### duk_compile() のためのコンパイルフラグ §。

duk_compile() や duk_eval() などのためのコンパイルフラグです。

- DUK_COMPILE_EVAL: Compile eval code (instead of program)
- DUK_COMPILE_FUNCTION: Compile function code (instead of program)
- DUK_COMPILE_STRICT: Use strict (outer) context for program, eval, or function


### duk_def_prop() のフラグについて

duk_def_prop() とその派生型のためのフラグです。

- DUK_DEFPROP_WRITABLE: Set writable (effective if DUK_DEFPROP_HAVE_WRITABLE set)
- DUK_DEFPROP_ENUMERABLE: Set enumerable (effective if DUK_DEFPROP_HAVE_ENUMERABLE set)
- DUK_DEFPROP_CONFIGURABLE: Set configurable (effective if DUK_DEFPROP_HAVE_CONFIGURABLE set)
- DUK_DEFPROP_HAVE_WRITABLE: Set/clear writable
- DUK_DEFPROP_HAVE_ENUMERABLE: Set/clear enumerable
- DUK_DEFPROP_HAVE_CONFIGURABLE: Set/clear configurable
- DUK_DEFPROP_HAVE_VALUE: Set value (given on value stack)
- DUK_DEFPROP_HAVE_GETTER: Set getter (given on value stack)
- DUK_DEFPROP_HAVE_SETTER: Set setter (given on value stack)
- DUK_DEFPROP_FORCE: Force change if possible, may still fail for e.g. virtual properties
- DUK_DEFPROP_SET_WRITABLE: (DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_WRITABLE)
- DUK_DEFPROP_CLEAR_WRITABLE: DUK_DEFPROP_HAVE_WRITABLE
- DUK_DEFPROP_SET_ENUMERABLE: (DUK_DEFPROP_HAVE_ENUMERABLE | DUK_DEFPROP_ENUMERABLE)
- DUK_DEFPROP_CLEAR_ENUMERABLE: DUK_DEFPROP_HAVE_ENUMERABLE
- DUK_DEFPROP_SET_CONFIGURABLE: (DUK_DEFPROP_HAVE_CONFIGURABLE | DUK_DEFPROP_CONFIGURABLE)
- DUK_DEFPROP_CLEAR_CONFIGURABLE: DUK_DEFPROP_HAVE_CONFIGURABLE

いくつかの便利なバリアントは省略され、 duk_def_prop() を参照。


### duk_enum() の列挙型フラグ

duk_enum() の列挙フラグ。

- DUK_ENUM_INCLUDE_NONENUMERABLE: Enumerate non-numerable properties in addition to enumerable
- DUK_ENUM_INCLUDE_HIDDEN: Enumerate hidden Symbols too (in Duktape 1.x called internal properties)
- DUK_ENUM_INCLUDE_SYMBOLS: Enumerate Symbol keys (default is not to enumerate them)
- DUK_ENUM_EXCLUDE_STRINGS: Do not enumerate string keys (default is to enumerate them)
- DUK_ENUM_OWN_PROPERTIES_ONLY: Don't walk prototype chain, only check own properties
- DUK_ENUM_ARRAY_INDICES_ONLY: Only enumerate array indices
- DUK_ENUM_SORT_ARRAY_INDICES: Sort array indices (applied to full enumeration result, including inherited array indices)
- DUK_ENUM_NO_PROXY_BEHAVIOR: Enumerate a proxy object itself without invoking proxy behavior


### duk_gc() のガーベッジコレクションフラグ

duk_gc() のフラグ。

- DUK_GC_COMPACT: Compact heap objects


### 強制力のヒント

強制力のヒント

- DUK_HINT_NONE: Prefer number, unless coercion input is a Date, in which case prefer string (E5 Section 8.12.8)
- DUK_HINT_STRING: Prefer string
- DUK_HINT_NUMBER: Prefer number


### シンボルリテラルマクロ

以下のマクロは、C リテラルとして内部 Symbol 表現を作成するために定義されています。すべての引数は文字列リテラルでなければならず、計算値であってはなりません。

- DUK_HIDDEN_SYMBOL(x): A C literal for a Duktape specific hidden Symbol
- DUK_GLOBAL_SYMBOL(x): A C literal for a global symbol, equivalent to Symbol.for(x)
- DUK_LOCAL_SYMBOL(x,uniq): A C literal for a local symbol, equivalent to Symbol(x), unique part provided in 'uniq' must not conflict with Duktape internal format, recommendation is to prefix the unique part with a "!"
- DUK_WELLKNOWN_SYMBOL(x): A C literal for a well-known symbol like Symbol.iterator
- DUK_INTERNAL_SYMBOL(x): A C literal for a Duktape internal symbol; an application shouldn't normally use this macro at all, it is reserved for Duktape internal symbols only (with no versioning guarantees)


### その他の定義

- DUK_INVALID_INDEX: Stack index is invalid, missing, or n/a
- DUK_VARARGS: Function takes variable arguments
- DUK_API_ENTRY_STACK: Number of value stack entries guaranteed to be reserved at function entry


### タグ別APIコール

- 1.0.0
 - duk_alloc
 - duk_alloc_raw
 - duk_base64_decode
 - duk_base64_encode
 - duk_call
 - duk_call_method
 - duk_call_prop
 - duk_char_code_at
 - duk_check_stack
 - duk_check_stack_top
 - duk_check_type
 - duk_check_type_mask
 - duk_compact
 - duk_compile
 - duk_compile_lstring
 - duk_compile_lstring_filename
 - duk_compile_string
 - duk_compile_string_filename
 - duk_concat
 - duk_copy
 - duk_create_heap
 - duk_create_heap_default
 - duk_decode_string
 - duk_del_prop
 - duk_del_prop_index
 - duk_del_prop_string
 - duk_destroy_heap
 - duk_dup
 - duk_dup_top
 - duk_enum
 - duk_equals
 - duk_error
 - duk_eval
 - duk_eval_lstring
 - duk_eval_lstring_noresult
 - duk_eval_noresult
 - duk_eval_string
 - duk_eval_string_noresult
 - duk_fatal
 - duk_free
 - duk_free_raw
 - duk_gc
 - duk_get_boolean
 - duk_get_buffer
 - duk_get_c_function
 - duk_get_context
 - duk_get_current_magic
 - duk_get_finalizer
 - duk_get_global_string
 - duk_get_int
 - duk_get_length
 - duk_get_lstring
 - duk_get_magic
 - duk_get_memory_functions
 - duk_get_number
 - duk_get_pointer
 - duk_get_prop
 - duk_get_prop_index
 - duk_get_prop_string
 - duk_get_prototype
 - duk_get_string
 - duk_get_top
 - duk_get_top_index
 - duk_get_type
 - duk_get_type_mask
 - duk_get_uint
 - duk_has_prop
 - duk_has_prop_index
 - duk_has_prop_string
 - duk_hex_decode
 - duk_hex_encode
 - duk_insert
 - duk_is_array
 - duk_is_boolean
 - duk_is_bound_function
 - duk_is_buffer
 - duk_is_c_function
 - duk_is_callable
 - duk_is_constructor_call
 - duk_is_dynamic_buffer
 - duk_is_ecmascript_function
 - duk_is_fixed_buffer
 - duk_is_function
 - duk_is_nan
 - duk_is_null
 - duk_is_null_or_undefined
 - duk_is_number
 - duk_is_object
 - duk_is_object_coercible
 - duk_is_pointer
 - duk_is_primitive
 - duk_is_strict_call
 - duk_is_string
 - duk_is_thread
 - duk_is_undefined
 - duk_is_valid_index
 - duk_join
 - duk_json_decode
 - duk_json_encode
 - duk_map_string
 - duk_new
 - duk_next
 - duk_normalize_index
 - duk_pcall
 - duk_pcall_method
 - duk_pcall_prop
 - duk_pcompile
 - duk_pcompile_lstring
 - duk_pcompile_lstring_filename
 - duk_pcompile_string
 - duk_pcompile_string_filename
 - duk_peval
 - duk_peval_lstring
 - duk_peval_lstring_noresult
 - duk_peval_noresult
 - duk_peval_string
 - duk_peval_string_noresult
 - duk_pop
 - duk_pop_2
 - duk_pop_3
 - duk_pop_n
 - duk_push_array
 - duk_push_boolean
 - duk_push_buffer
 - duk_push_c_function
 - duk_push_context_dump
 - duk_push_current_function
 - duk_push_current_thread
 - duk_push_dynamic_buffer
 - duk_push_error_object
 - duk_push_false
 - duk_push_fixed_buffer
 - duk_push_global_object
 - duk_push_global_stash
 - duk_push_heap_stash
 - duk_push_int
 - duk_push_lstring
 - duk_push_nan
 - duk_push_null
 - duk_push_number
 - duk_push_object
 - duk_push_pointer
 - duk_push_sprintf
 - duk_push_string
 - duk_push_this
 - duk_push_thread
 - duk_push_thread_new_globalenv
 - duk_push_thread_stash
 - duk_push_true
 - duk_push_uint
 - duk_push_undefined
 - duk_push_vsprintf
 - duk_put_function_list
 - duk_put_global_string
 - duk_put_number_list
 - duk_put_prop
 - duk_put_prop_index
 - duk_put_prop_string
 - duk_realloc
 - duk_realloc_raw
 - duk_remove
 - duk_replace
 - duk_require_boolean
 - duk_require_buffer
 - duk_require_c_function
 - duk_require_context
 - duk_require_int
 - duk_require_lstring
 - duk_require_normalize_index
 - duk_require_null
 - duk_require_number
 - duk_require_object_coercible
 - duk_require_pointer
 - duk_require_stack
 - duk_require_stack_top
 - duk_require_string
 - duk_require_top_index
 - duk_require_type_mask
 - duk_require_uint
 - duk_require_undefined
 - duk_require_valid_index
 - duk_resize_buffer
 - duk_safe_call
 - duk_safe_to_lstring
 - duk_safe_to_string
 - duk_set_finalizer
 - duk_set_global_object
 - duk_set_magic
 - duk_set_prototype
 - duk_set_top
 - duk_strict_equals
 - duk_substring
 - duk_swap
 - duk_swap_top
 - duk_throw
 - duk_to_boolean
 - duk_to_buffer
 - duk_to_dynamic_buffer
 - duk_to_fixed_buffer
 - duk_to_int
 - duk_to_int32
 - duk_to_lstring
 - duk_to_null
 - duk_to_number
 - duk_to_object
 - duk_to_pointer
 - duk_to_primitive
 - duk_to_string
 - duk_to_uint
 - duk_to_uint16
 - duk_to_uint32
 - duk_to_undefined
 - duk_trim
 - duk_xcopy_top
 - duk_xmove_top
- 1.1.0
 - duk_def_prop
 - duk_error_va
 - duk_get_error_code
 - duk_get_heapptr
 - duk_is_error
 - duk_push_error_object_va
 - duk_push_heapptr
 - duk_require_heapptr
- 1.2.0
 - duk_debugger_attach
 - duk_debugger_cooperate
 - duk_debugger_detach
- 1.3.0
 - duk_config_buffer
 - duk_dump_function
 - duk_get_buffer_data
 - duk_instanceof
 - duk_load_function
 - duk_pnew
 - duk_push_buffer_object
 - duk_push_external_buffer
 - duk_require_buffer_data
 - duk_steal_buffer
- 1.4.0
 - duk_is_eval_error
 - duk_is_range_error
 - duk_is_reference_error
 - duk_is_syntax_error
 - duk_is_type_error
 - duk_is_uri_error
 - duk_require_callable
 - duk_require_function
- 1.5.0
 - duk_debugger_notify
 - duk_debugger_pause
- 1.6.0
 - duk_resume
 - duk_suspend
- 2.0.0
 - duk_buffer_to_string
 - duk_components_to_time
 - duk_del_prop_lstring
 - duk_eval_error
 - duk_eval_error_va
 - duk_generic_error
 - duk_generic_error_va
 - duk_get_global_lstring
 - duk_get_now
 - duk_get_prop_desc
 - duk_get_prop_lstring
 - duk_has_prop_lstring
 - duk_inspect_callstack_entry
 - duk_inspect_value
 - duk_is_buffer_data
 - duk_is_symbol
 - duk_push_bare_object
 - duk_put_global_lstring
 - duk_put_prop_lstring
 - duk_range_error
 - duk_range_error_va
 - duk_reference_error
 - duk_reference_error_va
 - duk_samevalue
 - duk_set_length
 - duk_syntax_error
 - duk_syntax_error_va
 - duk_time_to_components
 - duk_type_error
 - duk_type_error_va
 - duk_uri_error
 - duk_uri_error_va
- 2.1.0
 - duk_get_boolean_default
 - duk_get_buffer_data_default
 - duk_get_buffer_default
 - duk_get_c_function_default
 - duk_get_context_default
 - duk_get_heapptr_default
 - duk_get_int_default
 - duk_get_lstring_default
 - duk_get_number_default
 - duk_get_pointer_default
 - duk_get_string_default
 - duk_get_uint_default
 - duk_opt_boolean
 - duk_opt_buffer
 - duk_opt_buffer_data
 - duk_opt_c_function
 - duk_opt_context
 - duk_opt_heapptr
 - duk_opt_int
 - duk_opt_lstring
 - duk_opt_number
 - duk_opt_pointer
 - duk_opt_string
 - duk_opt_uint
- 2.2.0
 - duk_del_prop_heapptr
 - duk_freeze
 - duk_get_prop_heapptr
 - duk_has_prop_heapptr
 - duk_is_constructable
 - duk_push_proxy
 - duk_put_prop_heapptr
 - duk_require_object
 - duk_seal
- 2.3.0
 - duk_del_prop_literal
 - duk_get_global_heapptr
 - duk_get_global_literal
 - duk_get_prop_literal
 - duk_has_prop_literal
 - duk_push_literal
 - duk_push_new_target
 - duk_put_global_heapptr
 - duk_put_global_literal
 - duk_put_prop_literal
 - duk_random
- 2.4.0
 - duk_push_bare_array
 - duk_require_constructable
 - duk_require_constructor_call
 - duk_safe_to_stacktrace
 - duk_to_stacktrace
- 2.5.0
 - duk_cbor_decode
 - duk_cbor_encode
 - duk_pull
- base64
 - duk_base64_decode
 - duk_base64_encode
- borrowed
 - duk_del_prop_heapptr
 - duk_get_context
 - duk_get_context_default
 - duk_get_global_heapptr
 - duk_get_heapptr
 - duk_get_heapptr_default
 - duk_get_prop_heapptr
 - duk_has_prop_heapptr
 - duk_opt_context
 - duk_opt_heapptr
 - duk_push_current_thread
 - duk_push_heapptr
 - duk_push_thread
 - duk_push_thread_new_globalenv
 - duk_put_global_heapptr
 - duk_put_prop_heapptr
 - duk_require_context
 - duk_require_heapptr
- buffer
 - duk_buffer_to_string
 - duk_config_buffer
 - duk_get_buffer
 - duk_get_buffer_data
 - duk_get_buffer_data_default
 - duk_get_buffer_default
 - duk_is_buffer
 - duk_is_buffer_data
 - duk_is_dynamic_buffer
 - duk_is_fixed_buffer
 - duk_opt_buffer
 - duk_opt_buffer_data
 - duk_push_buffer
 - duk_push_dynamic_buffer
 - duk_push_external_buffer
 - duk_push_fixed_buffer
 - duk_require_buffer
 - duk_require_buffer_data
 - duk_resize_buffer
 - duk_steal_buffer
 - duk_to_buffer
 - duk_to_dynamic_buffer
 - duk_to_fixed_buffer
- bufferobject
 - duk_get_buffer_data
 - duk_is_buffer_data
 - duk_opt_buffer_data
 - duk_push_buffer_object
 - duk_require_buffer_data
- bytecode
 - duk_dump_function
 - duk_load_function
- call
 - duk_call
 - duk_call_method
 - duk_call_prop
 - duk_new
 - duk_pcall
 - duk_pcall_method
 - duk_pcall_prop
 - duk_pnew
 - duk_safe_call
- cbor
 - duk_cbor_decode
 - duk_cbor_encode
- codec
 - duk_base64_decode
 - duk_base64_encode
 - duk_cbor_decode
 - duk_cbor_encode
 - duk_hex_decode
 - duk_hex_encode
 - duk_json_decode
 - duk_json_encode
- compare
 - duk_equals
 - duk_instanceof
 - duk_samevalue
 - duk_strict_equals
- compile
 - duk_compile
 - duk_compile_lstring
 - duk_compile_lstring_filename
 - duk_compile_string
 - duk_compile_string_filename
 - duk_eval
 - duk_eval_lstring
 - duk_eval_lstring_noresult
 - duk_eval_noresult
 - duk_eval_string
 - duk_eval_string_noresult
 - duk_pcompile
 - duk_pcompile_lstring
 - duk_pcompile_lstring_filename
 - duk_pcompile_string
 - duk_pcompile_string_filename
 - duk_peval
 - duk_peval_lstring
 - duk_peval_lstring_noresult
 - duk_peval_noresult
 - duk_peval_string
 - duk_peval_string_noresult
- debug
 - duk_push_context_dump
- debugger
 - duk_debugger_attach
 - duk_debugger_cooperate
 - duk_debugger_detach
 - duk_debugger_notify
 - duk_debugger_pause
- error
 - duk_error
 - duk_error_va
 - duk_eval_error
 - duk_eval_error_va
 - duk_fatal
 - duk_generic_error
 - duk_generic_error_va
 - duk_get_error_code
 - duk_is_error
 - duk_is_eval_error
 - duk_is_range_error
 - duk_is_reference_error
 - duk_is_syntax_error
 - duk_is_type_error
 - duk_is_uri_error
 - duk_push_error_object
 - duk_push_error_object_va
 - duk_range_error
 - duk_range_error_va
 - duk_reference_error
 - duk_reference_error_va
 - duk_syntax_error
 - duk_syntax_error_va
 - duk_throw
 - duk_type_error
 - duk_type_error_va
 - duk_uri_error
 - duk_uri_error_va
- experimental
 - duk_cbor_decode
 - duk_cbor_encode
- finalizer
 - duk_get_finalizer
 - duk_set_finalizer
- function
 - duk_get_c_function
 - duk_get_c_function_default
 - duk_get_current_magic
 - duk_get_magic
 - duk_is_bound_function
 - duk_is_c_function
 - duk_is_ecmascript_function
 - duk_is_function
 - duk_is_strict_call
 - duk_opt_c_function
 - duk_push_c_function
 - duk_push_c_lightfunc
 - duk_push_current_function
 - duk_push_current_thread
 - duk_push_new_target
 - duk_push_this
 - duk_require_c_function
 - duk_set_magic
- heap
 - duk_create_heap
 - duk_create_heap_default
 - duk_destroy_heap
 - duk_gc
 - duk_get_memory_functions
- heapptr
 - duk_del_prop_heapptr
 - duk_get_global_heapptr
 - duk_get_heapptr
 - duk_get_heapptr_default
 - duk_get_prop_heapptr
 - duk_has_prop_heapptr
 - duk_opt_heapptr
 - duk_push_heapptr
 - duk_put_global_heapptr
 - duk_put_prop_heapptr
 - duk_require_heapptr
- hex
 - duk_hex_decode
 - duk_hex_encode
- inspect
 - duk_inspect_callstack_entry
 - duk_inspect_value
- json
 - duk_json_decode
 - duk_json_encode
- lightfunc
 - duk_push_c_lightfunc
- literal
 - duk_del_prop_literal
 - duk_get_global_literal
 - duk_get_prop_literal
 - duk_has_prop_literal
 - duk_push_literal
 - duk_put_global_literal
 - duk_put_prop_literal
- magic
 - duk_get_current_magic
 - duk_get_magic
 - duk_set_magic
- memory
 - duk_alloc
 - duk_alloc_raw
 - duk_compact
 - duk_free
 - duk_free_raw
 - duk_gc
 - duk_get_memory_functions
 - duk_realloc
 - duk_realloc_raw
- module
 - duk_push_global_stash
 - duk_push_heap_stash
 - duk_push_thread_stash
 - duk_put_function_list
 - duk_put_number_list
- object
 - duk_compact
 - duk_enum
 - duk_freeze
 - duk_get_finalizer
 - duk_get_prototype
 - duk_is_object
 - duk_is_object_coercible
 - duk_new
 - duk_next
 - duk_pnew
 - duk_push_array
 - duk_push_bare_array
 - duk_push_bare_object
 - duk_push_error_object
 - duk_push_error_object_va
 - duk_push_global_object
 - duk_push_global_stash
 - duk_push_heap_stash
 - duk_push_heapptr
 - duk_push_object
 - duk_push_proxy
 - duk_push_thread_stash
 - duk_require_object_coercible
 - duk_seal
 - duk_set_finalizer
 - duk_set_prototype
 - duk_to_object
- property
 - duk_call_prop
 - duk_compact
 - duk_def_prop
 - duk_del_prop
 - duk_del_prop_heapptr
 - duk_del_prop_index
 - duk_del_prop_literal
 - duk_del_prop_lstring
 - duk_del_prop_string
 - duk_enum
 - duk_freeze
 - duk_get_global_heapptr
 - duk_get_global_literal
 - duk_get_global_lstring
 - duk_get_global_string
 - duk_get_prop
 - duk_get_prop_desc
 - duk_get_prop_heapptr
 - duk_get_prop_index
 - duk_get_prop_literal
 - duk_get_prop_lstring
 - duk_get_prop_string
 - duk_has_prop
 - duk_has_prop_heapptr
 - duk_has_prop_index
 - duk_has_prop_literal
 - duk_has_prop_lstring
 - duk_has_prop_string
 - duk_next
 - duk_pcall_prop
 - duk_put_function_list
 - duk_put_global_heapptr
 - duk_put_global_literal
 - duk_put_global_lstring
 - duk_put_global_string
 - duk_put_number_list
 - duk_put_prop
 - duk_put_prop_heapptr
 - duk_put_prop_index
 - duk_put_prop_literal
 - duk_put_prop_lstring
 - duk_put_prop_string
 - duk_seal
- protected
 - duk_pcall
 - duk_pcall_method
 - duk_pcall_prop
 - duk_pcompile
 - duk_pcompile_lstring
 - duk_pcompile_lstring_filename
 - duk_pcompile_string
 - duk_pcompile_string_filename
 - duk_peval
 - duk_peval_lstring
 - duk_peval_lstring_noresult
 - duk_peval_noresult
 - duk_peval_string
 - duk_peval_string_noresult
 - duk_pnew
 - duk_safe_call
 - duk_safe_to_stacktrace
 - duk_safe_to_string
- prototype
 - duk_get_prototype
 - duk_set_prototype
- random
 - duk_random
- sandbox
 - duk_def_prop
 - duk_get_prop_desc
 - duk_push_global_stash
 - duk_push_heap_stash
 - duk_push_thread_stash
 - duk_set_global_object
- slice
 - duk_xcopy_top
 - duk_xmove_top
- stack
 - duk_buffer_to_string
 - duk_check_stack
 - duk_check_stack_top
 - duk_check_type
 - duk_check_type_mask
 - duk_config_buffer
 - duk_copy
 - duk_dump_function
 - duk_dup
 - duk_dup_top
 - duk_get_boolean
 - duk_get_boolean_default
 - duk_get_buffer
 - duk_get_buffer_data
 - duk_get_buffer_data_default
 - duk_get_buffer_default
 - duk_get_c_function
 - duk_get_c_function_default
 - duk_get_context
 - duk_get_context_default
 - duk_get_error_code
 - duk_get_heapptr
 - duk_get_heapptr_default
 - duk_get_int
 - duk_get_int_default
 - duk_get_length
 - duk_get_lstring
 - duk_get_lstring_default
 - duk_get_number
 - duk_get_number_default
 - duk_get_pointer
 - duk_get_pointer_default
 - duk_get_string
 - duk_get_string_default
 - duk_get_top
 - duk_get_top_index
 - duk_get_type
 - duk_get_type_mask
 - duk_get_uint
 - duk_get_uint_default
 - duk_insert
 - duk_inspect_callstack_entry
 - duk_inspect_value
 - duk_is_array
 - duk_is_boolean
 - duk_is_bound_function
 - duk_is_buffer
 - duk_is_buffer_data
 - duk_is_c_function
 - duk_is_callable
 - duk_is_constructable
 - duk_is_constructor_call
 - duk_is_dynamic_buffer
 - duk_is_ecmascript_function
 - duk_is_error
 - duk_is_eval_error
 - duk_is_fixed_buffer
 - duk_is_function
 - duk_is_lightfunc
 - duk_is_nan
 - duk_is_null
 - duk_is_null_or_undefined
 - duk_is_number
 - duk_is_object
 - duk_is_object_coercible
 - duk_is_pointer
 - duk_is_primitive
 - duk_is_range_error
 - duk_is_reference_error
 - duk_is_string
 - duk_is_symbol
 - duk_is_syntax_error
 - duk_is_thread
 - duk_is_type_error
 - duk_is_undefined
 - duk_is_uri_error
 - duk_is_valid_index
 - duk_load_function
 - duk_normalize_index
 - duk_opt_boolean
 - duk_opt_buffer
 - duk_opt_buffer_data
 - duk_opt_c_function
 - duk_opt_context
 - duk_opt_heapptr
 - duk_opt_int
 - duk_opt_lstring
 - duk_opt_number
 - duk_opt_pointer
 - duk_opt_string
 - duk_opt_uint
 - duk_pop
 - duk_pop_2
 - duk_pop_3
 - duk_pop_n
 - duk_pull
 - duk_push_array
 - duk_push_bare_array
 - duk_push_bare_object
 - duk_push_boolean
 - duk_push_buffer
 - duk_push_buffer_object
 - duk_push_c_function
 - duk_push_c_lightfunc
 - duk_push_context_dump
 - duk_push_current_function
 - duk_push_current_thread
 - duk_push_dynamic_buffer
 - duk_push_error_object
 - duk_push_error_object_va
 - duk_push_external_buffer
 - duk_push_false
 - duk_push_fixed_buffer
 - duk_push_global_object
 - duk_push_global_stash
 - duk_push_heap_stash
 - duk_push_heapptr
 - duk_push_int
 - duk_push_literal
 - duk_push_lstring
 - duk_push_nan
 - duk_push_new_target
 - duk_push_null
 - duk_push_number
 - duk_push_object
 - duk_push_pointer
 - duk_push_proxy
 - duk_push_sprintf
 - duk_push_string
 - duk_push_this
 - duk_push_thread
 - duk_push_thread_new_globalenv
 - duk_push_thread_stash
 - duk_push_true
 - duk_push_uint
 - duk_push_undefined
 - duk_push_vsprintf
 - duk_remove
 - duk_replace
 - duk_require_boolean
 - duk_require_buffer
 - duk_require_buffer_data
 - duk_require_c_function
 - duk_require_callable
 - duk_require_constructable
 - duk_require_constructor_call
 - duk_require_context
 - duk_require_function
 - duk_require_heapptr
 - duk_require_int
 - duk_require_lstring
 - duk_require_normalize_index
 - duk_require_null
 - duk_require_number
 - duk_require_object
 - duk_require_object_coercible
 - duk_require_pointer
 - duk_require_stack
 - duk_require_stack_top
 - duk_require_string
 - duk_require_top_index
 - duk_require_type_mask
 - duk_require_uint
 - duk_require_undefined
 - duk_require_valid_index
 - duk_resize_buffer
 - duk_safe_to_lstring
 - duk_safe_to_stacktrace
 - duk_safe_to_string
 - duk_set_global_object
 - duk_set_length
 - duk_set_top
 - duk_steal_buffer
 - duk_swap
 - duk_swap_top
 - duk_to_boolean
 - duk_to_buffer
 - duk_to_dynamic_buffer
 - duk_to_fixed_buffer
 - duk_to_int
 - duk_to_int32
 - duk_to_lstring
 - duk_to_null
 - duk_to_number
 - duk_to_object
 - duk_to_pointer
 - duk_to_primitive
 - duk_to_stacktrace
 - duk_to_string
 - duk_to_uint
 - duk_to_uint16
 - duk_to_uint32
 - duk_to_undefined
 - duk_trim
 - duk_xcopy_top
 - duk_xmove_top
- stash
 - duk_push_global_stash
 - duk_push_heap_stash
 - duk_push_thread_stash
- string
 - duk_buffer_to_string
 - duk_char_code_at
 - duk_compile_lstring
 - duk_compile_lstring_filename
 - duk_compile_string
 - duk_compile_string_filename
 - duk_concat
 - duk_decode_string
 - duk_del_prop_lstring
 - duk_del_prop_string
 - duk_eval_lstring
 - duk_eval_lstring_noresult
 - duk_eval_string
 - duk_eval_string_noresult
 - duk_get_global_lstring
 - duk_get_global_string
 - duk_get_lstring
 - duk_get_lstring_default
 - duk_get_prop_lstring
 - duk_get_prop_string
 - duk_get_string
 - duk_get_string_default
 - duk_has_prop_lstring
 - duk_has_prop_string
 - duk_is_string
 - duk_is_symbol
 - duk_join
 - duk_map_string
 - duk_opt_lstring
 - duk_opt_string
 - duk_pcompile_lstring
 - duk_pcompile_lstring_filename
 - duk_pcompile_string
 - duk_pcompile_string_filename
 - duk_peval_lstring
 - duk_peval_lstring_noresult
 - duk_peval_string
 - duk_peval_string_noresult
 - duk_push_lstring
 - duk_push_sprintf
 - duk_push_string
 - duk_push_vsprintf
 - duk_put_global_lstring
 - duk_put_global_string
 - duk_put_prop_lstring
 - duk_put_prop_string
 - duk_require_lstring
 - duk_require_string
 - duk_safe_to_lstring
 - duk_safe_to_stacktrace
 - duk_safe_to_string
 - duk_substring
 - duk_to_lstring
 - duk_to_stacktrace
 - duk_to_string
 - duk_trim
- symbol
 - duk_is_symbol
- thread
 - duk_is_thread
 - duk_push_current_thread
 - duk_push_thread
 - duk_push_thread_new_globalenv
 - duk_push_thread_stash
 - duk_resume
 - duk_set_global_object
 - duk_suspend
- time
 - duk_components_to_time
 - duk_get_now
 - duk_time_to_components
- vararg
 - duk_error_va
 - duk_eval_error_va
 - duk_generic_error_va
 - duk_push_error_object_va
 - duk_push_vsprintf
 - duk_range_error_va
 - duk_reference_error_va
 - duk_syntax_error_va
 - duk_type_error_va
 - duk_uri_error_va




## duk_alloc()

### プロトタイプ

```c
void *duk_alloc(duk_context *ctx, duk_size_t size);
```

### スタック

(バリュースタックに影響なし)

### 概要

duk_alloc_raw() と同様ですが、要求を満たすためにガベージコレクションを起動させることがあります。しかし、割り当てられたメモリ自体は自動的にガベージコレクションされません。ガベージコレクションの後でも、割り当て要求は失敗することがあり、その場合は NULL が返される。割り当てられたメモリは自動的にゼロにされないので、任意のゴミを含む可能性があります。

duk_alloc() で割り当てられたメモリは、duk_free() または duk_free_raw() で解放することができる。

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

### プロトタイプ

```c
void *duk_alloc_raw(duk_context *ctx, duk_size_t size);
```

### スタック

(値スタックに影響なし)

### 要約

コンテキストに登録された生の割り当て関数を使用して、サイズバイトを割り当てる。割り当てに失敗した場合、NULL を返す。size が 0 の場合、この呼び出しは NULL を返すか、あるいは duk_free_raw() などに安全に渡すことができる非 NULL 値を返すかもしれない。また、割り当てられたメモリは自動的にガベージコレクションされません。割り当てられたメモリは自動的にゼロにされず、任意のゴミを含む可能性があります。

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


## duk_base64_decode() 1.0.0codecbase64§

### プロトタイプ

```c
void duk_base64_decode(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . base64_val . . . → . . . val . . .

### 要約

Decodes a base-64 encoded value into a buffer as an in-place operation. If the input is invalid, throws an error.

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


## duk_base64_encode() 1.0.0codecbase64§

### プロトタイプ

```c
const char *duk_base64_encode(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . base64_val . . .

### 要約

Coerces an arbitrary value into a buffer and then encodes the result into base-64 as an in-place operation. Returns a pointer to the resulting string for convenience.

Coercing to a buffer first coerces a non-buffer value into a string, and then coerces the string into a buffer. The resulting buffer contains the string in CESU-8 encoding.

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


## duk_buffer_to_string() 2.0.0stringstackbuffer§

### プロトタイプ

```c
const char *duk_buffer_to_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . buffer_to_string(val) . . .

### 要約

Replace the buffer value (plain buffer or any buffer object) at idx with a string whose internal byte representation is taken 1:1 from the buffer (same as if the buffer data was pushed using duk_push_lstring()). Returns a non-NULL pointer to the read-only, NUL-terminated string data. A TypeError is thrown if (1) the argument is not a buffer value, (2) the argument is a buffer object whose backing buffer doesn't cover the apparent buffer size, (3) the index is invalid.

Because the buffer data is used as is for the string created, this API call allows creation of Symbol values unless the buffer data is first sanitized.

Custom type coercion is described in Type conversion and testing.

Since Duktape 2.0 the duk_to_string() coercion for a buffer value is typically something like "[object Uint8Array]", which is not usually intended.

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



## duk_call() 1.0.0call§

### プロトタイプ

```c
void duk_call(duk_context *ctx, duk_idx_t nargs);
```

### スタック

. . . func arg1 . . . argN → . . . retval

### 要約

Call target function func with nargs arguments (not counting the function itself). The function and its arguments are replaced by a single return value. An error thrown during the function call is not automatically caught.

The target function this binding is initially set to undefined. If the target function is not strict, the binding is replaced by the global object before the function is invoked; see Entering Function Code. If you want to control the this binding, you can use duk_call_method() or duk_call_prop() instead.

This API call is equivalent to:

var retval = func(arg1, ..., argN);
or:

var retval = func.call(undefined, arg1, ..., argN);

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


## duk_call_method() 1.0.0call§

### プロトタイプ

```c
void duk_call_method(duk_context *ctx, duk_idx_t nargs);
```

### スタック

. . . func this arg . . . argN → . . . retval

### 要約

Call target function func with an explicit this binding and nargs arguments (not counting the function or the this binding value). The function object, the this binding value, and the function arguments are replaced by a single return value. An error thrown during the function call is not automatically caught.

If the target function is not strict, the binding value seen by the target function may be modified by processing specified in Entering Function Code.

This API call is equivalent to:

var retval = func.call(this_binding, arg1, ..., argN);

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



## duk_call_prop() 1.0.0propertycall§

### プロトタイプ

```c
void duk_call_prop(duk_context *ctx, duk_idx_t obj_idx, duk_idx_t nargs);
```

### スタック

. . . obj . . . key arg1 . . . argN → . . . obj . . . retval

### 要約

Call obj.key with nargs arguments, with this binding set to obj. The property name and the function arguments are replaced by a single return value; the target object is not touched. An error during the function call is not automatically caught.

If the target function is not strict, the binding value seen by the target function may be modified by processing specified in Entering Function Code.

This API call is equivalent to:

var retval = obj[key](arg1, ..., argN);
or:

var func = obj[key];
var retval = func.call(obj, arg1, ..., argN);
Although the base value for property accesses is usually an object, it can technically be an arbitrary value. Plain string and buffer values have virtual index properties so you can access "foo"[2], for instance. Most primitive values also inherit from some prototype object so that you can e.g. call methods on them: (12345).toString(16).

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



## duk_cbor_decode() 2.5.0experimentalcodeccbor§

### プロトタイプ

```c
void duk_cbor_decode(duk_context *ctx, duk_idx_t idx, duk_uint_t decode_flags);
```

### スタック

. . . cbor_val . . . → . . . val . . .

### 要約

Decodes a CBOR encoded value (given in any buffer type) as an in-place operation. The resulting value can be any ECMAScript value. No flags are defined at present, so pass in a 0 for flags.

Mapping from CBOR to ECMAScript values is experimental and the decoding result may change over time. For example, custom CBOR tags will be added to improve encode-decode roundtripping of ECMAScript values.

### 例

```c
duk_cbor_decode(ctx, -1, 0);
```

### 参照

duk_cbor_encode


## duk_cbor_encode() 2.5.0experimentalcodeccbor§

### プロトタイプ

```c
void duk_cbor_encode(duk_context *ctx, duk_idx_t idx, duk_uint_t encode_flags);
```

### スタック

. . . val . . . → . . . cbor_val . . .

### 要約

Encodes an arbitrary value into its CBOR representation as an in-place operation. The resulting value is an ArrayBuffer (for now). No flags are defined at present, so pass in a 0 for flags.

Mapping from ECMAScript values to CBOR is experimental and the encoding result may change over time. For example, custom CBOR tags will be added to improve encode-decode roundtripping of ECMAScript values. Also the result type (ArrayBuffer) may change later.

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


## duk_char_code_at() 1.0.0string§

### プロトタイプ

```c
duk_codepoint_t duk_char_code_at(duk_context *ctx, duk_idx_t idx, duk_size_t char_offset);
```

### スタック

. . . str . . .

### 要約

Get the codepoint of a character at character offset char_offset of a string at idx. If the value at idx is not a string, an error is thrown. If the string cannot be (extended) UTF-8 decoded, the result is U+FFFD (Unicode replacement character). If char_offset is invalid (outside the string) a zero is returned.


### 例

```c
printf("char code at char index 12: %ld\n", (long) duk_char_code_at(ctx, -3, 12));
```



## duk_check_stack() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_check_stack(duk_context *ctx, duk_idx_t extra);
```

### スタック

(No effect on value stack.)


### 要約

Ensure that the value stack has at least extra reserved (allocated) elements for caller's use, relative to the current stack top. Returns 1 if successful, 0 otherwise. If the call is successful, the caller is guaranteed that extra elements can be pushed to the value stack without a value stack related error (other errors like out-of-memory can still occur). The caller MUST NOT rely on being able to push more than extra values; although this is possible, such elements are reserved for Duktape's internal use.

Upon entry to a Duktape/C function and when outside any call there is an automatic reserve (of DUK_API_ENTRY_STACK elements) allocated for the caller in addition to call arguments on the value stack. If more value stack space is needed, the caller must reserve more space explicitly either in the beginning of the function (e.g. if the number of elements required is known or can be computed based on arguments) or dynamically (e.g. inside a loop). Note that an attempt to push a value beyond the currently allocated value stack causes an error: it does not cause the value stack to be automatically extended. This simplifies the internal implementation.

In addition to user reserved elements, Duktape keeps an automatic internal value stack reserve to ensure all API calls have enough value stack space to work without further allocations. The value stack is also extended in somewhat large steps to minimize memory reallocation activity. As a result the internal number of value stack elements available beyond the caller specified extra varies considerably. The caller does not need to take this into account and should never rely on any additional elements being available.

As a general rule duk_require_stack() should be used instead of this function to reserve more stack space. If value stack cannot be extended, there is almost never a useful recovery strategy except to throw an error and unwind.


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


## duk_check_stack_top() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_check_stack_top(duk_context *ctx, duk_idx_t top);
```

### スタック

(No effect on value stack.)


### 要約

Like duk_check_stack(), but ensures there is space up to top for caller's use. This is sometimes more convenient than specifying the number of additional elements (relative to current stack top) to reserve.

The caller specifies the maximum stack top ensured for caller's use, not the highest value stack index reserved for the caller. For instance, if top is 500, the highest value stack index reserved for the caller is 499.
As a general rule duk_require_stack_top() should be used instead of this function to reserve more stack space. If value stack cannot be extended, there is almost never a useful recovery strategy except to throw an error and unwind.


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


## duk_check_type() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_check_type(duk_context *ctx, duk_idx_t idx, duk_int_t type);
```

### スタック

. . . val . . .

### 要約

Matches the type of the value at idx against type. Returns 1 if the type matches, zero otherwise.


### 例

```c
if (duk_check_type(ctx, -3, DUK_TYPE_NUMBER)) {
    printf("value is a number\n");
}
```


## duk_check_type_mask() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_check_type_mask(duk_context *ctx, duk_idx_t idx, duk_uint_t mask);
```

### スタック

. . . val . . .

### 要約

Matches the type of the value at idx against the type mask bits given in mask. Returns 1 if the value type matches one of the type mask bits and zero otherwise.


### 例

```c
if (duk_check_type_mask(ctx, -3, DUK_TYPE_MASK_STRING |
                                 DUK_TYPE_MASK_NUMBER)) {
    printf("value is a string or a number\n");
}
```


## duk_compact() 1.0.0propertyobjectmemory§

### プロトタイプ

```c
void duk_compact(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

(No effect on value stack.)


### 要約

Resizes an object's internal memory allocation to minimize memory usage. If the value at obj_idx is not an object, does nothing. Although compaction is normally safe, it can fail due to an internal error (such as out-of-memory error). Compaction only affects to an object's "own properties", not its inherited properties.

Compaction is not a final operation and has no impact on object semantics. Adding new properties to the object is still possible (assuming the object is extensible), but causes an object resize. Existing property values can be changed (assuming the properties are writable) without affecting the object's internal memory allocation. An object can be compacted multiple times: for instance, if a new property is added to a previously compacted object, the object can be compacted again to minimize memory footprint after the property addition.

Compacting an object saves a small amount of memory per object. It is generally useful when (1) memory footprint is very important, (2) an object is unlikely to gain new properties, (3) an object is relatively long-lived, and (4) when compaction can be applied to enough many objects to make a significant difference.

If Object.seal(), Object.freeze(), or Object.preventExtensions() is called, an object is automatically compacted.


### 例

```c
duk_push_object(ctx);                           /* [ ... obj ] */
duk_push_int(ctx, 42);                          /* [ ... obj 42 ] */
duk_put_prop_string(ctx, -2, "meaningOfLife");  /* [ ... obj ] */
duk_compact(ctx, -1);                           /* [ ... obj ] */
```



## duk_compile() 1.0.0compile§

### プロトタイプ

```c
void duk_compile(duk_context *ctx, duk_uint_t flags);
```

### スタック

. . . source filename → . . . function

### 要約

Compile ECMAScript source code and replace it with a compiled function object (the code is not executed). The filename argument is stored as the fileName property of the resulting function, and is the name used in e.g. tracebacks to identify the function. May throw a SyntaxError for any compile-time errors (in addition to the usual internal errors like out-of-memory, internal limit errors, etc).

The following flags may be given:

DUK_COMPILE_EVAL	Compile the input as eval code instead of as an ECMAScript program
DUK_COMPILE_FUNCTION	Compile the input as a function instead of as an ECMAScript program
DUK_COMPILE_STRICT	Force the input to be compiled in strict mode
DUK_COMPILE_SHEBANG	Allow non-standard shebang comment (#! ...) on first line of the input
The source code being compiled may be:

Global code: compiles into a function with zero arguments, which executes like a top level ECMAScript program (default)
Eval code: compiles into a function with zero arguments, which executes like an ECMAScript eval call (flag DUK_COMPILE_EVAL)
Function code: compiles into a function with zero or more arguments (flag DUK_COMPILE_FUNCTION)
All of these have slightly different semantics in ECMAScript. See Establishing an Execution Context for a detailed discussion. One major difference is that global and eval contexts have an implicit return value: the last non-empty statement value is an automatic return value for the program or eval code, whereas functions don't have an automatic return value.

Global and eval code don't have an explicit function syntax. For instance, the following can be compiled both as a global and as an eval expression:

print("Hello world!");
123;  // implicit return value
Function code follows the ECMAScript function syntax (the function name is optional):

function adder(x,y) {
    return x+y;
}
Compiling a function is equivalent to compiling eval code which contains a function expression. Note that the outermost parentheses are required, otherwise the eval code will register a global function named "adder" instead of returning a plain function value:

(function adder(x,y) {
    return x+y;
})
The bytecode generated for global and eval code is currently slower than that generated for functions: a "slow path" is used for all variable accesses in program and eval code, and the implicit return value handling of program and eval code generates some unnecessary bytecode. From a performance point of view (both memory and execution performance) it is thus preferable to have as much code inside functions as possible.

When compiling eval and global expressions, be careful to avoid the usual ECMAScript gotchas, such as:

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


## duk_compile_lstring() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_compile_lstring(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

. . . → . . . function

### 要約

Like duk_compile(), but the compile input is given as a C string with explicit length. The filename associated with the function is "input".

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_compile_lstring(ctx, 0, src, len);
```


## duk_compile_lstring_filename() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_compile_lstring_filename(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

. . . filename → . . . function

### 要約

Like duk_compile(), but the compile input is given as a C string with explicit length.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_push_string(ctx, "myFile.js");
duk_compile_lstring_filename(ctx, 0, src, len);
```


## duk_compile_string() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_compile_string(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

. . . → . . . function

### 要約

Like duk_compile(), but the compile input is given as a C string. The filename associated with the function is "input".

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
duk_compile_string(ctx, 0, "print('program code');");
```


## duk_compile_string_filename() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_compile_string_filename(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

. . . filename → . . . function

### 要約

Like duk_compile(), but the compile input is given as a C string.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
duk_push_string(ctx, "myFile.js");
duk_compile_string_filename(ctx, 0, "print('program code');");
```


## duk_components_to_time() 2.0.0time§

### プロトタイプ

```c
duk_double_t duk_components_to_time(duk_context *ctx, duk_time_components *comp);
```

### スタック

(No effect on value stack.)


### 要約

Convert components (year, month, day, etc), interpreted in UTC, into a time value. The comp->weekday argument is ignored in the conversion. If the component values are invalid, an error is thrown.

There are some differences to the ECMAScript Date.UTC() built-in:

There's no special handling of two-digit years. For example, Date.UTC(99, 0, 1) gets interpreted as 1999-01-01. If comp->time is 99, it's interpreted as the year 99.
The milliseconds component is allowed fractions (sub-millisecond resolution) so that the resulting time value may have fractions.
Like the ECMAScript primitives, the components can exceed their natural range and are normalized. For example, specifying comp->minutes as 120 is interpreted as adding 2 hours to the time value. The components are expressed as IEEE doubles to allow large and negative values to be used.


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


## duk_concat() 1.0.0string§

### プロトタイプ

```c
void duk_concat(duk_context *ctx, duk_idx_t count);
```

### スタック

. . . val1 . . . valN → . . . result

### 要約

Concatenate zero or more values into a result string. The input values are automatically coerced with ToString().

This primitive minimizes the number of intermediate string interning operations and is better than concatenating strings manually.

Unlike Array.prototype.concat(), this API call does not flatten array arguments or support Symbol.isConcatSpreadable.

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


## duk_config_buffer() 1.3.0stackbuffer§

### プロトタイプ

```c
void duk_config_buffer(duk_context *ctx, duk_idx_t idx, void *ptr, duk_size_t len);
```

### スタック

. . . buf . . . → . . . buf . . .

### 要約

Set the external buffer pointer and length of an external buffer value.


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


## duk_copy() 1.0.0stack§

### プロトタイプ

```c
void duk_copy(duk_context *ctx, duk_idx_t from_idx, duk_idx_t to_idx);
```

### スタック

. . . old(to_idx) . . . val(from_idx) . . . → . . . val(to_idx) . . . val(from_idx) . . .

### 要約

Copy a value from from_idx to to_idx, overwriting the previous value. If either index is invalid, throws an error.

This is a shorthand for:

/* to_index must be normalized in case it is negative and would change its
 * meaning after duk_dup().
 */
to_idx = duk_normalize_idx(ctx, to_idx);
duk_dup(ctx, from_idx);
duk_replace(ctx, to_idx);

### 例

```c
duk_copy(ctx, -3, 1);
```

### 参照

duk_insert
duk_replace


## duk_create_heap() 1.0.0heap§

### プロトタイプ

```c
duk_context *duk_create_heap(duk_alloc_function alloc_func,
                             duk_realloc_function realloc_func,
                             duk_free_function free_func,
                             void *heap_udata,
                             duk_fatal_function fatal_handler);
```

### スタック

(No effect on value stack.)


### 要約

Create a new Duktape heap and return an initial context (thread). If heap initialization fails, a NULL is returned. There is currently no way to obtain more detailed error information.

The caller may provide custom memory management functions in alloc_func, realloc_func, and free_func; the pointers must either all be NULL or all be non-NULL. If the pointers are NULL, default memory management functions (ANSI C malloc(), realloc() , and free()) are used. The memory management functions share the same opaque userdata pointer, heap_udata. This userdata pointer is also used for other Duktape features like fatal error handling and low memory pointer compression macros.

A fatal error handler is provided in fatal_handler. This handler is called in unrecoverable error situations such as uncaught errors, out-of-memory errors not resolved by garbage collection, self test errors, etc. A caller SHOULD implement a fatal error handler in most applications. If not given, a default fatal error handler built into Duktape is used instead. The default fatal error handler (unless overridden by duk_config.h) calls abort() with no error message printed to stdout or stderr. See How to handle fatal errors for more detail and examples.

To create a Duktape heap with default settings, use duk_create_heap_default().

New contexts linked to the same heap can be created with duk_push_thread() and duk_push_thread_new_globalenv().


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


## duk_create_heap_default() 1.0.0heap§

### プロトタイプ

```c
duk_context *duk_create_heap_default(void);
```

### スタック

(No effect on value stack.)


### 要約

Create a new Duktape heap and return an initial context (thread). If heap initialization fails, a NULL is returned. There is currently no way to obtain more detailed error information.

The created heap will use default memory management and fatal error handler functions. This API call is equivalent to:

ctx = duk_create_heap(NULL, NULL, NULL, NULL, NULL);

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


## duk_debugger_attach() 1.2.0debugger§

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

(No effect on value stack.)


### 要約

Attach a debugger to the Duktape heap. The debugger will automatically enter paused state when the attach is complete. If debugger support is not compiled into Duktape, throws an error. See debugger.rst for more information on debugger integration. The callbacks are as follows, optional callbacks may be NULL:

Callback	Required	Description
read_cb	Yes	Debug transport read callback
write_cb	Yes	Debug transport write callback
peek_cb	No	Debug transport peek callback, strongly recommended but optional
read_flush_cb	No	Debug transport read flush callback
write_flush_cb	No	Debug transport write flush callback
request_cb	No	Application specific command (AppRequest) callback, NULL indicates no AppRequest support
detached_cb	No	Debugger detached callback
Important: The callbacks are not allowed to call any Duktape API functions (doing so may cause memory unsafe behavior) except for the following:

The request_cb AppRequest callback may use the value stack within the guidelines described in debugger.rst.
Since Duktape 1.4.0 the debugger detached callback is allowed to call duk_debugger_attach() to immediately reattach the debugger. Before Duktape 1.4.0 the immediate reattach potentially caused some issues.
The duk_debugger_attach_custom() and duk_debugger_attach() API calls were merged in Duktape 2.x.

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


## duk_debugger_cooperate() 1.2.0debugger§

### プロトタイプ

```c
void duk_debugger_cooperate(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Check for and execute inbound debug commands without blocking. Debug commands are executed in the context of the ctx argument. May only be called when no calls into Duktape are in progress (from any context). See debugger.rst for more information.


### 例

```c
duk_debugger_cooperate(ctx);
```

### 参照

duk_debugger_attach
duk_debugger_detach


## duk_debugger_detach() 1.2.0debugger§

### プロトタイプ

```c
void duk_debugger_detach(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Detach a debugger from the Duktape heap. If debugger support is not compiled into Duktape, throws an error. See debugger.rst for more information.


### 例

```c
duk_debugger_detach(ctx);
```

### 参照

duk_debugger_attach
duk_debugger_cooperate


## duk_debugger_notify() 1.5.0debugger§

### プロトタイプ

```c
duk_bool_t duk_debugger_notify(duk_context *ctx, duk_idx_t nvalues);
```

### スタック

. . . val1 . . . valN → . . .

### 要約

Send an application specific debugger notify (AppNotify) containing the nvalues values on the value stack top mapped to debug protocol dvalues. The return value indicates whether the notify was successfully sent (non-zero) or not (zero). The nvalues arguments are always popped off the stack top. The call is a no-op if debugger support has not been compiled in, or if the debugger is not attached; in both cases the call will return zero to indicate that the notify was not sent.

See the debugger documentation for more information and examples on how to use application specific notifications.


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


## duk_debugger_pause() 1.5.0debugger§

### プロトタイプ

```c
void duk_debugger_pause(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Request a debugger pause as soon as possible and return without blocking. The pause will be triggered the next time ECMAScript bytecode is executed, which is usually almost immediate. However, if a native call such as a Duktape/C function is in progress or no ECMAScript code is currently executing, it may take longer for the pause to take effect.

The call is a no-op if (1) debugger support has not been compiled in, (2) the debugger is not attached, or (3) Duktape is already paused. This mimics the semantics of the ECMAScript debugger statement.

Like all Duktape API calls, this call is not thread safe. It may be tempting to trigger a pause from a thread different than one running ECMAScript code for the context, but this is unsafe and not currently supported.

### 例

```c
/* In your event loop: */
if (key_pressed == KEY_F12) {
    duk_debugger_pause(ctx);
}
```


## duk_decode_string() 1.0.0string§

### プロトタイプ

```c
void duk_decode_string(duk_context *ctx, duk_idx_t idx, duk_decode_char_function callback, void *udata);
```

### スタック

. . . val . . .

### 要約

Process string at idx, calling callback for each codepoint of the string. The callback is given the udata argument and a codepoint. The input string is not modified. If the value is not a string or the index is invalid, throws an error.


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


## duk_def_prop() 1.1.0sandboxproperty§

### プロトタイプ

```c
void duk_def_prop(duk_context *ctx, duk_idx_t obj_idx, duk_uint_t flags);
```

### スタック

. . . obj . . . key → . . . obj . . . (if have no value, setter, getter)
. . . obj . . . key value → . . . obj . . . (if have value)
. . . obj . . . key getter → . . . obj . . . (if have getter, but no setter)
. . . obj . . . key setter → . . . obj . . . (if have setter, but no getter)
. . . obj . . . key getter setter → . . . obj . . . (if have both getter and setter)

### 要約

Create or alter the attributes of property key of object at obj_idx, with semantics like Object.defineProperty(). When the requested change is not allowed (e.g. property is not configurable), a TypeError is thrown. If the target is not an object (or the index is invalid) an error is thrown.

The flags field provides "have" flags to indicate what property attributes are changed, which models the partial property descriptors allowed by Object.defineProperty(). Values for the writable, configurable, and enumerable attributes are given in the flags field, while property value, getter, and setter are given as value stack arguments. When creating a new property, missing attribute values cause ECMAScript defaults to be used (false for all boolean attributes, undefined for value, getter, setter); when modifying an existing property missing attribute values mean that existing attribute values are not touched.

As a concrete example:

// Set value, set writable, clear enumerable, leave configurable untouched.
Object.defineProperty(obj, { value: 123, writable: true, enumerable: false });
Would map to:

duk_push_uint(ctx, 123);  /* value is taken from stack */
duk_def_prop(ctx, obj_idx,
             DUK_DEFPROP_HAVE_VALUE |  /* <=> value: 123 */
             DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_WRITABLE |  /* <=> writable: true */
             DUK_DEFPROP_HAVE_ENUMERABLE);  /* <=> enumerable: false */
With the DUK_DEFPROP_FORCE flag property changes can be forced even when an operation would normally fail due to a non-extensible target object or a non-configurable property. This cannot be done from ECMAScript code with Object.defineProperty() and is useful for e.g. sandboxing setup. In some cases even a forced change is not possible and will cause an error to be thrown. For instance, properties implemented internally as virtual properties may not be modifiable (such as string .length and index properties) or may have limitations (such as array .length property which cannot be made configurable or enumerable due to internal limitations).

The following base flags are defined:

Define	Description
DUK_DEFPROP_WRITABLE	Set writable attribute, effective only if DUK_DEFPROP_HAVE_WRITABLE is set.
DUK_DEFPROP_ENUMERABLE	Set enumerable attribute, effective only if DUK_DEFPROP_HAVE_ENUMERABLE is set.
DUK_DEFPROP_CONFIGURABLE	Set configurable attribute, effective only if DUK_DEFPROP_HAVE_CONFIGURABLE is set.
DUK_DEFPROP_HAVE_WRITABLE	Set or clear writable attribute (no change if unset).
DUK_DEFPROP_HAVE_ENUMERABLE	Set or clear enumerable attribute (no change if unset).
DUK_DEFPROP_HAVE_CONFIGURABLE	Set or clear configurable attribute (no change if unset).
DUK_DEFPROP_HAVE_VALUE	Set value attribute, value is given on value stack.
DUK_DEFPROP_HAVE_GETTER	Set getter attribute, value is given on value stack.
DUK_DEFPROP_HAVE_SETTER	Set setter attribute, value is given on value stack.
DUK_DEFPROP_FORCE	Force attribute change if possible, even if property is non-configurable.
There are a also convenience defines mapping to common flag combinations. For example, DUK_DEFPROP_SET_WRITABLE is equivalent to DUK_DEFPROP_HAVE_WRITABLE and DUK_DEFPROP_WRITABLE. Currently the following are defined:

Define	Description
DUK_DEFPROP_{SET,CLEAR}_WRITABLE	Set or clear the 'writable' attribute.
DUK_DEFPROP_{SET,CLEAR}_ENUMERABLE	Set or clear the 'enumerable' attribute.
DUK_DEFPROP_{SET,CLEAR}_CONFIGURABLE	Set or clear the 'configurable' attribute.
DUK_DEFPROP_SET_{W,E,C,WE,WC,EC,WEC}	Set one or more attributes, don't touch other attributes.
DUK_DEFPROP_CLEAR_{W,E,C,WE,WC,EC,WEC}	Clear one or more attributes, don't touch other attributes.
DUK_DEFPROP_ATTR_{W,E,C,WE,WC,EC,WEC}	Set or clear writable, enumerable, and configurable attributes. For example, DUK_DEFPROP_ATTR_WC sets writable and configurable, and clears enumerable.
DUK_DEFPROP_HAVE_{W,E,C,WE,WC,EC,WEC}	Indicate one or more attributes are set/cleared, e.g. DUK_DEFPROP_HAVE_WC is equivalent to DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_HAVE_CONFIGURABLE.
DUK_DEFPROP_{W,E,C,WE,WC,EC,WEC}	Provide attribute values (effective if matching "have" flag is set), e.g. DUK_DEFPROP_WE is equivalent to DUK_DEFPROP_WRITABLE | DUK_DEFPROP_ENUMERABLE.
Some examples (see more examples below):

To set value, set writable, clear enumerable, and set configurable, push the value onto the value stack and set flags to:
Base flags: DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_WRITABLE | DUK_DEFPROP_HAVE_ENUMERABLE | DUK_DEFPROP_HAVE_CONFIGURABLE | DUK_DEFPROP_CONFIGURABLE; or
Convenience: DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_ATTR_WC.
To clear the writable attribute (leaving other attributes untouched), set flags to:
Base flags: DUK_DEFPROP_HAVE_WRITABLE; or
Convenience: DUK_DEFPROP_CLEAR_WRITABLE; or
Convenience: DUK_DEFPROP_CLEAR_W.
To set value, clear writable, and set enumerable (leaving other attributes untouched), push the value onto the value stack and set flags to:
Base flags: DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_HAVE_WRITABLE | DUK_DEFPROP_HAVE_ENUMERABLE | DUK_DEFPROP_ENUMERABLE; or
Convenience: DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_CLEAR_W | DUK_DEFPROP_SET_E; or
Convenience: DUK_DEFPROP_HAVE_VALUE | DUK_DEFPROP_HAVE_WE | DUK_DEFPROP_E.
This API call is useful for various things:

To create properties with non-default attributes directly from C code.
To create accessor (getter/setter) properties directly from C code.
To modify the property attributes of existing properties directly from C code.
See API testcase test-def-prop.c for more examples.

If the target is a Proxy object which implements the defineProperty trap, the trap should be invoked. However, Duktape doesn't currently support the defineProperty trap, and the defineProperty() operation isn't currently forwarded to the target. When support is added, this API call will start invoking the trap or forwarding the operation to the target object.

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


## duk_del_prop() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_del_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

. . . obj . . . key → . . . obj . . .

### 要約

Delete the property key of a value at obj_idx. key is removed from the stack. Return code and error throwing behavior:

If property exists and is configurable (deletable), deletes the property and returns 1.
If property exists but is not configurable, throws an error (strict mode semantics).
If property does not exist, returns 1 (not 0).
If the value at obj_idx is not object coercible, throws an error.
If obj_idx is invalid, throws an error.
The property deletion is equivalent to the ECMAScript expression res = delete obj[key]. For precise semantics, see Property Accessors, The delete operator and [[Delete]] (P, Throw). The return value and error throwing behavior mirrors the ECMAScript delete operator behavior. Both the target value and the key are coerced:

The target value is automatically coerced to an object. However, this object is a temporary one, so deleting its properties is not very useful.
The key argument is internally coerced using ToPropertyKey() coercion which results in a string or a Symbol. There is an internal fast path for arrays and numeric indices which avoids an explicit string coercion, so use a numeric key when applicable.
If the target is a Proxy object which implements the deleteProperty trap, the trap is invoked and the API call return value matches the trap return value.

This API call returns 1 when the target property does not exist. This is not very intuitive, but follows ECMAScript semantics: delete obj.nonexistent also evaluates to true.
If the key is a fixed string you can avoid one API call and use the duk_del_prop_string() variant. Similarly, if the key is an array index, you can use the duk_del_prop_index() variant.

Although the base value for property accesses is usually an object, it can technically be an arbitrary value. Plain string and buffer values have virtual index properties so you can access "foo"[2], for instance. Most primitive values also inherit from some prototype object so that you can e.g. call methods on them: (12345).toString(16).

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


## duk_del_prop_heapptr() 2.2.0propertyheapptrborrowed§

### プロトタイプ

```c
duk_bool_t duk_del_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_del_prop(), but the property name is given as a Duktape heap pointer obtained e.g. using duk_get_heapptr(). If ptr is NULL, undefined is used as the key.


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


## duk_del_prop_index() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_del_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_del_prop(), but the property name is given as an unsigned integer arr_idx. This is especially useful for deleting array elements (but is not limited to that).

Conceptually the number is coerced to a string for the property deletion, e.g. 123 would be equivalent to a property name "123". Duktape avoids an explicit coercion whenever possible.


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


## duk_del_prop_literal() 2.3.0propertyliteral§

### プロトタイプ

```c
duk_bool_t duk_del_prop_literal(duk_context *ctx, duk_idx_t obj_idx, const char *key_literal);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_del_prop(), but the property name is given as a string literal (see duk_push_literal()).


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


## duk_del_prop_lstring() 2.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_del_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_del_prop(), but the property name is given as a string with explicit length.


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


## duk_del_prop_string() 1.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_del_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

. . . obj . . . → . . . obj . . .

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


## duk_destroy_heap() 1.0.0heap§

### プロトタイプ

```c
void duk_destroy_heap(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Destroy a Duktape heap. The argument context can be any context linked to the heap. All resources related to the heap are freed and must not be referenced after the call completes. These resources include all contexts linked to the heap, and also all string and buffer pointers within the heap.

If ctx is NULL, the call is a no-op.


### 例

```c
duk_destroy_heap(ctx);
```


## duk_dump_function() 1.3.0stackbytecode§

### プロトタイプ

```c
void duk_dump_function(duk_context *ctx);
```

### スタック

. . . function → . . . bytecode

### 要約

Dump an ECMAScript function at stack top into bytecode, replacing the function with a buffer containing the bytecode data. The bytecode can be loaded back using duk_load_function().

For more information on Duktape bytecode dump/load, supported features, and known limitations, see bytecode.rst. Duktape bytecode format is not intended for obfuscation, see notes on Obfuscation.

### 例

```c
duk_eval_string(ctx, "(function helloWorld() { print('hello world'); })");
duk_dump_function(ctx);
/* stack top now contains a buffer containing helloWorld bytecode */
```

### 参照

duk_load_function


## duk_dup() 1.0.0stack§

### プロトタイプ

```c
void duk_dup(duk_context *ctx, duk_idx_t from_idx);
```

### スタック

. . . val . . . → . . . val . . . val

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


## duk_dup_top() 1.0.0stack§

### プロトタイプ

```c
void duk_dup_top(duk_context *ctx);
```

### スタック

. . . val → . . . val val

### 要約

Push a duplicate of the value at stack top to the stack. If the value stack is empty, throws an error.

Equivalent to calling duk_dup(ctx, -1).


### 例

```c
duk_push_int(ctx, 123);  /* -> [ ... 123 ] */
duk_push_int(ctx, 234);  /* -> [ ... 123 234 ] */
duk_dup_top(ctx);        /* -> [ ... 123 234 234 ] */
```


## duk_enum() 1.0.0propertyobject§

### プロトタイプ

```c
void duk_enum(duk_context *ctx, duk_idx_t obj_idx, duk_uint_t enum_flags);
```

### スタック

. . . obj . . . → . . . obj . . . enum

### 要約

Create an enumerator for object at obj_idx. Enumeration details can be controlled with enum_flags. If the target value is not an object, throws an error.

Enumeration flags:

DUK_ENUM_INCLUDE_NONENUMERABLE	Enumerate also non-enumerable properties, by default only enumerable properties are enumerated.
DUK_ENUM_INCLUDE_HIDDEN	Enumerate also hidden Symbols, by default hidden Symbols are not enumerated. Use together with DUK_ENUM_INCLUDE_SYMBOLS. In Duktape 1.x this flag was called DUK_ENUM_INCLUDE_INTERNAL.
DUK_ENUM_INCLUDE_SYMBOLS	Include Symbols in the enumeration result. Hidden Symbols are not included unless DUK_ENUM_INCLUDE_HIDDEN is specified.
DUK_ENUM_EXCLUDE_STRINGS	Exclude strings from the enumeration result. By default strings are included.
DUK_ENUM_OWN_PROPERTIES_ONLY	Enumerate only an object's "own" properties, by default also inherited properties are enumerated.
DUK_ENUM_ARRAY_INDICES_ONLY	Enumerate only array indices, i.e. property names of the form "0", "1", "2", etc.
DUK_ENUM_SORT_ARRAY_INDICES	Apply the ES2015 [[OwnPropertyKeys]] enumeration order over the whole enumeration result rather than per inheritance level, this has the effect of sorting array indices (even when inherited). Also symbols are sorted after ordinary string keys (both in insertion order).
DUK_ENUM_NO_PROXY_BEHAVIOR	Enumerate a Proxy object itself without invoking Proxy behaviors.
Without any flags the enumeration behaves like for-in: own and inherited enumerable properties are included, and enumeration order follows the ECMAScript ES2015 [[OwnPropertyKeys]] enumeration order, applied for each inheritance level.

Once the enumerator has been created, use duk_next() to extract keys (or key/value pairs) from the enumerator.

The ES2015 [[OwnPropertyKeys]] enumeration order is: (1) array indices in ascending order, (2) non-array-index keys in their insertion order, and (3) symbols in their insertion order. This rule is applied separately for each inheritance level, so that if array index keys are inherited, they will appear out-of-order in the result. For most practical code this is not an issue because array indices are very rarely inherited. You can force the whole enumeration sequence to be sorted using DUK_ENUM_SORT_ARRAY_INDICES.

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


## duk_equals() 1.0.0compare§

### プロトタイプ

```c
duk_bool_t duk_equals(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

. . . val1 . . . val2 . . .

### 要約

Compare values at idx1 and idx2 for equality. Returns 1 if values are considered equal using ECMAScript Equals operator (==) semantics, otherwise returns 0. Also returns 0 if either index is invalid.

Because The Abstract Equality Comparison Algorithm used by the Equals operator performs value coercion (ToNumber() and ToPrimitive()), the comparison may have side effects and may throw an error. The strict equality comparison, available through duk_strict_equals(), has no side effects.

Comparison algorithm for Duktape custom types is described in Equality (non-strict).


### 例

```c
if (duk_equals(ctx, -3, -7)) {
    printf("values at indices -3 and -7 are equal\n");
}
```

### 参照

duk_strict_equals


## duk_error() 1.0.0error§

### プロトタイプ

```c
duk_ret_t duk_error(duk_context *ctx, duk_errcode_t err_code, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Push a new Error object to the stack and throw it. This call never returns.

The message property of the error object will be set to a sprintf-formatted string using fmt and the remaining arguments. The internal prototype for the created error object is chosen based on err_code. For instance, DUK_ERR_RANGE_ERROR causes the built-in RangeError prototype to be used. The valid range for user error codes is [1,16777215].

To push an Error object to the stack without throwing it, use duk_push_error_object().

Even though the function never returns, the prototype describes a return value which allows code such as:

if (argvalue < 0) {
    return duk_error(ctx, DUK_ERR_TYPE_ERROR, "invalid argument value: %d", (int) argvalue);
}
If the return value is ignored, cast to void to avoid compilation warnings:

if (argvalue < 0) {
    (void) duk_error(ctx, DUK_ERR_TYPE_ERROR, "invalid argument value: %d", (int) argvalue);
}

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


## duk_error_va() 1.1.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_error_va(duk_context *ctx, duk_errcode_t err_code, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Vararg variant of duk_error().

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_eval() 1.0.0compile§

### プロトタイプ

```c
void duk_eval(duk_context *ctx);
```

### スタック

. . . source → . . . result

### 要約

Evaluate the ECMAScript source code at the top of the stack, and leave a single return value on top of the stack. May throw an error; such errors are not automatically caught. The filename associated with the temporary eval function is "eval".

This API call is essentially a shortcut for:

duk_push_string(ctx, "eval");
duk_compile(ctx, DUK_COMPILE_EVAL);  /* [ ... source filename ] -> [ function ] */
duk_call(ctx, 0);
The source code is evaluated in non-strict mode unless it contains an explicit "use strict" directive. In particular, strictness of the current context is not transferred into the eval code. This avoids confusing behavior where eval strictness would depend on whether eval is used inside a Duktape/C function call (strict mode) or outside of one (non-strict mode).

If the eval input is a fixed string, you can also use duk_eval_string().


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


## duk_eval_error() 2.0.0error§

### プロトタイプ

```c
duk_ret_t duk_eval_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error() with an error code DUK_ERR_EVAL_ERROR.


### 例

```c
return duk_eval_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_eval_error_va() 2.0.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_eval_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error_va() with an error code DUK_ERR_EVAL_ERROR.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_eval_lstring() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_eval_lstring(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

. . . → . . . result

### 要約

Like duk_eval(), but the eval input is given as a C string with explicit length.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_eval_lstring_noresult() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_eval_lstring_noresult(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

. . . → . . .

### 要約

Like duk_eval_lstring(), but leaves no result on the value stack.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
const char *src = /* ... */;
duk_size_t len = /* ... */;

duk_eval_lstring_noresult(ctx, src, len);
```


## duk_eval_noresult() 1.0.0compile§

### プロトタイプ

```c
void duk_eval_noresult(duk_context *ctx);
```

### スタック

. . . source → . . .

### 要約

Like duk_eval(), but leaves no result on the value stack.


### 例

```c
duk_push_string(ctx, "print('Hello world!');");
duk_eval_noresult(ctx);
```

### 参照

duk_eval_string_noresult
duk_eval_lstring_noresult


## duk_eval_string() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_eval_string(duk_context *ctx, const char *src);
```

### スタック

. . . → . . . result

### 要約

Like duk_eval(), but the eval input is given as a C string.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
duk_eval_string(ctx, "'testString'.toUpperCase()");
printf("result is: %s\n", duk_get_string(ctx, -1));
duk_pop(ctx);
```

### 参照

duk_eval_string_noresult


## duk_eval_string_noresult() 1.0.0stringcompile§

### プロトタイプ

```c
void duk_eval_string_noresult(duk_context *ctx, const char *src);
```

### スタック

. . . → . . .

### 要約

Like duk_eval_string(), but leaves no result on the value stack.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
duk_eval_string_noresult(ctx, "print('testString'.toUpperCase())");
```


## duk_fatal() 1.0.0error§

### プロトタイプ

```c
duk_ret_t duk_fatal(duk_context *ctx, const char *err_msg);
```

### スタック

(No effect on value stack.)


### 要約

Call fatal error handler with an optional message (err_msg may be NULL). Like all strings in Duktape, the error message should be an UTF-8 string, although pure ASCII is strongly recommended.

A fatal error handler never returns and may e.g. exit the current process. Error catching points (like try-catch statements and error catching API calls) are bypassed, and finalizers are not executed. You should only call this function when a truly fatal, unrecoverable error has occurred.

Even though the function never returns, the prototype describes a return value which allows code such as:

if (argvalue < 0) {
    return duk_fatal(ctx, "argvalue invalid, cannot continue execution");
}

### 例

```c
duk_fatal(ctx, "assumption failed");
```


## duk_free() 1.0.0memory§

### プロトタイプ

```c
void duk_free(duk_context *ctx, void *ptr);
```

### スタック

(No effect on value stack.)


### 要約

Like duk_free_raw() but may involve garbage collection steps. The garbage collection interaction cannot cause the operation to fail.

duk_free() can be used to free memory allocated with either duk_alloc() or duk_alloc_raw() and their reallocation variants.

Currently a duk_free() never causes a garbage collection pass.

### 例

```c
void *buf = duk_alloc(ctx, 1024);
/* ... */

duk_free(ctx, buf);  /* safe even if 'buf' is NULL */
```

### 参照

duk_free_raw


## duk_free_raw() 1.0.0memory§

### プロトタイプ

```c
void duk_free_raw(duk_context *ctx, void *ptr);
```

### スタック

(No effect on value stack.)


### 要約

Free memory allocated with the allocation function registered to the context. The operation cannot fail. If ptr is NULL, the call is a no-op.

duk_free_raw() can be used to free memory allocated with either duk_alloc() or duk_alloc_raw() and their reallocation variants.


### 例

```c
void *buf = duk_alloc_raw(ctx, 1024);
/* ... */

duk_free_raw(ctx, buf);  /* safe even if 'buf' is NULL */
```

### 参照

duk_free


## duk_freeze() 2.2.0propertyobject§

### プロトタイプ

```c
void duk_freeze(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

(No effect on value stack.)


### 要約

Equivalent to Object.freeze() for object at obj_idx; if frozen, the object is automatically compacted. If the index is invalid an error is thrown.


### 例

```c
duk_freeze(ctx, -3);
```


## duk_gc() 1.0.0memoryheap§

### プロトタイプ

```c
void duk_gc(duk_context *ctx, duk_uint_t flags);
```

### スタック

(No effect on value stack.)


### 要約

Force a mark-and-sweep garbage collection round.

The following flags are defined:

Define	Description
DUK_GC_COMPACT	Force object property table compaction
You may want to call this function twice to ensure even objects with finalizers are collected. Currently it takes two mark-and-sweep rounds to collect such objects. First round marks the object as finalizable and runs the finalizer. Second round ensures the object is still unreachable after finalization and then frees the object.


### 例

```c
duk_gc(ctx, 0);
```


## duk_generic_error() 2.0.0error§

### プロトタイプ

```c
duk_ret_t duk_generic_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error() with an error code DUK_ERR_ERROR.


### 例

```c
return duk_generic_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_generic_error_va() 2.0.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_generic_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error_va() with an error code DUK_ERR_ERROR.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_get_boolean() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_get_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get the boolean value at idx without modifying or coercing the value. Returns 1 if the value is true, 0 if the value is false, not a boolean, or the index is invalid.

Note that the value is not coerced, so even a "truthy" ECMAScript value (like a non-empty string) will be treated as false. If you want to coerce the value, use duk_to_boolean().


### 例

```c
if (duk_get_boolean(ctx, -3)) {
    printf("value is true\n");
}
```

### 参照

duk_get_boolean_default


## duk_get_boolean_default() 2.1.0stack§

### プロトタイプ

```c
duk_bool_t duk_get_boolean_default(duk_context *ctx, duk_idx_t idx, duk_bool_t def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_boolean() but with an explicit default value, returned when the value is not a boolean or the index is invalid.


### 例

```c
duk_bool_t flag_xyz = duk_get_boolean_default(ctx, 2, 1);  /* default: true */
```

### 参照

duk_get_boolean


## duk_get_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_get_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . .

### 要約

Get the data pointer for a (plain) buffer value at idx without modifying or coercing the value. Returns a non-NULL pointer if the value is a valid buffer with a non-zero size. For a zero-size buffer, may return a NULL or a non-NULL pointer. Returns NULL if the value is not a buffer or the index is invalid. If out_size is non-NULL, the size of the buffer is written to *out_size; 0 is written if the return value is NULL.

If the target value is a fixed buffer, the returned pointer is stable and won't change during the lifetime of the buffer. For dynamic and external buffers the pointer may change as the buffer is resized or reconfigured; the caller is responsible for ensuring pointer values returned from this API call are not used after the buffer is resized/reconfigured.

There is no reliable way to distinguish a zero-size buffer from a non-buffer based on the return values alone: a NULL with zero size is returned for a non-buffer. The same values may be returned for a zero-size buffer (although it is also possible that a non-NULL pointer is returned). Use duk_is_buffer() or duk_is_buffer_data() or when type checking a buffer or a buffer object.

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


## duk_get_buffer_data() 1.3.0stackbufferobjectbuffer§

### プロトタイプ

```c
void *duk_get_buffer_data(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . .

### 要約

Get the data pointer for a plain buffer or a buffer object (ArrayBuffer, Node.js Buffer, DataView, or TypedArray view) value at idx without modifying or coercing the value. Return a non-NULL pointer if the value is a valid buffer with a non-zero size. For a zero-size buffer, may return a NULL or a non-NULL pointer. Returns NULL if the value is not a buffer, the value is a buffer object whose "backing buffer" doesn't fully cover the buffer object's apparent size, or the index is invalid. If out_size is non-NULL, the size of the buffer is written to *out_size; 0 is written if the return value is NULL.

The data area indicated by the return pointer and length is the full buffer for a plain buffer value, and the active "slice" for a buffer object. The length returned is expressed in bytes (instead of elements), so that you can always access ptr[0] to ptr[len - 1]. For example:

For new Uint8Array(16) the return pointer would point to start of the array and length would be 16.
For new Uint32Array(16) the return pointer would point to start of the array and length would be 64.
For new Uint32Array(16).subarray(2, 6) the return pointer would point to the start of the subarray (2 x 4 = 8 bytes from the start of the Uint32Array) and length would be 16 (= (6-2) x 4).
If the target value or the underlying buffer value of a buffer object is a fixed buffer, the returned pointer is stable and won't change during the lifetime of the buffer. For dynamic and external buffers the pointer may change as the buffer is resized or reconfigured; the caller is responsible for ensuring pointer values returned from this API call are not used after the buffer is resized/reconfigured. Buffers created from ECMAScript code directly e.g. as new Buffer(8) are backed by an automatic fixed buffer, so their pointers are always stable.

In special cases it's possible that a buffer object is backed by a plain buffer which is smaller than the buffer object's apparent size. In these cases a NULL is returned; this ensures that the pointer and size returned can always be used safely. (This behavior may change even in minor versions.)

There is no reliable way to distinguish a zero-size buffer from a non-buffer based on the return values alone: a NULL with zero size is returned for a non-buffer. The same values may be returned for a zero-size buffer (although it is also possible that a non-NULL pointer is returned). Use duk_is_buffer() or duk_is_buffer_data() or when type checking a buffer or a buffer object.

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


## duk_get_buffer_data_default() 2.1.0stackbuffer§

### プロトタイプ

```c
void *duk_get_buffer_data_default(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

. . . val . . .

### 要約

Like duk_get_buffer_data() but with an explicit default value, returned when the value is not a plain buffer, a buffer object, or the index is invalid.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

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


## duk_get_buffer_default() 2.1.0stackbuffer§

### プロトタイプ

```c
void *duk_get_buffer_default(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

. . . val . . .

### 要約

Like duk_get_buffer() but with an explicit default value, returned when the value is not a buffer or the index is invalid.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

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


## duk_get_c_function() 1.0.0stackfunction§

### プロトタイプ

```c
duk_c_function duk_get_c_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get the Duktape/C function pointer (a duk_c_function) from an ECMAScript function object associated with a Duktape/C function. If the value is not such a function or idx is invalid, returns NULL.

If you prefer an error to be thrown for an invalid value or index, use duk_require_c_function().


### 例

```c
duk_c_function funcptr;

funcptr = duk_get_c_function(ctx, -3);
```

### 参照

duk_get_c_function_default


## duk_get_c_function_default() 2.1.0stackfunction§

### プロトタイプ

```c
duk_c_function duk_get_c_function_default(duk_context *ctx, duk_idx_t idx, duk_c_function def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_c_function() but with an explicit default value, returned when the value is not a Duktape/C function or the index is invalid.


### 例

```c
duk_c_function funcptr;

/* Native callback, default to nop_callback. */
funcptr = duk_get_c_function_default(ctx, -3, nop_callback);
```

### 参照

duk_get_c_function


## duk_get_context() 1.0.0stackborrowed§

### プロトタイプ

```c
duk_context *duk_get_context(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get a context pointer for a Duktape thread at idx. If the value at idx is not a Duktape thread or the index is invalid, returns NULL.

The returned context pointer is only valid while the Duktape thread is reachable from a garbage collection point of view.

If you prefer an error to be thrown for an invalid value or index, use duk_require_context().


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


## duk_get_context_default() 2.1.0stackborrowed§

### プロトタイプ

```c
duk_context *duk_get_context_default(duk_context *ctx, duk_idx_t idx, duk_context *def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_context() but with an explicit default value, returned when the value is not a Duktape thread or the index is invalid.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

### 例

```c
duk_context *target_ctx;

target_ctx = duk_get_context_default(ctx, 2, default_ctx);
```

### 参照

duk_get_context


## duk_get_current_magic() 1.0.0magicfunction§

### プロトタイプ

```c
duk_int_t duk_get_current_magic(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Get the 16-bit signed "magic" value associated with the running Duktape/C function. If there is no current activation, zero is returned.

The magic function allows the same Duktape/C function to be used with slightly different behavior; behavior flags or other parameters can be passed in the magic field. This is less expensive than having behavior related flags or properties in the function object as normal properties.

If you need an unsigned 16-bit value, simply bitwise AND the result:

unsigned int my_flags = ((unsigned int) duk_get_current_magic(ctx)) & 0xffffU;

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


## duk_get_error_code() 1.1.0stackerror§

### プロトタイプ

```c
duk_errcode_t duk_get_error_code(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Map the value at idx to the error codes DUK_ERR_xxx based on which Error subclass the value inherits from. For example, if the value at the stack top is an user-defined error which inherits from ReferenceError, the return value will be DUK_ERR_REFERENCE_ERROR. If the value inherits from Error but doesn't inherit from any of the standard subclasses (EvalError, RangeError, ReferenceError, SyntaxError, TypeError, URIError) DUK_ERR_ERROR is returned. If the value is not an object, does not inherit from Error, or idx is invalid, returns 0 (= DUK_ERR_NONE).


### 例

```c
if (duk_get_error_code(ctx, -3) == DUK_ERR_URI_ERROR) {
    printf("Invalid URI\n");
}
```


## duk_get_finalizer() 1.0.0objectfinalizer§

### プロトタイプ

```c
void duk_get_finalizer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . val . . . finalizer

### 要約

Get the finalizer associated with a value at idx. If the value is not an object or it is an object without a finalizer, undefined is pushed to the stack instead.


### 例

```c
/* Get the finalizer of an object at index -3. */
duk_get_finalizer(ctx, -3);
```

### 参照

duk_set_finalizer


## duk_get_global_heapptr() 2.3.0propertyheapptrborrowed§

### プロトタイプ

```c
duk_bool_t duk_get_global_heapptr(duk_context *ctx, void *ptr);
```

### スタック

. . . → . . . val (if key exists)
. . . → . . . undefined (if key doesn't exist)

### 要約

Like duk_get_global_string(), but the property name is given as a Duktape heap pointer obtained e.g. using duk_get_heapptr(). If ptr is NULL, undefined is used as the key.


### 例

```c
(void) duk_get_global_heapptr(ctx, my_ptr_ref);
```

### 参照

duk_get_global_string
duk_get_global_lstring
duk_get_global_literal


## duk_get_global_literal() 2.3.0propertyliteral§

### プロトタイプ

```c
duk_bool_t duk_get_global_literal(duk_context *ctx, const char *key_literal);
```

### スタック

. . . → . . . val (if key exists)
. . . → . . . undefined (if key doesn't exist)

### 要約

Like duk_get_global_string(), but the property name is given as a string literal (see duk_push_literal()).


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


## duk_get_global_lstring() 2.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_get_global_lstring(duk_context *ctx, const char *key, duk_size_t key_len);
```

### スタック

. . . → . . . val (if key exists)
. . . → . . . undefined (if key doesn't exist)

### 要約

Like duk_get_global_string() but the key is given as a string with explicit length.


### 例

```c
(void) duk_get_global_lstring(ctx, "internal" "\x00" "nul", 12);
```

### 参照

duk_get_global_string
duk_get_global_literal
duk_get_global_heapptr


## duk_get_global_string() 1.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_get_global_string(duk_context *ctx, const char *key);
```

### スタック

. . . → . . . val (if key exists)
. . . → . . . undefined (if key doesn't exist)

### 要約

Get property named key from the global object. Returns non-zero if the property exists and zero otherwise. This is a convenience function which does the equivalent of:

duk_bool_t ret;

duk_push_global_object(ctx);
ret = duk_get_prop_string(ctx, -1, key);
duk_remove(ctx, -2);
/* 'ret' would be the return value from duk_get_global_string() */

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


## duk_get_heapptr() 1.1.0stackheapptrborrowed§

### プロトタイプ

```c
void *duk_get_heapptr(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get a borrowed void * reference to a Duktape heap allocated value (object, buffer, string) at idx. Return NULL if the index is invalid or the target value is not heap allocated. The returned pointer must not be interpreted or dereferenced, but duk_push_heapptr() can be used to push the original value into the value stack later.

The returned void pointer is only valid while the original value is reachable from a garbage collection point of view. If this is not the case, it is memory unsafe to use duk_push_heapptr().


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


## duk_get_heapptr_default() 2.1.0stackheapptrborrowed§

### プロトタイプ

```c
void *duk_get_heapptr_default(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_heapptr() but with an explicit default value, returned when the value is not a boolean or the index is invalid.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

### 例

```c
void *ptr;

ptr = duk_get_heapptr_default(ctx, 2, default_ptr);
```

### 参照

duk_get_heapptr


## duk_get_int() 1.0.0stack§

### プロトタイプ

```c
duk_int_t duk_get_int(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get the number at idx and convert it to a C duk_int_t by first clamping the value between [DUK_INT_MIN, DUK_INT_MAX] and then truncating towards zero. The value on the stack is not modified. If the value is a NaN, is not a number, or the index is invalid, returns 0.

Conversion examples:

Input	Output
-Infinity	DUK_INT_MIN
DUK_INT_MIN - 1	DUK_INT_MIN
-3.9	-3
3.9	3
DUK_INT_MAX + 1	DUK_INT_MAX
+Infinity	DUK_INT_MAX
NaN	0
"123"	0 (non-number)
The coercion is different from a basic C cast from double to integer, which may have counterintuitive (and non-portable) behavior like coercing NaN to DUK_INT_MIN. The coercion is also different from ECMAScript ToInt32() coercion because the full range of the native duk_int_t is allowed (which may be more than 32 bits).

### 例

```c
printf("int value: %ld\n", (long) duk_get_int(ctx, -3));
```

### 参照

duk_get_int_default


## duk_get_int_default() 2.1.0stack§

### プロトタイプ

```c
duk_int_t duk_get_int_default(duk_context *ctx, duk_idx_t idx, duk_int_t def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_int() but with an explicit default value, returned when the value is not a number or the index is invalid.


### 例

```c
int port = (int) duk_get_int_default(ctx, 1, 80);  /* default: 80 */
```

### 参照

duk_get_int


## duk_get_length() 1.0.0stack§

### プロトタイプ

```c
duk_size_t duk_get_length(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get a type-specific "length" for value at idx:

String: character length of string (not byte length)
Object: Math.floor(ToNumber(obj.length)) if result within duk_size_t unsigned range, otherwise 0 (for Arrays the result is Array .length)
Buffer: byte length of buffer
Other type or invalid stack index: 0
To get the byte length of a string, use duk_get_lstring().


### 例

```c
if (duk_is_string(ctx, -3)) {
    printf("string char len is %lu\n", (unsigned long) duk_get_length(ctx, -3));
}
```

### 参照

duk_set_length


## duk_get_lstring() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_get_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

. . . val . . .

### 要約

Get character data pointer and length for a string at idx without modifying or coercing the value. Returns a non-NULL pointer to the read-only, NUL-terminated string data, and writes the string byte length to *out_len (if out_len is non-NULL). Returns NULL and writes zero to *out_len (if out_len is non-NULL) if the value is not a string or the index is invalid.

To get the string character length (instead of byte length), use duk_get_length().

A non-NULL return value is guaranteed even for zero length strings; this differs from how buffer data pointers are handled (for technical reasons).

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


## duk_get_lstring_default() 2.1.0stringstack§

### プロトタイプ

```c
const char *duk_get_lstring_default(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len, const char *def_ptr, duk_size_t def_len);
```

### スタック

. . . val . . .

### 要約

Like duk_get_lstring() but with an explicit default value, returned when the value is not a string or the index is invalid.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

### 例

```c
const char *str;
duk_size_t len;

str = duk_get_lstring_default(ctx, -3, &len, "foo" "\x00" "bar", 7);
```

### 参照

duk_get_string_default


## duk_get_magic() 1.0.0magicfunction§

### プロトタイプ

```c
duk_int_t duk_get_magic(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get the 16-bit signed "magic" value associated with the Duktape/C function at idx. If the value is not a Duktape/C function, an error is thrown.

Lightweight functions have space for only 8 magic value bits which is interpreted as a signed integer (-128 to 127).

### 例

```c
duk_int_t my_flags = duk_get_magic(ctx, -3);
```

### 参照

duk_get_current_magic
duk_set_magic


## duk_get_memory_functions() 1.0.0memoryheap§

### プロトタイプ

```c
void duk_get_memory_functions(duk_context *ctx, duk_memory_functions *out_funcs);
```

### スタック

(No effect on value stack.)


### 要約

Get the memory management functions used by the context.

Normally there is no reason to call this function: you can use the memory management primitives through wrapped memory management functions such as duk_alloc(), duk_realloc(), and duk_free().


### 例

```c
duk_memory_functions funcs;

duk_get_memory_functions(ctx, &funcs);
```


## duk_get_now() 2.0.0time§

### プロトタイプ

```c
duk_double_t duk_get_now(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Get current time in POSIX milliseconds, as seen by the ECMAScript environment. The return value matches Date.now() with the reservation that sub-millisecond resolution may be available.


### 例

```c
duk_double_t now;

now = duk_get_now(ctx);
print("timestamp: %lf\n", (double) now);
```


## duk_get_number() 1.0.0stack§

### プロトタイプ

```c
duk_double_t duk_get_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get the number value at idx without modifying or coercing the value. Returns NaN if the value is not a number or the index is invalid.


### 例

```c
printf("value: %lf\n", (double) duk_get_number(ctx, -3));
```

### 参照

duk_get_number_default


## duk_get_number_default() 2.1.0stack§

### プロトタイプ

```c
duk_double_t duk_get_number_default(duk_context *ctx, duk_idx_t idx, duk_double_t def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_number() but with an explicit default value, returned when the value is not a number or the index is invalid.


### 例

```c
duk_double_t backoff_multiplier = duk_get_number_default(ctx, 2, 1.5);  /* default: 1.5 */
```

### 参照

duk_get_number


## duk_get_pointer() 1.0.0stack§

### プロトタイプ

```c
void *duk_get_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get the pointer value at idx as void * without modifying or coercing the value. Returns NULL if the value is not a pointer or the index is invalid.


### 例

```c
void *ptr;

ptr = duk_get_pointer(ctx, -3);
printf("my pointer is: %p\n", ptr);
```


## duk_get_pointer_default() 2.1.0stack§

### プロトタイプ

```c
void *duk_get_pointer_default(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_pointer() but with an explicit default value, returned when the value is not a pointer or the index is invalid.


### 例

```c
void *ptr;

ptr = duk_get_pointer_default(ctx, -3, (void *) 0x12345678);
printf("my pointer is: %p\n", ptr);
```

### 参照

duk_get_pointer


## duk_get_prop() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_get_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

. . . obj . . . key → . . . obj . . . val (if key exists)
. . . obj . . . key → . . . obj . . . undefined (if key doesn't exist)

### 要約

Get the property key of a value at obj_idx. Return code and error throwing behavior:

If the property exists, 1 is returned and key is replaced by the property value on the value stack. If the property is an accessor, the "getter" function may throw an error.
If the property does not exist, 0 is returned and key is replaced by undefined on the value stack.
If the value at obj_idx is not object coercible, an error is thrown.
If obj_idx is invalid, an error is thrown.
The property read is equivalent to the ECMAScript expression res = obj[key] with the exception that the presence or absence of the property is indicated by the call return value. For precise semantics, see Property Accessors, GetValue (V), and [[Get]] (P). Both the target value and the key are coerced:

The target value is automatically coerced to an object. For instance, a string is converted to a String and you can access its "length" property.
The key argument is internally coerced using ToPropertyKey() coercion which results in a string or a Symbol. There is an internal fast path for arrays and numeric indices which avoids an explicit string coercion, so use a numeric key when applicable.
If the target is a Proxy object which implements the get trap, the trap is invoked and the API call always returns 1 (i.e. property present): the absence/presence of properties is not indicated by the get Proxy trap. Thus, the API call return value may be of limited use if the target object is potentially a Proxy.

If the key is a fixed string you can avoid one API call and use the duk_get_prop_string() variant. Similarly, if the key is an array index, you can use the duk_get_prop_index() variant.

Although the base value for property accesses is usually an object, it can technically be an arbitrary value. Plain string and buffer values have virtual index properties so you can access "foo"[2], for instance. Most primitive values also inherit from some prototype object so that you can e.g. call methods on them: (12345).toString(16).

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


## duk_get_prop_desc() 2.0.0sandboxproperty§

### プロトタイプ

```c
void duk_get_prop_desc(duk_context *ctx, duk_idx_t obj_idx, duk_uint_t flags);
```

### スタック

. . . obj . . . key → . . . obj . . . desc (if property exists)
. . . obj . . . key → . . . obj . . . undefined (if property doesn't exist)

### 要約

Equivalent of Object.getOwnPropertyDescriptor() in the C API: pushes a property descriptor object for a named property of the object at obj_idx. If the target is not an object (or the index is invalid) an error is thrown.

No flags are defined yet, use 0 for flags.


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


## duk_get_prop_heapptr() 2.2.0propertyheapptrborrowed§

### プロトタイプ

```c
duk_bool_t duk_get_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

. . . obj . . . → . . . obj . . . val (if key exists)
. . . obj . . . → . . . obj . . . undefined (if key doesn't exist)

### 要約

Like duk_get_prop(), but the property name is given as a Duktape heap pointer obtained e.g. using duk_get_heapptr(). If ptr is NULL, undefined is used as the key.


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


## duk_get_prop_index() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_get_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

. . . obj . . . → . . . obj . . . val (if key exists)
. . . obj . . . → . . . obj . . . undefined (if key doesn't exist)

### 要約

Like duk_get_prop(), but the property name is given as an unsigned integer arr_idx. This is especially useful for accessing array elements (but is not limited to that).

Conceptually the number is coerced to a string for the property read, e.g. 123 would be equivalent to a property name "123". Duktape avoids an explicit coercion whenever possible.


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


## duk_get_prop_literal() 2.3.0propertyliteral§

### プロトタイプ

```c
duk_bool_t duk_get_prop_literal(duk_context *ctx, const char *key_literal);
```

### スタック

. . . obj . . . → . . . obj . . . val (if key exists)
. . . obj . . . → . . . obj . . . undefined (if key doesn't exist)

### 要約

Like duk_get_prop(), but the property name is given as a string literal (see duk_push_literal()).


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


## duk_get_prop_lstring() 2.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_get_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

. . . obj . . . → . . . obj . . . val (if key exists)
. . . obj . . . → . . . obj . . . undefined (if key doesn't exist)

### 要約

Like duk_get_prop(), but the property name is given as a string with explicit length.


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


## duk_get_prop_string() 1.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_get_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

. . . obj . . . → . . . obj . . . val (if key exists)
. . . obj . . . → . . . obj . . . undefined (if key doesn't exist)

### 要約

Like duk_get_prop(), but the property name is given as a NUL-terminated C string key.


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


## duk_get_prototype() 1.0.0prototypeobject§

### プロトタイプ

```c
void duk_get_prototype(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . val . . . proto

### 要約

Get the internal prototype of the value at idx. If the value is not an object, an error is thrown. If the object has no prototype (which is possible for "bare objects") undefined is pushed to the stack instead.


### 例

```c
/* Get the internal prototype of an object at index -3. */
duk_get_prototype(ctx, -3);
```

### 参照

duk_set_prototype


## duk_get_string() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_get_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get character data pointer for a string at idx without modifying or coercing the value. Returns a non-NULL pointer to the read-only, NUL-terminated string data. Returns NULL if the value is not a string or the index is invalid.

To get the string byte length explicitly (which is useful if the string contains embedded NUL characters), use duk_get_lstring().

A non-NULL return value is guaranteed even for zero length strings; this differs from how buffer data pointers are handled (for technical reasons).
Symbol values are visible in the C API as strings so that both duk_is_symbol() and duk_is_string() are true. This behavior is similar to Duktape 1.x internal strings. You can distinguish Symbols from ordinary strings using duk_is_symbol(). For the internal representation, see symbols.rst.

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


## duk_get_string_default() 2.1.0stringstack§

### プロトタイプ

```c
const char *duk_get_string_default(duk_context *ctx, duk_idx_t idx, const char *def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_string() but with an explicit default value, returned when the value is not a string or the index is invalid.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

### 例

```c
const char *host = duk_get_string_default(ctx, 3, "localhost");
```

### 参照

duk_get_lstring_default


## duk_get_top() 1.0.0stack§

### プロトタイプ

```c
duk_idx_t duk_get_top(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Get current stack top (>= 0), indicating the number of values currently on the value stack (of the current activation).


### 例

```c
printf("stack top is %ld\n", (long) duk_get_top(ctx));
```


## duk_get_top_index() 1.0.0stack§

### プロトタイプ

```c
duk_idx_t duk_get_top_index(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Get the absolute index (>= 0) of the topmost value on the stack. If the stack is empty, returns DUK_INVALID_INDEX.


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


## duk_get_type() 1.0.0stack§

### プロトタイプ

```c
duk_int_t duk_get_type(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns type of value at idx. The return value is one of DUK_TYPE_xxx or DUK_TYPE_NONE if idx is invalid.

Symbol values are visible in the C API as strings so that both duk_is_symbol() and duk_is_string() are true. This behavior is similar to Duktape 1.x internal strings. You can distinguish Symbols from ordinary strings using duk_is_symbol(). For the internal representation, see symbols.rst.

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


## duk_get_type_mask() 1.0.0stack§

### プロトタイプ

```c
duk_uint_t duk_get_type_mask(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns type mask of value at idx. The return value is one of DUK_TYPE_MASK_xxx or DUK_TYPE_MASK_NONE if idx is invalid.

Type masks allow e.g. for a convenient comparison for multiple types at once (the duk_check_type_mask() call is even more convenient for this purpose).

Symbol values are visible in the C API as strings so that both duk_is_symbol() and duk_is_string() are true. This behavior is similar to Duktape 1.x internal strings. You can distinguish Symbols from ordinary strings using duk_is_symbol(). For the internal representation, see symbols.rst.

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


## duk_get_uint() 1.0.0stack§

### プロトタイプ

```c
duk_uint_t duk_get_uint(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Get the number at idx and convert it to a C duk_uint_t by first clamping the value between [0, DUK_UINT_MAX] and then truncating towards zero. The value on the stack is not modified. If the value is a NaN, is not a number, or the index is invalid, returns 0.

Conversion examples:

Input	Output
-Infinity	0
-1	0
-3.9	0
3.9	3
DUK_UINT_MAX + 1	DUK_UINT_MAX
+Infinity	DUK_UINT_MAX
NaN	0
"123"	0 (non-number)
The coercion is different from a basic C cast from double to unsigned integer, which may have counterintuitive (and non-portable) behavior for e.g. NaN values. The coercion is also different from ECMAScript ToUint32() coercion because the full range of the native duk_uint_t is allowed (which may be more than 32 bits).

### 例

```c
printf("unsigned int value: %lu\n", (unsigned long) duk_get_uint(ctx, -3));
```

### 参照

duk_get_uint_default


## duk_get_uint_default() 2.1.0stack§

### プロトタイプ

```c
duk_uint_t duk_get_uint_default(duk_context *ctx, duk_idx_t idx, duk_uint_t def_value);
```

### スタック

. . . val . . .

### 要約

Like duk_get_uint() but with an explicit default value, returned when the value is not a number or the index is invalid.


### 例

```c
unsigned int count = (unsigned int) duk_get_uint_default(ctx, 1, 3);  /* default: 3 */
```

### 参照

duk_get_uint


## duk_has_prop() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_has_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

. . . obj . . . key → . . . obj . . .

### 要約

Check whether value at obj_idx has a property key. key is removed from the stack. Return code and error throwing behavior:

If the property exists, 1 is returned.
If the property doesn't exist, 0 is returned.
If the value at obj_idx is not an object, an error is thrown.
If obj_idx is invalid, an error is thrown.
The property existence check is equivalent to the ECMAScript expression res = key in obj. For semantics, see Property Accessors, The in operator, and [[HasProperty]] (P). The key is coerced:

The key argument is internally coerced using ToPropertyKey() coercion which results in a string or a Symbol. There is an internal fast path for arrays and numeric indices which avoids an explicit string coercion, so use a numeric key when applicable.
If the target is a Proxy object which implements the has trap, the trap is invoked and the API call return value matches the trap return value.

Instead of accepting any object coercible value (like most property related API calls) this call accepts only an object as its target value. This is intentional as it follows ECMAScript operator semantics.
If the key is a fixed string you can avoid one API call and use the duk_has_prop_string() variant. Similarly, if the key is an array index, you can use the duk_has_prop_index() variant.


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


## duk_has_prop_heapptr() 2.2.0propertyheapptrborrowed§

### プロトタイプ

```c
duk_bool_t duk_has_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_has_prop(), but the property name is given as a Duktape heap pointer obtained e.g. using duk_get_heapptr(). If ptr is NULL, undefined is used as the key.


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


## duk_has_prop_index() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_has_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_has_prop(), but the property name is given as an unsigned integer arr_idx. This is especially useful for checking existence of array elements (but is not limited to that).

Conceptually the number is coerced to a string for property existence check, e.g. 123 would be equivalent to a property name "123". Duktape avoids an explicit coercion whenever possible.


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


## duk_has_prop_literal() 2.3.0propertyliteral§

### プロトタイプ

```c
duk_bool_t duk_has_prop_literal(duk_context *ctx, duk_idx_t obj_idx, const char *key_literal);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_has_prop(), but the property name is given as a string literal (see duk_push_literal()).


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


## duk_has_prop_lstring() 2.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_has_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_has_prop(), but the property name is given as a string with explicit length.


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


## duk_has_prop_string() 1.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_has_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Like duk_has_prop(), but the property name is given as a NUL-terminated C string key.


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


## duk_hex_decode() 1.0.0hexcodec§

### プロトタイプ

```c
void duk_hex_decode(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . hex_val . . . → . . . val . . .

### 要約

Decodes a hex encoded value into a buffer as an in-place operation. If the input is invalid, throws an error.


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


## duk_hex_encode() 1.0.0hexcodec§

### プロトタイプ

```c
const char *duk_hex_encode(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . hex_val . . .

### 要約

Coerces an arbitrary value into a buffer and then encodes the result into hex as an in-place operation. Returns pointer to the resulting string for convenience.

Coercing to a buffer first coerces a non-buffer value into a string, and then coerces the string into a buffer. The resulting buffer contains the string in CESU-8 encoding.

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


## duk_insert() 1.0.0stack§

### プロトタイプ

```c
void duk_insert(duk_context *ctx, duk_idx_t to_idx);
```

### スタック

. . . old(to_idx) . . . val → . . . val(to_idx) old . . .

### 要約

Insert a value at to_idx with a value popped from the stack top. The previous value at to_idx and any values above it are moved up the stack by a step. If to_idx is an invalid index, throws an error.

Negative indices are evaluated prior to popping the value at the stack top. This is also illustrated by the example.

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


## duk_inspect_callstack_entry() 2.0.0stackinspect§

### プロトタイプ

```c
void duk_inspect_callstack_entry(duk_context *ctx, duk_int_t level);
```

### スタック

. . . → . . . info

### 要約

Inspect callstack entry at level and push an object containing Duktape specific internal information about the entry. The level argument must be negative and mimics the value stack index convention: -1 is the most recent (innermost) call, -2 is its caller, and so on. If the level argument is invalid (e.g. outside of current callstack) undefined is pushed instead.

The result object is not under versioning guarantees so its properties may change even in minor releases (but not patch releases). This is a practical compromise: internals change quite frequently, so the choices are either to make no versioning guarantees or avoid exposing internals at all. As such, calling code should never rely on having a certain set of fields available, and it may be necessary to check DUK_VERSION when interpreting the result fields.
The following table summarizes current properties.

Property	Description
function	Function being executed. Note that this is a potential sandboxing concern if exposed to untrusted code.
pc	Program counter for ECMAScript functions. Zero if executing a native function.
lineNumber	Line number for ECMAScript functions. Zero if executing a native function or if pc-to-line translation data is not available.

### 例

```c
duk_inspect_callstack_entry(ctx, -1);
duk_get_prop_string(ctx, -1, "lineNumber");
printf("immediate caller is executing on line %ld\n", (long) duk_to_int(ctx, -1));
duk_pop_2(ctx);
```


## duk_inspect_value() 2.0.0stackinspect§

### プロトタイプ

```c
void duk_inspect_value(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . val . . . info

### 要約

Inspect value at idx and push an object containing Duktape specific internal information about the object. If the value stack index is invalid, pushes an object describing a "none" value.

The result object is not under versioning guarantees so its properties may change even in minor releases (but not patch releases). This is a practical compromise: internals change quite frequently, so the choices are either to make no versioning guarantees or avoid exposing internals at all. As such, calling code should never rely on having a certain set of fields available, and it may be necessary to check DUK_VERSION when interpreting the result fields.
The following table summarizes current properties. Memory byte sizes don't include any heap overhead which may vary between 0-16 bytes (or more) depending on the allocation functions being used.

Property	Description
type	Type number matching DUK_TYPE_xxx from duktape.h.
itag	Internal type tag matching internal DUK_TAG_xxx defines. Values may change between versions, and are dependent on config options and the memory layout used for tagged values internally.
hptr	Heap pointer for a heap-allocated value. Points to the internal Duktape header structure related to the value type. Same value would be returned from duk_get_heapptr().
refc	Reference count. Reference counts are not adjusted in any way, and include references to the value caused by the duk_inspect_value() call.
class	For objects, internal class number, matches internal DUK_HOBJECT_CLASS_xxx defines.
hbytes	Byte size of main heap object allocation. For some values this is the only allocation, other values have additional allocations.
pbytes	Byte size of an object's property table. The property table includes a possible array part, a possible hash part, and a key/value entry part.
bcbytes	Byte size of ECMAScript function bytecode (instructions, constants). Shared between all instances (closures) of a certain function template.
dbytes	Byte size of the current allocation of a dynamic or external buffer. Note that external buffer allocations are not part of the Duktape heap.
esize	Object entry part size in elements.
enext	Object entry part first free index (= index of next property slot to be used). In practice matches number of own properties for objects that don't have uncompacted deleted keys.
asize	Object array part size in elements, zero if no array part or array part has been abandoned (sparse array). May be larger or smaller than the apparent array .length.
hsize	Object hash part size in elements.
tstate	Internal thread state, matches internal DUK_HTHREAD_STATE_xxx defines.
variant	Identifies type variants for certain types. For strings, variant 0 is an ordinary heap allocated string while variant 1 is an external string. For buffers, variant 0 is a fixed buffer, 1 is a dynamic buffer, and 2 is an external buffer.

### 例

```c
duk_inspect_value(ctx, -3);
duk_get_prop_string(ctx, -1, "refc");
printf("refcount of value at -3: %ld\n", (long) duk_to_int(ctx, -1));
duk_pop_2(ctx);
```


## duk_instanceof() 1.3.0compare§

### プロトタイプ

```c
duk_bool_t duk_instanceof(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

. . . val1 . . . val2 . . .

### 要約

Compare values at idx1 and idx2 using the ECMAScript instanceof operator. Returns 1 if val1 instanceof val2, 0 if not. Throws an error if either index is invalid; instanceof itself also throws errors for invalid argument types.

The error throwing behavior for invalid indices differs from the behavior of e.g. duk_equals(), and matches the strictness of instanceof. For example, if rval (idx2) is not a callable object instanceof throws a TypeError. Throwing an error for an invalid index is consistent with instanceof strictness.

### 例

```c
duk_idx_t idx_val;

duk_get_global_string(ctx, "Error");

if (duk_instanceof(ctx, idx_val, -1)) {
    printf("value at idx_val is an instanceof Error\n");
}
```


## duk_is_array() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_array(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is an object and is an ECMAScript array (has the internal class Array), otherwise returns 0. Also returns 1 if the value is a Proxy wrapping an Array. If idx is invalid, also returns 0.

This function returns 1 when the following ECMAScript expression is true:

Object.prototype.toString.call(val) === '[object Array]'

### 例

```c
if (duk_is_array(ctx, -3)) {
    /* ... */
}
```


## duk_is_boolean() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a boolean, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_boolean(ctx, -3)) {
    /* ... */
}
```


## duk_is_bound_function() 1.0.0stackfunction§

### プロトタイプ

```c
duk_bool_t duk_is_bound_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a Function object which has been created with ECMAScript Function.prototype.bind(), otherwise returns 0. If idx is invalid, also returns 0.

Bound functions are an ECMAScript concept: they point to a target function and supply a this binding and zero or more argument bindings. See Function.prototype.bind (thisArg [, arg1 [, arg2, ...]]).


### 例

```c
if (duk_is_bound_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
duk_bool_t duk_is_buffer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a plain buffer, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_buffer(ctx, -3)) {
    /* ... */
}
```

### 参照

duk_is_buffer_data


## duk_is_buffer_data() 2.0.0stackbufferobjectbuffer§

### プロトタイプ

```c
duk_bool_t duk_is_buffer_data(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a plain buffer or any buffer object type, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_buffer_data(ctx, -3)) {
    /* ... */
}
```

### 参照

duk_is_buffer


## duk_is_c_function() 1.0.0stackfunction§

### プロトタイプ

```c
duk_bool_t duk_is_c_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a Function object and is associated with a C function (called a "Duktape/C function"), otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_c_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_callable() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_callable(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Return 1 if value at idx is callable, otherwise returns 0. Also returns 0 if idx is invalid.

Currently this is the same as duk_is_function().


### 例

```c
if (duk_is_callable(ctx, -3)) {
    /* ... */
}
```


## duk_is_constructable() 2.2.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_constructable(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Return 1 if value at idx is constructable, otherwise returns 0. Also returns 0 if idx is invalid.


### 例

```c
if (duk_is_constructable(ctx, -3)) {
    /* ... */
}
```


## duk_is_constructor_call() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_constructor_call(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Return non-zero if current function was called as a constructor (new Foo() instead of Foo()); otherwise returns 0.

This call allows a C function to have different behavior for normal and constructor calls (as is the case for many built-in functions).


### 例

```c
if (duk_is_constructor_call(ctx)) {
    printf("called as a constructor\n");
} else {
    printf("called as a normal function\n");
}
```


## duk_is_dynamic_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
duk_bool_t duk_is_dynamic_buffer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a dynamic buffer, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_dynamic_buffer(ctx, -3)) {
    /* ... */
}
```


## duk_is_ecmascript_function() 1.0.0stackfunction§

### プロトタイプ

```c
duk_bool_t duk_is_ecmascript_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a Function object which has been compiled from ECMAScript source code, otherwise returns 0. If idx is invalid, also returns 0.

Internally ECMAScript functions are compiled into bytecode and executed with the ECMAScript bytecode interpreter.


### 例

```c
if (duk_is_ecmascript_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_error() 1.1.0stackerror§

### プロトタイプ

```c
duk_bool_t duk_is_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx inherits from Error, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_error(ctx, -3)) {
    /* Inherits from Error, attempt to print stack trace */
    duk_get_prop_string(ctx, -3, "stack");
    printf("%s\n", duk_safe_to_string(ctx, -1));
    duk_pop(ctx);
}
```


## duk_is_eval_error() 1.4.0stackerror§

### プロトタイプ

```c
duk_bool_t duk_is_eval_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx inherits from EvalError, otherwise returns 0. If idx is invalid, also returns 0. This is a convenience call for using duk_get_error_code() == DUK_ERR_EVAL_ERROR.


### 例

```c
if (duk_is_eval_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_fixed_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
duk_bool_t duk_is_fixed_buffer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a fixed buffer, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_fixed_buffer(ctx, -3)) {
    /* ... */
}
```


## duk_is_function() 1.0.0stackfunction§

### プロトタイプ

```c
duk_bool_t duk_is_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is an object and is a function (has the internal class Function), otherwise returns 0. If idx is invalid, also returns 0.

This function returns 1 when the following ECMAScript expression is true:

Object.prototype.toString.call(val) === '[object Function]'
To determine the specific type of the function, use:

duk_is_c_function()
duk_is_ecmascript_function()
duk_is_bound_function()

### 例

```c
if (duk_is_function(ctx, -3)) {
    /* ... */
}
```


## duk_is_lightfunc() stack§

### プロトタイプ

```c
duk_bool_t duk_is_lightfunc(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a lightfunc, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_lightfunc(ctx, -3)) {
    /* ... */
}
```


## duk_is_nan() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_nan(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a NaN (a special number value), otherwise returns 0. If idx is invalid, also returns 0.

IEEE doubles have a large number of different NaN values. Duktape may normalize NaN values internally. This function returns 1 for any kind of a NaN.


### 例

```c
if (duk_is_nan(ctx, -3)) {
    /* ... */
}
```


## duk_is_null() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_null(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is either null, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_null(ctx, -3)) {
    /* ... */
}
```


## duk_is_null_or_undefined() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_null_or_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is either null or undefined, otherwise returns 0. If idx is invalid, also returns 0.

This API call is similar to a (x == null) comparison in ECMAScript, which is true for both null and undefined.


### 例

```c
if (duk_is_null_or_undefined(ctx, -3)) {
    /* ... */
}
```


## duk_is_number() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a number, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_number(ctx, -3)) {
    /* ... */
}
```


## duk_is_object() 1.0.0stackobject§

### プロトタイプ

```c
duk_bool_t duk_is_object(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is an object, otherwise returns 0. If idx is invalid, also returns 0.

Note that many values are considered to be an object, e.g.:

ECMAScript object
ECMAScript array
ECMAScript function
Duktape thread (coroutine)
Duktape internal objects
Specific object types can be checked with separate API calls, e.g. duk_is_array().


### 例

```c
if (duk_is_object(ctx, -3)) {
    /* ... */
}
```


## duk_is_object_coercible() 1.0.0stackobject§

### プロトタイプ

```c
duk_bool_t duk_is_object_coercible(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is object coercible, as defined in CheckObjectCoercible, otherwise returns 0. If idx is invalid, also returns 0.

All ECMAScript types are object coercible except undefined and null. The custom buffer and pointer types are object coercible.


### 例

```c
if (duk_is_object_coercible(ctx, -3)) {
    /* ... */
}
```


## duk_is_pointer() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a pointer, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_pointer(ctx, -3)) {
    /* ... */
}
```


## duk_is_primitive() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_primitive(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a primitive type, as defined in ToPrimitive, otherwise returns 0. If idx is invalid, also returns 0.

Any standard type other than an object is a primitive type. The custom plain pointer type is also considered a primitive type. However, the custom plain buffer type (which behaves like an Uint8Array object in most situations) and lightfunc type (which behaves like a Function object in most situations) are not considered a primitive type. This matches the behavior of duk_to_primitive() which (usually) coerces e.g. a plain buffer to the string [object Uint8Array].


### 例

```c
if (duk_is_primitive(ctx, -3)) {
    /* ... */
}
```


## duk_is_range_error() 1.4.0stackerror§

### プロトタイプ

```c
duk_bool_t duk_is_range_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx inherits from RangeError, otherwise returns 0. If idx is invalid, also returns 0. This is a convenience call for using duk_get_error_code() == DUK_ERR_RANGE_ERROR.


### 例

```c
if (duk_is_range_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_reference_error() 1.4.0stackerror§

### プロトタイプ

```c
duk_bool_t duk_is_reference_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx inherits from ReferenceError, otherwise returns 0. If idx is invalid, also returns 0. This is a convenience call for using duk_get_error_code() == DUK_ERR_REFERENCE_ERROR.


### 例

```c
if (duk_is_reference_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_strict_call() 1.0.0function§

### プロトタイプ

```c
duk_bool_t duk_is_strict_call(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Check whether the current (Duktape/C) function call is strict or not. Returns 1 if the current function call is strict, 0 otherwise. As of Duktape 0.12.0, this function always returns 1 when called from user code (even if the call stack is empty).


### 例

```c
if (duk_is_strict_call(ctx)) {
    printf("strict call\n");
} else {
    printf("non-strict call\n");
}
```


## duk_is_string() 1.0.0stringstack§

### プロトタイプ

```c
duk_bool_t duk_is_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a string, otherwise returns 0. If idx is invalid, also returns 0.

Symbol values are visible in the C API as strings so that both duk_is_symbol() and duk_is_string() are true. This behavior is similar to Duktape 1.x internal strings. You can distinguish Symbols from ordinary strings using duk_is_symbol(). For the internal representation, see symbols.rst.

### 例

```c
if (duk_is_string(ctx, -3)) {
    /* ... */
}
```


## duk_is_symbol() 2.0.0symbolstringstack§

### プロトタイプ

```c
duk_bool_t duk_is_symbol(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is a symbol, otherwise returns 0. If idx is invalid, also returns 0.

Symbol values are visible in the C API as strings so that both duk_is_symbol() and duk_is_string() are true. This behavior is similar to Duktape 1.x internal strings. You can distinguish Symbols from ordinary strings using duk_is_symbol(). For the internal representation, see symbols.rst.

### 例

```c
if (duk_is_symbol(ctx, -3)) {
    /* ... */
}
```


## duk_is_syntax_error() 1.4.0stackerror§

### プロトタイプ

```c
duk_bool_t duk_is_syntax_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx inherits from SyntaxError, otherwise returns 0. If idx is invalid, also returns 0. This is a convenience call for using duk_get_error_code() == DUK_ERR_SYNTAX_ERROR.


### 例

```c
if (duk_is_syntax_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_thread() 1.0.0threadstack§

### プロトタイプ

```c
duk_bool_t duk_is_thread(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is an object and is a Duktape thread (coroutine), otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_thread(ctx, -3)) {
    /* ... */
}
```


## duk_is_type_error() 1.4.0stackerror§

### プロトタイプ

```c
duk_bool_t duk_is_type_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx inherits from TypeError, otherwise returns 0. If idx is invalid, also returns 0. This is a convenience call for using duk_get_error_code() == DUK_ERR_TYPE_ERROR.


### 例

```c
if (duk_is_type_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_undefined() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx is either undefined, otherwise returns 0. If idx is invalid, also returns 0.


### 例

```c
if (duk_is_undefined(ctx, -3)) {
    /* ... */
}
```


## duk_is_uri_error() 1.4.0stackerror§

### プロトタイプ

```c
duk_bool_t duk_is_uri_error(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Returns 1 if value at idx inherits from URIError, otherwise returns 0. If idx is invalid, also returns 0. This is a convenience call for using duk_get_error_code() == DUK_ERR_URI_ERROR.


### 例

```c
if (duk_is_uri_error(ctx, -3)) {
    /* ... */
}
```


## duk_is_valid_index() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_is_valid_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Validate argument index, return 1 if index is valid, 0 otherwise.


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


## duk_join() 1.0.0string§

### プロトタイプ

```c
void duk_join(duk_context *ctx, duk_idx_t count);
```

### スタック

. . . sep val1 . . . valN → . . . result

### 要約

Join zero or more values into a result string with a separator between each value. The separator and the input values are automatically coerced with ToString().

This primitive minimizes the number of intermediate string interning operations and is better than joining strings manually.


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


## duk_json_decode() 1.0.0jsoncodec§

### プロトタイプ

```c
void duk_json_decode(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . json_val . . . → . . . val . . .

### 要約

Decodes an arbitrary JSON value as an in-place operation. If the input is invalid, throws an error.


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


## duk_json_encode() 1.0.0jsoncodec§

### プロトタイプ

```c
const char *duk_json_encode(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . json_val . . .

### 要約

Encodes an arbitrary value into its JSON representation as an in-place operation. Returns pointer to the resulting string for convenience.


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


## duk_load_function() 1.3.0stackbytecode§

### プロトタイプ

```c
void duk_load_function(duk_context *ctx);
```

### スタック

. . . bytecode → . . . function

### 要約

Load a buffer containing bytecode, recreating the original ECMAScript function (with some limitations). You must ensure that the bytecode has been dumped with a compatible Duktape version and that the bytecode has not been modified since. Loading bytecode from an untrusted source is memory unsafe and may lead to exploitable vulnerabilities.

For more information on Duktape bytecode dump/load, supported features, and known limitations, see bytecode.rst. Duktape bytecode format is not intended for obfuscation, see notes on Obfuscation.

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


## duk_map_string() 1.0.0string§

### プロトタイプ

```c
void duk_map_string(duk_context *ctx, duk_idx_t idx, duk_map_char_function callback, void *udata);
```

### スタック

. . . val . . . → . . . val . . .

### 要約

Process string at idx, calling callback for each codepoint of the string. The callback is given the udata argument and a codepoint and returns a replacement codepoint. If successful, a new string consisting of the replacement codepoints replaces the original. If the value is not a string or the index is invalid, throws an error.


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


## duk_new() 1.0.0objectcall§

### プロトタイプ

```c
void duk_new(duk_context *ctx, duk_idx_t nargs);
```

### スタック

. . . constructor arg1 . . . argN → . . . retval

### 要約

Call a constructor function with nargs arguments (not counting the function itself). The function and its arguments are replaced by a single return value. An error thrown during the constructor call is not automatically caught.

The target function this binding is set to a freshly created empty object. If constructor.prototype is an object, the internal prototype of the new object is set to that value; otherwise the standard built-in Object prototype is used as the internal prototype. The return value of the target function determines what the result of the constructor call is. If the constructor returns an object, it replaces the fresh empty object; otherwise the fresh empty object (possibly modified by the constructor) is returned. See [[Construct]].


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


## duk_next() 1.0.0propertyobject§

### プロトタイプ

```c
duk_bool_t duk_next(duk_context *ctx, duk_idx_t enum_idx, duk_bool_t get_value);
```

### スタック

. . . enum . . . → . . . enum . . . (if enum empty; function returns zero)
. . . enum . . . → . . . enum . . . key (if enum not empty and get_value == 0; function returns non-zero)
. . . enum . . . → . . . enum . . . key value (if enum not empty and get_value != 0; function returns non-zero)

### 要約

Get the next key (and optionally value) from an enumerator created with duk_enum(). If the enumeration has been exhausted, nothing is pushed to the stack and the function returns zero. Otherwise the key is pushed to the stack, followed by the value if get_value is non-zero, and the function returns non-zero.

Note that getting the value may invoke a getter or trigger a Proxy trap which may have arbitrary side effects (and may throw an error).


### 例

```c
while (duk_next(ctx, enum_idx, 1)) {
    printf("key=%s, value=%s\n", duk_to_string(ctx, -2), duk_to_string(ctx, -1));
    duk_pop_2(ctx);
}
```

### 参照

duk_enum


## duk_normalize_index() 1.0.0stack§

### プロトタイプ

```c
duk_idx_t duk_normalize_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Normalize argument index relative to the bottom of the current frame. The resulting index will be 0 or greater and will be independent of later stack modifications. If the input index is invalid, returns DUK_INVALID_INDEX. If you prefer that an error be thrown for an invalid index, use duk_require_normalize_index().


### 例

```c
duk_idx_t idx = duk_normalize_index(ctx, -3);
```

### 参照

duk_require_normalize_index


## duk_opt_boolean() 2.1.0stack§

### プロトタイプ

```c
duk_bool_t duk_opt_boolean(duk_context *ctx, duk_idx_t idx, duk_bool_t def_value);
```

### スタック

. . . val . . .

### 要約

Get the boolean value at idx without modifying or coercing the value. Returns 1 if the value is true, 0 if the value is false. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.


### 例

```c
duk_bool_t flag_xyz = duk_opt_boolean(ctx, 2, 1);  /* default: true */
```


## duk_opt_buffer() 2.1.0stackbuffer§

### プロトタイプ

```c
void *duk_opt_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

. . . val . . .

### 要約

Get the data pointer for a (plain) buffer value at idx without modifying or coercing the value. Returns a non-NULL pointer if the value is a valid buffer with a non-zero size. For a zero-size buffer, may return a NULL or a non-NULL pointer. If out_size is non-NULL, the size of the buffer is written to *out_size. If the value is undefined or the index is invalid, def_ptr default value is returned and the def_len default length is written to *out_size (if out_size is non-NULL). In other cases (null and non-matching type) throws an error.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.
There is no reliable way to distinguish a zero-size buffer from a non-buffer based on the return values alone: a NULL with zero size is returned for a non-buffer. The same values may be returned for a zero-size buffer (although it is also possible that a non-NULL pointer is returned). Use duk_is_buffer() or duk_is_buffer_data() or when type checking a buffer or a buffer object.

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


## duk_opt_buffer_data() 2.1.0stackbufferobjectbuffer§

### プロトタイプ

```c
void *duk_opt_buffer_data(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size, void *def_ptr, duk_size_t def_len);
```

### スタック

. . . val . . .

### 要約

Get the data pointer for a plain buffer or a buffer object (ArrayBuffer, Node.js Buffer, DataView, or TypedArray view) value at idx without modifying or coercing the value. Return a non-NULL pointer if the value is a valid buffer with a non-zero size. For a zero-size buffer, may return a NULL or a non-NULL pointer. If out_size is non-NULL, the size of the buffer is written to *out_size. If the value is undefined or the index is invalid, def_ptr default value is returned and the def_len default length is written to *out_size (if out_size is non-NULL). In other cases (null and non-matching type) throws an error. Also throws if the value is a buffer object whose "backing buffer" doesn't fully cover the buffer object's apparent size.

The data area indicated by the return pointer and length is the full buffer for a plain buffer value, and the active "slice" for a buffer object. The length returned is expressed in bytes (instead of elements), so that you can always access ptr[0] to ptr[len - 1]. See duk_get_buffer_data() for examples.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

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


## duk_opt_c_function() 2.1.0stackfunction§

### プロトタイプ

```c
duk_c_function duk_opt_c_function(duk_context *ctx, duk_idx_t idx, duk_c_function def_value);
```

### スタック

. . . val . . .

### 要約

Get the Duktape/C function pointer (a duk_c_function) from an ECMAScript function object associated with a Duktape/C function. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.


### 例

```c
duk_c_function funcptr;

/* Native callback, default to nop_callback. */
funcptr = duk_opt_c_function(ctx, -3, nop_callback);
```


## duk_opt_context() 2.1.0stackborrowed§

### プロトタイプ

```c
duk_context *duk_opt_context(duk_context *ctx, duk_idx_t idx, duk_context *def_value);
```

### スタック

. . . val . . .

### 要約

Get a context pointer for a Duktape thread at idx. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

### 例

```c
duk_context *target_ctx;

target_ctx = duk_opt_context(ctx, 2, default_ctx);
```


## duk_opt_heapptr() 2.1.0stackheapptrborrowed§

### プロトタイプ

```c
void *duk_opt_heapptr(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

. . . val . . .

### 要約

Get a borrowed void * reference to a Duktape heap allocated value (object, buffer, string) at idx. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.

Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

### 例

```c
void *ptr;

ptr = duk_opt_heapptr(ctx, 2, default_ptr);
```


## duk_opt_int() 2.1.0stack§

### プロトタイプ

```c
duk_int_t duk_opt_int(duk_context *ctx, duk_idx_t idx, duk_int_t def_value);
```

### スタック

. . . val . . .

### 要約

Get the number at idx and convert it to a C duk_int_t by first clamping the value between [DUK_INT_MIN, DUK_INT_MAX] and then truncating towards zero. The value on the stack is not modified. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.


### 例

```c
int port = (int) duk_opt_int(ctx, 1, 80);  /* default: 80 */
```


## duk_opt_lstring() 2.1.0stringstack§

### プロトタイプ

```c
const char *duk_opt_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len, const char *def_ptr, duk_size_t def_len);
```

### スタック

. . . val . . .

### 要約

Get character data pointer and length for a string at idx without modifying or coercing the value. Returns a non-NULL pointer to the read-only, NUL-terminated string data, and writes the string byte length to *out_len (if out_len is non-NULL). If the value is undefined or the index is invalid, def_ptr default value is returned and the def_len default length is written to *out_len (if out_len is non-NULL). In other cases (null or non-matching type) throws an error.

A non-NULL return value is guaranteed even for zero length strings; this differs from how buffer data pointers are handled (for technical reasons).
Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.

### 例

```c
const char *str;
duk_size_t len;

str = duk_opt_lstring(ctx, -3, &len, "foo" "\x00" "bar", 7);
```

### 参照

duk_opt_string


## duk_opt_number() 2.1.0stack§

### プロトタイプ

```c
duk_double_t duk_opt_number(duk_context *ctx, duk_idx_t idx, duk_double_t def_value);
```

### スタック

. . . val . . .

### 要約

Get the number value at idx without modifying or coercing the value. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.


### 例

```c
double backoff_multiplier = (double) duk_opt_number(ctx, 2, 1.5);  /* default: 1.5 */
```


## duk_opt_pointer() 2.1.0stack§

### プロトタイプ

```c
void *duk_opt_pointer(duk_context *ctx, duk_idx_t idx, void *def_value);
```

### スタック

. . . val . . .

### 要約

Get the pointer value at idx as void * without modifying or coercing the value. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.


### 例

```c
void *ptr;

ptr = duk_opt_pointer(ctx, -3, (void *) 0x12345678);
printf("my pointer is: %p\n", ptr);
```


## duk_opt_string() 2.1.0stringstack§

### プロトタイプ

```c
const char *duk_opt_string(duk_context *ctx, duk_idx_t idx, const char *def_ptr);
```

### スタック

. . . val . . .

### 要約

Get character data pointer for a string at idx without modifying or coercing the value. Returns a non-NULL pointer to the read-only, NUL-terminated string data. If the value is undefined or the index is invalid, def_ptr default value is returned. In other cases (null or non-matching type) throws an error.

To get the string byte length explicitly (which is useful if the string contains embedded NUL characters), use duk_opt_lstring().

A non-NULL return value is guaranteed even for zero length strings; this differs from how buffer data pointers are handled (for technical reasons).
Default pointer values given to duk_opt_xxx() and duk_get_xxx_default() are not tracked by Duktape, e.g. duk_opt_string() does not make a copy of the default string argument. The caller is responsible for ensuring that the default pointer remains valid for its intended use. For example, duk_opt_string(ctx, 3, "localhost") works fine because a string constant is always valid, but if the argument is a libc allocated string, caller must ensure the pointer returned from duk_opt_string() is not used beyond the lifetime of the libc allocated string.
Symbol values are visible in the C API as strings so that both duk_is_symbol() and duk_is_string() are true. This behavior is similar to Duktape 1.x internal strings. You can distinguish Symbols from ordinary strings using duk_is_symbol(). For the internal representation, see symbols.rst.

### 例

```c
const char *host = duk_opt_string(ctx, 3, "localhost");
```

### 参照

duk_opt_lstring


## duk_opt_uint() 2.1.0stack§

### プロトタイプ

```c
duk_uint_t duk_opt_uint(duk_context *ctx, duk_idx_t idx, duk_uint_t def_value);
```

### スタック

. . . val . . .

### 要約

Get the number at idx and convert it to a C duk_uint_t by first clamping the value between [0, DUK_UINT_MAX] and then truncating towards zero. The value on the stack is not modified. If the value is undefined or the index is invalid, def_value default value is returned. In other cases (null or non-matching type) throws an error.


### 例

```c
unsigned int count = (unsigned int) duk_opt_uint(ctx, 1, 3);  /* default: 3 */
```


## duk_pcall() 1.0.0protectedcall§

### プロトタイプ

```c
duk_int_t duk_pcall(duk_context *ctx, duk_idx_t nargs);
```

### スタック

. . . func arg1 . . . argN → . . . retval (if success, return value == 0)
. . . func arg1 . . . argN → . . . err (if failure, return value != 0)

### 要約

Call target function func with nargs arguments (not counting the function itself). The function and its arguments are replaced by a single return value or a single error value. An error thrown during the function call is caught.

The return value is:

DUK_EXEC_SUCCESS (0): call succeeded, nargs arguments are replaced with a single return value. (This return code constant is guaranteed to be zero, so that one can check for success with a "zero or non-zero" check.)
DUK_EXEC_ERROR: call failed, nargs arguments are replaced with a single error value. (In exceptional cases, e.g. when there are too few arguments on the value stack, the call may throw.)
Unlike most Duktape API calls, this call returns zero on success. This allows multiple error codes to be defined later.
Error objects caught are typically instances of Error and have useful properties like .stack, .fileName, and .lineNumber. These can be accessed using the normal property methods. However, arbitrary values can be thrown so you should avoid assuming that's always the case.

The target function this binding is initially set to undefined. If the target function is not strict, the binding is replaced by the global object before the function is invoked; see Entering Function Code. If you want to control the this binding, you can use duk_pcall_method() or duk_pcall_prop() instead.


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


## duk_pcall_method() 1.0.0protectedcall§

### プロトタイプ

```c
duk_int_t duk_pcall_method(duk_context *ctx, duk_idx_t nargs);
```

### スタック

. . . func this arg . . . argN → . . . retval (if success, return value == 0)
. . . func this arg . . . argN → . . . err (if failure, return value != 0)

### 要約

Like duk_pcall(), but the target function func is called with an explicit this binding given on the value stack.


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


## duk_pcall_prop() 1.0.0protectedpropertycall§

### プロトタイプ

```c
duk_int_t duk_pcall_prop(duk_context *ctx, duk_idx_t obj_idx, duk_idx_t nargs);
```

### スタック

. . . obj . . . key arg1 . . . argN → . . . obj . . . retval (if success, return value == 0)
. . . obj . . . key arg1 . . . argN → . . . obj . . . err (if failure, return value != 0)

### 要約

Like duk_pcall(), but the target function is looked up from obj.key and obj is used as the function's this binding.


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


## duk_pcompile() 1.0.0protectedcompile§

### プロトタイプ

```c
duk_int_t duk_pcompile(duk_context *ctx, duk_uint_t flags);
```

### スタック

. . . source filename → . . . function (if success, return value == 0)
. . . source filename → . . . err (if failure, return value != 0)

### 要約

Like duk_compile() but catches errors related to compilation (such as syntax errors in the source). A zero return value indicates success and the compiled function is left on the stack top. A non-zero return value indicates an error, and the error is left on the stack top.

If value stack top is too low (smaller than 2), an error is thrown.


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


## duk_pcompile_lstring() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_pcompile_lstring(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

. . . → . . . function (if success, return value == 0)
. . . → . . . err (if failure, return value != 0)

### 要約

Like duk_pcompile(), but the compile input is given as a C string with explicit length. The filename associated with the function is "input".

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_pcompile_lstring_filename() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_pcompile_lstring_filename(duk_context *ctx, duk_uint_t flags, const char *src, duk_size_t len);
```

### スタック

. . . filename → . . . function (if success, return value == 0)
. . . filename → . . . err (if failure, return value != 0)

### 要約

Like duk_pcompile(), but the compile input is given as a C string with explicit length.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_pcompile_string() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_pcompile_string(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

. . . → . . . function (if success, return value == 0)
. . . → . . . err (if failure, return value != 0)

### 要約

Like duk_pcompile(), but the compile input is given as a C string. The filename associated with the function is "input".

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_pcompile_string_filename() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_pcompile_string_filename(duk_context *ctx, duk_uint_t flags, const char *src);
```

### スタック

. . . filename → . . . function (if success, return value == 0)
. . . filename → . . . err (if failure, return value != 0)

### 要約

Like duk_pcompile(), but the compile input is given as a C string.

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_peval() 1.0.0protectedcompile§

### プロトタイプ

```c
duk_int_t duk_peval(duk_context *ctx);
```

### スタック

. . . source → . . . result (if success, return value == 0)
. . . source → . . . err (if failure, return value != 0)

### 要約

Like duk_eval() but catches errors related to compilation (such as syntax errors in the source). A zero return value indicates success and the eval result is left on the stack top. A non-zero return value indicates an error, and the error is left on the stack top.

If value stack top is too low (smaller than 1), an error is thrown.


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


## duk_peval_lstring() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_peval_lstring(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

. . . → . . . result (if success, return value == 0)
. . . → . . . err (if failure, return value != 0)

### 要約

Like duk_peval(), but the eval input is given as a C string with explicit length. The filename associated with the temporary eval function is "eval".

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_peval_lstring_noresult() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_peval_lstring_noresult(duk_context *ctx, const char *src, duk_size_t len);
```

### スタック

. . . → . . .

### 要約

Like duk_peval_lstring(), but leaves no result on the value stack (regardless of success/error result).

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_peval_noresult() 1.0.0protectedcompile§

### プロトタイプ

```c
duk_int_t duk_peval_noresult(duk_context *ctx);
```

### スタック

. . . source → . . .

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


## duk_peval_string() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_peval_string(duk_context *ctx, const char *src);
```

### スタック

. . . → . . . result (if success, return value == 0)
. . . → . . . err (if failure, return value != 0)

### 要約

Like duk_peval(), but the eval input is given as a C string. The filename associated with the temporary eval function is "eval".

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

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


## duk_peval_string_noresult() 1.0.0stringprotectedcompile§

### プロトタイプ

```c
duk_int_t duk_peval_string_noresult(duk_context *ctx, const char *src);
```

### スタック

. . . → . . .

### 要約

Like duk_peval_string(), but leaves no result on the value stack (regardless of success/error result).

With this variant, the input source code is not interned by Duktape which is useful in low memory environments.

### 例

```c
if (duk_peval_string_noresult(ctx, "print('testString'.toUpperCase());") != 0) {
    printf("eval failed\n");
} else {
    printf("eval successful\n");
}
```


## duk_pnew() 1.3.0protectedobjectcall§

### プロトタイプ

```c
duk_ret_t duk_pnew(duk_context *ctx, duk_idx_t nargs);
```

### スタック

. . . constructor arg1 . . . argN → . . . retval (if success, return value == 0)
. . . constructor arg1 . . . argN → . . . err (if failure, return value != 0)

### 要約

Like duk_new() but catches errors. A zero return value indicates success and the constructor result is left on the stack top. A non-zero return value indicates an error, and the error is left on the stack top.

If value stack top is too low (smaller than nargs + 1), or nargs is negative, an error is thrown.


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


## duk_pop() 1.0.0stack§

### プロトタイプ

```c
void duk_pop(duk_context *ctx);
```

### スタック

. . . val → . . .

### 要約

Pop one element off the stack. If the stack is empty, throws an error.

To pop multiple elements, use duk_pop_n() or the shortcuts for common cases: duk_pop_2() and duk_pop_3().


### 例

```c
duk_pop(ctx);
```

### 参照

duk_pop_2
duk_pop_3
duk_pop_n


## duk_pop_2() 1.0.0stack§

### プロトタイプ

```c
void duk_pop_2(duk_context *ctx);
```

### スタック

. . . val1 val2 → . . .

### 要約

Pop two elements off the stack. If the stack has fewer than two elements, throws an error.


### 例

```c
duk_pop_2(ctx);
```


## duk_pop_3() 1.0.0stack§

### プロトタイプ

```c
void duk_pop_3(duk_context *ctx);
```

### スタック

. . . val1 val2 val3 → . . .

### 要約

Pop three elements off the stack. If the stack has fewer than three elements, throws an error.


### 例

```c
duk_pop_3(ctx);
```


## duk_pop_n() 1.0.0stack§

### プロトタイプ

```c
void duk_pop_n(duk_context *ctx, duk_idx_t count);
```

### スタック

. . . val1 . . . valN → . . .

### 要約

Pop count elements off the stack. If the stack has fewer than count elements, throws an error. If count is zero, the call is a no-op. Negative counts cause an error to be thrown.


### 例

```c
duk_pop_n(ctx, 10);
```


## duk_pull() 2.5.0stack§

### プロトタイプ

```c
void duk_pull(duk_context *ctx, duk_idx_t from_idx);
```

### スタック

. . . val . . . → . . . . . . val

### 要約

Remove value at from_idx and push it on the value stack top.

If from_idx is an invalid index, throws an error.

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


## duk_push_array() 1.0.0stackobject§

### プロトタイプ

```c
duk_idx_t duk_push_array(duk_context *ctx);
```

### スタック

. . . → . . . arr

### 要約

Push an empty array to the stack. Returns non-negative index (relative to stack bottom) of the pushed array.

The internal prototype of the created object is Array.prototype. Use duk_set_prototype() to change it.


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


## duk_push_bare_array() 2.4.0stackobject§

### プロトタイプ

```c
duk_idx_t duk_push_bare_array(duk_context *ctx);
```

### スタック

. . . → . . . arr

### 要約

Similar to duk_push_array() but the pushed array doesn't inherit from any other object, i.e. its internal prototype is null. Returns non-negative index (relative to stack bottom) of the pushed array.


### 例

```c
duk_idx_t arr_idx;

arr_idx = duk_push_bare_array(ctx);
```

### 参照

duk_push_array
duk_push_bare_object


## duk_push_bare_object() 2.0.0stackobject§

### プロトタイプ

```c
duk_idx_t duk_push_bare_object(duk_context *ctx);
```

### スタック

. . . → . . . obj

### 要約

Similar to duk_push_object() but the pushed object doesn't inherit from any other object, i.e. its internal prototype is null. This call is equivalent to Object.create(null). Returns non-negative index (relative to stack bottom) of the pushed object.


### 例

```c
duk_idx_t obj_idx;

obj_idx = duk_push_bare_object(ctx);
```

### 参照

duk_push_object
duk_push_bare_array


## duk_push_boolean() 1.0.0stack§

### プロトタイプ

```c
void duk_push_boolean(duk_context *ctx, duk_bool_t val);
```

### スタック

. . . → . . . true (if val != 0)
. . . → . . . false (if val == 0)

### 要約

Push true (if val != 0) or false (if val == 0) to the stack.


### 例

```c
duk_push_boolean(ctx, 0);  /* -> [ ... false ] */
duk_push_boolean(ctx, 1);  /* -> [ ... false true ] */
duk_push_boolean(ctx, 123);  /* -> [ ... false true true ] */
```


## duk_push_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_push_buffer(duk_context *ctx, duk_size_t size, duk_bool_t dynamic);
```

### スタック

. . . → . . . buf

### 要約

Allocate a new buffer of size bytes and push it to the value stack. Returns a non-NULL pointer to the buffer data area; for a zero-size buffer, may return either NULL or non-NULL. The buffer data area is automatically zeroed. If dynamic is non-zero, the buffer will be resizable, otherwise the buffer will have a fixed size. Throws an error if allocation fails.

There are also shortcut variants duk_push_fixed_buffer() and duk_push_dynamic_buffer().

A dynamic buffer requires two memory allocations internally: one for the buffer header and another for the currently allocated data area. A fixed buffer only requires a single allocation: the data area follows the buffer header.
Be careful when requesting a zero length dynamic buffer: a NULL data pointer is not an error and should not confuse calling code.
Duktape can be compiled with a config option which disables automatic zeroing of allocated buffer data (zeroing is the default). If this is the case, you need to zero the buffer manually if necessary.

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


## duk_push_buffer_object() 1.3.0stackbufferobject§

### プロトタイプ

```c
void duk_push_buffer_object(duk_context *ctx, duk_idx_t idx_buffer, duk_size_t byte_offset, duk_size_t byte_length, duk_uint_t flags);
```

### スタック

. . . buffer . . . → . . . buffer . . . bufobj (when creating an ArrayBuffer or a view)
. . . ArrayBuffer . . . → . . . ArrayBuffer . . . bufobj (when creating a view)

### 要約

Push a new buffer object or a buffer view object. An underlying plain buffer or an ArrayBuffer (accepted when creating a view) is provided at index idx_buffer. The type of the buffer or view is given in flags (e.g. DUK_BUFOBJ_UINT16ARRAY). The active range or "slice" used from the underlying buffer is indicated by byte_offset and byte_length.

The available buffer types are:

Define	Buffer/view type
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
When anything other than an ArrayBuffer is created and the backing buffer given as an argument is a plain buffer, an ArrayBuffer backing the view is automatically created. It is accessible through the buffer property of the view object. The ArrayBuffer's internal byteOffset will be zero so that the ArrayBuffer's index byteOffset matches the view's index 0. The ArrayBuffer's byteLength will be byte_offset + byte_length so that the view range is valid for the ArrayBuffer.

When creating an ArrayBuffer, it's strongly recommended that the byte_offset argument be zero. Otherwise the external .byteOffset property of any views constructed over the ArrayBuffer will be misleading: the value will be relative to the plain buffer underlying the ArrayBuffer, rather than relative to the ArrayBuffer. For a zero byte_offset there is no difference between the two offsets.

The underlying plain buffer should normally cover the range indicated by the byte_offset and byte_length arguments, but memory safety is guaranteed even if that is not the case. For example, an attempt to read values outside the underlying buffer will return zero. The underlying buffer size is intentionally not checked when creating a buffer object: even if the buffer fully covered the byte range during creation, it might be resized later.


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


## duk_push_c_function() 1.0.0stackfunction§

### プロトタイプ

```c
duk_idx_t duk_push_c_function(duk_context *ctx, duk_c_function func, duk_idx_t nargs);
```

### スタック

. . . → . . . func

### 要約

Push a new function object, associated with a C function, to the stack. The function object is an ECMAScript function object; when called, func will be called using the Duktape/C function interface. Returns non-negative index (relative to stack bottom) of the pushed function.

The nargs argument controls how the value stack looks like when func is entered:

If nargs is >= 0, it indicates the exact number of arguments the function expects to see; extra arguments are discarded and missing arguments are filled in with undefined values. Upon entry to the function, value stack top will always match nargs.
If nargs is set to DUK_VARARGS, the value stack will contain actual (variable) call arguments and the function needs to check actual argument count with duk_get_top().
The function created will be callable both as a normal function (func()) and as a constructor (new func()). You can differentiate between the two call styles using duk_is_constructor_call(). Although the function can be used as a constructor, it doesn't have an automatic prototype property like ECMAScript functions.

If you intend to use the pushed function as a constructor, you should usually create a prototype object and set the prototype property of the function manually.

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


## duk_push_c_lightfunc() stacklightfuncfunction§

### プロトタイプ

```c
duk_idx_t duk_push_c_lightfunc(duk_context *ctx, duk_c_function func, duk_idx_t nargs, duk_idx_t length, duk_int_t magic);
```

### スタック

. . . → . . . func

### 要約

Push a new lightfunc value, associated with a C function, to the stack. Returns non-negative index (relative to stack bottom) of the pushed lightfunc.

A lightfunc is a tagged value which contains a Duktape/C function pointer and a small set of internal control flags with no related heap allocations. The internal control flags encode the nargs, length, and magic values, which therefore have significant restrictions:

nargs must be [0,14] or DUK_VARARGS.
length must be [0,15] and maps to the virtual length property of the lightfunc.
magic must be [-128,127].
A lightfunc cannot hold any own properties, it only has virtual name and length properties, and inherits further properties from Function.prototype.

The nargs argument controls how the value stack looks like when func is entered, and behaves like for ordinary Duktape/C functions, see duk_push_c_function().

The function created will be callable both as a normal function (func()) and as a constructor (new func()). You can differentiate between the two call styles using duk_is_constructor_call(). Although the function can be used as a constructor, it cannot have a prototype property like normal Function objects.

If you intend to use the pushed lightfunc as a constructor, and want to use a custom prototype object (instead of Object.prototype), the lightfunc must return an object value. The object will then replace the default instance (bound to this) automatically created for the constructor, and will be the value of a new MyLightFunc() expression.

### 例

```c
duk_idx_t func_idx;

func_idx = duk_push_c_lightfunc(ctx, my_addtwo, 2 /*nargs*/, 2 /*length*/, 0 /*magic*/);
```

### 参照

duk_push_c_function


## duk_push_context_dump() 1.0.0stackdebug§

### プロトタイプ

```c
void duk_push_context_dump(duk_context *ctx);
```

### スタック

. . . → . . . str

### 要約

Push a one-line string summarizing the state of the current activation of context ctx. This is useful for debugging Duktape/C code and is not intended for production use.

The exact dump contents are version specific. The current format includes the stack top (i.e. number of elements on the stack) and prints out the current elements as an array of JX-formatted (Duktape's custom extended JSON format) values. The example below would print something like:

ctx: top=2, stack=[123,"foo"]
You should not leave dump calls in production code.

### 例

```c
duk_push_int(ctx, 123);
duk_push_string(ctx, "foo");
duk_push_context_dump(ctx);
printf("%s\n", duk_to_string(ctx, -1));
duk_pop(ctx);
```


## duk_push_current_function() 1.0.0stackfunction§

### プロトタイプ

```c
void duk_push_current_function(duk_context *ctx);
```

### スタック

. . . → . . . func (if current function exists)
. . . → . . . undefined (if no current function)

### 要約

Push the currently running function to the stack. The value pushed is an ECMAScript Function object. If there is no current function, undefined is pushed instead.

If the current function was called via one or more bound functions or a Proxy object, the function returned from this call is the final, resolved function (not the bound function or the Proxy).

This function allows a C function to gain access to its function object. Since multiple function objects can internally point to the same C function, a function object is a convenient place for function parameterization and can also act as an internal state stash.


### 例

```c
duk_push_current_function(ctx);
```


## duk_push_current_thread() 1.0.0threadstackfunctionborrowed§

### プロトタイプ

```c
void duk_push_current_thread(duk_context *ctx);
```

### スタック

. . . → . . . thread (if current thread exists)
. . . → . . . undefined (if no current thread)

### 要約

Push the currently running Duktape thread to the stack. The value pushed is a thread object which is also an ECMAScript object. If there is no current thread, undefined is pushed instead.

The current thread is (almost always) the thread represented by the ctx pointer.

To get the duk_context * associated with the thread, use duk_get_context().


### 例

```c
duk_push_thread(ctx);
```


## duk_push_dynamic_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_push_dynamic_buffer(duk_context *ctx, duk_size_t size);
```

### スタック

. . . → . . . buf

### 要約

Allocate a dynamic (resizable) buffer and push it to the value stack. Shortcut for duk_push_buffer() with dynamic = 1.


### 例

```c
void *p;

p = duk_push_dynamic_buffer(ctx, 1024);
printf("allocated buffer, data area: %p\n", p);
```


## duk_push_error_object() 1.0.0stackobjecterror§

### プロトタイプ

```c
duk_idx_t duk_push_error_object(duk_context *ctx, duk_errcode_t err_code, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Create a new error object and push it to the value stack (the error is not thrown). Returns non-negative index (relative to stack bottom) of the pushed error object.

The message property of the error object will be set to a sprintf-formatted string using fmt and the remaining arguments. The internal prototype for the created error object is chosen based on err_code. For instance, DUK_ERR_RANGE_ERROR causes the built-in RangeError prototype to be used. The valid range for user error codes is [1,16777215].


### 例

```c
duk_idx_t err_idx;

err_idx = duk_push_error_object(ctx, DUK_ERR_TYPE_ERROR, "invalid argument value: %d", arg_value);
```

### 参照

duk_push_error_object_va


## duk_push_error_object_va() 1.1.0varargstackobjecterror§

### プロトタイプ

```c
duk_idx_t duk_push_error_object_va(duk_context *ctx, duk_errcode_t err_code, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Vararg variant of duk_push_error_object().

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_push_external_buffer() 1.3.0stackbuffer§

### プロトタイプ

```c
void duk_push_external_buffer(duk_context *ctx);
```

### スタック

. . . → . . . buf

### 要約

Allocate an external buffer and push it to the value stack. An external buffer refers to a user allocated external buffer which is not memory managed by Duktape. The initial external buffer pointer is NULL and size is zero. You can use duk_config_buffer() to update the external buffer pointer and size.

External buffers are useful to allow ECMAScript code access externally allocated data structures. You can use it to e.g. write bytes to an externally allocated frame buffer. The external buffer has no alignment requirements (Duktape makes no alignment assumptions when accessing it).


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


## duk_push_false() 1.0.0stack§

### プロトタイプ

```c
void duk_push_false(duk_context *ctx);
```

### スタック

. . . → . . . false

### 要約

Push false to stack. Equivalent to calling duk_push_boolean(ctx, 0).


### 例

```c
duk_push_false(ctx);
```


## duk_push_fixed_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_push_fixed_buffer(duk_context *ctx, duk_size_t size);
```

### スタック

. . . → . . . buf

### 要約

Allocate a fixed size buffer and push it to the value stack. Shortcut for duk_push_buffer() with dynamic = 0.


### 例

```c
void *p;

p = duk_push_fixed_buffer(ctx, 1024);
printf("allocated buffer, data area: %p\n", p);
```


## duk_push_global_object() 1.0.0stackobject§

### プロトタイプ

```c
void duk_push_global_object(duk_context *ctx);
```

### スタック

. . . → . . . global

### 要約

Push the global object to the stack.


### 例

```c
duk_push_global_object(ctx);
```


## duk_push_global_stash() 1.0.0stashstacksandboxobjectmodule§

### プロトタイプ

```c
void duk_push_global_stash(duk_context *ctx);
```

### スタック

. . . → . . . stash

### 要約

Push the global stash object to the stack. The global stash is an internal object which can be used to store key/value pairs from C code so that they are reachable for garbage collection, but are not accessible from ECMAScript code. The stash is only accessible from C code with a ctx argument associated with the same global object.


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


## duk_push_heap_stash() 1.0.0stashstacksandboxobjectmodule§

### プロトタイプ

```c
void duk_push_heap_stash(duk_context *ctx);
```

### スタック

. . . → . . . stash

### 要約

Push the heap stash object to the stack. The heap stash is an internal object which can be used to store key/value pairs from C code so that they are reachable for garbage collection, but are not accessible from ECMAScript code. The stash is only accessible from C code; the same stash object is used by all code sharing the same Duktape heap (even if they don't share the same global object).


### 例

```c
duk_push_heap_stash(ctx);
```

### 参照

duk_push_global_stash
duk_push_thread_stash


## duk_push_heapptr() 1.1.0stackobjectheapptrborrowed§

### プロトタイプ

```c
duk_idx_t duk_push_heapptr(duk_context *ctx, void *ptr);
```

### スタック

. . . → . . . obj (if ptr != NULL)
. . . → . . . undefined (if ptr == NULL)

### 要約

Push a Duktape heap object into the value stack using a borrowed pointer reference from duk_get_heapptr() or its variants. If ptr is NULL, undefined is pushed.

The caller is responsible for ensuring that the argument ptr is still valid (not freed by Duktape garbage collection) when it is pushed. Failure to do so results in unpredictable and memory unsafe behavior.

There are two basic ways to ensure this:

Strong backing reference: ensure that the related heap object is always reachable for Duktape garbage collection between duk_get_heapptr() and duk_push_heapptr(). For example, ensure that the related object has been written to a stash while the borrowed void pointer is being used. In essence, the application is holding a borrowed reference which is backed by a strongly referenced value. This was the only supported approach before Duktape 2.1.
Weak reference + finalizer: add a (preferably native) finalizer for the related heap object and stop using the void pointer at the latest when the finalizer is called (assuming the object is not rescued by the finalizer). Starting from Duktape 2.1 duk_push_heapptr() is allowed for unreachable objects pending finalization, all the way up to the actual finalizer call; the object will be rescued and the finalizer call automatically cancelled. This approach allows an application to hold weak references to the related objects. However, see limitations below.
Finalizer calls may silently fail e.g. due to out-of-memory. When relying on finalizer calls to indicate an end of pointer validity, a missed finalizer call may cause a dangling pointer to be given to duk_push_heapptr(). There's currently no workaround for this situation, so if out-of-memory conditions are to be expected, the finalizer-based approach may not work reliably. Future work is to ensure that an object is not freed without at least a successful entry into a native finalizer function, see https://github.com/svaarala/duktape/issues/1456.

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


## duk_push_int() 1.0.0stack§

### プロトタイプ

```c
void duk_push_int(duk_context *ctx, duk_int_t val);
```

### スタック

. . . → . . . val

### 要約

Convert val to an IEEE double and push it to the stack.

This is a shorthand for calling duk_push_number(ctx, (duk_double_t) val).


### 例

```c
duk_push_int(ctx, 123);
```


## duk_push_literal() 2.3.0stackliteral§

### プロトタイプ

```c
const char *duk_push_literal(duk_context *ctx, const char *str_literal);
```

### スタック

. . . → . . . str

### 要約

Push a C literal into the stack and return a pointer to the interned string data area (which may or may not be the same as the argument literal). The argument str_literal:

must be a non-NULL C literal like "foo" which can be operated on using e.g. sizeof(str), where sizeof() yields string length, including the automatic NUL terminator (for example, sizeof("foo") is 4);
must have immutable data contents, i.e. read-only memory, or writable memory but guaranteed not to be changed; and
must not contain internal NUL characters, and must have a terminating NUL as usual for C strings.
The str_literal argument may be evaluated multiple times by the API macro. No runtime NULL pointer check is made for the str_literal argument so passing in a NULL causes memory unsafe behavior.

This call is conceptually equivalent to duk_push_string(). Calling code only needs to use it if the minor differences in footprint or speed matter. The properties of immutable C literals allows minor optimizations in Duktape internals:

The length of the string can be computed at compile time using sizeof(str_literal) - 1 at the call site.
By default heap strings accessed via C literals (duk_push_literal() or any of the literal convenience calls like duk_get_prop_literal()) are automatically pinned until the next mark-and-sweep round, and there's a lookup cache for mapping a C literal address to the pinned internal heap string. This optimization doesn't assume string deduplication (which is common but not guaranteed), i.e. there may be multiple addresses with the same literal data.
Because the string data is assumed to be immutable, the internal string representation could just point to the data instead of making a copy. (This optimization is not done as of Duktape 2.5.)
If input string might contain internal NUL characters, use duk_push_lstring() instead. For duk_push_literal() handling of embedded NULs depends on config options and calling code should never rely on the behavior.


### 例

```c
/* Basic case. */
duk_push_literal(ctx, "foo");

/* Argument may involve compile time concatenation and parentheses. */
duk_push_literal(ctx, ("foo" "bar"));

/* Argument may also be e.g. DUK_HIDDEN_SYMBOL() which produces a literal. */
duk_push_literal(ctx, DUK_HIDDEN_SYMBOL("mySymbol"));
```


## duk_push_lstring() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_push_lstring(duk_context *ctx, const char *str, duk_size_t len);
```

### スタック

. . . → . . . str

### 要約

Push a string of explicit length to the stack. The string may contain arbitrary data, including internal NUL characters. A pointer to the interned string data is returned. If the operation fails, throws an error.

If str is NULL, an empty string is pushed to the stack (regardless of what len is) and a non-NULL pointer to an empty string is returned. The returned pointer can be dereferenced and a NUL terminator character is guaranteed. This behavior differs from duk_push_string on purpose.

C code should normally only push valid CESU-8 strings to the stack.


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


## duk_push_nan() 1.0.0stack§

### プロトタイプ

```c
void duk_push_nan(duk_context *ctx);
```

### スタック

. . . → . . . NaN

### 要約

Pushes a NaN (not-a-number) to the stack.


### 例

```c
duk_push_nan(ctx);

printf("NaN is number: %d\n", (int) duk_is_number(ctx, -1));
```


## duk_push_new_target() 2.3.0stackfunction§

### プロトタイプ

```c
void duk_push_new_target(duk_context *ctx);
```

### スタック

. . . → . . . undefined (if no current function or not a constructor call)
. . . → . . . func (if current function call is a constructor call)

### 要約

Push the equivalent of new.target for the currently running function to the stack. The value pushed is undefined if the current call is not a constructor call, or if the call stack is empty.


### 例

```c
duk_push_new_target(ctx);
```


## duk_push_null() 1.0.0stack§

### プロトタイプ

```c
void duk_push_null(duk_context *ctx);
```

### スタック

. . . → . . . null

### 要約

Push null to the stack.


### 例

```c
duk_push_null(ctx);
```


## duk_push_number() 1.0.0stack§

### プロトタイプ

```c
void duk_push_number(duk_context *ctx, duk_double_t val);
```

### スタック

. . . → . . . val

### 要約

Push number (IEEE double) val to the stack.

If val is a NaN it may be normalized into another NaN form.


### 例

```c
duk_push_number(ctx, 123.0);
```


## duk_push_object() 1.0.0stackobject§

### プロトタイプ

```c
duk_idx_t duk_push_object(duk_context *ctx);
```

### スタック

. . . → . . . obj

### 要約

Push an empty object to the stack. Returns non-negative index (relative to stack bottom) of the pushed object.

The internal prototype of the created object is Object.prototype. Use duk_set_prototype() to change it.


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


## duk_push_pointer() 1.0.0stack§

### プロトタイプ

```c
void duk_push_pointer(duk_context *ctx, void *p);
```

### スタック

. . . → . . . ptr

### 要約

Push p into the stack as a pointer value. Duktape won't interpret the pointer in any manner.


### 例

```c
struct mystruct *p = /* ... */;

duk_push_pointer(ctx, (void *) p);
```


## duk_push_proxy() 2.2.0stackobject§

### プロトタイプ

```c
duk_idx_t duk_push_proxy(duk_context *ctx, duk_uint_t proxy_flags);
```

### スタック

. . . target handler → . . . proxy

### 要約

Push a new Proxy object for target and handler table given on the value stack, equivalent to new Proxy(target, handler). The proxy_flags argument is currently (up to Duktape 2.5) unused, calling code must pass in a zero.


### 例

```c
duk_idx_t proxy_idx;

duk_push_object(ctx);  /* target */
duk_push_object(ctx);  /* handler */
duk_push_c_function(ctx, my_get, 3);  /* 'get' trap */
duk_put_prop_string(ctx, -2, "get");
proxy_idx = duk_push_proxy(ctx, 0);  /* [ target handler ] -> [ proxy ] */
```


## duk_push_sprintf() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_push_sprintf(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . str

### 要約

Format a string like sprintf() (but safely) and push the result to the value stack. Returns a non-NULL pointer to the resulting string.

If fmt is NULL, an empty string is pushed to the stack and a non-NULL pointer to an empty string is returned (this behavior mimics what sprintf() does for a NULL format string, at least on Linux). The returned pointer can be dereferenced and a NUL terminator character is guaranteed.

Unlike sprintf() the string formatting is safe with respect to result length. Concretely, the implementation will try increasing temporary buffer sizes until a large enough buffer is found for the temporary formatted value.

There may be platform specific behavior for some format specifiers. For example, the behavior of %s with a NULL argument has officially undefined behavior, so actual behavior may vary across platforms and may even be memory unsafe.

### 例

```c
duk_push_sprintf(ctx, "meaning of life: %d, name: %s", 42, "Zaphod");
```


## duk_push_string() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_push_string(duk_context *ctx, const char *str);
```

### スタック

. . . → . . . str (if str != NULL)
. . . → . . . null (if str == NULL)

### 要約

Push a C string into the stack. String length is automatically detected with a strlen() equivalent (i.e. looking for the first NUL character). A pointer to the interned string data is returned. If the operation fails, throws an error.

If str is NULL, an ECMAScript null is pushed to the stack and NULL is returned. This behavior differs from duk_push_lstring on purpose.

C code should normally only push valid CESU-8 strings to the stack. Some invalid CESU-8/UTF-8 byte sequences are reserved for special uses such as representing Symbol values. When you push such an invalid byte sequence, the value on the value stack will behave like a string for C code but will appear as a Symbol for ECMAScript code. See Symbols for more discussion.
If input string might contain internal NUL characters, use duk_push_lstring() instead.


### 例

```c
duk_push_string(ctx, "foo");
duk_push_string(ctx, "foo\0bar");  /* push "foo", not "foo\0bar" */
duk_push_string(ctx, "");          /* push empty string */
duk_push_string(ctx, NULL);        /* push 'null' */
```


## duk_push_this() 1.0.0stackfunction§

### プロトタイプ

```c
void duk_push_this(duk_context *ctx);
```

### スタック

. . . → . . . this

### 要約

Push the this binding of the currently running C function to the stack.


### 例

```c
duk_push_this(ctx);
```


## duk_push_thread() 1.0.0threadstackborrowed§

### プロトタイプ

```c
duk_idx_t duk_push_thread(duk_context *ctx);
```

### スタック

. . . → . . . thr

### 要約

Push a new Duktape thread (context, coroutine) to the stack. Returns non-negative index (relative to stack bottom) of the pushed thread. The new thread will be associated with the same Duktape heap as the argument ctx, and will share the same global object environment.

To interact with the new thread with the Duktape API, use duk_get_context() to get a context pointer for API calls.


### 例

```c
duk_idx_t thr_idx;
duk_context *new_ctx;

thr_idx = duk_push_thread(ctx);
new_ctx = duk_get_context(ctx, thr_idx);
```

### 参照

duk_push_thread_new_globalenv


## duk_push_thread_new_globalenv() 1.0.0threadstackborrowed§

### プロトタイプ

```c
duk_idx_t duk_push_thread_new_globalenv(duk_context *ctx);
```

### スタック

. . . → . . . thr

### 要約

Push a new Duktape thread (context, coroutine) to the stack. Returns non-negative index (relative to stack bottom) of the pushed thread. The new thread will be associated with the same Duktape heap as the argument ctx, but will have a new global object environment (separate from the one used by ctx).

To interact with the new thread with the Duktape API, use duk_get_context() to get a context pointer for API calls.


### 例

```c
duk_idx_t thr_idx;
duk_context *new_ctx;

thr_idx = duk_push_thread_new_globalenv(ctx);
new_ctx = duk_get_context(ctx, thr_idx);
```

### 参照

duk_push_thread


## duk_push_thread_stash() 1.0.0threadstashstacksandboxobjectmodule§

### プロトタイプ

```c
void duk_push_thread_stash(duk_context *ctx, duk_context *target_ctx);
```

### スタック

. . . → . . . stash

### 要約

Push the stash object related to target_ctx to the stack (the ctx and target_ctx arguments may refer to the same thread). The thread stash is an internal object which can be used to store key/value pairs from C code so that they are reachable for garbage collection, but are not accessible from ECMAScript code. The stash is only accessible from C code with a matching target_ctx argument.

If target_ctx is NULL, throws an error.


### 例

```c
duk_push_thread_stash(ctx, ctx2);
```

### 参照

duk_push_heap_stash
duk_push_global_stash


## duk_push_true() 1.0.0stack§

### プロトタイプ

```c
void duk_push_true(duk_context *ctx);
```

### スタック

. . . → . . . true

### 要約

Push true to stack. Equivalent to calling duk_push_boolean(ctx, 1).


### 例

```c
duk_push_true(ctx);
```


## duk_push_uint() 1.0.0stack§

### プロトタイプ

```c
void duk_push_uint(duk_context *ctx, duk_uint_t val);
```

### スタック

. . . → . . . val

### 要約

Convert val to an IEEE double and push it to the stack.

This is a shorthand for calling duk_push_number(ctx, (duk_double_t) val).


### 例

```c
duk_push_uint(ctx, 123);
```


## duk_push_undefined() 1.0.0stack§

### プロトタイプ

```c
void duk_push_undefined(duk_context *ctx);
```

### スタック

. . . → . . . undefined

### 要約

Push undefined to the stack.


### 例

```c
duk_push_undefined(ctx);
```


## duk_push_vsprintf() 1.0.0varargstringstack§

### プロトタイプ

```c
const char *duk_push_vsprintf(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . str

### 要約

Format a string like vsprintf() (but safely) and push the result value to the value stack. Returns a non-NULL pointer to the resulting string.

If fmt is NULL, an empty string is pushed to the stack and a non-NULL pointer to an empty string is returned (this behavior mimics what vsprintf() does for a NULL format string, at least on Linux). The returned pointer can be dereferenced and a NUL terminator character is guaranteed.

Unlike vsprintf() the string formatting is safe. Concretely, the implementation will try increasing temporary buffer sizes until a large enough buffer is found for the temporary formatted value.

The ap argument cannot be safely reused for multiple calls. This is a limitation of the vararg mechanism.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_put_function_list() 1.0.0propertymodule§

### プロトタイプ

```c
void duk_put_function_list(duk_context *ctx, duk_idx_t obj_idx, const duk_function_list_entry *funcs);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Set multiple function properties into a target object at obj_idx. The functions are specified as a list of triples (name, function, nargs), ending with a triple where name is NULL (preferably also function is NULL for sanity).

This is useful e.g. when defining modules or classes implemented as a set of Duktape/C functions.


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


## duk_put_global_heapptr() 2.3.0propertyheapptrborrowed§

### プロトタイプ

```c
duk_bool_t duk_put_global_heapptr(duk_context *ctx, void *ptr);
```

### スタック

. . . val → . . .

### 要約

Like duk_put_global_string(), but the property name is given as a Duktape heap pointer obtained e.g. using duk_get_heapptr(). If ptr is NULL, undefined is used as the key.


### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_heapptr(ctx, my_ptr_ref);
```

### 参照

duk_put_global_string
duk_put_global_lstring
duk_put_global_literal


## duk_put_global_literal() 2.3.0propertyliteral§

### プロトタイプ

```c
duk_bool_t duk_put_global_literal(duk_context *ctx, const char *key_literal);
```

### スタック

. . . val → . . .

### 要約

Like duk_put_global_string(), but the property name is given as a string literal (see duk_push_literal()).


### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_literal(ctx, "my_app_version");
```

### 参照

duk_put_global_string
duk_put_global_lstring
duk_put_global_heapptr


## duk_put_global_lstring() 2.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_put_global_lstring(duk_context *ctx, const char *key, duk_size_t key_len);
```

### スタック

. . . val → . . .

### 要約

Like duk_put_global_string() but the key is given as a string with explicit length.


### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_lstring(ctx, "internal" "\x00" "nul", 12);
```

### 参照

duk_put_global_string
duk_put_global_literal
duk_put_global_heapptr


## duk_put_global_string() 1.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_put_global_string(duk_context *ctx, const char *key);
```

### スタック

. . . val → . . .

### 要約

Put property named key to the global object. Return value behaves similarly to duk_put_prop(). This is a convenience function which does the equivalent of:

duk_bool_t ret;

duk_push_global_object(ctx);
duk_insert(ctx, -2);
ret = duk_put_prop_string(ctx, -2, key);
duk_pop(ctx);
/* 'ret' would be the return value from duk_put_global_string() */

### 例

```c
duk_push_string(ctx, "1.2.3");
(void) duk_put_global_string(ctx, "my_app_version");
```

### 参照

duk_put_global_lstring
duk_put_global_literal
duk_put_global_heapptr


## duk_put_number_list() 1.0.0propertymodule§

### プロトタイプ

```c
void duk_put_number_list(duk_context *ctx, duk_idx_t obj_idx, const duk_number_list_entry *numbers);
```

### スタック

. . . obj . . . → . . . obj . . .

### 要約

Set multiple number (double) properties into a target object at obj_idx. The number list is given as a list of pairs (name, number), ending with a pair where the name is NULL.

This is useful e.g. when defining numeric constants for modules or classes implemented in C.


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


## duk_put_prop() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_put_prop(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

. . . obj . . . key val → . . . obj . . .

### 要約

Write val to the property key of a value at obj_idx. key and val are removed from the stack. Return code and error throwing behavior:

If the property write succeeds, returns 1.
If the property write fails, throws an error (strict mode semantics). An error may also be thrown by the "setter" function of an accessor property.
If the value at obj_idx is not object coercible, throws an error.
If obj_idx is invalid, throws an error.
The property write is equivalent to the ECMAScript expression obj[key] = val. The exact rules of when a property write succeeds or fails are the same as for ECMAScript code making the equivalent assignment. For precise semantics, see Property Accessors, PutValue (V, W), and [[Put]] (P, V, Throw). Both the target value and the key are coerced:

The target value is automatically coerced to an object. However, the object is transitory so writing its properties is not very useful. Moreover, ECMAScript semantics prevent new properties from being created for such transitory objects (see PutValue (V, W), step 7 of the special [[Put]] variant).
The key argument is internally coerced using ToPropertyKey() coercion which results in a string or a Symbol. There is an internal fast path for arrays and numeric indices which avoids an explicit string coercion, so use a numeric key when applicable.
If the target is a Proxy object which implements the set trap, the trap is invoked and the API call return value matches the trap return value.

In ECMAScript an assignment expression has the value of the right-hand-side expression, regardless of whether or not the assignment succeeds. The return value for this API call is not specified by ECMAScript or available to ECMAScript code: the API call returns 0 or 1 depending on whether the assignment succeeded or not (with the 0 return value promoted to an error in strict code).
If the key is a fixed string you can avoid one API call and use the duk_put_prop_string() variant. Similarly, if the key is an array index, you can use the duk_put_prop_index() variant.

Although the base value for property accesses is usually an object, it can technically be an arbitrary value. Plain string and buffer values have virtual index properties so you can access "foo"[2], for instance. Most primitive values also inherit from some prototype object so that you can e.g. call methods on them: (12345).toString(16).

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


## duk_put_prop_heapptr() 2.2.0propertyheapptrborrowed§

### プロトタイプ

```c
duk_bool_t duk_put_prop_heapptr(duk_context *ctx, duk_idx_t obj_idx, void *ptr);
```

### スタック

. . . obj . . . val → . . . obj . . .

### 要約

Like duk_put_prop(), but the property name is given as a Duktape heap pointer obtained e.g. using duk_get_heapptr(). If ptr is NULL, undefined is used as the key.


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


## duk_put_prop_index() 1.0.0property§

### プロトタイプ

```c
duk_bool_t duk_put_prop_index(duk_context *ctx, duk_idx_t obj_idx, duk_uarridx_t arr_idx);
```

### スタック

. . . obj . . . val → . . . obj . . .

### 要約

Like duk_put_prop(), but the property name is given as an unsigned integer arr_idx. This is especially useful for writing to array elements (but is not limited to that).

Conceptually the number is coerced to a string for the property write, e.g. 123 would be equivalent to a property name "123". Duktape avoids an explicit coercion whenever possible.


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


## duk_put_prop_literal() 2.3.0propertyliteral§

### プロトタイプ

```c
duk_bool_t duk_put_prop_literal(duk_context *ctx, duk_idx_t obj_idx, const char *key_literal);
```

### スタック

. . . obj . . . val → . . . obj . . .

### 要約

Like duk_put_prop(), but the property name is given as a string literal (see duk_push_literal()).


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


## duk_put_prop_lstring() 2.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_put_prop_lstring(duk_context *ctx, duk_idx_t obj_idx, const char *key, duk_size_t key_len);
```

### スタック

. . . obj . . . val → . . . obj . . .

### 要約

Like duk_put_prop(), but the property name is given as a string with explicit length.


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


## duk_put_prop_string() 1.0.0stringproperty§

### プロトタイプ

```c
duk_bool_t duk_put_prop_string(duk_context *ctx, duk_idx_t obj_idx, const char *key);
```

### スタック

. . . obj . . . val → . . . obj . . .

### 要約

Like duk_put_prop(), but the property name is given as a NUL-terminated C string key.


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


## duk_random() 2.3.0random§

### プロトタイプ

```c
duk_double_t duk_random(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

C API counterpart to Math.random(); returns a random IEEE double in the range [0,1[.


### 例

```c
if (duk_random(ctx) <= 0.01) {
    printf("surprise!\n");
}
```


## duk_range_error() 2.0.0error§

### プロトタイプ

```c
duk_ret_t duk_range_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error() with an error code DUK_ERR_RANGE_ERROR.


### 例

```c
return duk_range_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_range_error_va() 2.0.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_range_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error_va() with an error code DUK_ERR_RANGE_ERROR.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_realloc() 1.0.0memory§

### プロトタイプ

```c
void *duk_realloc(duk_context *ctx, void *ptr, duk_size_t size);
```

### スタック

(No effect on value stack.)


### 要約

Like duk_realloc_raw() but may trigger a garbage collection to satisfy the request. However, the allocated memory itself is not automatically garbage collected. If allocated size is extended from previous allocation, the newly allocated bytes are not automatically zeroed and may contain arbitrary garbage.

Memory reallocated with duk_realloc() can be freed with either duk_free() or duk_free_raw().


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


## duk_realloc_raw() 1.0.0memory§

### プロトタイプ

```c
void *duk_realloc_raw(duk_context *ctx, void *ptr, duk_size_t size);
```

### スタック

(No effect on value stack.)


### 要約

Resize a previous allocation made with the allocation functions registered to the context. The ptr argument points to the previous allocation while size is the new allocation size. The call returns a pointer to the new allocation which may have a different pointer than the previous one. If the reallocation fails, a NULL is returned and the previous allocation is still valid. Reallocation failure should only be possible when the new size is larger than the previous size (i.e. caller tries to grow the allocation). The attempt to reallocate cannot trigger a garbage collection, and the allocated memory is not automatically garbage collected. If allocated size is extended from previous allocation, the newly allocated bytes are not automatically zeroed and may contain arbitrary garbage.

The exact behavior depends on the ptr and size arguments as follows:

If ptr is non-NULL and size is greater than zero, previous allocation is resized.
If ptr is non-NULL and size is zero, the call is equivalent to a duk_free_raw().
If ptr is NULL, and size is greater than zero, the call is equivalent to a duk_alloc_raw().
If ptr is NULL, and size is zero, the call is equivalent to a duk_alloc_raw() with a zero size argument; the call may return NULL or some other value which is safe to give to duk_free_raw().
Memory reallocated with duk_realloc_raw() can be freed with either duk_free() or duk_free_raw().


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


## duk_reference_error() 2.0.0error§

### プロトタイプ

```c
duk_ret_t duk_reference_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error() with an error code DUK_ERR_REFERENCE_ERROR.


### 例

```c
return duk_reference_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_reference_error_va() 2.0.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_reference_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error_va() with an error code DUK_ERR_REFERENCE_ERROR.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_remove() 1.0.0stack§

### プロトタイプ

```c
void duk_remove(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val(idx) . . . → . . . . . .

### 要約

Remove value at idx. Elements above idx are shifted down the stack by a step. If idx is an invalid index, throws an error.


### 例

```c
duk_push_int(ctx, 123);
duk_push_int(ctx, 234);
duk_push_int(ctx, 345);       /* -> [ 123 234 345 ] */
duk_remove(ctx, -2);          /* -> [ 123 345 ] */
```


## duk_replace() 1.0.0stack§

### プロトタイプ

```c
void duk_replace(duk_context *ctx, duk_idx_t to_idx);
```

### スタック

. . . old(to_idx) . . . val → . . . val(to_idx) . . .

### 要約

Replace value at to_idx with a value popped from the stack top. If to_idx is an invalid index, throws an error.

Negative indices are evaluated prior to popping the value at the stack top. This is also illustrated by the example.

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


## duk_require_boolean() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_require_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_boolean(), but throws an error if the value at idx is not a boolean or if the index is invalid.


### 例

```c
if (duk_require_boolean(ctx, -3)) {
    printf("value is true\n");
}
```


## duk_require_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_require_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . .

### 要約

Like duk_get_buffer(), but throws an error if the value at idx is not a (plain) buffer or if the index is invalid.


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


## duk_require_buffer_data() 1.3.0stackbufferobjectbuffer§

### プロトタイプ

```c
void *duk_require_buffer_data(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . .

### 要約

Like duk_get_buffer_data(), but throws an error if the value at idx is not a plain buffer or a buffer object, the value is a buffer object whose "backing buffer" doesn't fully cover the buffer object's apparent size, or the index is invalid.


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


## duk_require_c_function() 1.0.0stackfunction§

### プロトタイプ

```c
duk_c_function duk_require_c_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_c_function(), but throws an error if the value at idx is not an ECMAScript function associated with a Duktape/C function or if the index is invalid.


### 例

```c
duk_c_function funcptr;

funcptr = duk_require_c_function(ctx, -3);
```


## duk_require_callable() 1.4.0stack§

### プロトタイプ

```c
void duk_require_callable(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Throw an error if the value at idx is not callable or if the index is invalid.

There is no "get" primitive (duk_get_callable()) because there's no useful C return value for an arbitrary callable value. At the moment this call is equivalent to calling duk_require_function().

### 例

```c
duk_require_callable(ctx, -3);
```


## duk_require_constructable() 2.4.0stack§

### プロトタイプ

```c
void duk_require_constructable(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Throw an error if value at idx is not a constructable function or if idx is invalid. Otherwise returns.


### 例

```c
duk_require_constructable(ctx, 3);
```


## duk_require_constructor_call() 2.4.0stack§

### プロトタイプ

```c
void duk_require_constructor_call(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Throw an error if current function was called as a normal function call (Foo()) rather than a constructor call (new Foo()). Also throws if no current call is in progress. Otherwise returns.


### 例

```c
duk_require_constructor_call(ctx);
```


## duk_require_context() 1.0.0stackborrowed§

### プロトタイプ

```c
duk_context *duk_require_context(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_context(), but throws an error if the value at idx is not a Duktape thread or if the index is invalid.


### 例

```c
duk_context *new_ctx;

(void) duk_push_thread(ctx);
new_ctx = duk_require_context(ctx, -1);
```


## duk_require_function() 1.4.0stack§

### プロトタイプ

```c
void duk_require_function(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Throw an error if the value at idx is not a function or if the index is invalid.

There is no "get" primitive (duk_get_function()) because there's no useful C return value for an arbitrary function.

### 例

```c
duk_require_function(ctx, -3);
```


## duk_require_heapptr() 1.1.0stackheapptrborrowed§

### プロトタイプ

```c
void *duk_require_heapptr(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_heapptr(), but throws an error if the value at idx is not a Duktape heap allocated value (object, buffer, string) or if the index is invalid.


### 例

```c
void *ptr;

ptr = duk_require_heapptr(ctx, -3);
```

### 参照

duk_get_heapptr
duk_push_heapptr


## duk_require_int() 1.0.0stack§

### プロトタイプ

```c
duk_int_t duk_require_int(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_int(), but throws an error if the value at idx is not a number or if the index is invalid.


### 例

```c
printf("int value: %ld\n", (long) duk_require_int(ctx, -3));
```


## duk_require_lstring() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_require_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

. . . val . . .

### 要約

Like duk_get_lstring(), but throws an error if the value at idx is not a string or if the index is invalid.

A non-NULL return value is guaranteed even for zero length strings; this differs from how buffer data pointers are handled (for technical reasons).

### 例

```c
const char *buf;
duk_size_t len;

buf = duk_require_lstring(ctx, -3, &len);
printf("value is a string, %lu bytes\n", (unsigned long) len);
```

### 参照

duk_require_string


## duk_require_normalize_index() 1.0.0stack§

### プロトタイプ

```c
duk_idx_t duk_require_normalize_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Normalize argument index relative to the bottom of the current frame. The resulting index will be 0 or greater and will be independent of later stack modifications. If the input index is invalid, throws an error.


### 例

```c
duk_idx_t idx = duk_require_normalize_index(-3);
```

### 参照

duk_normalize_index


## duk_require_null() 1.0.0stack§

### プロトタイプ

```c
void duk_require_null(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Throw an error if the value at idx is not null or if the index is invalid.

There is no "get" primitive (duk_get_null()) because such a function would be a no-op.

### 例

```c
duk_require_null(ctx, -3);
```


## duk_require_number() 1.0.0stack§

### プロトタイプ

```c
duk_double_t duk_require_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_number(), but throws an error if the value at idx is not a number or if the index is invalid.


### 例

```c
printf("value: %lf\n", (double) duk_require_number(ctx, -3));
```


## duk_require_object() 2.2.0stack§

### プロトタイプ

```c
void duk_require_object(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Throw an error if the value at idx is not an object or if the index is invalid.

There is no "get" primitive (duk_get_object()) because such a function would be a no-op.

### 例

```c
duk_require_object(ctx, -3);
```


## duk_require_object_coercible() 1.0.0stackobject§

### プロトタイプ

```c
void duk_require_object_coercible(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_is_object_coercible() but throws a TypeError if val is not object coercible.


### 例

```c
duk_require_object_coercible(ctx, -3);
```


## duk_require_pointer() 1.0.0stack§

### プロトタイプ

```c
void *duk_require_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_pointer(), but throws an error if the value at idx is not a pointer or if the index is invalid.


### 例

```c
void *ptr;

ptr = duk_require_pointer(ctx, -3);
printf("my pointer is: %p\n", ptr);
```


## duk_require_stack() 1.0.0stack§

### プロトタイプ

```c
void duk_require_stack(duk_context *ctx, duk_idx_t extra);
```

### スタック

(No effect on value stack.)


### 要約

Like duk_check_stack() but an error is thrown if the value stack needs to be reallocated and that reallocation fails.

As a general rule, callers should use this function to reserve more stack space. If value stack cannot be extended, there is almost never a useful recovery strategy except to throw an error and unwind.


### 例

```c
duk_idx_t nargs;

nargs = duk_get_top(ctx);  /* number or arguments */

/* reserve space for one temporary for each input argument */
duk_require_stack(ctx, nargs);
```

### 参照

duk_check_stack


## duk_require_stack_top() 1.0.0stack§

### プロトタイプ

```c
void duk_require_stack_top(duk_context *ctx, duk_idx_t top);
```

### スタック

(No effect on value stack.)


### 要約

Like duk_check_stack_top() but an error is thrown if the value stack needs to be reallocated and that reallocation fails.

As a general rule, callers should use this function to reserve more stack space. If value stack cannot be extended, there is almost never a useful recovery strategy except to throw an error and unwind.


### 例

```c
duk_require_stack_top(ctx, 100);
printf("value stack guaranteed up to index 99\n");
```

### 参照

duk_check_stack_top


## duk_require_string() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_require_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_string(), but throws an error if the value at idx is not a string or if the index is invalid.

A non-NULL return value is guaranteed even for zero length strings; this differs from how buffer data pointers are handled (for technical reasons).
Symbol values are visible in the C API as strings so that both duk_is_symbol() and duk_is_string() are true. This behavior is similar to Duktape 1.x internal strings. You can distinguish Symbols from ordinary strings using duk_is_symbol(). For the internal representation, see symbols.rst.

### 例

```c
const char *buf;

buf = duk_require_string(ctx, -3);
printf("value is a string: %s\n", buf);
```

### 参照

duk_require_lstring


## duk_require_top_index() 1.0.0stack§

### プロトタイプ

```c
duk_idx_t duk_require_top_index(duk_context *ctx);
```

### スタック

(No effect on value stack.)


### 要約

Get the absolute index (>= 0) of the topmost value on the stack. If the stack is empty, throws an error.


### 例

```c
duk_idx_t idx_top;

/* throws error if stack is empty */
idx_top = duk_require_top_index(ctx);
printf("index of top element: %ld\n", (long) idx_top);
```

### 参照

duk_get_top_index


## duk_require_type_mask() 1.0.0stack§

### プロトタイプ

```c
void duk_require_type_mask(duk_context *ctx, duk_idx_t idx, duk_uint_t mask);
```

### スタック

. . . val . . .

### 要約

Like duk_check_type_mask() but throws a TypeError if val type does not match any of the mask bits.


### 例

```c
duk_require_type_mask(ctx, -3, DUK_TYPE_MASK_STRING |
                               DUK_TYPE_MASK_NUMBER);
```


## duk_require_uint() 1.0.0stack§

### プロトタイプ

```c
duk_uint_t duk_require_uint(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Like duk_get_uint(), but throws an error if the value at idx is not a number or if the index is invalid.


### 例

```c
printf("unsigned int value: %lu\n", (unsigned long) duk_require_uint(ctx, -3));
```


## duk_require_undefined() 1.0.0stack§

### プロトタイプ

```c
void duk_require_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Throw an error if the value at idx is not undefined or if the index is invalid.

There is no "get" primitive (duk_get_undefined()) because such a function would be a no-op.

### 例

```c
duk_require_undefined(ctx, -3);
```


## duk_require_valid_index() 1.0.0stack§

### プロトタイプ

```c
void duk_require_valid_index(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . .

### 要約

Validate argument index. Throw an error if index is valid, return (without a return value) otherwise.


### 例

```c
duk_require_valid_index(ctx, -3);
printf("index -3 is valid\n");
```

### 参照

duk_is_valid_index


## duk_resize_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_resize_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t new_size);
```

### スタック

. . . val . . .

### 要約

Resize a dynamic buffer at idx to new_size bytes. If new_size is larger than the current size, newly allocated bytes (above old size) are automatically zeroed. Returns a pointer to the new buffer data area. If new_size is zero, may return either NULL or some non-NULL value. If resizing fails, the value at idx is not a dynamic buffer, or idx is invalid, throws an error.


### 例

```c
void *ptr;

ptr = duk_resize_buffer(ctx, -3, 4096);
```


## duk_resume() 1.6.0thread§

### プロトタイプ

```c
void duk_resume(duk_context *ctx, const duk_thread_state *state);
```

### スタック

. . . state(N) → . . . (number of popped stack entries may vary)

### 要約

Resume Duktape execution previously suspended using duk_suspend(). The state argument must not be NULL. Value stack and state must be in the state where they were left by duk_suspend(); if that's not the case, memory unsafe behavior will happen.


### 例

```c
/* See example for duk_suspend(). */
```

### 参照

duk_suspend


## duk_safe_call() 1.0.0protectedcall§

### プロトタイプ

```c
duk_int_t duk_safe_call(duk_context *ctx, duk_safe_call_function func, void *udata, duk_idx_t nargs, duk_idx_t nrets);
```

### スタック

. . . arg1 . . . argN → . . . ret1 . . . retN

### 要約

Perform a protected pure C function call inside the current value stack frame (the call is not visible on the call stack). nargs topmost values in the current value stack frame are identified as call arguments, and nrets return values are provided after the call returns. Calling code must ensure value stack has reserve for nrets using e.g. duk_check_stack(), and handling reservation errors.

The udata (userdata) pointer is passed on to func as is, and makes it easy to pass one or more C values to the target function without using the value stack. Multiple values can be passed by packing them into a stack-allocated C struct and passing a pointer to the struct as userdata. The userdata argument is not interpreted by Duktape; if it isn't needed simply pass a NULL and ignore the udata argument in the safe call target.

The return value is:

DUK_EXEC_SUCCESS (0): call succeeded, nargs arguments are replaced with nrets return values. (This return code constant is guaranteed to be zero, so that one can check for success with a "zero or non-zero" check.)
DUK_EXEC_ERROR: call failed, nargs arguments are replaced with nrets values, first of which is an error value and the rest are undefined. (In exceptional cases, e.g. when there are too few arguments on the value stack, the call may throw.)
Unlike most Duktape API calls, this call returns zero on success. This allows multiple error codes to be defined later.
Because this call operates on the current value stack frame, stack behavior and return code handling differ a bit from other call types.

The top nargs elements of the stack top are identified as arguments to func establishing a "base index" for the return stack as:

(duk_get_top() - nargs)
When func returns, it indicates with its return value the number of return values it has pushed on top of the stack; multiple or zero return values possible. The stack is then manipulated so that there are exactly nrets values starting at the "base index" established before the call.

Note that since func has full access to the value stack, it may modify the stack below the intended arguments and even pop elements below the "base index" off the stack. Such elements are restored with undefined values before returning, to ensure that the stack is always in a consistent state upon returning.

If an error occurs, the stack will still have nrets values at "base index"; the first of such values is the error, and the remaining values are undefined. If nrets is zero, the error will not be present on the stack (the return stack top will equal the "base index"), so calling this function with nrets as 0 may not be very useful as the cause of a possible error is lost.

Example value stack behavior with nargs = 3, nrets = 2, func returns 4. Pipe chars indicate logical value stack boundaries:

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
Note that func uses the caller's stack frame, so bottom-based references are dangerous within 'func' unless the calling context is known.
The userdata argument was added in Duktape 2.x.

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


## duk_safe_to_lstring() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_safe_to_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

. . . val . . . → . . . ToString(val) . . .

### 要約

Like duk_to_lstring() but if the initial string coercion fails, the error value is coerced to a string. If that also fails, a fixed error string is returned.

The caller can safely use this function to coerce a value to a string, which is useful in C code to print out a return value safely with printf(). The only uncaught errors possible are out-of-memory and other internal errors.


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


## duk_safe_to_stacktrace() 2.4.0stringstackprotected§

### プロトタイプ

```c
const char *duk_safe_to_stacktrace(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . val.stack . . .

### 要約

Like duk_to_stacktrace() but if the coercion fails, duk_to_stacktrace() is applied to the coercion error. If that also fails, a fixed pre-allocated error string "Error" is used instead (because the string is pre-allocated, this cannot fail due to out-of-memory).

ECMAScript equivalent:

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


## duk_safe_to_string() 1.0.0stringstackprotected§

### プロトタイプ

```c
const char *duk_safe_to_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToString(val) . . .

### 要約

Like duk_to_string() but if the initial string coercion fails, the error value is coerced to a string. If that also fails, a fixed pre-allocated error string "Error" is used instead (because the string is pre-allocated, this cannot fail due to out-of-memory).

The caller can safely use this function to coerce a value to a string, which is useful in C code to print out a return value safely with printf(). The only uncaught errors possible are out-of-memory and other internal errors.

ECMAScript equivalent:

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
While the string coercion is safe from error throws, it may have side effects in the current implementation. In particular, the string coercion may enter an infinite loop and never return.

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


## duk_samevalue() 2.0.0compare§

### プロトタイプ

```c
duk_bool_t duk_samevalue(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

. . . val1 . . . val2 . . .

### 要約

Compare values at idx1 and idx2 for equality. Returns 1 if values are considered equal using ECMAScript SameValue algorithm (Object.is() in ES2015) semantics, otherwise returns 0. Also returns 0 if either index is invalid.


### 例

```c
if (duk_samevalue(ctx, -3, -7)) {
    printf("values at indices -3 and -7 are SameValue() equal\n");
}
```


## duk_seal() 2.2.0propertyobject§

### プロトタイプ

```c
void duk_seal(duk_context *ctx, duk_idx_t obj_idx);
```

### スタック

(No effect on value stack.)


### 要約

Equivalent to Object.seal() for object at obj_idx; if sealed, the object is automatically compacted. If the index is invalid an error is thrown.


### 例

```c
duk_seal(ctx, -3);
```


## duk_set_finalizer() 1.0.0objectfinalizer§

### プロトタイプ

```c
void duk_set_finalizer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . finalizer → . . . val . . .

### 要約

Set the finalizer of the value at idx to the value at stack top. If the target value is not an object an error is thrown. The finalizer value can be an arbitrary one; non-function values are treated as if no finalizer was set. To delete a finalizer from an object, set it to undefined.

Finalizer on a Proxy object is currently unsupported. At present the finalizer is stored as a hidden Symbol; a finalizer cannot be set if the object is non-extensible (sealed or frozen), so set the finalizer before sealing/freezing the object.

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


## duk_set_global_object() 1.0.0threadstacksandbox§

### プロトタイプ

```c
void duk_set_global_object(duk_context *ctx);
```

### スタック

. . . new_global → . . .

### 要約

Replace the current context's global object with the object on top of the value stack. If the value is not an object, an error is thrown.

Note that this operation does not affect the global object of other contexts, even those that have up to this point shared the same global environment. To inherit the change to other contexts, replace the global object first before calling duk_push_thread().

See the test case test-set-global-object.c for discussion of detailed behavior after the change.


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


## duk_set_length() 2.0.0stack§

### プロトタイプ

```c
void duk_set_length(duk_context *ctx, duk_idx_t idx, duk_size_t len);
```

### スタック

. . . val . . .

### 要約

Set "length" for value at idx. Equivalent to the ECMAScript statement obj.length = len;.


### 例

```c
/* Set array length to zero, deleting elements as a side effect. */
duk_set_length(ctx, -3, 0);
```

### 参照

duk_get_length


## duk_set_magic() 1.0.0magicfunction§

### プロトタイプ

```c
void duk_set_magic(duk_context *ctx, duk_idx_t idx, duk_int_t magic);
```

### スタック

. . . val . . .

### 要約

Set the 16-bit signed "magic" value associated with the Duktape/C function at idx. If the value is not a Duktape/C function, an error is thrown.


### 例

```c
duk_set_magic(ctx, -3, 0x1234);
```

### 参照

duk_get_current_magic
duk_get_magic


## duk_set_prototype() 1.0.0prototypeobject§

### プロトタイプ

```c
void duk_set_prototype(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . proto → . . . val . . .

### 要約

Set the internal prototype of the value at idx to the value at stack top (which must be an object or undefined). If the target value is not an object or the prototype value is not an object or undefined, an error is thrown.

Unlike ECMAScript prototype manipulation primitives, this API call allows creation of a prototype loop. Calling code should always avoid creating a prototype loop: although Duktape detects long or looped prototype chains and throws an error when it encounters one, such behavior is only intended as a last resort to avoid an infinite loop.

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


## duk_set_top() 1.0.0stack§

### プロトタイプ

```c
void duk_set_top(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . → . . .

### 要約

Set stack top (stack size) to match the argument idx, normalizing negative index values. If the value stack shrinks, values above the new stack top are dropped; if the value stack grows, undefined values are pushed to the new stack slots.

Negative index values are interpreted relative to the current stack top as in other API calls. For example, index -1 would decrease the stack top by one.


### 例

```c
/* Assume stack is empty initially. */

duk_push_int(ctx, 123);  /* -> top=1, stack: [ 123 ] */
duk_set_top(ctx, 3);     /* -> top=3, stack: [ 123 undefined undefined ] */
duk_set_top(ctx, -1);    /* -> top=2, stack: [ 123 undefined ] */
duk_set_top(ctx, 0);     /* -> top=0, stack: [ ] */
```


## duk_steal_buffer() 1.3.0stackbuffer§

### プロトタイプ

```c
void *duk_steal_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . .

### 要約

Steal the current allocation of the dynamic buffer at idx. In concrete terms, Duktape forgets the previous allocation and resets the buffer to zero size (and NULL data pointer). Pointer to the previous allocation is returned and previous allocation length is written to out_size if non-NULL. The dynamic buffer itself remains on the value stack and can be reused. The caller becomes responsible for freeing the previous allocation using duk_free(), otherwise a memory leak happens. Duktape won't free the previous allocation through garbage collection or even when the Duktape heap is destroyed.

This API call is useful when a dynamic buffer is used as a safe temporary in a buffer manipulation algorithm (it's automatically memory managed in case an error occurs). At the end of such an algorithm, the caller may want to steal the buffer so that it won't be freed by Duktape garbage collection even when the dynamic buffer itself is freed.


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


## duk_strict_equals() 1.0.0compare§

### プロトタイプ

```c
duk_bool_t duk_strict_equals(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

. . . val1 . . . val2 . . .

### 要約

Compare values at idx1 and idx2 for equality. Returns 1 if values are considered equal using ECMAScript Strict Equals operator (===) semantics, otherwise returns 0. Also returns 0 if either index is invalid.

Because The Strict Equality Comparison Algorithm used by the Strict Equals operator performs no value coercion, the comparison cannot have side effects and cannot throw an error.

Comparison algorithm for Duktape custom types is described in Strict equality.


### 例

```c
if (duk_strict_equals(ctx, -3, -7)) {
    printf("values at indices -3 and -7 are strictly equal\n");
}
```

### 参照

duk_equals


## duk_substring() 1.0.0string§

### プロトタイプ

```c
void duk_substring(duk_context *ctx, duk_idx_t idx, duk_size_t start_char_offset, duk_size_t end_char_offset);
```

### スタック

. . . str . . . → . . . substr . . .

### 要約

Replace a string at idx with a substring [start_char_offset, end_char_offset[ of the string itself. If the value at idx is not a string or the index is invalid, throws an error.

The substring operation works with characters, not bytes, and the offsets are character offsets. The semantics of this call are similar to String.prototype.substring (start, end). Offset values are clamped to string length (there is no need to clamp negative values because size_t is unsigned) and if start offset is larger than end offset, the result is an empty string.

To get a byte-oriented substring, use duk_get_lstring() to get a string data pointer and length, and duk_push_lstring() to push a slice of bytes.

### 例

```c
/* String at index -3 is 'foobar'.  Substring [2,5[ is "oba". */

duk_substring(ctx, -3, 2, 5);
printf("substring: %s\n", duk_get_string(ctx, -3));
```


## duk_suspend() 1.6.0thread§

### プロトタイプ

```c
void duk_suspend(duk_context *ctx, duk_thread_state *state);
```

### スタック

. . . → . . . state(N) (number of pushed stack entries may vary)

### 要約

Suspend the current call stack so that another native thread may operate on the same Duktape heap. The necessary internal state is stored on the value stack and/or the provided state allocation. The state pointer must not be NULL, otherwise memory unsafe behavior occurs. Execution must later be resumed using duk_resume(); if execution is not resumed later some internal book-keeping will be left in an inconsistent state. The native call stack of the current native thread (which calls duk_suspend()) must not be unwound while Duktape execution is suspended; when the call returns, calling code typically switches to another native thread or performs a native C stack switch.

This API call must not be used directly or indirectly from:

A finalizer call
A Duktape.errCreate() error augmentation call
Duktape does not provide any locking for ensuring only one native thread accesses a certain Duktape heap at a time. Application code must provide such mechanisms.
See Threading.


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


## duk_swap() 1.0.0stack§

### プロトタイプ

```c
void duk_swap(duk_context *ctx, duk_idx_t idx1, duk_idx_t idx2);
```

### スタック

. . . val1 . . . val2 . . . → . . . val2 . . . val1 . . .

### 要約

Swap values at indices idx1 and idx2. If the indices are the same, the call is a no-op. If either index is invalid, throws an error.


### 例

```c
duk_swap(ctx, -5, -1);
```

### 参照

duk_swap_top


## duk_swap_top() 1.0.0stack§

### プロトタイプ

```c
void duk_swap_top(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val1 . . . val2 → . . . val2 . . . val1

### 要約

Swap stack top with value at idx. If idx refers to the stack top, the call is a no-op. If the value stack is empty or idx is invalid, throws an error.


### 例

```c
duk_swap_top(ctx, -3);
```


## duk_syntax_error() 2.0.0error§

### プロトタイプ

```c
duk_ret_t duk_syntax_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error() with an error code DUK_ERR_SYNTAX_ERROR.


### 例

```c
return duk_syntax_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_syntax_error_va() 2.0.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_syntax_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error_va() with an error code DUK_ERR_SYNTAX_ERROR.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_throw() 1.0.0error§

### プロトタイプ

```c
duk_ret_t duk_throw(duk_context *ctx);
```

### スタック

. . . val

### 要約

Throw the value on top of the stack. This call never returns.

Even though the function never returns, the prototype describes a return value which allows code such as:

if (argvalue < 0) {
    duk_push_error_object(ctx, DUK_ERR_TYPE_ERROR, "invalid argument: %d", (int) argvalue);
    return duk_throw(ctx);
}
If the return value is ignored, cast to void to avoid compilation warnings:

if (argvalue < 0) {
    duk_push_error_object(ctx, DUK_ERR_TYPE_ERROR, "invalid argument: %d", (int) argvalue);
    (void) duk_throw(ctx);
}

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


## duk_time_to_components() 2.0.0time§

### プロトタイプ

```c
void duk_time_to_components(duk_context *ctx, duk_double_t time, duk_time_components *comp);
```

### スタック

(No effect on value stack.)


### 要約

Convert a time value to components (year, month, day, etc) interpreted in UTC. If the time value is invalid, e.g. beyond the valid ECMAScript time range, an error is thrown.

There are some differences to the ECMAScript Date UTC accessors like Date.prototype.getUTCMinutes():

The time value is allowed to have fractions (sub-millisecond resolution) so that the millisecond component may also have fractions.

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


## duk_to_boolean() 1.0.0stack§

### プロトタイプ

```c
duk_bool_t duk_to_boolean(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToBoolean(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToBoolean() coerced value. Returns 1 if the result of the coercion true, 0 otherwise. If idx is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.


### 例

```c
if (duk_to_boolean(ctx, -3)) {
    printf("coerced value is true\n");
}
```


## duk_to_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_to_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . . → . . . buffer(val) . . .

### 要約

Replace the value at idx with a buffer-coerced value. Returns a pointer to the buffer data (may be NULL for a zero-size buffer), and writes the size of the buffer into *out_size (if out_size is non-NULL). If idx is invalid, throws an error.

Coercion rules:

Buffer: no change, dynamic/fixed/external nature not changed
String: coerces into a fixed size buffer, byte-for-byte
Other types: first apply ECMAScript ToString(), then coerce into a fixed size buffer, byte-for-byte

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


## duk_to_dynamic_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_to_dynamic_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . .

### 要約

Like duk_to_buffer() but the result is always a dynamic buffer (unless an error is thrown). If the value is a fixed or an external buffer, convert it to a dynamic buffer.


### 例

```c
duk_size_t sz;
void *buf = duk_to_dynamic_buffer(ctx, -3, &sz);
```


## duk_to_fixed_buffer() 1.0.0stackbuffer§

### プロトタイプ

```c
void *duk_to_fixed_buffer(duk_context *ctx, duk_idx_t idx, duk_size_t *out_size);
```

### スタック

. . . val . . .

### 要約

Like duk_to_buffer() but if the value is a dynamic or an external buffer, convert it to a fixed buffer. The result is thus always a fixed buffer (unless an error is thrown).


### 例

```c
duk_size_t sz;
void *buf = duk_to_fixed_buffer(ctx, -3, &sz);
```


## duk_to_int() 1.0.0stack§

### プロトタイプ

```c
duk_int_t duk_to_int(duk_context *ctx, duk_int_t index);
```

### スタック

. . . val . . . → . . . ToInteger(val) . . .

### 要約

Replace the value at index with an ECMAScript ToInteger() coerced value. Returns an duk_int_t further coerced from the ToInteger() result using the algorithm described in duk_get_int() (this second coercion is not reflected in the new stack value). If index is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.

If you want the double version of the ToInteger() coerced value, use:

double d;

(void) duk_to_int(ctx, -3);
d = (double) duk_get_number(ctx, -3);
duk_get_int() int coercion is applied only to the return value, it is not reflected on the value stack. For instance, if value stack contains the string "Infinity", the value on the stack will be coerced to the number Infinity and DUK_INT_MAX will be returned from the API call.

### 例

```c
printf("ToInteger() + int coercion: %ld\n", (long) duk_to_int(ctx, -3));
printf("ToInteger() coercion: %lf\n", (double) duk_get_number(ctx, -3));
```


## duk_to_int32() 1.0.0stack§

### プロトタイプ

```c
duk_int32_t duk_to_int32(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToInt32(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToInt32() coerced value. Returns the coerced value. If idx is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.


### 例

```c
printf("ToInt32(): %ld\n", (long) duk_to_int32(ctx, -3));
```


## duk_to_lstring() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_to_lstring(duk_context *ctx, duk_idx_t idx, duk_size_t *out_len);
```

### スタック

. . . val . . . → . . . ToString(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToString() coerced value. Returns a non-NULL pointer to the read-only, NUL-terminated string data, and writes the string byte length to *out_len (if out_len is non-NULL). If idx is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.


### 例

```c
const char *ptr;
duk_size_t sz;

ptr = duk_to_lstring(ctx, -3, &sz);
printf("coerced string: %s (length %lu)\n", ptr, (unsigned long) sz);
```

### 参照

duk_safe_to_lstring


## duk_to_null() 1.0.0stack§

### プロトタイプ

```c
void duk_to_null(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . null . . .

### 要約

Replace the value at idx with null, regardless of the previous value. Throws an error if index is invalid.

Equivalent to duk_push_null() followed by duk_replace() to target index.

### 例

```c
duk_to_null(ctx, -3);
```


## duk_to_number() 1.0.0stack§

### プロトタイプ

```c
duk_double_t duk_to_number(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToNumber(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToNumber() coerced value. Returns the coerced value. If idx is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.


### 例

```c
printf("coerced number: %lf\n", (double) duk_to_number(ctx, -3));
```


## duk_to_object() 1.0.0stackobject§

### プロトタイプ

```c
void duk_to_object(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToObject(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToObject() coerced value. There is no return value. Throws an error if value at idx is not object coercible, or if idx is invalid.

The following types are not object coercible:

undefined
null
Custom type behavior: see Type algorithms. Note in particular that plain buffer values coerce into equivalent Uint8Array objects.


### 例

```c
duk_to_object(ctx, -3);
```


## duk_to_pointer() 1.0.0stack§

### プロトタイプ

```c
void *duk_to_pointer(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . pointer(val) . . .

### 要約

Replaces the value at idx with a pointer-coerced value. Returns the resulting void * value. If idx is invalid, throws an error.

Coercion rules:

Pointer: coerces to itself, no change
All heap allocated objects (string, object, buffer): coerce to a pointer which points to the Duktape internal heap header (use for debugging only, do not read/write)
Other types: coerce to NULL
This API call is really only useful for debugging. Note in particular that the pointer returned must not be accessed, as it points to an internal heap header. This is the case even for string/buffer values: the returned pointer differs from the one returned by duk_get_string() and duk_get_buffer().

### 例

```c
/* Don't dereference the pointer. */
printf("coerced pointer: %p\n", duk_to_pointer(ctx, -3));
```


## duk_to_primitive() 1.0.0stack§

### プロトタイプ

```c
void duk_to_primitive(duk_context *ctx, duk_idx_t idx, duk_int_t hint);
```

### スタック

. . . val . . . → . . . ToPrimitive(val)

### 要約

Replace the object at idx with an ECMAScript ToPrimitive() coerced value. The hint argument affects coercion of an object into a primitive type, and indicates preference for a string (DUK_HINT_STRING), a number (DUK_HINT_NUMBER), or neither (DUK_HINT_NONE). DUK_HINT_NONE causes a preference to a number, unless the input value is a Date instance, in which case a string is preferred (the exact coercion behavior is described in the ECMAScript specification). If idx is invalid, throws an error.

Custom type coercion:

Buffer value (plain): treated like an Uint8Array and usually coerces to the string [object Uint8Array]
Pointer value (plain): keep existing value
Pointer object: coerces to the underlying plain pointer value
Lightfunc value (plain): treated like a Function and usually coerces to the string [object Function]
Custom type coercion is described in Type conversion and testing.


### 例

```c
duk_to_primitive(ctx, -3, DUK_HINT_NUMBER);
```


## duk_to_stacktrace() 2.4.0stringstack§

### プロトタイプ

```c
const char *duk_to_stacktrace(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . val.stack . . .

### 要約

Coerce an arbitrary value to a stack trace. The value at idx is expected but not required to be an Error instance. If the argument is an object, the call looks up the argument's .stack property which becomes the result if the property value is a string. Otherwise the argument is replaced with duk_to_string(val) to ensure a string result. An error may also be thrown if the coercion fails e.g. due to side effects or out-of-memory.

There is a safe variant of this API call, duk_safe_to_stacktrace(), which may be more useful when dealing with errors.

ECMAScript equivalent of the value coercion:

function duk_to_stacktrace(val) {
    if (typeof val === 'object' && val !== null) {  // Require val to be an object.
        var t = val.stack;  // Side effects, may throw.
        if (typeof t === 'string') {  // Require .stack to be a string.
            return t;
        }
    }
    return String(val);  // Side effects, may throw.
}
The coercion intentionally avoids an instanceof check (based on inheritance) so that the call can also work on foreign Error objects created in another global environment (Realm).


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


## duk_to_string() 1.0.0stringstack§

### プロトタイプ

```c
const char *duk_to_string(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToString(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToString() coerced value. Returns a non-NULL pointer to the read-only, NUL-terminated string data. If idx is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.

ToString() coercion for a Symbol value causes a TypeError.
In Duktape 2.x plain buffers mimic Uint8Array objects and will usually ToString() coerce to "[object Uint8Array]". To convert buffer or buffer object contents into a string, use duk_buffer_to_string() (new in Duktape 2.0).

### 例

```c
printf("coerced string: %s\n", duk_to_string(ctx, -3));
```

### 参照

duk_safe_to_string
duk_buffer_to_string


## duk_to_uint() 1.0.0stack§

### プロトタイプ

```c
duk_uint_t duk_to_uint(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToInteger(val) . . .

### 要約

Like duk_to_int() but the return value coercion is the same as in duk_get_uint().

duk_get_uint() int coercion is applied only to the return value, it is not reflected on the value stack. For instance, if value stack contains the string "Infinity", the value on the stack will be coerced to the number Infinity and DUK_UINT_MAX will be returned from the API call.

### 例

```c
printf("ToInteger() + uint coercion: %lu\n", (unsigned long) duk_to_uint(ctx, -3));
printf("ToInteger() coercion: %lf\n", (double) duk_get_number(ctx, -3));
```


## duk_to_uint16() 1.0.0stack§

### プロトタイプ

```c
duk_uint16_t duk_to_uint16(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToUint16(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToUint16() coerced value. Returns the coerced value. If idx is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.


### 例

```c
printf("ToUint16(): %u\n", (unsigned int) duk_to_uint16(ctx, -3));
```


## duk_to_uint32() 1.0.0stack§

### プロトタイプ

```c
duk_uint32_t duk_to_uint32(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . ToUint32(val) . . .

### 要約

Replace the value at idx with an ECMAScript ToUint32() coerced value. Returns the coerced value. If idx is invalid, throws an error.

Custom type coercion is described in Type conversion and testing.


### 例

```c
printf("ToUint32(): %lu\n", (unsigned long) duk_to_uint32(ctx, -3));
```


## duk_to_undefined() 1.0.0stack§

### プロトタイプ

```c
void duk_to_undefined(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . val . . . → . . . undefined . . .

### 要約

Replace the value at idx with undefined, regardless of the previous value. Throws an error if index is invalid.

Equivalent to duk_push_undefined() followed by duk_replace() to target index.

### 例

```c
duk_to_undefined(ctx, -3);
```


## duk_trim() 1.0.0stringstack§

### プロトタイプ

```c
void duk_trim(duk_context *ctx, duk_idx_t idx);
```

### スタック

. . . str . . . → . . . trimmed_str . . .

### 要約

Trim string at idx. If the value at idx is not a string or the index is invalid, throws an error.

Trimming removes white space characters from the beginning and end the end of the string. The result is an empty string if the string consists entirely of white space characters. The set of characters considered to be white space is defined by the StrWhiteSpace production, which contains both WhiteSpace and LineTerminator characters. The trimming behavior matches those of String.prototype.trim(), parseInt(), and parseFloat().


### 例

```c
duk_trim(ctx, -3);
```


## duk_type_error() 2.0.0error§

### プロトタイプ

```c
duk_ret_t duk_type_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error() with an error code DUK_ERR_TYPE_ERROR.


### 例

```c
return duk_type_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_type_error_va() 2.0.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_type_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error_va() with an error code DUK_ERR_TYPE_ERROR.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_uri_error() 2.0.0error§

### プロトタイプ

```c
duk_ret_t duk_uri_error(duk_context *ctx, const char *fmt, ...);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error() with an error code DUK_ERR_URI_ERROR.


### 例

```c
return duk_uri_error(ctx, "my error: %d", (int) argval);
```

### 参照

duk_error


## duk_uri_error_va() 2.0.0varargerror§

### プロトタイプ

```c
duk_ret_t duk_uri_error_va(duk_context *ctx, const char *fmt, va_list ap);
```

### スタック

. . . → . . . err

### 要約

Convenience API call, equivalent to duk_error_va() with an error code DUK_ERR_URI_ERROR.

This API call is not fully portable because the va_end() macro in the calling code may never be reached (e.g. due to an error throw). Some implementations rely on va_end() to e.g. free memory allocated by va_start(); see https://stackoverflow.com/questions/11645282/do-i-need-to-va-end-when-an-exception-is-thrown. However, such implementations are rare so this isn't usually a practical concern.

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


## duk_xcopy_top() 1.0.0stackslice§

### プロトタイプ

```c
void duk_xcopy_top(duk_context *to_ctx, duk_context *from_ctx, duk_idx_t count);
```

### スタック

. . . val1 . . . valN → . . . val1 . . . valN (on source stack, from_ctx)
. . . → . . . val1 . . . valN (on target stack, to_ctx)

### 要約

Like duk_xmove_top() but the elements being copied are not popped of the source stack. Both source and target stack must reside in the same Duktape heap.

The order of from/to stack is reversed as compared to Lua's lua_xmove().

### 例

```c
duk_xcopy_top(new_ctx, ctx, 7);
```

### 参照

duk_xmove_top


## duk_xmove_top() 1.0.0stackslice§

### プロトタイプ

```c
void duk_xmove_top(duk_context *to_ctx, duk_context *from_ctx, duk_idx_t count);
```

### スタック

. . . val1 . . . valN → . . . (on source stack, from_ctx)
. . . → . . . val1 . . . valN (on target stack, to_ctx)

### 要約

Remove count arguments from the top of the source stack and push them onto the target stack. The caller must ensure that the target stack has enough allocated space with e.g. duk_require_stack(). Both source and target stack must reside in the same Duktape heap.

If the source and target stacks are the same, an error is currently thrown.

The order of from/to stack is reversed as compared to Lua's lua_xmove().

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
        return DUK_RET_TYPE_ERROR;  /* missing func argument */
    }

    /* Create a new context. */
    duk_push_thread();
    new_ctx = duk_require_context(ctx, -1);

    /* Move arguments to the new context.  Note that we need to extend
     * the target stack allocation explicitly.
     */
    duk_require_stack(new_ctx, nargs);
    duk_xmove_top(new_ctx, ctx, nargs);

    /* Call the function; new_ctx is now: [ func arg1 ... argN ]. */
    duk_call(new_ctx, nargs - 1);

    /* Return the function call result by copying it to the original stack. */
    duk_xmove_top(ctx, new_ctx, 1);
    return 1;
}
```

### 参照

duk_xcopy_top