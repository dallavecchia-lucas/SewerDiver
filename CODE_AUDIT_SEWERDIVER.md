# Code Audit — SewerDiver: Descent (`sewerdiver420_v2.html`)

Scope: the uploaded single-file build (`SEWER DIVER — DESCENT`, ~4300 lines: ~1080
lines CSS, ~3140 lines JS). This file is not part of the repo's original game
(`index.html` is a different game, "Kanal Mrtvih"); the audited build now lives
alongside it as **`sewerdiver-descent.html`**. Focus: efficiency, redundancy,
and — as specifically requested — the infinite procedural level-generation
mechanism.

## Status: findings applied to `sewerdiver-descent.html`

All five numbered findings and the general-redundancy sweep below were
implemented directly in `sewerdiver-descent.html`, with two items left as
explicit, documented decisions rather than auto-applied (see "Deferred"
below). Verification: `node --check` on the extracted script, a normal
playthrough smoke test (title → boot-tutorial skip → start → move/mine, zero
console/page errors), and a synthetic stress test that force-generated and
rendered **62 tiers deep** (`growTier`/`update`/`render`/`doRespawn` called
directly via a debug hook) with zero console/page errors — the existing
death/lose flow ("you dived 3269m — 62 levels deep") fired correctly when the
forced off-air-line oxygen drain ran the player out of health, confirming the
new code paths hold up well past any depth a normal playthrough would reach.

