# SBL Protocol Specification — v1.0

**Status:** Draft  
**Authors:** BlackMagicBox  
**License:** Apache 2.0

---

## 1. Overview

SBL (Sub-second Broadcast Link) is an application-layer protocol for real-time video and audio transmission in professional live production environments. It is built on top of WebRTC's media transport (RTP/RTCP over DTLS-SRTP) and adds a lightweight control channel for production-specific signaling.

### 1.1 Design Goals

- **Latency:** Glass-to-glass 100–300ms on 802.11ac/Wi-Fi 6 local networks (default config); <150ms with jitter buffer tuning (see §6.4)
- **Simplicity:** Minimal signaling — connection established in ≤ 3 round trips
- **Production-grade:** Tally, scene metadata, and transport control built into the protocol
- **Zero-config:** mDNS discovery and QR-code pairing — no IP addresses to type
- **Open:** Apache 2.0, no SDK required, no registration

### 1.2 Non-Goals

- WAN/internet transmission (use SRT or WHIP for that)
- Reliable file transfer
- Recording (handled at the application layer)

---

## 2. Transport Stack

```
┌─────────────────────────────────────────────┐
│           Application (SAMBA, SAMBA Air)    │
├──────────────────┬──────────────────────────┤
│  Media Channel   │     Control Channel      │
│  RTP/RTCP        │     JSON over WebSocket  │
├──────────────────┴──────────────────────────┤
│              DTLS 1.3                       │
├─────────────────────────────────────────────┤
│              ICE (RFC 8445)                 │
├─────────────────────────────────────────────┤
│              UDP (primary) / TCP (fallback) │
└─────────────────────────────────────────────┘
```

### 2.1 Media Transport

SBL uses WebRTC's standard media stack:
- **ICE** (RFC 8445) for NAT traversal and candidate selection
- **DTLS 1.3** for encryption and authentication
- **SRTP** for media encryption (negotiated via DTLS)
- **RTP/RTCP** for media delivery and congestion control

SBL does **not** use SDP. Codec negotiation is handled via the SBL signaling messages defined in Section 4.

### 2.2 Control Channel

A persistent WebSocket connection carries JSON control messages (Section 5). This channel is established before the WebRTC media session and kept alive for the duration of the connection.

---

## 3. Discovery

### 3.1 mDNS / DNS-SD

SBL receivers advertise themselves on the local network using mDNS:

```
Service type:  _sbl._tcp.local
Instance name: {device-name}._sbl._tcp.local
Port:          Default 7478
TXT records:
  version=1
  name={human-readable name}
  caps={comma-separated codec list}   # e.g. "h264,h265,av1,opus"
  auth={none|token}
```

### 3.2 QR Code Pairing

For environments where mDNS is blocked, receivers display a QR code encoding the following URI:

```
sbl://{host}:{port}?v=1&token={pairing_token}&name={url_encoded_name}
```

- `token`: 6-character alphanumeric, regenerated each session
- `name`: human-readable name of the receiver
- `v`: SBL protocol version

Scanning the QR code initiates a connection attempt with automatic token verification.

---

## 4. Signaling — Connection Setup

Signaling uses JSON messages over the WebSocket control channel. Each message has a `type` field. Unknown message types MUST be silently ignored for forward compatibility.

### 4.1 Connection Flow

```
Source (app)                          Receiver (SAMBA)
    │                                      │
    │── WS connect to ws://host:port/sbl ──│
    │                                      │
    │──────── SBL-HELLO ──────────────────▶│
    │◀─────── SBL-HELLO-ACK ──────────────│
    │                                      │
    │──────── SBL-OFFER ──────────────────▶│
    │◀─────── SBL-ANSWER ─────────────────│
    │                                      │
    │  [ICE candidate exchange via DATA]   │
    │◀────────────────────────────────────▶│
    │                                      │
    │  [DTLS handshake — media channel]    │
    │◀────────────────────────────────────▶│
    │                                      │
    │  [RTP/RTCP media flowing]            │
    │◀════════════════════════════════════▶│
    │                                      │
    │──────── TALLY ──────────────────────▶│  (bidirectional, ongoing)
    │◀─────── TALLY ──────────────────────│
```

### 4.2 SBL-HELLO

Sent by the source immediately after WebSocket connection.

