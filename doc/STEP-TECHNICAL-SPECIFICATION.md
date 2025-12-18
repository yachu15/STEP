# STEP 技术规格文档 v1.0

> **S**treet **T**rader **E**cosystem **P**latform  
> 去中心化位置信息平台 - 让没有"官方资质"的人也能发出声音

**文档版本**: 1.0  
**最后更新**: 2024年12月  
**状态**: 可执行

---

## 1. 项目概述

### 1.1 核心使命

STEP是一个抗审查的位置信息平台，服务于街头摊贩、临时服务提供者和非正式经济参与者。在中心化平台上，这些人因缺乏营业执照、无法通过内容审核而被消声。STEP让信息的真假由用户社区判断，而非平台审核。

### 1.2 核心约束

| 约束 | 说明 |
|------|------|
| **抗审查** | 像比特币那样"无法被关闭"，任何单点都可被替代 |
| **零成本运营** | 不依赖付费服务，利用免费资源和P2P |
| **穿墙能力** | 参考币安/OKX，能在中国防火墙环境下使用 |
| **离线优先** | 地图数据本地存储，消息可延迟同步 |

### 1.3 目标用户

- 街边摊贩（煎饼、水果、小吃）
- 临时服务（搬家、维修、跑腿）
- 社区信息（拼车、二手、失物）
- 任何需要基于位置分享信息但无法通过正规平台的人

---

## 2. 系统架构

### 2.1 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户设备 (Flutter)                        │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   地图层    │  │   消息层    │  │   激励层    │             │
│  │ MapLibre GL │  │ Nostr协议   │  │ STEP代币    │             │
│  │ + PMTiles   │  │ + NIP-44    │  │ (应用内)    │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│  ┌──────┴────────────────┴────────────────┴──────┐             │
│  │              本地存储层 (SQLite + 文件系统)      │             │
│  │  - 地图瓦片 (PMTiles)                          │             │
│  │  - 消息缓存                                    │             │
│  │  - 密钥存储 (flutter_secure_storage)           │             │
│  └───────────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ Nostr Relay  │  │ Nostr Relay  │  │ Nostr Relay  │
    │ (社区运营)   │  │ (自建)       │  │ (公共)       │
    └──────────────┘  └──────────────┘  └──────────────┘
            │                 │                 │
            └─────────────────┼─────────────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
            ┌──────────────┐    ┌──────────────┐
            │ 地图种子服务器 │    │ 激励节点     │
            │ (BT/HTTP)    │    │ (质押存储)   │
            └──────────────┘    └──────────────┘
```

### 2.2 数据流

```
[用户发布消息]
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│ 1. 获取GPS坐标 (WGS-84)                                  │
│ 2. 计算H3索引 (resolution 9, 约174米六边形)              │
│ 3. 构造Nostr事件:                                        │
│    - kind: 30078 (可替换参数化事件)                      │
│    - tags: ["g", "H3索引"], ["t", "类别"]                │
│    - content: 加密或明文消息内容                         │
│ 4. 签名 (Ed25519/secp256k1)                             │
│ 5. 发布到已连接的Relay                                   │
└─────────────────────────────────────────────────────────┘
     │
     ▼
[其他用户接收]
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│ 1. 订阅当前位置H3索引 + 周围k-ring (7-19个Topic)         │
│ 2. 接收匹配的Nostr事件                                   │
│ 3. 验证签名                                              │
│ 4. 显示在地图/列表上                                     │
└─────────────────────────────────────────────────────────┘
```

---

## 3. 技术栈详细说明

### 3.1 开发框架: Flutter

**选择理由**:
- 跨平台一致性优于React Native
- Dart生态已有成熟的Nostr、地图、加密库
- 阿里巴巴、字节跳动等中国公司大规模使用，证明GFW环境可用
- 单一代码库降低维护成本

**版本要求**:
```yaml
environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'
```

### 3.2 消息层: Nostr协议

**核心包**:
```yaml
dependencies:
  ndk: ^0.6.0              # Nostr Development Kit
  nostr: ^1.5.0            # 协议实现，含NIP-44
  web_socket_client: ^1.0.0 # 带自动重连的WebSocket
  bip340: ^0.3.0           # Schnorr签名
