# 06a — Hero HUD portraits

Goal: replace Lash's HUD/hero images with portraits of the night elf.
Output is PNGs only; compiling and packing is a separate step.
Scripts live in `work/portraits/scripts/`.

## 1. Find the slots

```
tools/vrf/Source2Viewer-CLI.exe -i "C:/Program Files (x86)/Steam/steamapps/common/Deadlock/game/citadel/pak01_dir.vpk" --vpk_list > work/vpk_full_list.txt
grep 'panorama/images/heroes/lash' work/vpk_full_list.txt
tools/vrf/Source2Viewer-CLI.exe -i <pak01_dir.vpk> -f "panorama/images/heroes/lash_card_psd.vtex_c,..." -d -o work/portraits/orig
```

- There is no `--vpk_filter`. Use `-f` with a comma-separated path list.
- Exported PNGs land in `work/portraits/orig/panorama/images/heroes/`.

| VPK file (`panorama/images/heroes/`) | Use | Size | Alpha / framing |
|---|---|---|---|
| `lash_card_psd.vtex_c` | HUD portrait | 280×380 | figure cutout, bottom edge opaque, head 3/4 facing screen-left |
| `lash_card_critical_psd.vtex_c` | low HP | 280×380 | same (Lash: bruised, grimace) |
| `lash_card_gloat_psd.vtex_c` | kill streak | 280×380 | same (Lash: smirk + gold ✦ sparkles) |
| `lash_vertical_psd.vtex_c` | hero select | 120×200 | fully opaque, colored backdrop |
| `lash_sm_psd.vtex_c` | kill feed | 128×128 | head sticker, thick dark outline |
| `lash_sm_psd_8b6cde8.vtex_c` | faded variant | 128×128 | no outline, semi-transparent (≤0.4), lightened |
| `lash_mm_psd.vtex_c` | minimap | 50×50 | head sticker, outline |
| `backgrounds/lash_bg_psd.vtex_c`, `guns/lash_gun_psd.vtex_c` | not done | | |

- Every hero has the card / critical / gloat / sm / mm / vertical set.
- **There is no dead slot.** Our `lash_card_dead.png` is an extra; its look is reused for `lash_sm_8b6cde8`.

## 2. Compile convention (Divine Skins wiki, CSDK 12)

- Duplicate the `hero_image_template` addon and delete its sample (Grey Talon) images.
- PNG goes in `content/citadel_addons/<addon>/panorama/images/heroes/`, **named after the original without `_psd`**: `lash_card_psd.vtex_c` → `lash_card.png`. Exact original size; transparent PNG.
- Add one line per image to `panorama/image_compiler.vdata`: `panorama:"file://{images}/heroes/lash_card.png"`.
- Asset Browser → `image_compiler` → Recompile → Full Recompile.
- Copy the compiled `.vtex_c` files to staging and **rename them to the exact original names** (`lash_card_png.vtex_c` → `lash_card_psd.vtex_c`). Never pack `image_compiler.vdata_c`.
- The hashed file `lash_sm_psd_8b6cde8` has no documented naming; we name it `lash_sm_8b6cde8.png` and rename the output to the original.

## 3. Render (`portrait_render.py`)

Always render from the **WoW-native** `work/nightelf/nightelf_armor.blend`. It has the full face rig; the Lash-rigged mesh's face is rigid.

The script:
- appends the 400 WoW actions from `nightelf_f.blend`;
- places the helm and pads at their attachment bones (`bone_216`, `bone_208`, `bone_205`);
- bakes the pose at the chosen frame, then applies extra `bone_rot`.

```
# test grid: spec = "Action name prefix|frame[|FacePose action][|bone:x,y,z/bone:x,y,z]"
"$BLENDER" -b work/nightelf/nightelf_armor.blend --python work/portraits/scripts/portrait_render.py -- grid <abs>/work/portraits/grid1.png "EmoteRoar (ID 74|15;Stand (ID 0 variation 0)|10||bone_Head:0,0,-25"
python work/portraits/scripts/grid_sheet.py work/portraits/grid1.png.txt work/portraits/grid1.png 6
# finals (config work/portraits/scripts/portraits.json -> work/portraits/raw/<name>_raw.png)
"$BLENDER" -b work/nightelf/nightelf_armor.blend --python work/portraits/scripts/portrait_render.py -- final <abs>/work/portraits/scripts/portraits.json <abs>/work/portraits/raw
```

Current picks:

| Portrait | Action / frame | Extra |
|---|---|---|
| normal | `Stand (ID 0 variation 0)` f10 | `bone_Head` Y+10 (turn to camera) |
| critical | `EmoteRoar (ID 74)` f15 | open-mouth roar, red rim |
| gloat | `EmoteLaugh (ID 70)` f5 | head Z+14 (chin up), Y+8 |
| dead | `Stand` f10 | head Z−20, neck Z−10, chest Z−5; cool, dim lights |

Gotchas:
- **Head bone axes:** local Z = pitch (−Z bows the head), local Y = yaw. glTF bones are Y-up, not world-aligned.
- **`FacePose` actions do nothing visible.** Expression comes from emote frames only. The blindfold hides the eyes, so mood must read from the mouth and head angle.
- **NLA strips use `extrapolation='HOLD'`**; `extend_mode` does not exist.
- **Camera:** az 45° (her left-front), so her face turns screen-left like Lash's. 85 mm lens, dist about 1.8. Head-only icons set `show_only`.
- **Lighting:** Eevee with AgX + "Medium High Contrast" (Standard blows out the white hair). Warm key at front-left-above, two red rims from behind, weak cool fill, near-black world.

## 4. Post (`post.py`)

`python work/portraits/scripts/post.py` (needs `pip install --user opencv-python-headless`).

Steps:
1. Bilateral ×2 + `edgePreservingFilter`, plus a 25% soft posterise, then unsharp. This gives the painted look.
2. Per-state grade:
   - normal: warm
   - gloat: warm and brighter, plus 3 sparkles
   - critical: saturation ×1.15, hard contrast, red tint, red vignette
   - dead: 22% saturation, cool, ×0.9
3. Darken toward the silhouette edge.
4. Resize with LANCZOS to the exact slot size.
5. Icons:
   - `sm`/`mm`: head fitted by height so the long ear can be clipped at the edge (`max_w_ratio=1.0`), then a dark outline from the dilated alpha (4 px / 2 px).
   - `vertical`: composited on a crimson gradient backdrop (opaque, like the original).
   - `sm_8b6cde8`: dead head, no outline, lightened, alpha × luminance × 0.85.

Outputs go to `work/portraits/out/<basename>.png`, plus `_sheet.png` (originals on top, ours below, on a checkerboard). Check the sheet every iteration.

## Checklist

- [ ] Sizes match the originals exactly (`post.py` prints them).
- [ ] Alpha style per slot (cards cutout + opaque bottom, vertical opaque, icons outlined).
- [ ] Face reads at 50 px and 128 px.
- [ ] Each state is distinct at a glance (normal / red angry / smug + sparkles / grey bowed).
- [ ] Rename the compiled `.vtex_c` files to the original names before packing.
