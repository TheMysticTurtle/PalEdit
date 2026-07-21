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

## Ability search + legality filtering (IMPLEMENTED 2026-07-20)

- `update_data.py` now emits: `Rollable` bool in passives.json (from psp
  `add_pal`/`add_rare_pal`; 85/420 roll on wild pals), `InnatePassives` list
  in per-pal files (e.g. JetDragon = Legend + ElementBoost_Dragon_2_PAL), and
  `Exclusive` species lists on every `Unique_` attack (239 covered) — derived
  from skill_set membership, with fallback parsing of `Unique_<PalCode>_...`
  (exact species match, then startswith for families like Yakushima bosses).
- `PalInfo`: `PassiveRollable` dict, `GetLegalPassives(species)`,
  `PalObject._innate_passives`. Legal passives = rollable ∪ innate.
- `PalEdit`: the 4 passive OptionMenus + 3 equipped-attack OptionMenus now
  intercept `<Button-1>` → `open_ability_search(kind, num)`, a searchable
  Toplevel picker (Entry filters as you type, arrows/Enter/double-click,
  Esc closes). Rows show rating (`Swift  [+4]`) / power (`Fire Tackle  (115)`)
  which also disambiguates duplicate localized names — the picker passes the
  EXACT code (`changeskill(num, code=...)` bypasses the old ambiguous
  name→code index lookup; two passives are both named "Swift"!).
  `availableAttacks/availablePassives(pal)` honour the **Tools > "Legal
  abilities only"** checkbutton (`self.filterlegal`, default on); equipped
  abilities always stay listed. Fruit combobox filters as you type and uses
  the same legality source.
- Tested headless on palbox copies (popup open→search→choose→save→re-parse):
  Caprity can't get Legend when filtered, can when unfiltered; zero foreign
  uniques leak into any box pal's attack list; Rare+Legend and an equipped
  unique survived the disk round-trip. NOTE: many pals legitimately have no
  unique move (e.g. Kitsunebi/Foxparks — all-generic learnset). Synthetic
  `event_generate("<Return>")` is flaky without real focus — test via the
  direct code paths (`changeskill(n, code)`, `attacks[n].set + changeattack`).

## Progress log (2026-07-26 session, Opus continuing Fable's work)

Done and merged to main (each its own feature branch + --no-ff bubble, kept
for cherry-picking):
- `feature/session-backup` — DONE. backup_save() copies the .sav to a
  PalEdit-backups/ folder once per file per session before the first write;
  failed backup aborts the save.
- `feature/nickname-edit` — DONE. Double-click the name label; SetNickname
  writes NickName + FilteredNickName.
- `feature/palbox-add-remove` — DONE. Clone/Add New Pal/Delete work in the
  Global Palbox (storage_mode) against the flat 960-slot array. Add defaults
  to CubeTurtle (Tetroise). Helpers: _palbox_values/_palbox_container_id/
  _next_free_slot_index/_find_empty_palbox_entry/_palbox_insert etc.
- `feature/stale-warning-storage` — DONE. Hides the stale-player warning in
  palbox mode (see research below).
- Also: `.gitattributes` line-ending normalization; DeckIndex/TowerBoss
  metadata emitted by update_data.py; repo renamed src→PalEdit.

