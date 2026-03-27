# How AFM was built

This is the honest story behind a 90-line prompt. 8 days, 47 iterations, 100+ experiment runs, 10 times the methodology proved itself wrong. Every wrong turn is included.

## Day 0: The frustration (March 20, 2026)

I'm an indie developer. I write code with AI — games, apps, automation. I'm not a security researcher, not a computer scientist, not a methodologist.

The problem was simple: AI-generated code kept breaking in ways that AI couldn't fix. Tests all green, behavior wrong. Fix one thing, break another. The patches piled up but the root cause never got touched. After watching this cycle repeat enough times, I asked a question:

> "Why does AI keep patching at the same layer instead of stepping back to check whether the assumption behind the code is even correct?"

I didn't research the theory first. I just built something: assume the code is wrong → trace back to the upstream assumption → fix the root cause instead of the symptom.

First run on a real project: 396 tests passing, 13 bugs found. The tests and the code had encoded the same wrong assumption — the tests were protecting incorrect behavior.

## Day 0, continued: From debugging to creation

If you can find bugs by breaking assumptions, can you find opportunities the same way?

Analyzing a competitor's product: don't copy what they built. Ask what they *assumed*. Is that assumption true? If not, what grows in the crack?

This led to what I call "bidirectional assumption exploration":
- **If the assumption fails** → trace the damage, find the root
- **If the assumption holds** → push forward: what else depends on this? What new assumptions does it create?

A holds, so B follows. B assumes C. Test C. C fails — new finding. C holds — push to D...

The tree grows in both directions. Debug goes down. Creation goes up and out.

## The philosophical foundation

The core interrogation method came from an unexpected place — philosophy's three ultimate questions:

> **Where do I come from? / Who am I? / Where am I going?**

Mapped to code:
- **Where do I come from (Why) → Premise**: What conditions must be true for this code to exist?
- **Who am I (What) → Mechanism**: What does this code actually do, step by step?
- **Where am I going (Purpose) → Expected Effect**: What does this code promise to the outside world?

A human reviewer naturally thinks this way. They don't just scan for syntax errors — they ask "why was this written like this?" and "does what it does match what it promises?"

I tested this with a concrete example: a function that claims to handle UTF-8 but only processes ASCII. If you only check Purpose, the AI might hallucinate a fix for "UTF-8 handling." But if you check all three layers simultaneously — Purpose says UTF-8, Mechanism processes single bytes, Premise assumes ASCII input — the contradiction is immediate.

Three layers cross-checking each other. Any inconsistency between two layers is itself a finding.

## The two atomic operations

Everything in AFM reduces to two operations:

**REVERSE**: From an observation in code, trace backward to the hidden assumption.

**FORWARD**: From an assumption, predict what happens if it fails (or holds), then verify in code.

That's it. No phases. No steps. No predefined workflow. Just REVERSE and FORWARD, chaining freely, driven by a lead board that grows as you work.

This design principle — **describe atomic operations + loop conditions, let combinations emerge naturally** — was the single most important decision. Every time I tried adding structure (phases, steps, categories), results got worse. Every time I removed structure, results got better.

### The vision: parallel branching

What I actually imagined was more ambitious. At the very first assumption, REVERSE and FORWARD should fire simultaneously. Each result splits into "holds" and "fails." Four branches, immediately. Each branch splits again.

And it's not a flat tree — branches from different paths should be able to influence each other. If branch A discovers that a shared library has a bug, branch C (which assumed that library works) needs to know. The real structure is three-dimensional, not planar.

What I built is a compromise: a single agent walking a lead board, depth-first. Half-linear. It works, but it's not the full vision. The parallel version would need multi-agent coordination that doesn't exist yet in a way I can build. I'm honest about this gap.

## The experiments (all of them)

### Round 1: First real project (March 20)
First real test on a personal project. 396 green tests, 13 bugs. The concept works.

