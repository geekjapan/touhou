# TH01 SDL Port — SPEC

作業リポジトリ: `/Users/geekjapan/dev/projects/touhou/ReC98`(ブランチ `sdl-port`)。ポートコードは `port/` 以下。C++17、CMake ≥3.24、SDL3(Homebrew)。ビルド: `cmake -B port/build -S port && cmake --build port/build`。

## port/ レイアウト

```
port/
  CMakeLists.txt        # lib pc98emu, lib shim, exe th01op, exe th01main, exe demo, tests
  pc98emu/              # PC-98 ハードウェアモデル(ReC98 ヘッダに依存しない)
    vram.{hpp,cpp}      # planar 4面×2ページ、palette、render
    grcg.{hpp,cpp}      # GRCG レジスタ + RMW/TDW 書き込み
    egc.{hpp,cpp}       # EGC 16ドットパイプライン
    text.{hpp,cpp}      # TRAM 80×25 + JIS ビットマップフォント描画
    io.{hpp,cpp}        # outportb/inportb ディスパッチ
  compat/               # Borland → clang 吸収層(ReC98 ヘッダより先に include)
    borland.hpp         # far/near/pascal/__fastcall 空定義、MK_FP、peekb/pokeb、擬似レジスタ
    vram_ref.hpp        # VramRef proxy(VRAM_CHUNK 差し替え)
  shim/
    masterlib/          # master.lib API サブセット実装
    input.{hpp,cpp}     # SDL キーイベント → PC-98 keygroup ビットマップ
    snd_stub.cpp        # th01/snd/mdrv2.h API 無音実装
    dos.cpp             # DOS 呼び出しスタブ
  app/
    host.{hpp,cpp}      # SDL3 window/texture/フレームループ/vsync カウンタ
    main_op.cpp         # OP エントリ
    main_reiiden.cpp    # 本編エントリ
  tests/                # 合成データによるユニット+ゴールデンテスト
```

## コンポーネント仕様

### pc98emu/vram
- `uint8_t planes[2][4][32000]`(page, plane B/R/G/E, 640×400 ÷ 8)
- `VRAM_PLANE_B/R/G/E` と `VRAM_PLANE[4]`: アクセス中ページへの生ポインタ。`planar.h:77-83` の extern を満たす(port ビルドでは far は空)
- ページ切替: accessible / shown 独立(`page.hpp` 相当)。切替時にポインタ再バインド
- パレット 16 色 × RGB 4bit。`pc98emu_render(uint32_t*)`: shown ページ → RGBA8888(成分 ×17)。色インデックス: B=bit0, R=bit1, G=bit2, E=bit3

### pc98emu/grcg
- port 0x7C: bit7-6 モード(00=OFF, 10=TDW/TCR, 11=RMW)、bit3-0 プレーン除外マスク。書き込みでタイルローテーションリセット
- port 0x7E: タイルレジスタ B→R→G→E 巡回書き込み
- `grcg_vram_write(plane_hint, offset, src, bits)`:
  - RMW: 非除外プレーンごとに `p[off] = (p[off] & ~src) | (tile & src)`
  - TDW: CPU データ無視、非除外プレーンにタイル書き込み(※要一次資料検証、np2 の grcg.c と突き合わせ)
  - OFF: plane_hint のプレーンへ素通し
- マルチバイトはリトルエンディアン逐次

### pc98emu/egc
- 使用パターン優先: ページ間/ページ内 16 ドット矩形コピー(`egc.hpp:71` EGCCopy::rect / rect_interpage)
- レジスタ: リード/ライトプレーンマスク、パターンラッチ(4面×16bit)、ビットアドレス+シフト、モード
- 未実装モードは debug assert + TODO。np2 の egc.c を仕様の一次参照とする

### pc98emu/text
- TRAM 80×25、セルごと JIS コード + 属性バイト(`pc98.h:209-210` SEG_TRAM_JIS/ATRB 相当)
- フォント: 自由ライセンスの JIS ビットマップフォント同梱(8×16 半角 / 16×16 全角。候補: 美咲系ではなく 16px — JF ドットフォント or 東雲。ライセンス確認必須)
- gaiji(外字)16×16: TH01 が使う分を後続タスクで抽出

### compat/borland.hpp
- `#define far` / `near` / `pascal` / `__fastcall`(空)、`interrupt` 対応不要箇所は削除方針
- stdint は本物の `<stdint.h>` を使う(platform.h の Borland 分岐を `#elif defined(REC98_PORT)` で拡張)
- `MK_FP(seg, off)`: seg が SEG_PLANE_* / SEG_TRAM_* のとき対応バッファへ解決、他は assert
- `peekb/pokeb/peek/poke`: 同上のセグメント解決経由
- 擬似レジスタ `_AX/_BX/_CX/_DX/_AL/...`: thread_local 変数として定義(コード生成用途のみなので意味は保てる)
- `#pragma option` / `#pragma codeseg`: `-Wno-unknown-pragmas` で無視

### compat/vram_ref.hpp
- `planar.h` の `VRAM_CHUNK(plane, offset, bits)` を port ビルドで `VramRef<dotsN_t>(plane, offset)` に差し替え
- `operator=`(GRCG 状態考慮の書き込み)、`operator|=`, `&=`, `^=`、読み取り変換演算子
- ReC98 側 `planar.h` への変更は `#ifdef REC98_PORT` ブロック追加のみ(upstream diff 最小化)

### shim/masterlib
- OP.EXE 依存クロージャの調査結果(survey 継続中)に基づき必要関数のみ実装。想定カテゴリ: `dos_*`/ファイル I/O(stdio へ)、`graph_*`(pc98emu へ)、`text_*`/`gaiji_*`(pc98emu/text へ)、`palette_*`、乱数、`vsync_*`(host フレームカウンタへ)
- シグネチャは `libs/master.lib/master.hpp` に厳密一致

### shim/input
- PC-98 BIOS keygroup ビットマップ(`platform/x86real/pc98/keyboard.hpp:5-21`)を SDL_SCANCODE からマッピングして毎フレーム更新
- デフォルトキー: 矢印=移動、Z=ショット、X=ボム、Esc=ポーズ/終了(TH01 実キーは bring-up 時に input.cpp から確認)

### app/host
- SDL3、640×400 論理解像度(整数スケール)、streaming texture、vsync 同期
- `vsync_count_16/32` 相当をフレームカウンタで提供(`platform/x86real/pc98/vsync.hpp:8-19`)
- メインループ駆動: TH01 のゲームループは自前ループなので、ホストは「vsync 待ち API が呼ばれたら 1 フレーム提示」方式(コルーチン不要、ブロッキングで可)

### tests
- pc98emu: GRCG RMW/TDW、EGC コピー、render のゴールデン PPM 比較
- formats: 合成 .GRP/.PTN/.BOS を自前エンコーダで生成 → ロード → ピクセル検証
- CI スクリプト `port/ci.sh`: build + ctest。マージゲートとして全 worker ブランチに適用

## 検収基準(マイルストーン別)

- M0: demo がエミュ経路で描画、ctest 緑
- M1: `th01op` が実アセットでタイトル画面表示(アセット到着まで: OP クロージャ全 TU がコンパイル+リンク成功、合成データでの起動シーケンス到達)
- M2: メニュー項目移動・決定・Config 画面遷移
- M3: ステージ 1 開始〜ボス到達、クラッシュなし 10 分
- M4: OP BGM が MDRV98 互換で鳴る
