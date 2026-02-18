# Stream A: Trainer Values Inconsistent — Investigation Summary

**Stream:** Stream A — Trainer Values Inconsistent  
**Status:** IN PROGRESS  
**Started:** 2026-02-16  
**Last Updated:** 2026-02-18  

---

## Overview

User-reported inconsistencies in trainer feedback values (regret%, EV loss, feedback labels) between different views and sessions. Initial analysis identified 9 potential bugs spanning backend calculations, frontend fallbacks, and cross-version divergence.

**Investigation approach:**
1. Prioritize bugs by likelihood and impact
2. Conduct systematic testing with evidence collection
3. Document findings in bug-specific detail docs
4. Update this summary as investigation progresses

---

## System Architecture

### How the Trainer Works

The trainer has two API code paths that feed the same V3 frontend layout. The frontend decides which API to call based on whether `fromStrategiesConfig` is present (strategy page entry) or not (standalone trainer).

```
┌──────────────────┐         ┌──────────────────────────────┐
│  Standalone       │         │  Strategy Page Entry          │
│  Trainer          │         │  (via Train button)           │
│                   │         │                               │
│  No config        │         │  fromStrategiesConfig present │
└────────┬──────────┘         └──────────────┬────────────────┘
         │                                   │
         ▼                                   ▼
┌──────────────────┐         ┌──────────────────────────────┐
│  V1 API           │         │  V2 API                       │
│  /trainer/        │         │  /trainer/v2/                  │
│  generate-next-   │         │  generate-next-hand            │
│  hand             │         │                               │
│                   │         │  Uses StrategyTreeNodeResolver │
│  decode_range()   │         │  decode_or_calculate_range()   │
│  direct from node │         │  + expand_range()              │
│                   │         │  + apply_swaps()               │
└────────┬──────────┘         └──────────────┬────────────────┘
         │                                   │
         └──────────────┬────────────────────┘
                        ▼
            ┌───────────────────────┐
            │  _generate_hand_      │
            │  response()           │
            │                       │
            │  EV / regret calc     │
            │  pot_size adjustment  │
            └───────────┬───────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Frontend (V3 layout) │
            │                       │
            │  FeedbackPanel.tsx     │
            │  TrainerCenterPanel    │
            │  TrainerHistoryList   │
            │  FeedbackActions      │
            └───────────────────────┘
```

### Divergence Points

**Backend:** V1 raw-decodes range data; V2 applies resolver transformations (sparse index expansion, action tree walking, isomorphism swaps). Same tree node → potentially different range arrays.

**Frontend:** Backend computes regret using strategy-weighted EV. Frontend (pre-A-4 fix) recalculated regret using only the selected action's EV — fundamentally different formulas.

---

## Bug Status Dashboard

