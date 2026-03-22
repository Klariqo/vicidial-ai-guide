# Frequently Asked Questions

> Common questions about integrating AI voice agents with VICIdial.

---

### 1. Does this work with GoAutoDial?

**Yes.** GoAutoDial is a UI layer built on top of VICIdial and Asterisk. The underlying SIP and telephony infrastructure is identical. Everything in this guide applies directly — you'll find the Phone and Remote Agent settings in GoAutoDial's admin panel, and the Asterisk CLI commands work the same way.

---

### 2. What Asterisk versions are supported?

VICIdial typically runs on **Asterisk 13** or **Asterisk 16** with the **chan_sip** SIP channel driver. This guide assumes chan_sip.

If your installation uses **res_pjsip** (Asterisk 16+), the core concepts are the same but configuration syntax differs:
- Peer definitions use `pjsip.conf` instead of `sip.conf`
- The AOR (Address of Record) name must exactly match the SIP username
- Use `pjsip show endpoints` instead of `sip show peers`

Asterisk 18 and 20 also work, but VICIdial's official support matrix should be checked for your specific version combination.

---

### 3. Can the AI handle multiple concurrent calls?

**Yes.** Each call is an independent SIP session with its own RTP audio streams. VICIdial controls how many concurrent calls are routed to the AI via the "Number of Lines" setting on the Remote Agent.

The practical limit depends on your AI platform's capacity — specifically how many simultaneous STT/LLM/TTS pipelines it can run. Most cloud-based AI platforms handle 10-50+ concurrent calls per deployment. Start with 1-2 lines for testing and scale up.

---

### 4. What languages are supported?

Language support depends entirely on the AI platform's STT and TTS stack, not on VICIdial. VICIdial and Asterisk are language-agnostic at the SIP/RTP level — they transmit audio without caring about the language being spoken.

**English** is universally supported by all major STT and TTS providers. **Spanish**, **French**, **Portuguese**, **Hindi**, and other major languages are available depending on your provider. Check your STT/TTS vendor's language support documentation.

For multilingual campaigns, some AI platforms can detect the caller's language in real-time and switch the conversation language dynamically.

---

### 5. What's the response latency?

