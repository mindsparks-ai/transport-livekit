# LiveKit Self-Hosted Deployment Guide
# LiveKit 自建服务器部署指南

**状态 (Status)**: 📋 规划中 | **创建时间**: 2026-03-14

---

## 1. Overview / 概述

### 1.1 Purpose / 目的

本文档描述如何在阿里云国内服务器上自建 LiveKit 服务，用于替代 Daily.co 作为 WebRTC 实时音视频传输层。

**迁移目标**:
- 从 Daily.co 迁移到自建 LiveKit
- 与现有后端服务共置
- 支持 100 用户 + 100 Bot = 200 并发连接
- 保持 Daily/LiveKit 共存一段时间进行灰度切换

### 1.2 Architecture / 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        部署架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐     ┌──────────────────┐                  │
│   │   用户浏览器   │────►│   阿里云服务器      │                  │
│   │  (WebRTC)    │     │                   │                  │
│   └──────────────┘     │  ┌──────────────┐  │                  │
│                        │  │   Caddy       │  │  (80/443 TCP)    │
│                        │  │  (SSL终结)     │  │                  │
│                        │  └──────┬───────┘  │                  │
│                        │         │          │                  │
│                        │  ┌──────▼───────┐  │                  │
│                        │  │  LiveKit     │  │  (7880/7881 TCP) │
│                        │  │  Server      │  │                  │
│                        │  └──────┬───────┘  │                  │
│                        │         │          │                  │
│                        │  ┌──────▼───────┐  │                  │
│                        │  │   Redis       │  │                  │
│                        │  └───────────────┘  │                  │
│                        │         │          │                  │
│                        │  ┌──────▼───────┐  │                  │
│                        │  │  FastAPI     │  │                  │
│                        │  │  (Pipecat)   │  │                  │
│                        │  └──────────────┘  │                  │
│                        └─────────────────────┘                  │
│                                                                 │
│   端口说明:                                                     │
│   - 80/443: Caddy HTTP/HTTPS                                   │
│   - 7880: LiveKit HTTP API                                     │
│   - 7881: WebRTC over TCP                                      │
│   - 3478/5349: TURN/STUN                                       │
│   - 50000-60000: WebRTC UDP                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Requirements / 需求

| Component / 组件 | Requirement / 要求 | Notes / 备注 |
|-----------------|-------------------|--------------|
| Server / 服务器 | 4C8G | 推荐配置，可根据并发调整 |
| Bandwidth / 带宽 | 50Mbps+ | 音视频消耗较大 |
| Domain / 域名 | 已备案域名 | 国内服务器需要 |
| OS / 操作系统 | Ubuntu 20.04+ / Alibaba Cloud Linux | |
| Docker | 20.10+ | |
| Docker Compose | 2.0+ | |

---

## 2. 环境准备 / Environment Preparation

### 2.1 服务器要求 / Server Requirements

```bash
# 推荐配置 (Recommended)
- CPU: 4 cores
- RAM: 8 GB
- Disk: 40 GB SSD
- Bandwidth: 50 Mbps+

# 最低配置 (Minimum)
- CPU: 2 cores
- RAM: 4 GB
- Disk: 20 GB SSD
- Bandwidth: 20 Mbps
```

**说明**: 100用户 + 100Bot = 200并发连接，建议使用 4C8G 配置。如果并发更高，考虑增加 CPU 和带宽。

### 2.2 域名与SSL / Domain & SSL

**前提条件**:
- 拥有一个已备案的域名（国内服务器必需）
- 域名已解析到服务器 IP

**SSL 证书**:
- 使用 Caddy 自动管理 Let's Encrypt 证书
- 无需手动申请

### 2.3 端口配置 / Port Configuration

**TURN 穿透服务器**: LiveKit 内置 TURN 服务器，无需额外部署 Coturn。

#### 必需端口（必须开放）

| Port | Protocol | Purpose / 用途 |
|------|----------|----------------|
| 80 | TCP | HTTP / SSL 证书验证 |
| 443 | TCP | HTTPS |
| 7880 | TCP | LiveKit HTTP API |
| 7881 | TCP | WebRTC over TCP（UDP 不可用时的回退）|

#### 可选端口（建议开放，UDP 优先场景）

