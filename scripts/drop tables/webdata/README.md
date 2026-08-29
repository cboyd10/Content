# NPC drop-source web data (issue #57)

Hand-transcribed, **display-only** flattening of the live `[ai_queue3,...]`
drop-table scripts in `../scripts/*.rs2`, for the `/items` "NPC Drop Sources"
section on the engine's website. These files are **not** read by the game
server (`.dropdata` is not a recognized pack extension - the build pipeline
ignores it entirely) - only `ItemSourceIndex.ts`'s new lightweight parser
reads them. No live gameplay script was touched to build this feature.

## Format

Same `[name]` / `key=value` shape as the game's own `.dbrow`/`.npc` configs,
reusing `drop_table.dbtable`'s column layout (`total,int` /
`drop,namedobj,int,int,LIST`) for convention consistency, plus two extensions
specific to this display-only format:

- `data=subtable,<table>,<weight>` - composes another table file in this same
  directory at the given weight out of this table's `total` (used to flatten
  the shared procs `~randomherb`/`~randomjewel`/`~ultrarare_getitem`/
  `~megararetable` without re-deriving their distributions per calling NPC).
- `data=guaranteed,<item>,<qty>` - a 100%-per-kill drop, whether from a
  `param=death_drop,<item>` config entry or an unconditional `obj_add(...)`
  in the script body outside any random roll (e.g. Bear's fur/meat).

A table's listed `drop`/`subtable` weights may sum to **less** than its
declared `total` - the shortfall is a real, source-accurate "no additional
drop" outcome (e.g. Guard's explicit `// nothing dropped` branch, or the
tail of Man's/`~randomjewel`'s/`~megararetable`'s `if`-chains that no
`else if` covers). It is never renormalized away.

## Excluded from every table here (per the 2026-08-24 grilling session)

- Ring of Wealth-gated branches in `~randomjewel` (treated as the
  not-wearing-RoW / base case).
- The Legends' Quest-gated branch in `~randomjewel` that unlocks
  `~megararetable` early (treated as quest-incomplete / false).
- `~trail_easycluedrop`/`~trail_checkmediumdrop`/`~trail_mediumcluedrop`
  tertiary clue-scroll rolls (a separate NPC-agnostic system).
- `npc_findhero` (a loot-eligibility gate on *whether* a kill drops anything
  attributable to a player, not on *which* drops are possible - doesn't
  affect table contents).

## `~randomjewel`'s `coordz(coord) > 6400` branch

Resolved by checking real spawn coordinates rather than guessing: every
`firegiant` spawn in `content/maps/*.jm2` (npc id 110) has `z > 6400`
(mapsquare rows 148/149/154/161, all Wilderness), so this branch resolves to
`chaos_talisman` for every pilot NPC that can reach `~randomjewel`
(`fire_giant`, via `~ultrarare_getitem`). `randomjewel.dropdata` transcribes
that resolved branch directly rather than modeling the coordinate check.

## Trigger dispatch

`triggers.dropdata` maps each `[ai_queue3,<trigger>]` key used by this
issue's 7 pilot NPCs to the table file that computes its drops. Category
trigger keys (leading underscore) are resolved to their member debugnames at
runtime by `ItemSourceIndex.ts` (via `content/pack/category.pack` + every
`.npc` file's `category=` line, **including** `scripts/_unpack/<revision>/`
dumps - unlike `ItemSourceIndex.ts`'s existing shop/spawn loaders, which
skip `_unpack` for their own noise-reduction reasons, this resolution must
include it: `category.pack`'s citizen/bear/guard/cow/chicken categories are
only fully populated there (e.g. `man`/`man2`/`man3`/`woman`/`woman2`/
`woman3` have no `category=citizen` `.npc` entry anywhere else), and
`_unpack` content **is** compiled into the live pack (`PackFile.ts`'s
`_unpack` check only suppresses a folder-naming lint, `FsCache.ts`'s
`listDir` walks it like any other directory) - confirmed for `man`
specifically: only `_unpack/225/all.npc` defines `[man]`, so there's no
duplicate-revision collision to resolve.
