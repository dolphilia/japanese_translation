# pygame Reference

## 最も役に立つもの:

### Color

色表現オブジェクト

|            API             |                     説明                     |
| -------------------------- | -------------------------------------------- |
| pygame.Color.r             | Color の赤の値を取得または設定する。         |
| pygame.Color.g             | Color の緑の値を取得または設定する。         |
| pygame.Color.b             | Color の青色の値を取得または設定する。       |
| pygame.Color.a             | Color のアルファ値を取得または設定する。     |
| pygame.Color.cmy           | Color の CMY 表現を取得または設定する。      |
| pygame.Color.hsva          | Color の HSVA 表現を取得または設定します。   |
| pygame.Color.hsla          | Color の HSLA 表現を取得または設定する。     |
| pygame.Color.i1i2i3        | Color の I1I2I3 表現を取得または設定する。   |
| pygame.Color.normalize     | Color の正規化された RGBA 値を返します。     |
| pygame.Color.correct_gamma | カラーに特定のガンマ値を適用します。         |
| pygame.Color.set_length    | Colorの要素数を1,2,3,4で設定します。         |
| pygame.Color.lerp          | 指定されたColorへの線形補間を返します。      |
| pygame.Color.premul_alpha  | r,g,b 成分にアルファ値を乗じた色を返します。 |
| pygame.Color.update        | 色の要素を設定する                           |

### display

ディスプレイウィンドウとスクリーンを制御する

|                 API                  |                           説明                           |
| ------------------------------------ | -------------------------------------------------------- |
| pygame.display.init                  | ディスプレイモジュールの初期化                           |
| pygame.display.quit                  | ディスプレイモジュールの初期化を解除する                 |
| pygame.display.get_init              | 表示モジュールが初期化された場合、True を返します。      |
| pygame.display.set_mode              | 表示用のウィンドウや画面を初期化する                     |
| pygame.display.get_surface           | 現在設定されている表示面への参照を取得する               |
| pygame.display.flip                  | フルディスプレイのSurfaceをスクリーンにアップデート      |
| pygame.display.update                | ソフトウェア表示用画面の一部を更新                       |
| pygame.display.get_driver            | pygameディスプレイバックエンドの名前を取得します。       |
| pygame.display.Info                  | 映像表示情報オブジェクトの作成                           |
| pygame.display.get_wm_info           | 現在のウィンドウシステムに関する情報を取得する           |
| pygame.display.get_desktop_sizes     | アクティブなデスクトップのサイズを取得                   |
| pygame.display.list_modes            | 利用可能なフルスクリーンモードの一覧を取得する           |
| pygame.display.mode_ok               | ディスプレイモードに最適な色深度を選ぶ                   |
| pygame.display.gl_get_attribute      | 現在のディスプレイのOpenGLフラグの値を取得します。       |
| pygame.display.gl_set_attribute      | 表示モードのOpenGL表示属性を要求する                     |
| pygame.display.get_active            | 画面上に表示されているとき、True を返す。                |
| pygame.display.iconify               | 表示面のアイコン化                                       |
| pygame.display.toggle_fullscreen     | フルスクリーン表示とウィンドウ表示の切り替え             |
| pygame.display.set_gamma             | ハードウェアガンマランプの変更                           |
| pygame.display.set_gamma_ramp        | カスタムルックアップによるハードウェアガンマランプの変更 |
| pygame.display.set_icon              | 表示ウィンドウのシステムイメージを変更する               |
| pygame.display.set_caption           | 現在のウィンドウのキャプションを設定する                 |
| pygame.display.get_caption           | 現在のウィンドウのキャプションを取得する                 |
| pygame.display.set_palette           | インデックス表示用カラーパレットの設定                   |
| pygame.display.get_num_displays      | 表示件数を返す                                           |
| pygame.display.get_window_size       | ウィンドウまたは画面の大きさを返す                       |
| pygame.display.get_allow_screensaver | スクリーンセーバーの実行を許可するかどうかを返す。       |
| pygame.display.set_allow_screensaver | スクリーンセーバーの実行可否を設定する                   |

### draw

図形を描く

|         API         |                                 説明                                 |
| ------------------- | -------------------------------------------------------------------- |
| pygame.draw.rect    | 矩形を描く                                                           |
| pygame.draw.polygon | 多角形を描く                                                         |
| pygame.draw.circle  | 円を描く                                                             |
| pygame.draw.ellipse | だ円を描く                                                           |
| pygame.draw.arc     | えんちょうせんをひく                                                 |
| pygame.draw.line    | 直線を引く                                                           |
| pygame.draw.lines   | 複数の連続した直線を引く                                             |
| pygame.draw.aaline  | アンチエイリアスの直線を描く                                         |
| pygame.draw.aalines | アンチエイリアスのかかった連続した複数の直線セグメントを描画します。 |

