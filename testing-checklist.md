# Antenna 版本测试清单

每次 Han1 发新版本时跑一遍。目的：确认核心流程没被改坏，新功能能用，安全没退步。

**规则：全绿才发。有红的不发。**

参考：[Han1 的 TESTING.md](https://github.com/H1an1/Killluma/blob/main/TESTING.md)

---

## 0. 预检

```bash
npm update -g antenna-fyi
antenna help   # 确认版本号，对比新增/改动的命令
```

对比上一版的 help 输出，记录新增/改动的命令和参数。

---

## 1. Profile（必测）

| # | 测试 | 命令 | 预期 | 风险 |
|---|------|------|------|------|
| 1.1 | 查看 status | `status --id telegram:iamagenius00` | 显示 profile + watch 状态 | RLS 变更 |
| 1.2 | 更新 profile | `profile --id ... --name 空系 --line1 'test'` | 保存成功 | 参数名变化 |
| 1.3 | 隐藏/显示 | `profile --id ... --hide` → `--visible true` | visible 切换 | |

**回归风险**：RLS 策略变更导致 profile 读写失败（v1.2.11 发生过）

---

## 2. Scan + Match（核心流程）

| # | 测试 | 命令 | 预期 | 风险 |
|---|------|------|------|------|
| 2.1 | 扫描附近 | `scan --lat 31.168 --lng 121.399 --radius 1000 --id ...` | 返回附近的人 + 活动 | RPC 签名 |
| 2.2 | Accept（ref） | `accept --id ... --ref 1` | 成功 | ref 解析 |
| 2.3 | Accept（target） | `accept --id ... --target telegram:xxx` | 成功 | FK 约束 |
| 2.4 | Pass | `pass --id ... --ref 1` | 成功 | |
| 2.5 | Check matches | `matches --id ...` | 显示匹配列表 | RLS |
| 2.6 | Bind（profile） | `bind --id ...` | 返回 URL | token 类型 |
| 2.7 | Bind（event） | `bind --id ... --purpose event --event-code abc` | 返回 URL | |

**回归风险**：
- scan 返回空（RLS 或 RPC 问题）
- accept 报 undefined 或 FK error
- matches 读不到（RLS）

---

## 3. Event 全流程

| # | 测试 | 命令 | 预期 | 风险 |
|---|------|------|------|------|
| 3.1 | 创建（基础） | `event --create --name 'Test' --id ...` | 返回 code + URL | |
| 3.2 | 创建（审批） | `event --create --name 'Test' --id ... --requires-approval --screening-questions 'Q1\|Q2'` | requires_approval=true | 参数传递 |
| 3.3 | Join（无 profile） | 新 device_id join | 拒绝 "Create profile first" | profile gate |
| 3.4 | Join（需审批，无 context） | `event --join --code xxx --id ...` | 返回 screening questions | screening gate |
| 3.5 | Join（需审批，有 context） | `event --join --code xxx --id ... --application-context 'answer'` | status=pending | |
| 3.6 | Join（重复） | 同一 id 再 join | 返回当前状态 "already in" | 幂等 |
| 3.7 | Scan（主办方） | `event --scan --code xxx --id creator` | 看到 pending + application_context | p_device_id 传递 |
| 3.8 | Scan（参与者） | `event --scan --code xxx --id participant` | 只看到 active，无 application_context | 权限隔离 |
| 3.9 | Approve | `event --approve --code xxx --id creator --ref 1` | 通过 | |
| 3.10 | Reject | `event --reject --code xxx --id creator --ref 1` | 拒绝 | |
| 3.11 | Update | `event --update --code xxx --id creator --name 'New Name'` | 更新成功 | --update 不被 create 吃掉 |
| 3.12 | Add host | `event --add-host --code xxx --id creator --ref 1` | 添加 co-host | |
| 3.13 | Checkin | `event --checkin --code xxx --id ... --lat ... --lng ...` | ≤1km 通过 | GPS 校验 |
| 3.14 | End | `event --end --code xxx --id creator` | 仅 creator 可结束 | |
| 3.15 | End 后 join | `event --join --code xxx --id other` | 拒绝 "Event has ended" | |

**回归风险**：
- join 返回 undefined（RLS 问题）
- checkin 计数不更新
- scan 不显示自己（by design）
- 非主办方能 end（权限泄露）
- application_context 泄露给非 host

---

## 4. Discover（最后测，每天1次额度）

| # | 测试 | 命令 | 预期 | 风险 |
|---|------|------|------|------|
| 4.1 | 全球推荐 | `discover --id ...` | 推荐一个不是自己的人 | 自推（多 device_id） |
| 4.2 | 空结果不扣额度 | 推荐为空时再调 | 还能用 | |

⚠️ 每天1次额度，先测别的，最后测这个。

---

## 5. 安全检查（每次必做）

| # | 测试 | 方法 | 预期 | 风险 |
|---|------|------|------|------|
| 5.1 | profiles 全读 | anon key `profiles.select('*')` | 返回空 | RLS 退步 = 全表泄露 |
| 5.2 | matches 全读 | 同上 | 返回空 | |
| 5.3 | event_participants 全读 | 同上 | 返回空 | Realtime SELECT policy 不影响直接查询 |
| 5.4 | scan 不返回 device_id | scan 结果只有 ref | 无 device_id 泄露 | |
| 5.5 | application_context 隔离 | 非 host scan | 看不到其他人的申请答案 | |
| 5.6 | npm 包里的 key | 检查 core.js DEFAULT_KEY | 只有 anon key | service role key = 严重 |

**回归风险**：
- 新版本里 RLS 策略被覆盖/重置
- 新表没加 RLS
- anon key 权限被意外扩大

---

## 6. 通知检查

| # | 测试 | 方法 | 预期 |
|---|------|------|------|
| 6.1 | OC Plugin Realtime | matches INSERT → 推送到用户 | 收到通知 |
| 6.2 | OC Plugin event notify | event_participants UPDATE → 推送 | 收到通知 |
| 6.3 | antenna watch | `antenna watch --id xxx` 前台运行 | 显示 SUBSCRIBED + 接收事件 |
| 6.4 | watch PID | `antenna status --id xxx` | 显示 Watch: ✅/❌ |
| 6.5 | watch log | `cat ~/.antenna/watch.log` | 有输出 |
| 6.6 | DB trigger notify | approve 后 Edge Function 是否触发 | 检查 Supabase function logs |

---

## 7. 边界情况

| # | 测试 | 方法 | 预期 |
|---|------|------|------|
| 7.1 | 没 profile 就 join | 新 device_id 直接 join event | 提示先建 profile（不是静默失败） |
| 7.2 | 重复 join | 同一个 id join 两次同一个活动 | 幂等，不重新触发 screening |
| 7.3 | accept 不存在的人 | `accept --target fake:nobody` | 明确报错 |
| 7.4 | scan 超大半径 | `scan --radius 999999` | 被限制 |

---

## 8. 新功能专项

每次更新时，根据 changelog 列出新功能，逐个测试。格式：

```
### v1.2.XX 新功能
- [ ] 功能A：描述 → 测试命令 → 结果
- [ ] 功能B：描述 → 测试命令 → 结果
```

---

## 测试顺序

1. **预检**（版本 + help）
2. **status**（基础连通）
3. **scan + match**（核心流程）
4. **event 全流程**（最容易出问题）
5. **安全检查**（每次必做）
6. **通知检查**
7. **边界情况**
8. **discover**（最后测，额度有限）
9. **新功能专项**

---

## 历史 bug 档案

| 版本 | Bug | 状态 |
|------|-----|------|
| < v1.2.11 | scan Supabase 函数重载（double vs integer） | ✅ 已修 |
| < v1.2.11 | accept FK constraint error | ✅ 已修 |
| < v1.2.11 | discover 空结果扣额度 | ✅ 已修 |
| < v1.2.11 | anon key 能读全表（profiles + matches） | ✅ 已修 |
| < v1.2.13 | Hermes create 用 p_device_id 不是 p_created_by | ✅ 已修 |
| < v1.2.13 | Hermes join 缺 lat/lng + application_context | ✅ 已修 |
| < v1.2.15 | SKILL.md 说 "plugin tools" 导致 CLI 用户 agent 放弃 | ✅ 已修 |
| < v1.2.16 | OC Plugin notifyUser 用旧命令格式 | ✅ 已修 |
| < v1.2.17 | join_event screening gate 不返回 questions | ✅ 已修 |
| < v1.2.20 | CLI --update 被 create 分支吃掉 | ✅ 已修 |
| < v1.2.20 | CLI approve/reject/update/add-host handler 缺失 | ✅ 已修 |
| < v1.2.22 | OC Plugin parse error（重复 notifyUser 代码块） | ✅ 已修 |
| < v1.2.24 | event join 返回 undefined | ✅ 已修 |
| < v1.2.25 | Hermes event_scan 缺 p_device_id + 完整字段 | ✅ 已修 |
| < v1.2.25 | application_context 对所有人可见（应只对 host） | ✅ 已修 |
| < v1.2.25 | screening_questions 全角竖线拼成一个字符串 | ✅ 已修 |
| < v1.2.27 | join_event 重复 join 重新触发 screening | ✅ 已修 |
| < v1.2.27 | matches 表不在 Realtime publication | ✅ 已修 |
| < v1.2.27 | notify_webhook trigger 用错 net.http_post 签名 | ✅ 已修 |
| 持续 | discover 推自己（多 device_id） | ⚠️ 临时方案（hide），等 v1.6 user_id |
| v1.2.9 | checkin 计数不同活动表现不一致 | ❓ 未确认 |
