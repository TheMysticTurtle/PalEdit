# Changes in this fork

Fork of EternalWraith/PalEdit, updated for the Palworld 1.0 release and
extended with editor features and save-safety fixes. Each item below is a
self-contained branch merged into `main`, so changes can be cherry-picked
individually.

## Palworld 1.0 support

- Read and write the 1.0 save format (Oodle-compressed `PlM`, save type
  0x31), preserving the original compression on save.
- Load `GlobalPalStorage.sav` (the 1.0 Global Palbox), not just `Level.sav`.
- Fix every pal failing to load in 1.0 saves (1.0 writes `IsPlayer: false` on
  each pal; detection now checks the value instead of key presence).
- Refresh all game data for 1.0: new species, movesets, stat scaling, work
  suitabilities, attacks, and passive skills, plus new-pal icons.
- Raise the level cap to 80 and add the rank-5 passive tier.

## Editor features

- Add, clone, and delete pals in the Global Palbox. New pals start as a
  default species and can be re-typed from there.
- Edit pal nicknames (updates both `NickName` and the 1.0 `FilteredNickName`).
- Searchable attack picker with a tier toggle (learnset / fruit-teachable /
  all), element filter, and damage sort.
- Searchable passive picker grouped by effect (Attack, Defense, Work, etc.),
  with a per-passive effect description and a natural-only / all toggle.
- Automatic save backup: the loaded save is copied to a `PalEdit-backups`
  folder before the first write of each editing session.
- Hide the stale-player warning when editing the Global Palbox, where it does
  not apply.
- Modern dark theme (centralised palette applied across the UI).

## Fixes

Work-suitability, move, and species-change handling were audited against the
public 1.0 editor *palworld-save-pal* to confirm the correct 1.0 field model.

- Fix work-suitability corruption. Pals lost their farming/grazing/kindling
  behaviour because opening and saving injected a zero-rank entry into
  `GotWorkSuitabilityAddRankList` for every suitability (13 phantom entries
  per pal). Zero-rank entries are now pruned on load and never written; a
  suitability entry is created only for a real, non-zero bonus. Loading a
  previously affected save and re-saving removes the phantom entries.
- Fix work suitabilities "flip-flopping" between pals: on selecting a pal, the
  suitability controls set their value before their minimum, so a stale
  minimum from the previous pal could clamp and later write a wrong value. The
  range is now set before the value, and the write-back is skipped during
  selection refresh.
- Remove the pre-1.0 `CraftSpeeds` field. `SetType` wrote this field on every
  species change; 1.0 saves have no such field and the game derives work
  suitability from species data plus `GotWorkSuitabilityAddRankList`. It is no
  longer written and is stripped on load.
- Stop `MasteredWaza` pollution. The move list was auto-filled with the whole
  species learnset on load, so opening and saving added phantom "mastered"
  moves to every pal. `MasteredWaza` now holds only moves taught beyond the
  natural learnset (matching the game); the displayed move pool is derived and
  never written. Previously polluted lists are cleaned on load.
