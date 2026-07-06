# TH01 SDL Port — WORKFLOW(DAG + Orchestration 運用)

## DAG(確定)

```
T1  pc98emu-core     M0: vram/palette/grcg/egc/io + SDL3 host + demo     deps: —
T2  compat-layer     borland.hpp + vram_ref.hpp + planar.h/platform.h    deps: T1
                     への #ifdef REC98_PORT 追加
T14 ci-harness       port/ci.sh: build + ctest + ゴールデンテスト基盤     deps: T1
T6  text-font        TRAM + JIS フォントレンダラ + フォント調達           deps: T1
T7  egc-full         EGC 完全実装(np2 準拠)+ テスト                    deps: T1
T3  masterlib-shim   master.lib サブセット(OP 依存クロージャ分)         deps: T2
T4  input-shim       SDL → keygroup                                      deps: T2
T5  mdrv2-stub       無音サウンド API                                     deps: T2
T8  formats-tests    th01/formats コンパイル + 合成データテスト           deps: T3
T9  op-bringup       OP.EXE クロージャ全 TU コンパイル+リンク+起動       deps: T4,T5,T6,T8
T10 menu-m2          メニュー操作完動                                     deps: T9
T13 sound-m4         MDRV98 再実装 + ymfm(長期、並行)                   deps: T9
T11 reiiden-m3       REIIDEN ステージ1                                    deps: T7,T10
T12 fuuin            FUUIN                                                deps: T11
```

並行ウェーブ: W1={T1} → W2={T2,T6,T7,T14} → W3={T3,T4,T5} → W4={T8} → W5={T9} → W6={T10,T13} → W7={T11} → W8={T12}

最大同時 worker: 4。

## ブランチ・マージ規約

- 統合ブランチ: ReC98 clone の `sdl-port`(Orca repo `ReC98`、base ref 設定済み)
- worker は Orca worktree(base=sdl-port)で `wt-<task>` ブランチ、完了時 commit 済みで worker_done
- コーディネータ(このセッション)がマージゲート: `port/ci.sh`(存在するまでは cmake build)通過 → `sdl-port` へ merge --no-ff → 次の ready タスク dispatch
- コンフリクト時: 後着 worker ブランチを rebase 指示(再 dispatch)
- ReC98 既存ファイルへの変更は `#ifdef REC98_PORT` 追加のみ許可(T2 担当が原則一元管理)

## Orchestration 運用

- コーディネータ = このセッション。`orca orchestration task-create --spec ... --deps ...` で DAG 登録
- worker 起動: `orca worktree create --name <task>`(agent なし)→ `orca terminal create --worktree id:<wt> --command "claude --model sonnet --dangerously-skip-permissions"`(worktree 隔離済みなので許可プロンプトで停止させない) → `terminal wait --for tui-idle` → `dispatch --task <id> --to <handle> --inject`(**worker は Sonnet 5 指定、ユーザー指示**)
- 監視: `orca orchestration check --wait --types worker_done,escalation,decision_gate --timeout-ms 900000` のローリングループ
- worker の質問は `ask`(ブロッキング)。コーディネータが `reply`
- **task spec に必ず明記**: 「作業はカレントディレクトリ(自分の Orca worktree)で行う。メイン checkout(/Users/geekjapan/dev/projects/touhou/ReC98)へ cd しない」。絶対パスで repo に言及すると worker がそこへ移動して index を共有事故る(T6/T4/T3 で実際に発生、2026-07-06)
- 停止条件(完全ブロッカー): 原作アセット必要到達(T9 実機検証)、ライセンス判断、外部公開判断。それ以外は自走継続

## 検証

- 各 worker: 自タスクのテストを書き、ローカルで cmake build + ctest 緑にしてから worker_done
- コーディネータ: マージ前に統合ブランチで再ビルド。M 到達ごとに verifier 相当の実走確認