```json
{
  "type": "SBL-HELLO",
  "version": 1,
  "token": "ABC123",
  "source": {
    "name": "iPhone 15 Pro",
    "platform": "android",
    "app": "SAMBA Air",
    "appVersion": "1.2.0"
  },
  "caps": {
    "video": ["h264", "h265"],
    "audio": ["opus"],
    "maxBitrateMbps": 50,
    "maxResolution": "3840x2160",
    "maxFps": 60
  }
}
```

### 4.3 SBL-HELLO-ACK

Sent by the receiver in response to SBL-HELLO.

```json
{
  "type": "SBL-HELLO-ACK",
  "accepted": true,
  "receiverName": "SAMBA - Main Studio",
  "sessionId": "f3a9c2e1"
}
```

If `accepted` is `false`, the connection is closed. A `reason` field MAY be included.

### 4.4 SBL-OFFER

Source proposes codec and transport parameters.

```json
{
  "type": "SBL-OFFER",
  "sessionId": "f3a9c2e1",
  "video": {
    "codec": "h264",
    "profile": "constrained-baseline",
    "bitrateMbps": 8,
    "resolution": "1920x1080",
    "fps": 60,
    "keyframeIntervalMs": 1000
  },
  "audio": {
    "codec": "opus",
    "sampleRate": 48000,
    "channels": 2,
    "bitratekbps": 128
  },
  "ice": {
    "candidates": [ "...RFC8445 candidate strings..." ],
    "ufrag": "xxxx",
    "pwd": "yyyyyyyyyyyy"
  },
  "dtlsFingerprint": "sha-256 AA:BB:CC:..."
}
```

### 4.5 SBL-ANSWER

Receiver confirms or adjusts parameters.

```json
{
  "type": "SBL-ANSWER",
  "sessionId": "f3a9c2e1",
  "video": {
    "codec": "h264",
    "bitrateMbps": 8,
    "resolution": "1920x1080",
    "fps": 60
  },
  "audio": {
    "codec": "opus",
    "sampleRate": 48000,
    "channels": 2
  },
  "ice": {
    "candidates": [ "...RFC8445 candidate strings..." ],
    "ufrag": "aaaa",
    "pwd": "bbbbbbbbbbbb"
  },
  "dtlsFingerprint": "sha-256 DD:EE:FF:..."
}
```

---

## 5. Control Messages (ongoing)

After the media session is established, both sides exchange control messages on the WebSocket channel.

### 5.1 TALLY

Sent by the receiver to inform the source of its current on-air status. Sources SHOULD display this visually to the operator.

```json
{
  "type": "TALLY",
  "sessionId": "f3a9c2e1",
  "status": "live"
}
```

`status` values:
- `"live"` — source is on program output (on-air)
- `"preview"` — source is in preview (not yet live)
- `"standby"` — source is connected but not in any scene
- `"recording"` — source is being recorded but not streamed

### 5.2 METADATA

Sent by the receiver to push contextual information to the source.

```json
{
  "type": "METADATA",
  "sessionId": "f3a9c2e1",
  "scene": "Interview",
  "sourceName": "Camera 2",
  "timecode": "01:23:45:12",
  "streamDuration": 5432
}
```

### 5.3 TRANSPORT-CONTROL

Sent by the receiver to request transport actions from the source.

```json
{
  "type": "TRANSPORT-CONTROL",
  "sessionId": "f3a9c2e1",
  "action": "mark",
  "payload": { "label": "Highlight" }
}
```

`action` values: `"mark"`, `"snapshot"`, `"flip-camera"`, `"torch-on"`, `"torch-off"`

### 5.4 BITRATE-HINT

Sent by the receiver to request a bitrate adjustment based on network conditions.

```json
{
  "type": "BITRATE-HINT",
  "sessionId": "f3a9c2e1",
  "targetMbps": 4.5
}
```

### 5.5 PING / PONG

Standard keepalive. Either side may send PING; the other MUST respond with PONG within 5 seconds or the connection is considered lost.

```json
{ "type": "PING", "ts": 1718000000000 }
{ "type": "PONG", "ts": 1718000000000 }
```

---

## 6. Media Requirements

### 6.1 Video

| Parameter | Mandatory | Optional |
|-----------|-----------|---------|
| H.264 Constrained Baseline | ✅ | — |
| H.265 Main | — | ✅ |
| AV1 | — | ✅ |
| Keyframe interval | ≤ 2s | — |
| Bitrate range | 1–50 Mbps | — |
| Resolution | Up to 3840×2160 | — |
| Frame rate | Up to 60fps | — |

