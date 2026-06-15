# SBL Protocol Specification — v3

**Status:** Draft  
**Authors:** BlackMagicBox  
**License:** Apache 2.0

---

## 1. Overview

SBL (Sub-second Broadcast Link) is a lightweight UDP-based protocol for real-time video and audio transmission between mobile cameras and the SAMBA engine in professional live production environments.

Unlike WebRTC, SBL uses a thin custom header directly over UDP, eliminating ICE negotiation, DTLS handshake overhead, and SRTP framing. This enables glass-to-glass latency of 20–75ms on local networks.

### 1.1 Design Goals

- **Latency:** Glass-to-glass 20–75ms on 802.11ac/Wi-Fi 6 LAN; <300ms over WAN relay
- **Simplicity:** Handshake in 2 round trips (Hello → HelloAck), no SDP, no ICE
- **Resilience:** Reed-Solomon RS(10,3) FEC recovers up to 3 packet losses per group
- **WAN reach:** STUN-based public IP discovery + VPS registry for cross-network connections
- **Bidirectional:** Camera sends video+audio; engine sends talkback audio return (AudioReturn)
- **Open:** Apache 2.0, no SDK required, no registration

### 1.2 Non-Goals

- General-purpose internet transport (use SRT or WHIP for public streaming)
- Reliable file transfer
- Recording (handled at the application layer)
- Browser/WebRTC interoperability

---

## 2. Transport Stack

```
┌───────────────────────────────────────────────┐
│        Application (SAMBA, SAMBA Air)         │
├───────────────────────────────────────────────┤
│  SBL v3 framing (32-byte header, see §3)      │
├───────────────────────────────────────────────┤
│  Optional: AES-256-GCM (ECDH key exchange)    │
├───────────────────────────────────────────────┤
│  UDP (single port, bidirectional)             │
└───────────────────────────────────────────────┘
```

- **Default port:** 8890
- **MTU target:** 1200 bytes payload (safe under 1500-byte Ethernet MTU with headers)
- **Multiplexing:** Multiple streams (video, audio, alpha, talkback, metadata, control) share one port via the `streamID` field
- No TCP fallback in v3 — use SRT if reliability over WAN is required

---

## 3. Packet Format

### 3.1 Packet Header — 32 bytes (packed, network byte order)

```
Offset  Size  Field
──────  ────  ─────
 0       3    magic          — ASCII "SBL" (0x53 0x42 0x4C)
 3       1    version        — 3
 4       1    packetType     — SblPacketType (see §3.2)
 5       1    streamID       — SblStreamID (see §3.3)
 6       2    pktSeqNum      — global per-packet sequence (uint16, wraps)
 8       4    frameSeqNum    — per-frame monotonic counter (uint32)
12       2    fragmentIdx    — 0-based fragment index within frame
14       2    fragmentTotal  — total fragments in this frame
16       8    timestamp      — presentation timestamp in microseconds (uint64)
24       2    payloadLen     — bytes of payload following this header (uint16)
26       2    flags          — SblFrameFlags bitmask (see §3.5)
28       4    authTagPartial — first 4 bytes of AES-256-GCM auth tag; 0 = unencrypted
```

Receivers MUST validate:
1. `magic == "SBL"`
2. `version == 3`
3. `payloadLen + 32 == actual UDP datagram size`

### 3.2 Packet Types

| Value | Name       | Description |
|-------|------------|-------------|
| 0     | Data       | Media data fragment |
| 1     | FEC        | RS parity shard |
| 2     | Hello      | Handshake initiation |
| 3     | HelloAck   | Handshake acknowledgement |
| 4     | Keepalive  | Heartbeat (every 1 s) |
| 5     | Goodbye    | Graceful disconnect |
| 6     | Feedback   | Congestion / loss feedback (sender → ABR) |

### 3.3 Stream IDs

| Value | Name        | Codec       | Direction     |
|-------|-------------|-------------|---------------|
| 0     | VideoColor  | H.264 / H.265 / AV1 | Source → Engine |
| 1     | Audio       | Opus (up to 8ch)    | Source → Engine |
| 2     | Alpha       | Grayscale mask      | Source → Engine |
| 3     | AudioReturn | Opus mono 48kHz     | Engine → Source |
| 4     | Metadata    | Raw JSON            | Bidirectional |
| 5     | Control     | Raw JSON            | Engine → Source |

### 3.4 Codec IDs (used in SblFrameHeader.codec)

| Value | Codec |
|-------|-------|
| 0x01  | AV1 |
| 0x02  | H.265 |
| 0x03  | H.264 (maximum compatibility) |
| 0x10  | Opus |
| 0xFF  | Raw (metadata / control) |

### 3.5 Frame Flags (bits in the `flags` field)

