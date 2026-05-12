# Migration Approach Comparison: Daily → LiveKit

Two approaches are available for migrating the existing Pipecat framework from Daily.co to LiveKit.

---

## Approach 1: Switch Pipecat to LiveKit Transport

### Scope of Changes

| Layer | Files to Modify | Change |
|-------|----------------|--------|
| Backend deps | `pyproject.toml` | `pipecat-ai[daily,...]` → `pipecat-ai[livekit,...]` |
| Backend Transport | `interview_bot/bot.py`, `training_bot/bot.py` | `DailyTransport` → `LiveKitTransport` |
| Room management | `api.py` | Daily API → LiveKit Server SDK for room/token creation |
| Frontend deps | `package.json` | `@pipecat-ai/daily-transport` → `@pipecat-ai/livekit-transport` |
| Frontend Provider | `PipecatProvider.tsx` | `DailyTransport` → `LiveKitTransport` |

### Pros

- Changes are contained — Pipecat abstracts the Transport layer, so pipeline logic stays untouched
- Can test with LiveKit Cloud first, then switch to self-hosted later
- Officially supported by Pipecat

### Cons

- Need to verify that VAD, video output, and other features work correctly under LiveKit Transport
- If using LiveKit Cloud, latency from mainland China may still be a concern

---

## Approach 2: Self-Host LiveKit Server with Docker

### Deployment Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Your Server                       │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │   LiveKit    │  │   FastAPI    │  │   Redis   │  │
│  │   Server     │  │   + Bots     │  │ (optional)│  │
│  │ :7880/:7881  │  │    :8000     │  │           │  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
│         ↑                 ↑                          │
│         └─────────────────┘                          │
│              WebRTC + Internal API                   │
└─────────────────────────────────────────────────────┘
              ↑
          User Browser
```

### Docker Compose Example

```yaml
version: "3.9"
services:
  livekit:
    image: livekit/livekit-server:latest
    ports:
      - "7880:7880"  # HTTP/WebSocket
      - "7881:7881"  # RTC (TCP fallback)
    environment:
      - LIVEKIT_KEYS=APIKey:SecretKey
    command: --config /etc/livekit.yaml
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml

  app:
    build: .
    depends_on:
      - livekit
    environment:
      - LIVEKIT_URL=ws://livekit:7880
      - LIVEKIT_API_KEY=APIKey
      - LIVEKIT_API_SECRET=SecretKey
```

### Pros

- Full control — no data passes through third parties
- Lowest latency — server hosted in China, users connect directly
- Predictable cost — fixed server cost, no per-usage SaaS fees
- Data sovereignty — audio/video stays within China

### Cons

- High operational complexity: STUN/TURN setup, NAT traversal, TLS certificates, public IP, UDP port ranges
- Scaling requires building your own load balancing
- Debugging A/V issues requires deep WebRTC knowledge

---

## Decision Matrix

| Dimension | Approach 1 (Pipecat + LiveKit Cloud) | Approach 2 (Self-Hosted) |
|-----------|--------------------------------------|--------------------------|
| Dev effort | Medium (swap Transport) | Medium + ops work |
| China latency | Depends on LiveKit Cloud nodes | Optimal (domestic server) |
| Cost | LiveKit Cloud pay-per-use | Fixed server cost |
| Ops burden | Low | High (WebRTC expertise needed) |
| Data compliance | Data flows through LiveKit Cloud | Fully self-controlled |
| Time to launch | Fast (1–2 weeks) | Slow (network debugging needed) |
| Scalability | Auto-scaled by LiveKit Cloud | DIY |
