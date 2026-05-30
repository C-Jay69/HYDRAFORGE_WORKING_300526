# HydraForge — Deterministic Cross-Layer Reconciler (L3-B v3)

## Why this version, not the last one

v2 was a forced assertion table *inside the synthesis prompt*. It still printed "No conflicts" because it was the same LLM grading its own pass — in-context self-checks are unreliable by nature. Two of the three fixes that worked this session (floor clamp, disposition gate) worked because the check was **external and deterministic**. This applies that lesson: the reconciler becomes a post-processing function that reads the structured outputs the LLM already produced and computes conflicts. It does not ask the model whether conflicts exist.

The showcase case — classified `STATUTORY_MERGER` but suppression rationale says "equity acquisition" — is a pure string check. No model judgment required, so no rubber-stamping possible.

---

## Expected input shape

The reconciler reads structures your pipeline already emits. Map these to your actual field names:

```js
/**
 * pipeline output shape (adapt names to yours)
 * {
 *   dealType: 'STATUTORY_MERGER' | 'EQUITY_PURCHASE' | 'ASSET_PURCHASE',
 *   classificationConfidence: 'HIGH' | 'MEDIUM' | 'CONTESTED',
 *   suppressions: [{ item: 'TSA Absence', applied: true, rationale: 'equity acquisition of standalone entity' }],
 *   findings:     [{ topic: 'TSA', severity: 'CRITICAL' | 'HIGH' | 'MATERIAL', disposition: 'OMITTED' | 'ALLOCATED_ADVERSE' }],
 *   netTierBump: 0,
 *   recommendation: 'DO_NOT_PROCEED' | 'PROCEED_WITH_CONDITIONS' | 'PROCEED',
 * }
 */
```

The one thing to confirm: `suppressions` must arrive as structured objects with a `rationale` string, not as a prose blob. Your deal-type section already enumerates them ("TSA Absence as Critical: No — equity acquisition…"), so this is a parse, not new analysis.

---

## The reconciler