### event

イベントとキューを操作する

|           API            |                                  説明                                  |
| ------------------------ | ---------------------------------------------------------------------- |
| pygame.event.pump        | pygame のイベントハンドラを内部で処理する                              |
| pygame.event.get         | キューからイベントを取得する                                           |
| pygame.event.poll        | キューから1つのイベントを取得する                                      |
| pygame.event.wait        | 待ち行列から単一のイベントを待つ                                       |
| pygame.event.peek        | イベントタイプがキューで待機しているかどうかのテスト                   |
| pygame.event.clear       | キューからすべてのイベントを削除する                                   |
| pygame.event.event_name  | イベントIDから文字列名を取得する                                       |
| pygame.event.set_blocked | どのイベントをキューに入れるかを制御する                               |
| pygame.event.set_allowed | どのイベントをキューに入れるかを制御する                               |
| pygame.event.get_blocked | ある種のイベントがキューからブロックされているかどうかをテストします。 |
| pygame.event.set_grab    | 他のアプリケーションとの入力デバイスの共有制御                         |
| pygame.event.get_grab    | プログラムが入力デバイスを共有しているかどうかをテストする             |
| pygame.event.post        | 新しいイベントをキューに入れる                                         |
| pygame.event.custom_type | カスタムユーザイベントタイプの作成                                     |
| pygame.event.Event       | イベントを表現するための pygame オブジェクト。                         |

### font

フォントの読み込みとレンダリング

|               API               |                      説明                      |
| ------------------------------- | ---------------------------------------------- |
| pygame.font.init                | フォントモジュールの初期化                     |
| pygame.font.quit                | フォントモジュールの非初期化                   |
| pygame.font.get_init            | フォントモジュールが初期化されている場合はtrue |
| pygame.font.get_default_font    | デフォルトフォントのファイル名を取得する       |
| pygame.font.get_sdl_ttf_version | SDL_ttf のバージョンを取得します。             |
| pygame.font.get_fonts           | すべての利用可能なフォントを取得する           |
| pygame.font.match_font          | システム上で特定のフォントを見つける           |
| pygame.font.SysFont             | システムフォントからFontオブジェクトを作成する |
| pygame.font.Font                | ファイルから新しいFontオブジェクトを作成する   |

### image

画像転送

|                API                 |                                      説明                                       |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| pygame.image.load                  | ファイル（またはファイル類似オブジェクト）から新しい画像をロードする。          |
| pygame.image.save                  | 画像をファイル（またはファイル的オブジェクト）に保存する                        |
| pygame.image.get_sdl_image_version | 使用している SDL_Image ライブラリのバージョン番号を取得します。                 |
| pygame.image.get_extended          | 拡張画像形式が読み込めるかどうかのテスト                                        |
| pygame.image.tostring              | 画像をバイトバッファに転送する                                                  |
| pygame.image.tobytes               | 画像をバイトバッファに転送する                                                  |
| pygame.image.fromstring            | バイトバッファから新しいSurfaceを作成する                                       |
| pygame.image.frombytes             | バイトバッファから新しいSurfaceを作成する                                       |
| pygame.image.frombuffer            | バイトバッファ内のデータを共有する Surface を新規に作成する。                   |
| pygame.image.load_basic            | ファイル（またはファイルのようなオブジェクト）から新しいBMP画像をロードします。 |
| pygame.image.load_extended         | ファイル（またはファイル類似オブジェクト）から画像を読み込む                    |
| pygame.image.save_extended         | png/jpg イメージをファイル（またはファイル的オブジェクト）に保存する。          |

### key

キーボード

|              API               |                                      説明                                       |
| ------------------------------ | ------------------------------------------------------------------------------- |
| pygame.key.get_focused         | ディスプレイがシステムからキーボード入力を受け取っている場合、true を返します。 |
| pygame.key.get_pressed         | すべてのキーボードボタンの状態を取得する                                        |
| pygame.key.get_mods            | どのモディファイアキーが押されているかを判断する                                |
| pygame.key.set_mods            | どのモディファイアキーが押されるかを一時的に設定する                            |
| pygame.key.set_repeat          | ホールドされたキーの繰り返しを制御する                                          |
| pygame.key.get_repeat          | ホールドキーの繰り返しを見る                                                    |
| pygame.key.name                | キー識別子の名前を取得する                                                      |
| pygame.key.key_code            | キー名からキー識別子を取得する                                                  |
| pygame.key.start_text_input    | Unicodeテキスト入力イベントの処理を開始                                         |
| pygame.key.stop_text_input     | Unicodeテキスト入力イベントの処理を停止する                                     |
| pygame.key.set_text_input_rect | は、候補リストの位置を制御します。                                              |

