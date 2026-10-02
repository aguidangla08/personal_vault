# OneSpin Witness Generation — Notes

## The problem

Trying to calculate a witness for an assertion in OneSpin 360, with no apparent way to stop the search if it doesn't find one — the run just never finishes.

## Why it hangs

Witness generation is a search for a trace that satisfies some condition (a violation, a cover point, an antecedent firing). "Not found yet" and "doesn't exist" look identical to the engine until something external tells it to stop. Unlike a full proof, which can terminate by exhausting the state space, a witness search has no natural stopping point on its own — so if no such trace actually exists, it can run indefinitely.

## Practical ways to bound it

- **Time limit** — most prove/witness/exercise commands support a wall-clock limit so the run reports "undetermined" after N seconds instead of running forever. Check `help <command>` in the Tcl shell for the exact option on your version.
- **Trace-length / depth bound** — caps how deep the search goes before giving up, useful if you have a sense of how long a witness "should" take.
- **Background + abort** — run the check asynchronously, poll status, and abort manually once you've waited long enough, rather than blocking with no way out short of killing the process.
- **Engine selection** — sometimes one engine in the portfolio is the one that never converges; restricting/reordering engines for that property can turn a hang into a fast (even if inconclusive) result.
- **Ctrl-C** as a blunt fallback — safe to interrupt and treat as a timeout.

## Witness vs. counterexample vs. status

Property status is typically one of: **HOLD** (proven true), **FAIL** (violated), or **UNDETERMINED** (engines couldn't decide within their limits).

- **FAIL** — the falsifying engine produces the failing trace (the counterexample) automatically as part of disproving the property. This _is_ the witness to the failure; no separate witness step is needed.
- **HOLD** — the property is already proven for all reachable states. A witness request here serves a different purpose: a **non-vacuity check** — finding one concrete trace where the property's antecedent actually fires, to confirm the proof isn't trivially true because the precondition is unreachable.
- **UNDETERMINED** — no witness or counterexample exists yet, because the tool hasn't decided either way. Asking for a witness here isn't a well-formed question — there's no concrete trace to converge on — which is why it can hang indefinitely.

Cover goals are the other place witness is the primary artifact (independent of hold/fail): the witness shows the cover point is reachable.

## The specific plan that ran into trouble

The idea was to generate a witness for UNDETERMINED assertions too, so that cycles which haven't failed yet could be read as "holding for now."

That doesn't quite work, because "holds up to some depth" isn't a witness — a witness demonstrates something _did_ happen, and for an UNDETERMINED assertion nothing has happened yet for a trace to exhibit. There's no target for the engine to search toward, so it just runs.

**What actually answers that question** is the bound/depth the proving engines already reached while attempting (and failing) to complete the proof — e.g. "BMC explored up to cycle N with no counterexample." That number is typically already reported alongside the UNDETERMINED status itself (property report / `report_proof`-style output), not something you need a separate witness run to produce. It gives "holds through cycle N" directly, terminates immediately since it's already computed, and is the correct way to read an UNDETERMINED result — rather than triggering a witness search on it.