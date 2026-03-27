---
name: afm
description: "Assumption Failure Methodology v3.1 — seed-first + pure engine + WHY questioning. Phase 1: broad sweep to find anything wrong, inconsistent, or fragile — any dimension. Phase 2: for each seed, interrogate the designer (three questions), then freely chain REVERSE/FORWARD on the board. Verdicts: VULNERABLE (confirmed, go deeper + ask WHY), HOLDS (defense found — challenge the defense's own assumptions), UNCLEAR (defer). Every seed MUST get a verdict. Trigger: 'afm', 'assumption failure', 'run afm', 'assumption review'. Best for: code analysis, architecture reviews, finding hidden assumptions across all dimensions — why things break, why designs fail, why logic is wrong."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
---

# AFM Fusion: Seed + Engine + WHY

> Seed everything. Interrogate. REVERSE and FORWARD freely. At every VULNERABLE, ask WHY.
> Results are new seeds. The cycle continues until the board is clear.

## Phase 1: Seed

Sweep the entire codebase. Produce seeds — anything that seems wrong, inconsistent, fragile, or surprising. Any dimension. One line each, file:line. No analysis. Just mark.

Do not limit yourself to any single category. Follow what the code tells you.

**Output**: seed list.

## Phase 2: Assume + Verify (the loop)

For each seed, interrogate the simulated designer:

- **"Why did you write it this way?"** — premise
- **"What does it actually do?"** — mechanism
- **"What does this promise?"** — effect

Contradictions = leads. All leads go on the board (hot/warm/cold).

### Two operations, free combination

**REVERSE**: From observation → what assumption is behind this?

**FORWARD**: From assumption → if it holds, what follows? If it fails, what goes wrong? Predict → verify in code.

Chain in any order. The board drives the sequence.

### At every VULNERABLE, ask WHY

When you confirm VULNERABLE, before moving on:

- **WHY did the developers assume this?** What evidence created the false confidence?
- **Generalize the meta-pattern.** (e.g., "validation in one place creates the illusion of universal coverage")
- **Where else does this meta-pattern apply?** grep. Each match is a new hot lead.

This is not optional. Every VULNERABLE gets a WHY.

### Verdicts and what to do next

**VULNERABLE** — assumption proven false, impact traced.
→ NOW GO DEEPER: what else does this affect? What depends on this assumption? → new seeds → back to interrogation. Ask WHY (mandatory).

**HOLDS** — this assumption appears to hold. A check, validation, or defense exists.
→ Before confirming, ask: **"What evidence would prove this check doesn't actually protect?"**
→ Then actively search for that counter-evidence:
  - Does it cover ALL callers? grep every caller. Uncovered paths = new HOT leads.
  - Can it be bypassed? (different encoding, race condition, alternative entry point)
  - Does it depend on another assumption that might fail?
→ If you find counter-evidence → flip to VULNERABLE. If you genuinely cannot → confirm HOLDS.

**UNCLEAR** — hard to trace, not enough evidence yet.
→ Mark as `needs-more-time`, cold lead, come back later.

### Board management

After each operation: new leads → board. VULNERABLE/HOLDS → record. Pick hottest. Repeat.

**Every seed MUST receive a verdict.** No seed stays as "seed" at the end. Investigate each one and assign: VULNERABLE, HOLDS, or UNCLEAR. If you run low on context, batch remaining seeds with quick verdicts rather than leaving them unprocessed.

Board clear = done.

## Guards

**No phantoms**: grounded in code you read. Quote file:line.
**Concrete reversal**: grep, not speculation.
**Parsimony**: 3+ independent failures → find simpler shared root.

## Output

Seed map (total → VULNERABLE → noise).
Root causes (grouped by shared failed assumption + meta-pattern from WHY).
Assumption radiation map.
Impact chains.
Defense map (HOLDS + coverage gaps).
Unexplored (UNCLEAR, needs-more-time).
