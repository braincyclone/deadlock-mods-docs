# 02 — Re-rigging the WoW night elf onto Lash

Goal: drive a WoW character mesh with Lash's skeleton and animations, without touching Lash's bones.
Everything runs headless in Blender 5.1. Scripts live in `work/rig/scripts/`; outputs go to `work/rig/`.

```
BLENDER="C:\Program Files\Blender Foundation\Blender 5.1\blender.exe"
"$BLENDER" -b <file.blend> --python <script.py> -- <args>
```

Paths given to Blender or to the renderer must be **absolute**. Relative render paths fail silently.

## Inputs

| File | What it is |
|---|---|
| `work/nightelf/nightelf_f.blend` | Naked night elf (wow.export glTF). Contains about 400 WoW actions. |
| `work/nightelf/nightelf_armor.blend` | Bloodfang armored export, made by `import_armor.py` (below). |
| `work/nightelf/raw/nightelffemale_hd.m2`, `*_1839627.skel` | Raw WoW model; holds the attachment points. |
| `work/lash/lash.blend` | Lash armature (171 bones), `lash_body`, `sparkles`, and Lash's actions. |
| `work/weapons/gltf/item/objectcomponents/weapon/*.gltf`, `work/weapons/raw/*/*.m2` | Weapon glTFs and raw M2s. |

Orientation as imported:
- **WoW:** faces +X, her right is −Y, Z up, about 2.28 tall.
- **Lash:** faces −Y, his right is −X, Z up, about 2.64 tall.

WoW bones are named `bone_<X>`, and each has a `bone_<X>_p` pivot parent. The glTF bone tails are meaningless (they all point +Z), so the rig never relies on bone orientation; it works from joint **positions**.

## Run order

```
# 0. (armored only) import the re-exported glTF into a .blend
"$BLENDER" -b --python work/rig/scripts/import_armor.py -- <abs>/work/nightelf/export/character/nightelf/female/nightelffemale_hd.gltf <abs>/work/nightelf/nightelf_armor.blend
# 1. glove recolor texture (armored builds)
python work/rig/scripts/recolor_glove.py
# 2. build: naked / armored / armored + weapons
"$BLENDER" -b work/nightelf/nightelf_f.blend     --python work/rig/scripts/build_rig.py -- <abs>/work/rig/nelash_rigged.blend
"$BLENDER" -b work/nightelf/nightelf_armor.blend --python work/rig/scripts/build_rig.py -- <abs>/work/rig/nelash_armored.blend glove=<abs>/work/rig/textures/ne_glove_color.png
"$BLENDER" -b work/nightelf/nightelf_armor.blend --python work/rig/scripts/build_rig.py -- <abs>/work/rig/nelash_bf_weapons.blend glove=<abs>/work/rig/textures/ne_glove_color.png weapons
# 3. verify (see "Verification")
"$BLENDER" -b work/rig/nelash_armored.blend --python work/rig/scripts/render_check.py -- rest anim close palette out=<abs>/work/rig/renders_armored
```

`build_rig.py` arguments, all after `--`:
- **`<out.blend>`** (first argument): the output file.
- **`glove=<png>`**: builds the recolored `ne_glove` material.
- **`weapons`**: runs `add_weapons.py`.

The build is deterministic. Each variant is rebuilt from its source; never edit an output .blend by hand.

Pipe the output to a log (`... 2>&1 > work/rig/build_log*.txt`). It contains the `FIT` lines, the mapping table, the `WEAPON` and `ANCHOR` records, and the image and object checks.

## How the fit works (`build_rig.py`)

1. **Load.** Open the night elf file and delete `Icosphere` (glTF bone-shape junk; it is also re-imported with Lash, so it is removed again at the end). Append Lash's objects and **all** of Lash's actions with `use_fake_user=True`, otherwise orphaned actions are dropped on save. Clear Lash's assigned action and pose so he is in rest pose.
2. **Global transform `G`.** Rotate −90° about Z, so +X becomes −Y and she faces −Y like Lash. Scale uniformly by `S_GLOBAL` = Lash `arm_upper_R` z / WoW `ArmR` z ≈ 1.088.
   - Shoulder height was chosen because it gives the least distortion along the whole chain. Matching head height instead stretches the neck 1.8×.
