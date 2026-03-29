# The Vision (and the Compromise)

The full vision of AFM is more ambitious than what's implemented.

## What I imagined

At the very first assumption, REVERSE and FORWARD fire *simultaneously*. Each result branches into "assumption holds" and "assumption fails." Four paths open at once. Each path branches again. The tree grows in parallel — and it's not even a flat tree, because branches from different paths can cross-influence each other. It's closer to a three-dimensional structure than a two-dimensional tree.

## What I built

A single agent working through a lead board, picking the hottest lead, processing it, putting results back. Essentially depth-first search. Half-linear.

## Why the gap

A single context window can't truly run parallel branches. The board-based approach is a practical compromise that preserves the core logic (REVERSE/FORWARD freely chaining, every result becomes a new lead) while working within current limitations.

I believe the parallel branching version would be significantly more powerful. But it's too large for me to build right now. If this interests you, I'd love to hear your ideas.
