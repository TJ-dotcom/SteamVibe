# ADR-005: Singleton Pattern for All Heavy ML Components

**Date:** January 2026  
**Status:** Accepted  
**Context:** Memory and GPU resource management across a multi-threaded pipeline

---

## Context

StreamVibe's processing pipeline is multi-threaded by design. The main orchestrator (`main.py`) runs chat ingestion, audio processing, sentiment analysis, and viral detection concurrently. Without explicit control, each thread or Streamlit refresh cycle could instantiate its own copy of heavy ML components.

The problem observed during prototype development: Streamlit's execution model re-runs the entire script on each UI interaction. Without Singletons, this caused:

- Multiple Whisper model loads → VRAM exhaustion (OOM crash after ~3 refreshes)
- Multiple CNN instances → RAM exhaustion  
- Multiple IRC client connections → duplicate messages flooding the queue

## Decision

**`SingletonMeta` metaclass applied to all resource-intensive components.**

## Implementation

```python
class SingletonMeta(type):
    """Thread-safe Singleton metaclass with blocking protection"""
    _instances: Dict[Type, Any] = {}
    _lock: threading.Lock = threading.Lock()
    
    def __call__(cls, *args, **kwargs):
        with cls._lock:
            if cls not in cls._instances:
                instance = super().__call__(*args, **kwargs)
                cls._instances[cls] = instance
        return cls._instances[cls]
```

**Components using Singleton:**

| Class | Resource Protected |
|---|---|
| `INT8QuantizedSentimentAnalyzer` | CNN model + ONNX runtime session |
| `LiveAudioTranscriptionEngine` | Whisper model + audio buffer + VAD state |
| `ViralMomentDetector` | Detection state + sliding window history |

## Why Metaclass, Not Module-Level Global

A module-level `_instance = None` pattern is simpler but not thread-safe. In a multi-threaded pipeline, two threads can both read `_instance is None` as True simultaneously and both proceed to instantiate — defeating the purpose entirely.

`SingletonMeta` uses a class-level lock (`threading.Lock`) to serialize instantiation. Only the first thread through creates the instance; all subsequent threads block and then receive the already-created instance.

## Factory Pattern Complement

Singletons alone create a configuration problem — if the constructor takes arguments, which thread's arguments win? This is solved with the **Factory pattern**:

```python
# Factory handles configuration; Singleton handles lifecycle
analyzer = StreamVibeFactory.create_component('sentiment_analyzer', {
    'enable_quantization': True,
    'batch_size': 16
})
```

The Factory sets configuration on first instantiation. Subsequent calls via the Factory return the Singleton with its original configuration — arguments on subsequent calls are silently ignored.

## Measured Impact

From the architecture summary:
- **Memory:** ~75% reduction — no duplicate ML model loading
- **Stability:** Zero OOM crashes after implementing Singleton on audio engine
- **Correctness:** No duplicate IRC messages after Singleton on `TwitchIRCClient`

## Trade-offs Accepted

- Testing is harder — unit tests must explicitly reset Singleton state between test cases
- Configuration is immutable after first instantiation — runtime reconfiguration requires process restart
- Circular import risk if Singleton classes import each other (mitigated with lazy imports throughout the codebase)

## Consequences

No ML model is ever loaded more than once per process lifetime. The `streamvibe_patterns.py` module is the single source of truth for `SingletonMeta`. Any new heavy component added to the system must use `SingletonMeta` as its metaclass.
