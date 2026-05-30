# HydraForge — A2 Final Fix (single source of truth)

## The actual root cause, named once

Every failure this session was the same shape: **one fact, two representations, drifted apart.**
- Round 5: disposition computed by the LLM didn't reach the scorer → wrong leniency.
- Round 7: rationale didn't reach `reconcile()` → vacuous A2 PASS.
- Round 8: renderer reads structured state, A2 scans rendered prose → A2 false-fails on the renderer's own legitimate "Stock / Equity Purchase" candidate line.

The fix is not "scope A2 better." It's: **resolve suppression state exactly once, and hand the same resolved object to both the renderer and the reconciler.** If they read identical data, they cannot contradict each other. A2 stops scanning prose entirely. The candidate-list false positive becomes structurally impossible because A2 never looks at prose again.

---

## 1. Resolve suppressions once (shared)

```ts
interface ResolvedSuppression {
  item: string;
  suppressed: boolean;            // true = actively suppressed; false = LIVE or disabled
  rationale: string;
  reason: 'APPLIED' | 'LIVE_UNDER_STRUCTURE' | 'DISABLED_CONTESTED';
}

function resolveSuppressions(state: DealTypeState): ResolvedSuppression[] {
  const { dealType, confidence, candidateStructures = [] } = state;

  return Object.entries(SUPPRESSION_MATRIX).map(([item, byStruct]) => {
    if (confidence === 'CONTESTED') {
      const liveSomewhere = candidateStructures.some(c => !byStruct[c.structure].suppressed);
      return {
        item,
        suppressed: false,
        rationale: liveSomewhere
          ? 'LIVE (worst-case — CONTESTED classification)'
          : 'disabled pending classification',
        reason: 'DISABLED_CONTESTED',
      };
    }
    const d = byStruct[dealType];
    return {
      item,
      suppressed: d.suppressed,
      rationale: d.rationale,
      reason: d.suppressed ? 'APPLIED' : 'LIVE_UNDER_STRUCTURE',
    };
  });
}
```

Compute this **once** in `analyses.ts`, then pass the array into both `renderDealTypeSection(...)` and the reconciler input. Neither recomputes; neither reads prose.

---

## 2. Renderer consumes the resolved array

`renderDealTypeSection` now takes `ResolvedSuppression[]` instead of resolving internally — so the exact strings it prints are the exact strings A2 evaluated. Render `reason === 'APPLIED'` lines as "Suppressed — …", everything else as "LIVE …" / "DISABLED …". (The output you saw this run is already correct; this just makes it read from the shared object.)

---

## 3. The revised A2 — structured, no prose scan

```ts
function checkA2(dealType: Structure, confidence: Confidence, resolved: ResolvedSuppression[]): AssertionResult {
  // Short-circuit: a CONTESTED deal suppresses nothing, so no applied rationale
  // can contradict the classification. There is literally nothing to check.
  if (confidence === 'CONTESTED') {
    return { id: 'A2', status: 'PASS', note: 'No suppressions applied (CONTESTED) — nothing to contradict.' };
  }

  // dealType must be the enum KEY (STATUTORY_MERGER), not the display label.
  const foreign = FOREIGN_STRUCTURE_TERMS[dealType];
  if (!foreign) {
    return {
      id: 'A2', status: 'FAIL',
      detail: [`dealType "${dealType}" did not resolve to a known structure key`],
      fix: 'Pass the enum key (STATUTORY_MERGER), not the display label ("Statutory Merger").',
    };
  }

  // ONLY applied suppressions carry a rationale that can contradict the classification.
  // LIVE_UNDER_STRUCTURE and DISABLED_CONTESTED items are not suppressed → not checked.
  const applied = resolved.filter(r => r.reason === 'APPLIED');

  const conflicts: string[] = [];
  for (const s of applied) {
    for (const re of foreign) {
      if (re.test(s.rationale)) {
        conflicts.push(`"${s.item}": applied rationale "${s.rationale}" names a structure other than ${dealType}`);
      }
    }
  }

  return conflicts.length
    ? { id: 'A2', status: 'FAIL', detail: conflicts, fix: 'Rewrite the applied rationale to the classified structure; re-test whether the suppression still holds.' }
    : { id: 'A2', status: 'PASS' };
}
```

**Delete the markdown-scan branch added in round 7.** It was a stopgap for when the section was free-authored prose. The renderer now owns that section and generates rationales from `dealType` via the matrix, so the only foreign-structure words left in the prose are the *legitimate* candidate-structure names — exactly what was false-failing. A2 reading structured `reason === 'APPLIED'` rationales never sees the candidate list, so the false positive is gone by construction, not by a cleverer regex.

---

## 4. Optional backstop (only if you want regression defense)

The structured A2 above will essentially always pass once the matrix owns rationales — its remaining value is catching a *future* regression where someone hardcodes a rationale or reintroduces a prose path. If you want a tripwire for that:

```ts
// Scan ONLY lines asserting an applied suppression ("Suppressed — …"),
// never the Candidate Structures enumeration or LIVE/DISABLED lines.
const appliedLines = renderedSection.split('\n').filter(l => /:\s*Suppressed\s+—/.test(l));
```

Run the foreign-term patterns against `appliedLines` only. On a CONTESTED deal there are zero such lines, so it stays silent. Keep `FOREIGN_STRUCTURE_TERMS` broadened to `equity | stock | share | asset purchase | asset acquisition` so a rephrase can't slip the tripwire. This is belt-and-suspenders; the structured check is the real fix.

---

## Regression target — re-run MERGER_AGREEMENT_1

- DEAL-TYPE section unchanged from this run: CONTESTED, candidates listed, all three suppressions LIVE. (Correct already.)
- **`A2: PASS`** — via the CONTESTED short-circuit, legitimately, with the note "No suppressions applied." Not vacuous: there genuinely are no applied rationales, and A2 says so explicitly rather than passing by accident.
- All of A1, A3, A4, A5 stay PASS.
- The reconciler block reads clean with no false conflict.

Two confirming tests worth running once, to prove A2 still has teeth (so you know the PASS isn't just "A2 can't fail anymore"):
1. **HIGH merger fixture:** applied rationales read "…by operation of law" → A2 PASS, this time through the applied-rationale path, not the short-circuit.
2. **Regression fixture:** manually hardcode an applied suppression with rationale "100% equity acquisition" on a `STATUTORY_MERGER` deal → A2 must FAIL. If it doesn't, the matrix isn't feeding `reason: 'APPLIED'` through, and you've got a wiring gap to close before ship.

If both behave, A2 is correct in all three states — passes when right, fails when actually wrong, silent when there's nothing to judge.