| ID | Bug | Status | Original Priority | Investigated | Conclusion |
|----|-----|--------|------------------|--------------|------------|
| **A-4** | Frontend regret fallback formula | ✅ FIXED | P0 (HIGH) | ✅ 2026-02-10–16 | Fix merged (PR #6144), awaiting deployment |
| **A-1** | Range array divergence (V1 vs V2) | ✅ CLOSED | P0 (HIGH) | ✅ 2026-02-16/17 | Latent bug, defensive fix applied |
| **A-5** | Board randomization timing (stale ref) | 🔍 PENDING | P1 (HIGH) | ⏸️ Deferred | Postflop-only, trainer is preflop |
| **A-3** | Pot size divergence (V1 vs V2 configs) | ✅ CLOSED | P1 (MED) | ✅ 2026-02-18 | Not a bug — no divergence found |
| **A-2** | EV decode precision (pre-decoded vs fresh) | 🔍 PENDING | P2 (MED) | ⏸️ Deferred | Wait for A-4 validation |
| **A-6** | apply_swaps failures silently ignored | 🔍 PENDING | P2 (HIGH postflop) | ⏸️ Deferred | Postflop-only, coordinate with Akhil |
| **A-7** | 9s isomorphism bug (suit mapping) | 🔍 PENDING | P2 (HIGH postflop) | ⏸️ Deferred | Postflop-only, coordinate with Akhil |
| **A-8** | Cache key missing user ID | 🔍 PENDING | P2 (MED) | ⏸️ Deferred | Shared cache edge case |
| **A-9** | Spins config type safety (`as any`) | 🔍 PENDING | P2 (LOW) | ⏸️ Deferred | Type system polish |

---

## Bug Inventory (Original Analysis)

**Source:** [Stream A Proposal](https://github.com/SidVedi/Hack2024/blob/main/finalReq.md)

| Bug | Risk | Type | Description | Priority | Effort |
|-----|------|------|-------------|----------|--------|
| A-4 | HIGH | FE | Frontend regretPercent uses different formula than backend | P0 | 1 day |
| A-1 | HIGH | BE | Range array decoded via different paths in V1 vs V2 | P0 | 2-3 days |
| A-5 | HIGH | FE | Board randomization at wrong time with stale ref | P1 | 2 days |
| A-3 | MED | BE+FE | Pot size differs between V1/V2 configs | P1 | 1-2 days |
| A-2 | MED | BE | Pre-decoded vs fresh-decoded strategy/EV arrays | P2 | 1 day |
| A-6 | HIGH (postflop) | BE | apply_swaps failures silently ignored | P2 | 0.5 day |
| A-7 | HIGH (postflop) | BE | 9s isomorphism bug in suit mapping | P2 | 1 day |
| A-8 | MED | BE | Cache key missing user role/ID | P2 | 0.5 day |
| A-9 | LOW | FE | Spins config type safety (`as any` chain) | P2 | 0.5 day |

**Note:** Priorities and risk levels are from original analysis. Actual investigation showed A-1 is latent (not HIGH risk) and A-3 is not a bug (not MED risk).

---

## Completed Investigations

### ✅ A-1: Range Array Divergence (V1 vs V2)

**Detail Doc:** [`A1_RANGE_DIVERGENCE_INVESTIGATION.md`](./A1_RANGE_DIVERGENCE_INVESTIGATION.md)

**Hypothesis:**  
V1 and V2 use different code paths to produce `range_array`, potentially causing divergent hand selection and strategy lookups.

**Investigation:**
- **Scope:** 11 test scenarios (7 V1, 4 V2) across Holdem + Omaha, multiple spots, sites, and research types
- **Method:** Debugger breakpoints at range decoding points, byte-for-byte comparison
- **Duration:** ~0.5 day investigation + ~1 hour defensive fix

**Findings:**
- ✅ Bug is **latent** (not actively triggering in production)
- ✅ No preflop tree nodes use sparse index compression (`indices` field)
- ✅ V1 and V2 produce **byte-for-byte identical** range arrays for all tested scenarios
- ✅ Bug would only activate if sparse compression is introduced in future

**Outcome:**
- **Status:** CLOSED (latent, defensively fixed)
- **Fix:** Added `expand_range()` call to V1/V3 endpoints (no-op when `indices` absent)
- **Tests:** 5 new unit tests added, all passing
- **Impact:** Zero — bug not causing reported inconsistencies

**Key Insight:**  
A-1 is ruled out as root cause. Investigation shifted focus to A-4 (frontend fallback) as likely culprit.

---

### ✅ A-3: Pot Size and Rake Divergence (V1 vs V2)

**Detail Doc:** [`A3_POT_SIZE_INVESTIGATION.md`](./A3_POT_SIZE_INVESTIGATION.md)

**Hypothesis:**  
V1 and V2 might use different strategy configs (standalone vs strategy-page), leading to different `rakeAfterEachStreet` flags and pot sizes, which would cause regret% to diverge.

**Investigation:**
- **Scope:** 4 test scenarios (V1/V2 × GGPoker/NoRake)
- **Method:** Debug logging of pot calculation pipeline, direct comparison
- **Duration:** ~1.5 hours (setup, testing, analysis)

**Findings:**
- ✅ V1 and V2 calculate **identical pot sizes** (5.0bb in all 4 tests)
- ✅ Both use the **same strategy object** via `get_matching_strategy()`
- ✅ Both use the **same pot calculation logic** (`adjust_pot_size_with_rake()`)
- ✅ No evidence of config source divergence
- ⚠️ All tests showed `rake_flag=False` (expected for preflop — rake not collected yet)

**Outcome:**
- **Status:** CLOSED (not a bug)
- **Fix:** None needed
- **Impact:** Zero — no divergence exists

**Key Insight:**  
Frontend `fromStrategiesConfig` is only used to pre-populate V2 request params, not as a separate config source. Both endpoints derive params from the same request and hit the same Redis cache.

---

---

### ✅ A-4: Frontend Regret Fallback Formula

**Detail Doc:** [`A4_REGRET_FALLBACK_INVESTIGATION.md`](../../GameTrainer-Frontend/docs/A4_REGRET_FALLBACK_INVESTIGATION.md)

**Hypothesis:**  
Frontend recalculated regret using a different formula than backend when values were missing, causing inconsistent feedback labels across views.

**Investigation:**
- **Scope:** 7 affected components (trainer, replayer, mobile)
- **Method:** Code analysis, PR #6144 implementation
- **Duration:** ~1 week (Feb 10-16, 2026)

**Findings:**
- ✅ **Formula divergence confirmed** — Frontend used simple EV loss, backend used weighted-EV regret
- ✅ Different formulas → different regret% → different feedback labels (optimal/strong/weak/blunder)
- ✅ Affected **all user sessions** (not just edge cases)
- ✅ Impact was widespread: trainer center panel, feedback panel, history list, replayer, mobile

**Formula Comparison:**
- **Backend (Correct):** `((best_EV - weighted_EV) / pot) * 100` — Accounts for mixed strategies
- **Frontend Fallback (Wrong):** `((best_EV - selected_EV) / pot) * 100` — Assumes pure strategies

**Outcome:**
- **Status:** ✅ FIXED (merged Feb 16, 2026, awaiting production deployment)
- **PR:** #6144 (18 files, +1375/-292 lines)
- **Fix:** 
  - Added `getRegretPercent()` utility for backend value validation
  - Removed all client-side regret calculations
  - Show em dash (—) when backend value is invalid instead of recalculating
  - Debug users see dual BE/FE display for comparison
- **Tests:** 3 new test files, +391 lines of tests
- **Impact:** HIGH — Expected to resolve most user-reported inconsistencies

**Key Insight:**  
A-4 is the **primary root cause** of user-reported inconsistencies. Different formulas affected every hand, while A-1 and A-3 were either latent or non-existent.

---

## Active/Pending Investigations

---

### ⏸️ A-2: EV Decode Precision (Pre-decoded vs Fresh)

**Status:** PENDING (deferred until A-4 validated)  
**Original Priority:** P2  
**Type:** Backend  
**Estimated Effort:** 1 day

**Hypothesis:**  
V1 always decompresses EV data fresh (with 2-decimal rounding). V2 may use pre-decoded arrays from the resolver that were rounded at a different stage or not at all. This could cause subtle EV differences between endpoints.

**Plan:**
- Wait for A-4 fix deployment and validation
- If inconsistencies persist, investigate EV precision
- Compare float arrays from V1 vs V2 at decode time
- Standardize rounding if precision differs

---

### ⏸️ A-5: Board Randomization Timing (Stale Ref)

**Status:** PENDING (likely postflop-only issue)  
**Original Priority:** P1 (HIGH risk)  
**Type:** Frontend  
**Estimated Effort:** 2 days

**Hypothesis:**  
Board cards are randomized at buffer consumption time, not at generation time. EV values were computed for the original board, but the user sees a randomized board. The strategy tree doesn't match what's displayed.

**Why Deferred:**
- Trainer is currently **preflop-only** (no board cards in preflop spots)
- Issue only manifests in postflop scenarios
- A-4 (primary culprit) already addresses most inconsistencies

**Plan (if trainer adds postflop support):**
1. Move randomization to backend during hand generation
2. Or: Randomize at buffer fill time; regenerate buffer when settings change
3. Clean up stale ref logic (`isFirstHandRef`)

---

### ⏸️ A-6: apply_swaps Silent Failures

**Status:** PENDING (coordinate with Akhil before investigation)  
**Original Priority:** P2 (HIGH for postflop)  
**Type:** Backend  
**Estimated Effort:** 0.5 day

**Hypothesis:**  
Two locations catch `apply_swaps()` failures at DEBUG level and continue with unswapped (wrong) data. Users see wrong combo ordering and EV with zero indication.

**Why Deferred:**
- Affects **postflop precision strategies** only
- Trainer is currently preflop-only
- Akhil is actively working on RTS code that touches swap logic

**Plan (when needed):**
1. Coordinate with Akhil to avoid conflicts
2. Escalate swap failures from DEBUG to WARNING
3. Add monitoring metrics
4. Consider returning error indicator to frontend

---

### ⏸️ A-7: 9s Isomorphism Bug (Suit Mapping)

**Status:** PENDING (coordinate with Akhil before investigation)  
**Original Priority:** P2 (HIGH for postflop)  
**Type:** Backend  
**Estimated Effort:** 1 day

**Hypothesis:**  
Suit mapping uses the current round's ISO board (e.g., TURN) instead of the FLOP ISO board. Precision strategies index by flop ISO, so the suit map is wrong on turn/river when 9s appears.

**Why Deferred:**
- Affects **postflop precision strategies** only (9s on turn/river)
- Trainer is currently preflop-only
- Akhil is actively working on RTS/isomorphism code

**Plan (when needed):**
1. Coordinate with Akhil
2. Fix to use flop round number for suit mapping
3. Extend existing tests to verify fix

---

### ⏸️ A-8: Cache Key Missing User ID

**Status:** PENDING  
**Original Priority:** P2 (MEDIUM)  
**Type:** Backend  
**Estimated Effort:** 0.5 day

**Hypothesis:**  
Cache key includes role hash and path but not user ID. Users with the same role share cache, potentially serving wrong subscription tier data across different users.

**Plan:**
1. Audit cache key generation logic
2. Add user ID to cache key (matching `user_role_cache()` decorator pattern)
3. Deploy during low-traffic window (cache invalidation spike)
4. Monitor cache hit rate after deployment

---

### ⏸️ A-9: Spins Config Type Safety

**Status:** PENDING  
**Original Priority:** P2 (LOW)  
**Type:** Frontend  
**Estimated Effort:** 0.5 day

**Hypothesis:**  
Unsafe double `as any` cast when accessing spins config. If config structure changes, trainer silently uses wrong config, potentially causing crashes or wrong behavior.

**Plan:**
1. Audit existing spins config usage across codebase
2. Define proper TypeScript interface for spins config structure
3. Replace `as any` casts with typed access
4. TypeScript compilation verifies fix

---

## Root Cause Analysis

### Primary Culprit: A-4 (Frontend Regret Fallback) ✅

**Evidence:**
1. Frontend uses fundamentally different formula than backend
2. Formula differences affect **every hand** (not just edge cases)
3. A-1 and A-3 ruled out (no backend divergence)
4. A-4 fix already implemented and ready for deployment

**Impact Chain:**
```
Frontend fallback formula differs from backend
→ Different regret% calculated
→ Different feedback labels (optimal/strong/weak/blunder)
→ User sees inconsistent feedback between views
```

### Contributing Factors (Possible)

- **A-2 (EV precision):** May cause small rounding differences
- **A-3 (Pot size):** Ruled out — no divergence
- **A-5–A-9:** Lower priority, investigate only if issues persist after A-4

---

## Investigation Methodology

### Phase 1: Evidence Collection
1. Add targeted debug logging
2. Run parallel tests (V1 vs V2, different sites)
3. Capture logs and API responses
4. Compare byte-for-byte

### Phase 2: Analysis
1. Identify divergence points (if any)
2. Trace code paths to root cause
3. Determine impact scope

### Phase 3: Fix & Validation
1. Implement fix (backend or frontend)
2. Add regression tests
3. Document findings
4. Deploy and monitor

---

## Testing Coverage

### A-1 Testing
- ✅ 11 scenarios across Holdem + Omaha
- ✅ Multiple spots (unopened, facingOpen, facing3Bet, facing4Bet)
- ✅ Multiple sites (NoRake, GGPoker, PokerStars)
- ✅ Multiple research types (full_tree, postflop_only, preflop_only)
- ✅ Range types (full, narrowed)
- ✅ V1 vs V2 direct comparison

### A-3 Testing
- ✅ 4 scenarios (V1/V2 × GGPoker/NoRake)
- ✅ Identical game parameters
- ✅ Pot calculation pipeline instrumented
- ✅ Rake flag and adjustment validated

### A-4 Testing
- ⏳ In progress
- Plan: Compare frontend vs backend regret values across multiple hands

---

## Lessons Learned

### Investigation Best Practices
1. **Start with evidence, not assumptions** — A-1 and A-3 hypotheses were logical but disproven by testing
2. **Instrument the code** — Debug logging revealed exact divergence points (or lack thereof)
3. **Test comprehensively** — Multiple scenarios across variants, sites, spots revealed patterns
4. **Document findings** — Detailed docs prevent re-investigation and inform future work

### Code Quality Insights
1. **V1 and V2 are more aligned than expected** — Both use shared helpers and same strategy lookup
2. **Frontend fallbacks are risky** — A-4 shows danger of client-side recalculation with different formulas
3. **Defensive fixes are valuable** — A-1's `expand_range()` protects against future data format changes

---

## Execution Strategy (Aligned with Original Proposal)

### Phase 1: Stop the Bleeding (P0) ✅

**Approach:** Fix the bugs causing **every hand** to show wrong values

**Completed:**
- ✅ **A-4** (P0): Frontend regret fallback — **FIXED**, merged PR #6144, awaiting deployment
- ✅ **A-1** (P0): Range divergence — **Investigated and defensively fixed**, latent bug protected

**Impact:** These two fixes address the primary root causes of user-reported inconsistencies.

---

### Phase 2: Visible UX Fixes (P1) ⏸️

**Approach:** Fix bugs that affect specific scenarios or workflows

**Status:**
- ⏸️ **A-5** (P1): Board randomization — Deferred (postflop-only, trainer is preflop)
- ✅ **A-3** (P1): Pot size alignment — **Investigated**, ruled out as not a bug

**Decision:** Wait for A-4 deployment and user validation before proceeding with P1.

---

### Phase 3: Polish and Tech Debt (P2) ⏸️

**Approach:** Address edge cases, logging, and type safety

**Status:**
- ⏸️ **A-2** (P2): EV precision — Deferred until A-4 validated
- ⏸️ **A-6** (P2): apply_swaps logging — Deferred (postflop, coordinate with Akhil)
- ⏸️ **A-7** (P2): 9s isomorphism — Deferred (postflop, coordinate with Akhil)
- ⏸️ **A-8** (P2): Cache key isolation — Deferred (low impact edge case)
- ⏸️ **A-9** (P2): Type safety — Deferred (polish, low risk)

**Decision:** Focus on P0 validation first; only proceed with P2 if issues persist.

---

## Next Steps

### Immediate (Week of 2026-02-18)
1. ✅ Complete A-3 investigation (DONE — not a bug)
2. ⏳ **Deploy A-4 fix to production** (highest priority)
3. ⏳ Monitor user feedback for 1-2 weeks post-deployment
4. ⏳ Validate that inconsistencies are resolved

### Short-term (Following 2-3 Weeks)
1. **If issues resolved after A-4**: 
   - ✅ Close Stream A investigation
   - ✅ Document final root cause (A-4 was primary culprit)
   - ✅ Archive P1/P2 bugs as "not needed"

2. **If issues persist after A-4**:
   - 🔍 Investigate A-2 (EV precision)
   - 🔍 Collect new evidence on remaining inconsistencies
   - 🔍 Re-prioritize P1/P2 bugs based on findings

### Long-term (Backlog)
- Consider refactoring frontend to always trust backend values (no fallbacks)
- Add integration tests comparing V1/V2 responses
- If trainer adds postflop support: Investigate A-5, A-6, A-7
- Monitor for new inconsistency reports

---

## Coordination Points

| With | When | Topic |
|------|------|-------|
| **Akhil** | Before investigating A-6/A-7 | RTS/isomorphism work may conflict with swap/9s fixes |
| **Priyanka** | During A-5 (if investigated) | `FeedbackActions.tsx` shared between Stream A and B |
| **Product Team** | After A-4 deployment | User feedback monitoring and issue resolution validation |

---

## Effort Summary

| Bug | Investigation Time | Fix Time | Testing Time | Total |
|-----|-------------------|----------|--------------|-------|
| A-1 | 0.5 day | 1 hour | 0.5 hour | ~5 hours |
| A-3 | 1.5 hours | 0 (no fix needed) | - | ~1.5 hours |
| A-4 | ~1 week | Included in investigation | Included | ~1 week |
| **Total** | **~1.5 weeks** | **~1 hour** | **Included** | **~1.5 weeks** |

---

## References

### Detail Documents
- [A1_RANGE_DIVERGENCE_INVESTIGATION.md](./A1_RANGE_DIVERGENCE_INVESTIGATION.md) — Range array divergence (V1 vs V2)
- [A3_POT_SIZE_INVESTIGATION.md](./A3_POT_SIZE_INVESTIGATION.md) — Pot size and rake divergence
- [A4_REGRET_FALLBACK_INVESTIGATION.md](../../GameTrainer-Frontend/docs/A4_REGRET_FALLBACK_INVESTIGATION.md) — Frontend regret fallback formula

### Related Code
- `trainer/utils/trainer_helper.py` — Shared pot calculation and regret logic
- `trainer/views/spot_generation.py` — V1/V2/V3 endpoint implementations
- `strategies/utils/unfold_range.py` — Range decoding and expansion (A-1 fix)
- `frontend/components/Trainer/` — Trainer UI and regret display (A-4 fix)

### Related Issues
- GitHub Issue: [User-reported trainer inconsistencies]
- PRs: 
  - Backend: A-1 defensive fix
  - Frontend: A-4 regret fallback removal

---

## Status Legend

- ✅ **CLOSED** — Investigation complete, bug fixed or ruled out
- ✅ **FIXED** — Fix implemented, awaiting deployment/validation
- 🔍 **PENDING** — Not yet investigated
- ⏸️ **DEFERRED** — Deprioritized, investigate only if needed
- 🚧 **IN PROGRESS** — Currently under investigation

---

**Last Updated:** 2026-02-18  
**Next Review:** After A-4 deployment and validation
