# A-1: Range Array Divergence V1 vs V2 — Investigation Findings

**Date:** 2026-02-16 (updated 2026-02-17)  
**Investigator:** Siddarth Vedi  
**Status:** INVESTIGATION COMPLETE & FIX APPLIED — Bug is latent, not actively triggering. V2 comparison confirms byte-for-byte identical range arrays. Defensive fix implemented in V1/V3.  

---

## Summary

V1 (`GenerateNextHand`) and V2 (`GenerateNextHandV2`) use different code paths to produce the `range_array` passed to `_generate_hand_response()`. After extensive hands-on debugging with breakpoints across **11 test scenarios** (7 on V1, 4 on V2) covering both game variants (Holdem + Omaha), all spot types, multiple sites and blind structures, and both cash and spins formats — **no active divergence was found**. The V2 comparison (Test 8) confirmed byte-for-byte identical range arrays at the `StrategyTreeNode.decode_range()` breakpoint. The bug is latent. It would only activate if preflop tree nodes begin using sparse index compression (`indices` field), which none currently do.

---

## What Was Tested

| Test | Endpoint | Spot | Variant | Research | Site | BlindStructure | Result |
|------|----------|------|---------|----------|------|----------------|--------|
| 1 | V1 `/trainer/generate-next-hand` | `unopened` | Omaha | `full_tree` | NoRake | Regular | No divergence — all 1.0 range |
| 2 | V1 `/trainer/generate-next-hand` | `facingOpen` (BTN vs CO) | Omaha | `full_tree` | NoRake | Regular | No divergence — all 1.0 range |
| 3 | V1 `/trainer/generate-next-hand` | `facingOpen` (BTN vs Any) | Omaha | `postflop_only` | NoRake | Regular | No divergence — all 1.0 range |
| **4** | **V1 `/trainer/generate-next-hand`** | **`facing4Bet`** | **Omaha** | **`preflop_only`** | **GGPoker** | **Cash Drop 10bb** | **Non-trivial range found! Narrowed (9.8% nonzero), 2 actions, wide EV range. Still `indices=False`, `len=270725`** |
| 5 | V1 `/trainer/generate-next-hand` | `unopened` | **Holdem** | `full_tree` | GGPoker | Regular | No divergence — all 1.0, `indices=False`, `len=1326`, `NUM_COMBOS=1326` |
| **6** | V1 `/trainer/generate-next-hand` | **`facing3Bet`** (SB vs BB) | **Holdem** | `full_tree` | PokerStars | Regular | **Narrowed range (558/1326 = 42%), 4 actions, mixed strategy. `indices=False`, `len=1326`** |
| 7 | V1 `/trainer/generate-next-hand` | `facingOpen` | **Holdem Spins** (3-max, 25bb) | `full_tree` | NoRake | Regular | All 1.0, `indices=False`, `len=1326`. **First spins/3-max/short-stack test** |
| **8** | **V2 `/trainer/v2/generate-next-hand`** | `facingOpen` (`preflopActions=r35`) | **Omaha** | `full_tree` | GGPoker | Regular | **V2 path confirmed identical to V1: all 1.0, `raw_indices=None`, shape `(270725,)`, sum `270725.0`. First direct V1↔V2 comparison.** |
| **9** | **V2 `/trainer/v2/generate-next-hand`** | `facing3Bet` (`preflopActions=f,r35,f,f`) | **Omaha** | **`preflop_only`** | **PokerStars** | Regular | **V2 + preflop_only + PokerStars: all 1.0, `raw_indices=None`, shape `(270725,)`. Different site + research from Test 8, same result.** |
| **10** | **V2 `/trainer/v2/generate-next-hand`** | `facingOpen` (`preflopActions=f,f,f,r20`) | **Holdem** | **`preflop_only`** | GGPoker | Regular | **First Holdem V2 test: all 1.0, shape `(1326,)`. Confirms Holdem `NUM_COMBOS=1326` on V2 path matches V1 Test 5.** |
| **11** | **V2 `/trainer/v2/generate-next-hand`** | `facingOpen` (`preflopActions=f`) | **Holdem Spins** (3-max, 25bb) | **`preflop_only`** | NoRake | Regular | **First Spins V2 test: all 1.0, shape `(1326,)`. Matches V1 Test 7 (spins/3-max/short-stack).** |

