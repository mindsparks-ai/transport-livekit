# 迁移方案对比：Daily → LiveKit

目前有两种方案可以将现有 Pipecat 框架从 Daily.co 迁移到 LiveKit。

---

## 方案一：Pipecat 切换到 LiveKit Transport

### 改动范围

| 层级 | 需要修改的文件 | 改动内容 |
|------|--------------|---------|
| 后端依赖 | `pyproject.toml` | `pipecat-ai[daily,...]` → `pipecat-ai[livekit,...]` |
| 后端 Transport | `interview_bot/bot.py`、`training_bot/bot.py` | `DailyTransport` → `LiveKitTransport` |
| 房间管理 | `api.py` | Daily API → LiveKit Server SDK 创建房间/Token |
| 前端依赖 | `package.json` | `@pipecat-ai/daily-transport` → `@pipecat-ai/livekit-transport` |
| 前端 Provider | `PipecatProvider.tsx` | `DailyTransport` → `LiveKitTransport` |

### 优势

- 改动相对集中 — Pipecat 抽象了 Transport 层，Pipeline 逻辑无需改动
- 可先用 LiveKit Cloud 测试，后期再切换到自建服务器
- Pipecat 官方支持 LiveKit Transport

### 劣势

- 需要验证 VAD、视频输出等功能在 LiveKit Transport 下是否完善
- 使用 LiveKit Cloud 时，国内访问可能仍有延迟

---

## 方案二：Docker 自建 LiveKit 服务端

### 部署架构

```
┌─────────────────────────────────────────────────────┐
│                      你的服务器                       │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │   LiveKit    │  │   FastAPI    │  │   Redis   │  │
│  │   Server     │  │   + Bots     │  │  (可选)   │  │
│  │ :7880/:7881  │  │    :8000     │  │           │  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
│         ↑                 ↑                          │
│         └─────────────────┘                          │
│              WebRTC + 内部 API                        │
└─────────────────────────────────────────────────────┘
              ↑
          用户浏览器
```

### Docker Compose 示例

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

### 优势

- 完全自主可控，数据不经过第三方
- 服务器部署在国内，用户直连，延迟最优
- 无 SaaS 费用，只有服务器成本
- 音视频数据不出境，满足隐私合规要求

### 劣势

- 运维复杂度高：需要配置 STUN/TURN、NAT 穿透、TLS 证书、公网 IP 及 UDP 端口
- 高并发时需要自己做负载均衡
- 排查音视频问题需要深入理解 WebRTC

---

## 关键决策因素对比

| 维度 | 方案一（Pipecat + LiveKit Cloud） | 方案二（自建 LiveKit） |
|------|----------------------------------|----------------------|
| 开发工作量 | 中等（替换 Transport） | 中等 + 运维工作 |
| 国内延迟 | 取决于 LiveKit Cloud 节点 | 最优（自建国内服务器） |
| 成本 | LiveKit Cloud 按量付费 | 服务器固定成本 |
| 运维负担 | 低 | 高（需要 WebRTC 运维经验） |
| 数据合规 | 数据经过 LiveKit Cloud | 完全自主 |
| 上线速度 | 快（1-2 周） | 慢（需要调试网络） |
| 扩展性 | LiveKit Cloud 自动扩展 | 需要自己做 |
