# Latency Comparison: AI Integration Approaches

> Real-world latency measurements across different methods of connecting AI agents to VICIdial. Latency is measured as the time from when the caller stops speaking to when they hear the AI's first word.

---

## Summary

| Approach | End-to-End Latency | Notes |
|----------|-------------------|-------|
| **Direct SIP Registration** | **220-500ms** | Recommended. No intermediary. |
| SIP Trunk Relay (e.g., Twilio) | 600-800ms | Extra hop through trunk provider |
| API / AGI Integration | 1000-2000ms | HTTP overhead, no streaming |

---

## Why Latency Matters

Human conversation has a natural rhythm. Research on conversational turn-taking shows:

- **Normal turn gap**: 200-600ms between speakers
- **Noticeable delay**: Anything above ~700ms starts to feel unnatural
- **Disruptive delay**: Above ~1200ms, conversations break down — speakers start talking over each other, repeating themselves, or assuming the other person has disconnected

For AI voice agents handling lead qualification calls, latency directly impacts:

- **Caller engagement**: Slow responses cause callers to lose interest or hang up
- **Natural conversation flow**: High latency leads to overlapping speech and awkward pauses
- **Transfer rates**: Callers who feel they're "talking to a robot" are less likely to engage, reducing qualified transfer rates

Every 100ms of added latency measurably degrades the caller experience.

---

## Approach 1: Direct SIP Registration

```
┌──────────┐         ┌──────────────┐         ┌──────────────┐
│   Lead   │←──RTP──→│   Asterisk   │←──RTP──→│  AI Agent    │
│ (phone)  │         │  (VICIdial)  │         │ (SIP Ext)    │
└──────────┘         └──────────────┘         └──────────────┘
                     Direct RTP path
                     No intermediary
```

### Where the Milliseconds Go

| Stage | Time | Description |
|-------|------|-------------|
| RTP transit (lead → Asterisk) | 5-20ms | Depends on PSTN/carrier path |
| RTP transit (Asterisk → AI) | 5-15ms | Direct IP, typically same region |
| Audio transcode (mulaw → PCM) | 1-2ms | CPU operation, negligible |
| STT processing | 100-200ms | Streaming STT final result after end-of-speech |
| Turn detection confirmation | 20-50ms | Confirming speaker has finished |
| LLM inference (first token) | 50-100ms | Fast inference (Groq, Together, etc.) |
| TTS generation (first chunk) | 40-100ms | Streaming TTS first audio |
| Audio transcode (PCM → mulaw) | 1-2ms | CPU operation, negligible |
| RTP transit (AI → Asterisk) | 5-15ms | Direct IP return path |
| RTP transit (Asterisk → lead) | 5-20ms | PSTN/carrier return path |
| **Total** | **232-524ms** | |

### Why It's Fast

- **Zero intermediary**: RTP audio goes directly between Asterisk and the AI server. No third-party media server in the middle.
- **No protocol translation**: SIP in, SIP out. No WebSocket bridges, no HTTP encapsulation.
- **Native codec**: If the TTS engine can output mulaw 8kHz directly (some can), there's no transcode step at all.

---

## Approach 2: SIP Trunk Relay

```
┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────────┐
│   Lead   │←───→│ Asterisk │←───→│ Trunk Provider│←───→│  AI Agent    │
│ (phone)  │     │(VICIdial)│     │ (e.g. Twilio) │     │              │
└──────────┘     └──────────┘     └──────────────┘     └──────────────┘
                                  Extra hop adds
                                  100-300ms RTT
```

### Where the Milliseconds Go

| Stage | Time | Description |
|-------|------|-------------|
| RTP transit (lead → Asterisk) | 5-20ms | Same as direct |
| RTP transit (Asterisk → trunk provider) | 10-30ms | To trunk provider's media server |
| Trunk provider processing | 5-15ms | Media relay, codec handling |
| Trunk → AI agent (WebSocket or RTP) | 10-30ms | To AI platform |
| Audio transcode | 1-5ms | May transcode twice (mulaw → PCM → mulaw) |
| STT processing | 100-200ms | Same as direct |
| Turn detection | 20-50ms | Same |
| LLM inference | 50-100ms | Same |
| TTS generation | 40-100ms | Same |
| AI agent → trunk provider | 10-30ms | Return path through trunk |
| Trunk provider → Asterisk | 10-30ms | Return through trunk |
| Asterisk → lead | 5-20ms | Same as direct |
| **Total** | **266-650ms** | Typical: 600-800ms |

### Additional Latency Sources

The trunk relay adds latency from:

- **Geographic distance**: Trunk provider's media servers may be in a different region than your VICIdial server
- **Media processing**: The trunk provider may transcode audio (e.g., mulaw → G.722 → mulaw)
- **WebSocket bridging**: If the AI platform connects to the trunk via WebSocket (common with Twilio Media Streams), there's an additional protocol translation step
- **Jitter buffering**: The trunk provider applies its own jitter buffer, adding consistent delay

### Cost Impact

Beyond latency, trunk relays add per-minute costs:

