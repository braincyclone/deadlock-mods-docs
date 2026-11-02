# Pipeline overview: WoW character → Deadlock hero skin

A hero skin mod replaces the hero's compiled model (`models/<folder>/<codename>/<codename>.vmdl_c`) at the same
VPK path, keeping the hero's own skeleton, animations, hitboxes and attachments. The job is to put a WoW mesh onto
that skeleton, compile it with Valve's tools, and pack it as a `pakNN_dir.vpk`.

```
 WoW install ──wow.export──► glTF (+ raw M2/skel)      Deadlock pak01 ──VRF──► hero glb + decompiled vmdl
                    │                                                   │
                    └──────────► Blender: fit WoW mesh onto hero skeleton ◄┘   (02-rigging.md)
                                   │  verify with renders of hero animations
                                   ▼
                          DMX export → vmdl → resourcecompiler (CSDK 12) → DeadlockTools add ag2
                                   │                                         (03-export-compile-pack.md)
                                   ▼
             + materials / VFX (05-vfx.md) + portraits & names (06-*.md)
                                   ▼
                    pakNN_dir.vpk → zip → Deadlock Mod Manager "Add local mod"
```

Read the stage docs in order the first time; use `04-new-hero-adaptation.md` as the checklist when starting a new hero.

The docs use two real adaptations (Lash with a Night Elf, Billy with an Orc Warrior) as worked examples. They mention
helper scripts (`work/...`, `scripts/...`) that are not included here; treat those as descriptions of what to write.

## Stage docs

| Doc | Covers |
|---|---|
| [01-extraction.md](01-extraction.md) | Getting WoW models/items and the Deadlock hero out of the games |
| [02-rigging.md](02-rigging.md) | Fitting the WoW mesh onto the hero skeleton in Blender, rigid items, weapons placement |
| [03-export-compile-pack.md](03-export-compile-pack.md) | DMX export, vmdl build, compile, AnimGraph2, VPK, verification, install |
| [04-new-hero-adaptation.md](04-new-hero-adaptation.md) | Checklist for adapting a new hero |
| [05-vfx.md](05-vfx.md) | Animated self-illum materials and model-attached particle systems |
| [06-ui-images-and-names.md](06-ui-images-and-names.md) | HUD portraits/banners (vtex clone-and-swap) and hero name localization |
| [06a-portraits.md](06a-portraits.md) | Rendering portrait art with WoW facial expressions |

## Toolchain

| Tool | Use |
|---|---|
| Source2Viewer CLI (VRF) 20.1 | List/extract/decompile VPKs, export glTF, verify VPKs |
| Reduced_CSDK_12 | `game/bin/win64/resourcecompiler.exe`; addon content in `content/citadel_addons/<addon>/` |
| DeadlockTools v1.1.0 | `add ag2` injects AnimGraph2 + NmSkeleton refs CSDK 12 can't compile |
| wow.export (portable) | GUI; export characters/items to glTF (no headless mode) |
| Community listfile | WoW FileDataID ↔ filename |
| Blender 5.1 | Headless `-b --python script.py -- args`; BlenderSource2Tools (Fuji fork) addon for DMX |

CSDK 12 is a 2.6 GB Google Drive zip linked from the Deadlock Modding Notes page
(deadlockmodding.pages.dev → modding-tools/csdk-12). Drive often refuses automated downloads ("quota
exceeded"), so download it in a browser. Primary reference wiki: https://wiki.divineskins.gg/en/docs/deadlock/.

## Conventions that save time

- Verify every stage with renders you actually look at: rest pose with the original hero ghosted over it, frames
  from several hero animations, close-ups of trouble spots (shoulders, hands, items), and finally a VRF re-export
  of the *compiled* model from the VPK.
- Check `game/citadel/addons/` for other enabled mods that ship the same model path: the lowest pak number wins,
  so an installed skin will silently hide yours.
- Give each mod its own material/particle namespace so mods for different heroes never collide.
- Each mod can ship a full copy of the hero-names localization file with only its own hero renamed; Deadlock Mod
  Manager merges these per token (see `06-ui-images-and-names.md`). Without it, the lower pak number's file wins.
- Keep WoW items at WoW's real scale relative to the body rather than shrinking them for looks.
- Generalize scripts with arguments/configs rather than forking them per variant.

## Legal

WoW and Deadlock assets are copyrighted. This repo contains documentation only; extract assets from your own game
installs, keep them out of version control, and don't redistribute builds. Personal use only.
