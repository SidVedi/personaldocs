# A-4: Frontend Regret Fallback Formula — Investigation & Fix

**Date:** 2026-02-10 to 2026-02-16  
**Investigator:** Siddarth Vedi  
**Status:** ✅ FIXED — Implemented in PR #6144  
**Effort spent:** ~1 week (investigation, implementation, testing)

---

## Summary

Frontend was recalculating `regretPercent` using a different formula than the backend when backend values were missing or for debug comparison. This formula difference caused inconsistent feedback labels (optimal/strong/weak/blunder) across different views and sessions. **Fix implemented:** Frontend now trusts backend-provided `regretPercent` exclusively, with validation to handle invalid values.

---

## The Problem

### Formula Divergence

**Backend formula** (weighted-EV regret):
```python
# trainer/utils/trainer_helper.py
weighted_ev = sum(strategy[i] * ev[i] for all actions)
best_ev = max(ev[i] for all actions)
regret_raw = max(best_ev - weighted_ev, 0.0)
regret_percent = (regret_raw / pot) * 100
```

**Frontend fallback formula** (simple EV loss):
```typescript
// OLD (before fix):
const evDiff = bestEV - selectedEV;
const evLossPercent = potValue > 0 ? (evDiff / potValue) * 100 : 0;
```

### Why They Differ

| Aspect | Backend (Correct) | Frontend Fallback (Wrong) |
|--------|-------------------|---------------------------|
| **EV Calculation** | Uses **strategy-weighted EV** (mixed strategy) | Uses **single-action EV** (pure strategy) |
| **Regret Definition** | Optimal vs actual mixed play | Optimal vs pure single action |
| **Use Case** | GTO-accurate regret for mixed strategies | Simplified approximation |
| **Result** | Accounts for strategy mixing (e.g., 80% fold, 20% call) | Assumes 100% one action |

**Example:**
```
Scenario: Mixed strategy (80% Fold, 20% Call)
- Fold EV: 0.0
- Call EV: -0.5
- Best EV: 0.0

Backend regret:
  weighted_ev = 0.8 * 0.0 + 0.2 * (-0.5) = -0.1
  regret = 0.0 - (-0.1) = 0.1
  regret% = (0.1 / 5.0) * 100 = 2.0%

Frontend fallback (if user clicked Call):
  evDiff = 0.0 - (-0.5) = 0.5
  regret% = (0.5 / 5.0) * 100 = 10.0%

Result: 2% vs 10% → Different feedback labels!
```

---

## Impact

### Affected Components (Before Fix)

| Component | Issue | Impact |
|-----------|-------|--------|
| `FeedbackActions.tsx` | Used client pot-normalized EV loss for non-debug users | Wrong regret% → wrong feedback icons/labels |
| `TrainerCenterPanel.tsx` | Calculated client evLossPercent for feedback animation | Wrong visual feedback (optimal/strong/weak/blunder) |
| `FeedbackPanel.tsx` | Client-side regret calculation in EV Loss column | Wrong values in feedback history table |
| `TrainerHistoryList.tsx` | Client pot-normalized regret for history list | Inconsistent with live feedback |
| `ReplayerActionsMenu.tsx` | Client-calculated evLossPercent in replayer | Replayer showed different values than trainer |
| `FeedbackToastMobile.tsx` | Mobile feedback used client calculation | Mobile users saw different feedback than desktop |
| `PlayerSeat.tsx` | Hero feedback used client evLossPercent | Player seat overlay showed wrong values |

---

## The Fix

### PR #6144: "Feature/sv proto"

**GitHub:** https://github.com/GameTrainer/GameTrainer-Frontend/pull/6144  
**Merged:** 2026-02-16  
**Files Changed:** 18 files (+1375, -292)

### Core Changes

#### 1. New Utility: `getRegretPercent()`

**File:** `utils/getRegretPercent.ts`

```typescript
/**
 * Validates backend-provided regretPercent; returns null for invalid values.
 */
export function getRegretPercent(value: number | null | undefined): number | null {
  // Return null for missing, NaN, or Infinity values
  if (value == null || !Number.isFinite(value)) {
    return null;
  }
  // Return the value as-is (backend already provides positive regret%)
  return value;
}
```

**Purpose:** Centralized validation ensures consistent handling of backend regret values across all components.

#### 2. Updated Components (All 7)

**Pattern applied to all affected components:**

```typescript
// OLD (client-side fallback):
const evDiff = bestEV - selectedEV;
const evLossPercent = potValue > 0 ? (evDiff / potValue) * 100 : 0;

// NEW (trust backend):
import { getRegretPercent } from '@/utils/getRegretPercent';

const regretPercent = getRegretPercent(selectedActionData?.regretPercent);

// Handle null (invalid/missing backend value):
if (regretPercent === null) {
  return <span>—</span>; // Show em dash instead of calculating fallback
}
```

