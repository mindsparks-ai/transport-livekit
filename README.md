# transport-livekit

Documentation and deployment guides for migrating from Daily.co to LiveKit.

## Docs

### English
- [Approach Comparison](en/comparison.md) — Comparison of two migration approaches
- [Self-Hosted Deployment Guide](en/self-hosted.md) — Complete guide for self-hosting LiveKit on Alibaba Cloud

### 中文
- [方案对比](zh/comparison.md) — Daily vs LiveKit 两种迁移方案的对比分析
- [自建部署指南](zh/self-hosted.md) — 阿里云自建 LiveKit 服务器完整指南

## Background

Currently using Daily.co as the WebRTC real-time audio/video transport layer. The goal is to migrate to LiveKit to achieve:

- Lower latency in China (server hosted on Alibaba Cloud)
- Data sovereignty (audio/video stays within China)
- Predictable cost (no per-usage SaaS fees)

See [Approach Comparison](en/comparison.md) for the two migration paths.
