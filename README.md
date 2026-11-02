# WorkDaddy 自动化任务

仓库标记：`WorkDaddyAutomationRepository`

把任务 JSON 放进 `tasks/` 目录，并在仓库简介（Description）中保留上面的仓库标记，WorkDaddy 即可自动收录本仓库的公开任务。

## 任务列表

- [逐账号例行发送（Hy3，跳过活跃地图今日已活跃的账号）](tasks/keep-accounts-active-hy3-skip-active-today.json)
- [每日派猫猫旅行](tasks/daily-buddy-cat-travel.json)

## 任务说明

**逐账号例行发送（Hy3，跳过活跃地图今日已活跃的账号）**

判定依据是**活跃地图（成长热力墙）今日是否已点亮**，即 `account.status` 的 `activity.today` → `is_active`。

⚠️ 注意区分两个容易混淆的信号：

- `activity.today` → `is_active`：**活跃地图**今日是否已点亮，**完成一次对话才会被记录** ← 本任务用它
- `checkin.today`：WorkDaddy 的**签到**，与活跃地图无关

正因为"完成一次对话才记录"，本任务的执行动作（发消息 + 等回复完成）本身就是点亮今日热力墙的手段。

两阶段执行：先筛选，再发送。

1. **筛选（不切换账号）**：遍历全部账号，逐个查询 `activity.today`。
   - `is_active: true` → 今日已点亮，剔除，不参与后续发送；
   - `is_active: false` → 列入发送名单；
   - 查询失败 → 按未活跃处理，宁多发不漏发。
2. **发送（真实切换账号）**：名单为空时无需任何切换，直接结束；名单非空时只对名单内的账号依次切换，使用 `hy3` 新建会话发送 `1+1=`，按会话回执等待回复完成；循环结束时自动恢复开始时的账号。

行为要点：

- **需要自动化协议 V3**（`schemaVersion: 3`）。筛选用 `account.forEach` 的 `prepare` + `condition`，判断发生在**切换账号之前**——被剔除的账号根本不会被切换，省掉无谓的页面刷新与会话同步（这正是上一版"全完成时仍空跑逐个切号"的根源）。
- `activity.today` 会**请求网络**（用账号备份里的 accessToken 查成长中心接口），因此放在带 `logic.catch` 的筛选阶段：单个账号查询失败不会影响其余账号。
- 用账号级 `state` 在阶段之间传递筛选结论（按任务 ID + 账号 UID 隔离，24 小时自动过期）。
- **发送阶段按账号隔离错误**：单个账号的发送失败或等待超时只写运行日志并跳过，不会中断整轮、也不会拖住其余账号
  （`session.wait` 的 `timeoutMs` 是上限，正常回复一到就立即返回）。
- 触发方式为手动，不会自动运行；并发策略 `skip`，运行中不重入。

## 关于执行速度（重要）

本任务能省的只有"少切号"和"别被单个账号拖住"，**切号本身的固定开销不在任务控制范围内**，主要是两件事：

1. **会话自动复制**：只要 `meta.json → autoCopy` 里登记了会话（哪怕是逐条登记的），每次切换账号都会触发一次
   会话复制作业，且切号会**阻塞等它完成**。实测每次切号约 3.5–11 秒，其中绝大部分就是它。
2. **其他生命周期任务**：`pageReady` / `accountSwitched` 触发的任务（例如关闭首页弹窗类）会在每次切号后被拉起。

自动化协议里**没有**"运行时关闭会话同步"的开关——`accountSwitch` 无条件读取自动复制规则，能力清单里也没有相关 op。
想要更快，请到 WorkDaddy 的应用设置里处理上面两点（例如停用自动复制规则、临时停用切号触发的弹窗任务）。

## 每日派猫猫旅行

对应文件：`tasks/daily-buddy-cat-travel.json`

每天先**问服务端**现在是什么状态，再决定做什么，不靠本地记忆猜：

| 服务端 `data.state` | 任务动作 |
|---|---|
| `arrived`（猫猫已回来，待领取） | `POST .../travel/claim` 领取旅行积分 |
| `idle` 等非 `traveling` 状态 | `POST .../travel/depart` 派出猫猫 |
| `traveling`（正在路上） | 跳过，不重复派出 |

另外 `data.daily_limit_reached` 为真时直接不再派出——当日上限以服务端为准，客户端不做假设。

涉及的接口（`Bearer` 由 `http.requestAsAccount` 注入，`X-User-Id` 用 `{{account.uid}}` 显式补上）：

- `GET  https://www.workbuddy.cn/activity/growth/buddy/travel/status`
- `POST https://www.workbuddy.cn/activity/growth/buddy/travel/claim`（body `{}`）
- `POST https://www.workbuddy.cn/activity/growth/buddy/travel/depart`（body `{"location_id": ...}`）

行为要点：

- **先领后派**：处于 `arrived` 时先领取；领取完如果当日额度还在，同一轮会继续尝试派出下一趟，不用等下一次调度。
- **领取与派出分开去重**（第一版踩过的坑）：领取只看服务端是否回报 `arrived`，**不受"今天已经派过"限制**；只有派出用账号级 `state`（`departDoneOn` = 今日日期）限流，每账号每天一次。
  - 若把两者共用一个"今日已完成"标记，09:00 派出后猫猫 4 小时才回来，之后所有触发都会被判成"今天处理过了"直接跳过，**回来的奖励当天永远没人领**。
  - 去重标记只在派出**成功后**写入；派出失败不写，下一轮还能重试。
- **领取与派出各自包在 `logic.catch` 里**：领取失败不影响派出；派出失败只提示"稍后自动重试"，不会把整轮判成失败。
- **登录态失效不误报成功**：状态查询 `ok` 为假时只写日志并提示"请检查登录态"，不会进入派出分支。
- **多账号**：`account.forEach` 覆盖全部账号，每账号独立判定、独立去重；全程纯 API 调用，不需要切换账号。
- **目的地收益相同**：`location_id` 只影响路线与文案，不影响积分，默认 `1`，可在 `variables.locationId` 里换成 `1`–`4` 中任一目的地。

触发方式：`interval` 每 **120** 分钟巡检一次（行程约 4 小时，2 小时粒度能保证回来后 2 小时内被领取），叠加 `pageReady` / `panelOpened` 事件兜底；并发策略 `skip`，运行中不重入。

## 安装

WorkDaddy → 自动化 → 公开任务 → 找到本任务 → 导入。导入后默认停用，请先确认行为再启用。

## 免责声明

任务会切换本机已登录的账号并新建会话发送消息，或按账号调用 WorkBuddy 活动接口；请自行确认其行为符合你的预期。