### Round 2: Scrapling audit (March 22)
20 findings, 6 root causes. But 15% were false positives — design choices flagged as bugs. This led to a key insight: **"Don't restrict the front end. Add a review filter at the back end."** Generate findings aggressively, publish findings conservatively.

8 post-hoc review questions were designed to filter false positives. After applying them: 15% → 0%.

### Round 3: The first collapse (March 24)

Three runs on PanSou. Empty prompt: 160 seconds, 18 findings. My 200-line AFM protocol: 431 seconds, 9 findings.

**The blank prompt won.** The methodology was actively making things worse.

Four-way expert diagnosis: the engine had been trapped in rigid phases, format overhead was eating 50%+ of attention, scope was locked too tight. One expert said: "The methodology is dead. Retire it."

### Round 4: Rebirth — "assumptions are leads, not rules" (March 24)

Instead of retiring, I stripped everything. The key insight:

> "I kept building rules when I should have been building leads. Assumptions aren't rules to enforce — they're leads to follow."

6 skills merged into 1. 161 lines. No phases, no format requirements, no scope constraints. The lead board concept emerged — not a checklist to complete, but a board where the hottest lead gets picked next.

### Round 5: External confirmation (March 24)

Tirith project maintainer confirmed all 3 reported findings and fixed them in v0.2.9. First time someone outside the project said "yes, these are real bugs."

### Round 6: Darwin's tournament (March 24)

9 variants tested on Dagu. Key discovery: anti-hallucination guards were cages, not guardrails. The model's real failure mode is under-assertion (being too cautious), not hallucination. Guards were fighting the wrong enemy.

Strike was born here — an attack-driven variant that became its own tool.

### Round 7: The terminology crisis (March 25)

> "If you yourself get confused by the terms, how will the agent execute them?"

The word "Break" was supposed to mean "test this assumption." But in everyday English, "break" means "stop" or "destroy." The model was interpreting "Break this assumption" as "find ways to attack it" — biasing toward offense, missing defense.

Three rewrites. Break/Bounce → VULNERABLE/DEFENDED → VULNERABLE/HOLDS. "HOLDS" came from AFM's own philosophical framework — when an assumption holds, it's not "case closed," it's "new question: what does this defense depend on?"

### Round 8: The triple recursion (March 25)

I discovered that AFM's seed examples were all security patterns. The model was treating AFM as a security tool.

**AFM committed the exact error it was designed to detect.** "A check exists" created the illusion of coverage — the model saw security examples and assumed all dimensions were covered. This is the same "check coverage illusion" pattern AFM finds in code.

And when I tried to fix it by adding 10-dimensional examples? That created a *checklist illusion* — the model checked each example's dimension once and moved on.

Triple recursion: AFM found the pattern → AFM had the pattern → the fix for the pattern had the pattern.

The real fix: remove ALL seed examples. Zero examples produced the best results. Counterintuitive, but the data was clear.

### Round 9: Data invalidation (March 25)

Reviewing 12 benchmark runs, I discovered the prompt had included directional guidance. The data was contaminated.

I threw away the entire round. 12 runs, days of work, discarded.

5 new runs were designed as controlled experiments — isolating prompt influence, seed example bias, and sub-agent effects one variable at a time. This is when I learned that seed examples are the single strongest source of bias, stronger than prompt wording or execution architecture.

### Round 10: v3.1 crystallizes (March 25)

v3.0 → v3.1: deleted all seed examples, removed security-focused language. Tested on aide (VS Code AI plugin): 81.8% of findings were non-security. The skill automatically adapts to whatever the code actually needs.

### Round 11: Clean benchmark (March 26)

4 repos × 3 methods × empty prompts = 12 runs. First fully clean dataset.

Standard CR averaged 22.8 findings (remarkably stable, σ=1.9). AFM v3.1 averaged 10.5 VULNERABLE + 16.3 HOLDS. About 60% of findings were unique to one method.