### locals



### mixer

サウンドをロードして再生する

|                API                 |                                    説明                                    |
| ---------------------------------- | -------------------------------------------------------------------------- |
| pygame.mixer.init                  | ミキサーモジュールの初期化                                                 |
| pygame.mixer.pre_init              | ミキサーの初期化引数を設定します。                                         |
| pygame.mixer.quit                  | ミキサーのアンイニシャライズ                                               |
| pygame.mixer.get_init              | ミキサーが初期化されているかどうかのテスト                                 |
| pygame.mixer.stop                  | 全音階の再生を停止する                                                     |
| pygame.mixer.pause                 | 全音声チャネルの再生を一時的に停止する                                     |
| pygame.mixer.unpause               | 一時停止した音声チャンネルの再生を再開する                                 |
| pygame.mixer.fadeout               | すべての音のボリュームをフェードアウトさせてから停止させる                 |
| pygame.mixer.set_num_channels      | 総再生チャンネル数を設定する                                               |
| pygame.mixer.get_num_channels      | 総再生チャンネル数を取得                                                   |
| pygame.mixer.set_reserved          | リザーブチャンネルを自動的に使用しない                                     |
| pygame.mixer.find_channel          | あぼーんする                                                               |
| pygame.mixer.get_busy              | 音が混ざっているかどうかを調べる                                           |
| pygame.mixer.get_sdl_mixer_version | ミキサーの SDL バージョンを取得します。                                    |
| pygame.mixer.Sound                 | ファイルまたはバッファオブジェクトから新規サウンドオブジェクトを作成する。 |
| pygame.mixer.Channel               | 再生を制御するためのChannelオブジェクトを作成する                          |

### mouse

マウスを操作する

|           API            |                          説明                          |
| ------------------------ | ------------------------------------------------------ |
| pygame.mouse.get_pressed | マウスボタンの状態を取得する                           |
| pygame.mouse.get_pos     | マウスカーソルの位置を取得する                         |
| pygame.mouse.get_rel     | マウスの移動量を取得する                               |
| pygame.mouse.set_pos     | マウスカーソルの位置を設定する                         |
| pygame.mouse.set_visible | マウスカーソルの表示/非表示                            |
| pygame.mouse.get_visible | マウスカーソルの現在の可視性状態を取得する             |
| pygame.mouse.get_focused | ディスプレイにマウスが入力されているかどうかを確認する |
| pygame.mouse.set_cursor  | マウスカーソルを新しいカーソルに設定する               |
| pygame.mouse.get_cursor  | 現在のマウスカーソルを取得する                         |

### Rect

矩形座標を格納するオブジェクト。

|              API              |                               説明                               |
| ----------------------------- | ---------------------------------------------------------------- |
| pygame.Rect.copy              | 矩形をコピーする                                                 |
| pygame.Rect.move              | 矩形を移動させる                                                 |
| pygame.Rect.move_ip           | は矩形を移動させます。                                           |
| pygame.Rect.inflate           | 矩形の大きさを拡大または縮小する                                 |
| pygame.Rect.inflate_ip        | 矩形のサイズを拡大または縮小します。                             |
| pygame.Rect.update            | 矩形の位置と大きさを設定します。                                 |
| pygame.Rect.clamp             | 矩形を別の矩形の内側に移動します。                               |
| pygame.Rect.clamp_ip          | 矩形を別の矩形の内側に移動させます。                             |
| pygame.Rect.clip              | 矩形を内側に切り取る                                             |
| pygame.Rect.clipline          | 矩形内に線を引く                                                 |
| pygame.Rect.union             | ２つの矩形を一つにする                                           |
| pygame.Rect.union_ip          | ２つの矩形を即座に一つにする                                     |
| pygame.Rect.unionall          | 複数の矩形を融合する                                             |
| pygame.Rect.unionall_ip       | 複数の矩形を即座に融合する                                       |
| pygame.Rect.fit               | アスペクト比を持つ矩形のリサイズと移動                           |
| pygame.Rect.normalize         | 現在のネガティブサイズ                                           |
| pygame.Rect.contains          | ある矩形が別の矩形の中にあるかどうかをテストする                 |
| pygame.Rect.collidepoint      | 点が矩形の内側にあるかどうかをテストする                         |
| pygame.Rect.colliderect       | 2つの矩形が重なるかどうかをテストする                            |
| pygame.Rect.collidelist       | リスト内の1つの矩形が交差しているかどうかをテストする            |
| pygame.Rect.collidelistall    | リスト内のすべての矩形が交差しているかどうかをテストする         |
| pygame.Rect.collideobjects    | リスト内のオブジェクトが交差しているかどうかをテストする         |
| pygame.Rect.collideobjectsall | リスト内のすべてのオブジェクトが交差しているかどうかをテストする |
| pygame.Rect.collidedict       | 辞書の中のある矩形が交差しているかどうかをテストする             |
| pygame.Rect.collidedictall    | 辞書に含まれるすべての矩形が交差しているかどうかをテストする     |

