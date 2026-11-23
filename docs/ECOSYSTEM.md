# 生态：三个项目一起用

Playable 是三个自建项目里负责**播放**的那一环。三个项目相互引用、各自独立。

| 项目 | 定位 | 仓库 |
|---|---|---|
| **Docker-Link** | 运维整个栈 | [lichangzhou/Docker-link](https://github.com/lichangzhou/Docker-link) |
| **EvoLink** | 治理媒体库 | [lichangzhou/Evolink](https://github.com/lichangzhou/Evolink) · [EvoLink-CN](https://github.com/lichangzhou/EvoLink-CN) |
| **Playable** | 播放端 | [lichangzhou/Playable](https://github.com/lichangzhou/Playable) |

---

## 怎么配合

```
┌──────────────────────────────────────────────────────────────┐
│  Docker-Link                                                 │
│  容器 · 镜像 · 卷 · 网络 · Compose                          │
│  文件管理 · Web 终端 · 远程节点 · 监控告警                   │
│  → 把下面两个跑起来并盯着它们                                │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  EvoLink                                                     │
│  索引 Emby / Plex / Jellyfin / Playable                      │
│  与发行目录比对 → 找出缺口                                   │
│  归档 · 去重 · 分类 · 签发 302 链接                          │
│  → 维护 /app/Link，也就是 .strm 媒体库                       │
└───────────────────────────┬──────────────────────────────────┘
                            │  .strm 文件
                            │  http://…/api/swap115/<path>
                            ▼
┌──────────────────────────────────────────────────────────────┐
│  Playable                                                    │
│  Android · Windows · iOS                                    │
│  播放 .strm 媒体库，自动跟随 302 跳转                        │
│  → 电视、手机、投影仪上的客户端                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 配合细节

### Docker-Link → Playable

Docker-Link 负责把服务端那一侧（EvoLink、媒体服务器）部署好并盯着它跑；
Playable 是客户端侧的一环，通常直接装在电视、手机和投影仪上。

### EvoLink → Playable

EvoLink 产出媒体库，Playable 消费媒体库：

- **`.strm` 格式就是契约** —— 一个 `.strm` 文件只有一行 URL，Playable 解析它、跟随 `302`、
  播放网盘直链
- **`302` 换链** —— 网盘路径不会离开服务端，播放器拿到的只是一条带签名、会过期的直链
- **签发与取链使用同一个 User-Agent** —— 这条约定由 EvoLink 保证，播放器只需正常跟随跳转

### Playable → EvoLink

Playable 同时也是一个**索引源**。EvoLink 可以把 Playable 媒体库与 Emby / Plex / Jellyfin
一起索引，所以只在 Playable 上存在的作品同样会被计入库存，不会被当成缺失。

---

## 一套完整用法

1. **Docker-Link** —— 部署整个栈：EvoLink、媒体服务器、Playable 服务端
2. **EvoLink** —— 登录、添加媒体服务器、开始索引，然后跑一次比对看缺了什么；
   配好分类规则，生成归档计划，写出 `.strm`
3. **Playable** —— 指向同一个 `.strm` 库，它会拉取目录并直接播放

### 推荐端口

| 服务 | 端口 | 说明 |
|---|---|---|
| Docker-Link | `9080` | 面板；Agent 通道 `9081` |
| EvoLink | `9191` | 面板与 API |
| EvoLink → Emby | `8093` | `302` 反代 |
| EvoLink → Jellyfin | `8099` | `302` 反代 |
| EvoLink → Plex | `32411` | `302` 反代 |
| Playable 服务端 | `8092` | 可选 |

---

## 为什么拆成三个项目

因为它们解决的问题不同、生命周期也不同：

- **Docker-Link** 很少换 —— 它是基础设施
- **EvoLink** 很少换 —— 它贴着数据，索引库和规则都在 `data/` 里，那只是一个普通目录
- **Playable** 换得最勤 —— 它是客户端，要跟着新的设备形态走

拆开的收益是：升级其中一个不会把另外两个一起拖下水，而且每一个都可以凭自己的价值
单独被推荐。

---

## 互相推广

三个项目的 README 都有一节生态说明，各带一份同样结构的 `docs/ECOSYSTEM.md`。
用户点进任意一个仓库，都能顺着链接走完整条链路。

**为什么这有效**：单项目推广的转化瓶颈是「用户不知道它能解决什么」；三项目互推把
「一个工具」变成「一套方案」，认知成本大幅下降，同时每个项目的 star 都在为另外两个导流。