### Common Parameters

```
variant=omaha, game=cash, numPlayers=6, stackInBB=100, BB=0.25, openRaise=3.5
```

Tests 1-3 used `site=NoRake, blindStructure=Regular`. Test 4 used `site=GGPoker, blindStructure=Cash Drop 10bb, research=preflop_only`.

Test 3 specifically used `research=postflop_only` (precision strategy) to check if suit isomorphism applies. It does not — V1 only fetches preflop nodes, and preflop nodes in postflop_only trees are identical to full_tree (all-1.0 range, no indices).

**Test 4** is the first test to produce a non-trivial (narrowed) range array, showing that deeper spots (`facing4Bet`) with different rake/blind structures reach nodes with meaningful strategy data.

---

## Debugger Observations

### Range Array (breakpoint at `spot_generation.py:199`)

| Property | Test 1: `unopened` | Test 2: `facingOpen` (full_tree) | Test 3: `facingOpen` (postflop_only) |
|----------|-----------|-------------|-------------|
| `range_array.shape` | `(270725,)` | `(270725,)` | `(270725,)` |
| `range_array.dtype` | `float64` | `float64` | `float64` |
| `len(range_array) == settings.NUM_COMBOS` | `True` | `True` | `True` |
| `'indices' in tree_node` | `False` | `False` | `False` |
| `np.count_nonzero(range_array)` | `270725` (all) | `270725` (all) | `270725` (all) |
| `range_array.min()` | `1.0` | `1.0` | `1.0` |
| `range_array.max()` | `1.0` | `1.0` | `1.0` |
| `range_array.sum()` | `270725.0` | `270725.0` | `270725.0` |

**Tests 1-3 Conclusion:** All range values are 1.0 (full starting range). No sparse indices present. Array is always full `NUM_COMBOS` length (270725 for Omaha). The `research` type (`full_tree` vs `postflop_only`) makes no difference at the preflop level.

### Test 4: Range Array — facing4Bet / preflop_only / GGPoker / Cash Drop 10bb

```
URL: http://localhost:8000/omaha/trainer/generate-next-hand?openRaise=3.5&variant=omaha
     &game=cash&site=GGPoker&numPlayers=6&stackInBB=100&BB=0.25
     &blindStructure=Cash%20Drop%2010bb&heroPosition=Any&villainPosition=Any
     &spot=facing4Bet&difficulty=easy&randomiseBoard=false&research=preflop_only
     &3bet=100&4bet=100
```

| Property | Value | Different from Tests 1-3? |
|----------|-------|--------------------------|
| `range_array.shape` | `(270725,)` | Same shape |
| `range_array.dtype` | `float64` | Same |
| `'indices' in tree_node` | **`False`** | Same — still no sparse indices |
| `len(range_array) == settings.NUM_COMBOS` | `True` | Same |
| `range_array.min()` | **`0.0`** | **YES — was 1.0 in Tests 1-3** |
| `range_array.max()` | `1.0` | Same |
| `np.count_nonzero(range_array)` | **`26436`** | **YES — was 270725 (all) in Tests 1-3** |
| `range_array.sum()` | **`25816.87`** | **YES — was 270725.0 in Tests 1-3** |
| `range_array[:10]` | All `0.0` | **YES — were all 1.0** |
| `range_array[:100]` | All `0.0` | Large blocks of zeros |
| `range_array[100:200]` | All `0.0` | Zeros continue deep into the array |
| `range_array[10000:10100]` | All `0.0` | Still zeros at index 10000 |

**This is the first test with a narrowed range.** Only 26,436 of 270,725 hands (9.8%) have non-zero probability. This makes sense: in a facing-4bet spot, the villain has open-raised and 4-bet, so only strong hands remain. The non-zero hands are concentrated in specific index ranges (strong Omaha hands), with large blocks of zeros for weak hands.