| Bit | Flag           | Meaning |
|-----|----------------|---------|
| 0   | KEYFRAME       | IDR / I-frame |
| 1   | ENCRYPTED      | Payload is AES-256-GCM encrypted |
| 2   | FEC            | RS parity packets follow this frame group |
| 3   | HDR10          | HDR10 metadata present |
| 4   | HLG            | HLG transfer function |
| 5   | ALPHA          | Alpha channel frame |
| 6   | BT2020         | BT.2020 color primaries |
| 7   | 10BIT          | 10-bit depth (P010) |
| 8   | EOS            | End of stream |

---

## 4. Frame Header — 32 bytes

Prepended to the **first fragment's payload only** (fragmentIdx == 0).

```
Offset  Size  Field
──────  ────  ─────
 0       1    codec          — SblCodec (see §3.4)
 1       1    channels       — audio channels (0 for video)
 2       2    width          — frame width in pixels
 4       2    height         — frame height in pixels
 6       2    fpsNum         — FPS numerator
 8       2    fpsDen         — FPS denominator
10       4    flags          — SblFrameFlags bitfield
14       4    totalFrameSize — full compressed frame size in bytes
18       2    colorPrimaries — ITU-T H.273
20       2    transferFunc   — ITU-T H.273
22       2    matrixCoeff    — ITU-T H.273
24       4    sampleRate     — audio sample rate Hz (0 for video)
28       4    reserved       — set to 0
```

---

## 5. Handshake

### 5.1 Connection Flow

```
Source (SAMBA Air)                    Receiver (SAMBA engine)
       │                                        │
       │──── UDP Hello (pktType=2) ────────────▶│
       │◀─── UDP HelloAck (pktType=3) ──────────│
       │                                        │
       │════ Data / FEC / Audio packets ════════▶│
       │◀═══ AudioReturn / Feedback / Keepalive ─│
       │                                        │
       │──── Goodbye (pktType=5) ───────────────▶│
```

### 5.2 Hello Payload (105 bytes)

```
Offset  Size  Field
──────  ────  ─────
 0       1    version           — 3
 1      64    sourceName        — UTF-8, null-padded
65       1    codecCapabilities — bitmask: bit0=H.264 bit1=H.265 bit2=AV1
66       1    flags             — bit0=hasAlpha bit1=hasAudio bit2=hasCCU
67       1    wantEncryption    — 1 = request AES-256-GCM
68       1    reserved
69       4    maxBandwidthMbps  — sender-side cap (uint32)
73      32    ecdhPublicKey     — P-256 ECDH public key
```

### 5.3 HelloAck Payload (100 bytes)

```
Offset  Size  Field
──────  ────  ─────
 0       1    version           — 3
 1      64    receiverName      — UTF-8, null-padded
65       1    acceptedCodec     — SblCodec value
66       1    encryptionAccepted — 1 = both sides will encrypt
67       2    reserved
69      32    ecdhPublicKey     — receiver's P-256 ECDH public key
```

If `encryptionAccepted == 1`, both sides derive a shared AES-256 key via ECDH and encrypt all subsequent Data/FEC payloads. The first 4 bytes of each AES-GCM auth tag are stored in `authTagPartial`.

---

## 6. Forward Error Correction (RS FEC)

SBL uses Reed-Solomon RS(k=10, m=3) with a Cauchy matrix.

- Every **10 consecutive data packets** (one group) produce **3 parity shards**
- The receiver can recover any 3 missing packets from a group given the remaining 10 shards
- Partial groups (last group of a frame with < 10 packets) are encoded with `k = actual count`
- FEC packets use `packetType=1` and carry the same `frameSeqNum` and `streamID` as the data group
- The `flags` field in data packets has bit 2 (`SBL_FLAG_FEC`) set when parity follows

**Constants:**
- Group size: 10 data packets
- Parity shards: 3 per group (recovers up to 3 losses in any group)
- Shard size: padded to the size of the largest packet in the group

---

## 7. Adaptive Bitrate (ABR — AIMD)

The receiver sends `Feedback` packets (packetType=6) periodically containing:

```
Field             Type    Description
────────────────  ──────  ───────────
bwEstimateMbps    f32     estimated available bandwidth
lossRatePct       f32     packet loss rate 0.0–100.0
jitterMs          f32     inter-packet jitter
lastRecvFrameSeq  u32     last complete frame sequence received
missingFrameCount u16     count of missing frames (max 16)
missingFrames[]   u32[16] frame sequence numbers of losses
```

The sender applies AIMD on the encoder bitrate:

| Condition | Action |
|-----------|--------|
| `lossRate > 5%` | multiply bitrate by 0.8 (multiplicative decrease) |
| `lossRate < 1%` AND `bwEstimate > currentBitrate × 1.15` | multiply bitrate by 1.1 (additive-ish increase) |
| Otherwise | hold current bitrate |

