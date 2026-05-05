# ADR-007: FFmpeg Pipeline over Direct Streamlink Audio for Whisper Input

**Date:** January 2026  
**Status:** Accepted  
**Context:** Audio extraction and preprocessing before Whisper ASR

---

## Context

The audio path starts with a live Twitch stream URL and needs to deliver clean 16kHz mono PCM audio frames to Whisper ASR with minimal latency. Two approaches were evaluated:

1. Use Streamlink directly to capture stream → pass raw stream to Whisper
2. Use Streamlink for stream capture → pipe through FFmpeg → deliver preprocessed frames to Whisper

## Decision

**Streamlink → FFmpeg → WebRTC VAD → Whisper.** FFmpeg is an explicit stage in the pipeline, not a passthrough.

## Rationale

**CPU reduction.** FFmpeg handles audio decoding, resampling, and format conversion in a highly optimized native binary. Processing the same operations in Python (via librosa or scipy) is measurably slower. The architecture documentation notes a ~60% CPU reduction using FFmpeg vs. Python-based audio processing. In practice this matters because the CPU is also running Kafka consumers, Spark jobs, and the Streamlit dashboard.

**Whisper input format.** Whisper requires 16kHz mono PCM. Raw Twitch stream audio comes in as AAC at 44.1kHz or 48kHz stereo. FFmpeg handles this conversion with a single filter chain:

```bash
ffmpeg -i [streamlink_pipe] -vn -acodec pcm_s16le -ar 16000 -ac 1 -f s16le pipe:1
```

Doing this in Python requires librosa's `resample()` which is significantly slower and adds librosa as a hard dependency for a task FFmpeg does natively.

**Frame boundary control.** FFmpeg gives precise control over chunk size via `-blocksize`. This lets the pipeline tune the trade-off between latency (smaller chunks, more frequent Whisper calls) and throughput (larger chunks, fewer but more accurate transcriptions). The prototype used variable chunking based on WebRTC VAD output — only sending frames with detected speech.

**WebRTC VAD integration.** The pipeline gates FFmpeg output through `webrtcvad` before Whisper. Only frames classified as "speech" by VAD are sent to Whisper. This:
- Reduces Whisper inference calls by ~40–60% during music/silence periods
- Improves transcription quality (Whisper is confused by background music)
- Reduces VRAM usage (fewer concurrent inference sessions)

## Pipeline Code Pattern

```python
# Subprocess pipe: Streamlink → FFmpeg → Python audio buffer
ffmpeg_process = subprocess.Popen([
    'ffmpeg', '-i', streamlink_pipe,
    '-vn', '-acodec', 'pcm_s16le', '-ar', '16000', '-ac', '1',
    '-f', 's16le', 'pipe:1'
], stdout=subprocess.PIPE, stderr=subprocess.DEVNULL)

# VAD filters before Whisper
vad = webrtcvad.Vad(aggressiveness=2)
for frame in read_frames(ffmpeg_process.stdout, frame_duration_ms=30):
    if vad.is_speech(frame, sample_rate=16000):
        whisper_queue.put(frame)
```

## Trade-offs Accepted

- FFmpeg must be installed as a system dependency (not pip-installable)
- The subprocess pipe adds one process boundary, which has marginal overhead
- VAD aggressiveness level (0–3) needs tuning per stream — too aggressive misses quiet speech, too lenient lets music through

## Consequences

FFmpeg is a hard system dependency. The `docker-compose.yml` Dockerfile includes `ffmpeg` in the base image. The `live_audio_engine.py` checks `FFMPEG_AVAILABLE` at startup and refuses to run if FFmpeg is not found in PATH. The audio pipeline is entirely subprocess-based — no Python audio libraries are in the hot path.