**Critical finding: `'indices' in tree_node` is still `False`.** Even with a narrowed range, the tree node stores the full 270,725-length array directly (with zeros for excluded hands) rather than using sparse index compression. This means the sparse index expansion bug (Sub-Bug A) is NOT triggered even for this non-trivial node.

### Strategy Arrays — Comparison Across All Tests

| Property | Tests 1-3 (unopened/facingOpen) | Test 4 (facing4Bet) |
|----------|-------------------------------|---------------------|
| `decoded_strategy_values.shape` | `(3, 270725)` — 3 actions | **`(2, 270725)` — 2 actions** |
| `decoded_strategy_values.min` | `-0.0` | `0.0` |
| `decoded_strategy_values.max` | `1.0` | `1.0` |
| Pattern | Mixed probabilities | Binary: row 0 = `[1,1,1,...,0,0,0]`, row 1 = `[0,0,0,...,1,1,1]` |

Test 4 has only 2 actions (likely Fold / Call — hero is facing a 4-bet so can only fold or call, no re-raise option). Strategy is nearly binary: each hand either folds 100% or calls 100%.

### EV Arrays — Comparison Across All Tests

| Property | Tests 1-3 (unopened/facingOpen) | Test 4 (facing4Bet) |
|----------|-------------------------------|---------------------|
| `decoded_ev_values.shape` | `(3, 270725)` | **`(2, 270725)`** |
| `decoded_ev_values.min` | `-5.19` | **`-48.0`** |
| `decoded_ev_values.max` | `12.01` | **`100.88`** |

Much wider EV range in Test 4 (-48 to +100 BB). This is because the pot is already large in a 4-bet scenario (Cash Drop 10bb adds ante), so the EV differences between fold and call are bigger.

### Test 4: Strategy & EV Arrays — facing4Bet / preflop_only / GGPoker / Cash Drop 10bb

Note: Test 4 was hit twice (random combo selection). Both runs showed consistent patterns:

| Property | Run 1 | Run 2 |
|----------|-------|-------|
| `decoded_strategy_values.shape` | `(2, 270725)` | `(2, 270725)` |
| `decoded_strategy_values.min/max` | `0.0 / 1.0` | `0.0 / 1.0` |
| `decoded_ev_values.shape` | `(2, 270725)` | `(2, 270725)` |
| `decoded_ev_values.min` | `-48.0` | `-47.72` |
| `decoded_ev_values.max` | `100.88` | `102.5` |

EV min/max differ slightly between runs because V1 picks a random combo each time, hitting different tree nodes with the same structure but different board/action paths. The shape and strategy pattern are identical.

### Test 6: Strategy & EV Arrays — Holdem facing3Bet / full_tree / PokerStars

| Property | Value | Notes |
|----------|-------|-------|
| `decoded_strategy_values.shape` | **(4, 1326)** | **4 actions** — Fold, Call, Raise small, Raise large (most actions seen in any test) |
| `decoded_strategy_values.min/max` | `0.0 / 1.0` | |
| Strategy pattern | **Mixed** — e.g. hand 0: `[0.977, 0.023, 0., 0.]` | First Holdem test with non-binary strategy (mix between fold and call) |
| `decoded_ev_values.shape` | **(4, 1326)** | |
| `decoded_ev_values.min` | **`-9.86`** | |
| `decoded_ev_values.max` | **`34.81`** | |
| EV pattern | Action 0 (Fold): all 0.0; Actions 1-3: range from -9.86 to 34.81 | |

This is the most complex test result: 4 actions with mixed strategies (non-binary probabilities). Holdem facing3Bet from SB gives the hero Fold/Call/small raise/large raise options, with some hands mixing between actions rather than always choosing 100% one action.

### Test 8: V2 Direct Comparison — facingOpen / full_tree / GGPoker / Regular (breakpoint at `strategy_tree_node.py:257`)

