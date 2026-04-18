# Antenna 版本测试清单

每次 Han1 发新版本时跑一遍。目的：确认核心流程没被改坏，新功能能用，安全没退步。

---

## 0. 更新

```bash
npm update -g antenna-fyi
npx antenna-fyi help   # 确认版本，看有没有新命令
```

对比上一版的 help 输出，记录新增/改动的命令和参数。

---

## 1. Profile 基础（必测）

| 测试项 | 命令 | 预期 | 风险 |
|--------|------|------|------|
| 查看状态 | `status --id telegram:iamagenius00` | 显示 profile 信息 | RLS 变更可能导致读不到自己 |
| 更新 profile | `profile --id ... --line1 'test'` | 保存成功 | 参数名可能变 |
| 隐藏/显示 | `profile --id ... --hide` → `--visible` | visible 状态切换 | v1.2.11 新增，可能被改 |

**回归风险**：RLS 策略变更导致 profile 读写失败（v1.2.11 发生过）

---

## 2. Scan + Accept + Matches（核心流程）

| 测试项 | 命令 | 预期 | 风险 |
|--------|------|------|------|
| 扫描附近 | `scan --lat 31.168 --lng 121.399 --radius 1000 --id ...` | 返回附近的人 | Supabase RPC 函数重载（v1.2.11前的bug） |
| Accept（用 --ref） | `accept --id ... --ref 1` | 成功 | v1.2.11 才加的 --ref 支持 |
| Accept（用 --target） | `accept --id ... --target telegram:xxx` | 成功 | FK 约束（v1.2.11前的bug） |
| 查匹配 | `matches --id ...` | 显示匹配列表 | RLS 可能挡 |

**回归风险**：
- scan 返回空（RLS 或 RPC 问题）
- accept 报 undefined 或 FK error
- matches 读不到（RLS）

---

## 3. Discover（每天1次额度，谨慎测）

| 测试项 | 命令 | 预期 | 风险 |
|--------|------|------|------|
| 全球推荐 | `discover --id ...` | 推荐一个不是自己的人 | 自推（双 profile 问题） |
| 空结果不扣额度 | discover 返回空时再调一次 | 还能用 | v1.2.11 修的，可能退步 |

**回归风险**：
- 推荐自己（多 device_id 未去重）
- 空结果消耗额度

⚠️ 每天只有1次额度，测完就没了。先测别的，最后测这个。

---

## 4. Event 全流程

| 测试项 | 命令 | 预期 | 风险 |
|--------|------|------|------|
| 创建活动 | `event --create --name 'Test' --id ...` | 返回 event code | |
| 加入活动 | `event --join --code xxx --id ...` | 成功 | v1.2.11 RLS 误杀过 join |
| 需审批活动 join | `event --join --code xxx --id ... --application-context '...'` | 提交申请 | v1.2.24 新功能 |
| 审批 | `event --approve --code xxx --ref 1` | 通过 | v1.2.24 新功能 |
| 签到 | `event --checkin --code xxx --id ... --lat ... --lng ...` | 成功 | GPS 校验逻辑 |
| 扫描参与者 | `event --scan --code xxx --id ...` | 显示 joined/checked in 计数 | 计数不一致（v1.2.9 bug） |
| 结束活动 | `event --end --code xxx --id ...` | 仅主办方可结束 | |
| 结束后 join | `event --join --code xxx --id other` | 被拒绝 | |

**回归风险**：
- join 返回 undefined（RLS 问题）
- checkin 计数不更新
- scan 不显示自己（by design，但要知道）
- 非主办方能 end（权限泄露）

---

## 5. 安全检查（每次必做）

| 测试项 | 方法 | 预期 | 风险 |
|--------|------|------|------|
| profiles 表全读 | 用 anon key 直接查 Supabase `profiles.select('*')` | 返回空 | RLS 退步 = 全表泄露 |
| matches 表全读 | 同上查 matches 表 | 返回空 | 同上 |
| 跨用户查 profile | 用 anon key 按 device_id 查别人 | 返回空 | 精确查询也应该被挡 |
| npm 包里的 key | 检查 core.js 里的 DEFAULT_KEY | 应该只是 anon key | 如果是 service role key 就是严重问题 |

**回归风险**：
- 新版本里 RLS 策略被覆盖/重置
- 新表没加 RLS
- anon key 权限被意外扩大

---

## 6. 边界情况

| 测试项 | 方法 | 预期 |
|--------|------|------|
| 没 profile 就 join | 新 device_id 直接 join event | 应该提示先建 profile（不是静默失败） |
| 重复 join | 同一个 id join 两次同一个活动 | 幂等或明确提示 |
| accept 不存在的人 | `accept --target fake:nobody` | 明确报错 |
| scan 超大半径 | `scan --radius 999999` | 被限制（help 说 max 1000） |

---

## 7. 新功能专项

每次更新时，根据 changelog 列出新功能，逐个测试。格式：

```
### v1.2.XX 新功能
- [ ] 功能A：描述 → 测试命令 → 结果
- [ ] 功能B：描述 → 测试命令 → 结果
```

---

## 测试顺序建议

1. **更新 + help**（看变了什么）
2. **status**（确认基础连通）
3. **scan**（核心功能）
4. **accept + matches**（匹配流程）
5. **event 全流程**（最容易出问题的区域）
6. **安全检查**（每次必做）
7. **discover**（最后测，额度有限）
8. **新功能专项**

---

## 历史 bug 档案

| 版本 | Bug | 状态 |
|------|-----|------|
| < v1.2.11 | scan Supabase 函数重载（double vs integer） | ✅ 已修 |
| < v1.2.11 | accept FK constraint error | ✅ 已修 |
| < v1.2.11 | discover 空结果扣额度 | ✅ 已修 |
| < v1.2.11 | anon key 能读全表（profiles + matches） | ✅ 已修 |
| < v1.2.24 | event join 返回 undefined | ✅ 已修 |
| 持续 | discover 推自己（多 device_id） | ⚠️ 临时方案（hide），等 v1.6 user_id |
| v1.2.9 | checkin 计数不同活动表现不一致 | ❓ 未确认是否修复 |
| 持续 | 没 profile 就 join 静默成功但不可见 | ❓ 未确认是否修复 |
| v1.2.24 | event scan 泄露其他人的申请理由（screening answers） | 🆕 应只有主办方可见 |