| Port | Protocol | Purpose / 用途 |
|------|----------|----------------|
| 3478 | UDP | STUN / TURN UDP |
| 5349 | TCP | TURN TLS |
| 50000-60000 | UDP | WebRTC UDP（主要音视频传输）|

**说明**:
- 如果 UDP 端口受限，至少开放必需端口，LiveKit 会自动回退到 TCP (7881)
- 音视频功能依然可用，TCP 模式延迟略高但更稳定

### 2.4 安装依赖 / Install Dependencies

```bash
# 1. 更新系统 (Update system)
sudo apt update && sudo apt upgrade -y

# 2. 安装 Docker (Install Docker)
curl -fsSL https://get.docker.com | sh

# 3. 启动并启用 Docker (Start and enable Docker)
sudo systemctl enable docker
sudo systemctl start docker

# 4. 安装 Docker Compose (Install Docker Compose)
sudo apt-get install docker-compose-plugin

# 5. 验证安装 (Verify installation)
docker --version
docker compose version
```

---

## 3. 生成配置文件 / Generate Configuration

### 3.1 使用官方工具 / Use Official Generator

LiveKit 提供官方配置生成工具，可以自动生成所需的配置文件：

```bash
# 1. 创建工作目录 (Create working directory)
sudo mkdir -p /opt/livekit
cd /opt/livekit

# 2. 运行配置生成工具 (Run configuration generator)
docker run --rm -it -v$PWD:/output livekit/generate
```

**交互输入示例**:
```
What domain would you like to use? (e.g., livekit.example.com)
> livekit.yourdomain.com

Please enter your email for SSL certificate generation:
> admin@yourdomain.com

Would you like to enable LiveKit Ingress? (y/N)
> N

Would you like to enable LiveKit Egress? (y/N)
> N
```

### 3.2 生成的文件 / Generated Files

工具会在当前目录生成以下文件：

| File / 文件 | Description / 描述 |
|-------------|---------------------|
| `livekit.yaml` | LiveKit 主配置文件 |
| `docker-compose.yaml` | Docker Compose 编排文件 |
| `caddy.yaml` | Caddy 反向代理配置 |
| `redis.conf` | Redis 配置文件 |
| `cloud_init.yaml` | 云初始化脚本（可选）|

### 3.3 配置文件详解 / Configuration Details

#### 3.3.1 livekit.yaml

```yaml
# livekit.yaml - LiveKit 主配置
# 路径: /opt/livekit/livekit.yaml

port: 7880
log_level: info

rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  # use_external_ip 应在有公网IP但未暴露给进程的环境设置为 true
  use_external_ip: true
  # 开启 TURN 服务器
  use_external_turn: true

# API 密钥配置 (从生成的文件中获取)
keys:
  <YOUR_API_KEY>: <YOUR_API_SECRET>

# Redis 配置
redis:
  address: localhost:6379

# TURN 服务器配置
turn:
  enabled: true
  tls_port: 5349
  domain: turn.yourdomain.com

# 房间配置
room:
  auto_create: true
  empty_timeout: 3600  # 1小时无参与者后自动删除
  max_participants: 100
```

#### 3.3.2 docker-compose.yaml

```yaml
# docker-compose.yaml
# 路径: /opt/livekit/docker-compose.yaml

version: '3.8'

services:
  livekit:
    image: livekit/livekit-server:latest
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml
    cap_add:
      - NET_ADMIN  # 需要管理员权限处理 UDP

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
    environment:
      - CADDY_INGRESS_ENV=caddy

volumes:
  redis-data:
  caddy-data:
```

#### 3.3.3 caddy.yaml

```yaml
# caddy.yaml - Caddy 反向代理配置
# 路径: /opt/livekit/caddy.yaml

admin off  # 禁用内置 admin API

apps:
  tls:
    certs:
      - certs:
          - key:
              - key_file: /data/caddy/pki/acme/users/default/default.key
            ocsp:
              stun_cleared: 1

  http:
    servers:
      srv0:
        listen:
          - ":80"
        routes:
          - match:
              - path: /*
            handle:
              - reverse_proxy:
                  - "127.0.0.1:7880"

      srv1:
        listen:
          - ":443"
        tls:
          certificate_selection:
            any_available_cert: true
        routes:
          - match:
              - path: /*
            handle:
              - reverse_proxy:
                  - "127.0.0.1:7880"

  logging:
    sinks:
      default:
        level: info
```