3. **Per-segment fit transforms.** For every mapped Lash segment (head joint → tail joint):
   - The transform is `p' = hl + R·S·(p − hw)`. `hw` and `hl` are the WoW and Lash joint heads. `R` is the rotation taking the WoW segment direction onto Lash's.
   - `S` stretches along the bone axis only, by Lash length / WoW length. Cross-section is preserved, so she stays slender.
   - Why not pose bones and apply the modifier? WoW bone orientations are junk, and doing it ourselves gives full control: chained roll, the hand frame, per-vertex twist and normals.
   - **Spine, chest, clavicles, thighs:** minimal rotation.
   - **Forearm and shin:** parallel-transported from the upper segment (`rmin(R_parent·dW, dL)·R_parent`). This keeps the elbow and knee hinge consistent.
   - **Hand:** full frame match. The axis runs from the hand to the middle knuckle; the side vector runs from the index knuckle to the pinky knuckle.
   - **Forearm twist:** the wrist twist residual (about 19°) is shared across the forearm keys: `larm` 0, `larm_tw` 0.5, `larm_tw1` 0.85 (`FOREARM_TWIST`).
   - **Fingers:** chained from the hand. The WoW fingertip segment's end point is estimated from its vertices (`ext_end`).
   - **Foot and toe:** frame match with world Z as up, so the sole stays flat.
   - **Head:** **rigid**, rotation = identity, so the face stays upright. Neck stretch is capped at `NECK_STRETCH_MAX=1.35`. Lash's head joint is very high; without the cap she gets a giraffe neck. As a result her head pivot sits about 4 cm under Lash's `head`.
   - **Pelvis:** rigid, with an X widen of `(hipL/hipW)^0.5` ≈ 1.15.
