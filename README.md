# AFM — Assumption Failure Methodology

**A thinking framework for AI-assisted code review and debugging.**

## Why this exists

I write code with AI. When something breaks, AI patches it. When the patch breaks something else, AI patches that too. After enough rounds, you have a pile of fixes solving the wrong problems — the symptoms keep changing but the root cause never gets touched.

I'm not a security researcher or a methodologist. I'm an indie developer who got frustrated watching AI code reviewers keep patching at the same layer instead of stepping back and asking: **"What assumption made this code exist in the first place? Is that assumption actually true?"**

That question became this skill.

## What I learned building this

AFM went through 47 iterations over 8 days. Along the way, 10 of my own assumptions about how it should work turned out to be wrong. Some highlights:

- **Seed examples are poison.** I included example findings to "guide" the model. Controlled experiments showed they were the single strongest source of bias — the model pattern-matched to examples instead of analyzing actual code. Removing all examples improved results. Zero-shot beat few-shot.

- **My methodology committed its own error.** AFM is designed to detect "check coverage illusion" — when a check exists but doesn't actually cover all cases. AFM's own seed examples were all security-focused, creating the illusion that all dimensions were covered. I only caught this by running AFM on itself. The fix for this (adding 10-dimensional examples) created a *checklist illusion* — triple recursion.

- **A blank prompt beat my 200-line protocol.** Early version: 431 seconds, 9 findings. Empty prompt: 160 seconds, 18 findings. The structure was a cage, not a scaffold. Stripping it down to two atomic operations + a loop was the breakthrough.

- **"What's NOT broken" is half the value.** About 60% of AFM's output is HOLDS — places where code looks suspicious but actually works, with reasoning for why. For AI-assisted coding, this is a defense map: it tells you what NOT to change when your AI suggests "fixing" something that isn't broken.

- **Merging tools degrades both.** I built a proof-verification engine and tried merging it into AFM. A single agent can't be both prosecutor and defense attorney — confirmation bias from its own reasoning contaminated the proofs. Separate tools, separate context windows, better results.

Full story with all 10 failures: [`docs/origin-story.md`](docs/origin-story.md)

## The approach

AFM is a 90-line prompt protocol for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). It changes how the AI *reasons about* your code — not what tools it uses.

### Two atomic operations, infinite combination

Everything reduces to two operations:

**REVERSE**: Look at code → trace back to the hidden assumption behind it.
*"Why did the developer write it this way? What must be true for this to work?"*

**FORWARD**: Take an assumption → predict what happens if it fails → verify in code.
*"If this assumption breaks, what goes wrong? Let me check."*

These chain freely. No phases, no steps, no predefined sequence:

```
Observe something suspicious
    → REVERSE: what assumption is behind this?
        → Found assumption A
            → FORWARD: if A fails, what breaks?
                → Found broken behavior B
                    → REVERSE: what assumption caused B?
                        → Found deeper assumption C
                            → FORWARD: if C holds, what else depends on it?
                                → ...
```

Each result feeds back as a new lead. The board grows until it's clear.

### The WHY mechanism

At every confirmed vulnerability, AFM forces a question most reviewers skip:

> **"WHY did the developer assume this?"**

Not "what's wrong" — that's surface. But *why did a reasonable person believe this was correct?* What evidence created the false confidence? And critically: **where else in the codebase does this same pattern of false confidence appear?**

This turns point findings into pattern discoveries.

### Three verdicts

- **VULNERABLE** — assumption proven false, impact traced. Go deeper + ask WHY.
- **HOLDS** — defense exists and works. But challenge it: what evidence would prove this defense *doesn't* actually protect?
- **UNCLEAR** — not enough evidence yet. Come back later.

### Philosophical roots

The interrogation method came from mapping philosophy's three ultimate questions to code:

- **Where do I come from → Premise**: What conditions must be true for this code to exist?
- **Who am I → Mechanism**: What does this code actually do?
- **Where am I going → Expected Effect**: What does this code promise to the outside world?

Any inconsistency between two layers is itself a finding. A human reviewer naturally thinks this way — I tried to teach the AI the same pattern.

## The vision (and the compromise)

The full vision of AFM is more ambitious than what's implemented.

**What I imagined:** At the very first assumption, REVERSE and FORWARD fire *simultaneously*. Each result branches into "assumption holds" and "assumption fails." Four paths open at once. Each path branches again. The tree grows in parallel — and it's not even a flat tree, because branches from different paths can cross-influence each other. It's closer to a three-dimensional structure than a two-dimensional tree.