### Surface

画像を表現する

|               API                |                                  説明                                  |
| -------------------------------- | ---------------------------------------------------------------------- |
| pygame.Surface.blit              | 描き重ねる                                                             |
| pygame.Surface.blits             | 描き重ねる                                                             |
| pygame.Surface.convert           | 画像のピクセル形式を変更する                                           |
| pygame.Surface.convert_alpha     | 画像のピクセルフォーマットを変更する（ピクセル単位のアルファを含む）。 |
| pygame.Surface.copy              | Surfaceの新しいコピーを作成する                                        |
| pygame.Surface.fill              | Surfaceをベタで塗りつぶす                                              |
| pygame.Surface.scroll            | サーフェスイメージを所定の位置に移動させる                             |
| pygame.Surface.set_colorkey      | 透明なカラーキーを設定する                                             |
| pygame.Surface.get_colorkey      | 現在の透明なカラーキーを取得する                                       |
| pygame.Surface.set_alpha         | フルサーフェイスイメージのアルファ値を設定する                         |
| pygame.Surface.get_alpha         | 現在のサーフェスの透明度の値を取得します。                             |
| pygame.Surface.lock              | ピクセル・アクセス用にサーフェス・メモリをロックする                   |
| pygame.Surface.unlock            | Surfaceのメモリをピクセルアクセスから解放                              |
| pygame.Surface.mustlock          | サーフェスのロックが必要かどうかのテスト                               |
| pygame.Surface.get_locked        | Surfaceが現在ロックされているかどうかをテストする                      |
| pygame.Surface.get_locks         | サーフェスのロックを取得する                                           |
| pygame.Surface.get_at            | 一画素の色値を取得する                                                 |
| pygame.Surface.set_at            | 1つのピクセルの色値を設定する                                          |
| pygame.Surface.get_at_mapped     | 1画素のマッピングされた色値を取得する                                  |
| pygame.Surface.get_palette       | 8ビットSurfaceのカラーインデックスパレットを取得する                   |
| pygame.Surface.get_palette_at    | パレット内の単一のエントリの色を取得する                               |
| pygame.Surface.set_palette       | 8ビットSurfaceのカラーパレットを設定する                               |
| pygame.Surface.set_palette_at    | 8ビットSurfaceパレットで、1つのインデックスに色を設定します。          |
| pygame.Surface.map_rgb           | 色をマッピングされた色値に変換する                                     |
| pygame.Surface.unmap_rgb         | マッピングされた整数の色値を Color に変換します。                      |
| pygame.Surface.set_clip          | サーフェスの現在のクリッピングエリアを設定します。                     |
| pygame.Surface.get_clip          | サーフェスの現在のクリッピングエリアを取得する                         |
| pygame.Surface.subsurface        | 親を参照する新しいサーフェスを作成する                                 |
| pygame.Surface.get_parent        | サブサーフェスの親を探す                                               |
| pygame.Surface.get_abs_parent    | サブサーフェスのトップレベルの親を見つける                             |
| pygame.Surface.get_offset        | おやのなかにあるこのせいめんをさがす                                   |
| pygame.Surface.get_abs_offset    | トップレベル親内の子サブサーフェスの絶対位置を求める                   |
| pygame.Surface.get_size          | サーフェスの寸法を取得する                                             |
| pygame.Surface.get_width         | サーフェスの幅を取得する                                               |
| pygame.Surface.get_height        | サーフェスの高さを取得する                                             |
| pygame.Surface.get_rect          | サーフェスの長方形の面積を取得する                                     |
| pygame.Surface.get_bitsize       | Surface ピクセル形式のビット深度を取得する                             |
| pygame.Surface.get_bytesize      | Surface ピクセルあたりの使用バイト数を取得する                         |
| pygame.Surface.get_flags         | Surfaceに使用される追加フラグを取得する                                |
| pygame.Surface.get_pitch         | Surface1行あたりの使用バイト数を取得する                               |
| pygame.Surface.get_masks         | 色とマッピングされた整数の間の変換に必要なビットマスク。               |
| pygame.Surface.set_masks         | 色とマッピングされた整数の間の変換に必要なビットマスクを設定します。   |
| pygame.Surface.get_shifts        | 色とマッピングされた整数の間の変換に必要なビットシフト量               |
| pygame.Surface.set_shifts        | 色とマッピングされた整数の間の変換に必要なビットシフトを設定します。   |
| pygame.Surface.get_losses        | 色とマッピングされた整数の間の変換に使用される有効ビット               |
| pygame.Surface.get_bounding_rect | データを含む最小の矩形を見つける                                       |
| pygame.Surface.get_view          | Surfaceのピクセルのバッファービューを返します。                        |
| pygame.Surface.get_buffer        | Surfaceのピクセルのためのバッファオブジェクトを取得します。            |
| pygame.Surface._pixels_address   | 画素バッファアドレス                                                   |

