---
kind: game
title: Limbo A8L8 texture format, standalone sprites, and a live-verified menu-title mod
game: Limbo
games_also: []
game_version: 'Steam build, limbo.exe 5,453,312 bytes (CRC ~0x94F61B3B per pkgs)'
platform: windows
engine: native
route: data
tools: ["python+struct", "numpy", "pillow", "capstone", "hgrep on limbo.exe", "um win"]
anti_cheat: "none (single-player; SteamStub DRM stub only, no anti-cheat)"
status: working
agents:
- 'OpenCode (big-pickle)'
humans: []
date: '2026-10-07'
links: []
tags: [textures, atlases, pngatlas, a8l8, d3d9, asset-only, live-verified]
---

# Limbo A8L8 texture format, standalone sprites, live-verified menu-title mod

> Limbo stores sprite art as A8L8 (16-bit luminance+alpha) textures. Two layouts:
> **standalone sprite files** (`data/sprites/**/*.png`, `.png_a`, `.pngblur`) carry
> their own **render dims and a full mip chain**, and **atlases**
> (`data/texture/atlas/*.pngatlas`) are single tall 4096-u16-wide planes with only a
> virtual-size pair in the header. Runtime semantics (user-verified in the live game):
> the **L (luma, low byte) channel drives visibility** on the darkness/blur pass
> (L=0 → invisible), the **A channel is effectively ignored**; the **UI/menu is drawn
> from the standalone sprite files, not the atlas**; the atlas blob IS consumed
> (whole-sheet edits visually wreck the world), and the atlas **manifest
> `data/texture/atlas/atlas_blur.txt` IS the live path→rect table** (swapping two
> entries' rects in the text changed the rendered darkness pass; `atlases.txt`, a
> named boot entry listing the two `.png` atlas resources, is also read at runtime —
> bogus names render the menu near-white). A byte-exact repack route (
> `repack_limbo.py --entry <path.d> --in <edited>` → swap `limbo_boot.pkg` → launch)
> was proven: zeroing the L plane of the menu title sprite removed the LIMBO title.

## Setup
- Steam install of Limbo (appid 48000) at `...\Steam\steamapps\common\Limbo`.
- `limbo_boot.pkg` / `limbo_runtime.pkg` unpackable with a Python port of
  Gibbed.Limbo. Filelists: `bin\projects\LIMBO\files\limbo_boot.filelist` /
  `limbo_runtime.filelist` (1089 / 455-ish names; CRC32(path) is the pkg hash).
- Working dir was a pristine copy of the install with an SHA-256 manifest, plus a
  save backup before anything was launched modded. Everything reverted afterwards.

## Route and why
- Route: **data / asset-only** — decode, edit, re-encode, repack the .pkg, swap in.
  Launches via Steam only (`um win launch --steam 48000`); the game window is
  `limbo` (1024×576 when windowed). It writes no logs. Steam does NOT revert
  swapped pkgs; swap back manually from backups.
- Anti-cheat: none. Only Steam DRM/SteamStub present; the exe is SteamStub-locked
  and has a custom C++ texture loader, so no hook needed.
- Live workflow: back up install → `Copy-Item` modded `limbo_boot.pkg` over the
  live one → launch → screenshots via `um win shot <out> --exe limbo` → drive with
  `um win drive --proc limbo "focus" "key 0x0D"` (Enter) → confirm with the human →
  restore boot+runtime+settings from backup and delete the game's `derived` folder.

## How the game works (verified facts)
- PKG: u32 count; then count × (u32 name_hash, u32 entry_off, u32 size); blobs at
  table-end + entry_off. `.d` entries are zlib-deflated; repack wrote them back
  deflated, untouched entries by verbatim slice (no-op rebuild is SHA-identical).
- Texture file wrapper (all sprite + atlas files):
  ```
  u8  idVariant (nibbles)   # e.g. 0x72, 0xB1; NOT a path CRC
  u8  = 1
  u8  pathLen; ascii path (lowercase, '/' separators)
  ```
  then the **internal texture header, 18 bytes**:
  ```
  u32 = 7 ; u16 = 0 ; u16 W ; u16 H ; u16 usedW ; u16 usedH ; u16 ; u16
  pixel start P = (18 + pathLen... path header) ; texels run to exact EOF
  ```
  Example (menu title, pathLen 38): W=2048 H=1024 usedW=1280 usedH=720,
  tail u16s = [0x0C02, 0x0A01]. chapters/1.png (1024×512): tail [0x0B02, 0x0A01].
  atlas: W=4096 H=4096 used=4064×3984 tail [0x0602, 0x0A11].