```

**关键NIP支持**:

| NIP | 用途 | 说明 |
|-----|------|------|
| NIP-01 | 基础协议 | 事件结构和签名 |
| NIP-04 | 加密私信 | 基础端到端加密 |
| NIP-44 | 加密私信v2 | 更安全的加密，已审计 |
| NIP-65 | Relay列表 | 用户公开自己使用的relay |
| NIP-52 | 日历事件 | 可用于限时消息 |
| 自定义 | 地理消息 | kind 30078 + "g"标签存H3索引 |

**Relay策略**:
```dart
// 多relay冗余连接
final relays = [
  'wss://relay.damus.io',      // 公共relay
  'wss://nos.lol',             // 公共relay
  'wss://relay.step.network',  // 自建relay (示例)
  'wss://user-added-relay',    // 用户自定义
];

// 连接失败自动切换
// 用户可手动添加/删除relay
```

### 3.3 地图层: MapLibre + PMTiles

**核心包**:
```yaml
dependencies:
  maplibre_gl: ^0.24.1      # 地图渲染
  h3_flutter: ^0.5.0        # H3六边形索引
  coordtransform_dart: ^1.0.0 # GCJ-02坐标转换
```

**离线地图加载**:
```dart
// PMTiles本地加载 (需MapLibre Native 11.7+)
MaplibreMap(
  styleString: 'pmtiles://file:///path/to/nanchang.pmtiles',
  // ...
)

// 或使用MBTiles (更成熟)
MaplibreMap(
  styleString: 'mbtiles://file:///path/to/nanchang.mbtiles',
  // ...
)
```

**坐标转换 (中国大陆)**:
```dart
import 'package:coordtransform_dart/coordtransform_dart.dart';

// GPS坐标 (WGS-84) -> 显示坐标 (GCJ-02)
// 注意: 地图瓦片保持WGS-84，只转换用户定位点
List<double> displayCoord = CoordTransform.wgs84togcj02(lng, lat);

// 用户选点 -> 存储坐标 (保持WGS-84)
List<double> storageCoord = CoordTransform.gcj02towgs84(displayLng, displayLat);
```

### 3.4 P2P地图分发

**架构**:
```
┌────────────────────────────────────────────────────────────┐
│                     地图数据生产端                          │
├────────────────────────────────────────────────────────────┤
│  1. 下载OSM数据: download.geofabrik.de/asia/china.html     │
│  2. Planetiler生成PMTiles:                                 │
│     java -jar planetiler.jar --osm-path=china.osm.pbf      │
│          --output=china.pmtiles --maxzoom=14               │
│  3. 按城市/省份切割 (可选)                                  │
│  4. 生成磁力链接，发布到Nostr (kind 30000)                  │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                     地图数据分发                            │
├────────────────────────────────────────────────────────────┤
│  Android:                                                   │
│  - 优先: dtorrent_task_v2 (纯Dart BitTorrent)              │
│  - 备选: HTTP直接下载                                       │
│                                                            │
│  iOS:                                                       │
│  - 仅HTTP下载 (系统限制无法后台做种)                        │
│                                                            │
│  种子服务器:                                                │
│  - Oracle Cloud免费层 (2台ARM实例)                          │
│  - 用户自建 (家用电脑/NAS)                                  │
│  - 激励节点 (质押STEP获得奖励)                              │
└────────────────────────────────────────────────────────────┘
```

**Nostr地图索引事件 (kind 30000)**:
```json
{
  "kind": 30000,
  "tags": [
    ["d", "step-map-index"],
    ["region", "china-nanchang"],
    ["version", "2024-12"],
    ["size", "215000000"],
    ["magnet", "magnet:?xt=urn:btih:..."],
    ["http", "https://backup.example.com/nanchang.pmtiles"]
  ],
  "content": "南昌市地图包 2024年12月版",
  "created_at": 1702800000
}
```

### 3.5 身份与加密

**密钥生成**:
```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:bip340/bip340.dart';

// 生成新密钥对
String privateKey = generatePrivateKey(); // 32字节随机数
String publicKey = getPublicKey(privateKey);

// 安全存储
final storage = FlutterSecureStorage();
await storage.write(key: 'nostr_private_key', value: privateKey);

// did:key格式 (可选，用于跨协议互操作)
String did = 'did:key:z${base58Encode(publicKeyBytes)}';
```

**消息加密 (NIP-44)**:
```dart
import 'package:nostr/nostr.dart';

// 加密私信
String encrypted = nip44Encrypt(
  plaintext: '消息内容',
  senderPrivateKey: myPrivateKey,
  recipientPublicKey: theirPublicKey,
);