---

## 4. Docker 部署 / Docker Deployment

### 4.1 目录结构 / Directory Structure

```
/opt/livekit/
├── livekit.yaml       # LiveKit 配置
├── docker-compose.yaml # Docker Compose 配置
├── caddy.yaml         # Caddy 配置
├── redis.conf         # Redis 配置
├── redis-data/        # Redis 数据卷 (自动创建)
└── caddy-data/        # Caddy 数据卷 (自动创建)
```

### 4.2 部署步骤 / Deployment Steps

```bash
# 1. 进入工作目录
cd /opt/livekit

# 2. 启动服务 (Start services)
docker compose up -d

# 3. 查看服务状态 (Check service status)
docker compose ps

# 4. 查看日志 (View logs)
docker compose logs -f livekit
docker compose logs -f caddy
docker compose logs -f redis

# 5. 验证服务健康 (Verify health)
curl http://localhost:7880
# 应返回 LiveKit API 响应
```

### 4.3 验证部署 / Verification

```bash
# 1. 检查容器运行状态
docker compose ps
# 输出示例:
# NAME                IMAGE                    STATUS
# livekit-livekit-1   livekit/livekit-server   Up
# livekit-redis-1     redis:7-alpine           Up  
# livekit-caddy-1     caddy:2                 Up

# 2. 检查端口监听
sudo netstat -tlnp | grep -E '7880|7881|80|443'
# 应显示端口监听中

# 3. 测试 API
curl -s http://localhost:7880 | head -20
# 应返回 JSON 响应

# 4. 测试 HTTPS (需要先配置域名解析)
curl -k https://your-domain.com
# 应返回 LiveKit 响应
```

### 4.4 常用命令 / Common Commands

```bash
# 启动服务
docker compose up -d

# 停止服务
docker compose down

# 重启服务
docker compose restart

# 查看日志
docker compose logs -f

# 查看特定服务日志
docker compose logs -f livekit

# 进入容器调试
docker exec -it livekit-livekit-1 /bin/sh
docker exec -it livekit-redis-1 /bin/sh

# 查看资源使用
docker stats

# 更新镜像
docker compose pull
docker compose up -d
```

---

## 5. 阿里云安全组配置 / Aliyun Security Group

### 5.1 端口配置 / Port Configuration

**TURN 穿透服务器**: LiveKit 内置 TURN 服务器，无需额外部署。

#### 必需端口（必须开放）

| 端口范围 | 协议 | 用途 |
|---------|------|------|
| 80 | TCP | HTTP / SSL 证书验证 |
| 443 | TCP | HTTPS |
| 7880 | TCP | LiveKit HTTP API |
| 7881 | TCP | WebRTC over TCP |

#### 推荐开放（UDP 优先场景）

| 端口范围 | 协议 | 用途 |
|---------|------|------|
| 3478 | UDP | STUN / TURN UDP |
| 5349 | TCP | TURN TLS |
| 50000-60000 | UDP | WebRTC UDP |

**授权对象**: `0.0.0.0/0` (或限制为特定 IP)

### 5.2 配置步骤 / Configuration Steps