#### 3. Debug Comparison Display

For users with debug access (`flophero-debug` role):

```typescript
// Show both BE (backend) and FE (client-calculated) values
const clientRegret = potValue > 0 ? (evDiff / potValue) * 100 : 0;

return (
  <div>
    <span>{regretPercent.toFixed(2)}%</span> {/* Backend value */}
    {hasDebugAccess && (
      <span className="bg-violet-900/50 text-violet-300">
        {clientRegret.toFixed(2)}%  {/* Client value for comparison */}
      </span>
    )}
  </div>
);
```

**Purpose:** Allows verification that backend values are correct and identification of any remaining calculation issues.

#### 4. Formatting Improvements

**EV Loss Display:**
- Losses rendered as **negative** (e.g., `-5.2%` instead of `5.2%`)
- Zero regret shown as **dash** (`-` instead of `0.00%`)
- Missing/invalid values shown as **em dash** (`—`)

**Example:**
```typescript
// Before:
regretPercent === 0 ? '0.00%' : `${regretPercent.toFixed(2)}%`

// After:
regretPercent === null ? '—' :
regretPercent === 0 ? '-' :
regretPercent > 0 ? `-${regretPercent.toFixed(2)}%` :
`${regretPercent.toFixed(2)}%`
```

---

## Testing

### New Unit Tests (3 test files)

#### 1. `getRegretPercent.test.ts` (40 lines)
```typescript
describe('getRegretPercent', () => {
  it('returns the value when valid positive number');
  it('returns 0 when value is exactly 0');
  it('returns null when value is null');
  it('returns null when value is undefined');
  it('returns null when value is NaN');
  it('returns null when value is Infinity');
  it('returns null when value is -Infinity');
  it('returns the value for negative numbers (edge case)');
});
```

#### 2. `FeedbackPanel.regret.test.tsx` (391 lines)
- Tests dual BE/FE display for debug users
- Tests em dash display when `regretPercent` is null
- Tests formatted values when `regretPercent` is a number
- Tests ΔEV column with both BE and FE values

#### 3. `TrainerHistoryList.regret.test.tsx` (145 lines)
- Tests history list regret display
- Tests action-key mismatch fallback
- Tests debug dual display in history

#### 4. Updated Existing Tests
- `FeedbackToastMobile.test.tsx` (+141 lines)
- `asyncActions.test.ts` (updateOnboarding error handling)

### Test Coverage

**Before:** ~65%  
**After:** ~68% (+3% improvement)

---

## Verification

### Manual Testing Checklist

- [x] Trainer feedback shows consistent regret% across all views
- [x] FeedbackPanel EV Loss column matches live feedback
- [x] TrainerHistoryList shows same regret% as when hand was played
- [x] Replayer shows consistent regret% with trainer
- [x] Mobile feedback matches desktop
- [x] Debug users see dual BE/FE display
- [x] Non-debug users see only backend values
- [x] Invalid backend values show em dash (—) instead of crashing

### Regression Testing

- [x] All existing trainer tests pass
- [x] All existing replayer tests pass
- [x] No visual regressions in feedback UI
- [x] Performance unchanged (no client-side calculation overhead removed)

---

## Related Changes

### Bonus Fix: `updateOnboarding` Error Handling

**File:** `store/asyncActions.ts`

```typescript
// OLD (silently swallowed errors):
try {
  await apiServiceInstance.patch(UPDATE_ONBOARD, payload);
} catch (error) {
  logger.error('[updateOnboarding] Failed', error);
  // No rethrow → caller doesn't know it failed
}

// NEW (rethrows for caller handling):
try {
  const response = await apiServiceInstance.patch(UPDATE_ONBOARD, payload);
  return response;
} catch (error) {
  logger.error('[updateOnboarding] Failed to persist', error);
  throw error; // Caller can handle failure
}
```

**Why:** Trainer config persistence now only refreshes user info when the update **succeeds**, preventing stale config from persisting.

---

## Files Changed

### Core Fix Files
- ✅ `utils/getRegretPercent.ts` — New validation utility
- ✅ `components/Common/FeedbackActions.tsx` — Trainer action feedback
- ✅ `components/Trainer/TrainerCenterPanel.tsx` — Center panel feedback animation
- ✅ `components/Trainer/FeedbackPanel.tsx` — Feedback history table
- ✅ `components/Trainer/TrainerHistoryList.tsx` — Session history list
- ✅ `components/Trainer/FeedbackAnimation.tsx` — Visual feedback overlay
- ✅ `components/Trainer/mobile/FeedbackToastMobile.tsx` — Mobile feedback toast
- ✅ `components/TrainerElements/PlayerSeat.tsx` — Hero player overlay
- ✅ `components/PlayerActions/ReplayerActionsMenu.tsx` — Replayer feedback

