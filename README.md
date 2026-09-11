<p align="center">
  <img src="https://raw.gi
thubusercontent.com/DGZSbot/ai-icon/refs/head
s/main/WorkBuddy.png" alt="WorkBuddy2API" wid
th="120">
</p>

<h1 align="center">WorkBuddy2
API</h1>

<p align="center">
  <b>把腾讯 C
odeBuddy 账号变成 OpenAI 兼容 API 的�
�账号网关</b><br>
  OAuth 登录 · 账�
�池轮转 · 熔断与冷却 · 会话粘性
 · 定时签到保活 · 流式/非流式
</
p>

<p align="center">
  <img alt="Go" src="h
ttps://img.shields.io/badge/Go-1.22.5-00ADD8?
logo=go&logoColor=white&style=flat-square">
 
 <img alt="API" src="https://img.shields.io/b
adge/API-OpenAI_Compatible-412991?style=flat-
square">
  <img alt="Deploy" src="https://img
.shields.io/badge/Deploy-Docker_Compose-2496E
D?logo=docker&logoColor=white&style=flat-squa
re">
  <img alt="Transport" src="https://img.
shields.io/badge/Transport-SSE%20%2F%20Stream
ing-0DBD8B?style=flat-square">
</p>

---

## 
📖 项目简介

WorkBuddy2API 是一个自
托管的 **OpenAI 兼容反向代理网关**
，将腾讯 CodeBuddy（`copilot.tencent.com
`）账号包装为统一的 `/v1/chat/comple
tions` 服务。

- 官方不提供 OpenAI �
�态的开放 API，本项目通过 **OAuth �
��备授权** 获取账号凭证，在网关�
��做 token 自动刷新、账号池调度与
流量治理；
- 面向 **个人多账号** 
场景：多账号共享、单号故障自动
换号、冷却/熔断防止雪崩、会话�
�性保证多轮上下文不跳号；
- 对�
�户端只暴露 OpenAI 兼容接口，现有
 SDK / 前端 / 工具 **零改造接入**。


