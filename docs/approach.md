# The Approach

AFM is a 90-line prompt protocol for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). It changes how the AI *reasons about* your code — not what tools it uses.

## Two atomic operations, infinite combination

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

## The WHY mechanism

At every confirmed vulnerability, AFM forces a question most reviewers skip:

> **"WHY did the developer assume this?"**

Not "what's wrong" — that's surface. But *why did a reasonable person believe this was correct?* What evidence created the false confidence? And critically: **where else in the codebase does this same pattern of false confidence appear?**

This turns point findings into pattern discoveries.

## Three verdicts

- **VULNERABLE** — assumption proven false, impact traced. Go deeper + ask WHY.
- **HOLDS** — defense exists and works. But challenge it: what evidence would prove this defense *doesn't* actually protect?
- **UNCLEAR** — not enough evidence yet. Come back later.

## Philosophical roots

The interrogation method came from mapping philosophy's three ultimate questions to code:

- **Where do I come from → Premise**: What conditions must be true for this code to exist?
- **Who am I → Mechanism**: What does this code actually do?
- **Where am I going → Expected Effect**: What does this code promise to the outside world?

Any inconsistency between two layers is itself a finding. A human reviewer naturally thinks this way — I tried to teach the AI the same pattern.
