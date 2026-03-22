# SIP Registration Setup: Step-by-Step Guide

> How to configure VICIdial to accept an AI agent as a SIP extension and route calls to it.

---

## Prerequisites

Before you begin, ensure you have:

- [ ] **VICIdial server** with admin access (version 2.14+ recommended)
- [ ] **Asterisk** using **chan_sip** (check with `asterisk -rx "sip show peers"` — if this command works, you're on chan_sip)
- [ ] **AI platform** with SIP registration capability (e.g., [Klariqo](https://klariqo.com), or your own SIP-capable application)
- [ ] **Network connectivity** between the AI server and VICIdial server (direct IP, VPN, or public internet with firewall rules)
- [ ] **Firewall rules** allowing:
  - SIP signaling: UDP/TCP port **5060** (or your configured SIP port)
  - RTP audio: UDP ports **10000-20000** (or your configured RTP range)

---

## Step 1: Create a Phone Extension in VICIdial Admin

The Phone extension is the SIP identity that the AI agent will register as.

1. Log in to VICIdial Admin: `http://your-vicidial-server/vicidial/admin.php`
2. Navigate to **Admin → Phones → Add a New Phone**
3. Fill in the following fields:

| Field | Value | Notes |
|-------|-------|-------|
| **Extension** | `8001` | Any unused extension number. Some setups use descriptive names like `klariqo-ai` |
| **Dialplan Number** | `8001` | Usually matches the extension |
| **Voicemail ID** | `8001` | Usually matches the extension |
| **Phone Login** | `ai-agent-1` | The login name for this phone |
| **Phone Password** | (strong password) | This is the SIP authentication password — treat it like a credential |
| **Server IP** | Your VICIdial server IP | The IP of the Asterisk server |
| **Protocol** | `SIP` | Must be SIP |
| **Registration Password** | Same as Phone Password | Used for SIP REGISTER authentication |
| **Active** | `Y` | Must be active to receive calls |

4. Click **Submit** to create the phone

**Important:** After creating the phone, VICIdial writes the SIP peer configuration into Asterisk. You may need to reload the SIP configuration:

```bash
# On the VICIdial server
asterisk -rx "sip reload"
```

### Verify the SIP Peer Exists

```bash
asterisk -rx "sip show peer 8001"
```

You should see output showing the peer configuration. If the peer doesn't exist, check that VICIdial's `ADMIN_keepalive_ALL.pl` cron job has run (it syncs config to Asterisk).

---

## Step 2: Create a Remote Agent

The Remote Agent tells VICIdial to route campaign calls to this extension.

1. Navigate to **Admin → Remote Agents → Add a New Remote Agent**
2. Fill in the following fields:

| Field | Value | Notes |
|-------|-------|-------|
| **User ID** | Create a new user first (Admin → Users → Add) | The agent "identity" for reporting |
| **Server IP** | Your VICIdial server IP | Must match |
| **Extension** | `8001` | Must match the Phone extension from Step 1 |
| **Status** | `ACTIVE` | Set to ACTIVE to receive calls |
| **Number of Lines** | `1` - `20` | How many concurrent calls the AI can handle. Start with 1 for testing. |
| **Campaign** | Your target campaign | Which campaign routes calls to this agent |

3. Click **Submit**

### Understanding "Number of Lines"

This controls concurrency. If you set it to `5`, VICIdial will route up to 5 simultaneous calls to this extension. The AI platform must be able to handle this many concurrent calls.

Start with **1 line** for testing. Once you've verified calls are working correctly, increase gradually. Most AI platforms can handle 10-50+ concurrent calls per extension, but the bottleneck is usually your STT/LLM/TTS pipeline capacity.

---

## Step 3: Configure SIP Registration on the AI Side

The AI platform needs to send a SIP REGISTER request to your VICIdial's Asterisk server. The exact configuration depends on your AI platform, but here's what it needs:

### Registration Parameters

| Parameter | Value |
|-----------|-------|
| **SIP Server** | Your VICIdial server IP (e.g., `192.168.1.100`) |
| **SIP Port** | `5060` (default) |
| **Username** | Extension number (e.g., `8001`) |
| **Password** | The Phone Password / Registration Password from Step 1 |
| **Transport** | UDP (chan_sip default) |
| **Re-registration Interval** | 60-120 seconds |

### What the REGISTER Looks Like

```
REGISTER sip:192.168.1.100 SIP/2.0
Via: SIP/2.0/UDP ai-server-ip:5060;branch=z9hG4bK-xxx
From: <sip:8001@192.168.1.100>;tag=xxx
To: <sip:8001@192.168.1.100>
Contact: <sip:8001@ai-server-ip:5060>
Expires: 120
Call-ID: xxx
CSeq: 1 REGISTER
Content-Length: 0
```

Asterisk will respond with `401 Unauthorized` including a `WWW-Authenticate` header. The AI platform must respond with proper digest authentication credentials.

### Using Klariqo

If you're using [Klariqo](https://klariqo.com) as your AI platform, you provide three things:

1. Your VICIdial server IP
2. The extension number
3. The extension password

Klariqo's SIP bridge handles registration, re-registration, RTP audio, and codec negotiation automatically. No SIP expertise required on your end.

### Building Your Own

If you're building your own AI agent with SIP capability, consider these libraries:

- **Python**: `pjsua2` (PJSIP), `aioSIP`
- **Node.js**: `drachtio-srf`, `jsSIP`
- **Go**: `pion/sdp` + raw SIP handling
- **C/C++**: `pjsip`, `oSIP`

You'll need to implement:
- SIP REGISTER with digest authentication
- SIP INVITE handling (receive and answer incoming calls)
- RTP send/receive (mulaw 8kHz, 20ms packetization)
- SIP BYE handling (hangup)

---

## Step 4: Verify Registration

### On VICIdial

1. Go to **Admin → Phones**
2. Find your AI phone extension
3. The status should show as **Registered** or **Reachable**

### On the Asterisk CLI

```bash
# Check SIP peer status
asterisk -rx "sip show peers" | grep 8001

# Expected output:
# 8001/8001     ai-server-ip    D   N      5060     OK (35 ms)

# Detailed peer info
asterisk -rx "sip show peer 8001"

# Look for:
#   Status       : OK (35 ms)
#   Addr->IP     : ai-server-ip:5060
```

If the status shows `UNKNOWN` or `Unmonitored`, the registration hasn't been received. Check:

1. Network connectivity: Can the AI server reach the VICIdial server on port 5060?
2. Firewall: Is UDP 5060 open in both directions?
3. Credentials: Do the username and password match exactly?
4. SIP debug: Run `asterisk -rx "sip set debug on"` and watch for REGISTER messages

---

## Step 5: Route Calls to the AI Agent

### Assign to a Campaign

1. Navigate to **Admin → Campaigns → [Your Campaign]**
2. Make sure the campaign is **Active**
3. The Remote Agent you created in Step 2 is already assigned to this campaign

### Test with a Manual Call

Before using the predictive dialer, test with a manual call:

1. Log in as a regular VICIdial agent
2. Place a manual call to a test number
3. Transfer the call to extension `8001`
4. Verify the AI agent receives the SIP INVITE, answers, and audio flows correctly

### Enable Predictive Dialing

Once manual testing works:

1. Set the Remote Agent status to `ACTIVE`
2. Start the campaign
3. VICIdial's dialer will begin routing answered calls to the AI agent

### Monitoring

Watch the AI agent's activity in VICIdial:

- **Real-Time Report**: Shows the AI agent's status (INCALL, READY, PAUSED)
- **Agent Performance**: Shows calls handled, talk time, and disposition
- **Campaign Performance**: Shows overall campaign metrics including AI-handled calls

---

## Important Notes

### chan_sip vs res_pjsip

VICIdial predominantly uses **chan_sip**. If you're on a newer Asterisk installation using **res_pjsip**:

- The SIP configuration lives in `pjsip.conf` instead of `sip.conf`
- The AOR (Address of Record) name must **exactly match** the SIP username in the REGISTER request
- `qualify=yes` equivalent is `qualify_frequency=60` in the endpoint definition
- Use `pjsip show endpoints` instead of `sip show peers`

If you're unsure which SIP driver you're using:

```bash
# Check loaded modules
asterisk -rx "module show like sip"

# If you see chan_sip.so → you're on chan_sip
# If you see res_pjsip.so → you're on res_pjsip
# Both can be loaded simultaneously (but this is unusual on VICIdial)
```

### NAT / Firewall Configuration

If the AI server is behind NAT (common in cloud deployments):

1. **On the AI side**: Set `externaddr` or equivalent in your SIP stack to your public IP
2. **On VICIdial**: Ensure `nat=yes` is set for the SIP peer (VICIdial usually sets this by default)
3. **RTP ports**: Forward the RTP port range (e.g., UDP 16000-16200) from your NAT device to the AI server
4. **Keep-alives**: Send SIP OPTIONS or re-REGISTER frequently (every 30-60s) to keep NAT pinholes open

### RTP Port Range

Use a **separate RTP port range** for the AI agent to avoid conflicts with Asterisk's own RTP ports:

| Component | Suggested Port Range |
|-----------|---------------------|
| Asterisk (existing) | 10000-10200 |
| AI Agent | 16000-16200 |

Configure this in `rtp.conf` on VICIdial (for Asterisk's range) and in your AI platform's configuration.

### Codec Configuration

VICIdial/Asterisk uses **G.711 mulaw** (PCMU) by default. Ensure your AI platform:

- Advertises `PCMU` (payload type 0) in its SDP answer
- Sends and receives mulaw-encoded RTP at 8kHz
- Uses 20ms packetization (160 bytes per RTP packet, 50 packets per second)

If your AI platform's STT engine expects a different format (e.g., linear PCM 16kHz), you'll need to transcode on the AI side:
- mulaw 8kHz → linear PCM 16kHz (for STT input)
- TTS output → mulaw 8kHz (for RTP output to Asterisk)

---

## Next Steps

- [Call Flow Architecture](call-flow-architecture.md) — understand the full call lifecycle
- [Troubleshooting](troubleshooting.md) — when things don't work
- [Latency Comparison](latency-comparison.md) — why direct SIP registration is fastest
