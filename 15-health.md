# 15 — 手环健康数据：让 AI 知道你睡得好不好

## 这是什么

把手环/手表的健康数据（心率、步数、睡眠、卡路里）接进自己的服务器，AI 聊天时能看到「她昨晚只睡了五个小时」，你自己也有一张可视化卡片。全程不经过任何厂商云端分析，数据落在自己的 SQLite 里。

```
手环 → Gadgetbridge（开源，本地蓝牙）→ Health Connect（安卓系统健康中枢）
     → Tasker（定时读 HC，HTTP POST）→ moon-memory /vitals → 三端工具 + 可视化卡片
```

设计上是**设备无关**的：换手环/手表只动第一跳（新设备配对 Gadgetbridge、确认写进 Health Connect），Tasker、服务器、AI 工具全都不用动。

## 后端：一张快照表就够

```sql
CREATE TABLE health_snapshots (
  id INTEGER PRIMARY KEY,
  bpm_avg INTEGER, bpm_max INTEGER,
  steps INTEGER,          -- 当日累计值（不是增量！）
  calories REAL,          -- 同上
  sleep_ms INTEGER,       -- 昨晚整段睡眠
  created_at TEXT DEFAULT (datetime('now'))  -- ⚠️ 这是 UTC
);
```

- `POST /vitals` 收 Tasker 的上报（Bearer token 保护），`GET /vitals?hours=24` 给前端和工具
- Tasker 每 15~30 分钟报一次快照，一天不到 100 行，一年也就 3 万行，不用分区不用清理

## 三端都能看

- **聊天 AI**：加一个 `check_health` 工具（言叽前端工具 + MCP 各一份），AI 能主动查「她今天走了多少步」；再把最新快照摘要注入 system context，AI 不用调工具也知道大概
- **CC 里的 AI**：直接 curl
- **人**：前端做一张「身体气象站」卡片——今日四指标 + 近七天睡眠/步数柱状图

## 前端聚合的两个细节

1. **累计值按天取 max，不是求和**。steps/calories 是当日累计，同一天的快照序列是递增的，把每天所有快照加起来会得到天文数字。按本地日分桶后取每天最大值才对。
2. **created_at 是 UTC**，解析时补时区：`new Date(r.created_at.replace(' ', 'T') + 'Z')`，再按用户时区分日。差 8 小时足够把「昨晚的睡眠」算到错误的一天去。

顺手的运维信号：卡片底部显示「最近上报时间」——它停在几小时前，就说明 Tasker 被系统杀了，比任何监控都直观。

## Tasker 的坑（都是亲历）

- **配置会丢**：Tasker 修改后要**退回主界面**才写盘。改完直接按 Home 键杀后台 = 白改。防丢三件套：改完退到 Tasker 桌面、导出数据备份一份、手机开机自启白名单加上 Tasker
- **分应用代理名单**：如果用户在墙内、服务器在墙外、手机是分应用代理，**Tasker 必须加进代理名单**，否则 POST 直连被墙，静默失败没有任何报错
- Health Connect 读权限要在 HC 的设置里逐项授权给 Tasker，漏一项对应指标就是 null

## HTTP Request 头部的坑

Tasker 的 HTTP Request 动作里，Headers 字段格式是每行一个 `Key:Value`，**冒号后不要加空格**，Body 里的 JSON 得自己拼。建议先用 `curl` 在电脑上把请求调通，再往 Tasker 里搬——在手机小屏幕上调试 400 错误是酷刑。

## 换设备清单（用户可自助）

1. 新手环用 Gadgetbridge 配对（旧的可以留着，不冲突）
2. Gadgetbridge 设置里确认「写入 Health Connect」勾着
3. 等一个上报周期，打开身体气象站看最新上报时间刷没刷新

就这三步。管线其余部分感知不到设备换了。
