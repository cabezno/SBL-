# SBL — Sub-second Broadcast Link

**Open protocol for professional live production over local networks.**

SBL is a lightweight, open protocol designed to connect cameras, encoders, and production software with glass-to-glass latency of 100-300ms on standard WiFi (20-75ms in low-latency mode). It extends WebRTC's media transport with a production-grade control layer: tally signaling, scene metadata, transport controls, and zero-config discovery.

Developed by [BlackMagicBox](https://blackmagicbox.es) and used in [SAMBA](https://blackmagicbox.es/samba.html) and [SAMBA Air](https://github.com/cabezno/vortex-stream). Open-sourced under Apache 2.0 so any developer or manufacturer can implement it.

---

## Why SBL?

| Protocol | Latency (LAN) | Bandwidth | Production Metadata | Discovery | License |
|----------|--------------|-----------|--------------------|-----------|---------| 
| **SBL**  | **100-300ms** (1)   | 2–50 Mbps | ✓ Tally, Scene, TC | mDNS + QR | **Apache 2.0** |
| NDI      | 100–200ms    | 100–500 Mbps | Partial | mDNS | Proprietary SDK |
| WHIP/WebRTC | 100–500ms | Variable | ✗ None | Manual URL | IETF Open |
| SRT      | 200–2000ms   | Variable  | ✗ None | Manual | MPL-2.0 |
| RTMP     | 1–5s         | Variable  | ✗ None | Manual | Proprietary |

> (1) Default WebRTC jitter buffer: 100-260ms. With playout-delay extension (min=0): 40-110ms. Full low-latency mode: 20-75ms. See SPEC.md section 6.4 for tuning guide.

SBL sits between NDI and WHIP: it uses WebRTC's proven media stack for low latency and reliability, while adding the production metadata layer that WHIP deliberately omits.

---

## Quick Start

### Connecting to SAMBA

1. Open SAMBA on your production PC
2. Go to **Sources → Add → SAMBA Air / SBL Device**
3. A QR code appears — scan it with any SBL-compatible app
4. The source appears immediately at 100-300ms latency (default); configure low-latency mode for <75ms

### Implementing SBL in your app

```
1. Discover the SBL host via mDNS (_sbl._tcp.local) or QR code
2. Open a WebSocket to ws://{host}:{port}/sbl
3. Exchange SBL-OFFER / SBL-ANSWER messages (see SPEC.md)
4. Establish WebRTC media session
5. Send/receive TALLY, METADATA messages on the control channel
```

Full implementation guide: [docs/implementation-guide.md](docs/implementation-guide.md)

---

## Repository Structure

```
SBL/
├── README.md                    # This file
├── LICENSE                      # Apache 2.0
├── SPEC.md                      # Full protocol specification
├── docs/
│   ├── protocol.md              # Deep-dive: wire format, message types
│   ├── comparison.md            # NDI / SRT / WHIP comparison
│   └── implementation-guide.md  # Step-by-step for app developers
└── examples/
    └── README.md                # Reference implementations
```

---

## Reference Implementations

| Platform | Status | Repository |
|----------|--------|-----------|
| Android (Flutter) | ✅ Production | [SAMBA Air](https://github.com/cabezno/vortex-stream) |
| Windows (C++) | ✅ Production | [SAMBA](https://blackmagicbox.es/samba.html) |
| iOS | Planned | — |
| Web (JS) | Planned | — |

---

## Ecosystem

If you implement SBL in your app or device, open a PR to add it to this list. We'll link it from the official BlackMagicBox website.

---

## License

Apache 2.0 — free to implement in commercial and open-source projects. No royalties. No registration required.

See [LICENSE](LICENSE).

---

## Contributing

Issues and PRs welcome. For protocol changes, open an issue first to discuss the proposal before implementing.

Made with ❤️ by [BlackMagicBox](https://blackmagicbox.es)