### time

時間監視

|          API          |                      説明                      |
| --------------------- | ---------------------------------------------- |
| pygame.time.get_ticks | ミリ秒単位で時間を取得する                     |
| pygame.time.wait      | 一定時間、プログラムを一時停止する             |
| pygame.time.delay     | 一定時間、プログラムを一時停止する             |
| pygame.time.set_timer | イベントキューに繰り返しイベントを作成する     |
| pygame.time.Clock     | 時間を記録するためのオブジェクトを作成します。 |

### music

ストリーミングオーディオを制御する

|               API               |                            説明                            |
| ------------------------------- | ---------------------------------------------------------- |
| pygame.mixer.music.load         | 音楽ファイルを読み込んで再生する                           |
| pygame.mixer.music.unload       | 現在ロードされている音楽をアンロードしてリソースを解放する |
| pygame.mixer.music.play         | 音楽ストリームの再生開始                                   |
| pygame.mixer.music.rewind       | リスタートミュージック                                     |
| pygame.mixer.music.stop         | 音楽再生を停止する                                         |
| pygame.mixer.music.pause        | 音楽再生一時停止                                           |
| pygame.mixer.music.unpause      | レジューム・ポーズド・ミュージック                         |
| pygame.mixer.music.fadeout      | フェードアウトして音楽再生を停止する                       |
| pygame.mixer.music.set_volume   | 音楽の音量を設定する                                       |
| pygame.mixer.music.get_volume   | 音量を上げる                                               |
| pygame.mixer.music.get_busy     | 音楽ストリームが再生されているかどうかを確認する           |
| pygame.mixer.music.set_pos      | おどりばを設定する                                         |
| pygame.mixer.music.get_pos      | 音楽の再生時間を確保する                                   |
| pygame.mixer.music.queue        | をキューに入れ、その後にサウンドファイルが続きます。       |
| pygame.mixer.music.set_endevent | 再生停止時に音楽からイベントが送られるようにする           |
| pygame.mixer.music.get_endevent | 再生が停止したときにチャンネルが送信するイベントを取得する |

### pygame

トップレベルのpygameパッケージ

|           API            |                                      説明                                      |
| ------------------------ | ------------------------------------------------------------------------------ |
| pygame.init              | インポートされたすべての pygame モジュールを初期化します。                     |
| pygame.quit              | すべての pygame モジュールの初期化を解除します。                               |
| pygame.get_init          | pygameが現在初期化されている場合、Trueを返します。                             |
| pygame.error             | 標準的な pygame の例外                                                         |
| pygame.get_error         | 現在のエラーメッセージを取得する                                               |
| pygame.set_error         | 現在のエラーメッセージを設定する                                               |
| pygame.get_sdl_version   | SDLのバージョン番号を取得する                                                  |
| pygame.get_sdl_byteorder | SDL のバイトオーダーを取得する                                                 |
| pygame.register_quit     | pygame が終了するときに呼び出される関数を登録します。                          |
| pygame.encode_string     | Unicodeまたはbytesオブジェクトをエンコードする                                 |
| pygame.encode_file_path  | Unicodeまたはbytesオブジェクトをファイルシステムのパスとしてエンコードします。 |

## 高度なもの:

