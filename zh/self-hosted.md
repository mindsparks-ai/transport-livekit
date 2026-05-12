# LiveKit 自建服务器部署指南

**状态**: 规划中 | **创建时间**: 2026-03-14

---

## 1. 概述

### 1.1 目的

本文档描述如何在阿里云国内服务器上自建 LiveKit 服务，用于替代 Daily.co 作为 WebRTC 实时音视频传输层。

**迁移目标**:
- 从 Daily.co 迁移到自建 LiveKit
- 与现有后端服务共置
- 支持 100 用户 + 100 Bot = 200 并发连接
- 保持 Daily/LiveKit 共存一段时间进行灰度切换

### 1.2 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                          部署架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐     ┌──────────────────────────────────┐    │
│   │   用户浏览器   │────►│           阿里云服务器              │    │
│   │  (WebRTC)    │     │                                  │    │
│   └──────────────┘     │  ┌──────────────┐               │    │
│                        │  │    Caddy     │  (80/443 TCP)  │    │
│                        │  │  (SSL 终结)   │               │    │
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
│  端口说明:                                                       │
│  - 80/443: Caddy HTTP/HTTPS                                     │
│  - 7880: LiveKit HTTP API                                       │
│  - 7881: WebRTC over TCP                                        │
│  - 3478/5349: TURN/STUN                                         │
│  - 50000-60000: WebRTC UDP                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 需求

| 组件 | 要求 | 备注 |
|------|------|------|
| 服务器 | 4C8G | 推荐配置，可根据并发调整 |
| 带宽 | 50Mbps+ | 音视频消耗较大 |
| 域名 | 已备案域名 | 国内服务器需要 |
| 操作系统 | Ubuntu 20.04+ / Alibaba Cloud Linux | |
| Docker | 20.10+ | |
| Docker Compose | 2.0+ | |

---

## 2. 环境准备

### 2.1 服务器要求

```bash
# 推荐配置
- CPU: 4 cores
- RAM: 8 GB
- Disk: 40 GB SSD
- Bandwidth: 50 Mbps+

# 最低配置
- CPU: 2 cores
- RAM: 4 GB
- Disk: 20 GB SSD
- Bandwidth: 20 Mbps
```

100 用户 + 100 Bot = 200 并发连接，建议使用 4C8G 配置。并发更高时考虑增加 CPU 和带宽。

### 2.2 域名与 SSL

- 拥有一个已备案的域名（国内服务器必需）
- 域名已解析到服务器 IP
- SSL 证书由 Caddy 自动管理（Let's Encrypt），无需手动申请

### 2.3 端口配置

LiveKit 内置 TURN 服务器，无需额外部署 Coturn。

#### 必需端口

| 端口 | 协议 | 用途 |
|------|------|------|
| 80 | TCP | HTTP / SSL 证书验证 |
| 443 | TCP | HTTPS |
| 7880 | TCP | LiveKit HTTP API |
| 7881 | TCP | WebRTC over TCP（UDP 不可用时的回退）|

#### 推荐开放（UDP 优先场景）

| 端口 | 协议 | 用途 |
|------|------|------|
| 3478 | UDP | STUN / TURN UDP |
| 5349 | TCP | TURN TLS |
| 50000-60000 | UDP | WebRTC UDP（主要音视频传输）|

如果 UDP 端口受限，至少开放必需端口，LiveKit 会自动回退到 TCP (7881)，延迟略高但功能正常。

### 2.4 安装依赖

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sh
sudo systemctl enable docker && sudo systemctl start docker
sudo apt-get install docker-compose-plugin
docker --version && docker compose version
```

---

## 3. 生成配置文件

### 3.1 使用官方工具

```bash
sudo mkdir -p /opt/livekit
cd /opt/livekit
docker run --rm -it -v$PWD:/output livekit/generate
```

交互输入示例：
```
What domain would you like to use? > livekit.yourdomain.com
Please enter your email for SSL certificate generation: > admin@yourdomain.com
Would you like to enable LiveKit Ingress? (y/N) > N
Would you like to enable LiveKit Egress? (y/N) > N
```

### 3.2 生成的文件

| 文件 | 描述 |
|------|------|
| `livekit.yaml` | LiveKit 主配置文件 |
| `docker-compose.yaml` | Docker Compose 编排文件 |
| `caddy.yaml` | Caddy 反向代理配置 |
| `redis.conf` | Redis 配置文件 |
| `cloud_init.yaml` | 云初始化脚本（可选）|

### 3.3 配置文件详解

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

## 4. Docker 部署

### 4.1 目录结构

```
/opt/livekit/
├── livekit.yaml
├── docker-compose.yaml
├── caddy.yaml
├── redis.conf
├── redis-data/        # 自动创建
└── caddy-data/        # 自动创建
```

### 4.2 部署步骤

```bash
cd /opt/livekit
docker compose up -d
docker compose ps
docker compose logs -f livekit
curl http://localhost:7880
```

### 4.3 验证部署

```bash
# 检查容器状态
docker compose ps

# 检查端口监听
sudo netstat -tlnp | grep -E '7880|7881|80|443'

# 测试 API
curl -s http://localhost:7880 | head -20

# 测试 HTTPS（需先配置域名解析）
curl -k https://your-domain.com
```

### 4.4 常用命令

```bash
docker compose up -d          # 启动
docker compose down           # 停止
docker compose restart        # 重启
docker compose logs -f        # 查看日志
docker compose logs -f livekit
docker exec -it livekit-livekit-1 /bin/sh
docker stats                  # 查看资源使用
docker compose pull && docker compose up -d  # 更新
```

---

## 5. 阿里云安全组配置

### 必需端口

| 端口范围 | 协议 | 用途 |
|---------|------|------|
| 80 | TCP | HTTP / SSL 证书验证 |
| 443 | TCP | HTTPS |
| 7880 | TCP | LiveKit HTTP API |
| 7881 | TCP | WebRTC over TCP |

### 推荐开放

| 端口范围 | 协议 | 用途 |
|---------|------|------|
| 3478 | UDP | STUN / TURN UDP |
| 5349 | TCP | TURN TLS |
| 50000-60000 | UDP | WebRTC UDP |

授权对象：`0.0.0.0/0`（或限制为特定 IP）

### 配置步骤

1. 登录阿里云 ECS 控制台，找到目标实例
2. 点击「安全组」→「配置规则」
3. 添加「入方向」规则，按上表填写协议、端口、授权对象
4. 保存规则

### UDP 端口范围受限时的替代方案

**方案 A**：使用 TCP 回退（端口 7881），LiveKit 自动降级，功能正常，延迟略高。

**方案 B**：缩小 UDP 端口范围：
```yaml
# livekit.yaml
rtc:
  port_range_start: 50000
  port_range_end: 50100
```

---

## 6. 后端集成

### 6.1 环境变量

```bash
LIVEKIT_URL=https://your-domain.com
LIVEKIT_API_KEY=your_api_key_from_config
LIVEKIT_API_SECRET=your_api_secret_from_config

# 共存期间保留 Daily
DAILY_API_KEY=your_daily_key

# 传输类型切换
BOT_TRANSPORT=daily  # 切换时改为 livekit
```

### 6.2 Python 依赖

```toml
# pyproject.toml
# 修改前
"pipecat-ai[daily,google,openai,runner,silero,simli,webrtc]>=0.0.101",
# 修改后
"pipecat-ai[daily,livekit,google,openai,runner,silero,simli,webrtc]>=0.0.101",
```

```bash
cd server/fastapi && pip install -e ".[daily,livekit]"
```

### 6.3 房间创建 API

在 `server/fastapi/features/bot/` 下创建 `livekit_room.py`：

```python
import os
import json
from typing import Tuple
from livekit import api
from livekit.api import LiveKitAPI
from loguru import logger


