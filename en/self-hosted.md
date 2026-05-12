# LiveKit Self-Hosted Deployment Guide

**Status**: Planning | **Created**: 2026-03-14

---

## 1. Overview

### Purpose

This guide covers deploying a self-hosted LiveKit service on an Alibaba Cloud server in China, replacing Daily.co as the WebRTC real-time audio/video transport layer.

**Migration goals**:
- Migrate from Daily.co to self-hosted LiveKit
- Co-locate with existing backend services
- Support 100 users + 100 bots = 200 concurrent connections
- Keep Daily/LiveKit running in parallel during gradual rollout

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       Deployment Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐     ┌──────────────────────────────────┐    │
│   │ User Browser │────►│        Alibaba Cloud Server       │    │
│   │  (WebRTC)    │     │                                  │    │
│   └──────────────┘     │  ┌──────────────┐               │    │
│                        │  │    Caddy     │  (80/443 TCP)  │    │
│                        │  │ (SSL termination)             │    │
│                        │  └──────┬───────┘               │    │
│                        │         │                        │    │
│                        │  ┌──────▼───────┐               │    │
│                        │  │   LiveKit    │  (7880/7881)   │    │
│                        │  │   Server     │               │    │
│                        │  └──────┬───────┘               │    │
│                        │         │                        │    │
│                        │  ┌──────▼───────┐               │    │
│                        │  │    Redis     │               │    │
│                        │  └──────┬───────┘               │    │
│                        │         │                        │    │
│                        │  ┌──────▼───────┐               │    │
│                        │  │   FastAPI    │               │    │
│                        │  │  (Pipecat)   │               │    │
│                        │  └──────────────┘               │    │
│                        └──────────────────────────────────┘    │
│                                                                 │
│  Ports:                                                         │
│  - 80/443: Caddy HTTP/HTTPS                                     │
│  - 7880: LiveKit HTTP API                                       │
│  - 7881: WebRTC over TCP                                        │
│  - 3478/5349: TURN/STUN                                         │
│  - 50000-60000: WebRTC UDP                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Requirements

| Component | Requirement | Notes |
|-----------|-------------|-------|
| Server | 4 vCPU / 8 GB RAM | Recommended; adjust based on concurrency |
| Bandwidth | 50 Mbps+ | A/V is bandwidth-intensive |
| Domain | ICP-registered domain | Required for servers in China |
| OS | Ubuntu 20.04+ / Alibaba Cloud Linux | |
| Docker | 20.10+ | |
| Docker Compose | 2.0+ | |

---

## 2. Environment Setup

### Server Specs

```
Recommended: 4 vCPU, 8 GB RAM, 40 GB SSD, 50 Mbps+
Minimum:     2 vCPU, 4 GB RAM, 20 GB SSD, 20 Mbps
```

### Domain & SSL

- An ICP-registered domain is required for servers hosted in mainland China
- Point the domain's A record to the server IP
- Caddy handles SSL certificates automatically via Let's Encrypt — no manual setup needed

### Port Configuration

LiveKit ships with a built-in TURN server; no separate Coturn deployment is needed.

**Required ports:**

| Port | Protocol | Purpose |
|------|----------|---------|
| 80 | TCP | HTTP / SSL certificate validation |
| 443 | TCP | HTTPS |
| 7880 | TCP | LiveKit HTTP API |
| 7881 | TCP | WebRTC over TCP (fallback when UDP is unavailable) |

**Recommended (UDP-preferred scenarios):**

| Port | Protocol | Purpose |
|------|----------|---------|
| 3478 | UDP | STUN / TURN UDP |
| 5349 | TCP | TURN TLS |
| 50000-60000 | UDP | WebRTC UDP (primary A/V transport) |

If UDP ports are restricted, open at minimum the required ports. LiveKit will automatically fall back to TCP (7881) — slightly higher latency but fully functional.

### Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sh
sudo systemctl enable docker && sudo systemctl start docker
sudo apt-get install docker-compose-plugin
docker --version && docker compose version
```

---

## 3. Generate Configuration

### Use the Official Generator

```bash
sudo mkdir -p /opt/livekit
cd /opt/livekit
docker run --rm -it -v$PWD:/output livekit/generate
```

Example prompts:
```
What domain would you like to use? > livekit.yourdomain.com
Please enter your email for SSL certificate generation: > admin@yourdomain.com
Would you like to enable LiveKit Ingress? (y/N) > N
Would you like to enable LiveKit Egress? (y/N) > N
```

### Generated Files

| File | Description |
|------|-------------|
| `livekit.yaml` | LiveKit main config |
| `docker-compose.yaml` | Docker Compose orchestration |
| `caddy.yaml` | Caddy reverse proxy config |
| `redis.conf` | Redis config |
| `cloud_init.yaml` | Cloud init script (optional) |

### Configuration Details

#### livekit.yaml

```yaml
port: 7880
log_level: info

rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true
  use_external_turn: true

keys:
  <YOUR_API_KEY>: <YOUR_API_SECRET>

redis:
  address: localhost:6379

turn:
  enabled: true
  tls_port: 5349
  domain: turn.yourdomain.com

room:
  auto_create: true
  empty_timeout: 3600
  max_participants: 100
```

#### docker-compose.yaml

```yaml
version: '3.8'

services:
  livekit:
    image: livekit/livekit-server:latest
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml
    cap_add:
      - NET_ADMIN

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./redis.conf:/etc/redis.conf
      - redis-data:/data
    command: redis-server /etc/redis.conf --appendonly yes

  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy.yaml:/etc/caddy.yaml
      - caddy-data:/data

volumes:
  redis-data:
  caddy-data:
```

---

## 4. Docker Deployment

### Directory Structure

```
/opt/livekit/
├── livekit.yaml
├── docker-compose.yaml
├── caddy.yaml
├── redis.conf
├── redis-data/        # auto-created
└── caddy-data/        # auto-created
```

### Deploy

```bash
cd /opt/livekit
docker compose up -d
docker compose ps
docker compose logs -f livekit
curl http://localhost:7880
```

### Verify

```bash
docker compose ps
sudo netstat -tlnp | grep -E '7880|7881|80|443'
curl -s http://localhost:7880 | head -20
curl -k https://your-domain.com
```

### Common Commands

```bash
docker compose up -d          # start
docker compose down           # stop
docker compose restart        # restart
docker compose logs -f        # tail all logs
docker compose logs -f livekit
docker exec -it livekit-livekit-1 /bin/sh
docker stats
docker compose pull && docker compose up -d  # update
```

---

## 5. Alibaba Cloud Security Group

### Required Ports

| Port Range | Protocol | Purpose |
|-----------|----------|---------|
| 80 | TCP | HTTP / SSL certificate validation |
| 443 | TCP | HTTPS |
| 7880 | TCP | LiveKit HTTP API |
| 7881 | TCP | WebRTC over TCP |

### Recommended (UDP-preferred scenarios)

| Port Range | Protocol | Purpose |
|-----------|----------|---------|
| 3478 | UDP | STUN / TURN UDP |
| 5349 | TCP | TURN TLS |
| 50000-60000 | UDP | WebRTC UDP |

Authorization source: `0.0.0.0/0` (or restrict to specific IPs)

### Steps

1. Log in to the Alibaba Cloud ECS console and locate the target instance
2. Click **Security Groups** → **Configure Rules**
3. Add **Inbound** rules per the tables above
4. Save

### When UDP Port Ranges Are Restricted

**Option A**: Use TCP fallback (port 7881). LiveKit degrades automatically — fully functional, slightly higher latency.

**Option B**: Reduce the UDP port range:
```yaml
# livekit.yaml
rtc:
  port_range_start: 50000
  port_range_end: 50100
```

---

## 6. Backend Integration

### Environment Variables

```bash
LIVEKIT_URL=https://your-domain.com
LIVEKIT_API_KEY=your_api_key_from_config
LIVEKIT_API_SECRET=your_api_secret_from_config