> ⚠️ 合规须知：本项目是**非�
�方**网关，使用 CodeBuddy 账号作为�
��游，**仅限本人授权账号、本机/�
��有环境测试**。详细边界见 [安全
与合规](#-安全与合规)。

## ✨ 核�
��能力

| 能力 | 说明 |
|---|---|
| �
� **OAuth 一键登录** | `login.sh` 设备�
��权流程（无 PKCE），自动落盘凭�
�并重启容器 |
| 🔄 **多账号池** | 
三因子加权随机选号（积分比例 ×
10 + 闲置补偿 + 成功率 ×3），Top-5 
候选 + 防惊群 |
| 🛡️ **熔断与冷
却** | 429/限流文案软冷却 600s 起指
数退避（封顶 `soft_rate_max`）、404 �
��定 60s 短冷却、402/余额不足硬冷�
��至次日 04:00、连续失败指数退避�
��断、在途租约限流 |
| 🧲 **会话�
��性** | 同一会话（`conversation_id`）
尽量绑定同一账号，TTL 滚动续期�
�失败自动解绑 |
| ⏰ **定时任务** 
| 每日 09:00 / 21:00 自动签到 + 余额�
��询解冻 + 猫猫旅行（派猫/领奖）
；22:00 全账号 token 刷新保活 |
| ⚡
 **流式 + 非流式** | 上游 SSE 逐帧�
�范化透传；出站强制 `stream:true`，
非流式由本地聚合为单响应 |
| 🧠
 **推理模型兼容** | `reasoning_content`
 白名单保留、工具调用（`tool_calls
`）按 index 合并、effort 自动降级 |

| 📊 **可观测** | 每请求一行表格�
��志（TTFB/token 速率/uid）；`/healthz`
 带 `service` 身份标识可接负载均衡
/宿主探活 |
| 💾 **状态持久化** | 
池状态本地原子落盘 + Upstash Redis �
��步镜像（可选），重启择新恢复 
|
| 🗑️ **指纹脱敏** | 出站请求�
�黑名单指纹字段清洗（可关闭） |


## 🗺️ 架构总览

```mermaid
flowcha
rt LR
    Client["客户端 / SDK\nOpenAI 兼
容请求"] --> H

    subgraph GWI["WorkBudd
y2API 网关 :7863"]
        H["HTTP Handler\
n鉴权 · 日志 · 换号轮转"] --> P
   
     H --> S
        P["账号池\n三因子�
��权 · 熔断 · 冷却 · 租约"] --> U
 
       S["会话粘性路由"] -.绑定镜像
.-> REDIS
        T["定时调度\n签到 09/
21 · 保活 22 · 旅行 30m"] --> P
       
 U["上游 Client\nChatHTTP 流式 · 短 RPC
"]
    end

    P -. "读凭证 (0600)" .-> A
UTH[("auths/*.json")]
    P -. "状态镜像"
 .-> REDIS[("Upstash Redis\n可选")]
    U -
->|"v2/chat/completions (SSE)"| CB["CodeBuddy
\ncopilot.tencent.com"]
    U -->|"billing / 
auth / models"| CB
```

## 🚀 快速开始


### 环境要求

- **Docker + Docker Compos
e**（推荐部署方式，镜像内已含 `a
pp` 低权限用户）
- 一个（或多个�
�已注册的 CodeBuddy 账号，用于 OAuth
 登录
- 宿主机 Go ≥ 1.22（仅本地�
�接编译时需要）

### 1. 克隆并配�
�

```bash
git clone https://github.com/Slive
rkiss/workbuddy2api.git
cd workbuddy2api
cp c
onfig.example.json config.json
```

编辑 `c
onfig.json`，**至少设置 `api_key`**（`�
��空 = 不鉴权`，公网部署务必设置
）：

```bash
# 用编辑器把 "api_key" �
��成你自己的强随机串
```

### 2. 登
录添加账号

```bash
./login.sh
# 1) 脚�
��输出授权 URL
# 2) 浏览器打开完成
登录
# 3) 回到终端按 y → 自动签�
� → 落盘 auths/workbuddy-<uid>.json → �
��启容器
```

多账号只需重复执行�
��账号池自动发现 `auths/` 下新增凭
证文件（容器启动时 `SyncToDir` 对�
�）。

### 3. 启动服务

```bash
docker 
compose up -d --build
```

### 4. 验证

```
bash
# 健康检查（无可用账号时 503�
��；service 字段用于确认打到的是�
�网关
curl -s http://localhost:7863/healthz

# {"healthy":2,"total":3,"service":"workbudd
y2api"}

# 模型列表
curl -s http://localh
ost:7863/v1/models \
  -H "Authorization: Bea
rer your-api-key"

# 账号状态（汇总 + 
每账号详情）
curl -s http://localhost:7
863/status \
  -H "Authorization: Bearer your
-api-key"

# 流式聊天
curl -sN http://loc
alhost:7863/v1/chat/completions \
  -H "Autho
rization: Bearer your-api-key" \
  -H "Conten
t-Type: application/json" \
  -d '{"model":"d
eepseek-v4-flash","messages":[{"role":"user",
"content":"hi"}],"stream":true}'

# 非流式
聊天（本地聚合）
curl -s http://local
host:7863/v1/chat/completions \
  -H "Authori
zation: Bearer your-api-key" \
  -H "Content-
Type: application/json" \
  -d '{"model":"dee
pseek-v4-flash","messages":[{"role":"user","c
ontent":"hi"}],"stream":false}'
```

## ⚙�
� 配置说明

完整字段以 [`config.exam
ple.json`](config.example.json) 为样例（�
��表为各字段含义）。

```json
{
  "l
isten": ":7863",
  "api_key": "your-api-key-h
ere",
  "auth_dir": "./auths",
  "state_file"
: "./data/state.json",
  "cooldown": { "soft_
rate": "600s", "soft_rate_max": "2h" },
  "sc
hedule": {
    "checkin_hours": [9, 21],
    
"keepalive_hours": [22],
    "checkin_enabled
": true,
    "keepalive_enabled": true
  },
 
 "upstream": {
    "timeout_seconds": 120,
  
  "header_timeout_seconds": 120,
    "idle_ti
meout_seconds": 300
  },
  "features": { "san
itize_blacklist_fingerprints": true },
  "ups
tash": { "url": "", "token": "" },
  "pool": 
{
    "max_in_flight": 3,
    "breaker_thresh
old": 3,
    "breaker_cooldown": "30m",
    "
breaker_cooldown_max": "6h",
    "idle_weight
_per_hour": 0.5,
    "idle_weight_max": 5.0
 
 },
  "session_sticky": { "enabled": true, "t
tl": "30m", "gc_interval": "5m" }
}
```

### 
字段速查

| 字段 | 默认 | 说明 |
|-
--|---|---|
| `listen` | `:7863` | HTTP 监�
�地址 |
| `api_key` | 空 | 网关鉴权密
钥；**空 = 不鉴权直接放行**（公�
�必须设置） |
| `auth_dir` | `./auths` |
 账号凭证目录 |
| `state_file` | `./dat
a/state.json` | 账号池状态持久化文�
� |
| `cooldown.soft_rate` | `600s` | 软限�
��（429/限流文案）冷却**基数**；�
�一账号连续触发按 2 倍指数退避 |

| `cooldown.soft_rate_max` | `2h` | 软冷�
�指数退避的封顶时长 |
| `schedule.ch
eckin_hours` | `[9, 21]` | 每日本地时区
整点签到 + 余额查询；收尾顺带跑
一趟猫猫旅行。**空数组/`null` = 未
配置回落默认**（不是禁用） |
| `s
chedule.keepalive_hours` | `[22]` | 每日本
地时区整点刷新 token 保活。空数�
�/`null` 同上 |
| `schedule.checkin_enabled
` | `true` | 签到**总开关**；`false` �
�正关掉签到（**猫猫旅行随之停摆
**，见下） |
| `schedule.keepalive_enable
d` | `true` | token 保活总开关；`false`
 关掉保活 |
| `upstream.timeout_seconds` 
| `120` | 短 RPC（刷新/签到/余额/模�
��）总时长上限 |
| `upstream.header_tim
eout_seconds` | 回落 `timeout_seconds` | �
�天首字节前（响应头）上限 |
| `up
stream.idle_timeout_seconds` | `300` | 聊天
流中空闲上限（活跃续命，静默断
流） |
| `features.sanitize_blacklist_finge
rprints` | `true` | 出站请求体黑名单�
��纹脱敏 |
| `upstash.url` / `token` | 空
 | 空 = 纯内存模式（Noop 降级，功�
��照常） |
| `pool.max_in_flight` | `3` | 
单账号最大在途请求数（`0` = 不限
） |
| `pool.breaker_threshold` | `3` | 连�
��失败触发熔断阈值 |
| `pool.breaker_
cooldown` | `30m` | 熔断基础退避时长 
|
| `pool.breaker_cooldown_max` | `6h` | 指�
��退避封顶 |
| `pool.idle_weight_per_hour
` | `0.5` | 闲置补偿：每小时未使用
 +0.5 权重 |
| `pool.idle_weight_max` | `5.
0` | 闲置补偿权重封顶 |
| `session_st
icky.enabled` | `true` | 会话粘性路由�
�关 |
| `session_sticky.ttl` | `30m` | 会�
�绑定 TTL（滚动续期） |
| `session_st
icky.gc_interval` | `5m` | 过期绑定 GC �
�期 |

### 上游超时语义（三段各归
其位）

| 字段 | 作用对象 | 默认 |
 行为 |
|---|---|---|---|
| `timeout_second
s` | 短 RPC（token 刷新 / 签到 / 余额
 / 模型列表） | `120` | 总时长硬上�
��，到期报错走换号/熔断 |
| `header
_timeout_seconds` | 聊天 SSE **首字节前
** | `120` | 由 `Transport.ResponseHeaderTim
eout` 约束；超时 = 换号重发 |
| `idl
e_timeout_seconds` | 聊天 SSE **流中空�
�** | `300` | 活跃吐数据**续命**不掐
；静默超时才断流释放租约 |

聊�
�流（`stream` true/false 均同）**没有�
��时长上限**：聊天使用 `Timeout=0` �
��专用 client，长思考/长输出（如�
�长 reasoning）不会被 120s 掐断。

##
# 环境变量覆盖

加载顺序：JSON 文
件 → `WB2A_*` 环境变量（变量非空�
��覆盖）：

`WB2A_LISTEN` · `WB2A_API_KE
Y` · `WB2A_AUTH_DIR` · `WB2A_STATE_FILE` ·
 `WB2A_SOFT_RATE`（duration） · `WB2A_SOFT
_RATE_MAX`（duration） · `WB2A_TIMEOUT_SEC
ONDS` · `WB2A_HEADER_TIMEOUT_SECONDS` · `WB
2A_IDLE_TIMEOUT_SECONDS` · `WB2A_SANITIZE_FI
NGERPRINTS`（bool）

## 🧠 账号池与�
�量治理

### 账号状态机

每个账号
由三个正交维度描述：

| 维度 | �
�段 | 说明 |
|---|---|---|
| 健康 | `dis
abled` / `until` / `breakerUntil` | `healthy 
= !disabled && !until && !breakerUntil` |
| �
��发 | `inFlight` | 在途租约（运行态
，不持久化），上限 `max_in_flight` |

| 统计 | `successCount` / `errTotal` / `la
stUsed` | 供成功率权重与闲置补偿 |


```text
  Healthy ──429/404 软冷却 /
 402 硬冷却 / 5xx 熔断──▶ 冷却·
熔断期
     ▲                           
                     │
     │       到�
�自动恢复 / 签到余额解冻 / 成功�
�零     │
     └────────
───────────────
───────────────
──────────┘

  Disabled
（session 死亡，永久，需人工重新 
login.sh）
```

### 错误分类与处置

|
 分类 | 触发条件 | 账号处置 | 恢�
� |
|---|---|---|---|
| 余额不足 | HTTP 4
02 / body 含余额关键词 | 硬冷却到**
次日 04:00**（本地时区） | 签到（0
9/21 点）余额恢复自动解冻 |
| 频�
� | HTTP 429 / 限流文案（不限状态码
） | 软冷却 `soft_rate`（600s 起，连�
��触发指数退避，封顶 `soft_rate_max`
） | 到期自动恢复 / 成功清零退避
 |
| Session 失效 | body 含 `Offline user 
session not found` / `12153` | **永久禁用
** | 人工重新登录 |
| 上游 404 | HTTP
 404 | 软冷却固定 60s（不随 `soft_rat
e`、不单独退避） | 到期自动恢复 
|
| 服务端错误 | HTTP ≥500 | 喂连续
失败计数，达阈值熔断 | 熔断到期
 / 成功清零 |
| 客户端错误 | 其余 
4xx / 业务 `code≠0` | 不处罚，换号�
��试 | 即时 |

**熔断器**：所有冷�
�入口（429/404/402）与 5xx 共用唯一�
��续失败计数器 `fails`；累计达 `bre
aker_threshold`（默认 3）触发熔断，�
��避 `breaker_cooldown × 2^retryCount`，�
�顶 `6h`；成功清零。

**软冷却指�
�退避**（与熔断器并存的第二条升
级线）：软限流的**冷却时长**本�
�也按连续次数退避——同一账号�
�续触发软冷却时 `soft_rate × 2^(连�
�次数-1)`，封顶 `soft_rate_max`。计数
 `soft_streak` 独立于熔断器的 `fails`�
��只在**成功**或**签到解冻**时清�
�，随 `state.json` 持久化。两者别混
淆：软退避管"近期被限流"（600s→
1200s→2400s…，分钟~小时级），熔�
��管"病态反复失败"（30m→1h→2h→6
h）。

### 选号策略

1. 过滤：禁用
 / 冷却 / 熔断 / 在途占满账号不参
与
2. 取 **Top-5** 候选（按三因子权
重降序，积分只是因子之一）
3. �
�因子加权随机：
   `weight = credits �
��例 ×10 + idleWeight + successRate ×3`
  
 - `credits 比例` = 该号积分 / 候选�
�最大积分
   - `idleWeight` = `min(闲置
小时 × idle_weight_per_hour, idle_weight_m
ax)`，从未使用给满分
   - `successRat
e` = `successCount/(successCount+errTotal)`�
�无记录给中性 1.5
4. 防惊群：跳过
 100ms 内刚被选中的账号；全冷却�
�从非禁用、非余额耗尽的软冷却/�
��断账号中选最早到期者顶班

### �
��话粘性

同一会话尽量复用同一�
�号，多轮对话不跳号：

- 会话键�
��取顺序：`metadata.conversation_id` → 
`metadata.user_id` → 顶层 `conversation_i
d`
- TTL 滚动续期（默认 30m），GC �
�期 5m；绑定可镜像到 Redis（7 天）
防重启丢失
- 请求失败自动解绑；
成功后绑定跟随最终成功账号

### 
定时任务

| 任务 | 开关 | 时刻（�
�地时区） | 行为 |
|---|---|---|---|
| 
签到 | `schedule.checkin_enabled` 默认 `t
rue` | `checkin_hours` 默认 `[9, 21]` 整�
� | 签到 + 余额查询；余额恢复则�
�冻冷却账号；**收尾顺带跑一趟猫
猫旅行** |
| 保活 | `schedule.keepalive_
enabled` 默认 `true` | `keepalive_hours` �
�认 `[22]` 整点 | 全账号刷新 token；
session 失效自动禁用 |

容器时区由
 `TZ` 控制（compose 默认 `Asia/Shanghai`
）。

#### 关闭定时任务

用 `schedul
e.checkin_enabled` / `schedule.keepalive_enab
led` 显式关闭，两者互相独立：

``
`json
"schedule": {
  "checkin_hours": [9, 21
],
  "keepalive_hours": [22],
  "checkin_enab
led": false,
  "keepalive_enabled": true
}
``
`

上例：**只关签到，保活照常 22:
00 跑**。两个都设 `false` 则调度器�
��任何时点可等，
`Run` 不空转、直
接阻塞等待退出信号（不会忙等空
烧 CPU）。

几条必须知道的语义：


- **为什么用独立开关，而不是把
小时数组留空**：空数组与 `null` �
�本项目里一贯表示
  **「未配置 �
� 回落默认」**（`[9, 21]` / `[22]`）�
�不是「禁用」。沿用该语义可保�
�
  老 config 行为逐字不变；真正关
闭请用 `*_enabled: false`。
- **关签到
 = 猫猫旅行也停**：旅行没有独立�
��关，它搭签到时点便车执行（见�
��节）。
  想让旅行继续跑就不能�
��签到——如需保留旅行请把 `check
in_hours` 调成你想要的时点。
- **禁
用不会擦除小时配置**：`checkin_hour
s` 原样保留，改回 `true` 即恢复原�
��点，无需补配。
- **小时值必须�
� 0-23**：写了 `-1`、`25` 之类的非法
值会在启动时**直接报错**并提示�
�用
  开关（不做静默兜底，避免�
�以为关掉了、实际却在别的整点�
�常执行）。
- 开关只影响**本进程
的定时排程**，不改变池内冷却/熔
断/禁用等既有状态机行为；
  独�
�的一次性工具（`signin.sh` / `cmd/sign
in`）是另一个进程，不受本开关约
束。
- **关签到的连带影响**：签�
�的余额查询会「余额恢复即解冻�
�被硬冷却的账号（402 余额不足）�
��
  关掉后这类账号只能等硬冷却*
*次日 04:00 自然到期**才回到池中�
�—当日余额回补不再提前解冻。


#### 猫猫旅行（随签到时点合并执�
��）

对池内每个可用账号在**签到
时点（`checkin_hours`，默认 9/21 点）
单趟推进一次**，
每趟只做一个动
作，不轮询不等待：

**为什么不�
�单独排程**：每日上限按「派出」
计 1 次/天，奖励在派出时即锁定�
�晚领不丢分；
旅行周期以小时计�
��30 分钟粒度的额外巡检不会多派�
��次，只是白白增加上游请求。
合
并到签到时点后每账号每天 2 趟，
签到 → 派猫 → 领奖一次跑完。
�
��签到排在旅行之前：先签到解冻�
��却账号，本轮旅行才能覆盖到它�
��。）

| 探测结果 | 动作 |
|---|---|

| 无猫（`buddy` 为 `null`） | 先同意
协议（幂等），再尝试领养；过门
槛则 +300 积分并获得猫 |
| `state=idl
e` 且今日未派出 | 派出 `location_id=4
`（古镇客栈；4 个地点收益/时长�
�间相同，无最优解） |
| `state=arriv
ed` | 领取到站奖励（带 `record_id`）
 |
| `state=traveling` / 今日已达上限 /
 未知状态 | 跳过 |

- **领养门槛**�
��conversation 门槛未达标时上游返回
 HTTP 400 `first_buddy task not completed yet
`，
  属预期行为——**每账号每自
然日只尝试一次**，失败后当日静�
��跳过，跨日（00:00 CST）自动重试�
��
  记录仅存内存，进程重启后清�
��。
- **限速**：账号间间隔 800ms（
46 个账号约 40s），避免触发上游�
�控。
- **每自然日 1 次派出**：按 
CST（Asia/Shanghai）自然日重置，与�
�器 `TZ` 无关。
- **失败隔离**：单�
��账号查询/动作失败只跳过该账号
本轮，不中断其他账号；401 不做�
�刷
  （token 刷新交 22:00 保活），�
��败信息按 `travel <uid>: <动作>: <错�
��>` 落日志。
- **关闭**：旅行无独
立开关——它随签到一起跑，不再
单独排程。
  因此 **`schedule.checkin_
enabled: false` 关掉签到的同时，旅�
�也一并停摆**；
  只想调整时点（
而非关闭）请改 `checkin_hours`。

签
到与保活配到同一小时（如都含 22
 点）时，两类任务都会执行。

## 
🔌 API 端点

| 端点 | 鉴权 | 说明 |

|---|---|---|
| `POST /v1/chat/completions` 
| Bearer（`api_key` 非空时） | OpenAI �
�容补全；流式/非流式；请求体上�
�� 8 MiB |
| `GET /v1/models` | Bearer（`api
_key` 非空时） | 模型列表（动态拉
取，缓存 1h；失败回落静态表 + 5mi
n 负缓存） |
| `GET /status` | Bearer（`
api_key` 非空时） | 账号状态汇总 + 
每账号详情（积分/冷却/熔断/在途
/粘性） |
| `GET /healthz` | 无 | 健康�
��查：有 healthy 且未占满账号返回 
200，否则 503；响应带身份标识（�
�下） |

> 鉴权规则：仅当 `api_key` 
非空才校验 `Authorization: Bearer <api_k
ey>`；**`api_key` 为空时上述端点直�
�放行**；`/healthz` 恒无鉴权。

`/hea
lthz` 响应示例（200/503 同结构，仅�
��态码与计数变化）：

```json
{"heal
thy": 2, "total": 3, "service": "workbuddy2ap
i"}
```

响应同时带 `X-Service: workbudd
y2api` 头。这两个身份标识用于区�
�**本网关**与同端口上
可能残留的
其他服务——后者即使返回 2xx 也�
��会带该字段/头，宿主探测据此避
免"假成功"。

### 宿主健康探测指�
��

宿主程序（如 workbuddy-switch 托�
�网关子进程）探活时，**"端口通 +
 返回 2xx" 不足以
证明打到了自己�
��网关**：同端口可能残留旧版本�
�程或别的服务，对方返回 2xx 会造
成假成功。
按校验强度从高到低�
�两种做法：

**① 强校验（推荐）
：`/status` + `api_key`**

```bash
# 期望 
200；若返回 401 则说明对面的 /statu
s 不认这个 api_key —— 不是自己的
网关
curl -s -o /dev/null -w '%{http_code}\
n' \
  -H "Authorization: Bearer <api_key>" \

  http://127.0.0.1:7863/status
```

`/status
` 挂在鉴权中间件上：只有持有正�
�� `api_key` 的本网关才会返回 200，�
��服务/其他服务
只会返回 401（或 
404）。**注意前提**：本网关 `api_ke
y` 非空才具备这个判别力；
`api_key
` 为空时 `/status` 直接放行，退化�
�弱校验。

宿主判定建议：`200` →
 健康；`401` → 不是自己的网关（�
��口被占）；连接失败 → 未就绪�
�
`5xx` → 网关已就位但池不可服务
（可再叠加 `/healthz` 的 503 语义）�
��

**② 弱校验（无凭据场景）：`/
healthz` + `service` 字段**

```bash
# 必�
��同时校验 service 字段；只判断 HTT
P 状态码仍可能假成功
curl -s http://
127.0.0.1:7863/healthz | grep -q '"service":"
workbuddy2api"'
```

适合负载均衡器 / 
容器编排这类**不该持有 api_key** �
�探活方（`/healthz` 恒无鉴权，
200=�
��服务、503=池内无可服务账号）。
判据是响应体 `service == "workbuddy2api
"`；
响应头 `X-Service` 可用于只读�
�部的探活实现。若对面返回 2xx 但
缺该标识 → 判为异常。

> 容器自
带的 `HEALTHCHECK` 用的就是 ②（仅�
�程内自检，够用）；
> 宿主做**跨
进程归属确认**时用 ①。

### 流式
行为细节

- 出站请求强制 `stream:tr
ue`；SSE 帧按 OpenAI 规范**白名单重�
��**（`reasoning_content` 保留、工具调
用按 index 合并、未知字段剥离）
-
 保证恰好一个 `data: [DONE]`（上游�
�发时兜底补写）；空流先写一帧 `
error` 再补 `[DONE]`；`error` 帧原样透
传

## 📋 请求级日志

每个 `/v1/cha
t/completions` 请求结束时输出一行表
格日志（stdout）：

```text
| #001 | 18
:31:31 | deepseek-v4 | stream | 200 | uid=085
1ce35 | TTFB=801ms | tok=60 | 23.5tok/s | tot
al=2.6s |
```

| 字段 | 说明 |
|---|---|

| `#001` | 进程级请求序号 |
| `18:31:3
1` | 结束时刻 |
| `deepseek-v4` | 模型�
��（超 11 字符截断） |
| `stream` / `s
ync` | 请求模式 |
| `200` | 状态码 |
|
 `uid=0851ce35` | 账号 UID 前 8 位 |
| `T
TFB` | 流式首帧耗时（非流式为 `-`�
�� |
| `tok` / `tok/s` / `total` | 输出 tok
en 数 / 速率 / 总时长 |

**敏感度**�
��日志不含任何 token 明文（详见[�
�全与合规](#-安全与合规)），无落
盘日志文件。

## 🛡️ 安全与合�
�

### 1. 凭据管理（auths）

- **位置
**：`./auths`（`auth_dir` 可配），文�
�名 `workbuddy-<uid>.json`
- **内容**：�
�文 `accessToken` / `refreshToken` + 账号�
��信息，结构见下：

```json
{
  "acco
unt": { "uid": "…", "enterpriseId": "…", 
"nickname": "…" },
  "auth": { "accessToken
": "明文", "refreshToken": "明文", "expir
esAt": 0, "domain": "" }
}
```

- **权限**�
��容器内以 `app` 用户（uid 10001）运
行；token 刷新由 `SaveAtomic` 以 `0600`
 原子写回（tmp + rename）；`login.sh` 
首次落盘遵循登录 umask，建议手动
 `chmod 600 auths/*.json`
- **备份**：备�
�� `auths/`（凭证）与 `data/state.json`�
��池状态：积分/冷却/计数）；配�
� Upstash 后状态另镜像至 Redis
- **切
勿提交 git**：`.gitignore` 已排除 `aut
hs/`、`data/`、`backups/`、`config.json`�
�`*.key`、`*.pem`

### 2. 网络暴露与日
志敏感度

- 默认监听 `:7863`，compos
e 暴露 `0.0.0.0:7863`，**无内置 TLS**�
�公网部署必须设置 `api_key`，建议�
��置反代/内网
- 请求日志字段：序
号/模型/模式/状态码/**uid 前 8 位**
/TTFB/token 数——**不含** `accessToken`
/`refreshToken`/`api_key` 明文（不读取 
`Authorization` 头）
- 日志写 **stdout/s
tderr**（容器内进入 `docker logs`），
代码无任何落盘日志文件

### 3. 上
游访问端点清单

| 端点 | 方法 | Ho
st | 用途 |
|---|---|---|---|
| `/v2/chat/c
ompletions` | POST | `copilot.tencent.com` | 
聊天补全（SSE） |
| `/console/enterpris
es/personal/models` | GET | 同上 | 动态�
�型列表 |
| `/v2/plugin/auth/token/refresh
` | POST | 同上 | token 刷新 |
| `/v2/bil
ling/meter/daily-checkin` | POST | `www.codeb
uddy.cn` | 每日签到 |
| `/v2/billing/mete
r/get-user-resource` | POST | 同上 | 余额
查询 |
| `/v2/plugin/auth/state?platform=CL
I` | POST | `copilot.tencent.com` | OAuth 取
授权 URL |
| `/v2/plugin/auth/token?state=`
 | GET | 同上 | OAuth 轮询取 token |
| `
/v2/plugin/login/account?state=` | GET | 同�
�� | OAuth 取账号信息 |
| `/activity/gro
wth/buddy/agreement` `first` `info` | POST/GE
T | 同上 | 猫猫旅行：同意协议 / �
�次领养 / 查询 |
| `/activity/growth/bud
dy/travel/status` `depart` `claim` | GET/POST
 | 同上 | 猫猫旅行：状态 / 派出 / 
领奖（随签到时点执行） |

> 上述
 `/v2/*` 端点是 CodeBuddy 官方 CLI/插�
�使用的接口，**未见公开 API 文档�
��属非公开/逆向接口**；本项目不�
��张任何上游接口的官方授权或稳�
��性承诺。出站统一携带 `CLI/2.63.2 
CodeBuddy/2.63.2` UA；聊天请求带账号�
��（`X-User-Id` 等），**永不携带 `X-R
efresh-Token`**。

### 4. 发布来源与合
规边界

- **无预编译 release**：仓�
�无 Release / tag，产物 = 源码自构建

- 构建命令：`CGO_ENABLED=0 go build -tr
impath -ldflags="-s -w" -o wb2api ./cmd/serve
r`（Dockerfile 多阶段：`golang:1.23-alpi
ne` 构建 → `alpine:3.20` 运行）
- 登�
��/签到/积分工具：`./login.sh` / `./si
gnin.sh` / `./credit.sh`（缺失时自动编
译对应 `cmd/*`）
- **无产物校验和**
：`go.sum` 仅约束 Go 模块依赖；Docke
r 镜像由本地 `docker compose build` 生�
��，未引用第三方镜像
- 上游 CodeBu
ddy 属腾讯系商业产品，本项目是�
�**非官方 OpenAI 兼容网关**；使用�
�账号做 API 网关涉及目标平台服务
条款与账号风险，作者不对账号封
禁、条款违约或使用结果负责

### 
5. 授权使用边界

- 仅限**本人授权
账号**、本机/私有环境测试
- 不得
共享、转售、违规分发，或用于违
反目标平台条款的用途
- 遵守 CodeB
uddy 平台服务条款与所在地法律
- �
��善保管 `auths/`（明文凭证）与网�
��端口

## 🧰 工具脚本

| 脚本 | �
�途 |
|---|---|
| `./login.sh` | OAuth 登�
� → 落盘 auth → 重启容器 |
| `./sig
nin.sh [auths_dir]` | 批量签到（过期�
�刷新） |
| `./credit.sh` / `./credit.sh -
json` | 积分日报（美化 / 原始 JSON�
� |

## 🛠️ 开发

### 本地构建与�
�试

```bash
go build ./...
go vet ./...
go 
test ./... -count=20   # 多次运行验证�
� flake
go test -race ./... -count=1
gofmt -l
 .
```

### 目录结构

```
cmd/
  server/ 
   # 主服务（config + main + 路由装配
）
  login/     # OAuth 登录工具
  credi
t/    # 积分查询工具
  signin/    # 批
量签到工具
internal/
  auth/      # 凭�
��解析 + token 刷新 + 原子写回
  pool
/      # 账号池（状态机/熔断/租约/
加权/持久化）
  scheduler/ # 定时签�
�� + 保活 + 猫猫旅行巡检
  server/   
 # HTTP handler + 鉴权 + 请求日志
  ses
sion/   # 会话粘性路由
  upstream/  # �
��游封装（chat/billing/auth/headers/sse/p
ayload/sanitize/idle）
  redisstore/# Upstas
h 持久化 + Noop 降级
```

## 免责声�
�

本项目仅供学习和研究使用。使
用者需遵守 CodeBuddy 服务条款，自�
��承担使用风险（包括账号封禁、�
��款违约等）。作者不对任何因使�
��本项目产生的直接或间接损失负�
��。

## License

本项目采用 [MIT Licen
se](LICENSE) 开源协议。

- 允许任意�
��用、复制、修改、合并、发布、�
��发、再授权及销售
- 再分发（源�
��或二进制形式，包括内嵌编译产�
��的整合项目）时，请保留原仓库�
�� MIT 版权声明与许可声明（如在 N
OTICE 或 README 中注明原始出处 `https
://github.com/Sliverkiss/workbuddy2api`，我
们将不胜感激）
- 本项目不授予任
何上游（CodeBuddy / 腾讯）接口或服
务的权利；使用者仍需自行遵守上
游服务条款（见上方免责声明）


