# 05 — VFX: animated materials and model-attached particles

Used for the sheathed Thunderfury (crackling lightning) and Perdition's Blade (embers/flame) on Lash. Confirmed in game:
the particles fire. Sources: `work/vfx/` (`gen_textures.py`, `gen_vmats.py`, `gen_particles.py`, `build_fx.sh`).
Research dumps (stock model/particle scans, round-trip tests) were in `build/vfx_research/` (gitignored, regenerable).

> Wiki rules (Divine Skins): effects must not be brighter/bigger/fainter than originals for *ability* effects; glowing
> or see-through hero skins are listed as prohibited. Keep weapon FX restrained; the user accepts this for a private mod.

## 1. Two layers

1. **Materials** (no particles): `pbr.vfx` self-illumination, UV scroll, `DynamicParams` expressions → pulsing/scrolling emissive geometry cards.
2. **Particles**: `.vpcf` systems attached to a model attachment through the model-embedded `particles_list` mechanism.

## 2. Materials (`gen_vmats.py`)

Only shader in the game is `pbr.vfx`. Useful flags/params (found in stock vmats, e.g. `models/heroes_wip/geist/materials/geist_armglow.vmat`):

| Want | Params |
|---|---|
| Glow | `F_SELF_ILLUM 1`, `g_flSelfIllumScale1`, `g_vSelfIllumTint1`, `TextureSelfIllumMask1`, `g_flSelfIllumAlbedoFactor1` |
| Additive card | `F_ADDITIVE_BLEND 1`, `F_TRANSLUCENT 1`, `F_WRITE_DEPTH_BEFORE_ALPHA_BLENDING 1`, `F_DISABLE_NPR_OUTLINE 1`, `F_DO_NOT_CAST_SHADOWS 1`, `F_RENDER_BACKFACES 1`; shape = `TextureTranslucency1` (mask), colour = tint, `TextureColor1 = [0 0 0 0]` |
| Scroll | `F_ENABLE_TEXTURE_TRANSFORMS 1`, `g_vAlbedoScrollSpeed1` / `g_vSelfIllumScrollSpeed1` `[u v 0 0]`, `g_flAlbedoScrollQuantize1` (stepped/jerky scroll), `g_vAlbedoTexcoordScale1` (tiling), `g_flAlbedoTexcoordRotation1` |
| Vertex wobble | `F_JITTER_VERTICES 2`, `g_vJitterAmplitudes{X,Y,Z}{A,B}1`, `g_vJitterFrequencies{A,B}1`, `TextureJitterMask` (needs a local PNG) |
| Opaque item with emissive | Lash NPR flags (`F_USE_NPR_LIGHTING`, `F_SOLID_COLOR_OUTLINE`, `F_USE_STATUS_EFFECTS_PROXY`) + `F_SELF_ILLUM` + illum mask |
| Alpha cut-out | `F_ALPHA_TEST 1`, `g_flAlphaTestReference1`, `TextureTranslucency1` |
| Animation | `"DynamicParams" { "g_flSelfIllumScale1" "return 3+2*sin(time()*5);" }` — also `$ent_age`, `step()`, `abs()`, `clamp()`, `float2()`; compile validates function names |
| Flipbook | **not available in pbr.vfx** (only in particles) |

Pseudo-random flicker = product of sines of different frequencies + a fast third sine; `step(-0.15, sin(a*t)*sin(b*t))` gates opacity on/off.
Knobs live in the MATERIAL TABLE at the bottom of `gen_vmats.py` (`tint`, `scale`/`opac` expressions, `scroll`, `tiles`, `quant`, `jitter`).
Textures: WoW BLPs → PNG (`work/weapons/`), post-processed by `gen_textures.py` (levels → grayscale masks, mirror-tiling for seamless U,
generated tileable flame noise). Materials in the model use full-path names `materials/models/heroes_wip/lash/nelash_wpn/<name>.vmat`.
Keys are case-exact; a misspelled key may be ignored without an error — verify with `VRF -b DATA` on the `vmat_c` (flags appear as `m_name = "F_..."`).

## 3. Particles (`gen_particles.py`)

### 3.1 Authoring rules (vpcf63, CP0 only)
- Plain-number form is valid input (`m_flEmitRate = 9`, `m_vecMin = [ -24, -0.8, -0.8 ]`) — the decompiler's verbose struct form is not required.
- **Header must be `format:vpcf63:version{a6e6a69e-52d3-4527-8b9c-ff3bb91aca3e}`.** Decompiled stock files are vpcf61…66;
  the CSDK compiler fails on 64/65 ("No valid format conversion from 'vpcf65' to 'vpcf63'"). Rewrite line 1 of any decompiled file and it
  compiles (round-trip DATA verified identical on 3 stock systems).
