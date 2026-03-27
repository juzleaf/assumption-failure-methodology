# Benchmark Data

All benchmarks: Claude Opus, empty prompt, no guidance. Same model for all methods.

## Summary reports

| File | What it covers |
|------|---------------|
| [round3-2026-03-26-summary.md](round3-2026-03-26-summary.md) | Full Round 3 report: 4 repos x 3 methods |
| [falsify-cross-repo-2026-03-27-summary.md](falsify-cross-repo-2026-03-27-summary.md) | Falsify proof engine across 4 repos |
| [falsify-cross-validation-2026-03-27.md](falsify-cross-validation-2026-03-27.md) | Honest cross-method validation (21 PROVEN, only 3 unique) |

## Per-repo comparisons (3-way: CR vs v3.1 vs Radiate)

| Repo | Language | Lines | Comparison |
|------|----------|-------|------------|
| 2fauth-worker | TypeScript | 18K | [comparison](2fauth-2026-03-26-comparison.md) |
| picolm | C | 2.5K | [comparison](picolm-2026-03-26-comparison.md) |
| OpenBSE | Rust | 15K | [comparison](openbse-2026-03-26-comparison.md) |
| zigpty | Zig+TS | 3.7K | [comparison](zigpty-2026-03-26-comparison.md) |

## Raw reports by method

### Standard Code Review (baseline)
- [2fauth CR](2fauth-2026-03-26-cr-debug.md) | [picolm CR](picolm-2026-03-26-cr-debug.md) | [openbse CR](openbse-2026-03-26-cr-debug.md) | [zigpty CR](zigpty-2026-03-26-cr-debug.md)

### AFM v3.1
- [2fauth v3.1](2fauth-2026-03-26-afm-v31.md) | [picolm v3.1](picolm-2026-03-26-afm-v31.md) | [openbse v3.1](openbse-2026-03-26-afm-v31.md) | [zigpty v3.1](zigpty-2026-03-26-afm-v31.md)

### AFM Radiate (experimental variant)
- [2fauth radiate](2fauth-2026-03-26-afm-radiate.md) | [picolm radiate](picolm-2026-03-26-afm-radiate.md) | [openbse radiate](openbse-2026-03-26-afm-radiate.md) | [zigpty radiate](zigpty-2026-03-26-afm-radiate.md)

### Falsify (proof engine, separate tool)
- [2fauth falsify](2fauth-2026-03-27-falsify.md) | [picolm falsify](picolm-2026-03-27-falsify.md) | [openbse falsify](openbse-2026-03-27-falsify.md)
- [zigpty falsify v1](zigpty-2026-03-27-falsify-v1.md) | [v2](zigpty-2026-03-27-falsify-v2.md) | [v3](zigpty-2026-03-27-falsify-v3.md) | [experiment summary](zigpty-2026-03-27-falsify-experiment.md)

### Pipeline experiments
- [picolm CR→Falsify serial](picolm-2026-03-27-cr-falsify.md) — CR seeds fed into Falsify
- [picolm v3.1 vs v3.2-exp](picolm-2026-03-27-v31-vs-v32exp.md) — Merger experiment (disproved)

### Siftly 4-way blind test
- [siftly-2026-03-26/](siftly-2026-03-26/) — Full directory with 6 reports + README

## How to read these

- **CR findings**: What Claude finds with standard code review (no skill loaded)
- **AFM VULNERABLE**: Assumptions confirmed broken, with reasoning chain
- **AFM HOLDS**: Assumptions tested and confirmed valid (defense map)
- **Falsify PROVEN**: Concrete failure proof constructed
- **Falsify HARDENED**: Survived adversarial attack attempt

## Methodology note

These benchmarks were run for methodology research, not as security audits. Findings have been cross-validated across methods but not manually confirmed by domain experts. Treat them as data points for evaluating the skill, not as vulnerability reports.