| # | Finding | Status |
|---|---|---|
| 1 | Unbounded thermal-vent loop in `genTier` | **Applied** — capped at `Math.min(i,6)`, matching `hazFor`'s pattern |
| 2 | Flat unbucketed entity arrays scanned in full every frame | **Applied** — every push site now tags `.tier`; `update()`, `updateCreatures()`, `render()`, `drawKelp()`, `drawScenery()`, and the mixer/scrap pickup loops (now a shared `collectNear()`) all filter to `[tAt(player.y)-1, tAt(player.y)+2]` via new `entTierLo()`/`entTierHi()` helpers |
| 3 | `doRespawn()` quadratic-ish full-array re-filtering | **Applied** — rewritten to one `O(total)` counting pass over `oreCells`/`mixers` instead of `tiers × .filter()` calls |
| 4 | `map` grows forever, never released | **Deferred** — the audit itself flagged this as a product decision (does upward backtracking need to stay meaningful?), not a clear-cut bug; left as-is rather than guessing |
| 5 | `_oreCanv`/`_oreLv` caches grow forever | **Applied** — new `evictFarCache()` mirrors the existing `prerenderTiles` tier-window eviction in `render()` |
| — | ~21 duplicated particle-spawn blocks | **Applied** — collapsed into `burst()` (sparks) / `bubbleBurst()` (bubbles); two single bespoke bubble spawns (diver's trail, ambient bubbles) left as-is since they aren't actually duplicated |
| — | Duplicated pickup scan (`update()` vs `updateCompanion()`) | **Applied** — shared `collectNear(list,x,y,r,onHit)` |
| — | Per-frame `hex2rgb`/`idTier` recomputation for static data | **Applied** — `RES[id].rgb` and `RES[id].tier` cached at creation (`R()`) and on re-skin (`setResCol()`); `drawOre`/`drawMixer` read the cached fields |
| — | `plotPxC`/`plotShimmer` duplicated coordinate transform | **Applied** — both now call a shared `faceXY(face,perp,par)` |
| — | `SCENE`/`ORE_ARCH` likely-dead fallback tables | **Left untouched** — audit flagged these as low-confidence ("likely dead… unless there's a path I missed"); removing code I'm not certain is unreachable wasn't worth the risk for a cosmetic cleanup |

## Original audit (as written before the fixes above)

## How the infinite descent actually works

- `genWorld()` builds the 4 base tiers and calls `genTheme()`, which assigns 4
  distinct biome archetypes (`ARCHETYPES`) and re-skins every resource's name/
  colour/shape for that run (`skinTier`).
- As the diver nears the bottom of the deepest generated tier, `update()` calls
  `ensureDepth()` every tick, which calls `growTier(n)` for each new tier needed:
  it extends the tile map (`growMapTo`), invents that tier's resource IDs and
  recipes (`buildTierData`), assigns it an archetype (`skinTier(n, pickArch())`),
  bakes its background texture, carves geometry/hazards/loot (`genTier`), and
  spawns creatures and scenery (`genCreatures`, `genTierEnv`).
- Per-tier gear stats and hazard counts scale via simple open-ended formulas
  (`gearStats(n)`, `hazFor(n)`), so there's no hard floor.

This is a solid, deterministic-per-id-but-randomized-per-run design. The
problems below are specifically about what happens to **cost** (CPU and
memory) as the number of generated tiers grows without bound — which is the
whole point of "infinite."

## Findings tied directly to infinite generation (highest priority)

### 1. Thermal-vent placement scales unbounded with tier depth (real bug)
`genTier()`:
```js
if(i>=1){for(let k=0;k<i;k++){ ... while(tries<60){ ... } }}
```
Every other hazard in the file is capped (`hazFor`: elec ≤6, blob ≤5, sludge
≤26), but thermal-vent count is literally the tier index `i` — uncapped. At
tier 50 that's 50 placement attempts × up to 60 tries each; at tier 300 it's
300× that, all run synchronously inside `growTier()` on the main thread the
moment the diver crosses into a new tier. This is the one clear-cut
correctness gap in the otherwise-capped hazard system, and it will produce a
growing hitch every time a new tier is generated the deeper the run goes.
**Fix:** cap it the same way `hazFor` caps everything else, e.g.
`Math.min(i, 6)` thermal pools per tier.

### 2. World-entity arrays are flat, unbucketed, and scanned in full every frame
`oreCells`, `mixers`, `nodes`, `blobs`, `creatures`, `scraps`, `kelp`,
`scenery`, `bulkheads` are single global arrays that every tier's worth of
content gets pushed into, and they are **never pruned**. Both `update()` and
`render()` iterate every one of these arrays in full, every frame:
- `update()`: the ore-proximity scan, fungus scan, mixer/scrap pickup loops,
  `updateCreatures()`, the electric-node loop, and the blob loop all walk the
  *entire* array regardless of which tier the diver is in.
- `render()`: `for(const o of oreCells)drawOre(o)` etc. — each draw call does
  correctly bail out early on an off-screen bounds check, but the array
  iteration itself, and the per-item coordinate math before that check, is
  paid for every item in every generated tier, not just the visible ones.

`bases[]` is already indexed by tier (`bases[i]`), proving the codebase
already has the right shape for this. The other arrays don't follow that
pattern. As a one-way descent game this means per-frame cost grows roughly
linearly with total depth reached — fine for a few dozen tiers, but it's
exactly the kind of cost an "infinite" mode will eventually surface in long
sessions (visible as gradual frame-time creep, not a sudden cliff).

**Fix (the single highest-value change here):** bucket these arrays by tier,
the same way `bases` already is (e.g. `oreCellsByTier[i]`, or simpler: keep
one flat array per category but also keep a `tier` field already present
on most entries and only iterate tiers in `[tAt(player.y)-1, tAt(player.y)+1]`
for both update and render). That turns every one of these loops from O(total
entities ever generated) into O(entities near the player) — effectively
constant, independent of how deep the run has gone.

### 3. `doRespawn()` re-scans the *entire* world per tier, per resource id
```js
function doRespawn(){const cur=tAt(player.y);
  for(let t=0;t<=cur;t++){const T=TIERS[t];
    for(const id of T.raw){if(oreCells.filter(o=>o.resId===id).length<ORE_FLOOR)...}
    for(const id of T.mix){if(mixers.filter(m=>m.resId===id).length<MIX_FLOOR)...}}}
```
This runs every `RESP_INT` (8s) and, for **every tier from the surface down to
the current one**, filters the *entire* `oreCells`/`mixers` arrays (which
themselves contain every tier's entities) three times each. Cost is
`O(tiersDescended × totalEntities)`, i.e. effectively quadratic in depth. At
shallow depth this is invisible; by tier 100+ it's doing tens of thousands of
redundant comparisons every 8 seconds for tiers the diver finished hours ago
and floors that already have plenty of stock are still re-filtered every
pass. Same root cause as #2, same fix: bucket by tier (or maintain a running
per-tier-per-id count instead of recomputing it via `.filter().length`).

### 4. `map` (the tile grid) grows forever and is never released
Unlike the band-canvas cache, which is explicitly pruned ("drop bands far from
the diver" — `render()` deletes `prerenderTiles[i]` outside `pti±2`), the
underlying `map` 2D array that those canvases are baked from is appended to
forever (`growMapTo`) and never trimmed. For a genuinely long infinite run
this is unbounded memory growth (every tier adds `34 × 44` more cells,
permanently). This is a reasonable design tradeoff if backtracking upward
must remain possible (the diver isn't hard-locked into one-way travel), but
it's worth being explicit about: the visual cache is depth-windowed, the data
it's drawn from is not. If upward travel is never actually meaningful in
practice (oxygen/air-line mechanics effectively force one-way descent), the
map and the entity arrays could both be pruned far above the diver and
regenerated/discarded with the same lazy-rebuild pattern already used for
`prerenderTiles`.

### 5. Per-id render caches (`_oreCanv`, `_oreLv`) also grow forever, but are safe to cap
`getOreCanvas`/`oreLevels` memoize by `id+variant+face` / `id+variant` and are
only ever cleared on `genTheme()` (new game). Since these are 100%
deterministic from `(id, variant, RUN_SEED)`, they're cheap to evict and
rebuild — unlike the entity arrays in #2-3, there's no game-state-loss risk.
**Fix:** apply the same tier-window eviction used for `prerenderTiles` (delete
cache entries for ids whose tier is more than ~2 tiers from the diver); they
will be regenerated identically on demand if revisited.

## General efficiency / redundancy findings (not specific to infinite gen)

- **Repeated spark/bubble particle-spawn boilerplate.** The exact pattern
  `for(let i=0;i<N;i++)particles.push({type:'spark',x:...,vx:(Math.random()-.5)*S,vy:(Math.random()-.5)*S,life:L,max:L,col:C,size:2})`
  appears ~21 times (e.g. `update()` blast handling, `fungusGrab`,
  `applyGear`, `activateBase`, `mgExplode`, `mgCollect`, mixer/scrap pickups).
  A single `burst(x,y,{n,spread,life,col,type})` helper would remove ~20
  near-duplicate blocks and make tuning consistent.
- **Pickup logic duplicated between `update()` and `updateCompanion()`.** Both
  contain separate loops that scan `mixers`/`scraps` by distance and apply the
  same add-to-inventory + sfx + splice pattern with only the radius differing
  (12 vs 22). Worth factoring into one `tryCollect(list, x, y, r, onHit)`.
- **Color conversion recomputed every frame for static colors.** `drawOre`,
  `drawMixer`, and `drawScrap` all call `hex2rgb(col).join(',')` (and `shade`)
  on every visible item, every frame, even though a resource's color is fixed
  for the whole run once `skinTier` runs. Precomputing `RES[id].rgb` once in
  `skinTier`/`R()` avoids re-parsing the hex string 60×/sec per visible item.
- **`idTier()`/`idSlot()` parse the id string on every call.** `idTier` does
  `parseInt(id.slice(1,-2))` and is called from hot per-frame draw paths
  (`drawOre`, `oreShapeFor` via `getOreCanvas`/`oreGridSVG`). Since an id's
  tier never changes, caching it on `RES[id].tier` at creation time (in `R()`/
  `buildTierData`) turns a string-parse into a property lookup.
- **`plotPxC` (canvas) and `plotShimmer` (live ctx) duplicate the same
  face-aware coordinate transform** (`up`/`down`/`left`/`right` → `rx,ry`).
  They differ only in which drawing target they write to; a single helper
  returning `[rx,ry]` (or taking the target context as a parameter) would
  remove the duplicated branch logic.
- **Likely-dead fallback paths**, given current call order: `SCENE` (the
  legacy 4-archetype scenery table) is only used via
  `(a&&a.scenery)||SCENE[...]` in `genTierEnv`, but `a` (`THEME[n]`) is always
  populated by `skinTier()` before `genTierEnv()` runs, for every tier,
  base or generated. Same for `ORE_ARCH` as the fallback in `oreShapeFor` —
  `THEME[tier].oreShapes[s]` is always set before any ore is drawn. Both look
  safe to delete (or at least worth a comment noting they're defensive-only)
  unless there's a code path that calls these before theming runs that wasn't
  obvious from a static read.
- **`RES`/`ALLRAW` grow for the life of a run and are never trimmed**, even
  though tiers far above the diver are (in practice) no longer relevant.
  Low priority — this is small per tier (3 ids) — but it's the same pattern
  as findings #2-4 and would be swept up by the same tier-windowing fix.

## Suggested priority order

1. Cap the thermal-vent loop in `genTier` (#1) — one-line fix, removes an
   actual unbounded-growth bug.
2. Bucket world-entity arrays by tier and restrict per-frame update/render and
   `doRespawn` to a small window around the diver's current tier (#2, #3) —
   the architectural change that actually makes "infinite" scale flat instead
   of linear/quadratic in depth.
3. Apply the existing `prerenderTiles` eviction pattern to `_oreCanv`/`_oreLv`
   (#5) — small, mechanical, reuses a pattern already in the codebase.
4. Decide intentionally whether upward backtracking needs to stay supported;
   if not, extend the same windowing to `map` itself (#4).
5. Sweep the smaller redundancies (particle-spawn helper, shared pickup
   helper, cached rgb/tier, unified face-coordinate transform, dead fallback
   tables) — pure cleanup, no behavior change.

None of the above are correctness bugs the player would notice at normal play
depths (roughly tiers 1–20); they're scaling costs that compound specifically
because the game's core premise is unbounded descent, which is why they're
called out ahead of general style nits.