- Unknown/mis-spelled members are **silently dropped**. After compiling, `VRF -i x.vpcf_c -b DATA` and check your fields survived
  (e.g. `C_INIT_CreateWithinBox` takes `m_vecMin/m_vecMax`, not `m_vMin/m_vMax`; defaults are omitted from the dump).
- Classes in use: emitter `C_OP_ContinuousEmitter`; init `C_INIT_CreateWithinBox` (`m_bLocalSpace=true`), `RandomLifeTime`, `RandomRadius`,
  `RandomRotation`, `RandomSequence`, `RandomColor`, `RandomAlpha`, `VelocityRandom`; ops `Decay`, `FadeInSimple`, `FadeOutSimple`,
  `BasicMovement` (`m_Gravity`, `m_fDrag`), `InterpolateRadius`, `ColorInterpolate`, **`PositionLock` with `m_bLockRot = true`**
  (particles follow the attachment); renderers `C_OP_RenderSprites` / `C_OP_RenderTrails`. Full class list: strings in `game/bin/win64/particles.dll`.
- Parent system with only `m_Children = [ { m_ChildRef = resource:"particles/…vpcf" } ]` groups children.
- Reuse base-game textures by path (nothing to pack): `materials/particle/electricity/electric_arc_stylized.vtex` (2 sequences),
  `particle/sparks/sparks.vtex`, `particle/glows/glow04.vtex`, `particle/anime/fire/fire_stylized_d_loop.vtex` (flame sheet;
  `m_nAnimationType = ANIMATION_TYPE_FIT_LIFETIME` + 1D colour-lookup gradient texture entry).
- Convention: **CP0 = attachment; X axis along the blade toward the tip, origin mid-blade**; spawn box ±`HALF_LEN` along X.

### 3.2 Attaching to the model (model-embedded `particles_list`)
In the vmdl `GameDataList` (`build_vmdl.py --particle=fx_tf:particles/nelash/nelash_tf_lightning.vpcf`):
```
{ _class = "GenericGameData"  name = "fx_tf_fx"  game_class = "particle"
  game_keys = { name = resource:"particles/nelash/nelash_tf_lightning.vpcf"
                attachment_point = "fx_tf"  attachment_type = "point_follow"  attachment_offset = null } }
```
plus an `Attachment` node `fx_tf` on a bone (`--attach=`). Compiles to `particles_list` in the model keyvalues and pre-loads the vpcf.
The mechanism is the stock one for NPC/props (`shrine_sapphire`, `core_amber`); it works on a hero pawn (confirmed). It only drives CP0.
Not usable here: `particle_cfg_list` (tools preview only), `CCitadelHeroModelGameData_t.m_hAmbientParticle` (all stock values empty;
hero ambient lives in `scripts/heroes.vdata`, which mods must not ship), `CitadelTeamRelativeParticleSettings_t` (NPCs; untested on heroes),
animation events `AE_CL_CREATE_PARTICLE_EFFECT` (per-animation, would restart each loop; keys: `name`, `attachment_point`, `attachment_type`,
`stop_on_seq_change`, `always_run`).

### 3.3 Knobs (top of `gen_particles.py`, inches/seconds)
`TF_HALF_LEN`, `PD_HALF_LEN` (**set to half the measured blade length**), `*_RATE` (emits/s — scale with length to keep density/inch),
`*_SIZE`, `*_LIFE`, `*_BRIGHT` (renderer overbright), `TF_SPARK_GRAV`, `PD_EMBER_RISE`, `TF_HALO_ALPHA`, `PD_GUARD_*` (second attachment at the
guard end for Perdition's, mirroring the WoW emitters). Raise `m_nMaxParticles` (`system(N, …)`) when raising rates.
Visibility history: first pass was too faint; the true-scale pass uses ×1.3–1.5 size/brightness.

### 3.4 Build
`bash work/vfx/build_fx.sh` regenerates textures/vmats/particles and compiles them in the throw-away addon `nelash_fx`. For a real build run
`gen_vmats.py <repo> nelash nelash_wpn` and `gen_particles.py <repo> nelash` (addon `nelash`), delete unused vmat sources, then compile/pack
(`03-export-compile-pack.md`; pack `materials/models/heroes_wip/lash/nelash_wpn/*` and `particles/nelash/*`).

## 4. Checklist
1. `vmat_c` DATA shows `m_dynamicParams`, scroll values, flags. 2. `vpcf_c` DATA shows every field you wrote.
3. `check_refs.py` unresolved = 0. 4. Model DATA has `particles_list` with each attachment name. 5. Marker render along the blade.
6. glTF preview can't show additive/particles: ask the user to test in game and report (visibility, size, axis direction).
