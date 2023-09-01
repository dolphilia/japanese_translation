# ソート

本ドキュメントは、実装に使用したソートアルゴリズムについてまとめたものである。

ソートアルゴリズムは以下のような場合に必要となる。

* ``Array.prototype.sort()`` のために必要です。

* ``JSON.stringify()`` で、配列のインデックス付きキーをソートする場合。
  replacer`` が指定されている場合

以下をを参照してください。

* http://en.wikipedia.org/wiki/Sorting_algorithm#Comparison_of_algorithms

## Array.prototype.sort()

望ましい資質がある。

* コードフットプリントが小さい
* 制限付きC再帰
* インプレースソート(大規模な一時領域確保が不要)
* O(n log n)の平均性能(弱い)
* O(n log n) ワーストケース性能 (強め)
* 安定性 (等しい比較要素を並べ替えない)

C 言語の再帰は、小さな定数に対して再帰の深さが O(log n) である場合、実質的に境界が設定されます。  例えば、2^32個の要素と小さな定数の場合、再帰呼び出しは100回以下となり、ほとんどの環境では十分な性能となる。

``Array.prototype.sort()`` には、さらにいくつかの懸念事項があります。

* 値の比較は比較的高価で、強制演算や比較関数の呼び出しを伴います。  値の比較には比較的コストがかかり、強制演算や比較関数の呼び出しが発生します。強制演算の結果がキャッシュされない限り、要素は通常複数回強制演算されます。
* 未定義``や存在しない配列要素は特別に扱われる必要があります．この特別な動作は ``SortCompare()`` の指定アルゴリズムにカプセル化されています。
* 配列部分を持つオブジェクトの場合、要素の交換は安価です。
* 配列部分を持たないオブジェクトの場合、要素の交換は非常に高価です。 ``[[Get]]``, ``[[Put]]``, ``[[Delete]]`` の呼び出しが必要で、セッター/ゲッターを呼び出す可能性があります。

現在使用されているアルゴリズムは、ランダムなピボット選択を伴うクイックソートです。クイックソートは平均ケースでは O(n log n) ですが、ワーストケースでは O(n^2) であり、安定したソートではありません。  再帰の深さは最悪の場合で O(n) です。  現在のところ、配列部分に対する高速なパスは存在しません。

ランダムピボットは、より複雑な（そして大きなコードフットプリント）ピボット選択アルゴリズムに頼ることなく、最悪の場合 O(n^2) の動作の可能性を最小化するために使用されます。  乱数生成器は暗号強度ではないので、最悪の場合 O(n^2) の動作を引き起こす悪意のある入力を細工することが可能です。

現在のソリューションはプレースホルダであり、ソートアルゴリズムを完全に変更する必要があるかもしれません。  O(log n)の再帰深度を得ることは重要である。なぜなら、それがなければ、不運な原因でソートが失敗してしまうからであり、いくつかの組み込み環境ではスタックの増加を制限することが重要である。O(n log n)の最悪の場合の挙動を得ることは良いことですが、重要ではありません。

## DUK_ENUM_SORT_ARRAY_INDICES

このソートの必要性は ``JSON.stringify()`` において、 ``replacer`` 引数が Array である場合に発生します。  この場合、replacerは配列のインデックスの昇順で列挙されなければなりません。 

- E5.1 セクション 15.12.3, 最初のアルゴリズム, ステップ 4.b.ii:
  - replacerのプロパティのうち、配列インデックスのプロパティ名を持つ各値vについて。プロパティはその名前の配列インデックスの昇順で列挙される。

これは，疎な配列，つまり配列部分を持たない配列に対しても成り立つ必要があります．このような配列は、必ずしも配列インデックスのキーが正しい昇順になるとは限らないので、列挙のためにソートする必要があります。  列挙 API のフラグ ``DUK_ENUM_SORT_ARRAY_INDICES`` は、必要な動作を提供します。このソートの必要性から予想されるデータサイズは比較的小さく、要素はほとんど常に既に順番通りであることが予想されます。

現在使用されているアルゴリズムは挿入ソートであり、インプレースでうまく動作し、すでに（ほぼ）順番になっているケースを効率的に処理することができます。