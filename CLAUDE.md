# PalEdit — Palworld 1.0 fork (working notes)

Fork of [EternalWraith/PalEdit](https://github.com/EternalWraith/PalEdit) (Nexus mod 104,
upstream stale at v0.12.1 / pre-1.0) updated for **Palworld 1.0** (released 2026-07-10).
Owner runs it via a Vortex dashboard tool tile pointing at the frozen exe.

## Layout on this machine

- This repo: `C:\Users\Turtle\Documents\Claude\Projects\PalworldEditor\src`
- Frozen build the owner actually launches: `..\PalEdit-1.0\PalEdit.exe`
  (copy of `build\exe.win-amd64-3.14\` — rebuild + recopy after code changes)
- Original Nexus zip + extracted v0.12.1 for reference: `..\` (project root)
- Owner's saves (**NEVER test against these — always copy to scratchpad first**):
  `%LOCALAPPDATA%\Pal\Saved\SaveGames\76561197997626279\`
  - `GlobalPalStorage.sav` — the Global Palbox; **this is what the owner edits.**
    Their world is not hosted on this machine: there is NO `Level.sav` locally,
    only per-world `LocalData.sav` (client-side) + the global box.

## Build / run

```
python PalEdit.py                  # run from source (needs pillow, pyperclip)
python CxFreezeCompile.py build    # freeze -> build/exe.win-amd64-3.14/
python update_data.py              # refresh game data from psp repo dumps
python update_data.py --icons      # + fetch missing pal icons from paldb CDN
```

Python 3.14.4 locally; cx_Freeze 8.6.4. `palworld_save_tools/` is vendored in-tree
(upstream ships it zipped as `palworld_save_tools.zip`; the extracted dir is what
imports resolve to — includes native Oodle libs under `lib/<platform>/ooz.pyd`).

## What 1.0 changed and how this fork handles it

1. **Compression**: 1.0 saves are Oodle-compressed, magic `PlM`, save_type 0x31
   (pre-1.0 world saves were zlib `PlZ` 0x32). The vendored save-tools fork
   already handled PlM. `loadfile` now stashes `self.save_type` from
   `decompress_sav_to_gvas` and `savefile` reuses it, so round-trips preserve
   the original format. Verified: GVAS write of an unmodified GlobalPalStorage
   is **byte-identical** to the input.
2. **`IsPlayer` key**: 1.0 writes `IsPlayer: False` on every pal's
   SaveParameter. PalInfo used key-presence to detect (and reject) player
   characters → every pal failed to load. Now checks the value.
3. **GlobalPalStorage.sav support** (`storage_mode`): top-level property is
   `SaveParameterArray` → 960 fixed slots, each `{SaveParameter, InstanceId}`;
   empty slots have `CharacterID == "None"`. `loaddata` wraps each occupied
   slot into the Level.sav entry shape
   (`{'key': {'InstanceId': ...}, 'value': {'RawData': {'value': {'object': {'SaveParameter': sp}}}}}`)
   so `PalEntity` mutates the same dicts **by reference** and edits flow back
   into the GvasFile on save. In storage mode: `palguidmanager is None`,
   players dict = `{"Global Palbox": PalInfo.PalStoragePlayer()}`,
   `FilteredPals` returns everything. Spawn/clone/delete already bail on
   `palguidmanager is None` (upstream guards) — see "Feature ideas" below.
4. **Data**: level cap 65 → **80**; passives now have **rank 5**
   (`PalEditConfig.skill_col` gained a 9th color — index = rating + 3, would
   IndexError otherwise); 1.0 SaveParameter has no `Talent_Melee` anymore
   (PalEntity re-adds a default; harmless, game ignores it).

## Game data pipeline (`update_data.py`)

Source of truth: JSON dumps in the **oMaN-Rod/palworld-save-pal** repo
(`data/json/*` — actively maintained, was current within a day of 1.0 patches).
Cached in `psp_data_cache/` (gitignored). Field mapping:

| PalEdit                          | psp source                                  |
|----------------------------------|---------------------------------------------|
| `data/pals/<Code>.json` Type     | `element_types` (+ pad "None"; empty+human → `["None"]`) |
| Moveset `EPalWazaID::X: lvl`     | `skill_set` (add `EPalWazaID::` prefix)     |
| Scaling HP/PHY/MAG/DEF           | `scaling.hp/attack/attack/defense` (PHY=MAG; melee stat is legacy) |
| Suitabilities                    | `work_suitability`                          |
| Human                            | `not is_pal`                                |
| `data/attacks/*` Type/Power/Category | `element` / `power` / `type`           |
| `passives.json` Rating (string!) | `rank` (int, −3..5)                         |
| `<lang>/…` display names         | `l10n/<lang>/…` `localized_name`            |

Gotchas learned the hard way:
- `PalInfo.LoadPals` reads the **per-file dirs** (`data/pals/`, `data/attacks/`),
  NOT the aggregate `data/pals.json` — the aggregate is dead weight.
- `LoadPassives` does `l[code]["Name"]` **unguarded** → every passives.json key
  must exist in every lang's passives.json (updater guarantees fallbacks).
- Per-pal files are read with locale-default encoding (`open(..., "r")`) in the
  frozen 3.14 app → keep generated JSON ASCII (`ensure_ascii` default).
- Never emit a `"Tower"` key in generated pal files: the Tower branch in
  LoadPals does a base-species lookup that breaks when the base sorts after
  GYM_* alphabetically (e.g. WorldTreeDragon). GYM files carry full data instead.
- Icons: `T_<Code>_icon_normal.png` in `resources/pals/` after stripping
  `RAID_`/`_2` (see `GetImage`). Missing → `#ERROR.png` fallback (safe).
  paldb CDN mirrors game texture paths:
  `https://cdn.paldb.cc/image/Pal/Texture/PalIcon/Normal/T_<Code>_icon_normal.webp`.
  ~27 scrapped/quest entities (BeardedDragon, YakushimaBoss*, Quest_Farmer03_*…)
  have no icon anywhere; owner is fine with the placeholder.

## Testing recipe (all against scratchpad copies!)

Headless e2e that exercises the real code paths (no dialogs):
instantiate `PalEdit()`, `gui.withdraw()`, decompress a **copy** of
GlobalPalStorage.sav, `GvasFile.read(..., PALEDIT_PALWORLD_CUSTOM_PROPERTIES)`,
`app.loaddata(gvas)`, edit via real setters (`SetLevel/SetTalentHP/...`),
`app.savefile()` (uses `app.filename`), re-parse and assert. Last run: set a
BerryGoat to lvl 80, IVs 100/100/100 → verified on disk. Note
`PalInfo.logger` is only set inside PalEdit's main; standalone PalInfo use
needs a stub logger.

## Feature ideas discussed with owner (2026-07-20)

1. **Legality filtering for passives/attacks** (owner wants dropdowns limited
   to what a pal can normally have):
   - Passives: psp `passive_skills.json` has `add_pal` (85/420 can roll on
     wild pals), `add_rare_pal` (lucky), and psp `pals.json` has per-species
     innate `passive_skills` lists (e.g. JetDragon = `["Legend",
     "ElementBoost_Dragon_2_PAL"]`). Legal set ≈ add_pal pool ∪ species
     innates. Data NOT currently carried into PalEdit resources — updater
     would need to emit it (e.g. `Rollable` flag in passives.json + innate
     list in per-pal files).
   - Attacks: per-species learnsets already exist (`PalLearnSet`); skill
     fruits can teach any **non-unique** move, and unique moves are
     identifiable by `Unique_` in the EPalWazaID. Legal set ≈ learnset ∪
     non-unique moves (optionally element-filtered "strict" mode).
   - Plan: filter the 4 passive dropdowns + attack add-list per selected pal,
     with a "show all" escape hatch in Config menu.
2. **Species swap in Global Palbox**: UI already exists upstream
   (`speciesvar` dropdown → `pal.SetType`, PalEdit.py ~line 1102) and is not
   gated on `palguidmanager`, so it likely already works in storage mode —
   needs a scratchpad-copy test. Polish: after swap, moveset can be illegal
   for the new species; offer to replace EquipWaza/MasteredWaza from the new
   species' learnset at the pal's level (learnset data already loaded).
3. **UI redo "a bit"**: incremental Tk reorganization is the sane scope
   (group stat/skill panels, filter toggles); a full web/modern UI rewrite is
   a different project — don't start it casually.

## Publishing

Repo intended for the owner's GitHub (github.com/TheMysticTestudine). Upstream
remote kept as `upstream` (EternalWraith/PalEdit). Owner's Nexus/Vortex flow
unchanged — they launch the local exe, not a download.
