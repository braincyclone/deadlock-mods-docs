# 04 — Adapting the pipeline to a new hero (checklist)

Learned on Billy (`punkgoat`) after Lash. Worked example scripts: `work/billy/scripts/` (`run_all.sh` = run order).
**Automated path:** `app/pipeline/forge_pipeline` (`python -m forge_pipeline build ...`, see `app/pipeline/README.md`)
does the per-hero steps below data-driven (survey → hero profile → automap → fit → export → compile → pack → zip).
Use this checklist when doing a hero by hand, or to debug/extend a forge hero profile (`app/pipeline/heroes/<codename>.json`).

## 1. Identify the hero
- [ ] Decompile `scripts/heroes.vdata_c` from `game/citadel/pak01_dir.vpk`. Each `hero_<codename>` block has `m_strModelName`:
  `tools/vrf/Source2Viewer-CLI.exe -i <pak01_dir.vpk> -f scripts/heroes.vdata_c -d -o <dir>`
  Codenames differ from display names (Billy = `punkgoat`, model `models/heroes_wip/punkgoat/punkgoat.vmdl`). Folder is `heroes_wip` or `heroes_staging`; DeadlockTools needs it.
- [ ] List the hero's files: `-l -f models/<folder>/<codename>/`. Clips (`*.vnmclip_c`) sit in `clips/` and sometimes in `dmx/animation/`.
- [ ] Decompile the vmdl (`-d`, writes vmdl + mesh/anim DMX + hull). Decompile its materials too (`-e vmat_c -d`).
  Some meshes use `materials/rebuild/*.vmat`; check which paths the mesh DMX actually reference.
- [ ] Export a glb with materials and a handful of clips: `--gltf_export_format glb --gltf_export_materials --gltf_textures_adapt --gltf_export_animations --gltf_animation_list "ui_hero_pose,<folder>/clips/melee_quick_1,..."`.
  Clip names need the full `models/.../clips/<name>` path. Filter misses are only warned about.
- [ ] Dump compiled DATA (`-b DATA` on the extracted vmdl_c) → `work/billy/scripts/make_skeleton_json.py` gives the ORIGINAL bind pose for export.
  Also take `TaggedSoundSettings_t` from it: the decompiler drops it and `build_vmdl.py` re-adds it.

## 2. Inspect hero skeleton, rest pose, meshes
- [ ] `blender -b --python work/billy/scripts/inspect_billy.py -- <glb> <hero.blend>` prints the bone tree with world heads, each mesh's materials, and bone weight totals per material.
- [ ] Note: forward axis (heroes face −Y after glTF import), rest pose (A-pose vs arms-down), proportions.
  Billy is about 3 m tall with long legs. The fit stretches segments, so this matters more than total height.
- [ ] Constraint bones (`*_TWIST`, `*_NOTWIST`, `elbow/knee_helper`, `neck_0_TWIST`) come from the vmdl `AnimConstraintList`. Clips leave them constant, so Blender previews show no twist; the game solves them at runtime.
  Weight to them the way the hero's own mesh does: shoulder end → `arm_upper_X_TWIST`, mid → `_TWIST1`, elbow end → `arm_upper_X`. The forearm goes the other way round.
- [ ] Render the original with `work/billy/scripts/render_check.py` (rest / anim / sword / gun sheets) to see weapon states before changing anything.

## 3. WoW side: skeleton type
- [ ] Import the wow.export glTF (`work/billy/scripts/inspect_orc.py`) and dump the bone tree and weight totals. Character faces +X.
- [ ] **HD** models (`*_hd.m2`): EXP_ twist bones, full fingers, numbered face bones; attachments in the `.skel` SKA1 chunk.
  **Classic** models (e.g. anniversary `orcfemale.m2`): about 1k verts, a one-bone finger "mitten" (`RingFinger`/`Tip`) plus `Thumb`, no twist bones, attachments in the `.m2` itself (`work/rig/scripts/m2_attachments.py`).
  Raw files: FDID from `tools/listfile_dl/lf.csv` → `https://wago.tools/api/casc/<FDID>?download`.
- [ ] Unskinned item meshes (helm, shoulders) need M2 attachment position + bone (ShoulderR = 5, ShoulderL = 6, Helm = 11). Map the attachment bone to its WoW node by position.

## 4. Rebuild the bone mapping (`work/billy/scripts/build_rig.py` + `tune.json`)
- [ ] Global scale = hero shoulder height / WoW shoulder height. Rotate −90° about Z.
- [ ] Fit segments are WoW joint → hero joint pairs. Stretch is along the bone only; optional radial `bulk`; pelvis is rigid and widened.
  Big rest-pose differences (WoW arms down vs hero A-pose) are fine. The fit is LBS with WoW's own weights.