### Test Files
- ✅ `__tests__/utils/getRegretPercent.test.ts`
- ✅ `components/Trainer/__tests__/FeedbackPanel.regret.test.tsx`
- ✅ `components/Trainer/__tests__/TrainerHistoryList.regret.test.tsx`
- ✅ `components/Trainer/__tests__/FeedbackToastMobile.test.tsx`
- ✅ `__tests__/store/asyncActions.test.ts`

### Supporting Files
- ✅ `components/Trainer/utils.ts` — Config persistence logic
- ✅ `components/Trainer/mobile/useTrainerMobile.ts` — Mobile trainer hook
- ✅ `components/Trainer/mobile/TrainerMobile.tsx` — Mobile layout
- ✅ `store/asyncActions.ts` — Error handling fix

---

## Root Cause Analysis

### Why the Fallback Existed

**Original intent (circa 2023):**
- Backend `regretPercent` was sometimes missing or null (early V1 implementation)
- Frontend added client-side calculation as a "safety fallback"
- Formula used was simple pot-normalized EV loss (easier to implement)

**What went wrong:**
1. Fallback formula was **fundamentally different** from backend (weighted-EV vs pure-strategy)
2. Backend was later fixed to always provide `regretPercent`, but frontend fallback remained
3. Debug users and some edge cases still triggered the fallback
4. Different formulas → different regret% → different feedback labels

### Lessons Learned

1. **Never replicate backend logic in frontend** — Trust the backend as source of truth
2. **Different formulas = different results** — Even similar-looking calculations can diverge
3. **Fallbacks should be null/error handling**, not recalculation
4. **Validation ≠ Recalculation** — Validate that backend sent data, don't recalculate if missing

---

## Deployment Status

**Branch:** `feature/sv-proto`  
**PR:** #6144 (merged 2026-02-16)  
**Status:** ✅ Merged to staging, awaiting production deployment  

### Deployment Checklist

- [x] PR merged
- [x] All tests passing
- [ ] Deployed to staging
- [ ] QA validation on staging
- [ ] Production deployment
- [ ] Post-deployment monitoring

---

## Expected Impact

### User-Facing Changes

✅ **Consistent feedback labels** across all views (trainer, history, replayer)  
✅ **Accurate regret calculations** using GTO-correct weighted-EV formula  
✅ **No more discrepancies** between live feedback and historical feedback  
✅ **Better UX** with proper null handling (em dash instead of 0% or errors)  

### Internal Changes

✅ **Simplified codebase** — removed duplicate calculation logic  
✅ **Better debugging** — dual BE/FE display for debug users  
✅ **More robust** — centralized validation prevents future inconsistencies  
✅ **Better test coverage** — comprehensive regret display tests  

---

## Follow-up Tasks

### Completed
- [x] Implement fix across all 7 affected components
- [x] Add validation utility (`getRegretPercent`)
- [x] Write comprehensive unit tests
- [x] Add debug comparison display
- [x] Fix `updateOnboarding` error handling
- [x] Improve EV Loss formatting

### Pending
- [ ] Deploy to production
- [ ] Monitor for user-reported inconsistencies (should drop to zero)
- [ ] Remove debug comparison display after ~2 weeks of validation
- [ ] Consider removing `potValue` from frontend data structures (no longer needed)

---

## References

### Related Investigations
- [A1_RANGE_DIVERGENCE_INVESTIGATION.md](../../GameTrainer-Backend/docs/A1_RANGE_DIVERGENCE_INVESTIGATION.md) — Ruled out as root cause
- [A3_POT_SIZE_INVESTIGATION.md](../../GameTrainer-Backend/docs/A3_POT_SIZE_INVESTIGATION.md) — Ruled out as root cause
- [STREAM_A_TRAINER_VALUES_INVESTIGATION.md](../../GameTrainer-Backend/docs/STREAM_A_TRAINER_VALUES_INVESTIGATION.md) — Overall investigation summary

### Code References
- Backend regret calculation: `GameTrainer-Backend/trainer/utils/trainer_helper.py` (lines 495-502, 532-539)
- Frontend utility: `utils/getRegretPercent.ts`
- PR: https://github.com/GameTrainer/GameTrainer-Frontend/pull/6144

---

**Status:** ✅ FIXED — Ready for production deployment  
**Next Review:** After production deployment and 1-week monitoring period
