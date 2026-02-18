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

## Bug Status Dashboard

| ID | Bug | Status | Priority | Investigated | Conclusion |
|----|-----|--------|----------|--------------|------------|
| **A-1** | Range array divergence (V1 vs V2) | ✅ CLOSED | P0 → P3 | ✅ 2026-02-16/17 | Latent bug, defensive fix applied |
| **A-2** | EV decode precision loss | 🔍 PENDING | P1 | ⏸️ Deferred | Not yet investigated |
| **A-3** | Pot size divergence (V1 vs V2) | ✅ CLOSED | P1 → P3 | ✅ 2026-02-18 | Not a bug — no divergence found |
| **A-4** | Frontend regret fallback formula | ✅ FIXED | P0 | ✅ 2026-02-17 | Fix implemented in frontend branch |
| A-5 | EV diff sign inconsistency | 🔍 PENDING | P2 | ⏸️ Deferred | - |
| A-6 | Regret% negative values | 🔍 PENDING | P2 | ⏸️ Deferred | - |
| A-7 | Feedback label thresholds | 🔍 PENDING | P2 | ⏸️ Deferred | - |
| A-8 | Cache staleness | 🔍 PENDING | P2 | ⏸️ Deferred | - |
| A-9 | Rounding inconsistencies | 🔍 PENDING | P3 | ⏸️ Deferred | - |

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

## Active/Pending Investigations

### 🔍 A-4: Frontend Regret Fallback Formula

**Status:** ✅ FIXED (implementation complete, awaiting deployment)

**Hypothesis:**  
Frontend recalculates regret when backend `regretPercent` is missing/null using a different formula:
- **Frontend:** `(evDiff / potValue) * 100` (simple EV loss as % of pot)
- **Backend:** `((bestEV - weightedEV) / pot) * 100` (strategy-weighted regret)

These formulas produce different results, causing inconsistencies.

**Fix Implemented:**
- **Frontend branch:** `feature/sv-proto` (or current working branch)
- **Changes:**
  - Removed frontend regret calculation fallback
  - Always trust backend-provided `regretPercent`
  - Added `getRegretPercent()` utility to validate backend values (returns `null` for invalid)
  - Debug users see both BE and FE values for comparison

**Next Steps:**
- ✅ Implementation complete
- ⏳ Validation testing in progress
- ⏳ Deployment pending

**Expected Impact:** HIGH — Should resolve most user-reported inconsistencies

---

### ⏸️ A-2: EV Decode Precision Loss

**Status:** PENDING (deferred until A-4 validated)

**Hypothesis:**  
EV values decoded from float16 → float64 may lose precision, causing small rounding differences that compound in regret calculations.

**Plan:**
- Wait for A-4 fix deployment and validation
- If inconsistencies persist, investigate EV precision
- Compare EV values between V1/V2 at decode time

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

## Next Steps

### Immediate (Week of 2026-02-18)
1. ✅ Complete A-3 investigation (DONE)
2. ⏳ Validate A-4 fix in staging/dev environment
3. ⏳ Deploy A-4 fix to production
4. ⏳ Monitor user feedback for resolution confirmation

### Short-term (Following Week)
1. If issues persist after A-4: Investigate A-2 (EV precision)
2. If issues resolved: Close Stream A investigation
3. Document final root cause analysis

### Long-term
- Consider refactoring frontend to always trust backend values (no fallbacks)
- Add integration tests comparing V1/V2 responses
- Monitor for new inconsistency reports

---

## Effort Summary

| Bug | Investigation Time | Fix Time | Testing Time | Total |
|-----|-------------------|----------|--------------|-------|
| A-1 | 0.5 day | 1 hour | 0.5 hour | ~5 hours |
| A-3 | 1.5 hours | 0 (no fix needed) | - | ~1.5 hours |
| A-4 | (frontend team) | (frontend team) | TBD | TBD |
| **Total** | **~6.5 hours** | **~1 hour** | **TBD** | **~7.5 hours** |

---

## References

### Detail Documents
- [A1_RANGE_DIVERGENCE_INVESTIGATION.md](./A1_RANGE_DIVERGENCE_INVESTIGATION.md) — Range array divergence (V1 vs V2)
- [A3_POT_SIZE_INVESTIGATION.md](./A3_POT_SIZE_INVESTIGATION.md) — Pot size and rake divergence

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