- [ ] Where the WoW rig has fewer bones, distribute weights along the WoW segment (`RULES`): spine/chest onto spine_0..3, upper arm/forearm/thigh onto the twist chain.
- [ ] Fingers: the HD per-finger map works directly. A classic mitten is split geometrically after the fit: distance to each hero finger chain gives Gaussian finger weights, then phalanx blending (`mitten_parts`).
- [ ] Face → head; Jaw → `jaw_0`; tusks, ears and hair roots follow the head; long hair/ponytail → spine. Tabard/loincloth flaps → pelvis, lower part blended to thighs.
- [ ] Rigid items: weight 100% to one bone (pads → `arm_upper_X_TWIST`, no roll). Rest rotation is a slerp between the clavicle and upper-arm fits (`pad_arm_share`).
- [ ] Iterate on renders (rest with hero ghosted, anim sheets, close-ups). Look at every sheet.

## 5. Weapon replacement
- [ ] Find the weapon bone(s): look at weights per material (`inspect_billy.py`). Billy's bat is part of the body mesh, 100% on `bat_0` (left hand, under `attachHand_L`).
  Its shoulder rest and swings are all that bone animating, so there is one state to cover. Check every clip that moves the weapon (`*_transition_*`, ult) for extra states such as stowed or separate bones.
- [ ] Shared materials: bat, gun and bottle shared `punkgoat_weapons`. Remove weapon faces by bone, keep props the abilities need (bottle → own mesh with the base-game material path), and give the recolored part its own material.
- [ ] Place the new weapon on the old axis: grip at the bone head, blade along `bone → bone_end`, edge along the knuckles. Scale by length relative to the old weapon; compress a 2H hilt for a 1H grip (`tune.json` → `sword`).
- [ ] FX anchor = empty parented to the weapon bone. `export_dmx.py` writes `anchors.json` and `build_vmdl.py --anchors` adds the attachment. Set the particle half-length (`gen_particles.py` `TF_HALF_LEN`) from the logged blade length.
- [ ] The gun usually stays a separate mesh on the `weapon` bone (bind pose at the origin). Re-export it from the glb with its own material, and copy the other maps from the decompile (`assemble_content.py`).

## 6. Outline / rage shell meshes
- [ ] Bodygroups whose meshes are offset copies of the hero (Billy: `rage` → `head_jitter01/02`, `bat` → `bat_jitter`) must be replaced with shells of the NEW mesh, or the old silhouette floats around the new model.
  `build_rig.py shell()` pushes the mesh out along its normals and keeps the hero's base-game jitter material path. Shells go in `LODGroupAll`.

## 7. Namespacing, packing, conflicts
- [ ] Own paths per mod: `materials/models/<folder>/<codename>/<mod>/`, `<mod>_wpn/`, `particles/<mod>/`. Only the vmdl_c (and portraits) overwrite base paths. Base-game materials may be referenced without shipping them; the compiler warns "missing material", which is expected.
- [ ] Pak number: unique per mod. Lash 77, Billy 78. List `game/citadel/addons/` and avoid every number in use (active `pakNN_dir.vpk` and mod-manager stash names).
- [ ] Conflict check: `Source2Viewer-CLI -i <addon vpk> -l | grep <codename>` for every addon VPK. Only `models/.../<codename>` or `materials/models/.../<codename>` hits matter; voice and UI files mentioning the codename are harmless.
- [ ] Compile: `work/billy/scripts/run_compile.sh` (strip AG2 → resourcecompiler → `DeadlockTools add ag2 --hero <codename> --hero-folder <folder>` → VPK).
  Re-pack only: `run_pack.sh`. Zip with `pakNN_dir.vpk` at the root → `build/dmm/<Name>.zip`.

## 8. Portraits (`docs/06a-portraits.md`; Billy copy in `work/billy/portraits/scripts/`)
- [ ] Slots: `panorama/images/heroes/<codename>_{card,card_critical,card_gloat,vertical,sm,mm}_psd.vtex_c` plus a hashed faded `_sm_psd_<hash>` (BC7).
  Decode the originals for size, alpha and framing (Billy's mm is 50×51, and he faces screen-right).
- [ ] Render the WoW-native model with WoW clips. A classic-model glTF has no clips: export the same race/sex with `app/wowbridge` (`python -m wowbridge export-character --spec <spec.json> --out <dir> --animations`), import it, and append its actions to the armored blend (`work/billy/portraits/orc_anim.blend`).
- [ ] Turn pngs into vtex with `build_vtex.py <original.vtex_c> <png> <out>` (clone the original, swap pixels). For BC7 slots clone a BGRA template of the same size with `mips 8 flags 0`.

## 9. Verify (always, before handing over)
- [ ] `max head deviation` from `export_dmx.py` ≈ 0. Compile log has no errors (single-LOD and base-material warnings are expected).
- [ ] `check_refs.py` on the extracted VPK: 0 unresolved.
- [ ] VRF re-export of the compiled model from the VPK (`--game <Deadlock gameinfo.gi>`, with clips) → `inspect_billy.py` → `render_check.py` rest/anim/sword/gun. Look at every render.
  Additive FX cards render black in VRF exports; that is expected.
- [ ] Portraits: decode the packed vtex_c back to png and diff against the source pngs (must be 0).
- [ ] Tell the user that in-game behaviour (particles, twist, rage shells) is untested until they confirm.