4. **Blend.** The transforms are combined per vertex with the WoW skin weights, normalised, in numpy (linear blend skinning). Custom split normals are transformed by the inverse-transpose of the blended 3×3 and written back with `normals_split_custom_set`. Skipping that gives wrong shading on rotated limbs.
5. **Weights.** WoW groups are remapped to Lash bones (tables below), the splits are applied, then each vertex is cut to its 4 largest influences and renormalised.
6. **Bind.** Remove the WoW armature modifier and the WoW armature. Join all night elf meshes into `nelash_body`, parent it to Lash's armature and add an Armature modifier. Transforms stay identity.
7. **Glove.** In `lash_body`, keep only the faces with material `lash_glove`, giving object `lash_glove` (right gauntlet, about 10k verts, keeps Lash's weights). Delete the rest of Lash's body; keep `sparkles`.
8. **Materials.** `MAT_RENAME`: `data-1`→`ne_body`, `data-6`→`ne_hair`, `data-19`→`ne_eyes`, `data-9`→`ne_blank`, `nightelffemale_hd_3621913`→`ne_circlet`, `..._eyereflect`→`ne_eyeglow`, helm→`ne_helm`, pads→`ne_shoulder`. The log warns on any `.0xx` suffix.
9. **Save.** Images are packed. The output has exactly one armature, Lash's, in rest pose, with no bones renamed or moved.

## Bone mapping (WoW `bone_` names → Lash). Shown for R; L mirrors (WoW `EXP_L1_*`, `*L`, 28/44)

| WoW | Lash | Note |
|---|---|---|
| Root, EXP_C1_Pelvis1, LoinBk1-3, Buckle, 185, 4, 5, 7, 10, 15 | pelvis | LoinBk and 4/10/15 are the tabard flaps (see splits) |
| SpineLow | spine_0 | |
| EXP_C1_Spine2, EXP_C1_Cape2 | spine_1 | Cape2 = lower back hair, uses the `chest` fit |
| Chest | **split** spine_2 / spine_3 | linear blend over Lash spine_3 head z ±0.08 (`CHEST_BAND`) |
| 23, 26 (breast helpers), EXP_C1_Cape1 | spine_3 / spine_2 | `chest` fit |
| EXP_C1_Neck1, Neck2 | neck_0 | |
| Head and every face, eyelid and ear bone (66/127/147, 72/129/149, hair chains 54/55/57/91/94/95…) | head | ears and hair are rigid |
| EXP_R1_Eye1 | **head** (not eye_0_R) | Lash's eye pivots are not in her sockets; NE eyes are glowing blanks |
| Jaw, 113–122 | jaw_0 | |
| ShoulderR | clavicle_R | stretched about 1.9× to reach Lash's wide shoulders |
| ArmR / EXP_R1_Arm1Twist2 / Arm1Twist3 | arm_upper_R / _TWIST / _TWIST1 | |
| ElbowR / EXP_R1_Arm2Twist2 / Arm2Twist3 | arm_lower_R / _TWIST / _TWIST1 | |
| HandR | hand_R | |
| ThumbR, <X>FingerR | finger_thumb_0_R, finger_<x>_0_R | |
| ThumbTipR, <X>FingerTipR | **split** finger_<x>_1_R / _2_R | by projection along Lash's finger, band ±0.15 |
| EXP_R1_Leg1Twist1 | leg_upper_R | |
| EXP_R1_Leg1Twist3 | leg_upper_R_TWIST1 | weighted verts lie just above the knee |
| EXP_R1_Leg2Twist1 | leg_lower_R | |
| 27 (ankle helper), FootR | ankle_R | 27 uses the `shin` fit |
| 43 | ball_R | |
| any unlisted weighted bone | nearest mapped ancestor (`resolve`) | full table printed in the build log |

Splits and blends (post-fit, by fitted vertex position):
- **Chest:** Lash `spine_3` z ± `CHEST_BAND`.
- **Finger tips:** split between `_1` and `_2` along the Lash finger.
- **Tabard / loincloth** (`TABARD_BONES`): WoW skins these flaps to the pelvis only. Their lower part blends up to `TABARD_LEG=0.7` onto `leg_upper_L`/`leg_upper_R`, over `TABARD_LEN=0.45` below the hip, split left/right by x (`TABARD_XBAND`). Without this, the legs punch through the flaps in run and kick.

## Rigid items (helm, shoulder pads)

wow.export leaves item meshes **unplaced**: they sit at the origin in item space, with no node transform. Their placement comes from the M2 attachment data:

```
python work/rig/scripts/m2_attachments.py work/nightelf/raw/nightelffemale_hd.m2 work/nightelf/raw/nightelffemale_hd_1839627.skel <gltf>
```

- The HD model keeps attachments in the `.skel` `SKA1` chunk (40-byte records).
- Each attachment bone index maps to the glTF non-`_p` joint order.
- `build_rig.py` `ITEMS`:
  - Helm `Head_*` → att 11, bone `216` (−0.0538, 0, 2.0415).
  - Pads `Shoulder (R)`/`(L)` → att 5/6, bones `208`/`205` (−0.0943, ∓0.151, 1.9463), children of ArmR/ArmL.
- Items get 100% weight on the attachment bone, which resolves to `head` / `arm_upper_R/L`.
- Pads on `arm_upper` is WoW's own behaviour. The clavicle fit would stretch them 1.9×.

## Weapons (`add_weapons.py`, `weapon_lib.py`, `weapon_tweaks.json`)

- **Import:** `weapon_lib.import_weapon` imports the glTF, keeps the rest shape, and strips the armature and bone groups.
- **Split:**
  - Solid geometry → `<p>_blade`: `tf_blade` (material `tf_blade`), `pd_blade` (`pd_blade`).
  - Effect cards, identified by texture `zap1`/`blue_glow2`/`lightningball`/`glow`/`ember`, → `<p>_fx` with materials `tf_zap`, `tf_glow`, `tf_ball`. These are additive approximations: emission mixed with transparent by luminance.
  - The Perdition dagger glTF has **no** fx geometry; its fire was 2 M2 particle emitters.
- **Bones:** sword → `spine_3`, dagger → `pelvis`, 100% weight.
- **Anchors:** empties `fx_tf`, `fx_pd` (mid-blade) and `fx_pd_guard` (item X 0.38). Each has X pointing at the tip and is parented to the bone.
  - Bone-local transforms are printed in Source terms: the glTF node frame of `work/lash/gltf/lash.glb`, which equals the Source bone frame; inches = m × 39.37; QAngle pitch/yaw/roll.
  - Lash's Blender bone axes are **not** Source frames, because of the importer bone heuristic. Never report Blender bone-local values.
- **Placement modes**, per weapon in `weapon_tweaks.json`:
  - `place`: art-directed rest-pose `start` (pommel) → `end` (tip) in Lash metres. This sets the scale. `normal` aims the blade's flat (item +Y).
  - No `place`: WoW sheath. Uses the attachment position + rotation key + 0.95 scale from the `.skel`; see the `weapon_lib.SHEATHS` docstring for att 26 back / 32 left hip. It is carried into Lash's frame by the rigid part of the bone's fit (`FITR`), then optional `lean`/`offset`/`rot`/`loc`.
- **Current values are true WoW scale:** fractions of body height from `measure_wow_weapons.py`, sword 1.3016 / dagger 0.5365 of the WoW elf height, × fitted height 2.5555. Directions follow the WoW sheath with the backward stand-off removed.
- **Gotcha:** a diagonal sword's vertical span (about 1.07× height) looks much shorter than its true length (1.30×).
- **Ground truth check:** `preview_weapons_wow.py` (WoW-native placement on the unfitted elf) plus `render_check.py ... wowcompare`, combined in `work/rig/renders_weapons/wow_vs_ours.png`.
- **Dagger fire emitters:** read with `python work/rig/scripts/m2_bones.py <weapon.m2>` (M2Particle records are 492 bytes in v272).

## Glove recolor (`recolor_glove.py`)

`work/lash/textures/lash_glove_color.png` → `work/rig/textures/ne_glove_color.png`.
- **Region split:** "redness" (r−g)/max. Brown leather → crimson, darkest folds → black. Olive metal → dark brass, with contrast amplified because the metal is flat.
- **Material:** `build_rig.py` copies `lash_glove` to `ne_glove` and **removes the Emission link**. The importer wired Source2's mask into emission, which washes everything pink. It also sets specular to 0.2 and puts a floor of 0.55 on roughness.

## Verification (`render_check.py`)

`"$BLENDER" -b <rig.blend> --python work/rig/scripts/render_check.py -- <modes> out=<abs dir>`

| Mode | Output | Look for |
|---|---|---|
| `rest` | `rest_sheet.png`: front/side/back with Lash ghosted, plus skeleton overlay | joints inside her limbs, head/neck plausible, glove on her hand |
| `hands` | `hands_sheet.png` | fingers inside the glove, left hand natural |
| `anim` | `anim_<action>.png` (run, flog, melee, hero_pose, jump, idle; 4 frames, 3/4 view + side) | tearing, candy-wrapper wrists, collapsing shoulders/hips, detached head or hair |
| `close` | `closeups_sheet.png` | shoulders with arms raised, groin on kicks, tabard in run, face |
| `palette` | `palette_sheet.png` | glove vs armor colors |
| `wclose` | `weapons_closeups.png` | sword vs pads/hair/arms, dagger vs thigh |
| `wowcompare` | `tmp/ours_back/side.png` | same relative framing as the WoW-native preview |

Read every PNG yourself. Image sheets are about 1.5k tokens each.
- **Actions:** assigning one needs `ad.action_slot = act.slots[0]` (Blender 4.4+ slotted actions).
- **Framing:** the animation sheets frame on the evaluated bounding box of body + glove + weapons.

## Known defects / fixes

| Symptom | Cause | Fix / status |
|---|---|---|
| Giraffe neck | Lash's head joint is very high | `NECK_STRETCH_MAX` (head pivot about 4 cm low, accepted) |
| Pink/grey glove | emission link from the import | removed in the `ne_glove` build |
| Legs through tabard | flaps pelvis-only | tabard thigh blend |
| Fist into face (flog), ears clip arms, pads cross face with arms raised | proportions / rigid items | accepted |
| Sword near left arm / pads at true size | WoW scale | accepted; tweak `weapon_tweaks.json` |
| Missing actions after save | no users | `use_fake_user=True` |
| Weird hand roll | minimal-rotation hand | hand frame match (index−pinky side vector) |

## Other scripts

- `inspect_ne.py` / `inspect_lash.py`: bone trees and per-group weight centroids.
- `inspect_gltf.py`: glTF meshes and materials.
- `measure_body.py`: back and side surface profiles, used for weapon placement.