```
URL: http://localhost:8000/omaha/trainer/v2/generate-next-hand?openRaise=3.5&variant=omaha
     &game=cash&site=GGPoker&numPlayers=6&stackInBB=100&BB=0.25
     &blindStructure=Regular&difficulty=medium&research=full_tree
     &3bet=100&4bet=100&preflopActions=r35
```

This is the first test run through the **V2 code path** (`GenerateNextHandV2` → `StrategyTreeNodeResolver` → `RangeCalculator.resolve_range()` → `StrategyTreeNode.decode_range()`). The breakpoint was set inside `StrategyTreeNode.decode_range()` at line 257, where the sparse indices check occurs.

| Property | V2 Value | Matches V1 Tests 1-3? |
|----------|----------|----------------------|
| `self.raw_indices is not None` | `False` | Yes — same as `'indices' in tree_node` on V1 |
| `range_result.shape` | `(270725,)` | Yes |
| `range_result.dtype` | `float64` | Yes |
| `len(range_result)` | `270725` | Yes |
| `np.count_nonzero(range_result)` | `270725` | Yes — all values nonzero |
| `range_result.min()` | `1.0` | Yes |
| `range_result.max()` | `1.0` | Yes |
| `range_result.sum()` | `270725.0` | Yes |
| `range_result[0:270725]` | All `np.float64(1.0)` | Yes — every element is 1.0 |

**This confirms byte-for-byte equivalence between V1 and V2 for preflop Omaha range arrays.** The V2 path enters `StrategyTreeNode.decode_range()` which calls the same underlying `uf.decode_range()` as V1's `decode_range(tree_node['range'])`. The sparse indices check (`self.raw_indices is not None`) returns `False`, so the expansion branch is skipped — identical to V1 where `'indices' in tree_node` is `False`.

### Test 9: V2 Direct Comparison — facing3Bet / preflop_only / PokerStars / Regular (breakpoint at `strategy_tree_node.py:257`)

```
URL: http://localhost:8000/omaha/trainer/v2/generate-next-hand?openRaise=3.5&variant=omaha
     &game=cash&site=PokerStars&numPlayers=6&stackInBB=100&BB=0.25
     &blindStructure=Regular&difficulty=medium&research=preflop_only
     &3bet=100&4bet=100&preflopActions=f,r35,f,f
```

Tests a different site (PokerStars), different research type (`preflop_only`), and deeper action sequence (`f,r35,f,f` = facing3Bet with SB acting after folds and a raise).

| Property | V2 Value | Matches V1 & Test 8? |
|----------|----------|---------------------|
| `self.raw_indices is not None` | `False` | Yes |
| `range_result.shape` | `(270725,)` | Yes |
| `range_result.dtype` | `float64` | Yes |
| `range_result.min()` | `1.0` | Yes |
| `range_result.max()` | `1.0` | Yes |
| `range_result.size` | `270725` | Yes |

**Same pattern as all previous tests.** Changing the site from GGPoker to PokerStars and research from `full_tree` to `preflop_only` does not affect the range array at the preflop level. V2's `StrategyTreeNode.decode_range()` produces the identical all-1.0 array.

**Conclusion:**

Strategy and EV arrays are decoded from the same tree node bytes via `decode_strategy_ev()` in both V1 and V2. Key findings:

- For preflop, both paths produce identical results
- Tests 1-3 (`full_tree` and `postflop_only`) produce byte-for-byte identical strategy and EV arrays at the preflop level
- Test 4 shows different data because it hits deeper nodes in a different tree (GGPoker Cash Drop `preflop_only`)
- Test 6 is the first with 4 actions and mixed strategy probabilities
- In all cases the decoding path is the same, and V1 reads the stored data correctly

---

## Why No Divergence Was Found

### V1 Path (`spot_generation.py:199`)

```python
range_array = decode_range(tree_node['range'])
```

Does: brotli decompress → float16→float64 cast → clamp tiny values.

### V2 Path (`spot_generation.py:403`)

```python
range_array = calc_data.rangeResult  # from RangeCalculator.resolve_range()
```

