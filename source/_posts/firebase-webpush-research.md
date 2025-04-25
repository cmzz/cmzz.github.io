---
title: Firebase WebPush 技术能力调研
description: 对 Firebase WebPush 技术的全面调研，包括传统 GCM 与最新 Firebase Cloud Messaging 的对比、Service Worker 支持、生命周期与限制、延迟与优先级、监控方案、广告应用场景等方面的详细分析。
keyword: Firebase, WebPush, Push API, Service Worker, FCM
date: 2025-04-24 10:00:00
tags: 
- 技术
- Firebase
- WebPush
- Push API
- Service Worker
---


## 1. 传统 GCM WebPush 的优势与弊端

| 维度 | 优势 | 弊端 |
| ---- | ---- | ---- |
| 历史沉淀 | 最早期 Google 推送方案，安卓生态广泛使用 | 2019 年 4 月已宣布停止服务，Chrome 版本 72+ 不再支持 Web GCM |
| 开发模型 | `chrome.gcm` API 简单、直接 | 仅 Chrome 支持，且需打包为扩展 / 打包应用，违背 Web 标准 |
| 协议 | 走 Google 专有通道，后端接入简单 | 基于 HTTP/1，效率低；不支持 VAPID 标准；不支持标准 Push API |
| 功能 | 可指定 TTL、优先级 | 最大载荷 4 KB；无分组/话题；无统计分析 |

## 2. Firebase WebPush / GCM / HTTP&#x2F;2 Server Push 对比

| 能力 | Firebase WebPush (FCM) | GCM (已废弃) | HTTP&#x2F;2 Server Push |
| ---- | ----------------------- | ------------ | ----------------------- |
| 标准化 | 基于 W3C Push API + VAPID | 专有 | IETF RFC 7540 (资源推送) |
| 传输协议 | HTTP&#x2F;2 (FCM endpoint) | HTTP&#x2F;1 | HTTP&#x2F;2 |
| 适用场景 | 后台消息 + 通知 | 后台消息 + 通知 | 同一连接内预推资源，页面生命周期内有效 |
| 浏览器支持 | Chrome, Edge, Firefox, Opera, Samsung Internet，Safari 需 APNS；iOS 16.4+ PWA | 仅旧版 Chrome | 所有支持 HTTP&#x2F;2 的浏览器，但非通知 |
| 身份认证 | VAPID / FCM Server Key | GCM Key | TLS 证书即可 |
| 最大载荷 | 4 KB（可附 data） | 4 KB | N&#x2F;A（推送资源大小） |
| 优先级 | normal &#x2F; high | normal &#x2F; high | 无 |
| 分组/topic | 支持 | 不支持 | N&#x2F;A |
| 统计分析 | Firebase Analytics/BigQuery | 无 | 无 |

## 3. Service Worker 对 WebPush 的支持

1. 仅在 HTTPS 环境注册 `navigator.serviceWorker.register()`。
2. 通过 `PushManager.subscribe()` 与浏览器推送服务（如 FCM、Mozilla Push Service、WebPush Proxy）建立订阅，生成 `endpoint` 与公钥。
3. Service Worker 监听事件：
   - `push` — 收到推送数据
   - `notificationclick` — 用户点击通知
   - `pushsubscriptionchange` — 订阅失效时自动重新订阅

## 4. 基于 Service Worker 实现 WebPush 可用能力

| 能力 | 说明 |
| ---- | ---- |
| 展示通知 | `self.registration.showNotification()`，支持标题、正文、图片、动作按钮、标签、badge、音效等 |
| 后台网络请求 | 在 `push` 事件里可使用 `fetch()` 从服务器拉取增量数据或确认投递；受 [30 s 事件超时] 制约 |
| 本地存储 | 可访问 Cache API、IndexedDB；读写小量数据（如离线消息） |
| 逻辑处理 | 可执行 JS 算法、过滤、去重、根据数据动态决定是否展示通知 |
| 受限点 | 无 DOM 访问；长时或大量 CPU 运算可能被杀；某些平台（iOS）禁止后台 fetch |

## 5. Service Worker 生命周期与后台限制

```
安装 (install) ➜ 激活 (activate) ➜ 空闲 ➜ 事件触发 (如 push) ➜ 终止 (terminate)
```

- **Chrome 桌面**：浏览器未运行时，FCM 进程不会常驻；系统唤醒 Chrome 后才会触发 `push` 事件 → 通知存在延迟。
- **Chrome Android**：依赖系统级 FCM 组件，即使 Chrome 未启动也能收到通知。
- **Firefox**：后台进程 `firefox --headless` 维持 WebPush 通道；完全退出后无法收到。
- **Safari**：iOS 16.4+ 仅 PWA 安装后可收，可被 iOS "通知摘要""专注模式"限制。

