## デバッガーについて {#debugger}

Duktapeには、コンパイル時に有効にできるオプションとして、ビルトインのデバッガ・サポートがあります。デバッガ・サポートは、約15-20kBのコード・フットプリントを追加し（どのようなデバッガ機能が有効になっているかに依存）、メモリ・フットプリントは非常に小さくなっています。デバッガ機能には以下のものがあります。

- ファイル/ラインでの実行/一時停止、コールスタック、異なるコールスタックレベルのローカル変数などの実行ステータス情報
- 一時停止/再開、ステップオーバー/イント/アウト、ファイル/行を対象としたブレークポイント、デバッガステートメントなどの実行制御
- 任意のコールスタックレベルでの汎用Eval、任意のコールスタックレベルでの変数get/put
- アプリケーション定義のリクエスト（AppRequest）および通知（AppNotify）のための機構
- ヒープオブジェクトの詳細な検査、Duktapeヒープウォーキング、ヒープダンプの完全取得

デバッガーは、以下の主要なコンセプトに基づいています。

- Duktapeは、全てのアプリケーションに共通する組み込みのデバッグ・プロトコルを提供します。アプリケーションは、デバッグ・プロトコルを解析したり、理解したりする必要はありません。デバッグ・プロトコルはコンパクトなバイナリ・プロトコルで、低速接続の低メモリ・ターゲットでも問題なく動作します。デバッグプロトコルのJSONマッピングとJSONデバッグプロキシがあり、デバッグクライアントの統合を容易にする。
- デバッグプロトコルは信頼性の高いストリームベースのデバッグトランスポート上で実行される。移植性を最大化するために、具体的なトランスポートはストリームインターフェースを実装したコールバックのセットとしてアプリケーションコードから提供されます。ストリームベースのトランスポートでは、デバッグメッセージのバッファなしストリーミングが可能で、メモリ使用量を非常に低く抑えることができます。
- デバッグ・クライアントはトランスポート接続を終了し、Duktapeデバッグ・プロトコルを使ってDuktape内部（一時停止／再開、ステップ、ブレークポイント、evalなど）と対話します。また、より簡単に統合するために、JSONデバッグ・プロキシーを使用することもできます。
- 非常に狭いデバッグAPIは、デバッガーをアタッチしたりデタッチしたり、デバッグ・トランスポートの実装に必要なコールバックを提供するために、アプリケーション・コードによって使用されます。その他のすべてのデバッグ活動は、アプリケーションの関与なしにDuktapeによって直接実装されるデバッグ・プロトコルを介して行われます。


最適なデバッグトランスポートは、Wi-Fi、Bluetooth、シリアル回線、カスタム管理プロトコルに組み込まれたストリームなど、デバッグターゲットによって大きく異なります。標準的な」トランスポートはありませんが、TCP接続は便利なデフォルトです。Duktapeの配布物には、TCPトランスポートを使用したデバッグを始めるために必要なすべてのパーツが含まれています。

- TCPトランスポートに必要なコールバックの実装例： duk_trans_socket_unix.c (Windowsの例もあります)
- Duktape コマンドラインツール (duk) の TCP トランスポートを使ったデバッガサポート。--debugger オプション
- Node.js、Express、socket.io に基づくデバッガ・ウェブ UI: duk_debug.js

Node.jsベースのデバッガ・ウェブUI（duk_debug.js）は、Duktapeコマンドラインに接続できますが、TCPトランスポートを実装した他のターゲットと直接会話することもできます。また、別のトランスポートを使用するようにカスタマイズしたり、TCPとあなたのカスタムトランスポートの間を変換するプロキシを使用したりすることも可能です。また、独自のデバッグクライアントをスクラッチから作成し、カスタム IDE に統合することも可能です。バイナリデバッグプロトコルを使用してデバッグターゲットと直接統合するか、duk_debug.js (Node.js) または duk_debug_proxy.js (DukLuv) が提供する JSON プロキシを使用することができます。

デバッグターゲットとデバッグクライアントは混在することを意図しています。トランスポート（通常はTCPか適応しやすい）を除けば、デバッグプロトコルは同じです。中核機能はデバッグクライアントやデバッグターゲットに関係なく同じですが、一部のオプション機能が欠落している可能性があります。デバッグクライアントとデバッグターゲットは、アプリケーション固有のコマンド (AppRequest) と通知 (AppNotify) を実装して、クライアントとターゲットの両方がサポートしている場合に使用できる統合機能を充実させることができます (サポートしていない場合は簡単に無視してかまいません)。カスタムコマンドと通知により、例えば、ターゲットから直接ソースファイルをダウンロードしたり、カスタムメモリアロケータの状態を深く調査したり、コマンドでターゲットを再起動したりすることができる。

実装の詳細と開始方法については、以下を参照してください。

- debugger/README.rst
- debugger.rst
- duk_trans_dvalue.c: ローカルデバッグプロトコルのデコード/エンコードによるデバッグトランスポートの例
- duk_debug_proxy.js
