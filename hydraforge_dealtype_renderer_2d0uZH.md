# HydraForge — Data-Derived DEAL-TYPE CLASSIFICATION Renderer

## The principle

The Adjudicator has been *authoring* the DEAL-TYPE CLASSIFICATION section as prose, including suppression rationales like "TSA Absence: No — equity acquisition of standalone entity." That text has been wrong in every run: it says "equity" on a deal classified `STATUTORY_MERGER`, and it fires suppressions that should be *disabled* on a `CONTESTED` classification. The A2 markdown scan catches the symptom. This kills the source.

Same move you already made for the L3-B reconciler block: **render the section from structured state; don't let the LLM narrate it.** Once this ships, there is no foreign-structure prose for A2 to catch — and that silence is the success signal.

A second correctness gain, bigger than the wording: these three suppressions are not universally valid. Under an **asset purchase**, TSA absence, source-code escrow, and assumption-of-liabilities are **live risks, not suppressible** — assets are extracted from the seller, liabilities don't ride along by operation of law, and the dev team may not come with the code. The hardcoded "equity" rationale was masking that the suppression's *validity* depends on structure. The renderer below encodes that, so an asset-purchase deal stops getting these risks silently waved off.

---

## Data shape consumed

```ts
interface DealTypeState {
  dealType: 'STATUTORY_MERGER' | 'EQUITY_PURCHASE' | 'ASSET_PURCHASE';
  confidence: 'HIGH' | 'MEDIUM' | 'CONTESTED';
  candidateStructures?: { structure: DealTypeState['dealType']; drivingClause: string }[];
}
```

The Analyst already emits all three (the log confirmed `deal_type` and `classification_confidence` arrive correctly; `candidateStructures` is the field to make sure is populated when `CONTESTED`).

---

## Suppression rule → per-structure disposition

This table replaces the hardcoded rationale strings. Each structure-keyed rule resolves to `{ suppressed, rationale }` based on `dealType` — not a fixed "equity" string. Add rows for any other structure-keyed rules you run.

```ts
type Structure = DealTypeState['dealType'];
interface SuppressionDisposition { suppressed: boolean; rationale: string; }

const SUPPRESSION_MATRIX: Record<string, Record<Structure, SuppressionDisposition>> = {
  'TSA Absence as Critical': {
    STATUTORY_MERGER: { suppressed: true,  rationale: 'surviving entity inherits operations by operation of law; no standalone separation required' },
    EQUITY_PURCHASE:  { suppressed: true,  rationale: 'entity acquired intact as a going concern; no carve-out from a parent' },
    ASSET_PURCHASE:   { suppressed: false, rationale: 'LIVE RISK — assets are extracted from seller; transition services are typically required and their absence is material' },
  },
  'Source Code Escrow as Material Risk': {
    STATUTORY_MERGER: { suppressed: true,  rationale: 'code remains within the surviving entity' },
    EQUITY_PURCHASE:  { suppressed: true,  rationale: 'code remains within the acquired entity' },
    ASSET_PURCHASE:   { suppressed: false, rationale: 'LIVE RISK — code is among transferred assets; escrow/dev-team continuity is material' },
  },
  'Assumption of Liabilities as Distinct Mechanism': {
    STATUTORY_MERGER: { suppressed: true,  rationale: 'liabilities vest in the surviving entity by operation of law; no separate assumption mechanism' },
    EQUITY_PURCHASE:  { suppressed: true,  rationale: 'liabilities remain in the entity; no separate assumption mechanism' },
    ASSET_PURCHASE:   { suppressed: false, rationale: 'LIVE RISK — liabilities do NOT transfer unless expressly assumed; assumption is a distinct, material mechanism' },
  },
};
```

---

## The renderer

