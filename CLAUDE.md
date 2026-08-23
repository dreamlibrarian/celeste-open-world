# Celeste (Open World) apworld

An Archipelago randomizer world for Celeste. This directory is a **separate git repository**
(`/Users/triss/dev/celeste_open_world`), symlinked into an Archipelago checkout at
`worlds/celeste_open_world` so it can be loaded and tested. Its git history is independent of the
Archipelago monorepo's — `git log`/`git status` run from inside this directory show *this* repo's
commits, not Archipelago's. `git stash` from the Archipelago repo root will fail with "beyond a
symbolic link"; `cd` into this directory (or its real path) to use git normally.

Because it's symlinked, an Archipelago checkout's own git state (branch, detached HEAD, etc.) is
irrelevant to this world's code — don't infer this repo's status from the outer repo's `git status`.

## Layout

- `Levels.py` — data model (`Level`, `Room`, `PreRegion`, `RegionConnection`, `Door`,
  `LocationType` enum) and static per-goal-area name/option lookup tables.
- `data/CelesteLevelData.json` — source-of-truth level/room/door/access data.
- `data/ParseData.py` — reads the JSON and **generates** `data/CelesteLevelData.py` (run as
  `__main__`, `data_file = open('CelesteLevelData.json')` → `out_file = open("CelesteLevelData.py", "w")`).
  `CelesteLevelData.py` is ~10k lines of generated `Room(...)`/`RegionConnection(...)` construction
  calls — **don't hand-edit it**; edit the JSON and rerun `ParseData.py` for room/door/access-list
  corrections. Logic changes that aren't raw per-room access-list edits (option-gated behavior,
  cross-cutting item substitutions, etc.) belong in the Python layer below instead.
- `Items.py` — item id/classification tables. `interactable_item_data_table` holds the "base"
  interactable items; `add_interactable_to_table` auto-derives per-level / per-side /
  per-level-and-side variants (driven by the `split_interactables` option) for every item that
  actually appears in a `Level.items` set — this happens automatically in
  `Locations.generate_location_table()`, not by hand.
- `Locations.py` — builds the location table and, in `create_regions_and_locations`, the actual
  `Region`/`Entrance` graph and access rules (using `rule_builder.rules.Has`/`HasAll`/`Or`) from the
  `Level`/`Room`/`PreRegion` data. `convert_item`/`convert_item_list`/`convert_item_list_list` apply
  the `split_interactables` name conversion to raw item-name strings from `possible_access` lists.
- `Options.py`, `__init__.py` (the `World` subclass: `generate_early`, `create_items`,
  `create_regions`, `fill_slot_data`, etc.)

## Rule-building conventions

- `possible_access` (and its `_vanilla` / `_assist` siblings, selected by the `logic_difficulty`
  option) is `list[list[str]]`: an OR of AND-groups of `ItemName.*` string constants. Rule
  construction always goes through `convert_item_list_list` → `Has`/`HasAll`/`Or` in
  `create_regions_and_locations`; there should be no bespoke `CollectionState`-inspecting rule
  functions in this world (a previous one, `summit_a_badeline_boosters_rule`, checked
  `strawberry_count % 7` and was **not monotonic** — collecting more strawberries could revoke
  already-granted access, which reliably broke Summit A generation. It was replaced by real
  per-altitude items; see "per_altitude_boosters design" below). If you're tempted to write a
  custom access-rule function instead of a real item, that's very likely the wrong move here —
  prefer expressing the requirement as an item and let the existing `Has`/`HasAll`/`Or` path handle it, since Archipelago's fill assumes accessibility is monotonic in collected items.
- **Gotcha:** any name you add to `interactable_item_data_table` directly (rather than through the
  normal per-level `Level.items` pipeline) will *also* get silently rewritten by
  `convert_item`'s `split_interactables` logic if it happens to still be present in that table when
  `convert_item` checks membership — because that check is a plain `in interactable_item_data_table`,
  not "was this item introduced through the normal per-level split machinery". If you register a
  one-off/synthetic item name there (as `per_altitude_boosters` does), you must explicitly exclude it
  in `convert_item`, or `split_interactables != None` will silently rename it in rules but not in the
  actual pool (item exists as name A, rule asks for name B) → mass unreachability. Caught and fixed
  for `summit_a_altitude_booster_item_names` in `Locations.convert_item`; watch for this pattern
  whenever adding another synthetic item outside the standard split pipeline.

## per_altitude_boosters design (as of this writing)

Summit A (`7a`)'s rooms are named `7a_<letter>-...` where the letter groups rooms into the level's
real in-game altitude checkpoints: `a`=Start, `b`=500 M, `c`=1000 M, `d`=1500 M, `e`=2000 M,
`f`=2500 M, `g`=3000 M (see the `Room(...)` checkpoint args in `CelesteLevelData.py` around the
`7a_*` entries). When `per_altitude_boosters` is on, the single `Badeline Boosters` item is dropped
from Summit A's item set (never added to `self.active_items`, so it's never placed anywhere or
sent over the network — see `generate_early`) and access is instead expressed via 7 items named
`"Badeline Boosters (<altitude>)"` (`Items.summit_a_altitude_sections` /
`summit_a_altitude_booster_item_name`). `Locations.apply_altitude_boosters` still substitutes the
right section-specific name into a room's `possible_access` lists (keyed off `room_name[3]`, the
letter right after the `"7a_"` prefix) before the normal `Has`/`HasAll`/`Or` rule-building runs —
that part is unchanged from the real-item design.