1. 登录 [阿里云 ECS 控制台](https://ecs.console.aliyun.com)
2. 找到目标实例，点击 "安全组"
3. 点击 "配置规则"
4. 添加 "入方向" 规则：
   - 协议: TCP/UDP
   - 端口: 如上表
   - 授权对象: 0.0.0.0/0
5. 保存规则

### 5.3 UDP 端口范围问题 / UDP Port Range Issue

如果无法申请大范围 UDP 端口 (50000-60000)，可以：

**方案 A**: 使用 TCP 回退 (TCP Fallback)
- LiveKit 支持 TCP 传输，端口 7881
- 在网络不稳定时自动降级到 TCP

**方案 B**: 申请成为 ECS 专有网络
- 联系阿里云客服申请

**方案 C**: 使用更小的端口范围
```yaml
# livekit.yaml
rtc:
  port_range_start: 50000
  port_range_end: 50100  # 只用 100 个端口
```

---

## 6. 后端集成 / Backend Integration

### 6.1 环境变量 / Environment Variables

在服务器环境变量或 `.env` 文件中配置：

```bash
# LiveKit 配置
LIVEKIT_URL=https://your-domain.com
LIVEKIT_API_KEY=your_api_key_from_config
LIVEKIT_API_SECRET=your_api_secret_from_config

# 保留 Daily 一段时间 (Keep Daily for gradual migration)
DAILY_API_KEY=your_daily_key

# 传输类型切换 (Transport switch)
BOT_TRANSPORT=daily  # 切换时改为 livekit
```

### 6.2 Python 依赖更新 / Python Dependencies

```toml
# server/fastapi/pyproject.toml

# 修改前
"pipecat-ai[daily,google,openai,runner,silero,simli,webrtc]>=0.0.101",

# 修改后 (同时支持 Daily 和 LiveKit)
"pipecat-ai[daily,livekit,google,openai,runner,silero,simli,webrtc]>=0.0.101",
```

安装依赖：
```bash
cd server/fastapi
pip install -e ".[daily,livekit]"
```

### 6.3 房间创建 API / Room Creation API

在 `server/fastapi/features/bot/` 目录下创建 `livekit_room.py`：

```python
# livekit_room.py
"""
LiveKit 房间创建模块
"""

import os
import json
from typing import Tuple
from livekit import api
from livekit.api import LiveKitAPI
from loguru import logger


async def create_livekit_room(session_id: str) -> Tuple[str, str, str]:
    """
    创建 LiveKit 房间并生成 Token
    
    Args:
        session_id: 会话 ID
        
    Returns:
        (room_url, bot_token, user_token)
    """
    livekit_url = os.getenv("LIVEKIT_URL")
    livekit_api_key = os.getenv("LIVEKIT_API_KEY")
    livekit_api_secret = os.getenv("LIVEKIT_API_SECRET")
    
    if not all([livekit_url, livekit_api_key, livekit_api_secret]):
        raise ValueError(
            "LiveKit 环境变量未配置: "
            "LIVEKIT_URL, LIVEKIT_API_KEY, LIVEKIT_API_SECRET"
        )
    
    lk_api = LiveKitAPI(livekit_url, livekit_api_key, livekit_api_secret)
    
    try:
        # 创建房间
        room = await lk_api.room.create(api.CreateRoom(
            name=f"bot-{session_id}",
            max_participants=2,
            empty_timeout=3600,  # 1小时无参与者后自动删除
            metadata=json.dumps({"session_id": session_id}),
        ))
        
        logger.info(f"✅ LiveKit 房间已创建: {room.name}")
        
        # 生成 Bot Token
        bot_token = api.AccessToken(
            livekit_api_key,
            livekit_api_secret
        ).add_grant(api.RoomGrant(
            room_join=True,
            room=room.name,
            can_publish=True,
            can_subscribe=True,
            can_publish_data=True,
        )).to_jwt()
        
        # 生成用户 Token
        user_token = api.AccessToken(
            livekit_api_key,
            livekit_api_secret
        ).add_grant(api.RoomGrant(
            room_join=True,
            room=room.name,
            can_publish=True,
            can_subscribe=True,
        )).to_jwt()
        
        room_url = f"{livekit_url}/{room.name}"
        logger.info(f"✅ LiveKit Token 已生成")
        
        return room_url, bot_token, user_token
        
    finally:
        await lk_api.aclose()
```

### 6.4 Bot Transport 切换 / Bot Transport Switch

修改 `server/fastapi/features/bot/interview_bot/bot.py` 和 `training_bot/bot.py`：

```python
# interview_bot/bot.py (示例)

import os
import asyncio
from pipecat.transports.daily.transport import DailyParams, DailyTransport
from pipecat.transports.livekit.transport import LiveKitParams, LiveKitTransport

# 获取传输类型配置
TRANSPORT_TYPE = os.getenv("BOT_TRANSPORT", "daily")

def get_transport(room_url: str, token: str, bot_name: str = "Bot"):
    """
    根据配置创建相应的 Transport
    
    Args:
        room_url: 房间 URL
        token: 认证 Token
        bot_name: Bot 名称
        
    Returns:
        Transport 实例
    """
    if TRANSPORT_TYPE == "livekit":
        return LiveKitTransport(
            url=room_url,
            token=token,
            room_name=room_url.split("/")[-1],
            params=LiveKitParams(
                audio_in_enabled=True,
                audio_out_enabled=True,
                video_out_enabled=True,
                video_out_width=512,
                video_out_height=512,
                video_out_framerate=5,
                # VAD 配置
                vad_analyzer=SileroVADAnalyzer(
                    params=VADParams(
                        confidence=0.85,
                        start_secs=0.2,
                        stop_secs=0.5,
                        min_volume=0.6,
                    )
                ),
            ),
        )
    else:
        # 默认使用 Daily
        return DailyTransport(
            room_url,
            token,
            bot_name,
            params=DailyParams(
                audio_in_enabled=True,
                audio_out_enabled=True,
                video_out_enabled=True,
                video_out_width=512,
                video_out_height=512,
                video_out_framerate=5,
                vad_analyzer=SileroVADAnalyzer(
                    params=VADParams(
                        confidence=0.85,
                        start_secs=0.2,
                        stop_secs=0.5,
                        min_volume=0.6,
                    )
                ),
            ),
        )

# 在 bot() 函数中使用
async def bot(runner_args: RunnerArguments):
    # ... 省略其他代码 ...
    
    # 根据配置创建 Transport
    transport = get_transport(
        runner_args.room_url,
        runner_args.token,
        "Interview Bot"
    )
    
    await run_interview_bot(transport, user_id, ...)
```

### 6.5 API 层切换 / API Layer Switch

修改 `server/fastapi/features/bot/api.py`：

```python
# api.py

import os

def get_transport_type() -> str:
    """获取传输类型配置"""
    return os.getenv("BOT_TRANSPORT", "daily")

async def create_room(session_id: str):
    """创建房间的统一入口"""
    if get_transport_type() == "livekit":
        return await create_livekit_room(session_id)
    else:
        return await create_daily_room(session_id)

# 需要添加 create_livekit_room 函数
async def create_livekit_room(session_id: str):
    """创建 LiveKit 房间"""
    from .livekit_room import create_livekit_room as _create
    return await _create(session_id)
```

---

## 7. 前端适配 / Frontend Adaptation

### 7.1 Pipecat Client 限制 / Pipecat Client Limitation

**重要提示**: 截至 2025-12，Pipecat Client JavaScript SDK 还不支持 LiveKit Transport。

**可选方案**:

| 方案 | 工作量 | 风险 | 推荐度 |
|------|--------|------|--------|
| A. 切换到 LiveKit Web SDK | 中 | 低 | ✅ 推荐 |
| B. 等待 Pipecat 官方支持 | - | 高 | ❌ 不推荐 |
| C. 自己实现 Pipecat Client | 大 | 中 | ⚠️ 备选 |

### 7.2 方案 A: 使用 LiveKit Web SDK

#### 7.2.1 安装依赖

```bash
cd frontend
npm install livekit-client
```

#### 7.2.2 创建连接组件

```typescript
// src/hooks/useLiveKit.ts
import { useState, useEffect, useRef, useCallback } from 'react';
import { Room, RoomEvent, LocalParticipant, RemoteParticipant } from 'livekit-client';

interface UseLiveKitOptions {
  url: string;
  token: string;
}

interface UseLiveKitReturn {
  room: Room | null;
  isConnected: boolean;
  error: Error | null;
  connect: () => Promise<void>;
  disconnect: () => void;
}

export function useLiveKit({ url, token }: UseLiveKitOptions): UseLiveKitReturn {
  const roomRef = useRef<Room | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const connect = useCallback(async () => {
    try {
      const room = new Room({
        adaptiveStream: true,
        dynacast: true,
      });

      room.on(RoomEvent.Connected, () => {
        setIsConnected(true);
        console.log('Connected to LiveKit room');
      });

      room.on(RoomEvent.Disconnected, () => {
        setIsConnected(false);
        console.log('Disconnected from LiveKit room');
      });

      room.on(RoomEvent.ParticipantConnected, (participant: RemoteParticipant) => {
        console.log('Participant connected:', participant.identity);
      });

      await room.connect(url, token);
      roomRef.current = room;
    } catch (err) {
      setError(err as Error);
      console.error('Failed to connect to LiveKit:', err);
    }
  }, [url, token]);

  const disconnect = useCallback(() => {
    if (roomRef.current) {
      roomRef.current.disconnect();
      roomRef.current = null;
    }
  }, []);

  useEffect(() => {
    return () => {
      disconnect();
    };
  }, [disconnect]);

  return {
    room: roomRef.current,
    isConnected,
    error,
    connect,
    disconnect,
  };
}
```

#### 7.2.3 修改 PipecatProvider

```typescript
// src/providers/PipecatProvider.tsx

import { useLiveKit } from '@/hooks/useLiveKit';

// 根据配置选择传输类型
const getTransportType = () => {
  // 可以从环境变量或配置中读取
  return import.meta.env.VITE_TRANSPORT_TYPE || 'daily';
};

export function PipecatProvider({ children }: PropsWithChildren) {
  // ... 原有逻辑 ...
  
  // 如果是 LiveKit，使用自定义连接
  if (getTransportType() === 'livekit') {
    return (
      <LiveKitProvider>
        {children}
      </LiveKitProvider>
    );
  }
  
  // 原有 Daily Transport
  return (
    <PipecatClientProvider client={client}>
      {children}
    </PipecatClientProvider>
  );
}
```

#### 7.2.4 创建 LiveKitProvider

```typescript
// src/providers/LiveKitProvider.tsx

import { PropsWithChildren, createContext, useContext, useEffect } from 'react';
import { useLiveKit } from '@/hooks/useLiveKit';

interface LiveKitContextValue {
  room: any;
  isConnected: boolean;
}

const LiveKitContext = createContext<LiveKitContextValue>({
  room: null,
  isConnected: false,
});

export function useLiveKitContext() {
  return useContext(LiveKitContext);
}

export function LiveKitProvider({ 
  children, 
  url, 
  token 
}: PropsWithChildren<{ url: string; token: string }>) {
  const { room, isConnected, connect, disconnect } = useLiveKit({ url, token });

  useEffect(() => {
    connect();
    return () => disconnect();
  }, [connect, disconnect]);

  return (
    <LiveKitContext.Provider value={{ room, isConnected }}>
      {children}
    </LiveKitContext.Provider>
  );
}
```

---

## 8. 灰度发布 / Gray Release

### 8.1 切换策略 / Switch Strategy

**方案 A: 按用户 ID 哈希**

```python
import hashlib

def should_use_livekit(user_id: str, percentage: int = 30) -> bool:
    """
    根据用户 ID 哈希值决定是否使用 LiveKit
    
    Args:
        user_id: 用户 ID
        percentage: 切换到 LiveKit 的百分比 (0-100)
        
    Returns:
        True: 使用 LiveKit, False: 使用 Daily
    """
    # 使用 MD5 哈希
    hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    # 取模得到 0-99 的值
    return (hash_value % 100) < percentage

# 使用示例
if should_use_livekit(user_id, percentage=30):
    room_url, token = await create_livekit_room(session_id)
else:
    room_url, token = await create_daily_room(session_id)
```

**方案 B: 按配置百分比随机**

```python
import random
import os

def should_use_livekit() -> bool:
    percentage = int(os.getenv("LIVEKIT_PERCENTAGE", "0"))
    return random.randint(1, 100) <= percentage
```

### 8.2 监控指标 / Monitoring Metrics

| 指标 | Daily | LiveKit | 说明 |
|------|-------|---------|------|
| 连接成功率 | - | - |  |
| 延迟 P50 | - | - |  |
| 延迟 P99 | - | - |  |
| 音频卡顿率 | - | - |  |
| 用户满意度 | - | - |  |

### 8.3 切换步骤 / Switch Steps

| 阶段 | 比例 | 对象 | 时间 |
|------|------|------|------|
| 1 | 0% | 仅开发测试 | Day 1-2 |
| 2 | 10% | 内部用户 | Day 3-4 |
| 3 | 30% | 部分外部用户 | Day 5-7 |
| 4 | 50% | 50% 用户 | Day 8-10 |
| 5 | 100% | 全量 | Day 11+ |

---

## 9. 运维 / Operations

### 9.1 日志 / Logs

```bash
# 查看 LiveKit 日志
docker compose logs -f livekit

# 查看 Redis 日志
docker compose logs -f redis

# 查看 Caddy 日志
docker compose logs -f caddy

# 按时间查看 (最近 100 行)
docker compose logs --tail=100 livekit

# 保存日志到文件
docker compose logs livekit > livekit.log
```

### 9.2 监控 / Monitoring

```bash
# 查看资源使用
docker stats

# 输出示例:
# CONTAINER           CPU %   MEM USAGE / LIMIT     NET I/O           BLOCK I/O
# livekit-livekit-1   45.23%  1.2GiB / 8GiB        125MB / 85MB      0B / 0B
# livekit-redis-1     0.15%   12MiB / 512MiB       1.2MB / 850KB    0B / 0B
# livekit-caddy-1     0.08%   25MiB / 128MiB       45MB / 32MB       0B / 0B
```

### 9.3 升级 / Upgrade

```bash
# 1. 拉取最新镜像
docker compose pull

# 2. 重启服务
docker compose up -d

# 3. 查看版本
docker compose exec livekit livekit-server --version
```

---

## 10. 故障排查 / Troubleshooting

### 10.1 常见问题 / Common Issues

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 连接失败 | 端口未开放 | 检查阿里云安全组 |
| SSL 失败 | 域名未解析 | 检查 DNS 解析 |
| TURN 失败 | UDP 端口问题 | 使用 TCP fallback (7881) |
| 延迟高 | 带宽不足 | 升级带宽 |
| 房间创建失败 | API 密钥错误 | 检查 livekit.yaml 中的 keys |

### 10.2 调试命令 / Debug Commands

```bash
# 检查端口监听
sudo netstat -tlnp | grep livekit
# 或
sudo ss -tlnp | grep livekit

# 检查 UDP 端口
sudo ss -ulnp | grep -E '3478|50000'

# 检查容器网络
docker compose exec livekit sh -c "apk add && curl http://localhost:7880"

# 测试 TURN 服务器
# 使用 https://webrtc.github.io/stun-or-turn/ 在线测试

# 检查日志中的错误
docker compose logs livekit | grep -i error

# 进入容器调试
docker compose exec livekit sh
```

### 10.3 健康检查 / Health Check

```bash
# LiveKit API 健康检查
curl -s http://localhost:7880 | head -20
# 应返回类似: {"status":"ok","version":"1.x.x"}

# 使用域名 HTTPS
curl -k https://your-domain.com
# 应返回 LiveKit API 响应
```

---

## 11. Checklist / 检查清单

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

## 参考资料 / References

### Official / 官方文档
- [LiveKit Official Docs](https://docs.livekit.io)
- [LiveKit Self-Hosted Deployment](https://docs.livekit.io/transport/self-hosting/vm/)
- [LiveKit Docker Deployment](https://docs.livekit.io/home/self-hosting/deployment)
- [LiveKit Configuration](https://docs.livekit.io/home/self-hosting/configuration)

### Pipecat / Pipecat 集成
- [Pipecat LiveKit Transport](https://docs.pipecat.ai/transports/livekit)
- [Pipecat LiveKit Example](https://docs.pipecat.ai/server/services/transport/livekit)

### 工具 / Tools
- [LiveKit Config Generator](https://github.com/livekit/deploy)
- [Caddy Documentation](https://caddyserver.com/docs/)

---

## 附录 / Appendix

### A. 环境变量完整列表

```bash
# LiveKit
LIVEKIT_URL=https://livekit.yourdomain.com
LIVEKIT_API_KEY=your_api_key
LIVEKIT_API_SECRET=your_api_secret

# 迁移开关
BOT_TRANSPORT=daily  # daily / livekit
LIVEKIT_PERCENTAGE=0  # 0-100, 灰度百分比

# 保留 Daily (共存期间)
DAILY_API_KEY=your_daily_key

# 日志级别
LOG_LEVEL=info
```

### B. 快速命令参考

```bash
# 启动
docker compose up -d

# 停止
docker compose down

# 重启
docker compose restart

# 查看状态
docker compose ps

# 查看日志
docker compose logs -f

# 更新
docker compose pull && docker compose up -d
```

---

**文档版本**: 1.0  
**最后更新**: 2026-03-14  
**作者**: Claude