```ts
function renderDealTypeSection(state: DealTypeState): string {
  const { dealType, confidence, candidateStructures = [] } = state;
  const label = { STATUTORY_MERGER: 'Statutory Merger', EQUITY_PURCHASE: 'Equity Purchase', ASSET_PURCHASE: 'Asset Purchase' }[dealType];

  const lines: string[] = [];
  lines.push('DEAL-TYPE CLASSIFICATION');
  lines.push(`Transaction Structure: ${label}${confidence === 'CONTESTED' ? ' (stated intent) — CONTESTED' : ''}`);
  lines.push(`Classification Confidence: ${confidence}`);

  if (confidence === 'CONTESTED') {
    lines.push('Candidate Structures:');
    for (const c of candidateStructures) {
      lines.push(`  • ${({ STATUTORY_MERGER: 'Statutory Merger', EQUITY_PURCHASE: 'Equity Purchase', ASSET_PURCHASE: 'Asset Purchase' })[c.structure]} — ${c.drivingClause}`);
    }
    lines.push('Structure-Keyed Suppressions: DISABLED (CONTESTED)');

    // If any candidate would make a suppressed risk live, say so explicitly.
    const liveUnderCandidate = Object.entries(SUPPRESSION_MATRIX)
      .filter(([, byStruct]) => candidateStructures.some(c => !byStruct[c.structure].suppressed))
      .map(([rule]) => rule);
    if (liveUnderCandidate.length) {
      lines.push(`  Rationale: under candidate structure(s), these are LIVE risks and cannot be suppressed: ${liveUnderCandidate.join('; ')}. Evaluated worst-case.`);
    } else {
      lines.push('  Rationale: deal type unresolved; suppression withheld pending classification.');
    }
  } else {
    lines.push('Structure-Keyed Suppressions: APPLIED');
    for (const [rule, byStruct] of Object.entries(SUPPRESSION_MATRIX)) {
      const d = byStruct[dealType];
      lines.push(`  • ${rule}: ${d.suppressed ? 'Suppressed' : 'NOT suppressed'} — ${d.rationale}`);
    }
  }
  return lines.join('\n');
}
```

---

## Wiring (both halves required)

1. **Render this in place of the Adjudicator's section.** In `analyses.ts`, call `renderDealTypeSection(dealTypeState)` and splice its output into the report where the DEAL-TYPE CLASSIFICATION block goes — the same pattern as `formatReconcilerResult()`.
2. **Delete the DEAL-TYPE CLASSIFICATION block from the Adjudicator prompt.** As long as the prompt asks the model to write it, you get two competing sections and the prose one re-introduces the bug. Remove it entirely; the data path owns this section now.

---

## What this does to A2

A2's markdown scan should now **go quiet** — there's no LLM-authored "equity acquisition" prose left to match, because the section is rendered from `dealType`. Keep the scan as a backstop, but its job changes from "fires every run" to "fires only if someone reintroduces a prose path." That's the correct end state: the check exists, but the thing it guards against is now structurally hard to produce.

Two follow-ups on A2 while you're here, per the brittleness note:
- Broaden foreign-term patterns beyond `equity` to include `stock`, `share`, `asset purchase`, `asset acquisition` — so a rephrase doesn't slip past.
- Anchor the scan to a stable marker, not the header text, if you can — heading strings get renamed.

---

## Regression target — re-run MERGER_AGREEMENT_1

This contract is `CONTESTED` (merger §1 vs. asset-flavored §5). Expect the rendered section to read:

```
DEAL-TYPE CLASSIFICATION
Transaction Structure: Statutory Merger (stated intent) — CONTESTED
Classification Confidence: CONTESTED
Candidate Structures:
  • Statutory Merger — §1 "parties intend this to be a merger"
  • Asset Purchase — §5 "all material contracts of Target shall be assumed by Buyer"
Structure-Keyed Suppressions: DISABLED (CONTESTED)
  Rationale: under candidate structure(s), these are LIVE risks and cannot be suppressed:
  TSA Absence as Critical; Source Code Escrow as Material Risk; Assumption of Liabilities
  as Distinct Mechanism. Evaluated worst-case.
```

- No "equity acquisition" string anywhere — the seven-run-old bug is gone at the source.
- `A2: PASS`, and this time the PASS is **legitimate**, not vacuous — there's genuinely no contradictory prose, rather than contradictory prose the scan happened not to read.
- If the old "No — equity acquisition of standalone entity" lines still appear, the Adjudicator prompt block wasn't deleted (wiring step 2). That's the first thing to check.
