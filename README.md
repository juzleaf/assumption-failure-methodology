# AFM — Assumption Failure Methodology

You have a bug. AI patches it. The patch breaks something else. AI patches that too. After enough rounds, you have a pile of fixes solving the wrong problems — the root cause never gets touched.

AFM breaks this cycle. Instead of asking "what's wrong," it asks **"what assumption made this code exist, and is that assumption actually true?"**

## What the difference looks like

Standard code review says:
> "Missing input validation on line 42."

AFM says:
> **Seed**: `auth.ts:42` — token refresh assumes old token is still valid during refresh window
>
> **REVERSE**: Developer assumed refresh and validation are atomic.
>
> **FORWARD**: They're not — 200ms window. Second request during refresh → stale token → old permissions.
>
> **WHY**: OAuth library docs show synchronous example. Developer assumed async follows the same guarantee.
>
> **Verdict**: VULNERABLE — same pattern in `session.ts:18`, `api.ts:95`, `middleware.ts:33`

AFM doesn't just find the bug. It finds *why the bug exists*, *what assumption caused it*, and *where else that assumption appears*.

## Install

```bash
npx skills add juzleaf/assumption-failure-methodology
```

Or manually:

```bash
mkdir -p ~/.claude/skills/afm
curl -o ~/.claude/skills/afm/SKILL.md \
  https://raw.githubusercontent.com/juzleaf/assumption-failure-methodology/main/SKILL.md
```

Then trigger with: `run afm on this codebase`

## How it works

Two atomic operations that chain freely:

- **REVERSE**: Code → trace back to the hidden assumption behind it
- **FORWARD**: Assumption → predict what breaks if it fails → verify in code

No phases, no steps. Each result feeds back as a new lead. Three verdicts: **VULNERABLE** (assumption broken — go deeper + ask WHY), **HOLDS** (defense works — challenge it), **UNCLEAR** (defer).

The full approach with examples: [`docs/approach.md`](docs/approach.md)

## Results

Tested on 4 repos across 4 languages. CR and AFM find ~60% non-overlapping issues — they complement, not compete.

| Repo | Language | Lines | CR findings | AFM VULNERABLE | AFM HOLDS |
|------|----------|-------|-------------|----------------|-----------|
| 2fauth-worker | TypeScript | 18K | 24 | 14 | 4 |
| picolm | C | 2.5K | 21 | 8 | 8 |
| OpenBSE | Rust | 15K | 21 | 7 | 9 |
| zigpty | Zig+TS | 3.7K | 25 | 13 | 44 |

- Token cost: AFM uses ~35% more tokens than standard CR (126K vs 93K per repo average)
- Each repo tested once (N=1). LLM outputs are stochastic — treat numbers as directional, not precise
- Overlap judged by the author. No third-party blind evaluation
- One external validation: a project maintainer confirmed and fixed 3/3 reported findings

Full reports: [`benchmarks/`](benchmarks/)

## Key discovery

Seed examples in prompts are poison — they cause pattern-matching instead of analysis. Zero-shot beat few-shot in every controlled test. A blank prompt outperformed a 200-line structured protocol.

## Limitations

**I'm not a security researcher.** I built this because I was frustrated with AI patching symptoms. The findings have been cross-validated across 4 methods with 86% agreement rate, but they have NOT been manually confirmed by domain experts. Some may be hallucinations.

**N=1 benchmarks.** Each repo was tested once per method. I don't have variance data yet.

**If you find a false positive, please open an issue.** That data is the most valuable contribution you can make.

**Works best on assumption-dense code.** Cross-boundary logic (auth, crypto, FFI, cross-thread, cross-platform) produces more findings. Simple CRUD apps won't see much benefit.

**Single-model testing.** All benchmarks use Claude Opus. Behavior on other models is unknown.

**Costs more tokens.** ~35% more than standard code review.

**I'm learning.** This is a beginner's experiment in prompt engineering for code reasoning. I don't claim expertise. If you see something I'm doing wrong, tell me.

## More

- [Origin story — 10 assumptions that turned out wrong](docs/origin-story.md)
- [The full approach — REVERSE/FORWARD + WHY mechanism](docs/approach.md)
- [The parallel branching vision](docs/vision.md)
- [Beyond code review](docs/beyond-code-review.md)

## Contributing

- **Report false positives** — the single most valuable contribution
- **Report true positives** — confirms the approach works
- **Try it on your codebase** — different languages/domains produce different results
- **Ideas for parallel branching** — if you have thoughts on implementing the full tree-shaped exploration, I'd love to hear them

## License

[MIT](LICENSE)