### Research findings (task: caps + stale warning)
- **Stale-player warning**: CONFIRMED owner's hunch. Warning text (PalEdit.py
  ~2722) is purely a world-save (Level.sav + Players/*.sav) concern — pals
  break when their owning player's Players/<guid>.sav is stale. Global Palbox
  has NO player data, so it's irrelevant there. Fixed: hidden in storage_mode.
- **Editable caps in the current UI**: Pal Souls (Rank_HP/Attack/Defence/
  CraftSpeed) sliders already 0–20 (matches in-game soul max). Condensation
  Rank UI 1–5 (0–4 stars) = game max. IVs (Talent_*) 0–100. **Work
  suitability spinboxes are capped at 5** (PalEdit.py ~2471 `to=5`) — this is
  the "caps at 5" the owner hit. Raising it (owner wants up to 10 for
  make-anything pals) is an UNRESTRICTED-mode toggle to build in
  `feature/stats-panel` (#18): default = standard caps, toggle = raise the
  work-suitability (and optionally soul/condensation) maxima. Note the game
  itself caps work suitability at 5 by normal means; >5 is a power-user knob.

## STOP-THE-DAY STATUS (2026-07-26 end) — READ FIRST TOMORROW

**PR is BLOCKED** until the work-suitability editor is finished (owner's
call). Do not open the pull request yet.

### Shipped today and deployed to PalEdit-1.0 (owner can launch now)
session-backup, nickname-edit, palbox-add-remove, stale-warning-storage,
attack-search, passive-search, and the critical **fix/work-suitability-
corruption** (merged to main, exe rebuilt + deployed).

### The save-corruption bug (FIXED) — what it was
Owner's farm/ranch pals (Penking, Incineram grazing) produced nothing and
showed no work levels. Root cause was TWO defects:
1. `PalEntity.__init__` wrote a zero-rank entry into
   `GotWorkSuitabilityAddRankList` for every suitability on load → opening +
   saving injected 13 phantom entries/pal (286 for the owner's 22-pal box)
   into a list the game leaves empty, breaking in-game work assignment.
   FIXED: prune zero-rank entries on load; `SetSuit` creates an entry only
   for a non-zero bonus and removes it at zero. Loading a corrupted save in
   the new build and saving REPAIRS it (286 → 0, verified).
2. `onselect` set each suitability spinbox value before its minimum, so a
   stale minimum from the previous pal clamped the value; a later click wrote
   the wrong value back (the "flip-flop"). FIXED: configure range before
   value; `setsuits` bails while `is_onselect`.
**Owner recovery:** open the real GlobalPalStorage.sav in the new build and
save (game closed) — the phantom entries are pruned out. Session-backup keeps
the pre-edit copy in a PalEdit-backups/ folder just in case.

### TOMORROW — first job: the work-suitability EDITOR (finishes the PR blocker)
Owner wants: an EXACT readout of every pal's work suitabilities AND the
ability to fully adjust ALL of them (natural + modded, e.g. give Penking a
ranch level it doesn't have naturally). Current UI is 13 spinboxes
(base+added, min=species base, to=5). Needed:
- Show, per suit, the effective level clearly (species base + AddRank bonus),
  ideally labelled so you can SEE which are natural vs added.
- Allow setting any suit 0..N freely (a "standard vs unrestricted" toggle like
  the ability pickers — default caps at 5, unrestricted goes to 10 for the
  owner's make-anything pals). Data path: `GotWorkSuitabilityAddRankList`
  rank = desired_total − species_base (already how SetSuit-via-setsuits works;
  just widen the range and relabel).
- INVESTIGATE (open question): does the game actually grant a suitability the
  pal has NO species base in, purely from a positive AddRank? If not, giving
  Penking ranch may need a different field. Candidates seen on real pals:
  `CurrentWorkSuitability`, `WorkSuitabilityOptionInfo`,
  `WorkSuitabilityOverflowGrantedRankList`. Test in-game with a scratchpad
  copy before promising the "give any pal any job" feature.
- SEPARATE latent bug to fix while here: `PalInfo.SetType` (species change)
  writes a `CraftSpeeds` field that real 1.0 saves DO NOT have (verified:
  0 pals have it). It's probably a pre-1.0 leftover and may itself break work
  calc on species-changed pals. Decide whether to stop writing CraftSpeeds
  entirely (likely yes) — check the game reads work suitability only from
  species data + GotWorkSuitabilityAddRankList.

### TOMORROW — remaining roadmap (after the suitability editor)
- `feature/species-browser` (#14): in-game-style popup — element + work-suit
  filters, name search, ICON thumbnails, category buckets (obtainable via
  DeckIndex>=0 / tower boss via TowerBoss / NPC via Human). Reuse for the
  species selector AND the palbox list. NPCs stay (owner catches merchants).
- `feature/stats-panel` (#18): computed standard stats near the portrait,
  raised/lowered indicators, collapsible fine-tune section. Fold in the
  soul/condensation "unrestricted caps" toggle (research done: souls 0-20 ok,
  condensation 1-5 game max, work-suit cap 5 is the one to raise).
- `feature/ui-facelift` (#19): modernize the whole layout last.
- THEN: write CONTRIBUTIONS.md and open the PR.

## Branch workflow (agreed 2026-07-26 — SUPERSEDES all earlier roadmap notes)

Goal: upstream (EternalWraith) should be able to cherry-pick features with a
clear record; owner gets solid fallback points if a feature breaks.

- `main` — integration line the owner's exe is built from. Merge finished
  features with `--no-ff` so each feature is one visible bubble.
- `base/palworld-1.0` — marker at the 1.0 core update (a13940f).
- `feature/<name>` — one branch per feature, branched from current main,
  self-contained commits with PR-quality messages. Test on scratchpad save
  copies BEFORE merging. Keep the branch after merge (cherry-pick record).
- When everything ships: write `CONTRIBUTIONS.md` mapping every feature →
  branch/commits with a summary (the "small document" for the upstream PR).
- Quality bar: industry best practice, no corner-cutting, comment the code,
  fix genuinely-out-of-place things when touched — without breaking core.

## Feature roadmap (owner's list, 2026-07-26 — OVERRIDES earlier notes where they differ)

1. `feature/session-backup` — automatic backup of the loaded .sav, once per
   edit session, before the first write.
2. `feature/palbox-add-remove` — add NEW pals to the Global Palbox (default
   species: a turtle pal — owner is "The Mystic Testudine", quiet nod), fix
   clone in storage mode (find CharacterID=="None" slot, deepcopy selected
   SaveParameter in, PRESERVE target slot's SlotId, fresh InstanceId GUID,
   loaddata to refresh), and delete (reset slot back to a None template).
3. `feature/nickname-edit` — rename pals (1.0 also has FilteredNickName +
   LastNickNameModifierPlayerUid — check whether they must be set together).
4. `feature/species-browser` — in-game-style browser for BOTH the species
   selector and the palbox list: element + work-suitability filters, name
   search, icon thumbnails next to names (analyze the in-game filter menu
   structure for layout). Category toggle: **obtainable pals / boss pals /
   NPCs** — NPCs STAY available (owner: you can catch merchants and use them
   in base!), just sorted into their own bucket. DeckIndex>=0 = obtainable,
   TowerBoss flag, Human flag = NPC bucket.
5. `feature/attack-search` — same browser aesthetic: filter by element, sort
   by damage, 3-tier toggle: learnset-only / + fruit-teachable / ALL attacks.
   Never hard-restrict — "make it clear where the standard and whacko line
   stands", don't stop anyone from going whacko.
6. `feature/passive-search` — grouped by effect type with a blurb of what
   each adjusts (PassiveDescriptions), updates the visual attribute
   indicators, toggle natural-for-this-pal / all passives.
7. Rank/soul caps: UI caps enhancement at 5 but owner says pals reach 10 via
   mutations — research the real 1.0 caps in game data (psp) and raise;
   toggle standard vs unrestricted attribute editing.
8. `feature/stats-panel` — near the portrait: computed standard attributes
   for the level, indicators for raised/lowered stats, collapsible detailed
   section for fine-tuning breeding-relevant values.
9. `feature/ui-facelift` — modernize the whole UI last ("forgotten 2005
   software"), free rein on layout.
10. RESEARCH: upstream's "players who haven't joined in a while can break
    your save" warning — likely about Level.sav worlds referencing missing
    Players/*.sav; determine if it applies at all to GlobalPalStorage-only
    editing (owner suspects stale; verify and document).

Owner explicitly does NOT want upstream's open-issue backlog solved — only
this list. Palworld Save Pal (oMaN-Rod, open source) may be consulted for
implementation insight, but PalEdit stays streamlined and compact.

## Publishing

Owner publishes via GitHub Desktop (repo folder renamed src→PalEdit so the
default repo name is right). Upstream remote = `upstream`. Owner's
Nexus/Vortex flow unchanged — they launch the local exe, not a download.
