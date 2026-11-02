# 03 — Export, vmdl, compile, pack, verify

Input: a Blender file with the hero's **unchanged armature** and your meshes weighted to it (see `02-rigging.md`).
Output: `build/<mod>/pakNN_dir.vpk` + `build/dmm/<Mod>.zip`.

> `app/pipeline/forge_pipeline` (`stages.py`, `content.py`, `vmdl.py`, `vpk.py`, `blender/export_dmx.py`) is the
> generalized, automated version of every step below, driven by hero profiles (`app/pipeline/heroes/*.json`).
> This document describes the hand-built scripts it was generalized from; the gotchas apply to both.

## Scripts (repo-root relative)

| Step | Script | Notes |
|---|---|---|
| DMX export | `work/export/scripts/export_dmx.py` | Blender, headless; config JSON in `work/export/configs/` |
| Content assembly | `work/export/scripts/assemble_content.py` | copies DMX/PNG, writes vmats, calls `build_vmdl.py` |
| vmdl | `work/export/scripts/build_vmdl.py` | edits the VRF-decompiled original vmdl |
| Compile + AG2 + pack | `scripts/compile_and_pack.ps1` | resourcecompiler, DeadlockTools, VPK writer |
| Edit existing pak | `scripts/vpk_tool.py` | list / add / rm / build VPK v2 (pure Python) |
| Ref check | `work/export/scripts/check_refs.py <extracted dir>` | every `_c` ref must be in the pak or base game |
| Render verify | `work/export/scripts/render_verify.py` | VRF glb → Eevee front/side/back (+ attachment markers) |
| Billy variants | `work/billy/scripts/*` | parameterized copies of the above for punkgoat |

Base-game path list used for "already in the game" checks: `build/diff/pak01_list.txt`
(`tools/vrf/Source2Viewer-CLI.exe -i <pak01_dir.vpk> -l > build/diff/pak01_list.txt`).

## 1. One-time prerequisites

- Blender addon **BlenderSource2Tools (Fuji fork)** `io_scene_valvesource2` 3.3.1_2f: unzip the release
  (`github.com/FellOffFuji/BlenderSource2Tools/releases`) into
  `%APPDATA%\Blender Foundation\Blender\5.1\scripts\addons\io_scene_valvesource2\`; enable in scripts with
  `addon_utils.enable('io_scene_valvesource2', default_set=True, persistent=False)`.
- Hero skeleton JSON (`work/export/lash_skeleton_local.json`, from `make_skeleton_json.py` / the original compiled model):
  `VRF -i <orig vmdl_c> -b DATA` → `m_modelSkeleton.m_boneName / m_nParent / m_bonePosParent / m_boneRotParent`
  (local, parent-relative, quaternion `[x,y,z,w]`).
- Decompiled original vmdl: `VRF -i pak01_dir.vpk -f models/heroes_wip/<hero>/<hero>.vmdl_c -d -o <abs dir>`
  (**create the output dir first**; with an exact single-file `-f` and a missing dir, `-o` is treated as a *file*).
- A hero tag `-f` takes a comma-separated list of exact paths or prefixes; there is no `--vpk_filter`.

## 2. DMX export (`export_dmx.py`)

```
"%BLENDER%" -b work/rig/<file>.blend --python work/export/scripts/export_dmx.py -- \
   <skeleton.json> <out/dmx> <out/derived.blend> <out/png> work/export/configs/<variant>.json