> 规范：事件脚本须在 **30 s** 内 `event.waitUntil()` 结束；否则浏览器可能终止。

**何时会被 Terminate？**
- 脚本空闲继续 30 s（Chrome 桌面/Android）、≈ 2–3 min（Firefox）后，浏览器为释放内存而回收。
- 前台无相关页面且设备进入低电量 / 内存压力场景。
- 浏览器升级、标签崩溃或调用 `registration.unregister()`。
- 用户在设置中清除站点数据 / 缓存。

**Terminate 后还能接收推送吗？**
- 可以。终止仅意味着 *运行时实例* 被销毁，Service Worker 注册信息与 Push 订阅仍保留。
- 当下一条推送到达时，浏览器会"冷启动" Service Worker，再次触发 `push` 事件，随后可继续展示通知或执行逻辑。
- 若站点已被用户撤销通知权限、取消订阅或注销 SW，则后续推送不会再投递。

## 6. WebPush 延迟与优先级

- FCM 提供 `priority`: `high` (即时) / `normal` (省电，Doze 时可能批量投递)。
- TTL (`time_to_live`) 决定缓存时长 (0–2419200 s)。
- 实测在 Wi-Fi 环境 `high` 消息 < 1 s；移动网 + Doze 可能延迟 15 min+。

### 推送频率限制

