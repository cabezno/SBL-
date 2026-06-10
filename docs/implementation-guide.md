# SBL Implementation Guide

This guide walks you through implementing SBL in your app or device. The reference implementation is [SAMBA Air](https://github.com/cabezno/vortex-stream) (Flutter/Android).

---

## Overview

Implementing SBL requires two components:

1. **WebSocket client** — for SBL signaling (connection setup + control messages)
2. **WebRTC stack** — for media transport (any standard WebRTC library works)

---

## Step 1 — Discover the receiver

### Option A: mDNS (recommended for LAN)

Browse for `_sbl._tcp.local` services. When found, extract host, port, and TXT records.

```dart
// Flutter example using multicast_dns
final MDnsClient client = MDnsClient();
await client.start();
await for (final PtrResourceRecord ptr in client.lookup<PtrResourceRecord>(
    ResourceRecordQuery.serverPointer('_sbl._tcp'))) {
  // resolve SRV record for host/port
}
```

```swift
// iOS/macOS example
let browser = NWBrowser(for: .bonjourWithTXTRecord(type: "_sbl._tcp", domain: nil), using: .tcp)
```

```javascript
// Node.js / Electron example
const mdns = require('mdns');
const browser = mdns.createBrowser(mdns.tcp('sbl'));
browser.on('serviceUp', (service) => {
  console.log(service.host, service.port, service.txtRecord);
});
```

### Option B: QR code

Parse the URI: `sbl://{host}:{port}?v=1&token={token}&name={name}`

Extract host, port, token from the URI. The token is required for the SBL-HELLO message.

---

## Step 2 — Open WebSocket and send SBL-HELLO

```dart
// Flutter
final ws = await WebSocket.connect('ws://$host:$port/sbl');
final hello = jsonEncode({
  'type': 'SBL-HELLO',
  'version': 1,
  'token': token,
  'source': {
    'name': deviceName,
    'platform': 'android',
    'app': 'YourAppName',
    'appVersion': '1.0.0',
  },
  'caps': {
    'video': ['h264'],
    'audio': ['opus'],
    'maxBitrateMbps': 20,
    'maxResolution': '1920x1080',
    'maxFps': 60,
  }
});
ws.add(hello);
```

Wait for `SBL-HELLO-ACK`. If `accepted` is false, display an error and close.

---

## Step 3 — Gather ICE candidates and send SBL-OFFER

```dart
// Initialize WebRTC peer connection
final pc = await createPeerConnection({
  'iceServers': [], // LAN only — no STUN/TURN needed
  'iceTransportPolicy': 'all',
});

// Add local media track (camera + mic)
final stream = await navigator.mediaDevices.getUserMedia({
  'video': {'width': 1920, 'height': 1080, 'frameRate': 60},
  'audio': true,
});
stream.getTracks().forEach((t) => pc.addTrack(t, stream));

// Gather ICE candidates
final candidates = [];
pc.onIceCandidate = (candidate) {
  if (candidate != null) candidates.add(candidate.candidate);
};

// Wait for ICE gathering to complete, then send SBL-OFFER
final offer = await pc.createOffer();
await pc.setLocalDescription(offer);

ws.add(jsonEncode({
  'type': 'SBL-OFFER',
  'sessionId': sessionId,
  'video': {'codec': 'h264', 'bitrateMbps': 8, 'resolution': '1920x1080', 'fps': 60},
  'audio': {'codec': 'opus', 'sampleRate': 48000, 'channels': 2},
  'ice': {
    'candidates': candidates,
    'ufrag': extractUfrag(offer.sdp),
    'pwd': extractPwd(offer.sdp),
  },
  'dtlsFingerprint': extractFingerprint(offer.sdp),
}));
```

---

## Step 4 — Receive SBL-ANSWER and connect

```dart
ws.listen((message) async {
  final msg = jsonDecode(message);

  if (msg['type'] == 'SBL-ANSWER') {
    // Build a synthetic SDP from SBL-ANSWER fields and set remote description
    final remoteSdp = buildSdpFromAnswer(msg);
    await pc.setRemoteDescription(RTCSessionDescription(remoteSdp, 'answer'));
    // Add remote ICE candidates
    for (final c in msg['ice']['candidates']) {
      await pc.addCandidate(RTCIceCandidate(c, '', 0));
    }
  }

  if (msg['type'] == 'TALLY') {
    updateTallyDisplay(msg['status']); // 'live', 'preview', 'standby', 'recording'
  }

  if (msg['type'] == 'METADATA') {
    updateSceneInfo(msg['scene'], msg['sourceName'], msg['timecode']);
  }

  if (msg['type'] == 'BITRATE-HINT') {
    adjustBitrate(msg['targetMbps']);
  }

  if (msg['type'] == 'PING') {
    ws.add(jsonEncode({'type': 'PONG', 'ts': msg['ts']}));
  }
});
```

---

## Step 5 — Handle ongoing control messages

```dart
// Send periodic PING
Timer.periodic(Duration(seconds: 5), (_) {
  ws.add(jsonEncode({'type': 'PING', 'ts': DateTime.now().millisecondsSinceEpoch}));
});

// React to TRANSPORT-CONTROL commands from receiver
if (msg['type'] == 'TRANSPORT-CONTROL') {
  switch (msg['action']) {
    case 'flip-camera': flipCamera(); break;
    case 'torch-on':    enableTorch(true); break;
    case 'torch-off':   enableTorch(false); break;
    case 'snapshot':    takeSnapshot(); break;
    case 'mark':        addMarker(msg['payload']['label']); break;
  }
}
```

---

## Tally Display Guidelines

When implementing a source (camera app), the TALLY status SHOULD be displayed prominently:

| Status | Recommended display |
|--------|---------------------|
| `live` | Red indicator — "ON AIR" or "LIVE" |
| `preview` | Amber/orange indicator — "PREVIEW" |
| `standby` | Dim/grey indicator — "STANDBY" |
| `recording` | Red indicator — "REC" |

Reference: [SAMBA Air tally widget](https://github.com/cabezno/vortex-stream)

---

## Implementing a Receiver

If you are implementing an SBL receiver (e.g., a desktop application or hardware device that accepts SBL sources):

1. Advertise `_sbl._tcp.local` via mDNS on your local network port
2. Display a QR code with `sbl://{host}:{port}?v=1&token={token}&name={name}`
3. Accept WebSocket connections at `ws://{host}:{port}/sbl`
4. Handle SBL-HELLO, send SBL-HELLO-ACK
5. Handle SBL-OFFER, send SBL-ANSWER
6. Send TALLY updates when source status changes
7. Send BITRATE-HINT when you detect packet loss or buffer buildup

The SAMBA desktop application (by BlackMagicBox) is the reference receiver implementation.

---

## Testing Your Implementation

To test against the reference receiver:

1. Install [SAMBA](https://blackmagicbox.es) (Windows)
2. Go to Sources → Add Source → SAMBA Air / SBL
3. Scan the QR code with your implementation
4. Verify media flows and TALLY updates display correctly

---

## Getting Help

- Open an issue on this repository
- Join the discussion: [github.com/cabezno/SBL-/discussions](https://github.com/cabezno/SBL-/discussions)