Minimum bitrate: 500 kbps. Maximum: `SblOutputConfig.maxBitrateKbps`.

---

## 8. Discovery and Routing

SBL supports three routing tiers:

### 8.1 LAN — Direct UDP

Source and receiver on the same network connect directly using LAN IP and port 8890. Discovery via mDNS or manual QR code pairing.

**QR code URI format:**
```
sbl://{lan_ip}:{port}?v=3&name={url_encoded_name}&codec={H264|H265|AV1}
```

### 8.2 WAN — Direct UDP (STUN)

For cross-network connections, the source resolves its public IP via STUN (`stun.l.google.com:19302`) and registers with the VPS registry.

**VPS Registry endpoints** (`https://api.blackmagicbox.es`):

```
POST /sbl/register
Body: { "name": string, "localIp": string, "publicIp": string,
        "port": number, "codec": string }

GET  /sbl/sources
Response: [{ "name", "localIp", "publicIp", "port", "codec", "updatedAtMs" }]
```

Sources re-register every 30 seconds. Entries older than 60 seconds are considered stale.

The SAMBA engine polls `/sbl/sources` to display a "Remote Sources (WAN)" panel with a Connect button.

### 8.3 Relay — VPS UDP Proxy

When NAT prevents direct WAN connections, traffic is proxied through the VPS relay.

- **Address:** `2.25.129.82:8893`
- **Protocol:** UDP — source and receivers all connect to the single relay port
- The relay forwards packets between sender and receivers without inspecting payload
- Adds ~1–5ms RTT overhead (same datacenter as registry)

---

## 9. AudioReturn — Talkback (Engine → Camera)

`streamID=3` carries Opus-encoded mono audio from the engine back to the camera operator.

- **Codec:** Opus, 48 kHz, mono, ~32–64 kbps
- **Direction:** Engine → SAMBA Air (received on the same UDP socket as outgoing video)
- **Use case:** Program audio talkback so the camera operator hears the live mix via earpiece
- **Mix-minus:** The engine SHOULD exclude the source's own audio from the return mix to prevent echo

SAMBA Air decodes AudioReturn via MediaCodec (Opus) and plays it through `AudioTrack` with usage `USAGE_VOICE_COMMUNICATION`.

---

## 10. Latency Budget

Measured on 802.11ac LAN (SAMBA Air → SAMBA, 1080p60 H.264):

| Stage | Typical |
|-------|---------|
| Capture → HW encode (mobile) | 10–20 ms |
| Network (UDP, WiFi LAN) | 1–5 ms |
| Jitter buffer (8ms target) | 8–16 ms |
| Decode → display | 5–15 ms |
| **Total glass-to-glass** | **~24–56 ms** |

Over WAN relay add ~20–50ms RTT depending on network path.

---

## 11. Security

- **Optional encryption:** AES-256-GCM, negotiated in Hello/HelloAck via P-256 ECDH
- **Key derivation:** ECDH shared secret → HKDF-SHA256 → 32-byte AES key
- **Auth tag:** First 4 bytes stored in `authTagPartial` for fast rejection of corrupt packets
- **No encryption (default):** `authTagPartial = 0`; use on trusted LAN only
- **Transport:** No TLS — SBL is designed for LAN and managed relay; apply network-level security (VPN, WPA3) for sensitive environments

---

## 12. Keepalive and Timeout

- Source sends `Keepalive` (packetType=4) every **1 second** of inactivity
- Connection is considered lost after **5 seconds** without any packet
- On disconnect: send `Goodbye` (packetType=5) if possible

---

## 13. Versioning

The protocol version (3) is carried in both the packet header (`version` field) and the Hello payload. Implementations MUST reject connections with an unsupported version. Future versions increment this field.

---

## 14. Conformance

A **conformant SBL v3 source** MUST:
- Send Hello on connect; handle HelloAck before sending media
- Support H.264 video and Opus audio as minimum codecs
- Fragment frames to ≤ 1200-byte payloads
- Send RS(10,3) FEC when `cfg.fecGroups > 0`
- Send Keepalive every 1 s during inactivity
- Send Goodbye on clean disconnect
- Process incoming `AudioReturn` packets (streamID=3) and play via speaker/earpiece

A **conformant SBL v3 receiver** MUST:
- Send HelloAck in response to Hello
- Reassemble fragmented frames via frameSeqNum + fragmentIdx/Total
- Apply RS FEC recovery when parity shards are available
- Send Feedback packets with loss rate and bandwidth estimate
- Send AudioReturn (streamID=3) with Opus talkback audio when configured
- Send Keepalive every 1 s during inactivity
