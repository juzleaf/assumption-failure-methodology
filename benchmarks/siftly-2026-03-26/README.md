# Siftly 4-way Blind Benchmark — 2026-03-26

## Target
- Repo: viperrcrypto/Siftly (commit 61e7bde)
- Stack: Next.js + Prisma + Chrome Extension
- Prompt: empty (blind test, no guidance)

## Methods Tested

| Method | Report File | VULN | HOLDS | UNCLEAR | Tool Calls | Tokens |
|--------|-------------|------|-------|---------|------------|--------|
| AFM v3.1 | `afm-v31-report.md` | 19 | 7 | 0 | — | — |
| AFM v3.2 (ABSTRACT) | `afm-v32-report.md` | 12 | 13 | 2 | 51 | 137K |
| AFM v3.2 rerun | `afm-v32-rerun-report.md` | 12 | 13 | 2 | 51 | 137K |
| AFM Radiate (expert-tweaked) | `afm-radiate-report.md` | 11 | 15 | 0 | 72 | 143K |
| C-plan dendritic | `c-plan-report.md` | 6 | 20 | 0 | — | — |
| CR+debug baseline | `cr-debug-report.md` | 47 findings (4C/8H/19M/16L) | — | — | — | — |

## Key Findings

1. **v3.2 calibration best**: 12V/13H/2U — balanced between aggressive v3.1 and conservative C-plan
2. **ABSTRACT/RADIATE exclusive discoveries**:
   - A1: bookmarklet missing `decodeHtmlEntities` — XSS vector
   - A2: cache invalidation missing from categorize route — confirmed real bug via code verification
3. **v3.1 most aggressive**: 19 VULN, potentially over-asserting
4. **C-plan most conservative**: 6 VULN, 20 HOLDS — two-phase architecture adds caution
5. **Expert-tweaked Radiate underperformed v3.2 original** — meta-commentary may provide "deceleration calibration" effect
6. **v3.2 rerun identical to first run** — reproducible results (n=2)

## A/B: Expert Tweaks vs Original

The linguistic + prompt engineering expert panel recommended 8 modifications to v3.2.
The modified version (Radiate) used more tool calls (72 vs 51) and tokens (143K vs 137K)
but produced worse calibration (11V/15H vs 12V/13H/2U).

Hypothesis: meta-commentary in v3.2 original acts as a "speed bump" forcing calibration reflection.
Status: unverified, n=1.

## Decisions Made

- **v3.1 frozen** as production primary
- **v3.2 original text saved as afm-radiate** (only ABSTRACT→RADIATE rename)
- **Expert 8 tweaks rejected** — data doesn't support at n=1
