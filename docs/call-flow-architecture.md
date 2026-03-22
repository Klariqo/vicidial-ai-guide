# Call Flow Architecture

> A detailed breakdown of what happens during an AI-handled call on VICIdial — from the moment the dialer places the call to final disposition.

---

## High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        VICIdial + AI Agent Call Flow                       │
│                                                                            │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────┐                   │
│  │ VICIdial │     │   Asterisk   │     │  AI Agent    │                   │
│  │ Dialer   │     │  (chan_sip)  │     │ (SIP Ext)    │                   │
│  └────┬─────┘     └──────┬───────┘     └──────┬───────┘                   │
│       │                  │                     │                           │
│       │  1. Place call   │                     │                           │
│       │─────────────────→│                     │                           │
│       │                  │                     │                           │
│       │  2. Lead answers │                     │                           │
│       │  (or AMD detect) │                     │                           │
│       │←─────────────────│                     │                           │
│       │                  │                     │                           │
│       │  3. Route to AI  │                     │                           │
│       │─────────────────→│                     │                           │
│       │                  │  4. SIP INVITE      │                           │
│       │                  │────────────────────→│                           │
│       │                  │                     │                           │
│       │                  │  5. 200 OK + SDP    │                           │
│       │                  │←────────────────────│                           │
│       │                  │                     │                           │
│       │                  │  6. ACK             │                           │
│       │                  │────────────────────→│                           │
│       │                  │                     │                           │
│       │                  │  ═══ RTP Audio ═══  │                           │
│       │                  │←══════════════════→│                           │
│       │                  │                     │                           │
│       │                  │                     │  ┌─────────────────────┐  │
│       │                  │                     │  │  Voice AI Pipeline  │  │
│       │                  │                     │  │                     │  │
│       │                  │                     │  │  RTP ──→ STT        │  │
│       │                  │                     │  │  (mulaw)  (text)    │  │
│       │                  │                     │  │            │        │  │
│       │                  │                     │  │            ▼        │  │
│       │                  │                     │  │          LLM        │  │
│       │                  │                     │  │        (response)   │  │
│       │                  │                     │  │            │        │  │
│       │                  │                     │  │            ▼        │  │
│       │                  │                     │  │          TTS        │  │
│       │                  │                     │  │        (speech)     │  │
│       │                  │                     │  │            │        │  │
│       │                  │                     │  │            ▼        │  │
│       │                  │                     │  │  TTS ──→ RTP        │  │
│       │                  │                     │  │  (mulaw)            │  │
│       │                  │                     │  └─────────────────────┘  │
│       │                  │                     │                           │
│       │                  │  7. Transfer or BYE │                           │
│       │                  │←────────────────────│                           │
│       │                  │                     │                           │
│       │  8. Disposition  │                     │                           │
│       │←─────────────────│                     │                           │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Call Origination

### VICIdial's Predictive Dialer

VICIdial's dialer places outbound calls based on the campaign configuration:

- **Dial ratio**: How many calls to place per available agent (e.g., 3:1 means 3 calls per 1 idle agent)
- **Trunk selection**: Which SIP trunk or carrier to use for outbound calls
- **Caller ID**: The outbound caller ID displayed to the lead

The dialer continuously calculates optimal dial ratios based on answer rates, agent availability, and drop-rate limits (FTC compliance for US calls: max 3% drop rate).

### Answer Detection

When a call connects, VICIdial/Asterisk determines if a human answered or if it reached a voicemail:

- **Answer supervision**: The carrier signals when the called party picks up
- **AMD (Answering Machine Detection)**: Analyzes the first 2-4 seconds of audio to detect voicemail greetings. VICIdial supports both synchronous AMD (blocks until detection) and asynchronous AMD (connects immediately, detects in background)