**What changed from the real-item design**: these 7 items are no longer real, randomly-placed
pool items (they're deliberately *not* registered in `interactable_item_data_table` anymore, so
`create_item()` falls into its `code=None` fallback for them). Instead, each is an **event**:
`create_items()` (right after `self.strawberries_required` is computed) creates 7 locations
(`address=None`) in the Menu region via `menu_region.add_locations({event_name: None},
CelesteLocation)`, `place_locked_item`s the corresponding event item there, and sets that
location's own access rule to `Has(ItemName.strawberry, count=threshold)`. Thresholds are
`ceil(strawberries_required * (tier_index+1) / 7)` — non-decreasing, and the last tier's threshold
always equals `strawberries_required` exactly — computed once and exposed to the mod via
`fill_slot_data["per_altitude_booster_thresholds"]` (a 7-element list) so the client never has to
re-derive the formula itself.

This is **not** the `strawberry_count % 7` mistake described above: `Has(item, count=N)` is a
plain monotonic threshold (once you have ≥N strawberries you always have ≥N as you collect more),
so the derived event is monotonic too. It reuses the standard `Has`/`HasAll`/`Or` rule-builder
pipeline via a real (event) item — it isn't a bespoke `CollectionState`-inspecting function — so it
doesn't violate the "prefer expressing the requirement as an item" guidance above; the item itself
is just granted by a rule instead of found in the pool.

**Testing gotcha**: because these items are sweep-derived, `assertAccessDependency` doesn't work
for them — collecting the Strawberries needed to satisfy an event's rule auto-collects the event
too (`CollectionState.collect` sweeps for advancements after every collect), so "hold everything
except this one item" is incoherent by construction for an event. See
`test/test_per_altitude_boosters.py`'s `test_altitude_booster_thresholds_are_monotonic` for the
right pattern: build a `CollectionState` by hand, collect `Strawberry` items one at a time, and
assert each event's `state.has(...)` flips from `False` to `True` exactly at its threshold.

## Testing

Tests live in `test/` (added — none existed before). Pattern: `test/bases.py` defines
`CelesteTestBase(WorldTestBase)`; test modules under `test/test_*.py` subclass it. See
[docs/tests.md](../../docs/tests.md) in the Archipelago repo for the general framework
(`assertAccessDependency`, the automatic `test_all_state_can_reach_everything` /
`test_empty_state_can_reach_something` / `test_fill` per test class, etc.).

Run from the **Archipelago repo root** (the `test` package and `venv` both live there, not here):

```bash
cd /Users/triss/dev/archipelago
venv/bin/pytest worlds/celeste_open_world/test -q       # this world's own tests
venv/bin/pytest test/general -q -k Celeste               # generic cross-world tests, filtered
```

For ad hoc exploration/stress scripts outside pytest (e.g. probing `load_logic_data()`,
constructing a `WorldTestBase` subclass by hand, running `Fill.distribute_items_restrictive`
directly across many seeds), run with `PYTHONPATH` pointing at the Archipelago root, e.g.:
`PYTHONPATH=/Users/triss/dev/archipelago venv/bin/python your_script.py` (needed because the script
isn't inside the `test` package itself, so Python won't find it on `sys.path` otherwise).

`WorldTestBase.world_setup()`'s `gen_steps` only runs through `pre_fill` — it does **not** run fill.
To actually exercise `Fill.distribute_items_restrictive` (the strongest signal for "does this
generate reliably"), call it explicitly after `world_setup()`; `test_fill` (auto-run per test class)
does this via a real multiworld, or drive it by hand for a quick reachability/fill probe across many
seeds when checking whether a change destabilizes generation.

## Goal Area lock and fill deadlocks (fixed, but understand this before touching `create_items`)

`lock_goal_area` (default on) connects the Goal Area's start region (and any `goal_checkpoint_names`
region) to Menu behind a single `Has(Strawberry, count=strawberries_required)` requirement — see the
`menu_region.add_exits(...)` calls in `create_items`. Every location whose name contains the Goal
Area's display name (`goal_area_locations` in `create_items`) sits behind that one gate and is
unreachable until the count is met.

Nothing stopped the shuffled item pool (Interactables, Strawberries, Keys, Gems, Checkpoints) from
being placed *inside* that locked cluster, even when a copy was also needed to satisfy an access
rule *outside* it (e.g. a shared-name Interactable used both in an early level and in the Goal Area
when `split_interactables: 0`, or a Checkpoint/Key item for a different level). A copy stranded that
way deadlocks generation: it's needed to progress before the gate opens, but it can't be reached
until after. Archipelago's swap-based `fill_restrictive` (capped at ~2 swap attempts per item) can't
reliably recover from this, which is what actually caused the failures below — it was never a
generic "restrictive fill is flaky for small worlds" issue, and don't reach for `panic_method` or
priority-location tuning to paper over it; marking the locked locations as `LocationProgressType.PRIORITY`
was tried during the investigation and made things *worse*, not better.