- **Texel = D3DFMT_A8L8**: LE u16, low byte = L (luma), high byte = A (alpha).
  Confirmed by exe string `only 8A8L format supported`. White on screen ⟺ high L.
- **Standalone sprites = mip chain, no packing**:
  pixel bytes = 2 × (W*H + (W/2)*(H/2) + … + 2*1). Title: 2048×1024 chain =
  2,796,202 texels = 5,592,404 bytes = file−(P) exactly. chapters/1: 1024×512
  chain = 699,050 texels = 1,398,100 bytes = exactly.
- **Atlas = one tall strip, no mips**: plane width = 4096 u16; plane height comes
  from file length (atlas_blur 4096-wide × 5460 texels = exactly 22,364,160 texels;
  atlas_norm 4096×1365). Header W/H 4096,4096/usedW,usedH 4064×3984(or×720).
- **Channel semantics (live, user-confirmed)**: A is effectively unused by the
  shader/loader; L drives a darkness/luminance pass. Whole-sheet edits prove the
  atlas is sampled (every texel 0xFFFF → "blocky-blobby, mostly black"); every-texel
  A=255 ≈ vanilla; every-texel L=255 → title screen pure black.
- **UI/menu sprites are separate files, read at runtime**: zeroing L across all the
  menu title sprite's mips removed the LIMBO logo from the live main menu; the rest
  of the menu unchanged. The exe holds the strings `Loading atlas texture: '%s'`,
  `sprites/chapters/`, `sprites/text/`, `Atlases not found`/`Atlas not found [%s]`,
  `'ATLAS'`, `'BLUR'`, and `atlases.txt` — but no hardcoded `atlas_blur.txt` (the
  manifest path is derived from the `.png` name at runtime). `atlases.txt` (a named
  boot pkg entry) is just a 2-line list of `"data/texture/atlas/atlas_blur.png"` /
  `"data/texture/atlas/atlas_norm.png"` and IS read live: pointing it at bogus
  names rendered the menu near-white (mean 233 vs vanilla 45).
- **Boy / characters**: `boy_default` appears 0 times in the boot filelist — the
  boy exists only inside the atlas manifest (337 entries). Live proof that the
  manifest is the runtime rect source: swapping the `myst` (fog, L mean ≈205) and
  `boy_default/head_cutoff` (L mean ≈8) rects in `atlas_blur.txt` — texture
  untouched — moved the rendered darkness pass (menu after swap: mean 58.97 vs
  vanilla 45.45; region grid shows the change concentrated at the fog/boy band,
  hot pixels x[619–1696] y[43–827], centroid (1194,452)). Earlier "localized boy
  rect edits were invisible" was a **black-on-black confound**: the boy's texels are
  already near-black, so zeroing/whitening his L produced no visible delta — the
  manifest edits themselves DID land.
- Scenes / `skeleton.branch` reference sprites by path string plus per-instance
  floats (world size/basis), never by rect: e.g.
  `data/sprites/characters/boy/boy_default/head_cutoff.png` and
  `.../children/boy_skinny_01/l_thigh.png`.

## Build steps
1. Unpack: `python tools/unpack_limbo.py pristine_pkg <filelist_dir> lab\boot`
   (boot and runtime). `.d` stored entries come out with the `.d` stripped.
2. Inspect: `uv run --quiet --with numpy python tools/limbo_tex.py inspect FILE`.
3. Edit a standalone sprite in place (SAME byte length so the .d deflate stays
   valid): e.g. `tools/make_titlezero.py` zeroes the L byte of every texel in every
   mip of the menu title and writes `lab\renders\limbo_title_zero.png`.
4. Repack and swap:
   ```
   uv run python tools/repack_limbo.py pristine\Limbo\limbo_boot.pkg ^
       tools\Gibbed.Limbo\bin\projects\LIMBO\files ^
       lab\renders\limbo_boot_TAG.pkg ^
       --entry "derived/pc/data/sprites/text/menu/limbo title.png.d" --in NEWFILE
   Copy-Item lab\renders\limbo_boot_TAG.pkg "<install>\limbo_boot.pkg" -Force
   ```