async def create_livekit_room(session_id: str) -> Tuple[str, str, str]:
    livekit_url = os.getenv("LIVEKIT_URL")
    livekit_api_key = os.getenv("LIVEKIT_API_KEY")
    livekit_api_secret = os.getenv("LIVEKIT_API_SECRET")

    if not all([livekit_url, livekit_api_key, livekit_api_secret]):
        raise ValueError("LiveKit 环境变量未配置")

    lk_api = LiveKitAPI(livekit_url, livekit_api_key, livekit_api_secret)
    try:
        room = await lk_api.room.create(api.CreateRoom(
            name=f"bot-{session_id}",
            max_participants=2,
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

### 6.4 Bot Transport 切换

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

### 6.5 API 层切换

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

## 7. 前端适配

截至 2025-12，Pipecat Client JavaScript SDK 尚不支持 LiveKit Transport，推荐直接使用 LiveKit Web SDK。

```bash
cd frontend && npm install livekit-client
```

```typescript
// src/hooks/useLiveKit.ts
import { useState, useRef, useCallback, useEffect } from 'react';
import { Room, RoomEvent, RemoteParticipant } from 'livekit-client';

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

## 8. 灰度发布

### 切换策略

```python
import hashlib

def should_use_livekit(user_id: str, percentage: int = 30) -> bool:
    hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    return (hash_value % 100) < percentage
```

### 切换阶段

| 阶段 | 比例 | 对象 | 时间 |
|------|------|------|------|
| 1 | 0% | 仅开发测试 | Day 1-2 |
| 2 | 10% | 内部用户 | Day 3-4 |
| 3 | 30% | 部分外部用户 | Day 5-7 |
| 4 | 50% | 50% 用户 | Day 8-10 |
| 5 | 100% | 全量 | Day 11+ |

---

## 9. 运维

### 日志

```bash
docker compose logs -f livekit
docker compose logs -f redis
docker compose logs -f caddy
docker compose logs --tail=100 livekit
docker compose logs livekit > livekit.log
```

### 监控

```bash
docker stats
```

### 升级

```bash
docker compose pull
docker compose up -d
docker compose exec livekit livekit-server --version
```

---

## 10. 故障排查

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 连接失败 | 端口未开放 | 检查阿里云安全组 |
| SSL 失败 | 域名未解析 | 检查 DNS 解析 |
| TURN 失败 | UDP 端口问题 | 使用 TCP fallback (7881) |
| 延迟高 | 带宽不足 | 升级带宽 |
| 房间创建失败 | API 密钥错误 | 检查 livekit.yaml 中的 keys |

### 调试命令

```bash
sudo netstat -tlnp | grep livekit
sudo ss -ulnp | grep -E '3478|50000'
docker compose logs livekit | grep -i error
docker compose exec livekit sh
curl -s http://localhost:7880 | head -20
```

---

## 11. 检查清单

- [ ] 服务器准备完成
- [ ] 域名解析配置
- [ ] 阿里云安全组端口开放
- [ ] Docker & Docker Compose 安装
- [ ] 配置文件生成
- [ ] 服务启动成功
- [ ] API 可访问测试
- [ ] 后端环境变量配置
- [ ] 后端代码改造完成
- [ ] 前端 SDK 切换
- [ ] 1% 灰度测试
- [ ] 30% 灰度测试
- [ ] 100% 全量切换

---

## 参考资料

- [LiveKit 官方文档](https://docs.livekit.io)
- [LiveKit 自建部署](https://docs.livekit.io/transport/self-hosting/vm/)
- [LiveKit 配置参考](https://docs.livekit.io/home/self-hosting/configuration)
- [Pipecat LiveKit Transport](https://docs.pipecat.ai/transports/livekit)
- [LiveKit 配置生成工具](https://github.com/livekit/deploy)
- [Caddy 文档](https://caddyserver.com/docs/)

---

## 附录：环境变量完整列表

```bash
LIVEKIT_URL=https://livekit.yourdomain.com
LIVEKIT_API_KEY=your_api_key
LIVEKIT_API_SECRET=your_api_secret

BOT_TRANSPORT=daily          # daily / livekit
LIVEKIT_PERCENTAGE=0         # 0-100，灰度百分比

DAILY_API_KEY=your_daily_key # 共存期间保留
LOG_LEVEL=info
```