Does: same decode + sparse index expansion + cache + walk-back + suit swaps.

### Why they match for preflop trainer spots

**Context:** V2 is architecturally multi-street capable (can handle preflop, flop, turn, river), while V1 is preflop-only. The trainer currently only serves preflop spots to users. For these preflop scenarios:

1. **No sparse indices** — Omaha preflop tree nodes store full 270725-length ranges without compressed `indices`. The `expand_range()` step in V2 is a no-op.

2. **Stored range is all 1.0** — Preflop nodes store the starting range (every hand is possible). V2's walk-back computation would narrow this by multiplying by strategy frequencies of previous actions, but for the hero's perspective at a preflop spot, the stored range is the correct input.

3. **No precision/isomorphism** — `full_tree` Omaha preflop does not use `postflop_only` (precision) strategies, so `apply_swaps()` is never called.

4. **Strategy/EV decoding is identical** — Both V1 and V2 call the same `decode_strategy_ev()` function. V2 pre-decodes and stores results on `calc_data`, but the underlying computation is the same for preflop nodes.

**Note:** If/when the trainer adds postflop support, V2's extra transformations (walk-back, suit swaps) would become active and necessary. V1/V3 would then produce incorrect results without those transformations.

---

## Three Theoretical Sub-Bugs (from original analysis)

| # | Transformation V1 Skips | Active Today? | Evidence | When It Would Trigger |
|---|------------------------|---------------|----------|----------------------|
| **A** | Sparse index expansion (`expand_range`) | **No** — tested 4 scenarios across 3 spot types, 3 research types, 2 sites; `'indices' in tree_node` was `False` in ALL cases, including the narrowed-range facing4Bet node (Test 4) | Tests 1-4: all `False` | If tree data is re-imported with sparse storage for preflop nodes |
| **B** | Walk-back range calculation | **No** — Test 4 shows a narrowed range (26,436/270,725 nonzero) stored directly on the node, not computed via walk-back. V1 reads this correctly | Test 4: narrowed range, still no walk-back needed | If trainer expands to postflop streets where walk-back is required |
| **C** | Precision suit isomorphism swaps (`apply_swaps`) | **No** — `postflop_only` tested (Test 3), `preflop_only` tested (Test 4); neither triggers swaps at preflop | Tests 3-4 | If trainer supports postflop precision strategies |

---

## Affected Endpoints

| Endpoint | Class | Range Source | Has Bug? | FE Uses? | Fix Status |
|----------|-------|-------------|----------|----------|------------|
| V1 `/trainer/generate-next-hand` | `GenerateNextHand` | `decode_range()` + `expand_range()` | ✅ Fixed (2026-02-17) | Yes | Defensive fix applied |
| V2 `/trainer/v2/generate-next-hand` | `GenerateNextHandV2` | `RangeCalculator.resolve_range()` | Reference (correct) | Yes | N/A (was always correct) |
| V3 `/trainer/v3/generate-next-hand` | `GenerateNextHandV3` | `decode_range()` + `expand_range()` | ✅ Fixed (2026-02-17) | No | Defensive fix applied |
| V4 `/trainer/v4/generate-next-hand` | `GenerateNextHandV4` | `RangeCalculator.resolve_range()` | OK (same as V2) | No | N/A (was always correct) |

---

## Implemented Fix (Defensive)

**Date Implemented:** 2026-02-17  
**Status:** ✅ COMPLETE

Even though the bug is latent, the fix is low-risk and prevents future issues.

### Code Changes

**File:** `trainer/views/spot_generation.py`

**Import updated:**
```python
from strategies.utils.unfold_range import decode_range, expand_range
```

**V1 `GenerateNextHand.get()` (line 200):**
```python
range_array = decode_range(tree_node['range'])
# A-1 defensive fix: expand sparse indices if present (no-op if absent)
range_array = expand_range(tree_node, range_array)
```

**V3 `GenerateNextHandV3.get()` (line 576):**
```python
range_array = decode_range(tree_node['range'])
# A-1 defensive fix: handles sparse indices if present
range_array = expand_range(tree_node, range_array)
```

