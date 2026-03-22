# Troubleshooting Guide

> Common issues when integrating AI agents with VICIdial via SIP extension registration, and how to fix them.

---

## Quick Diagnostics

Before diving into specific issues, gather this baseline information:

```bash
# On the VICIdial/Asterisk server:

# 1. Check if the AI extension is registered
asterisk -rx "sip show peers" | grep 8001

# 2. Check active channels
asterisk -rx "core show channels"

# 3. Enable SIP debug (WARNING: verbose output)
asterisk -rx "sip set debug on"

# 4. Check Asterisk logs
tail -f /var/log/asterisk/full

# 5. Check VICIdial logs
tail -f /var/log/astguiclient/carrier_log*
```

---

## Issue: Extension Shows as "Unavailable" or "UNKNOWN"

**Symptom:** In VICIdial Admin → Phones, the AI extension shows as unavailable. `sip show peers` shows `UNKNOWN` status.

**Causes and Fixes:**

### 1. Registration never reached Asterisk

Check if the REGISTER message is arriving:

```bash
# Enable SIP debug and watch for REGISTER
asterisk -rx "sip set debug on"
# Look for: "REGISTER sip:... From: <sip:8001@...>"
```

If no REGISTER appears:
- **Firewall**: Ensure UDP port 5060 is open from the AI server to the VICIdial server
- **IP address**: Verify the AI agent is pointing at the correct VICIdial server IP
- **Network path**: Try `ping` and `traceroute` from AI server to VICIdial server

### 2. Authentication failure

If you see a REGISTER followed by `403 Forbidden` or repeated `401 Unauthorized`:

- **Username mismatch**: The SIP username in the REGISTER must exactly match the extension number in VICIdial (e.g., `8001`)
- **Password mismatch**: The registration password must match what's configured in VICIdial Admin → Phones
- **Realm mismatch**: Some SIP stacks require the authentication realm to match. Check `asterisk -rx "sip show peer 8001"` for the configured realm

### 3. VICIdial hasn't synced the config

After creating a new Phone in VICIdial Admin, the configuration needs to be written to Asterisk:

```bash
# Force a SIP reload
asterisk -rx "sip reload"

# Or restart the VICIdial keepalive process
/usr/share/astguiclient/ADMIN_keepalive_ALL.pl --force
```

### 4. Peer definition missing

```bash
# Check if the peer exists in Asterisk
asterisk -rx "sip show peer 8001"

# If "Peer 8001 not found", the VICIdial config hasn't been applied
# Check the extensions_additional.conf or sip_additional.conf files
```

---

## Issue: No Audio on Calls

**Symptom:** The SIP call connects (you see a 200 OK), but one or both sides hear silence.

### 1. RTP port blocked by firewall

This is the most common cause of no audio. SIP signaling (port 5060) may work while RTP audio (ports 10000-20000) is blocked.

```bash
# On the AI server, check if RTP ports are reachable
# from the VICIdial server's perspective:
# Try sending a UDP packet to the AI server's RTP port
nc -u -z ai-server-ip 16000

# On VICIdial, check what port Asterisk is trying to send to:
asterisk -rx "rtp set debug on"
# Look for: "Sent RTP packet to ai-server-ip:XXXXX"
```

**Fix**: Open the RTP port range (e.g., UDP 16000-16200) in both directions between the VICIdial server and AI server.

### 2. NAT / externaddr misconfiguration

If either server is behind NAT, the SDP (Session Description Protocol) in the SIP messages may contain private IP addresses instead of public ones.

```bash
# Check the SDP in the INVITE/200 OK
asterisk -rx "sip set debug on"
# Look for "c=IN IP4 X.X.X.X" — this should be the public/reachable IP
```

**Fixes:**
- On VICIdial: Set `externaddr=your-public-ip` in `sip.conf` for the AI peer
- On the AI agent: Configure your SIP stack to use your public IP in the SDP Contact and Connection fields
- Set `nat=yes` on the SIP peer in VICIdial

### 3. Codec mismatch

If the AI agent doesn't support mulaw (PCMU), the call will connect but audio won't decode properly.

```bash
# Check negotiated codec
asterisk -rx "core show channel SIP/8001-xxxxx"
# Look for "NativeFormats" and "WriteFormat" — should show "ulaw"
```

