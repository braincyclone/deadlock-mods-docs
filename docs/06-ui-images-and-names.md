# 06 — HUD images ("portraits/banners") and hero names

Rendering the portrait art itself: `06a-portraits.md`. This file: how to turn PNGs into valid game textures, which slots exist, and
how the hero's displayed name is changed.

## 1. Image slots per hero (inventory)

`python -I work/ui_inventory/scripts/inventory.py <outname> <codename> [display-name]` →
`work/ui_inventory/<outname>/{<slot>.png|svg, inventory.md, inventory.json, contact_sheet.png}` (originals + exact specs; done for `lash`, `billy`).
Slots come from `scripts/heroes.vdata` (`m_strIconHeroCard*`, `m_strTopBarVertical`, `m_strIconImageSmall`, `m_strMinimapImage`,
`m_strWeaponImage`, `m_strLogoImage{English,Localized}`) and `panorama/styles/hero_backgrounds/hero_background_default.css`.
Note Billy's files use the display name for the logo/background (`billy_*`) and the codename elsewhere (`punkgoat_*`).

| slot | VPK path (`panorama/images/heroes/…`) | size | format / mips / flags | alpha | shown in |
|---|---|---|---|---|---|
| card / card_critical / card_gloat | `<c>_card[_critical\|_gloat]_psd.vtex_c` | 280×380 | BGRA8888 / 9 / NO_LOD | cutout | player HUD panel, hero select (critical = low HP, gloat = streak) |
| vertical | `<c>_vertical_psd.vtex_c` | 120×200 | BGRA8888 / 1 / NO_LOD | opaque | top bar / hero select strip |
| sm | `<c>_sm_psd.vtex_c` | 128×128 | BGRA8888 / 1 / NO_LOD | sticker | kill feed, scoreboard |
| sm_hash | `<c>_sm_psd_<hash>.vtex_c` (`lash` 8b6cde8, `punkgoat` 3a39969e) | 128×128 | **BC7 / 6 / 0** | semi-transparent | faded variant |
| mm | `<c>_mm_psd.vtex_c` | 50×50 (Billy 50×51) | BGRA8888 / 1 / NO_LOD | sticker | minimap |
| **bg** | `backgrounds/lash_bg_psd.vtex_c`, `backgrounds/billy_bg_psd.vtex_c` | **4096×2312** | BGRA8888 / 1 / NO_LOD | opaque art | hero page / spotlight background (the likely "banner") |
| gun | `guns/lash_gun_psd.vtex_c`, `guns/punkgoat_gun_psd.vtex_c` | 600×300 | BGRA8888 / 1 / NO_LOD | cutout | hero stats weapon image |
| **logo**, logo_localized | `hero_names/lash.vsvg_c`, `lash_localized.vsvg_c`, `billy.vsvg_c`, `billy_localized.vsvg_c` | viewBox 1364×549 (Lash), 1253×520 (Billy) | SVG (white title lettering "LASH"/"BILLY") | n/a | hero page header, loading — shows the NAME as art (candidate "banner"; English one used when `citadel_english_hero_names`) |

Also hero-specific but not 2D art: `maps/ui/hero_prefabs|hero_shop|team_reveal_hero|hero_postgame_portrait/<codename>*.vmap`
render the hero **model** (already covered by the skin), `panorama/images/hud/abilities/<c>/*` are ability icons (no likeness),
`hud/crosshair/reticle*_punkgoat_psd` is the reticle. Uncompiled sources of the Lash 7 portraits: `work/portraits/out/`.

## 2. PNG → vtex_c by clone-and-swap (`work/portraits/build_vtex.py`)

`resourcecompiler` on `panorama/image_compiler.vdata` works headless (`-i .../panorama/image_compiler.vdata` with the usual flags;
list entries `panorama:"file://{images}/heroes/lash_card.png"`, PNGs in `content/citadel_addons/<addon>/panorama/images/heroes/`) but its
output differs from the originals: PNG-in-vtex (format 16, 1 mip, flag 0x80 dilate), the 120×200 vertical comes out as a 128×256 DXT5,
BC7 sources lose their alpha. So we **clone an original vtex_c of identical size and replace only the pixels**:

```
python -I work/portraits/build_vtex.py <template.vtex_c> <source.png> <out.vtex_c> [mips flags]
# templates: the ORIGINAL game vtex_c extracted with  VRF -i pak01_dir.vpk -f panorama/images/heroes/... -o <existing dir>
```
- Layout: resource header | RED2 | DATA | mips **smallest first, largest last**, BGRA8888, straight alpha (VRF-decoded PNG RGBA ↔ BGRA byte swap, no premultiply).
- Rewrites reflectivity, the 16×16 fallback-bits extra data and the mip chain (area filter on premultiplied alpha, sizes `floor(w/2)`, min 1).
  Level 0 is bit-exact; deep mips (<35 px) differ slightly from Valve's filter (irrelevant, slots are NO_LOD).
