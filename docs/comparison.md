# SBL vs NDI vs WHIP vs SRT — Protocol Comparison

## Summary Table

| | **SBL** | **NDI** | **WHIP** | **SRT** | **RTMP** |
|---|---|---|---|---|---|
| **Latency (LAN)** | <300ms | 100–200ms | 100–500ms | 200ms–2s | 1–5s |
| **Latency (WAN)** | Not designed for WAN | 200–500ms | 100–500ms | 200ms–5s | 1–10s |
| **Bandwidth** | 2–50 Mbps | 100–500 Mbps | Variable | Variable | Variable |
| **Video codecs** | H.264, H.265, AV1 | SpeedHQ (proprietary), H.264 | H.264, H.265, AV1, VP8/9 | H.264, H.265 | H.264, H.265, AV1 |
| **Audio** | Opus, AAC | PCM (lossless) | Opus | AAC, MP3 | AAC, MP3 |
| **Discovery** | mDNS + QR | mDNS | Manual URL | Manual | Manual |
| **Tally** | ✅ Built-in | ✅ Built-in | ❌ None | ❌ None | ❌ None |
| **Scene metadata** | ✅ Built-in | ✅ Built-in | ❌ None | ❌ None | ❌ None |
| **Transport control** | ✅ Built-in | Partial | ❌ None | ❌ None | ❌ None |
| **Encryption** | DTLS-SRTP (mandatory) | Optional | DTLS-SRTP | AES-128/256 | None (RTMPS optional) |
| **License** | **Apache 2.0** | Proprietary SDK | IETF Open | MPL-2.0 | Proprietary |
| **WiFi-optimized** | ✅ Yes | ⚠️ High bandwidth | ✅ Yes | ⚠️ No | ❌ No |
| **Zero-config** | ✅ mDNS + QR | ✅ mDNS | ❌ No | ❌ No | ❌ No |
| **Mobile-first** | ✅ Yes | ❌ No | ⚠️ Partial | ❌ No | ❌ No |

---

## Deep Dive

### NDI (Network Device Interface)

Developed by NewTek (now Vizrt), NDI is the dominant standard for professional video over IP in broadcast facilities. It is well-proven in studio environments with gigabit Ethernet.

**Strengths:**
- Industry-standard, supported by hundreds of devices and software tools
- Lossless/near-lossless quality with SpeedHQ codec
- Built-in tally and metadata
- Very mature ecosystem

**Weaknesses:**
- **High bandwidth** (100–500 Mbps) makes it impractical on WiFi
- SpeedHQ is a proprietary codec — NDI requires Vizrt SDK for full support
- SDK is free for software, but some uses require licensing
- Designed for studio LAN, not mobile or wireless workflows
- No native mobile support

**When to choose NDI over SBL:**
- Studio environment with gigabit Ethernet
- Connecting professional hardware that already supports NDI
- Maximum quality is required (lossless)

---

### WHIP (WebRTC-HTTP Ingestion Protocol)

WHIP (RFC draft) standardizes how WebRTC streams are pushed to a server using a simple HTTP POST for signaling. It is gaining adoption for cloud ingest (Cloudflare Stream, Mux, Livekit).

**Strengths:**
- Fully open IETF standard
- Low latency, WebRTC-based
- Growing support in broadcast tools

**Weaknesses:**
- **No production metadata** — no tally, no scene info, no transport control
- **No discovery** — you need to know the server URL
- Designed for cloud ingest, not local LAN production
- Requires HTTP server for signaling (complex for mobile cameras)

**Relationship to SBL:**
SBL uses WebRTC's media transport (the same underlying stack as WHIP) but replaces HTTP/SDP signaling with SBL's simpler JSON signaling and adds the production metadata layer that WHIP explicitly omits.

SAMBA supports WHIP *in addition to* SBL, so WHIP devices (cameras, encoders) can also connect to SAMBA. SBL is recommended when you need tally and zero-config pairing.

---

### SRT (Secure Reliable Transport)

Developed by Haivision and open-sourced (MPL-2.0), SRT is designed for reliable transmission over unreliable networks (internet, wireless with packet loss). It is a retransmission-based protocol with configurable latency.

**Strengths:**
- Excellent for WAN/internet transmission
- Handles packet loss gracefully (ARQ retransmission)
- Good ecosystem, widely supported
- Open source (MPL-2.0)

**Weaknesses:**
- **Minimum latency ~200ms** (4× the retransmission window is recommended)
- No production metadata (no tally, no scene info)
- No discovery mechanism
- Adds latency proportional to network jitter — on a stable LAN, this overhead is wasted

**When to choose SRT over SBL:**
- Sending video over the internet or unreliable networks
- Connecting to remote locations or cloud ingest
- When packet loss > 0.1% is expected

---

## Why SBL?

SBL is not trying to replace NDI or SRT. It fills a gap: **local WiFi production with production metadata and zero-config pairing**, specifically optimized for mobile cameras (phones, tablets) connecting to production software.

```
Internet / WAN          →  SRT, WHIP
Studio Gigabit LAN      →  NDI
Local WiFi + Mobile     →  SBL  ←
WebRTC Cloud Ingest     →  WHIP
```

SBL's open license (Apache 2.0) means any developer can implement it without SDK agreements, royalties, or registration — the same model that made SRT successful.