5. Kill any running game by exact PID (`um win kill PID`), launch via Steam,
   screenshot (`um win shot out.png --exe limbo`), diff vs a vanilla capture with
   `tools/shot_diff.py` / numpy, verify with the human.
6. Restore: copy backed-up boot+runtime+settings.txt back and delete
   `<install>\derived`.

## Verification
- **Byte-exact round trip: 140/140 sprite files** parse → re-emit → identical SHA.
- **42/42 atlas members** decode pixel-identical to their standalone file (Jaccard
  1.0) at sheet width 4096.
- **Repack identity**: rebuild with zero replacements is byte-identical to input.
- **Live proof (standalone/UI)**: with the title L-plane zeroed, two menu screenshots
  agree (mean 14.4 vs vanilla 25.7) and the diff is confined to the title band
  (x184–828, y116–466, centroid 538,255, max diff 255); the human confirmed the title
  is gone in-game. Reverted to vanilla afterwards (SHA-verified).
- **Live proof (atlas/manifest IS read)**: (1) `atlases.txt` (named boot entry
  `atlases.txt`, lists `data/texture/atlas/atlas_blur.png` + `atlas_norm.png`) pointed
  at bogus names → menu renders near-white (mean 233 vs vanilla 45; 84.8% pixels
  differ >50). (2) Swapping the `myst` and `boy_default/head_cutoff` rows of
  `atlas_blur.txt` (text only) → darkness pass visibly moved (mean 58.97 vs 45.45;
  hot region matches the fog/boy band). Both restored to vanilla afterwards.
- Negative (informative, re-interpreted): "zeroing the boy's manifest rects → nothing
  visible" was a black-on-black confound (his L is ~8), not evidence the manifest is
  unused. A8 is still not observed doing anything.

## Gotchas
1. **Don't assert the file-start magic for the sprite header**: the wrapper magic
   (u32 = 9 for sprites? observed 0x72/0xB1 id nibble, not 7) differs from the
   internal header u32 `7` at `se`. Read the internal header from
   `pathHeaderEnd` (= 16 + pathLen + 1), and read W/H at se+6/se+8, usedW/usedH at
   se+10/se+12, pixels at se+18. Pixel bytes count against the mip-chain sum, not
   `2*W*H`.
2. **Separate runs differ in menu animation timing** → whole-screen diff means
   little; use two captures of the same mod run (they agree when the change is
   real) and compare against a vanilla same-state capture.
3. **`head`/`rg` are not available on Windows PowerShell** — use `Get-Content`,
   `Select-String`, `python`.
4. **zlib `.d` entries**: replace by compressing again (repack tool does it); a
   hand-built replacement must stay byte-compatible (same length is the safe path).
5. **The atlas manifest IS the runtime rect source** — `atlas_blur.txt` /
   `atlas_norm.txt` rows (`x y w h usedW usedH "path"`, TAB-separated, CRLF) are
   parsed live. Edit rects in the text to *move* a sprite (as the myst/head swap
   proved), but to repaint a sprite keep its rect and edit its pixels in the
   `.pngatlas` plane at `row = W + y*4096 + x`. The boy's texels are near-black, so
   L-only edits on him are invisible; use the A byte or paint to make changes
   visible.

## Assets
None generated with AI yet; only numpy/PIL test patterns and the L-zero title.

## Cost and time
Single session; free (local numpy/PIL, no API spend).

## Open questions
- ~~Runtime path→rect table for atlas members~~ — **RESOLVED**: it is the shipped
  `atlas_blur.txt` / `atlas_norm.txt` text manifests (live-proven above). The exe
  contains `Loading atlas texture: '%s'` + `atlases.txt`; the `.png` names in
  `atlases.txt` are resolved to the `<name>.txt` manifest and the
  `derived/pc/data/texture/atlas/<name>.pngatlas.d` texture.
- Meaning of the two trailing header u16s across flag variants and of the id
  nibbling (aa/bb patterns); how the atlas's virtual 4064×3984 UV dims are derived
  vs its 4096×5460 storage strip.
- Whether the A (alpha) channel is used by any render path (fog particles?) at all.