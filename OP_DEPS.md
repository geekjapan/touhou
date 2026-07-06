# TH01 OP.EXE dependency inventory (for port workers)

Source: dependency-closure survey of ReC98 `sdl-port` tree, 2026-07-06. Workers: read the section for your task.

## OP.EXE functional closure (M1 = title + menu)

Entry: `th01/pc98/entry.cpp` (main → main_op) + `th01/pc98/game_bat.cpp` + `th01/pc98/blitters.cpp` (`#pragma startup` → call `blitter_setup()` explicitly in port).
Main TU: `th01/op_01.cpp` (`main_op` at :661, title_init at :250).

Required support TUs: `th01/hardware/{egc,egcrect,frmdelay,graph,grp2xscs,grp_text,grppsafx,planar,vsync}.cpp`, `th01/core/{initexit,resident}.cpp`, `th01/formats/grp.cpp`, `th01/snd/mdrv2.cpp`(stub), plus `input_ok`/`input_shot` globals extracted from `th01/hardware/input.cpp` (do NOT compile the rest of input.cpp — it drags in REIIDEN gameplay headers).

NOT needed for M1: scoredat, pf (packfile), tram_x16/ztext (text RAM), hiscore/regist.

**T9 correction (2026-07-06, actual `port/CMakeLists.txt` `th01op_core` target):**
- `th01/hardware/ztext.cpp` **is** needed: `th01/core/initexit.cpp`'s `game_init()`/`game_switch_binary()` call `z_text_init()`/`z_text_25line()`/etc. directly and unconditionally — it's not gated behind anything TRAM-text-specific from `main_op()`'s own perspective.
- `th01/hardware/planar.cpp` must **not** be compiled under `REC98_PORT` — it (re)defines `VRAM_PLANE_B/R/G/E`, which `pc98emu/vram.cpp` already defines and binds; compiling both is a duplicate-symbol link error.
- `th01/hardware/grp2xscs.cpp` (`graph_2xscale_byterect_1_to_0_slow`) is **not** called anywhere in `main_op()`'s closure (verified) and was dropped — also avoids needing Borland's `_lrotl`/`_lrotr` intrinsics.
- `th01/pc98/{entry,game_bat,blitters}.cpp` are **not** built: `port/app/main_op.cpp` calls `main_op()` directly instead of reproducing the DOS multi-entrypoint `main()` dispatcher (`blitter_setup()`'s `#pragma startup` auto-run has no callers in the closure either, confirmed by grep).
- Real gaps beyond "compile the right TUs" that needed code (see `port/compat/x86real_shim.cpp`, `port/compat/borland.hpp`, `port/compat/mbstring_shim.hpp`, `platform/x86real/pc98/font.cpp`'s `REC98_PORT` branch, `th01/hardware/vsync.{hpp,cpp}`'s `REC98_PORT` branch, and `th01/hardware/graph.cpp`'s `Planes_declare` fix): `int86`/`outportb`/`inportb`/`getvect`/`setvect` definitions, Shift-JIS `_ismbblead`/`_ismbbkana`/`_mbcjmstojis` (no host `<mbctype.h>`/`<mbstring.h>`), `font_read()`'s BIOS call (unreconstructable seg:off on a flat host), the vsync-interrupt-driven frame counter (replaced by presenting a real host frame per read, see `port/app/host.hpp`), and `Planes_declare`'s raw `reinterpret_cast<... __seg *>(SEG_PLANE_B)` (garbage pointer once `__seg` is emptied out — must use the already-bound `VRAM_PLANE_B/R/G/E` globals instead).

## master.lib functions to shim (T3)

- File I/O → stdio: `file_ropen, file_read, file_close, file_create, file_write, file_size` (+later: `file_exist, file_seek`)
- Graphics → pc98emu: `grcg_setcolor, grcg_setcolor_rmw, grcg_setcolor_tcr, grcg_off, grcg_off_func, grcg_put, grcg_boxfill, egc_copy_rect_1_to_0_16, egc_copy_rect_1_to_0_16_word_w, graph_copy_page_to_other, graph_move_byterect_interpage, graph_access_and_show_0, graph_2xscale_byterect_1_to_0_slow, z_graph_{init,exit,show,hide,clear,clear_0,400line}, vram_offset_shift, vram_offset_*, vram_row_offset`
- Palette: `z_palette_{black,black_in,black_out,white_in,set_all_show,set_show,show_single,show_single_col}` + `z_Palettes` global
- Text glyphs: `text_extent` (grppsafx uses it); text_* TRAM funcs not needed M1
- Input: `key_start, key_end, key_sense_bios` (+`key_sense` later)
- RNG: `irand, irand_init`
- Misc: `dos_puts2`, `resdata_free` + `ResData<>` (resident.cpp), `farmalloc/farfree` (RTL)

Signatures: match `libs/master.lib/master.hpp` exactly.

## Direct BIOS/DOS/port I/O to shim

- `int86(0x18, ...)` keyboard BIOS: op_01.cpp:135,714; graph.cpp:101,130,137 (CRT mode); ztext.cpp (not M1)
- BIOS memory pokes: op_01.cpp:722 (BIOS_FLAG 0:0x500), :772-773 (KB buffer 0:0x524/0x526/0x528) — shim peek/poke segment 0 to a fake BDA buffer
- `outportb`: graph.cpp:146-157 (CRTC/GRCG/palette), grppsafx.cpp:49,156 (**CG-ROM port 0x68 glyph read — T6 must emulate CG-ROM glyph fetch interface**), vsync.cpp
- `MK_FP(SEG_PLANE_B,...)`: graph.cpp:337, grppsafx.cpp:54, grp.cpp:104
- `page_access` macro = `_outportb_(0xA6, page)` (platform/x86real/pc98/page.hpp:7)

## Files needing C rewrites (Borland-heavy, #ifdef REC98_PORT replacements)

- `th01/snd/mdrv2.cpp` — `_asm{}` :66-73,:120, pseudoregs :32-34, `geninterrupt(0xF2)` → whole file replaced by stub (T5). **`mdrv2_resident()` stub MUST return true** (op_01.cpp:672 exits otherwise).
- `th01/hardware/vsync.cpp` — `void interrupt` ISR, `_AL` pseudoreg → host frame counter (T1 host provides).
- `th01/core/initexit.cpp` — int06 ISR + `setvect` → no-op/signal, keep `vsync_init()` call path.
- `th01/hardware/ztext.cpp` — pseudoregs (not M1).
- `libs/piloadc/piloadm.asm` — PiLoad .PI decoder is ASM. **T8 must write a C .PI/.GRP decoder** (.GRP = PI variant, magic "ZN", 4-bit palette, decodes to VRAM; used via `graph_pi_load_pack` for REIIDEN2.grp/REIIDEN3.grp/op_win.grp).
- master.lib keyboard ASM (`key_start/key_end/key_sense_bios`) → T4 C reimplementation over SDL.

## main_op startup sequence (T9 acceptance path)

1. mdrv2_resident() → true (stub)
2. argv parse → 3. mdrv2_enable_if_board_installed (stub)
4. game_init(): setvect(6) no-op + vsync_init()
5. cfg_load() REIIDEN.CFG (defaults on missing — fine without assets)
6. int86(0x18) KB init + key_start()
7. title_init(): bgm load (stub); page_access(1); grp_put("REIIDEN2.grp") ← **first asset load, graceful-fail point without assets**; palette fades; whitelines_animate() (grcg_put + egc_copy_rect + irand + frame_delay)
8. HIT KEY loop (graph_putsa_fx blink) → title_window_put(op_win.grp) → menu loop (frame_delay(1)/frame)