| Provider | Media relay cost |
|----------|-----------------|
| Twilio (Media Streams) | $0.004/min |
| Telnyx | $0.003/min |
| SignalWire | $0.003/min |

At 50,000 minutes/month, that's $150-200/month just for the relay — on top of the AI platform's own costs.

---

## Approach 3: API / AGI Integration

```
┌──────────┐     ┌──────────┐     ┌──────────────┐
│   Lead   │←───→│ Asterisk │────→│  AI API      │
│ (phone)  │     │(VICIdial)│     │  (HTTP)      │
└──────────┘     └──────────┘     └──────────────┘
                      │                   │
                      │  AGI script       │
                      │  records audio,   │
                      │  sends via HTTP,  │
                      │  plays response   │
                      │                   │
```

### Where the Milliseconds Go

| Stage | Time | Description |
|-------|------|-------------|
| Audio capture in AGI | 50-200ms | Buffer audio before sending |
| HTTP request setup | 10-30ms | TCP/TLS handshake (if not pooled) |
| Audio upload | 50-200ms | Send audio chunk to API |
| STT processing | 200-500ms | Non-streaming STT processes full chunk |
| LLM inference | 100-300ms | May not stream, waits for full response |
| TTS generation | 100-300ms | May not stream, generates full audio |
| Audio download | 50-200ms | Download generated audio file |
| AGI playback setup | 20-50ms | Asterisk starts playing the file |
| **Total** | **580-1780ms** | Typical: 1000-2000ms |

### Why It's Slow

- **No streaming**: HTTP request/response is inherently batch-oriented. Audio must be buffered, sent, processed, and returned as complete chunks.
- **TCP/TLS overhead**: Each HTTP request involves connection setup (unless using keep-alive or HTTP/2)
- **Asterisk AGI blocking**: AGI scripts run synchronously in the dialplan. While waiting for the API response, the dialplan thread is blocked.
- **File I/O**: Audio often needs to be written to disk before Asterisk can play it

### When API Integration Makes Sense

Despite the latency, API/AGI integration can work for:

- **IVR replacement**: Where callers expect some delay (e.g., "Please wait while I look that up")
- **Post-call processing**: Processing after the call ends (transcription, analysis, CRM updates)
- **Hybrid approaches**: Use SIP for real-time audio, HTTP for async tasks like disposition updates

---

## Real-World Measurements

The following measurements are from production deployments, not synthetic benchmarks:

### Direct SIP Registration (Klariqo SIP Bridge)

```
Measurement conditions:
- AI server: GCP us-central1
- VICIdial: US-based BPO data center
- STT: Deepgram Flux v2
- LLM: Llama 3.1 8B on Groq
- TTS: Cartesia Sonic Turbo

Results (100 call sample):
- P50: 340ms
- P75: 410ms
- P90: 470ms
- P95: 510ms
- P99: 580ms
```

### SIP Trunk via Twilio

```
Measurement conditions:
- Same AI stack as above
- Added: Twilio SIP Domain + Media Streams
- Twilio media server: US East

Results (100 call sample):
- P50: 620ms
- P75: 690ms
- P90: 740ms
- P95: 810ms
- P99: 920ms
```

### Difference

The trunk relay adds approximately **280ms** at the median. This is the cost of routing audio through an intermediary media server.

---

## Optimization Tips

Regardless of which approach you use, these optimizations reduce latency:

### 1. Use Streaming STT

Non-streaming STT waits for the full utterance before processing. Streaming STT transcribes in real-time, providing results as words are spoken. This alone can save 200-400ms.

### 2. Use Fast LLM Inference

LLM latency varies dramatically by provider:

| Provider | Typical First-Token Latency |
|----------|----------------------------|
| Groq | 30-80ms |
| Together | 50-100ms |
| Fireworks | 50-100ms |
| OpenAI GPT-4 | 300-800ms |
| Claude (Anthropic) | 200-500ms |

For real-time voice, use the fastest inference provider available. Model quality matters less than speed for simple qualification scripts.

### 3. Use Streaming TTS

Like STT, streaming TTS begins generating audio before the full text is ready. Combined with sentence-level streaming from the LLM, this allows the first audio to play while the rest of the response is still being generated.

### 4. Co-locate Servers

Put the AI server in the same region (or same data center) as the VICIdial server. Network latency between US-East and US-West is ~60ms round trip. Same-region is typically <5ms.

### 5. Native Mulaw TTS

If your TTS engine can output mulaw 8kHz directly (Cartesia and Deepgram both support this), you skip the transcode step entirely. This saves 1-5ms per audio chunk, but more importantly eliminates a source of complexity and potential bugs.

### 6. Pre-warm Connections

Establish STT, LLM, and TTS connections before the call arrives. WebSocket connections to STT and TTS services can be pre-opened and reused across calls. This eliminates connection setup latency from the critical path.

---

## Next Steps

- [SIP Registration Setup](sip-registration-setup.md) — configure the fastest approach
- [Call Flow Architecture](call-flow-architecture.md) — understand the full pipeline
- [Troubleshooting](troubleshooting.md) — fix issues when they arise