// 解密私信
String decrypted = nip44Decrypt(
  ciphertext: encrypted,
  recipientPrivateKey: myPrivateKey,
  senderPublicKey: theirPublicKey,
);
```

### 3.6 STEP代币 (应用内)

**设计原则**:
- 仅应用内流通，不做跨链
- 签名余额声明，无需区块链
- 激励真实信息和节点贡献

**代币用途**:

| 场景 | 消耗/获得 | 说明 |
|------|----------|------|
| 发布地图消息 | 消耗 | 质押代币由激励节点存储并广播 |
| 提供有价值信息 | 获得 | 被点赞/感谢获得代币 |
| 运行激励节点 | 获得 | 存储和转发消息获得质押分成 |
| 做种地图数据 | 获得 | P2P上传流量奖励 |

**简化实现 (MVP)**:
```dart
// 签名余额声明
class BalanceStatement {
  final String pubkey;
  final int balance;
  final int timestamp;
  final String signature;
  
  // 验证签名有效性
  bool verify() { /* ... */ }
}

// 后续可升级为Cashu盲签名代币
```

---

## 4. 地图数据流水线

### 4.1 生产流程 (项目方/社区维护者)

```bash
# 1. 下载OSM数据
wget https://download.geofabrik.de/asia/china-latest.osm.pbf

# 2. (可选) 提取特定区域
osmium extract -p nanchang.poly china-latest.osm.pbf -o nanchang.osm.pbf

# 3. 生成PMTiles
java -Xmx8g -jar planetiler.jar \
  --osm-path=nanchang.osm.pbf \
  --output=nanchang.pmtiles \
  --maxzoom=15 \
  --download

# 4. 创建种子文件
transmission-create -o nanchang.pmtiles.torrent nanchang.pmtiles

# 5. 发布到Nostr
# (使用脚本发布kind 30000事件，包含磁力链接)
```

### 4.2 客户端下载流程

```dart
class MapDownloader {
  // 1. 从Nostr获取地图索引
  Future<MapPackage?> findMapPackage(String region) async {
    final events = await nostr.query(
      kinds: [30000],
      tags: {'d': ['step-map-index'], 'region': [region]},
    );
    return events.isNotEmpty ? MapPackage.fromEvent(events.first) : null;
  }
  
  // 2. 下载地图
  Future<void> downloadMap(MapPackage package) async {
    if (Platform.isAndroid && await canUseTorrent()) {
      // Android: 优先使用种子下载
      await torrentDownload(package.magnetLink);
    } else {
      // iOS或种子失败: HTTP下载
      await httpDownload(package.httpUrl);
    }
  }
  
  // 3. 验证完整性
  Future<bool> verifyMap(String filePath, String expectedHash) async {
    final hash = await computeSha256(filePath);
    return hash == expectedHash;
  }
}
```

### 4.3 地图更新策略

- **更新频率**: 每月发布新版本
- **增量更新**: 暂不实现，直接全量替换
- **版本共存**: 旧版本继续可用，新版本发布后提示更新
- **自动清理**: 保留最近2个版本，自动删除更旧的

---

## 5. 穿墙策略

### 5.1 借鉴币安/OKX模式

```
普通App:
  用户 → 防火墙 → 被墙网站 ❌

币安/OKX模式:
  用户 → App内置代理 → 币安服务器 → 目标内容 ✅
```

**STEP实现**:

1. **多Relay冗余**: 连接多个Nostr relay，被封一个自动切换
2. **自建Relay**: 用户可添加自建relay地址
3. **流量伪装**: Nostr WebSocket看起来像普通HTTPS
4. **域名轮换**: relay域名被封后快速更换

### 5.2 Relay部署建议

```yaml
# 推荐配置: 3个不同司法管辖区的relay
relays:
  - location: 新加坡
    provider: Oracle Cloud (免费)
    domain: sg.relay.example.com
    
  - location: 欧洲
    provider: Hetzner (~$5/月)
    domain: eu.relay.example.com
    
  - location: 美国
    provider: DigitalOcean (~$5/月)
    domain: us.relay.example.com