```js
const STRUCTURE_LABEL = {
  STATUTORY_MERGER: 'merger',
  EQUITY_PURCHASE: 'equity / stock purchase',
  ASSET_PURCHASE: 'asset purchase',
};

// Terms that name a DIFFERENT structure than the one classified.
const FOREIGN_STRUCTURE_TERMS = {
  STATUTORY_MERGER: [/equity (deal|acquisition|purchase)/i, /stock purchase/i, /share purchase/i, /asset (purchase|acquisition)/i, /100%\s*equity/i],
  EQUITY_PURCHASE:  [/\bmerger\b/i, /surviving (corporation|entity)/i, /asset (purchase|acquisition)/i],
  ASSET_PURCHASE:   [/\bmerger\b/i, /surviving (corporation|entity)/i, /equity (deal|acquisition)/i, /stock purchase/i],
};

const STRUCTURE_KEYED_SUPPRESSIONS = ['TSA Absence', 'Source Code Escrow', 'Assumption of Liabilities'];

function reconcile(out) {
  const results = [];
  const fail = (id, detail, fix) => results.push({ id, status: 'FAIL', detail, fix });
  const pass = (id) => results.push({ id, status: 'PASS' });

  // A2 — deal-type vocabulary coherence (the live bug). Pure string check.
  const foreign = FOREIGN_STRUCTURE_TERMS[out.dealType] || [];
  const a2 = [];
  for (const s of out.suppressions) {
    for (const re of foreign) {
      if (re.test(s.rationale)) a2.push(`"${s.item}": rationale says "${s.rationale}" but deal is ${STRUCTURE_LABEL[out.dealType]}`);
    }
  }
  a2.length
    ? fail('A2', a2, 'Rewrite rationale to the classified structure; re-test whether the suppression still holds under it.')
    : pass('A2');

  // A1 — severity contradiction: a suppressed item rated CRITICAL/HIGH elsewhere.
  const a1 = [];
  for (const s of out.suppressions.filter(x => x.applied)) {
    const hit = out.findings.find(
      f => normalize(f.topic) === normalize(s.item) && ['CRITICAL', 'HIGH'].includes(f.severity)
    );
    if (hit) a1.push(`"${s.item}" suppressed as not-critical, but rated ${hit.severity} in findings`);
  }
  a1.length
    ? fail('A1', a1, 'Un-suppress and surface at the highest stated severity. Suppression loses to an explicit critical finding.')
    : pass('A1');

  // A3 — leniency must not touch ALLOCATED_ADVERSE criticals. Pure check.
  const adverseCritical = out.findings.some(f => f.severity === 'CRITICAL' && f.disposition === 'ALLOCATED_ADVERSE');
  (adverseCritical && out.netTierBump > 0)
    ? fail('A3', [`netTierBump=${out.netTierBump} with an ALLOCATED_ADVERSE critical present`], 'Recompute calibration; adverse criticals get no floor.')
    : pass('A3');

  // A4 — verdict / leniency coherence. Pure check.
  (out.recommendation === 'DO_NOT_PROCEED' && out.netTierBump > 0)
    ? fail('A4', [`recommendation=DO_NOT_PROCEED with netTierBump=${out.netTierBump}`], 'Verdict wins; the mercy goes.')
    : pass('A4');

  // A5 — no structure-keyed suppression on a CONTESTED classification. Pure check.
  if (out.classificationConfidence === 'CONTESTED') {
    const a5 = out.suppressions
      .filter(s => s.applied && STRUCTURE_KEYED_SUPPRESSIONS.includes(s.item))
      .map(s => `"${s.item}" suppressed despite CONTESTED deal-type`);
    a5.length
      ? fail('A5', a5, 'Disable structure-keyed suppression; evaluate worst-case across candidate structures.')
      : pass('A5');
  } else {
    pass('A5');
  }

  const conflicts = results.filter(r => r.status === 'FAIL');
  return { results, conflicts, clean: conflicts.length === 0 };
}

function normalize(s) {
  return (s || '').toLowerCase().replace(/[^a-z0-9]/g, '');
}
```

---

## What to do with the result

Two options. I'd ship (1) first — it's the one that makes the bug impossible to print clean.

**(1) Emit the actual results table, never a self-judged summary.** Replace the "No cross-layer premise conflicts identified" line with the rendered output of `reconcile()`. "A2: PASS" is a verifiable claim because code produced it; "no conflicts identified" was an opinion. On any FAIL, the `fix` field tells the operator (or the next LLM call) exactly what to correct.

**(2) Auto-resolve the deterministic ones before render.** A2 → rewrite the rationale to the classified label. A1/A5 → flip `applied` to false and re-inject the finding. A3/A4 → re-run the scorer with leniency forced to 0. Then re-render. This closes the loop without a human, for the conflicts that have a single correct resolution.

The only assertion that may need an LLM is A1 *when* your suppressions and findings don't share a `topic` key — then matching is fuzzy and you'd route the candidate pairs to a model to confirm. Everything else is deterministic and should never live in the synthesis prompt.

---

## Regression target

Re-run MERGER_AGREEMENT_1:
- **A2 → FAIL**, citing all three suppressions ("equity" rationale on a `STATUTORY_MERGER`). This is the conflict the current run reports as clean.
- A3 → PASS (you already got netTierBump to 0).
- A4 → PASS (DO_NOT_PROCEED + bump 0).
- A1 → PASS or FAIL depending on whether TSA is keyed; with the current "TSA not critical" suppression and SYNTH-02 now rating it Material (not Critical), this may legitimately pass — which is itself correct behavior, not a miss.

If A2 still renders PASS after install, the suppressions aren't reaching `reconcile()` as structured objects — check the parse, same failure mode as the disposition field not reaching the scorer last round.