```
The input .blend is never saved. The script works on a derived scene and:

1. Rotates the meshes **+90° about Z** and scales **×39.37** (glTF-import space is meters facing −Y; Source is inches facing +X)
   with `mesh.transform()` — never object scale.
2. Overwrites every bone's rest matrix with the **original compiled skeleton** (`edit_bone.matrix = world`, length ≥ 1 in,
   `use_connect=False`). Bone orientation matters: animations store absolute local rotations, so a different bone roll
   makes every animation play wrong. Check the printed `MAX head deviation` (≈0.0003 in is right).
3. Keeps ≤ 4 weights per vertex, drops the 2nd UV layer, deletes faces of `delete_materials` (e.g. a hidden quad).
4. Renames materials to **full VPK paths with `.vmat`** (`materials/models/heroes_wip/lash/nelash_bf/ne_body.vmat`) — the DMX
   `mtlName` is used verbatim, so no ModelDoc material remap is needed. Hero-original materials keep the original path
   (`models/heroes_wip/lash/materials/lash_glove.vmat`) and are never packed.
5. Saves each material's first image as `<name>_color.png`.
6. One collection per mesh object (object + armature) → one `.dmx` each. Settings: DMX **Binary 9 / Model 22**
   (`dmx_format = '22'`, not `22_modeldoc`), `export_path` = out dir, `material_path = ''`.
7. Prints `ANCHOR` lines for empties (attachments): position (in) + QAngle relative to their parent bone — copy these into
   `--attach=` for the vmdl. Empties are not exported as geometry. Capture empty matrices **before** the armature edit
   (bones change → parented empties move).

Config keys: `mat_prefix`, `body_materials`, `glove` (blender name or null), `delete_materials`, `extra_objects`,
`extra_materials` (blender name → full vpk path).

## 3. vmdl (`build_vmdl.py`) and content (`assemble_content.py`)

`assemble_content.py <repo> --export-dir <dir> --matdir <mat folder> --meshdir <dmx folder> [--vmats vmats.json] [--glove-vmat NAME]
 [--extra-mesh=NAME ...] [--attach=NAME:BONE:ox,oy,oz:pitch,yaw,roll ...] [--particle=ATT:particles/x.vpcf[:point_follow] ...]`

`build_vmdl.py` starts from `work/lash/decomp/.../lash.vmdl` (VRF decompile) and:

- replaces `RenderMeshList` (+ extra meshes) and `LODGroupList` (one group is enough; the compiler warns "ignoring" — harmless);
- `--attach`: adds `Attachment` nodes (`parent_bone`, `relative_origin` inches, `relative_angles` = Source **QAngle**
  pitch,yaw,roll; X axis = forward). Verified: compiled quaternion == `AngleQuaternion(pitch,yaw,roll)`;
- `--particle`: adds `GenericGameData` with `game_class = "particle"` → compiled `particles_list` (see `05-vfx.md`);
- restores `TaggedSoundSettings_t` (`scripts/tagged_sounds/<hero>_anim_sounds.vdata`) that the decompile drops
  (GenericGameData, `game_class = "TaggedSoundSettings_t"`, `m_taggedSounds = [ resource:"...", resource:"" ]`).

Things the CSDK 12 compiler **rejects / cannot do** (don't retry):
- `NmSkeletonList` / `AnimGraph2List` nodes ("Failed to allocate an instance of class") → stripped, then re-added by DeadlockTools;
- `AttachmentCamera` / `AttachmentCameraPreview` nodes (`AttachmentCameraData`: gunaim/near/far/portrait cameras) → lost.
  Deadlock Mod Manager's compatibility overlay restores the camera interface at launch (see §8); `--cams` flag kept for experiments;
- cloth / `m_pFeModel`, `DSTF` block and `root_motion` in PHYS are lost in the decompile → no cloth.

## 4. Compile (CSDK 12 resourcecompiler, headless)

```
cd tools/csdk12/Reduced_CSDK_12/game/bin/win64
resourcecompiler.exe -danger_mode_ignore_schema_mismatches -i "<abs>/content/citadel_addons/<addon>/<path>/x.vmdl" -game "<abs>/game/citadel" -f
```
- **Use `game/bin/win64`**, and **always** `-danger_mode_ignore_schema_mismatches` (the tools mix Aug/Dec 2025 DLLs; without it:
  `ParticleFloatType_t Mismatch`). `-f` forces; wildcards (`*.vmat`) and `-r` work. Input paths absolute with forward slashes.
- Output lands in `game/citadel_addons/<addon>/<same relative path>_c`. Addon dirs are plain folders (we use `nelash`, `nelash_fx`, `nelash_ui`).
- **Compile materials first, then the model.** A material compile fails (and stops the model) if a texture source PNG/TGA is
  missing in the addon content (`Unable to read file ...`); base-game textures referenced by *path* only need to exist in the game.
  The "referencing missing material … lash_glove.vmat" warnings for hero-original materials are expected.
- Vpcf compiled by this tool must be `vpcf63` (see 05); material `DynamicParams` expressions are validated at compile time.
- Compile failures are non-fatal for the shell — `compile_and_pack.ps1` now throws on a non-zero exit so stale output isn't packed.
- Default textures referenced as `materials/default/*.tga` compile to hashed `materials/default/*_tga_<hash>.vtex_c`; the CSDK's hash
  can differ from the game's (e.g. `default_mask_tga_2aa88af4` is not in pak01) → pack it (`-PackInclude`), the rest are in the base game.

## 5. AnimGraph2 (`DeadlockTools`)

```
tools/deadlocktools/DeadlockTools.exe add ag2 <compiled lash.vmdl_c> --hero lash --hero-folder heroes_wip
```
Adds `m_animGraph2Refs` (`animgraphs/animgraph2/hero/hero.vnmgraph+<hero>.vnmgraph`, `..._ui...`, identifier `"ui"`),
`m_vecNmSkeletonRefs` and `m_bUseAG2HeroGraph/UIGraph`. Side effect: the ANIM block is rewritten uncompressed (model 5 MB → ~14 MB) — expected.
`compile_and_pack.ps1 -StripAg2` does strip → compile → add in one go (original saved as `*.vmdl.ag2orig`; the ps1 re-adds only for vmdls that had AG2).
Hero folder is `heroes_wip` or `heroes_staging` (`models/<folder>/<hero>/`).

## 6. Pack (`compile_and_pack.ps1`)

```
powershell -NoProfile -ExecutionPolicy Bypass -File scripts/compile_and_pack.ps1 -StripAg2 \
  -Vmdl 'models/heroes_wip/lash/lash.vmdl' \
  -ExtraSource 'materials/models/heroes_wip/lash/nelash_bf/*.vmat,particles/nelash/*.vpcf' \
  -PackInclude 'models/heroes_wip/lash/lash.vmdl_c,materials/models/heroes_wip/lash/nelash_bf/*,particles/nelash/*,panorama/images/heroes/*,resource/localization/*' \
  -BaseGameList build/diff/pak01_list.txt -Out build/nelash_bf/pak77_dir.vpk
```
- `powershell -File` passes list parameters as one comma string; the script splits on `,` and accepts `/` or `\`.
- `-PackInclude` globs are relative to the addon's *game* output dir; `-BaseGameList` skips files already in the base game.
- Only **`_c` files** and localization `.txt` belong in a pak. Never pack `image_compiler.vdata_c`, sources, or `.ag2orig`.
- The VPK writer is VPK v2, one `_dir.vpk`, all data embedded (archive index 0x7FFF), correct CRC/MD5 → `VRF --vpk_verify` passes.
  `scripts/vpk_tool.py add <in> <out> path=file` patches an existing pak (used to add the name file to Billy's pak).
- **Pak numbering**: the lowest `pakNN` wins when two paks ship the same path. Lash = 77, Billy = 78; "Fancy Lash Skin" (pak14)
  hid ours until disabled. Different heroes never collide (own material/particle namespaces `nelash*`, `nebilly*`).

## 7. Verify (do all before saying "works"; in-game is only confirmed by the user)

```
VRF=tools/vrf/Source2Viewer-CLI.exe
$VRF -i build/x/pak77_dir.vpk --vpk_verify                      # CRC
mkdir build/x/ex; $VRF -i build/x/pak77_dir.vpk -o <abs>/build/x/ex
python -I work/export/scripts/check_refs.py build/x/ex          # "unresolved refs: 0"
$VRF -i build/x/ex/models/.../lash.vmdl_c -b DATA > data.txt    # skeleton (compare to original dump), m_animGraph2Refs, particles_list, TaggedSoundSettings_t
$VRF -i build/x/pak77_dir.vpk -f models/heroes_wip/lash/lash.vmdl_c --game "<Deadlock>/game/citadel/gameinfo.gi" -d \
     --gltf_export_format glb --gltf_export_materials --gltf_textures_adapt -o <abs existing dir>
"%BLENDER%" -b --python work/export/scripts/render_verify.py -- <abs>/lash.glb <abs>/build/x/verify_ [markers.json] front,side,back
```
Expect: 171 bones with identical names/parents (bone *order* differs — fine), local pos ≤ 0.0001 in, rot ≤ 0.16°; height ≈ original (2.67 u);
glove on her right (viewer-left in the front view); facing −Y in glTF = Source +X. Render outputs need **absolute paths**.
Attachment check: markers (origin sphere + X-axis arrow + blade-length line) from the compiled attachment quaternions must lie along the
blade, centred, pointing to the tip. glTF cannot show additive FX cards/particles (dark smudge) — only the game can.

## 8. Zip, install, mod manager

- Zip = `pakNN_dir.vpk` at the zip root (`Compress-Archive -Path pak77_dir.vpk -DestinationPath build/dmm/<Name>.zip`), imported with
  Deadlock Mod Manager "Add local mod". DMM copies it to `game/citadel/addons/local-<uuid>_pak77_dir.vpk`.
- DMM's **compatibility overlay** (`game/citadel/dmm_localization_default/pak01_dir.vpk`, first in `gameinfo.gi` SearchPaths; state in
  `%LOCALAPPDATA%\dev.stormix.deadlock-mod-manager\{compatibility,logs}`) rebuilds at launch: restores a missing camera interface /
  tagged sounds on hero models, merges localization tokens and vdata rows across mods. Log lines: `localization_overlay::model_camera`,
  `... Built repair plan for N mods`. Don't hand-edit its files.
- Never write into `game/citadel/addons/` yourself without the user's OK.

## 9. Gotchas that cost time

- `--vpk_list` is `-l`; extraction `-e ext1,ext2 -f prefix`; decompile needs `-d`; glTF export needs `-d` and an existing `-o` dir.
- Bash heredocs mangle `\n`/`\t` in generated Python — write scripts with the editor/Write tool or use `chr(10)`.
- Windows Python can't open `/c/...` paths; use relative or `C:/...`.
- Blender `ops.export_scene.smd(collection=name)` exports a collection; the export operator undoes itself (state restored).
- A single particle/model `_c` can be 14 MB+; keep an eye on the zip size (DMM has no hard limit, game loads fine).
- CSDK download from Google Drive often fails with "quota exceeded": ask the user to download in a browser.
- Material remaps (`DefaultMaterialGroup`) are unnecessary when DMX material names are full paths.
- Weapons/rigid items: weight 100% to one bone (`spine_3`, `pelvis`); the exporter requires ≥1 vertex group per mesh.