End-to-end latency (from when the caller stops speaking to when they hear the AI's first word):

| Approach | Latency |
|----------|---------|
| Direct SIP Registration | 220-500ms |
| SIP Trunk Relay (Twilio, etc.) | 600-800ms |
| API / AGI Integration | 1000-2000ms |

Sub-500ms latency with direct SIP registration is within the range of natural human conversation turn-taking (200-600ms). See the [Latency Comparison](docs/latency-comparison.md) for a detailed breakdown of where the milliseconds go.

---

### 6. Can the AI transfer qualified leads to human agents?

**Yes.** There are three transfer mechanisms:

1. **VICIdial Agent API**: The AI calls VICIdial's Agent API (`/agc/api.php`) to initiate a conference transfer to the next available human agent
2. **SIP REFER**: The AI sends a SIP REFER message to redirect the call within Asterisk's dialplan
3. **Conference drop**: The AI drops its leg from a VICIdial conference call, leaving the lead connected while VICIdial bridges in a human agent

Both cold transfers (AI hangs up, human picks up) and warm transfers (AI introduces the lead to the human agent) are possible depending on your campaign workflow.

---

### 7. Do I need to modify my VICIdial server?

**Minimal changes.** You need to:

1. **Add a Phone extension** — 2 minutes in VICIdial Admin
2. **Add a Remote Agent** — 2 minutes in VICIdial Admin
3. **Whitelist the AI server's IP** — if your VICIdial server has a firewall (most do)

No code changes. No Asterisk dialplan modifications. No VICIdial patches or version upgrades. You're using VICIdial's existing Remote Agent feature exactly as it was designed.

---

### 8. What about call recording?

Recording can happen at multiple levels:

- **VICIdial/Asterisk recording**: VICIdial's built-in recording works the same whether a call goes to a human or AI agent. The recording captures everything that passes through Asterisk.
- **AI-side recording**: The AI platform records both inbound (lead) and outbound (AI) audio separately. This is useful for quality analysis, AI training, and compliance.
- **Both**: Most production deployments record on both sides. VICIdial's recording is the system of record; the AI recording is used for optimization.

For compliance (TCPA, FCC, state regulations), check your jurisdiction's consent requirements. Many campaigns play a brief disclosure ("This call may be recorded for quality assurance") at the start of the call.

---

### 9. How does voicemail detection work?

Two approaches:

1. **Asterisk AMD**: VICIdial's built-in Answering Machine Detection analyzes the first 2-4 seconds of audio. Configured per-campaign. Can route detected voicemails to the AI for voicemail drop (play a pre-recorded message) or straight to hangup.

2. **AI-side detection**: The AI's STT engine transcribes the first few seconds. If it detects a voicemail greeting pattern ("You've reached...", "Please leave a message..."), the AI either drops a voicemail message or hangs up. This can be more accurate than Asterisk's AMD because it uses language understanding, not just audio pattern matching.

Most deployments use a combination: Asterisk AMD for fast detection (< 2 seconds), AI-side as a backup for cases AMD misses.

---

### 10. What's the cost of adding AI to VICIdial?

The total cost has three components:

| Component | Typical Cost | Notes |
|-----------|-------------|-------|
| AI platform | $0.05 - $0.15/min | Depends on provider, volume, and model choices |
| VICIdial changes | $0 | Uses existing infrastructure |
| Telephony (SIP relay) | $0 - $0.008/min | $0 with direct SIP registration; more with trunk relay |

**ROI calculation**: If your human agents cost $12/hour ($0.20/min) and the AI handles the first 60 seconds of each call (filtering out ~70% of calls as unqualified), your effective human agent cost per qualified conversation drops significantly. The AI cost is offset by reduced agent hours and higher transfer quality.

At 50,000 AI-handled minutes per month at $0.10/min = $5,000/month. If this replaces 3-4 full-time agents who were previously handling unqualified calls ($8,000-10,000/month in wages), the ROI is immediate.

---

### 11. Can I use this for inbound calls too?

**Yes.** While this guide focuses on outbound (VICIdial dialer → lead → AI), the same SIP extension approach works for inbound:

- Configure VICIdial's inbound DID routing to send calls to the AI extension
- The AI answers, qualifies the caller, and transfers to an agent if appropriate
- Common use case: after-hours AI receptionist that handles calls when human agents aren't available

---

### 12. What if my AI platform doesn't support SIP registration?

If your AI platform only supports WebSocket or HTTP connections (not native SIP), you have options:

1. **SIP-to-WebSocket bridge**: A lightweight bridge (Python, Go, or Node.js) registers as the SIP extension and bridges audio to your AI platform via WebSocket. This is a common pattern — [Klariqo](https://klariqo.com) uses this approach with a Python-based SIP bridge.

2. **SIP trunk relay**: Use Twilio, Telnyx, or similar as an intermediary. They handle SIP on the VICIdial side and provide WebSocket/API access on the AI side. Adds latency and cost, but simplifies the integration.

3. **FreeSWITCH bridge**: If you have FreeSWITCH expertise, it can act as a SIP-to-WebSocket bridge with extensive protocol support.

---

### 13. Is this compliant with TCPA / FCC regulations?

The SIP integration itself is technology-neutral from a compliance perspective. However, using AI for outbound calling has regulatory considerations:

- **FCC AI disclosure**: As of 2024, the FCC requires disclosure when AI or prerecorded voices are used in calls. Check current rules for your use case.
- **TCPA consent**: Prior express consent requirements apply the same whether a human or AI makes the call
- **State regulations**: Some US states have additional rules about AI-generated voice calls
- **DNC compliance**: Your VICIdial DNC scrubbing works the same regardless of whether calls go to human or AI agents

Consult a telecom compliance attorney for your specific use case. This guide covers the technical integration, not legal compliance.

---

### 14. How do I monitor AI agent performance?

VICIdial's existing reporting works for AI agents:

- **Real-Time Report**: Shows AI agent status (INCALL, READY, etc.)
- **Agent Performance Report**: Calls handled, talk time, dispositions
- **Campaign Reports**: Overall metrics including AI-handled calls

Additionally, the AI platform typically provides its own analytics:
- Conversation transcripts
- Qualification rates
- Average call duration
- Transfer success rates
- Common caller objections and topics

The combination gives you both the telephony view (VICIdial) and the conversation intelligence view (AI platform).

---

### 15. Can I run multiple AI agents on the same VICIdial server?

**Yes.** Create multiple Phone extensions and Remote Agents, each pointing to a different AI instance. Use cases:

- **Different campaigns**: Each campaign gets its own AI agent with campaign-specific scripts
- **A/B testing**: Run two AI agents with different prompts on the same campaign and compare transfer rates
- **Load distribution**: Spread calls across multiple AI agent instances for redundancy

VICIdial treats each extension independently — it doesn't know or care that they're all AI agents.

---

## Still Have Questions?

- Check the [Troubleshooting Guide](docs/troubleshooting.md) for technical issues
- Review the [detailed documentation](docs/) for deeper dives
- For VICIdial-specific questions: [VICIdial Community Forum](https://www.vicidial.org/VICIDIALforum/)
- For Klariqo-specific integration help: hello@klariqo.com
