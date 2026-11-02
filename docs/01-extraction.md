# 01 – Extraction

## Deadlock hero

1. Find the hero's codename and model path: decompile `scripts/heroes.vdata_c` from
   `<Deadlock>/game/citadel/pak01_dir.vpk` and read `m_strModelName` for `hero_<codename>`
   (codename list: https://wiki.divineskins.gg/en/docs/deadlock/basics/hero-codenames — e.g. Lash = `lash`,
   Billy = `punkgoat`). Models live in `models/heroes_wip/` or `models/heroes_staging/`; a `_v2` folder may
   contain only clips/materials, not the shipping `.vmdl_c`.
2. Export with VRF (`tools/vrf/Source2Viewer-CLI.exe --help` for flags):
   - glTF/glb of the model + textures + a handful of animations (`--gltf_animation_list` names **without** the hero
     prefix). Create the output folder first or VRF writes `<folder>.glb` next to it.
   - Decompiled `.vmdl` text (ModelDoc) + `.dmx` meshes + `.vmat` for reference (`-d`).
   - Raw compiled files (keep VPK-relative paths) for later diffs.
3. Inspect in Blender headless (`work/lash/scripts/inspect.py`, `joints.py`): bone tree, joint positions, rest
   pose, materials, mesh/vertex counts. Hero glTF imports in metres, facing **−Y**, character's right = −X.
   A stray `Icosphere` object appears in VRF/wow.export glTF imports — delete it.

## WoW character (wow.export)

wow.export has no headless mode, so **the user exports** from its GUI:
- Characters tab → load a saved character (`.json` save; it records race, customization choices and equipment
  slot item IDs) → export **glTF**. Animations can be off for the armored export; the base body export with
  animations (~400 actions) is useful for portrait expressions.
- Settings → Export Directory must be set and *saved* before exporting (it silently reverted once and
  overwrote a previous export).
- Prefer the **HD** character models (e.g. `nightelffemale_hd.m2`). The orc came out as the classic ~1k-vertex
  model with mitten hands — ask the user to re-export the HD model when quality matters.

What you get: skinned geosets (body, hair, ears, equipment geosets like Gloves3/Boots3/Tabard2), armor textures
baked into the body texture (`data-1.png`), and **unskinned** item meshes (helm, shoulders) that must be placed
from WoW attachment data (see 02-rigging.md).

Raw files (`.m2`, `.skin`, `.skel`, `.blp`) are sometimes needed for attachments/emitters: fetch by FileDataID
from `https://wago.tools/api/casc/<FDID>?download`, convert BLP→PNG with Pillow.

## WoW items (weapons)

Item names don't appear in model filenames, so search by item → model chain via wago.tools DB2 tables:
`ItemModifiedAppearance → ItemAppearance → ItemDisplayInfo (ModelResourcesID, ModelMaterialResourcesID) →
ModelFileData → FileDataID → listfile`. Also read `Item.SheatheType` (1 = back, 3 = hip).
Scripts: `work/weapons/scripts/fetch_weapons.py`.

| Item | ID | displayID | Model (FDID) | Texture |
|---|---|---|---|---|
| Thunderfury, Blessed Blade of the Windseeker | 19019 | 30606 | `sword_2h_ashbringer02.m2` (148234) | `sword_2h_ashbringer_a_01blue` |
| Perdition's Blade | 18816 | 33615 | `knife_1h_epic_a_03.m2` (146143) | `knife_1h_epic_a_03` |

Then have the user export the model by FileDataID in wow.export's Models tab (pick the right texture variant —
Thunderfury's is the `_a_01blue` skin) as glTF. Item VFX may be geometry inside the M2 (Thunderfury's lightning
cards) or M2 particle emitters (Perdition's fire) — emitters don't survive glTF; read them from the raw M2.