# Keep Daily during coexistence period
DAILY_API_KEY=your_daily_key

# Transport switch
BOT_TRANSPORT=daily  # change to livekit when ready
```

### Python Dependencies

```toml
# pyproject.toml
# Before
"pipecat-ai[daily,google,openai,runner,silero,simli,webrtc]>=0.0.101",
# After
"pipecat-ai[daily,livekit,google,openai,runner,silero,simli,webrtc]>=0.0.101",
```

```bash
cd server/fastapi && pip install -e ".[daily,livekit]"
```

### Room Creation API

Create `server/fastapi/features/bot/livekit_room.py`:

```python
import os, json
from typing import Tuple
from livekit import api
from livekit.api import LiveKitAPI
from loguru import logger


async def create_livekit_room(session_id: str) -> Tuple[str, str, str]:
    livekit_url = os.getenv("LIVEKIT_URL")
    livekit_api_key = os.getenv("LIVEKIT_API_KEY")
    livekit_api_secret = os.getenv("LIVEKIT_API_SECRET")

    if not all([livekit_url, livekit_api_key, livekit_api_secret]):
        raise ValueError("LiveKit env vars not configured")

    lk_api = LiveKitAPI(livekit_url, livekit_api_key, livekit_api_secret)
    try:
        room = await lk_api.room.create(api.CreateRoom(
            name=f"bot-{session_id}", max_participants=2,
            empty_timeout=3600,
            metadata=json.dumps({"session_id": session_id}),
        ))
        bot_token = (api.AccessToken(livekit_api_key, livekit_api_secret)
            .add_grant(api.RoomGrant(room_join=True, room=room.name,
                can_publish=True, can_subscribe=True, can_publish_data=True))
            .to_jwt())
        user_token = (api.AccessToken(livekit_api_key, livekit_api_secret)
            .add_grant(api.RoomGrant(room_join=True, room=room.name,
                can_publish=True, can_subscribe=True))
            .to_jwt())
        return f"{livekit_url}/{room.name}", bot_token, user_token
    finally:
        await lk_api.aclose()
```

### Bot Transport Switch

```python
# interview_bot/bot.py
import os
TRANSPORT_TYPE = os.getenv("BOT_TRANSPORT", "daily")

def get_transport(room_url: str, token: str, bot_name: str = "Bot"):
    if TRANSPORT_TYPE == "livekit":
        return LiveKitTransport(
            url=room_url, token=token,
            room_name=room_url.split("/")[-1],
            params=LiveKitParams(
                audio_in_enabled=True, audio_out_enabled=True,
                video_out_enabled=True, video_out_width=512,
                video_out_height=512, video_out_framerate=5,
                vad_analyzer=SileroVADAnalyzer(params=VADParams(
                    confidence=0.85, start_secs=0.2,
                    stop_secs=0.5, min_volume=0.6)),
            ),
        )
    return DailyTransport(room_url, token, bot_name,
        params=DailyParams(audio_in_enabled=True, audio_out_enabled=True,
            video_out_enabled=True, video_out_width=512,
            video_out_height=512, video_out_framerate=5,
            vad_analyzer=SileroVADAnalyzer(params=VADParams(
                confidence=0.85, start_secs=0.2,
                stop_secs=0.5, min_volume=0.6))))
```

### API Layer Switch

```python
# api.py
import os

def get_transport_type() -> str:
    return os.getenv("BOT_TRANSPORT", "daily")

async def create_room(session_id: str):
    if get_transport_type() == "livekit":
        from .livekit_room import create_livekit_room
        return await create_livekit_room(session_id)
    return await create_daily_room(session_id)
```

---

## 7. Frontend Adaptation

As of 2025-12, the Pipecat Client JavaScript SDK does not support LiveKit Transport. Use the LiveKit Web SDK directly instead.

```bash
cd frontend && npm install livekit-client
```

```typescript
// src/hooks/useLiveKit.ts
import { useState, useRef, useCallback, useEffect } from 'react';
import { Room, RoomEvent } from 'livekit-client';

