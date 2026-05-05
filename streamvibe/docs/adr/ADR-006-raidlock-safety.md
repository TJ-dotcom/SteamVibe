# ADR-006: RaidLock as a Non-Bypassable Hardware Safety Gate

**Date:** January 2026  
**Status:** Accepted  
**Context:** Brand safety for automated raid recommendations

---

## Context

StreamVibe includes a raid recommendation engine — it can suggest which streamer a host should raid next based on vibe similarity (cosine similarity between sentiment vectors). This is a product feature with direct user impact.

The risk: an automated raid recommendation fires at the wrong moment. The streamer is visibly tilted (physical rage, slamming desk, yelling), or the chat just witnessed something shocking (stunned silence pattern). Raiding another streamer in this state reflects poorly on the host and can harm the target streamer's community.

The safety requirement: **the system must never recommend a raid when unsafe conditions are active, regardless of how confident the vibe-match algorithm is.**

## Decision

**`RaidSafetyLock` as a hard gate — not a soft warning, not a confidence penalty, a full block.**

## Unsafe Conditions Detected

| Condition | Signal | Logic |
|---|---|---|
| `PHYSICAL_RAGE` | Audio energy above threshold | Streamer vocal intensity spike — likely tilting/angry |
| `STUNNED_SILENCE` | Chat MPS > 50 AND audio energy ≈ 0 | Chat is flooding but streamer has gone silent — shocked/stunned |
| `TOXIC_CLUSTER` | Sustained negative sentiment vector | Chat toxicity above threshold for >60 seconds |

## Implementation

```python
class RaidSafetyLock:
    """Thread-safe atomic safety status management"""
    
    def __init__(self):
        self._lock = threading.RLock()  # Reentrant lock
        self._unsafe_conditions: Set[str] = set()
    
    def is_safe_to_raid(self) -> bool:
        with self._lock:
            return len(self._unsafe_conditions) == 0
    
    def set_unsafe(self, condition: str):
        with self._lock:
            self._unsafe_conditions.add(condition)
    
    def clear_condition(self, condition: str):
        with self._lock:
            self._unsafe_conditions.discard(condition)
```

**Why `threading.RLock()` specifically:** The reentrant lock allows the same thread to acquire the lock multiple times without deadlocking — important because safety check and condition update can happen from the same callback chain in the async event loop.

## The Race Condition Bug Fixed

The testing report documented a real bug: non-atomic safety status updates could produce inconsistent state under concurrent load. Specifically:

1. Audio thread reads `is_safe = True`
2. Chat thread writes `TOXIC_CLUSTER` condition
3. Audio thread proceeds with raid recommendation (reads stale state)

Fixed with `RLock` — all reads and writes are atomic. The safety state cannot be observed in a partially-updated condition.

## Why a Hard Gate, Not a Soft Penalty

An alternative design would reduce the raid recommendation confidence score when unsafe conditions are present (e.g., multiply confidence by 0.1). This was rejected because:

- A very high vibe-match score (0.98) multiplied by a penalty factor (0.1) still produces a recommendation (0.098 vs threshold 0.05)
- Soft penalties create edge cases that are hard to reason about and test
- Brand safety failures are asymmetric — one bad automated raid is more damaging than a thousand missed opportunities

The hard gate is binary, auditable, and testable with a simple assertion: if any unsafe condition is set, `is_safe_to_raid()` returns False, always.

## Trade-offs Accepted

- False positives — the gate may block a raid recommendation during a valid high-energy moment (streamer excited, not angry). Acceptable: the cost of a missed raid is low.
- No user override — the gate cannot be bypassed by the streamer in the current implementation. Intentional for prototype safety.

## Consequences

`RaidSafetyLock` is checked as the first gate in the raid recommendation pipeline. No vibe similarity computation, no target selection, no UI update fires unless `is_safe_to_raid()` returns True. The lock state is also surfaced on the Streamlit dashboard so the streamer can see why recommendations are suppressed.