```

### 5.3 应对封锁流程

```
1. 监测: App定期检测relay连接状态
2. 报警: 连接失败超过阈值触发
3. 切换: 自动切换到备用relay
4. 更新: 通过Nostr/其他渠道分发新relay地址
5. 恢复: 用户手动或自动添加新relay
```

---

## 6. 开发阶段划分

### Phase 0: 环境搭建 (1周)

**目标**: 建立开发环境和项目骨架

**任务**:
- [ ] 创建Flutter项目，配置依赖
- [ ] 设置GitHub仓库
- [ ] 配置CI/CD (GitHub Actions)
- [ ] 创建基础文件结构

**产出**:
```
step_app/
├── lib/
│   ├── main.dart
│   ├── core/           # 核心服务
│   ├── features/       # 功能模块
│   ├── shared/         # 共享组件
│   └── config/         # 配置
├── test/
├── pubspec.yaml
└── README.md
```

---

### Phase 1: Nostr基础 + 列表消息 (2-3周)

**目标**: 实现消息收发核心功能，无地图

**任务**:
- [ ] 密钥生成和安全存储
- [ ] Nostr relay连接管理
- [ ] 消息发布 (kind 30078 + H3标签)
- [ ] 消息订阅和接收
- [ ] 基于位置的列表显示
- [ ] 消息详情页

**核心代码结构**:
```dart
// lib/core/nostr/
├── nostr_service.dart      // 主服务
├── relay_manager.dart      // relay连接管理
├── event_builder.dart      // 事件构造
└── subscription_manager.dart // 订阅管理

// lib/features/messages/
├── message_list_screen.dart
├── message_detail_screen.dart
├── compose_message_screen.dart
└── widgets/
```

**验收标准**:
- 能生成密钥并安全存储
- 能连接至少2个公共relay
- 能发布带位置的消息
- 能接收并显示附近的消息
- 无地图，用列表+距离+方向显示

---

### Phase 2: 地图集成 (2-3周)

**目标**: 集成MapLibre，显示地图和消息标记

**任务**:
- [ ] MapLibre基础集成
- [ ] 在线瓦片显示
- [ ] 用户定位显示
- [ ] 消息标记渲染
- [ ] 点击标记显示详情
- [ ] H3六边形可视化 (可选)
- [ ] GCJ-02坐标转换

**验收标准**:
- 地图正常显示 (在线瓦片)
- 用户位置正确标注
- 消息以标记形式显示在地图上
- 中国大陆坐标偏移问题已处理

---

### Phase 3: 离线地图 (2-3周)

**目标**: 实现地图数据本地存储和P2P分发

**任务**:
- [ ] PMTiles/MBTiles本地加载
- [ ] 地图包下载管理器
- [ ] Nostr地图索引查询
- [ ] HTTP下载实现
- [ ] Android种子下载实现 (dtorrent)
- [ ] 下载进度UI
- [ ] 地图版本管理

**验收标准**:
- 能从HTTP下载南昌市地图包
- 能加载本地地图文件
- Android能通过磁力链接下载
- 无网络时地图仍可使用

---

### Phase 4: 私信聊天 (2周)

**目标**: 实现1v1加密聊天

**任务**:
- [ ] NIP-44加密实现
- [ ] 聊天会话管理
- [ ] 消息收发
- [ ] 聊天列表UI
- [ ] 聊天详情UI
- [ ] 离线消息缓存

**验收标准**:
- 能与其他用户建立加密聊天
- 消息端到端加密
- 支持离线查看历史消息

---

### Phase 5: STEP代币激励 (3-4周)

**目标**: 实现基础代币系统

**任务**:
- [ ] 余额管理
- [ ] 代币转账
- [ ] 发布消息质押
- [ ] 点赞/感谢奖励
- [ ] 交易历史
- [ ] 激励节点框架

**验收标准**:
- 用户有代币余额
- 发布消息消耗代币
- 被感谢获得代币
- 交易记录可查

---

### Phase 6: 打磨与发布 (2-3周)

**目标**: 优化体验，准备发布

**任务**:
- [ ] UI/UX优化
- [ ] 性能优化
- [ ] 错误处理完善
- [ ] 多语言支持 (中/英)
- [ ] 用户引导
- [ ] Beta测试
- [ ] 应用商店提交

---

## 7. Claude Code 执行指令

以下指令可直接复制给Claude Code执行。

### 指令0: 项目初始化

```
请为STEP项目创建Flutter项目骨架:

1. 创建Flutter项目:
   flutter create step_app --org com.step --platforms android,ios

2. 配置pubspec.yaml，添加以下依赖:
   - ndk: ^0.6.0
   - nostr: ^1.5.0
   - maplibre_gl: ^0.24.1
   - h3_flutter: ^0.5.0
   - coordtransform_dart: ^1.0.0
   - flutter_secure_storage: ^9.0.0
   - web_socket_client: ^1.0.0
   - bip340: ^0.3.0
   - provider: ^6.0.0
   - go_router: ^12.0.0
   - sqflite: ^2.3.0
   
