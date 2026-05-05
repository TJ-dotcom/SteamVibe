# ADR-003: OpenAI Whisper (Local GPU) over Cloud ASR APIs

**Date:** January 2026  
**Status:** Accepted  
**Context:** Audio transcription for live stream speech-to-text

---

## Context

StreamVibe needs real-time speech-to-text on a live Twitch stream audio feed. The target latency is under 200ms from audio capture to transcription output. The system runs on a local GPU (RTX 4060, 8GB VRAM) in prototype and on EC2 g4dn.xlarge (NVIDIA T4) in AWS deployment.

Alternatives considered: AWS Transcribe Streaming, Google Speech-to-Text, AssemblyAI, NVIDIA Parakeet (TDT via Riva/ONNX).

## Decision

**OpenAI Whisper running locally via GPU inference**, with ONNX/INT8 quantization for latency optimization.

## Rationale

**Latency.** Cloud ASR APIs introduce network round-trip latency on every audio chunk. At 200ms target, a cloud call adds 50–150ms of variable network overhead before any inference begins. Local GPU inference keeps the entire transcription pipeline on-device.

**Cost at scale.** AWS Transcribe charges per second of transcribed audio. At 3 concurrent streams running continuously, that cost compounds quickly. A one-time GPU EC2 instance (g4dn.xlarge, ~$379/mo) covers unlimited transcription volume.

**VRAM budget.** Whisper `base` or `small` models fit comfortably in the RTX 4060's 8GB VRAM alongside the CNN sentiment model. The `RTX4060ResourceGovernor` class in the codebase manages VRAM explicitly:

```python
self.max_vram_gb = 6.5  # Leave 1.5GB for system/other models
self.warning_threshold = 0.8  # 80% VRAM usage warning
```

**NVIDIA Parakeet was the original target** (TDT architecture via Riva/ONNX for <200ms). Whisper was used as the production fallback because Parakeet requires NVIDIA Riva setup that added deployment complexity to the prototype. The architecture remains designed to swap in Parakeet — the `parakeet_audio_engine.py` module exists as the intended upgrade path.

**INT8 quantization.** Whisper inference is quantized to INT8 via ONNX Runtime, reducing VRAM usage and improving throughput without significant accuracy loss for conversational speech.

## Audio Pipeline

```
Twitch stream → Streamlink → FFmpeg → WebRTC VAD → 
  [voice frames only] → Whisper INT8 → timestamped transcription segments → Kafka
```

WebRTC VAD (Voice Activity Detection) gates the Whisper calls — only frames with detected speech are sent for transcription, reducing GPU load by ~40-60% during silent/music periods.

## Trade-offs Accepted

- Whisper has higher latency than Parakeet TDT for the same hardware — the <200ms target is met on GPU but tight
- Requires GPU hardware (no CPU fallback in production)
- VRAM must be managed explicitly — CUDA memory cleanup (`torch.cuda.empty_cache()`) required after each inference batch to prevent memory leaks (this was a critical bug found and fixed during testing)

## VRAM Bug Fixed

The testing report identified a VRAM memory leak in early iterations: GPU memory was not cleaned up between Whisper inference calls, causing OOM crashes after ~30 minutes. Fixed by adding explicit cleanup:

```python
torch.cuda.empty_cache()
gc.collect()
```

## Consequences

All audio transcription runs on-device GPU. EC2 g4dn.xlarge becomes a hard infrastructure requirement for AWS deployment. The `LiveAudioTranscriptionEngine` is a Singleton (see ADR-005) to ensure only one Whisper model is loaded at any time.