**Fixed** in `create_items` (right after `self.active_items` is finalized): whenever
`lock_goal_area` is set and `goal_area_locations` is non-empty, `forbid_items_for_player` is called
on every Goal Area location for the union of `self.active_items`, `ItemName.strawberry`,
`self.active_checkpoint_names`, `self.active_key_names`, and `self.active_gem_names`. This is safe
because `real_total_strawberries`'s sizing math already subtracts `goal_area_location_count` from the
denominator specifically to guarantee enough non-gated locations exist for all of the above — the
bug was that nothing *enforced* that assumption at placement time, only implied it via pool sizing.
If you add a new item category that gets shuffled into the general pool (not `place_locked_item`'d),
check whether it needs adding to `forbidden_in_goal_area` too, using the same reasoning: does a copy
of this item ever gate something *outside* the Goal Area? If yes and it's not already covered, it can
deadlock the same way.

Symptoms if this regresses: `Fill.FillError: No more spots to place N items` during the
Progression fill step, unplaced items skewing toward Strawberries/Interactables/Keys, most unfilled
locations named after the current `goal_area`.

Confirmed before the fix: `goal_area: "the_summit_a"` + `include_a_sides: True` (all other options
default) failed ~25–30% of random seeds; `split_interactables: 1`/`3` combined with a restricted
`goal_area` failed even more reliably; disabling `lock_goal_area` or setting
`strawberries_required_percentage: 0` made failures disappear entirely (this was the key clue that
pointed at the lock rather than at region/rule data). A full sweep across every `goal_area` ×
`split_interactables` × `lock_goal_area` × all sanity options (keysanity/gemsanity/checkpointsanity/
carsanity/binosanity) on, 8 seeds each, is 0/640 failures after the fix.

**Note:** the "sizing math already subtracts `goal_area_location_count`... to guarantee enough
non-gated locations exist" claim above is about deadlock prevention specifically, and still holds
for that purpose. It does *not* mean `real_total_strawberries` itself was ever correctly bounded —
see the next section, a related but distinct bug in the same formula.

## strawberries_required could go negative (fixed, same formula as above)

`real_total_strawberries = min(total_strawberries, location_count - goal_area_location_count -
len(item_pool))` (in `create_items`, the "Strawberries" section) could compute negative. Root
cause: `goal_area_location_count` is a *static* snapshot of every location in the Goal Area, taken
early (right after `create_regions`, before any locking happens). `location_count` by contrast
starts as the total location count and gets decremented throughout `create_items` for every
`place_locked_item`'d location (checkpoints/keys/gems/clutter/breaker boxes when their sanity
option is off, plus the goal item) — including ones *inside* the Goal Area. Subtracting the static
`goal_area_location_count` on top of that double-subtracts every Goal Area location that also got
locked. With checkpointsanity/keysanity/gemsanity off (the defaults) and an active-level selection
where the Goal Area is most or all of it, this reliably went negative.

This matters far beyond the strawberry item count: `self.strawberries_required` feeds every
`Has(Strawberry, count=strawberries_required)` rule in the world — the Goal Area lock, the
Epilogue lock, and the `per_altitude_boosters` events (see above) all use it. A negative count
makes `state.prog_items[...] >= negative_number` trivially true for *any* state, so this bug
silently unlocked the Goal Area, the Epilogue, and every altitude-booster tier regardless of actual
progress — the reported symptom was boosters/goal access unlocking "too easily," not a crash or
generation failure, which is why it went unnoticed by the deadlock-focused sweep above (that sweep
never checked whether `strawberries_required` itself was sane, only whether generation completed).

**Fixed**: recompute the Goal-Area-locations term right before the `real_total_strawberries` line,
counting only locations that are *still* pool-eligible at that point (`loc.item is None`) instead
of the static early snapshot, and clamp the whole expression to `max(0, ...)` as a hard floor
against any other edge case in the same formula. Regression test:
`test/test_strawberries_required.py`, which sweeps Goal-Area-dominates-active-levels configs ×
sanity-option combos × `total_strawberries`/`strawberries_required_percentage` values and asserts
`strawberries_required` (and every `per_altitude_booster_thresholds` entry) is never negative.

Repro that found it: `goal_area: "farewell"`, `include_farewell: "farewell"`,
`include_a_sides: False`, `total_strawberries: 7`, `strawberries_required_percentage: 100` —
active levels collapsed to almost entirely the Goal Area, giving `location_count=2`,
`goal_area_location_count=17` (static), `len(item_pool)=2` at that point →
`real_total_strawberries = min(7, 2 - 17 - 2) = -17`.

## Known pre-existing issues (not caused by recent work, don't re-diagnose from scratch)

- `archipelago.json` currently defines a `version` key, which `test/general/test_world_manifest.py`
  (`test_no_container_version`) flags as invalid per the apworld manifest spec. Pre-existing, unrelated
  to any of the above, not yet fixed.