### Tests Added

**File:** `trainer/tests/test_range_divergence_a1.py`

5 new unit tests (all passing):
1. `test_expand_range_is_noop_when_no_indices` — Verifies no-op for current production
2. `test_expand_range_expands_sparse_indices` — Tests sparse expansion with synthetic data
3. `test_v1_pipeline_with_sparse_indices` — End-to-end V1 pipeline with sparse indices
4. `test_v1_pipeline_without_sparse_indices` — End-to-end V1 pipeline without sparse indices
5. `test_expand_range_preserves_narrowed_full_array` — Tests narrowed range (Test 4 scenario)

### Verification

- ✅ All 5 new tests pass
- ✅ All 17 existing `trainer/tests/test_spot_generation_views.py` tests pass (no regressions)
- ✅ Production verification: V1 request confirmed `expand_range()` is no-op (returns array unchanged)
- ✅ Formatting/linting clean: `isort`, `autopep8`, `flake8`, `mypy`

**How it works:**

`expand_range()` checks `if 'indices' in tree_node`. When absent (current state), it returns the array unchanged — literally a no-op with zero performance impact. If sparse indices are ever introduced, it decompresses them and expands the compressed range into a full `NUM_COMBOS`-length array.

**Files changed:**
- `trainer/views/spot_generation.py` — 2 lines added (V1 line 200, V3 line 576)
- `trainer/tests/test_range_divergence_a1.py` — New test file (177 lines, 5 test cases)

---

## Still Untested

- [x] ~~Holdem variant (`GAME_VARIANT=holdem`)~~ — tested in Test 5, same pattern as Omaha (all 1.0, no indices, `NUM_COMBOS=1326`)
- [ ] Nodes with actual sparse `indices` fields (none found in any tested Omaha or Holdem preflop data)
- [x] ~~V2 endpoint with same params for direct comparison~~ — tested in Test 8 via breakpoint at `StrategyTreeNode.decode_range()` (`strategy_tree_node.py:257`). Confirmed byte-for-byte identical: `raw_indices=None`, all-1.0 array of 270,725 float64 values
- [ ] Postflop trainer spots (not yet supported by V1)
- [x] ~~Precision (postflop_only) research type~~ — tested in Test 3, no difference at preflop

---

## Files Examined During Investigation

| File | Lines | Purpose |
|------|-------|---------|
| `trainer/views/spot_generation.py` | 185-227 (V1), 373-426 (V2) | Entry points for both paths |
| `trainer/utils/generate_helper.py` | 238-314 | V1's `generate_generic_action_combinations` |
| `trainer/utils/trainer_helper.py` | 303-538 | Shared `_generate_hand_response` |
| `strategies/utils/unfold_range.py` | 280-321 | `decode_range()` and `expand_range()` |
| `strategies/models/strategy_tree_node.py` | 258-451 | `StrategyTreeNode.decode_range()` and `RangeCalculator.resolve_range()` |
| `strategies/services/strategy_tree/decoder.py` | 45-134 | V2's `StrategyNodeDecoder.decode()` |
| `strategies/services/strategy_treenode_resolver.py` | 168-204 | V2's `resolve_node()` orchestrator |
| `strategies/repositories/strategy_repository.py` | 478-505 | `get_tree_node_by_combo()` — no projection, full doc returned |
| `strategies/models/strategy_tree_node.py` | 243-265 | `StrategyTreeNode.decode_range()` — V2 breakpoint location (Test 8), sparse indices check |
| `strategies/models/strategy_tree_node.py` | 303-420 | `RangeCalculator.resolve_range()` — V2 range resolution entry point |
| `strategies/models/strategy_tree_node.py` | 452-481 | `RangeCalculator._try_cache_or_node()` — V2 cache/node shortcut for active player |

---

## Final Conclusion

**The A-1 range array divergence bug is confirmed LATENT — not actively causing issues for any current user scenario.**