| 维度 | 限制来源 | 说明 |
| ---- | -------- | ---- |
| FCM 发送速率 | 后端 ➜ FCM | 官方未公布硬性 QPS，但文档建议单 token < **1k msg/s**，大量并发应分批并遵循 [Retry-After] 429 退避；Topic 推送存在后台配额，超限会被排队或 4xx。|
| 浏览器 UI 策略 | Chrome | 如站点 1 周内被用户 **连续关闭 3 次通知** 或频繁忽略，Chrome 会启用 *Quieter Notification UI* 并提示用户自动屏蔽；Manifest 内 `description`/**purpose string** 可降低触发。 |
| OS 级策略 | Android/iOS | Android 7+ Doze 会合并低优先级消息；同一 `collapse_key`/`tag` 的通知会被覆盖；iOS Focus/Notification Summary 会将高频消息分组延后。 |
| 用户体验 | 产品侧 | 行业普遍遵循 **≤ 3 推/天 / 用户**；重要事件即时推送，其余批量 or Digest；提供偏好中心让用户选择类别/频率。 |

**最佳实践**

1. 后端维护 *Frequency Control*：Redis 计数器或滑动窗口，按用户 &amp; 通知类型限流，如 `5/min`, `20/day`。
2. 在 Payload 中使用 `collapse_key`（FCM HTTP v1 可用 `android.collapse_key`、Web 可用 `notification.tag`）合并同类事件。
3. 对低优先级消息使用 `priority=normal` 并延长 `ttl`，让系统在省电模式下批量下发。
4. 监控 429/5xx 响应，对同一 token 采用指数退避避免被 FCM 封禁。
5. 观察 Chrome DevTools *Application → Push Messaging* 报告中的 Quota usage，避免触达上限（实验性）。

## 7. 到达率 / 点击率监控实现

1. **服务器侧**：在 Firebase 控制台开启 Cloud Messaging Analytics，可自动收集到达、打开事件，结合 Google Analytics 报表展示。
2. **前端侧**：在 `push` 事件里上报 "收到"; 在 `notificationclick` 里上报 "点击" 并 clients.openWindow。
3. **整合**：可将 messageId 置于 payload，在前端回传即可关联。

### 自建统计服务方案

若不依赖 Firebase Analytics / BigQuery，可通过以下方式自行采集与计算送达、展示、点击等指标：

1. **事件模型设计**  
   定义统一的事件类型与字段，例如：  
   • `delivered` —— Service Worker 收到 `push` 事件后立即上报。  
   • `displayed` —— `showNotification()` 成功后上报（可在 Promise 解析后）。  
   • `clicked` —— `notificationclick` 里上报。  
   • `closed` —— `notificationclose`（部分浏览器支持）上报。  
   建议字段：`messageId`、`endpoint`、`token`/`registrationId`、`timestamp`、`userAgent`、`extra`。

2. **Service Worker 代码埋点**  
   ```js
   // push 事件
   self.addEventListener('push', event => {
     const data = event.data?.json() || {};
     const { messageId } = data;
     event.waitUntil(
       fetch('/stats/track', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify({ type: 'delivered', messageId, ts: Date.now() })
       })
     );
     // ...显示通知...
   });

   self.addEventListener('notificationclick', event => {
     const { messageId } = event.notification.data || {};
     event.waitUntil(
       fetch('/stats/track', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify({ type: 'clicked', messageId, ts: Date.now() })
       })
     );
   });
   ```

3. **后端埋点收集**  
   搭建 `/stats/track` API（REST、GraphQL 或 gRPC）写入高性能时序/分析数据库：  
   - 轻量场景：MySQL/PostgreSQL + 表分区。  
   - 大规模：ClickHouse、Apache Druid、InfluxDB、Kafka ➜ Flink ➜ OLAP。  
   同时保留发送日志（`messageId`, `endpoint`, `sent_at`），即可在离线任务中计算送达率、点击率、平均延迟等。

4. **离线与实时分析**  
   - 离线：每日批量 SQL 聚合或 Spark/Flink 批处理。  
   - 实时：Flink / Kafka Streams 维表关联，分钟级监控仪表盘（Grafana、Superset、Metabase）。

5. **可靠性细节**  
   - 端上采用 `navigator.sendBeacon` 或 IndexedDB 队列，在离线时缓存数据，恢复网络后重传。  
   - 为避免阻塞 `push` 事件，可使用 `event.waitUntil()` 并吞掉 fetch 异常。  
   - HTTPS + CORS 配置，确保 SW 可向自建域名上报。

> 注意：Web Push 标准与 FCM 均 **不提供官方投递回执**（delivery receipt）。因此"送达"只能靠 Service Worker 主动回报，可能受设备离线、杀进程等因素影响，需要结合 TTL 与重试策略评估数据完整性。

## 8. 卸载与取消订阅

### 取消订阅

```js
const sub = await registration.pushManager.getSubscription();
if (sub) {
  await sub.unsubscribe(); // 向浏览器推送服务注销
  await fetch('/api/removeToken', { body: sub.endpoint, method: 'POST' });
}
```

### 清理 Service Worker

```js
for (const reg of await navigator.serviceWorker.getRegistrations()) {
  await reg.unregister();
}
```

用户亦可在浏览器 **设置 → 隐私与安全 → 网站设置 → 通知** 手动移除。

## 9. FCM Web Push 的消息类型

| 类型 | 字段 | 典型用途 |
| ---- | ---- | ---- |
| Notification Message | `notification` | 后端直接定义标题/正文/图标；浏览器自动展示 |
| Data Message | `data` | 仅传输键值对，完全由前端代码决定展示 |
| Mixed | `notification` + `data` | 同时由系统展示 + 传输自定义数据 |
| Webpush Options | `webpush` | 针对 Web 的 `headers`, `fcm_options`, `notification` 子字段 |

## 10. Firebase 控制台简介

- Cloud Messaging 面板：生成 Web API Key，查看注册 Token 数；可按 **Topic**、**Segment**、**测试设备** 发送。
- Campaign：A&#x2F;B 测试、计划发送、目标分析。
- 报表：送达率、打开率、转化漏斗；可导出到 BigQuery。

## 11. Service Worker 是否必须依赖 PWA？

否。推送功能只要求：

1. 站点 HTTPS。
2. 注册 Service Worker。
3. 获得用户通知权限。

是否被安装为 PWA (`beforeinstallprompt`) 不影响推送，虽然安装后能获得 iOS 通知、后台运行等额外好处。

## 12. Push API 规范之外的厂商自定义层

浏览器/平台厂商在标准 Push API 之上仍掌控了**推送服务（Push Service）**及**系统级 UX 行为**，主要差异体现在以下方面：

| 组件 | 可由厂商自定义 | 风险 / 限制 | 带来的好处 |
| ---- | -------------- | ---------- | -------- |
| Push Service Endpoint | 域名、认证方式（VAPID/Server Key）、HTTP v1/v2 API 细节 | 端点变更或停服（GCM 实例），不同浏览器需适配多套服务 | 全球 CDN、长连接优化、DDoS 保护 |
| 消息排队与 TTL 处理 | 排队深度、`ttl` 上限、过期策略 | 超限丢弃、队列延迟不可见 | 节省终端流量、电量；在离线时缓存消息 |


## 13. Push 推送在广告领域的应用

| 场景 | 具体做法 | 价值 | 风险与合规 |
| ---- | -------- | ---- | -------- |
| 重定向 / 再营销 | 识别近期开启但未转化的用户，推送限时优惠、购物车遗留提醒 | CTR 10–30%，转化率高于 EDM | 需征得通知权限；避免高频打扰导致退订 |
| 精准分群投放 | 结合浏览行为、购买力、兴趣标签，将 token 映射至 DMP 受众包 | 提高投放 ROI | GDPR/CCPA 要求可追溯同意；需做好数据脱敏 |
| 地理围栏广告 | 端上 Service Worker 在 `push` 时判断 `navigator.geolocation` 或携带定位，推送附近门店优惠 | 促进到店转化 | iOS/Safari 不支持后台定位；需请求额外权限 |
| A/B Creative 测试 | 同一活动随机发送不同素材，前端上报打开/点击回流 | 快速验证文案、图片效果 | 样本量需足够；注意同时控制频率 |
| Programmatic Buy | DSP 实时竞价系统针对 webpush 订阅 ID 发送竞价消息，Service Worker 拉取广告素材并显示 | 构建新渠道，低流量成本 | 生态尚不成熟；浏览器政策审查严格 |
| 品牌曝光 & 回忆曲线 | 计划性每日/每周推送品牌故事或内容 | 维系品牌认知 | 需要 Frequency Capping；易被视为骚扰 |

### 广告素材规范

- 通知标题 ≤ 50 字符，正文 ≤ 120 字符，避免误导或高频营销词。
- `icon` 建议使用品牌 Logo；`image` (Chrome v59+) 可传 2:1 Banner 展示更丰富视觉。
- 必须提供 *action button* 描述，例如"查看详情 / 立即购买"。
- 禁止使用自动下载、成人内容、虚假中奖等违规素材，违反可被浏览器封禁。

### 效果测量与归因

1. 在 payload 写入 `campaign_id`、`creative_id`，前端所有埋点回流该字段。
2. 打开落地页时在 URL 带上 UTM & deep link，如 `utm_source=webpush&utm_campaign=summer_sale`。
3. 后端结合 **发送 ➜ 展示 ➜ 点击 ➜ 转化** 四级漏斗计算 eCPM / eCPC / eCPA。

### 最佳实践

1. **Frequency Capping**：结合第 6 节策略，广告型通知建议 ≤1 次/日。 
2. **用户偏好中心**：允许用户订阅/退订不同广告类别，提升留存。 
3. **渐进授权**：先获取通知权限，再在设置页引导同意"个性化广告"。 
4. **合规透明**：隐私政策列明数据用途，提供"一键退订"。 

## 14. 监控层 | 数据采集点 / 工具 | 关键指标 | 说明

| 监控层 | 数据采集点 / 工具 | 关键指标 | 说明 |
| ---- | ---------------- | -------- | ---- |
| 服务器侧 | Firebase Cloud Messaging Analytics (或 BigQuery Export) | delivered_count、open_count、error_code | 零开发即得；支持与 GA/GTM 关联做漏斗分析 |
| 前端侧 | Service Worker 埋点 (`push`, `notificationclick`) | delivered、displayed、clicked | 自定义上报 URL，可实时监控到达/点击；可拓展关闭事件 |
| 数据整合 | messageId / campaignId 关联 | push ➜ open ➜ conversion | 通过数据库 JOIN 或 BigQuery SQL 将发送日志与行为事件拼接以计算到达率、点击率、转化率 |


参考：
- https://developer.chrome.com/docs/extensions/how-to/integrate/web-push?hl=zh-cn
- https://developer.chrome.com/docs/extensions/how-to/integrate/chrome.gcm?hl=zh-cn
- https://firebase.google.com/docs/cloud-messaging/fcm-architecture?hl=zh-cn
- https://firebase.google.com/docs/cloud-messaging/js/receive?hl=zh-cn#web_6
- https://developer.mozilla.org/zh-CN/docs/Web/API/Push_API
- https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API
- https://blog.mozilla.org/services/2016/08/23/sending-vapid-identified-webpush-notifications-via-mozillas-push-service/
- https://web.developers.google.cn/articles/push-notifications-overview?hl=zh-cn
- https://web.developers.google.cn/articles/push-notifications-notification-behaviour?hl=zh-cn#notification_click_event
- https://web.developers.google.cn/articles/push-notifications-display-a-notification?hl=zh-cn#browsers_and_feature_detection
- https://web.dev/articles/service-worker-lifecycle?hl=zh-cn
- https://w3c.github.io/ServiceWorker/#service-worker-lifetime