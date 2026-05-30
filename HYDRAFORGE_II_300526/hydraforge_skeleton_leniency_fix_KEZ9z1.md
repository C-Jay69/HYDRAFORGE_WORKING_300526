# HydraForge — Skeleton-Leniency Root-Cause Fix

The OMITTED/ALLOCATED_ADVERSE taxonomy (lines 1901–1914) is correct and already present. The failure is in three seams, not in the rule. Three edits below. Do **not** delete the qualified skeleton rules at 1999 / 1792 / 1872 — they are correct; they just never fired because §8 was misclassified as OMITTED.

---

## EDIT 1 — Fix the disposition definitions + examples (around lines 1904–1910)

The gap: a protection that is *absent* while a *separate clause shifts that exact risk onto the reviewed party* is currently routing to OMITTED. The absence is the mechanism of the adverse allocation, not a draft gap. Replace the definitions and examples with:

> `OMITTED` — Provision entirely absent or blank **AND no other clause transfers that risk onto the reviewed party.** Tier leniency applies; score using floor and incompleteness framing.
>
> `ALLOCATED_ADVERSE` — Operative text assigns the risk against the reviewed party. **This INCLUDES the case where a protection is absent and a separate clause affirmatively transfers the now-unprotected risk to the reviewed party** ("as is," "no further information required," "Buyer accepts all liabilities," asymmetric rights to the counterparty). In that case the absence is weaponized — treat it as hostile drafting. Tier leniency does NOT apply, regardless of draft tier.
>
> Examples:
> - No indemnification section, and nothing else addresses liability → `OMITTED` → floor applies, incompleteness framing.
> - No indemnification section, **but a clause states "Buyer accepts all liabilities of Target"** → `ALLOCATED_ADVERSE` → absence is weaponized; score as hostile, NO floor.
> - "As is" acceptance + acknowledgment that no further information is required → `ALLOCATED_ADVERSE` (affirmative waiver of recourse).
> - Indemnification present but flips indemnity to Seller's benefit → `ALLOCATED_ADVERSE`.
>
> Tie-break rule: if a provision matches both an OMITTED and an ALLOCATED_ADVERSE pattern, it is `ALLOCATED_ADVERSE`. Absence never downgrades an adverse allocation.

This is the edit that makes §8 classify correctly. Once it's `ALLOCATED_ADVERSE`, your existing qualified rules (1999/1792) already withhold the floor on their own.

---

## EDIT 2 — Fix the output template example (line 2052)

The model is copying the example string as its answer. Replace the `Score Calibration Applied` example with a conditional one that does not hand over the leniency conclusion:

> **Score Calibration Applied:** [State the disposition split. e.g., "Tier 1 skeleton. Floor leniency applied ONLY to OMITTED provisions: [list]. NOT applied to ALLOCATED_ADVERSE provisions: [§4 'as is', §8 'Buyer accepts all liabilities', §13 asymmetric termination], scored as drafted hostile terms. Net tier adjustment: +0 — all CRITICAL findings are ALLOCATED_ADVERSE, so the floor is suppressed."]

Note what the new example *models*: it shows the +0 outcome for an all-adverse doc, so the model patterns toward suppressing the floor rather than printing "skeleton cannot be dangerous."

---

## EDIT 3 — Gate the code clamp on disposition (line 2559)

The clamp is currently unconditional on tier. It must not rescue a document whose critical findings are affirmatively adverse. This requires each finding passed into the scorer to carry `severity` and `disposition` (per the 1914 labeling mandate — see wiring note below).

Current (paraphrased from line 2553 / 2559):

```js
const { rawScore, tier, detectedConditions } = input;

if (tier <= 2) {
  const floor = TIER_FLOORS[tier];
  if (rawScore < floor) {
    // clamp up to floor
  }
}
```

Replace with:

```js
const { rawScore, tier, detectedConditions, findings = [] } = input;

// A Tier 1/2 floor is leniency for INCOMPLETENESS only.
// It must not rescue a document containing affirmatively adverse critical terms.
const hasAdverseCritical = findings.some(
  f => f.severity === 'CRITICAL' && f.disposition === 'ALLOCATED_ADVERSE'
);

if (tier <= 2 && !hasAdverseCritical) {
  const floor = TIER_FLOORS[tier];
  if (rawScore < floor) {
    // clamp up to floor, log as before
  }
}
// else: no floor. Adverse critical terms are scored as drafted, regardless of tier.
```

**Wiring note (the one thing to verify):** this assumes `findings` reaches the scorer with `severity` and `disposition` fields. Line 1914 already mandates the model *label* each finding OMITTED/ALLOCATED_ADVERSE, so the data exists in the LLM output — confirm it's parsed into the structured findings array the scorer receives, not left only in prose. If it's prose-only, that parse is the single addition you need.

---

## WHY BOTH PROMPT AND CODE

Two independent leniency mechanisms are in play:
- **Prompt-side (+15):** what you see today. The model narrates the bump itself. EDIT 1 + EDIT 2 stop it.
- **Code-side floor clamp:** dormant right now because your displayed 32 sits below the Tier-1 floor. But it springs the moment EDIT 1/2 work and the LLM starts emitting a correct sub-45 raw — the clamp would yank it back to ~55. EDIT 3 disarms it.

Fix only the prompt and the next run may *look* like the fix failed (score clamps to floor). Fix only the code and the model keeps narrating +15. They have to land together.

---

## REGRESSION TARGET

Re-run MERGER_AGREEMENT_1. Expect:
- §4, §8, §13 labeled `ALLOCATED_ADVERSE` in the disposition output.
- `Score Calibration Applied: ... Net tier adjustment: +0.`
- Headline score **below 32** and below 45 (adverse criticals, no floor).
- No "skeleton cannot be scored as dangerous" string anywhere — it's no longer in the template to copy.

If the score lands at ~55–60, EDIT 3 didn't take — the code clamp is still firing because `findings` isn't carrying `disposition` into the scorer. That's the first thing to check.
