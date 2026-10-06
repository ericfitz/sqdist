# ADR 0001: Defer rayon parallelism (performance plan Tier 4)

- Status: Accepted (deferred, not rejected)
- Date: 2026-08-29 (recorded 2026-10-06)
- Decision maker: Eric Fitzgerald (human decision)

## Context

Tiers 1–2 of [docs/performance-plan.md](../performance-plan.md), plus the banded Damerau emit gate
and the prepared query, made thresholded list mode about 15.5x faster than 0.5.0, with
byte-identical output (shipped in 0.6.0). Tier 4 would parallelize scoring over candidates with
rayon. The work is embarrassingly parallel, and it is the last big multiplier left for the
top-15k × full-PyPI sweep.

## Decision

Eric deferred Tier 4 as a "future potential improvement". It was not rejected.

## Consequences

- sqdist stays single-threaded and keeps one runtime dependency (`unicode-security`).
- Adding rayon is a design decision, not a refactor: it adds a second runtime dependency. Revisit
  it here before implementing.
- Non-ASCII corpus synthesis (performance plan §5.4) should come before any ASCII fast path in the
  DP kernel, whether or not Tier 4 goes ahead.
