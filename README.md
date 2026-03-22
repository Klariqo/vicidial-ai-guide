# How to Add AI Agents to VICIdial: Complete Guide

> A practical, technical guide for BPO operators and call center managers who want to integrate AI voice agents with their VICIdial dialer. No fluff — just architecture, setup steps, and real-world lessons from production deployments.

---

## Table of Contents

- [Who This Guide Is For](#who-this-guide-is-for)
- [Why AI + VICIdial](#why-ai--vicidial)
- [Three Integration Approaches](#three-integration-approaches)
- [How SIP Extension Registration Works](#how-sip-extension-registration-works)
- [Call Flow Overview](#call-flow-overview)
- [Getting Started](#getting-started)
- [Detailed Documentation](#detailed-documentation)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

---

## Who This Guide Is For

This guide is written for:

- **BPO operators** running VICIdial or GoAutoDial who want to add AI-powered lead qualification to their outbound campaigns
- **Call center managers** looking to reduce agent idle time by having AI handle the first 30-60 seconds of every call
- **VoIP engineers** who need to understand the SIP-level integration between AI platforms and Asterisk-based dialers
- **Developers** building AI voice agent platforms who want to integrate with VICIdial's call routing

You should have basic familiarity with VICIdial's admin interface and general SIP/VoIP concepts. Deep Asterisk expertise is helpful but not required — this guide explains what you need to know.

---

## Why AI + VICIdial

### The Problem Every Outbound BPO Faces

If you run an outbound call center on VICIdial, you already know the numbers:

- **60-80% of dialed calls** reach voicemails, wrong numbers, disconnected lines, or people who immediately hang up
- **Human agents spend most of their time waiting**, listening to rings, leaving voicemails, or talking to people who have zero interest
- **Cost per qualified transfer** is high because you're paying agents for all that dead time
- **Agent burnout** is real — nobody wants to make 200 calls to have 15 real conversations

The math is brutal. A 10-seat outbound team at $12/hour costs $960/day. If each agent produces 8-12 qualified transfers per day, your cost per transfer is $8-12 before you factor in dialer infrastructure, telephony, and management overhead.

### What AI Changes

An AI agent can handle the first portion of every call — the part that's mostly wasted time. The AI:

1. **Detects voicemails** and hangs up (or drops a pre-recorded message)
2. **Screens out wrong numbers** and disconnected lines
3. **Qualifies interested leads** by asking initial screening questions (age, location, insurance status, interest level)
4. **Transfers qualified leads** to human agents who only talk to people ready for a real conversation

The result: your human agents spend close to 100% of their time on qualified conversations. Instead of making 200 calls to get 10 transfers, a human agent receives 10 pre-qualified transfers and closes them.

### Why VICIdial Specifically

VICIdial is the most widely deployed open-source predictive dialer in the BPO industry. It runs on Asterisk, which means it speaks SIP natively. This is a massive advantage for AI integration — you don't need proprietary APIs or vendor lock-in. Any system that can register as a SIP extension can plug into VICIdial's call routing.

VICIdial's Remote Agent feature was originally designed for work-from-home human agents connecting via softphone. It turns out this same mechanism works perfectly for AI agents. VICIdial doesn't know or care whether the extension is a human with a headset or an AI processing audio — it just routes calls via SIP.

---

## Three Integration Approaches

There are three ways to connect an AI agent to VICIdial. Each has different tradeoffs in complexity, latency, and cost.

### Approach 1: SIP Trunk Relay

**How it works:** You set up a SIP trunk between VICIdial and a cloud telephony provider (like Twilio, Telnyx, or SignalWire). Calls are routed from VICIdial through the trunk to the AI platform, which processes the audio and sends responses back through the same trunk.

**Pros:**
- Relatively simple to set up if you're already familiar with SIP trunking
- The AI platform doesn't need direct network access to your VICIdial server
- Cloud telephony providers handle NAT traversal and codec negotiation

**Cons:**
- **Added latency**: Every audio packet makes an extra hop through the trunk provider's media servers. This typically adds 100-300ms of round-trip delay, pushing total response time to 600-800ms
- **Ongoing per-minute costs**: Trunk providers charge $0.004-0.015/min for media relay, which adds up at volume
- **Dependency on a third party**: If the trunk provider has an outage, your AI integration goes down even if VICIdial and the AI platform are both healthy

**Best for:** Quick proof-of-concept deployments where latency isn't critical, or situations where you can't allow direct network access to your VICIdial server.

### Approach 2: API/AGI Integration

**How it works:** You write an Asterisk AGI (Asterisk Gateway Interface) script or AMI (Asterisk Manager Interface) listener that intercepts calls at the dialplan level. When a call is answered, the AGI script sends audio to an external AI API via HTTP, receives the response, and plays it back.

**Pros:**
- Deep integration with Asterisk's call handling
- Can intercept calls at any point in the dialplan
- No SIP registration required

**Cons:**
- **High complexity**: Requires deep Asterisk dialplan expertise and custom AGI scripting
- **Poor latency**: HTTP request/response cycles for each audio chunk add 1000-2000ms of delay. Streaming helps but adds significant complexity
- **Brittle**: AGI scripts run synchronously in the dialplan — if the AI API is slow or down, the call hangs
- **Maintenance burden**: Tightly coupled to your Asterisk version and dialplan structure

**Best for:** Scenarios where you need to intercept calls before they reach an agent (e.g., IVR replacement), or when you have strong Asterisk engineering talent in-house.

### Approach 3: SIP Extension Registration (Recommended)

**How it works:** The AI platform registers as a SIP phone extension on VICIdial's Asterisk server, exactly like a human agent's softphone would. VICIdial treats the AI as just another agent. When calls are routed to it, Asterisk sends a SIP INVITE, the AI answers, and RTP audio flows directly between them.

**Pros:**
- **Lowest latency**: Direct RTP audio stream with no intermediary. Sub-500ms end-to-end response time
- **Zero VICIdial modifications**: You're using VICIdial's existing Remote Agent feature exactly as designed
- **Zero per-minute relay costs**: No trunk provider in the middle
- **Simple setup**: Add a phone extension, add a remote agent, point the AI at it. 10-minute setup on the VICIdial side
- **VICIdial treats AI like any other agent**: Reporting, campaign assignment, pause codes — everything works as normal

**Cons:**
- Requires network connectivity between the AI server and VICIdial's Asterisk (direct IP or VPN)
- AI platform must implement SIP registration and RTP audio handling
- NAT/firewall configuration can be tricky if servers are on different networks

**Best for:** Production deployments where latency and call quality matter. This is the approach used in most large-scale AI + VICIdial integrations.

> **Note on this guide:** The detailed setup instructions in this guide focus on Approach 3 (SIP Extension Registration) because it offers the best combination of simplicity, performance, and cost. We use [Klariqo](https://klariqo.com) as the example AI agent platform throughout, but the SIP registration concept applies to any AI platform that supports SIP — the VICIdial-side configuration is identical regardless of which AI system you connect.

---

## How SIP Extension Registration Works

Understanding the mechanics of SIP registration helps you debug issues and optimize your setup. Here's what happens under the hood.

### VICIdial's Asterisk Architecture

VICIdial uses Asterisk as its telephony engine. Specifically, most VICIdial installations use **chan_sip** (Asterisk's original SIP channel driver) rather than the newer **res_pjsip**. This matters because the registration process differs slightly between the two.

When you create a Phone extension in VICIdial's admin interface, VICIdial writes a SIP peer definition into Asterisk's configuration. This peer definition specifies:

- The extension number (e.g., `8001`)
- Authentication credentials (username/password)
- Allowed codecs (typically G.711 mulaw/alaw)
- The context for incoming calls

### The Registration Process

When the AI agent starts up, it sends a SIP REGISTER request to VICIdial's Asterisk server:

```
REGISTER sip:vicidial-server-ip SIP/2.0
From: <sip:8001@vicidial-server-ip>
To: <sip:8001@vicidial-server-ip>
Contact: <sip:8001@ai-server-ip:5060>
Expires: 120
```

Asterisk challenges this with a `401 Unauthorized` response containing a nonce for digest authentication. The AI agent responds with proper credentials, and Asterisk accepts the registration. From this point on, Asterisk knows that extension `8001` is reachable at the AI server's IP address.

The AI agent must re-register periodically (typically every 60-120 seconds) to keep the registration alive. If registration lapses, Asterisk considers the extension offline and won't route calls to it.

### What Happens When a Call Arrives

1. VICIdial's predictive dialer places an outbound call to a lead
2. The lead answers (or VICIdial's AMD detects a human pickup)
3. VICIdial's campaign routing assigns the call to the AI agent (which VICIdial sees as a Remote Agent)
4. Asterisk sends a SIP INVITE to the AI agent's registered contact address
5. The AI agent responds with `200 OK`, establishing the call
6. RTP audio begins flowing bidirectionally:
   - Lead's audio → Asterisk → AI agent (for speech-to-text processing)
   - AI agent's synthesized speech → Asterisk → Lead (TTS audio)
7. The call continues until the AI agent either transfers the lead or hangs up

### Audio Format

VICIdial/Asterisk typically uses **G.711 mulaw** (also written as u-law or PCMU) at **8kHz** sample rate. This is the standard telephony codec — 8 bits per sample, 8000 samples per second, resulting in 64 kbps of audio data.

RTP packets are sent every 20 milliseconds, containing 160 bytes of audio data per packet. Your AI platform needs to:

- Receive and decode mulaw RTP packets into PCM audio for speech-to-text
- Encode TTS output as mulaw and send it back as RTP packets at the correct timing interval

---

## Call Flow Overview

Here's the complete lifecycle of an AI-handled call on VICIdial:

```
┌──────────────┐    ┌───────────────┐    ┌──────────────┐    ┌─────────────┐
│   VICIdial   │    │   Asterisk    │    │   AI Agent   │    │  Voice AI   │
│   Dialer     │    │  (chan_sip)   │    │  (SIP Ext)   │    │  Pipeline   │
└──────┬───────┘    └───────┬───────┘    └──────┬───────┘    └──────┬──────┘
       │                    │                    │                   │
       │ 1. Dial lead       │                    │                   │
       │───────────────────→│                    │                   │
       │                    │                    │                   │
       │ 2. Lead answers    │                    │                   │
       │←───────────────────│                    │                   │
       │                    │                    │                   │
       │ 3. Route to AI ext │                    │                   │
       │───────────────────→│                    │                   │
       │                    │ 4. SIP INVITE      │                   │
       │                    │───────────────────→│                   │
       │                    │                    │                   │
       │                    │ 5. 200 OK          │                   │
       │                    │←───────────────────│                   │
       │                    │                    │                   │
       │                    │ 6. RTP audio flows │                   │
       │                    │←──────────────────→│                   │
       │                    │                    │ 7. Audio → STT    │
       │                    │                    │──────────────────→│
       │                    │                    │                   │
       │                    │                    │ 8. LLM response   │
       │                    │                    │←──────────────────│
       │                    │                    │                   │
       │                    │                    │ 9. TTS → RTP      │
       │                    │                    │──────────────────→│
       │                    │                    │                   │
       │                    │  10. AI qualifies, │                   │
       │                    │      transfers or  │                   │
       │                    │      hangs up      │                   │
       │                    │←───────────────────│                   │
       │                    │                    │                   │
```

### Phase 1: Dialing (Steps 1-2)

VICIdial's predictive dialer places outbound calls based on your campaign's dial ratio. When a lead answers (detected by Asterisk's answer supervision or AMD), the call is ready to be connected to an agent.

### Phase 2: Routing (Steps 3-5)

VICIdial's campaign routing selects the next available agent. If the AI agent is configured as a Remote Agent in the campaign, VICIdial routes the call to it. Asterisk sends a SIP INVITE to the AI agent's registered IP address. The AI agent accepts with a `200 OK`.

### Phase 3: Conversation (Steps 6-9)

This is where the AI does its work. The voice pipeline runs in real-time:

1. **Speech-to-Text (STT)**: The lead's audio is transcribed in real-time using a streaming STT engine. Modern STT models like Deepgram or AssemblyAI provide word-level streaming with minimal latency.

2. **Turn Detection**: The system detects when the caller has finished speaking (end-of-turn). This uses a combination of silence detection, speech pattern analysis, and sometimes specialized models trained on conversational turn-taking.

3. **Language Model (LLM)**: The transcribed text is sent to an LLM (like Llama, GPT, or Gemini) along with the conversation context and campaign-specific instructions. The LLM generates an appropriate response.

4. **Text-to-Speech (TTS)**: The LLM's response is converted to speech using a TTS engine. Modern TTS (Cartesia, ElevenLabs, Deepgram) can produce natural-sounding speech with sub-100ms latency.

5. **Barge-in Handling**: If the caller starts speaking while the AI is talking, the AI should stop its current utterance and listen. This requires canceling the current TTS output and re-engaging the STT pipeline.

### Phase 4: Disposition (Step 10)

Based on the conversation, the AI makes a decision:

- **Transfer**: The lead is qualified. The AI signals a transfer to a human agent (via SIP REFER, VICIdial API, or by simply ending its leg while VICIdial's conferencing handles the bridge).
- **Hangup**: The lead is not qualified, requested to be removed from the list, or the conversation has naturally concluded.
- **Voicemail Drop**: If AMD detects a voicemail, the AI can play a pre-recorded message and hang up.

The AI should also report disposition data back to VICIdial (via the VICIdial Agent API) so that lead statuses are updated in the CRM.

---

## Getting Started

The fastest path to a working integration:

1. **Read the [SIP Registration Setup Guide](docs/sip-registration-setup.md)** — step-by-step VICIdial configuration
2. **Review the [Call Flow Architecture](docs/call-flow-architecture.md)** — understand what happens at each stage
3. **Check the [Latency Comparison](docs/latency-comparison.md)** — understand why direct SIP registration outperforms other approaches
4. **Bookmark the [Troubleshooting Guide](docs/troubleshooting.md)** — you'll need it during setup

If you want to skip building your own AI voice pipeline and use an existing platform, [Klariqo](https://klariqo.com) provides a production-ready AI agent that registers directly as a SIP extension on your VICIdial server. Setup takes about 10 minutes on the VICIdial side — you create a phone extension, share the credentials, and Klariqo handles the rest. Contact hello@klariqo.com for a free pilot.

---

## Detailed Documentation

| Document | Description |
|----------|-------------|
| [SIP Registration Setup](docs/sip-registration-setup.md) | Step-by-step VICIdial + AI agent configuration |
| [Call Flow Architecture](docs/call-flow-architecture.md) | Detailed call lifecycle, audio formats, transfer mechanisms |
| [Latency Comparison](docs/latency-comparison.md) | Real latency data across integration approaches |
| [Troubleshooting](docs/troubleshooting.md) | Common issues and fixes |
| [FAQ](FAQ.md) | Frequently asked questions |

---

## FAQ

### 1. Does this work with GoAutoDial?

**Yes.** GoAutoDial is built on top of VICIdial. The admin interface looks different, but the underlying Asterisk and SIP infrastructure is the same. The setup steps in this guide apply directly — you'll just find the Phone and Remote Agent settings in GoAutoDial's admin panel instead of VICIdial's.

### 2. What about Asterisk version compatibility?

VICIdial typically runs on Asterisk 13 or 16 with **chan_sip**. This guide assumes chan_sip. If your Asterisk installation uses **res_pjsip** (the newer SIP stack), the registration process is slightly different — the AOR (Address of Record) name must exactly match the SIP username, and the configuration lives in `pjsip.conf` rather than `sip.conf`. The core concept is the same, but the config syntax differs.

### 3. Can the AI handle multiple concurrent calls?

**Yes.** Each call is an independent SIP session with its own RTP streams. The AI platform handles multiple simultaneous calls by spawning separate processing pipelines for each. In VICIdial, you control how many calls are routed to the AI by setting the "Number of Lines" on the Remote Agent — if you set it to 10, VICIdial will send up to 10 concurrent calls to the AI extension.

### 4. What languages does this support?

The language capability depends on the AI platform's STT and TTS stack, not on VICIdial. English is universally supported. Spanish, French, Hindi, and other major languages are available depending on your STT/TTS provider. VICIdial and Asterisk are language-agnostic at the SIP/RTP level — they just pass audio.

### 5. What's the response latency?

With direct SIP registration: **sub-500ms** from end-of-caller-speech to start-of-AI-response. This is fast enough that callers don't notice any unnatural delay. Via a SIP trunk relay (e.g., Twilio), expect 600-800ms. Via API/AGI integration, 1000-2000ms. See the [Latency Comparison](docs/latency-comparison.md) for a detailed breakdown.

### 6. Can the AI transfer qualified leads back to human agents?

**Yes.** There are several transfer mechanisms:

- **VICIdial API transfer**: The AI calls VICIdial's Agent API to initiate a transfer, which moves the call to the next available human agent in the campaign
- **SIP REFER**: The AI sends a SIP REFER message to redirect the call
- **Conference bridge**: The AI drops its leg from a VICIdial conference, leaving the lead connected to the human agent that VICIdial bridges in

The best approach depends on your campaign setup and whether you want a cold transfer (AI drops, human picks up) or warm transfer (AI stays on briefly to introduce the lead).

### 7. Do I need to modify my VICIdial server?

**Minimal changes.** You need to:

1. Add a Phone extension (2-minute task in VICIdial admin)
2. Add a Remote Agent entry (2-minute task)
3. Whitelist the AI server's IP address in your firewall (if applicable)

No code changes, no Asterisk dialplan modifications, no VICIdial patches. You're using existing features exactly as they were designed.

### 8. What about call recording?

Recording can happen at multiple points:

- **On VICIdial/Asterisk**: VICIdial's built-in recording captures the full call as it flows through Asterisk. This works regardless of whether the call goes to a human or AI agent.
- **On the AI side**: The AI platform can record both sides of the audio for quality analysis, training data, and compliance. This gives you a separate recording that you fully control.
- **Both**: Many deployments record on both sides for redundancy. VICIdial's recording serves as the system of record; the AI-side recording is used for AI improvement.

### 9. How does the AI know what to say?

The AI agent is configured with campaign-specific instructions — a system prompt that defines its persona, qualifying questions, transfer criteria, and conversation guardrails. For example, an ACA (Affordable Care Act) health insurance campaign might instruct the AI to:

- Introduce itself as calling from a specific organization
- Ask the lead's age, zip code, and current insurance status
- Transfer leads over 26 who are uninsured or underinsured
- Politely end the call if the lead doesn't qualify

These instructions are configured on the AI platform side, not in VICIdial. VICIdial just routes calls — the AI handles the conversation.

### 10. What's the cost structure?

The cost of adding AI to VICIdial breaks down into:

- **AI platform costs**: Typically $0.05-0.15/min depending on the provider, volume, and which STT/LLM/TTS models are used
- **VICIdial infrastructure**: No additional cost — you're using existing capacity
- **Telephony**: With direct SIP registration, there's no per-minute relay cost. If you go through a SIP trunk provider, add their per-minute rate
- **Offset savings**: Reduced human agent hours. If an AI agent handles the first 60 seconds of every call and only transfers qualified leads, your human agents are dramatically more productive

At scale, the economics are compelling. A 10-seat human team at $12/hour costs ~$250K/year. An AI front-end handling first contact at $0.10/min, processing 50,000 minutes/month, costs ~$60K/year — and your human team can be smaller because they're only handling qualified transfers.

---

## Contributing

This guide is maintained by [Klariqo](https://klariqo.com) and the VICIdial AI integration community. Contributions are welcome:

- **Found an error?** Open an issue or submit a PR
- **Have a different integration approach?** We'd love to document it
- **Running this in production?** Share your experience — what worked, what didn't

---

## License

This guide is released under the [MIT License](LICENSE). Use it freely.
