# ((o)) Duktape Wiki

Duktapeの公式Wikiへようこそ!


## ドキュメンテーション

- http://duktape.org/guide.html - 過去のバージョン: 1.5 1.4 1.3 1.2 1.1 1.0
- http://duktape.org/api.html - 過去のバージョン: 1.5 1.4 1.3 1.2 1.1 1.0


## はじめに

- [ライン処理](process_lines.md)
- [プライマリーテスト](prime_test.md)


## How-To

- 致命的なエラーの処理方法
- 値スタック型の扱い方
- 関数呼び出しの方法
- 仮想プロパティの使い方
- ファイナライゼーションの使い方
- バッファの扱い方 (Duktape 1.x、Duktape 2.x)
- lightfuncsの使い方
- [モジュールの使い方](how_to_modules.md)
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
- Duktape 1.5.0のパフォーマンス測定
- Duktape 2.0.0のパフォーマンス測定
- Duktape 2.1.0のパフォーマンス測定
- Duktape 2.2.0のパフォーマンス測定
- Duktape 2.3.0の性能測定
- Duktape 2.4.0のパフォーマンス測定
- Duktape 2.5.0のパフォーマンス測定


## ロー・メモリの最適化

- ロー・メモリー環境: low-memory.rst
- ローメモリ設定オプションの提案: low-memory.yaml
- ハイブリッドプールアロケータの例: alloc-hybrid


## その他

- Duktapeを使ったプロジェクト
- デバッグ・クライアント


## 寄稿、著作権、ライセンス

- https://github.com/svaarala/duktape-wiki