# Cycle detection — superseded

This sketch has been promoted to a real memo: [`../path-emergence/cycle-detection.md`](../path-emergence/cycle-detection.md) ("Flood safety: termination, deduplication, and loop-checking"). Its three original bullets are folded into that memo:

- "remember packet content / packet ID" → remember-and-drop on the discovery ID `D` (§2, termination) and destination dedup on `(author, seq)`.
- "detect cycle in a hop-by-hop log" → loop-checking the recovered node-ID sequence across a multi-segment route (§3); a single-flood route cannot loop, so the check bites only where a cached suffix segment joins a fresh prefix.

The hop bound `H_max` is committed there (§1); the rest is that memo's stated agenda.
