# TH01 SDL Port — PLAN

目的: ReC98(`debloated` 起点)の TH01 三本(OP / REIIDEN / FUUIN)を macOS でネイティブ動作させる。SDL3 + PC-98 ハードウェアエミュレーション方式。

## 戦略(確定済み設計判断)

1. **ゲームコードは書き換えない**。VRAM 書き込みは `planar.h` マクロ、GRCG は `outportb(0x7C/0x7E)` に集約されている。そこを差し替え、PC-98 グラフィックスをソフトウェアエミュレートする。
2. Borland 方言(far/near/pascal、MK_FP、peekb/pokeb、_AX/_DX、inline asm)は互換ヘッダで吸収。inline asm ブロックのみ `#ifdef REC98_PORT` で C 実装に置換。
3. `VRAM_CHUNK` 系マクロは C++ proxy クラス(`VramRef<T>`)化して lvalue/rvalue 両用途を保つ(`|=` 等の使用箇所が多数)。
4. サウンドは最後(MDRV98 再実装が最難関)。それまで `mdrv2.h` API を無音スタブで満たす。
5. 統合ブランチ = ReC98 clone の `sdl-port`。ポートコードは `port/` 以下。worker は worktree で並行開発、コーディネータがマージ。

## マイルストーン

| M | 内容 | 完了条件 |
|---|---|---|
| M0 | pc98emu コア + SDL3 presenter | デモがエミュ経路(GRCG 含む)で描画、60fps |
| M1 | OP.EXE bring-up | タイトル画面がレンダリングされる |
| M2 | メニュー操作 | キー入力でメニュー遷移 |
| M3 | REIIDEN ステージ1 | 無音でプレイ可能 |
| M4 | サウンド | MDRV98 互換再生(ymfm) |
| M5 | FUUIN + 全ステージ | 通しプレイ |

## 既知のブロッカー候補

- **原作アセット不在**(ディスク検索済み、見つからず)。formats のユニットテストは合成データで進められるが、M1 の「実タイトル画面」検証には東方靈異伝の原本データが必要。**ここが自走の停止点になる可能性が高い** — 到達時にユーザーへ要求。
- GRCG TDW / EGC の正確なセマンティクス。一次資料([PC-9800 Series Technical Data Book]、np2 ソース)で検証必要。
- `int` 16bit 前提の挙動差(オーバーフロー依存コード)。発見ベースで int16_t 化。

## 参照

- SPEC.md — 各コンポーネントの仕様
- WORKFLOW.md — DAG・orchestration 運用
- ReC98 調査結果(2026-07-06 survey): platform/ は PC-98 バックエンドのみ、TH01 は C++ 化 100%(BSS スタブ 1 個除く)