- Size must equal the template (assert). Output named exactly like the original (`…_psd.vtex_c`, `…_psd_8b6cde8.vtex_c`).
- BC7 slots (`sm_hash`): no BC7 encoder in our chain → clone a BGRA8888 sibling (`…_sm_psd.vtex_c`) with `mips=8 flags=0` (ships as BGRA with a mip
  chain; alpha exact). Round trip verified; in-game acceptance of that exact variant is unconfirmed.
- Verify: `VRF -i <dir> -o <existing out dir> -d` and compare decoded PNGs to the sources (`exact` expected). Done for all 7 Lash portraits.
- `bg` (4096×2312, ~38 MB raw) and `gun` use the same BGRA8888/1 mip/NO_LOD template path → `build_vtex.py` works as is.
  Logos are **vsvg_c**: not attempted; options are compiling an `.svg` with resourcecompiler or cloning the original `vsvg_c` wrapper and swapping
  the SVG payload (untested). The *text* name is independent of these (see §3).
- Pack under `panorama/images/heroes/…` (`-PackInclude 'panorama/images/heroes/*'`); never pack `image_compiler.vdata_c`.

## 3. Hero names (localization)

**Where names live.** Loose game files (not in pak01): `game/citadel/resource/localization/<category>/<category>_<lang>.txt`
(UTF-8 BOM, CRLF, KeyValues `"lang" { "Language" "english" "Tokens" { "key" "value" … } }`). Hero display names are in
`citadel_gc_hero_names/citadel_gc_hero_names_english.txt`:

| token | meaning |
|---|---|
| `hero_<codename>:n` | the name everywhere in UI (HUD, select, scoreboard, kill feed, shop) |
| `hero_<codename>_search:n`, `hero_<codename>_sort:n` | hero-select search / sort key (`_sort` exists for some heroes, e.g. punkgoat) |
| `Steam_RP_hero_<codename>` | Steam rich presence |

Codenames: `lash`, `punkgoat` (= Billy). No other file has the bare name as a token; lore/ability text (`citadel_heroes_english.txt`, "Chained to Billy", …)
mentions it in prose and is left alone. Uppercase/short forms are produced by CSS, there are no extra tokens. Other languages keep their defaults.

**How mods override it.** `gameinfo.gi` search paths put `citadel/dmm_localization_default`, then `citadel/addons` (all mod VPKs) before `citadel`, so a file at
the same relative path inside a mod VPK shadows the loose game file **as a whole** (no engine-side merge). Two mods shipping that file would
clobber each other — except through **Deadlock Mod Manager's compatibility overlay**: it parses every enabled mod's
`resource/localization/*.txt`, diffs each token against the installed game file, ignores unchanged tokens, groups changed tokens per
`(file, token)` and writes the merged result into its overlay VPK (`dmm_localization_default/pak01_dir.vpk`). Different tokens from different mods
therefore merge; the same token conflicts (priority/user choice). The English hero-name and hero-text files are "protected": tokens equal to *past* vanilla
values are ignored (stale snapshot protection). A partial file works with DMM (it restores omitted game text) but hides all other English text without it — so
we ship a **full copy** of the current game file with only our hero's tokens changed.

**Build** (`scripts/make_hero_names.py`):
```
python -I scripts/make_hero_names.py --codename lash     --name "Night Elf Lash" --out build/names/lash
python -I scripts/make_hero_names.py --codename punkgoat --name "Orc Billy"      --out build/names/punkgoat
python -I scripts/vpk_tool.py add <pak_before.vpk> <pak_after.vpk> "resource/localization/citadel_gc_hero_names/citadel_gc_hero_names_english.txt=build/names/<hero>/resource/localization/citadel_gc_hero_names/citadel_gc_hero_names_english.txt"
```
(or `-PackInclude 'resource/localization/*'` with the file placed under the addon's game dir). Regenerate after each game update — it is a snapshot.
Result in shipped zips: `resource/localization/...` in `pak77` (Night Elf Lash, both variants) and `pak78` (Orc Billy). VPK-only checks done (verify passes); the in-game
display of the merged names is to be confirmed by the user. Without DMM, whichever pak has the lower number wins and the other hero keeps its old name.

DMM diagnostics: `%LOCALAPPDATA%\dev.stormix.deadlock-mod-manager\logs` — look for `Prepared merged mod-data overlay for launch: N localization tokens`
(was 0 before the name files) and `Built repair plan for N mods`; overlay state in `...\compatibility\` and `game/citadel/dmm_localization_default/compatibility.json`.
DMM source (reference): `github.com/deadlock-mod-manager/deadlock-mod-manager`, `apps/desktop/src-tauri/src/mod_manager/localization_overlay.rs`.

## 4. Checklist for a new hero
1. Run the inventory script; look at `contact_sheet.png`; decide which slots to replace (cards/vertical/sm/mm minimum; bg + logo for a full "banner" change).
2. Render PNGs at the exact sizes/alpha conventions (`06a-portraits.md`); `build_vtex.py` each into `…/panorama/images/heroes/<original name>.vtex_c`.
3. `make_hero_names.py` → add to the pak. 4. `VRF --vpk_verify`, decode the packed vtex_c back and compare. 5. Zip, user tests in DMM.
