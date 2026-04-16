# Antenna 架构笔记

读 core.js (606 行, 20.6KB) 之后的理解。2026-04-15。

## 整体结构

单 npm 包 (`antenna-fyi`)，三个接口共享同一个 core：
- **CLI** — 终端直接用
- **MCP Server** — agent 通过 MCP 协议调用
- **OpenClaw Plugin** — OpenClaw 框架的插件

数据层是 Supabase (PostGIS)。

## 两条主线

### 1. 日常社交发现
```
profile → checkin → scan → accept/pass → matches
```
24 小时过期，每天重新开始。

### 2. 活动模式
```
create → join → checkin → scan → end
```
活动有独立作用域，只扫到同一活动的参与者。

## 关键设计

### Ref 系统
scan 返回的结果用编号（1, 2, 3），不暴露 device_id。ref 持久化在 DB 里，跨 session 保持一致。这意味着 agent 可以说"accept 3"而不用处理内部 ID。

### Embedding 匹配
profile 保存时触发 Supabase Edge Function 生成 embedding。匹配基于语义相似度，不是关键词。

### Match Reason
discover 调用另一个 Edge Function 生成匹配理由——"你们可能聊得来，因为______"。这是 Antenna 区别于列表类产品的核心。

### GPS 模糊化
所有坐标四舍五入到 3 位小数（~100m 精度）。保护隐私但保留"附近"的判断。

### Bind 机制
agent 没有 GPS，所以有 bind 命令：生成 `antenna.fyi/locate?token=xxx` 链接 → 用户在浏览器打开 → 浏览器请求 GPS 权限 → 坐标发回服务器。解决了"agent 怎么知道你在哪"的问题。

### Global Discover 兜底
附近没人的时候，fallback 到全球推荐（每天 1 次，去重）。

## 版本迭代速度

从 0.5.0 (Apr 3) 到 1.2.9 (Apr 15)，38 个版本。4/15 一天发了 7 个版本。Event 系统在 1.0 (Apr 14) 加入。