3. 创建目录结构:
   lib/
   ├── main.dart
   ├── app.dart
   ├── core/
   │   ├── nostr/
   │   ├── storage/
   │   ├── location/
   │   └── crypto/
   ├── features/
   │   ├── home/
   │   ├── messages/
   │   ├── map/
   │   ├── chat/
   │   └── profile/
   ├── shared/
   │   ├── widgets/
   │   ├── models/
   │   └── utils/
   └── config/
       ├── routes.dart
       ├── theme.dart
       └── constants.dart

4. 配置Android权限 (位置、网络、存储)

5. 配置iOS权限 (位置、网络)

6. 创建基础路由配置

7. 确保项目能成功编译运行
```

### 指令1: Nostr核心服务

```
基于STEP项目，实现Nostr核心服务:

1. 创建 lib/core/nostr/nostr_service.dart:
   - 单例模式
   - 管理relay连接
   - 发布事件
   - 订阅事件
   - 自动重连

2. 创建 lib/core/nostr/relay_manager.dart:
   - 连接多个relay
   - 连接状态监控
   - 失败自动切换
   - 用户自定义relay

3. 创建 lib/core/nostr/event_builder.dart:
   - 构造地理消息事件 (kind 30078)
   - 构造私信事件 (kind 4/14)
   - 签名事件

4. 创建 lib/core/crypto/key_manager.dart:
   - 生成新密钥对
   - 安全存储密钥
   - 导入/导出密钥

5. 默认relay列表:
   - wss://relay.damus.io
   - wss://nos.lol
   - wss://relay.nostr.band

6. 编写单元测试验证基础功能
```

### 指令2: 位置消息功能

```
基于STEP项目，实现位置消息功能:

1. 创建 lib/core/location/location_service.dart:
   - 获取当前位置
   - 权限管理
   - 持续定位更新

2. 创建 lib/core/location/h3_service.dart:
   - 坐标转H3索引 (resolution 9)
   - 计算k-ring邻居
   - H3索引转边界多边形

3. 创建 lib/shared/models/geo_message.dart:
   - 消息模型
   - 从Nostr事件解析
   - 序列化为Nostr事件

4. 创建消息列表页面 lib/features/messages/:
   - 显示附近消息列表
   - 按距离排序
   - 显示距离和方向
   - 下拉刷新
   - 点击查看详情

5. 创建消息发布页面:
   - 输入消息内容
   - 选择类别
   - 显示当前位置
   - 发布按钮

6. 实现位置订阅:
   - 订阅当前H3 + k-ring(k=2) 的所有消息
   - 位置变化时更新订阅
```

### 指令3: 地图集成

```
基于STEP项目，集成MapLibre地图:

1. 配置MapLibre:
   - Android: 添加必要的gradle配置
   - iOS: 添加必要的podfile配置

2. 创建 lib/features/map/map_screen.dart:
   - 全屏地图视图
   - 用户位置标记
   - 消息标记图层
   - 点击标记显示卡片

3. 创建 lib/features/map/widgets/message_marker.dart:
   - 自定义标记样式
   - 按类别显示不同图标
   - 聚合密集标记

4. 创建 lib/features/map/widgets/message_card.dart:
   - 底部卡片显示消息详情
   - 滑动关闭
   - 查看详情/联系发布者按钮

5. 实现坐标转换:
   - 检测设备是否在中国大陆
   - 自动应用GCJ-02转换
   - 确保标记位置准确

6. 初始使用在线瓦片:
   - https://api.maptiler.com/maps/streets/{z}/{x}/{y}.png?key=YOUR_KEY
   - 或其他免费瓦片服务