export function useLiveKit({ url, token }: { url: string; token: string }) {
  const roomRef = useRef<Room | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const connect = useCallback(async () => {
    try {
      const room = new Room({ adaptiveStream: true, dynacast: true });
      room.on(RoomEvent.Connected, () => setIsConnected(true));
      room.on(RoomEvent.Disconnected, () => setIsConnected(false));
      await room.connect(url, token);
      roomRef.current = room;
    } catch (err) {
      setError(err as Error);
    }
  }, [url, token]);

  const disconnect = useCallback(() => {
    roomRef.current?.disconnect();
    roomRef.current = null;
  }, []);

  useEffect(() => () => disconnect(), [disconnect]);

  return { room: roomRef.current, isConnected, error, connect, disconnect };
}
```

---

## 8. Gradual Rollout

### Rollout Strategy

```python
import hashlib

def should_use_livekit(user_id: str, percentage: int = 30) -> bool:
    hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    return (hash_value % 100) < percentage
```

### Rollout Phases

| Phase | Traffic | Audience | Timeline |
|-------|---------|----------|----------|
| 1 | 0% | Dev/test only | Day 1–2 |
| 2 | 10% | Internal users | Day 3–4 |
| 3 | 30% | Subset of external users | Day 5–7 |
| 4 | 50% | Half of users | Day 8–10 |
| 5 | 100% | Full rollout | Day 11+ |

---

## 9. Operations

### Logs

```bash
docker compose logs -f livekit
docker compose logs -f redis
docker compose logs -f caddy
docker compose logs --tail=100 livekit
docker compose logs livekit > livekit.log
```

### Monitoring

```bash
docker stats
```

### Upgrade

```bash
docker compose pull
docker compose up -d
docker compose exec livekit livekit-server --version
```

---

## 10. Troubleshooting

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Connection fails | Port not open | Check Alibaba Cloud security group |
| SSL fails | Domain not resolved | Check DNS records |
| TURN fails | UDP port issue | Use TCP fallback (7881) |
| High latency | Insufficient bandwidth | Upgrade bandwidth |
| Room creation fails | Wrong API key | Check `keys` in livekit.yaml |

### Debug Commands

```bash
sudo netstat -tlnp | grep livekit
sudo ss -ulnp | grep -E '3478|50000'
docker compose logs livekit | grep -i error
docker compose exec livekit sh
curl -s http://localhost:7880 | head -20
```

---

## 11. Checklist

- [ ] Server provisioned
- [ ] Domain DNS configured
- [ ] Alibaba Cloud security group ports opened
- [ ] Docker & Docker Compose installed
- [ ] Config files generated
- [ ] Services started successfully
- [ ] API accessible
- [ ] Backend env vars configured
- [ ] Backend code updated
- [ ] Frontend SDK switched
- [ ] 1% canary test passed
- [ ] 30% rollout tested
- [ ] 100% full rollout complete

---

## References

- [LiveKit Docs](https://docs.livekit.io)
- [LiveKit Self-Hosting on VM](https://docs.livekit.io/transport/self-hosting/vm/)
- [LiveKit Configuration Reference](https://docs.livekit.io/home/self-hosting/configuration)
- [Pipecat LiveKit Transport](https://docs.pipecat.ai/transports/livekit)
- [LiveKit Config Generator](https://github.com/livekit/deploy)
- [Caddy Docs](https://caddyserver.com/docs/)

---

## Appendix: Full Environment Variable Reference

```bash
LIVEKIT_URL=https://livekit.yourdomain.com
LIVEKIT_API_KEY=your_api_key
LIVEKIT_API_SECRET=your_api_secret

BOT_TRANSPORT=daily          # daily / livekit
LIVEKIT_PERCENTAGE=0         # 0–100, gradual rollout percentage

DAILY_API_KEY=your_daily_key # keep during coexistence period
LOG_LEVEL=info
```