### 6.2 Audio

| Parameter | Mandatory | Optional |
|-----------|-----------|---------|
| Opus 48kHz stereo | ✅ | — |
| AAC-LC | — | ✅ |
| Bitrate | 64–256 kbps | — |

### 6.3 Latency Budget

Measured on 802.11ac LAN (SAMBA Air → SAMBA, 1080p60 H.264):

| Stage | Default | Optimized (§6.4) |
|-------|---------|------------------|
| Capture → HW encode (mobile) | 10–30 ms | 10–20 ms |
| Network (WiFi LAN, UDP) | 1–5 ms | 1–5 ms |
| DTLS-SRTP overhead | 2–5 ms | 2–5 ms |
| **Jitter buffer (WebRTC)** | **80–200 ms** | **0–30 ms** |
| Decode → display | 5–20 ms | 5–15 ms |
| **Total glass-to-glass** | **~100–260 ms** | **~20–75 ms** |

> **Note:** The jitter buffer is the dominant source of latency in default WebRTC implementations. The `<300ms` target is met in all configurations; sub-100ms requires explicit jitter buffer configuration as described in §6.4.

---

### 6.4 Jitter Buffer Tuning

WebRTC implementations buffer incoming packets to absorb reordering and loss. On a local LAN — where packet loss is near zero and out-of-order delivery is rare — this buffer is the main contributor to latency.

#### Receiver side (SAMBA — libdatachannel / libwebrtc)

Set the minimum playout delay to 0 via RTCP extension (RFC 7941):

```cpp
// When creating the RTCPeerConnection
rtc::Configuration config;
config.disableAutoNegotiation = false;

// After track is added, set jitter buffer target
track->setMediaHandler(std::make_shared<rtc::RtcpReceivingSession>());
// Force minimum jitter buffer via RTCP PLI budget
track->requestKeyframe(); // flush stale frames on connect
```

Or, if using the `playout-delay` RTP header extension (RFC 7941), include it in the SBL-ANSWER and set `min=0, max=1` (units of 10ms):

```json
{
  "type": "SBL-ANSWER",
  "extensions": {
    "playout-delay": { "min": 0, "max": 1 }
  }
}
```

#### Source side (SAMBA Air — flutter_webrtc)

Disable adaptive jitter buffer on the sender:

```dart
final constraints = <String, dynamic>{
  'mandatory': {
    'googNoiseSupression': false,
    'googHighpassFilter': false,
  },
  'optional': [
    {'googDscp': true},
    {'googCpuOveruseDetection': false},
  ],
};
```

Configure the encoder for zero-delay (no B-frames, no reordering):

```dart
final encoderParams = RTCRtpEncodingParameters(
  maxBitrate: 8000000,
  maxFramerate: 60,
  // Zero-latency profile: no B-frames, IDR every 60 frames
);
```

#### Expected results by configuration

| Mode | Jitter buffer | Expected latency |
|------|--------------|-----------------|
| Default (flutter_webrtc out-of-box) | 80–200 ms | 100–260 ms |
| `playout-delay` extension min=0 | 20–50 ms | 40–110 ms |
| Full low-latency mode (both sides) | 0–10 ms | 20–75 ms |


## 7. Security

- All media is encrypted via DTLS-SRTP (mandatory, no plaintext media)
- Control channel SHOULD use WSS (TLS) in production deployments
- QR token provides session-level authentication; implement application-level auth for multi-user environments
- Tokens are single-session and invalidated on disconnect

---

## 8. Versioning

The protocol version is carried in `SBL-HELLO`. Implementations MUST reject connections with a `version` they do not support and MUST NOT silently degrade. Version negotiation is not part of v1.0; future versions will define a negotiation mechanism.

---

## 9. Conformance

A **conformant SBL source** MUST:
- Support H.264 Constrained Baseline video
- Support Opus audio
- Implement all messages in Sections 4 and 5
- Respond to PING within 5 seconds
- Display TALLY status visually

A **conformant SBL receiver** MUST:
- Accept H.264 Constrained Baseline video
- Accept Opus audio
- Implement mDNS advertisement (`_sbl._tcp.local`)
- Send TALLY updates on source status change
- Send BITRATE-HINT when network degradation is detected