### Round 12: Falsify — the proof engine (March 27)

Built a separate tool that tries to construct concrete proofs for each finding using an adversary sub-agent. Tested across 4 repos, 4 languages.

Key discovery: 21 findings were "proven" with concrete failure demonstrations. But when I cross-validated against other methods, **only 3 of 21 (14%) were truly unique.** The other 86% had been found by at least one other method.

This was my 9th assumption failure: Falsify's real value isn't finding unique bugs — it's converting suspicions into evidence and filtering noise.

### Round 13: The merger experiment (March 27)

Tried merging the proof engine into AFM v3.1 to save tokens. Created v3.2-exp, ran a controlled comparison.

Result: the merged version produced a false proof (ignoring an existing guard), missed 4 findings v3.1 caught, and lost most of its defense map.

A single agent can't be both prosecutor and defense attorney. The confirmation bias from its own reasoning chain contaminated the proof attempts. Role separation requires separate context windows.

10th assumption failure. **Combination beats merger.** The experiment was deleted.

## The 10 assumption failures

The methodology was built by applying its own principle to itself:

1. **"Rigid phases help"** → Blank prompt beat 200-line protocol (Day 4)
2. **"Security focus is the right framing"** → 82% of findings were non-security (Day 5)
3. **"Radiation search finds unique things"** → ~70% reproducible with simple prompts (Day 5)
4. **"English words mean what I intend"** → Break/Hold/Clear all had wrong connotations (Day 5)
5. **"Seed examples help the model"** → They're the strongest bias source (Day 5)
6. **"AFM is a security tool"** → The AI assumed this, not me. AFM's own seeds encoded the bias (Day 5)
7. **"My experiment data is clean"** → Prompt guidance contaminated 12 runs (Day 5)
8. **"Guards prevent hallucination"** → Simple citation rules beat complex guards (Day 5)
9. **"Proof verification finds unique bugs"** → Only 14% unique; real value is proof quality (Day 7)
10. **"Merging tools is more efficient"** → Single agent can't do adversarial self-verification (Day 7)

Day 5 was brutal. Five assumptions fell in a single day. Each one forced a redesign before I could move to the next.

## What I wanted but couldn't build

1. **True parallel branching**: Every assumption simultaneously explored in both directions. Not depth-first board-walking but genuine parallel tree growth. I believe this would be significantly better. I can't build it with current single-agent architecture.

2. **Three-dimensional assumption graphs**: Branches that cross-reference each other. If one branch discovers a shared dependency is broken, all branches depending on it update. Not a tree but a graph — in 3D, not 2D.

3. **Multi-agent pipeline**: Agent 1 seeds assumptions → Agent 2 attacks each one → Agent 3 synthesizes and finds cross-chain patterns. Falsify partially implemented the first two steps, but the full pipeline remains unbuilt.

4. **Community seed checklists**: The engine handles depth. Community contributors provide breadth by maintaining per-language, per-framework seed lists (common assumption patterns in React, common traps in Rust unsafe, etc.). Designed but not implemented.

5. **Statistical significance**: All data is N=1. No variance analysis. I know this is a weakness.

## Current state

v3.1 is stable. 90 lines. Tested on 4 repos across 4 languages with clean data. The core works: two atomic operations, a lead board, WHY questioning at every vulnerability, HOLDS that map defenses.

The full internal development log (47 steps, 833 lines, in Chinese) is the raw record of every decision, debate, and mistake. Available on request.

## What I'd like from you

I'm not an expert. I'm an indie developer trying to make code review work better through honest experimentation. If you:

- Find a false positive → tell me, it's the most valuable data
- Find a true positive → tell me, it confirms the approach
- Have ideas about parallel branching → I'd love to hear them
- Can verify findings on repos I tested → I literally can't do this myself
- Think I'm wrong about something → I'd rather know now

This is a seed, not a harvest. Help me grow it.