For AI agent deployments, **asynchronous AMD** is recommended — the AI receives the call immediately and can handle voicemail detection itself (often more accurately than Asterisk's AMD).

---

## Phase 2: SIP Signaling

### The SIP INVITE

When VICIdial routes an answered call to the AI agent, Asterisk sends a SIP INVITE:

```
INVITE sip:8001@ai-server-ip:5060 SIP/2.0
Via: SIP/2.0/UDP vicidial-ip:5060;branch=z9hG4bK-xxx
From: <sip:vicidial@vicidial-ip>;tag=xxx
To: <sip:8001@ai-server-ip>
Call-ID: unique-call-id@vicidial-ip
CSeq: 1 INVITE
Content-Type: application/sdp

v=0
o=- 12345 12345 IN IP4 vicidial-ip
s=Asterisk
c=IN IP4 vicidial-ip
t=0 0
m=audio 10000 RTP/AVP 0 101
a=rtpmap:0 PCMU/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=ptime:20
a=sendrecv
```

Key elements in the SDP:
- **Codec**: `PCMU/8000` = G.711 mulaw at 8kHz
- **Payload type 0**: Standard mulaw
- **Payload type 101**: DTMF (telephone-event / RFC 2833)
- **ptime 20**: 20ms packetization interval
- **sendrecv**: Bidirectional audio

### SIP Headers from VICIdial

VICIdial can pass useful metadata in custom SIP headers:

| Header | Value | Use |
|--------|-------|-----|
| `X-VICIDial-Lead-Id` | Lead ID from the database | Link to lead record in CRM |
| `X-VICIDial-Caller-Id` | The phone number dialed | Caller identification |
| `X-VICIDial-Lead-Name` | Lead's name from CRM | Personalize the greeting |
| `X-VICIDial-Campaign-Id` | Campaign identifier | Route to correct AI persona |
| `X-VICIDial-List-Id` | List identifier | Track lead source |

These headers are invaluable for the AI agent — it can greet the lead by name, use the correct campaign script, and link conversation data back to VICIdial's CRM.

### The AI Agent's Response

The AI agent responds with `200 OK` and its own SDP:

```
SIP/2.0 200 OK
From: <sip:vicidial@vicidial-ip>;tag=xxx
To: <sip:8001@ai-server-ip>;tag=yyy
Call-ID: unique-call-id@vicidial-ip
CSeq: 1 INVITE
Contact: <sip:8001@ai-server-ip:5060>
Content-Type: application/sdp

v=0
o=- 67890 67890 IN IP4 ai-server-ip
s=AI Agent
c=IN IP4 ai-server-ip
t=0 0
m=audio 16000 RTP/AVP 0 101
a=rtpmap:0 PCMU/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=ptime:20
a=sendrecv
```

After the `ACK`, the call is established and RTP audio begins flowing.

---

## Phase 3: Audio Processing

### RTP Audio Stream

Audio flows as RTP packets between Asterisk and the AI agent:

```
Direction: Lead → Asterisk → AI Agent
┌────────────────────────────────────────┐
│ RTP Packet                             │
│ ┌─────────────┬───────────────────────┐│
│ │ RTP Header  │ Payload (160 bytes)   ││
│ │ (12 bytes)  │ mulaw audio           ││
│ │             │ = 20ms of speech      ││
│ └─────────────┴───────────────────────┘│
│ Total: 172 bytes, sent every 20ms      │
│ = 50 packets/second per direction      │
└────────────────────────────────────────┘
```

### Audio Format Details

| Property | Value |
|----------|-------|
| Codec | G.711 mulaw (PCMU) |
| Sample Rate | 8000 Hz |
| Bit Depth | 8 bits (after mulaw compression) |
| Bitrate | 64 kbps |
| Packet Interval | 20 ms |
| Samples per Packet | 160 |
| Bytes per Packet | 160 (payload) + 12 (RTP header) = 172 |

### The Voice AI Pipeline

The AI agent processes audio through a real-time pipeline:

#### 1. Speech-to-Text (STT)

Incoming RTP audio is fed to a streaming STT engine:

- Audio arrives as mulaw 8kHz RTP packets
- Most STT engines expect linear PCM — the AI agent transcodes mulaw → linear PCM 16kHz
- Transcription results arrive as partial (interim) and final results
- Streaming STT provides word-level results with minimal delay

**Typical STT latency**: 100-200ms from end of utterance to final transcript.

Common STT providers: Deepgram, AssemblyAI, Google Cloud Speech, Azure Speech, Whisper (self-hosted).

#### 2. Turn Detection

The system must detect when the caller has finished speaking (end-of-turn). This is surprisingly hard — humans pause mid-sentence, and silence doesn't always mean the person is done talking.

Approaches:
- **Voice Activity Detection (VAD)**: Detects silence after speech. Simple but prone to false triggers on mid-sentence pauses.
- **Endpoint detection with STT**: Modern STT engines (like Deepgram) provide built-in turn detection that considers linguistic context, not just silence duration.
- **LLM-assisted**: Some systems send partial transcripts to the LLM to predict if the speaker is done.

A typical configuration: require 600-800ms of silence after speech, with a higher threshold for mid-sentence detection.

#### 3. Language Model (LLM)

Once a complete utterance is detected, the transcript is sent to an LLM:

```
System: You are an AI agent calling on behalf of [Company].
Your goal is to qualify leads by asking about [criteria].
If qualified, transfer to a human agent.

Conversation so far:
AI: Hi, this is Sarah calling from Health Benefits Center...
Lead: Yeah, what is this about?

Generate the next response.
```

The LLM generates a response based on:
- Campaign-specific instructions (persona, qualifying questions, objection handling)
- Full conversation context
- Lead metadata from VICIdial headers (name, source, previous interactions)

**Typical LLM latency**: 50-150ms for first token (using fast inference providers like Groq, Together, or Fireworks).

#### 4. Text-to-Speech (TTS)

The LLM's response is converted to speech:

- Text is sent to a TTS engine
- TTS generates audio (typically PCM or mulaw)
- Audio is streamed back as it's generated (sentence-by-sentence or word-by-word)
- The AI agent encodes the audio as mulaw 8kHz RTP packets and sends them to Asterisk

**Typical TTS latency**: 40-150ms for first audio chunk.

Common TTS providers: Cartesia, ElevenLabs, Deepgram Aura, PlayHT, XTTS (self-hosted).

#### 5. Sentence Streaming

For natural conversation flow, the LLM response is streamed sentence by sentence:

1. LLM generates: "Sure, I can help you with that."
2. This sentence is immediately sent to TTS while the LLM continues generating
3. TTS begins generating audio for this sentence
4. Audio starts playing to the caller
5. Meanwhile, the LLM generates the next sentence: "Can I ask you a few questions?"
6. Steps 2-4 repeat for each sentence

This pipelining means the caller hears the first sentence while the AI is still generating the rest of the response. It dramatically reduces perceived latency.

### Barge-In Handling

When the caller starts speaking while the AI is talking, the system needs to handle "barge-in":

1. STT detects speech from the caller while TTS audio is being sent
2. The AI immediately stops sending TTS audio (cancels remaining RTP packets)
3. STT continues processing the caller's speech
4. Once the caller finishes, the pipeline generates a new response

Good barge-in handling requires:
- Filtering out echo (the caller's microphone may pick up the AI's voice)
- Minimum speech threshold (don't interrupt on background noise — typically require 2+ words)
- Fast TTS cancellation (stop sending audio within 50-100ms of detecting barge-in)

---

## Phase 4: Call Disposition

### Transfer to Human Agent

When the AI determines a lead is qualified, it initiates a transfer:

#### Option A: VICIdial Agent API

The AI calls VICIdial's Agent API to transfer the call:

```
# Example: Transfer to the next available human agent
POST http://vicidial-ip/agc/api.php
Content-Type: application/x-www-form-urlencoded

function=transfer_conference
agent_user=ai-agent-user
value=AXFER
phone_number=8101    # Human agent's extension
```

This tells VICIdial to bridge the lead to another agent, maintaining the call within VICIdial's conference system.

#### Option B: SIP REFER

The AI sends a SIP REFER message to Asterisk, requesting the call be redirected:

```
REFER sip:call-id@vicidial-ip SIP/2.0
Refer-To: <sip:8101@vicidial-ip>
Referred-By: <sip:8001@ai-server-ip>
```

Asterisk processes the REFER and redirects the call to the target extension.

#### Option C: AI Drops from Conference

If the call was set up as a VICIdial conference (3-way call between lead, AI, and Asterisk), the AI simply sends a SIP BYE to drop its leg. VICIdial then connects the lead to the next available human agent automatically.

### Hangup

If the lead is not qualified, the AI sends a SIP BYE:

```
BYE sip:call-id@vicidial-ip SIP/2.0
From: <sip:8001@ai-server-ip>;tag=yyy
To: <sip:vicidial@vicidial-ip>;tag=xxx
Call-ID: unique-call-id@vicidial-ip
CSeq: 2 BYE
```

### Disposition Reporting

After the call ends, the AI should report the disposition back to VICIdial:

```
POST http://vicidial-ip/agc/api.php
Content-Type: application/x-www-form-urlencoded

function=external_status
agent_user=ai-agent-user
value=NQ          # Not Qualified (or SALE, CALLBK, DNC, etc.)
lead_id=12345
```

Common dispositions:
| Code | Meaning |
|------|---------|
| `SALE` / `XFER` | Qualified, transferred to human |
| `NQ` | Not qualified |
| `DNC` | Do not call (requested removal) |
| `CALLBK` | Callback requested |
| `VM` | Reached voicemail |
| `NI` | Not interested |
| `DISC` | Disconnected number |

---

## Timing Budget

Here's where the milliseconds go in a typical AI response cycle (direct SIP registration):

```
Event                          Time (ms)    Cumulative
─────────────────────────────────────────────────────
Caller finishes speaking         0              0
RTP → AI server (network)      5-15          5-15
mulaw → PCM transcode           1-2          6-17
STT final transcript          100-200       106-217
Turn detection confirm         20-50        126-267
LLM first token               50-100       176-367
TTS first audio chunk          40-100       216-467
PCM → mulaw transcode           1-2        217-469
AI server → Asterisk (network)  5-15       222-484
Caller hears first word          0         222-484
─────────────────────────────────────────────────────
Total: 220-485ms end-to-end
```

This is within the range of natural human conversation turn-taking (typically 200-600ms between speakers).

---

## Next Steps

- [SIP Registration Setup](sip-registration-setup.md) — configure VICIdial step by step
- [Troubleshooting](troubleshooting.md) — fix common issues
- [Latency Comparison](latency-comparison.md) — compare integration approaches