**Fix**: Ensure the AI agent advertises `PCMU/8000` (payload type 0) in its SDP answer.

### 4. Asymmetric RTP / one-way audio

If you hear audio in one direction but not the other:

- **AI hears lead but lead doesn't hear AI**: AI is receiving RTP but not sending back correctly. Check TTS output encoding and RTP packetization.
- **Lead hears AI but AI doesn't hear lead**: AI isn't receiving RTP from Asterisk. Check firewall rules for inbound UDP to the AI server.

```bash
# Enable RTP debug on Asterisk
asterisk -rx "rtp set debug on"
# Watch for packets flowing in both directions
```

---

## Issue: Calls Connect but AI Doesn't Respond

**Symptom:** Audio flows (you can hear the lead), but the AI never speaks.

### 1. STT not receiving audio

The most common cause is a format mismatch between what Asterisk sends and what the STT engine expects:

- Asterisk sends **mulaw 8kHz** RTP
- Most STT engines expect **linear PCM 16kHz**
- The AI agent must transcode: mulaw 8kHz → linear PCM → resample to 16kHz

**Debug**: Log the raw audio bytes arriving at the AI agent. If they're all zeros or the byte pattern looks wrong, the issue is in receiving RTP. If audio arrives correctly but STT returns empty transcripts, the issue is in the transcode/format.

### 2. STT connection failed

The AI agent may have failed to establish a connection to the STT service:

- Check STT API key validity
- Check network connectivity from the AI server to the STT endpoint
- Check for rate limits on the STT API

### 3. LLM timeout or error

If STT works but the LLM fails to respond:

- Check LLM API key and endpoint
- Check for rate limits (common with Groq, OpenAI during peak hours)
- Set a timeout (e.g., 5 seconds) on LLM calls and have a fallback response ("I'm sorry, could you repeat that?")

### 4. TTS output not being sent

If the LLM generates a response but the caller never hears it:

- Check TTS API connectivity and credentials
- Verify TTS output format matches what you're packaging into RTP packets
- Ensure RTP sequence numbers and timestamps are incrementing correctly

---

## Issue: Transfer Fails

**Symptom:** The AI attempts to transfer a qualified lead to a human agent, but the transfer doesn't go through.

### 1. VICIdial API credentials

If using the Agent API for transfers:

```bash
# Test the API directly
curl "http://vicidial-ip/agc/api.php?function=transfer_conference&agent_user=ai-user&value=AXFER&phone_number=8101&source=test&user=api-user&pass=api-pass"
```

Common issues:
- API user doesn't have transfer permissions
- Agent user is not in an active call state
- The target extension (human agent) is not available

### 2. Dialplan routing

If using SIP REFER for transfers, Asterisk needs a dialplan context that can route to the target:

```bash
# Check the dialplan context for the AI extension
asterisk -rx "sip show peer 8001" | grep context

# Verify the context can route to the target number
asterisk -rx "dialplan show context-name"
```

### 3. Transfer to external number

If transferring to a PSTN number (not an internal extension), the call must route through a trunk:

- The dialplan context must have outbound trunk routes
- The trunk must be configured and functional
- Caller ID may need to be set for outbound PSTN calls

---

## Issue: Registration Drops Frequently

**Symptom:** The AI extension registers successfully but goes offline every few minutes, causing missed calls.

### 1. Re-registration interval too long

If the re-registration interval is longer than the NAT timeout, the registration will lapse:

```
# Typical NAT timeouts:
# - Home routers: 30-120 seconds
# - Enterprise firewalls: 60-300 seconds
# - Cloud NAT (AWS, GCP): 350 seconds
```

**Fix**: Set re-registration interval to 60 seconds (well under most NAT timeouts).

### 2. No NAT keepalives

Even with re-registration, some NAT devices close pinholes aggressively:

**Fix**: Enable SIP OPTIONS keepalives every 20-30 seconds. Most SIP libraries support this natively (e.g., `qualify=yes` on the Asterisk side sends OPTIONS pings).

### 3. DNS resolution changes

If the AI agent resolves the VICIdial server by hostname and the DNS changes:

**Fix**: Use IP addresses instead of hostnames, or set a short DNS TTL.

### 4. Server restarts