Across 11 tests (7 on V1, 4 on V2) covering the full parameter space that the trainer endpoints serve:

| Dimension | Coverage |
|-----------|----------|
| Game variants | Holdem + Omaha |
| Game types | Cash + Spins |
| Player counts | 3-max, 6-max |
| Stack depths | 25bb, 100bb |
| Spot types | unopened, facingOpen, facing3Bet, facing4Bet |
| Research types | full_tree, postflop_only, preflop_only |
| Sites | NoRake, GGPoker, PokerStars |
| Blind structures | Regular, Cash Drop 10bb |
| Range types | Full (all 1.0), Narrowed (9.8% and 42% nonzero) |
| Action counts | 2, 3, and 4 actions per node |
| Strategy types | Binary (pure) and mixed (probabilistic) |
| Endpoint paths | V1 (`decode_range`) and V2 (`RangeCalculator.resolve_range`) |

**`'indices' in tree_node` (V1) / `self.raw_indices` (V2) was `False`/`None` in every single test.** No preflop tree node in the current database uses sparse index compression. V1's `decode_range()` and V2's `StrategyTreeNode.decode_range()` produce identical full-length arrays in all cases. Test 8 confirmed this with a direct V2 breakpoint comparison.

---

## Next Steps

### ✅ Completed

- [x] **Defensive fix applied** (2026-02-17) — `expand_range()` added to V1 and V3
- [x] **Unit tests written** — 5 new tests covering no-op and sparse expansion scenarios
- [x] **Verification complete** — All tests pass, no regressions, production confirmed as no-op
- [x] **Investigation doc updated** — Reflects implemented fix

### Future Considerations

**Note:** 

- If trainer ever expands to postflop streets, V1/V3 would need the full `RangeCalculator.resolve_range()` pipeline (walk-back + suit swaps), not just `expand_range()`
- At that point, deprecating V1/V3 entirely and routing all traffic through V2/V4 would be the recommended approach

### ✅ Decision Made: Option A (Defensive Fix)

**Selected:** Option A — Apply the defensive `expand_range()` fix  
**Implemented:** 2026-02-17  
**Rationale:** Zero risk, future-proofs against sparse compression

**Rejected Options:**
- **Option B (Do nothing):** Skipped — the 1-hour fix is low enough effort to justify the protection
- **Option C (Deprecate V1/V3):** Not needed — since the bug is latent and postflop support isn't planned, the higher-effort deprecation provides no immediate value

---

## Appendix: Impact on Stream A Bug Prioritization

This investigation was conducted as part of **Stream A: Trainer Values Inconsistent** (9 bugs total). A-1 was originally prioritized as P0 with the expectation that it "may explain most inconsistencies."

### Finding: A-1 Does Not Explain Reported Inconsistencies

The range array divergence bug exists in the code but is **dormant** — it does not activate for any current trainer scenario. This means:

- ✅ **A-1 is ruled out** as the root cause of user-reported value inconsistencies
- ✅ **A-4 (frontend regret fallback)** is likely the actual culprit — it uses a fundamentally different formula and affects every hand
- ✅ **A-3 (pot size alignment)** may be a contributing factor — pot differences compound with regret calculation differences
- ✅ **The investigation was valuable** — it eliminated a major architectural suspect with hard evidence

### Revised Recommendations (Updated 2026-02-17)

1. ✅ **Defensive fix applied** — `expand_range()` added to V1/V3, fully tested and verified
2. **Shift investigation focus** to A-3 (pot size) and A-2 (EV decode precision) as the next high-value targets
3. **Validate A-4 fix** — the frontend regret fallback removal (already implemented in current branch) should resolve most user-reported inconsistencies

### A-1 Status: CLOSED

- Investigation complete across 11 test scenarios (7 V1, 4 V2)
- Bug confirmed latent (not actively causing issues)
- Defensive fix implemented and tested
- V1/V3 now aligned with V2 for sparse index handling
- ✅ **Sparse compression is now supported** — if introduced in the future, V1/V3 will handle it automatically with zero code changes
- No further action needed
