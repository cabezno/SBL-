# SBL Reference Implementations

## Official

| Platform | Repo | Status |
|----------|------|--------|
| Android / Flutter (source) | [SAMBA Air](https://github.com/cabezno/vortex-stream) | ✅ Production |
| Windows C++ (receiver) | [SAMBA](https://blackmagicbox.es/samba.html) | ✅ Production |

## Community

_Submit a PR to add your implementation here._

---

## Minimal implementation checklist

- [ ] WebSocket client connecting to `ws://{host}:{port}/sbl`
- [ ] SBL-HELLO sent on connect
- [ ] SBL-HELLO-ACK handled (check `accepted`)
- [ ] SBL-OFFER built from local WebRTC offer
- [ ] SBL-ANSWER applied to WebRTC peer connection
- [ ] TALLY messages handled and displayed
- [ ] PING / PONG keepalive
- [ ] Graceful disconnect on WebSocket close