VICIdial runs scheduled processes that may restart Asterisk:

```bash
# Check crontab for Asterisk restarts
crontab -l | grep asterisk

# Check if VICIdial's keepalive processes are causing restarts
ps aux | grep keepalive
```

**Fix**: The AI agent should handle registration failures gracefully — retry immediately with exponential backoff.

---

## Issue: Audio Quality Issues

### Jitter / Choppy Audio

**Symptom:** Audio sounds robotic, choppy, or has gaps.

- **Network jitter**: Implement a jitter buffer on the AI side (20-60ms buffer recommended)
- **Packet loss**: Check network quality between servers (`ping -c 100` and look at loss percentage)
- **CPU overload**: If the AI server is overloaded, RTP packets may be sent late

### Echo

**Symptom:** The lead hears their own voice echoed back.

- **Acoustic echo**: The AI's TTS output is being picked up by the STT engine. Implement echo cancellation or mute STT while TTS is playing
- **Network echo**: Check for impedance mismatches on the PSTN side (not fixable on your end)

### Robotic / Metallic Voice

**Symptom:** The AI's voice sounds distorted or metallic.

- **Codec mismatch**: Verify mulaw encoding is correct (companding law must match — mulaw, not alaw)
- **Sample rate mismatch**: TTS output at 16/24/44.1kHz must be properly downsampled to 8kHz before mulaw encoding
- **Bitrate issues**: Mulaw is 8-bit — don't try to send 16-bit PCM as mulaw

---

## Issue: VICIdial Shows Agent as Inactive

**Symptom:** Even though the AI extension is registered, VICIdial doesn't route calls to it.

### 1. Remote Agent not ACTIVE

Check Admin → Remote Agents — ensure status is `ACTIVE` and the campaign is assigned.

### 2. Campaign not running

The campaign must be in an active dialing state. Check Campaigns → your campaign → Active is `Y`.

### 3. Agent user paused

VICIdial may have paused the agent user. Check the agent's status in the real-time report.

### 4. Number of lines set to 0

If the Remote Agent's "Number of Lines" is 0, no calls will be routed. Set it to at least 1.

---

## Diagnostic Tools

### Asterisk CLI Commands

```bash
# SIP registration status
asterisk -rx "sip show peers"
asterisk -rx "sip show peer 8001"

# Active calls
asterisk -rx "core show channels"
asterisk -rx "core show channel SIP/8001-xxxxx"

# SIP debug (verbose — disable after debugging)
asterisk -rx "sip set debug on"
asterisk -rx "sip set debug off"

# RTP debug
asterisk -rx "rtp set debug on"
asterisk -rx "rtp set debug off"

# Reload SIP config
asterisk -rx "sip reload"

# Check dialplan
asterisk -rx "dialplan show default"
```

### Network Tools

```bash
# Test SIP port connectivity
nc -u -z vicidial-ip 5060

# Test RTP port range
for port in $(seq 16000 16010); do nc -u -z ai-server-ip $port && echo "$port open"; done

# Packet capture (on VICIdial server)
tcpdump -i eth0 host ai-server-ip -w /tmp/sip-debug.pcap

# Analyze with tshark or Wireshark
tshark -r /tmp/sip-debug.pcap -Y "sip || rtp"
```

### VICIdial Logs

```bash
# Carrier logs
tail -f /var/log/astguiclient/carrier_log*

# VICIdial process logs
tail -f /var/log/astguiclient/process_log*

# Agent logs
tail -f /var/log/astguiclient/agent_log*
```

---

## Getting Help

If you've exhausted this troubleshooting guide:

1. **VICIdial community**: [forum.vicidial.com](https://www.vicidial.org/VICIDIALforum/) — the community is active and helpful for VICIdial-specific issues
2. **Asterisk documentation**: [wiki.asterisk.org](https://wiki.asterisk.org/) — for SIP and RTP configuration details
3. **Klariqo support**: If you're using Klariqo as your AI platform, contact hello@klariqo.com — we've debugged hundreds of VICIdial SIP integrations

---

## Next Steps

- [SIP Registration Setup](sip-registration-setup.md) — initial configuration
- [Call Flow Architecture](call-flow-architecture.md) — understand the full pipeline
- [Latency Comparison](latency-comparison.md) — performance benchmarks
