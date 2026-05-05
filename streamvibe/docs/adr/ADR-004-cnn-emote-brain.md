# ADR-004: CNN + Twitch Lexica over Standard NLP (VADER/BERT) for Sentiment

**Date:** January 2026  
**Status:** Accepted  
**Context:** Chat sentiment analysis for a Twitch-native audience

---

## Context

Standard NLP sentiment analysis (VADER, TextBlob, BERT) is trained on natural language text — product reviews, news, social media posts. Twitch chat is not natural language. A typical message during a viral moment looks like:

```
KEKW KEKW KEKW Pog OMEGALUL ResidentSleeper 4Head
```

VADER scores this as neutral. The actual sentiment is highly positive with humor/hype tone. The gap between what standard NLP sees and what the chat is actually expressing is the core problem StreamVibe's Brain component solves.

## Decision

**CNN-based classifier trained on Twitch-specific emote lexica**, specifically the `emote-controlled` classifier integrated as `emote_brain_assets`.

## The Brain Architecture

The classifier has three layers:

1. **Tokenizer** (`TwitchTokenizer`) — handles emotes, slang, and Twitch-specific notation that standard tokenizers break on

2. **Emote Lexica** — two lookup tables:
   - `emote_average.tsv` / `emote_distribution.tsv` — Twitch-specific emote sentiment weights (e.g., `KEKW: 1.5`, `biblethump: -0.7`, `ResidentSleeper: -1.0`)
   - `vader_average.tsv` / `vader_distribution.tsv` — VADER scores for natural language fallback
   - `emoji_average.tsv` / `emoji_distribution.tsv` — Unicode emoji weights

3. **SentenceCNN** — convolutional neural network that processes tokenized message sequences and outputs a sentiment vector: `{positive, negative, neutral}`

## Why CNN over BERT/Transformers

**Speed.** CNN inference is significantly faster than transformer inference for short sequences. Twitch chat messages average 3–8 tokens. A CNN processes these in microseconds on CPU; BERT would require GPU and adds latency the pipeline can't absorb.

**Training data fit.** The CNN was trained on labeled Twitch chat data (`labeled_dataset.csv`, 200K+ rows). A BERT model fine-tuned on generic social media data performs worse on Twitch-specific emote patterns than a smaller CNN trained on the right domain.

**VRAM budget.** The CNN fits in CPU RAM entirely. Keeping sentiment inference off the GPU leaves full VRAM available for Whisper ASR — the more latency-critical component.

## INT8 Quantization

The `INT8QuantizedSentimentAnalyzer` wraps the CNN with ONNX Runtime INT8 quantization:

```python
# ~75% memory reduction, minimal accuracy loss for this task
enable_quantization=True
batch_size=16  # Process chat in micro-batches
```

## Zero Mock Rule

The codebase enforces a "ZERO-MOCK RULE" — if the real CNN model files aren't available, the system refuses to start rather than falling back to simulated sentiment scores:

```python
if not EMOTE_BRAIN_AVAILABLE:
    logger.critical("ZERO-MOCK RULE: Cannot proceed without real CNN & lexica")
    raise SystemExit(1)
```

This is intentional. A viral moment detector running on fake sentiment scores produces worse outcomes than no detector at all.

## Trade-offs Accepted

- The CNN model requires the `emote_brain_assets/` directory with trained weights — deployment is not zero-dependency
- Accuracy is bounded by Twitch lexica coverage — new emotes (e.g., post-training viral emotes) degrade performance until lexica are updated
- Not multilingual — Twitch channels with non-English chat are out of scope for this prototype

## Consequences

The `INT8QuantizedSentimentAnalyzer` is the sole sentiment provider. It is a Singleton (see ADR-005). Its output is a `SentimentVector` dataclass with `{sentiment_scores, emote_weights, confidence, processing_time_ms}` — structured output consumed by the `ViralMomentDetector`.