**What I built:** A single agent working through a lead board, picking the hottest lead, processing it, putting results back. Essentially depth-first search. Half-linear.

**Why the gap:** A single context window can't truly run parallel branches. The board-based approach is a practical compromise that preserves the core logic (REVERSE/FORWARD freely chaining, every result becomes a new lead) while working within current limitations.

I believe the parallel branching version would be significantly more powerful. But it's too large for me to build right now. If this interests you, I'd love to hear your ideas.

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

AFM works well standalone, but can also be combined with other tools — standard code review for breadth, AFM for depth, or used as a second pass after your regular review process.

## Does it work?

Honest answer: I think so, but the data has limitations.

I ran AFM and standard code review on the same 4 repos (TS, C, Rust, Zig), same model (Claude Opus), empty prompt, no guidance. The only variable was whether AFM was loaded.

**CR found more things. AFM found different things.** About 60% of findings don't overlap. They complement each other rather than compete.

Caveats I want to be upfront about:
- Each repo was tested once (N=1). LLM outputs have high variance. These numbers could shift on a rerun.
- I judged the overlap myself. No third-party blind evaluation.
- The one external validation: a project maintainer confirmed and fixed 3/3 findings I reported. Small sample, but it's real.

<details>
<summary>Benchmark data (4 repos, 4 languages)</summary>

### Round 3 — clean benchmark (2026-03-26)

| Repo | Language | Lines | CR findings | AFM VULNERABLE | AFM HOLDS |
|------|----------|-------|-------------|----------------|-----------|
| 2fauth-worker | TypeScript | 18K | 24 | 14 | 4 |
| picolm | C | 2.5K | 21 | 8 | 8 |
| OpenBSE | Rust | 15K | 21 | 7 | 9 |
| zigpty | Zig+TS | 3.7K | 25 | 13 | 44 |

Token cost: AFM uses ~35% more tokens than standard CR (126K vs 93K per repo average).

### Broader experiment history

AFM was developed through 100+ experiment runs across 20+ repositories in 9 languages. This included controlled variable experiments, cross-method validation against 3 other approaches, and cross-language generalization tests.

One full round of data (12 runs) was invalidated and discarded after discovering prompt-guidance contamination. I mention this because honesty about failed experiments matters more than impressive numbers.

Full reports in [`benchmarks/`](benchmarks/).

</details>

## Example: what AFM output looks like

Standard CR says:
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

The difference: AFM doesn't just find the bug. It finds *why the bug exists*, *what assumption caused it*, and *where else that assumption appears*.

## Limitations

**I'm not a security researcher.** I built this because I was frustrated with AI patching symptoms. The findings have been cross-validated across 4 methods with 86% agreement rate, but they have NOT been manually confirmed by domain experts. Some may be hallucinations.

**N=1 benchmarks.** Each repo was tested once per method. LLM outputs are stochastic. I don't have variance data yet. Treat the numbers as directional, not precise.

**If you find a false positive, please open an issue.** That data is the most valuable contribution you can make. Tell me what AFM flagged and why it was wrong. If you find a true positive, that's valuable too.

**Works best on assumption-dense code.** Cross-boundary logic (auth, crypto, FFI, cross-thread, cross-platform) produces more findings. Simple CRUD apps won't see much benefit.

**The parallel branching vision is unimplemented.** Current version is a sequential board-walker. The full tree-shaped simultaneous exploration remains a dream.

**Single-model testing.** All benchmarks use Claude Opus. Behavior on other models is unknown.

**Costs more tokens.** ~35% more than standard code review.

**I'm learning.** This is a beginner's experiment in prompt engineering for code reasoning. I don't claim expertise. If you see something I'm doing wrong, tell me.

## Beyond code review

The assumption-failure pattern isn't limited to code. I've used it experimentally for business decisions, architecture reviews, and competitive analysis — asking "what are we assuming that might not be true?" These are not benchmarked. Let me know if you're interested.

## Contributing

- **Report false positives** — the single most valuable contribution
- **Report true positives** — confirms the approach works
- **Try it on your codebase** — different languages/domains produce different results
- **Ideas for parallel branching** — if you have thoughts on implementing the full tree-shaped exploration, I'd love to hear them

## License

[MIT](LICENSE)
