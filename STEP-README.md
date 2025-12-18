# STEP - Street Trader Ecosystem Platform

> 去中心化位置信息平台 | Decentralized Location-Based Messaging

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Flutter](https://img.shields.io/badge/Flutter-3.10+-blue.svg)](https://flutter.dev)
[![Nostr](https://img.shields.io/badge/Protocol-Nostr-purple.svg)](https://nostr.com)

## 🎯 什么是STEP？

STEP让没有"官方资质"的人也能发出声音。

在中心化平台上，街头摊贩、临时服务提供者因缺乏营业执照而被消声。STEP是一个抗审查的位置信息平台，信息的真假由用户社区判断，而非平台审核。

**核心特性**:
- 🗺️ **位置消息** - 发布和发现附近的信息
- 🔒 **端到端加密** - 私聊消息完全加密
- 🌐 **抗审查** - 基于Nostr协议，无法被关闭
- 📴 **离线优先** - 地图数据本地存储
- 💰 **代币激励** - 奖励真实信息和节点贡献

## 🏗️ 技术栈

| 层级 | 技术 |
|------|------|
| 框架 | Flutter |
| 消息协议 | Nostr (NIP-44加密) |
| 地图 | MapLibre + PMTiles |
| 地图分发 | BitTorrent P2P |
| 地理索引 | Uber H3 |
| 身份 | Ed25519 (did:key) |

## 📱 截图

*开发中...*

## 🚀 快速开始

### 前置要求

- Flutter 3.10+
- Dart 3.0+
- Android Studio / Xcode

### 安装

```bash
# 克隆仓库
git clone https://github.com/yourusername/step-app.git
cd step-app

# 安装依赖
flutter pub get

# 运行
flutter run
```

## 📖 文档

- [技术规格文档](docs/STEP-TECHNICAL-SPECIFICATION.md)
- [开发指南](docs/DEVELOPMENT.md)
- [贡献指南](CONTRIBUTING.md)

## 🗺️ 开发路线图

- [x] Phase 0: 项目骨架
- [ ] Phase 1: Nostr基础 + 列表消息
- [ ] Phase 2: 地图集成
- [ ] Phase 3: 离线地图P2P分发
- [ ] Phase 4: 加密私聊
- [ ] Phase 5: STEP代币激励
- [ ] Phase 6: 发布

## 🤝 贡献

欢迎贡献！请查看 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 📄 许可证

MIT License - 详见 [LICENSE](LICENSE)

## 🔗 相关链接

- [Nostr协议](https://github.com/nostr-protocol/nostr)
- [MapLibre](https://maplibre.org/)
- [Planetiler](https://github.com/onthegomap/planetiler)

---

*STEP - 让每个人的声音都能被听到*