```

---

## 8. 文件清单

本项目需要以下文件，请按顺序创建:

```
step_app/
├── pubspec.yaml                          # 依赖配置
├── lib/
│   ├── main.dart                         # 入口
│   ├── app.dart                          # App组件
│   ├── core/
│   │   ├── nostr/
│   │   │   ├── nostr_service.dart        # Nostr主服务
│   │   │   ├── relay_manager.dart        # Relay管理
│   │   │   ├── event_builder.dart        # 事件构造
│   │   │   └── subscription_manager.dart # 订阅管理
│   │   ├── storage/
│   │   │   ├── local_storage.dart        # 本地存储
│   │   │   └── database.dart             # SQLite数据库
│   │   ├── location/
│   │   │   ├── location_service.dart     # 位置服务
│   │   │   └── h3_service.dart           # H3索引服务
│   │   └── crypto/
│   │       ├── key_manager.dart          # 密钥管理
│   │       └── encryption.dart           # 加密工具
│   ├── features/
│   │   ├── home/
│   │   │   └── home_screen.dart          # 首页
│   │   ├── messages/
│   │   │   ├── message_list_screen.dart  # 消息列表
│   │   │   ├── message_detail_screen.dart# 消息详情
│   │   │   └── compose_screen.dart       # 发布消息
│   │   ├── map/
│   │   │   ├── map_screen.dart           # 地图页
│   │   │   └── widgets/
│   │   │       ├── message_marker.dart   # 消息标记
│   │   │       └── message_card.dart     # 消息卡片
│   │   ├── chat/
│   │   │   ├── chat_list_screen.dart     # 聊天列表
│   │   │   └── chat_detail_screen.dart   # 聊天详情
│   │   └── profile/
│   │       ├── profile_screen.dart       # 个人页
│   │       └── settings_screen.dart      # 设置页
│   ├── shared/
│   │   ├── widgets/
│   │   │   └── common_widgets.dart       # 公共组件
│   │   ├── models/
│   │   │   ├── geo_message.dart          # 地理消息模型
│   │   │   ├── user.dart                 # 用户模型
│   │   │   └── chat_message.dart         # 聊天消息模型
│   │   └── utils/
│   │       ├── coord_transform.dart      # 坐标转换
│   │       └── helpers.dart              # 工具函数
│   └── config/
│       ├── routes.dart                   # 路由配置
│       ├── theme.dart                    # 主题配置
│       └── constants.dart                # 常量定义
├── android/
│   └── app/src/main/AndroidManifest.xml  # Android权限
├── ios/
│   └── Runner/Info.plist                 # iOS权限
└── test/
    └── (测试文件)
```

---

## 9. 部署与运维

### 9.1 Nostr Relay部署

**使用strfry (推荐)**:
```bash
# Oracle Cloud免费ARM实例
docker run -d \
  --name strfry \
  -p 7777:7777 \
  -v /data/strfry:/app/strfry-db \
  ghcr.io/hoytech/strfry:latest

# Nginx反向代理配置
server {
    listen 443 ssl;
    server_name relay.yourdomain.com;
    
    location / {
        proxy_pass http://localhost:7777;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 9.2 地图种子服务器

```bash
# 安装transmission (BT客户端)
apt install transmission-daemon

# 配置做种
transmission-remote -a nanchang.pmtiles.torrent
transmission-remote -t 1 --start
```

### 9.3 监控

- **Relay状态**: 定期检查WebSocket连接
- **种子健康**: 监控peer数量和下载速度
- **用户反馈**: 应用内反馈渠道

---

## 10. 风险与应对

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|----------|
| Relay被封 | 高 | 中 | 多relay冗余，用户可自添加 |
| 地图源被封 | 中 | 高 | P2P分发，多种子服务器 |
| 应用商店下架 | 中 | 高 | APK直接分发，F-Droid |
| 恶意内容 | 中 | 中 | 社区举报，用户屏蔽 |
| 代币滥用 | 低 | 低 | 质押机制，信誉系统 |

---

## 附录A: 术语表

| 术语 | 解释 |
|------|------|
| Nostr | Notes and Other Stuff Transmitted by Relays，去中心化消息协议 |
| Relay | Nostr网络中的消息中继服务器 |
| NIP | Nostr Implementation Possibilities，Nostr协议扩展标准 |
| PMTiles | Protomaps Tiles，单文件地图瓦片格式 |
| H3 | Uber的六边形地理索引系统 |
| GCJ-02 | 中国国测局坐标系，俗称"火星坐标" |
| WGS-84 | GPS使用的全球标准坐标系 |
| DHT | 分布式哈希表，用于P2P节点发现 |

---

## 附录B: 参考资源

**Nostr协议**:
- https://github.com/nostr-protocol/nostr
- https://github.com/nostr-protocol/nips

**MapLibre**:
- https://maplibre.org/
- https://pub.dev/packages/maplibre_gl

**Planetiler**:
- https://github.com/onthegomap/planetiler

**Flutter包**:
- https://pub.dev/packages/ndk
- https://pub.dev/packages/h3_flutter

---

*本文档由Claude生成，基于与Song的详细讨论整理*
