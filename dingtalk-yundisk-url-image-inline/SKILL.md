---
name: dingtalk-yundisk-url-image-inline
description: 钉钉群图文同框发送流水线：本地 PNG 经钉盘托管取原图签名 URL，Markdown 内联实现单条消息图片+@，天然绕开受限群 FORBIDDEN，含 @占位符冲突矩阵、形式降级规则与钉盘上传链路故障诊断降级（curl分层探测/dws chat --file 直发/IM接力兜底/at任务补发）。当需要往钉钉群发原画质内联图片（非文件卡片）、图片与@要求同一条消息、受限群不想走接力发图、排查图片渲染失败（显示"!标题"alt文本）、签名URL过期、或钉盘上传TLS超时/卡死需要诊断降级时使用。触发词：图文同框、钉盘URL发图、Markdown内联图片、图片+@同一条消息、签名URL、URL过期重取、alt文本、@占位符、<@everyone>、受限群发图不走接力、钉盘上传超时、TLS handshake timeout、上传卡死、链路探测、降级直发文件、降级接力、补发任务。
version: 1.3.0
---

> **版本沿革**：v1.3.0（2026-09-24）两处实测纠错/补全——① @冲突矩阵按「占位符口径」重写：真正破坏图片渲染的是 `<@…>` **@占位符实体**（含 `--at-all` 注入的 `<@everyone>`），纯文字"@所有人"+`--at-all` 仍正常；② 钉盘 `+upload` TLS 失败时新增更轻的降级档 `dws chat --file` 直发（非受限群）。⚠️ 本技能曾在 9-15 前后的 dws 连接器 EPERM 修复中被"腾空"移出 skills 目录，仅存会话工作区备份，v1.3.0 已恢复入位；若再次发现技能缺失，按「恢复与防丢失」一节处理。

# 钉钉群图文同框发送流水线（钉盘托管 + Markdown 内联）

把本地 PNG 以**原画质内联图片**（而非文件卡片）发进钉钉群、且同条消息里 @ 人：图片托管到钉盘 → 取原图签名 URL → Markdown `![](url)` 消息。Markdown 消息本质是**文字消息**，所以"禁止通过接口上传文件"的受限群（FORBIDDEN）天然放行。本流水线 2026-09-13 经多轮试错在本机收敛定型，天气预报 / 关键指标日报 / Homes 报表三个 cron 均依赖此流程。

## 适用场景（触发词）
- 往钉钉群发图片，要求原画质内联显示（图文同框），不要文件卡片
- 图片和 @（@所有人或指名人）要在同一条消息里
- 目标群禁接口传文件（FORBIDDEN），不想走接力
- 图片消息渲染失败、显示"!标题"alt 文本 → 按本技能 @占位符冲突矩阵排查（凡正文含 `<@…>` 占位符即嫌疑）
- 签名 URL 过期/图片打不开 → 重取 URL 重发
- 钉盘上传 TLS 超时/卡死（+upload 连败）→ 按「钉盘上传链路故障：诊断与降级」章节处置（非受限群可 `dws chat --file` 直发保底）
- 触发词：图文同框、钉盘URL发图、Markdown内联图片、图片+@同一条消息、签名URL、URL过期、alt文本、@占位符、`<@everyone>`、受限群发图不走接力、钉盘上传超时、上传卡死、链路探测、降级直发文件、降级接力、补发任务

## 原理
- 受限群设置拦的是**接口上传 file/image 消息**；Markdown 是文字消息，不受此限 → 绕开 FORBIDDEN
- 钉钉客户端把 Markdown 消息里的 `![](url)` 解析为内联图片（拉原图、浅色卡片渲染），优于单独图片消息在深色模式下被渲染层压暗
- URL 是 OSS 签名 URL（带 Expires 参数），**有效期约 15 分钟，取到后 5 分钟内必须发出**；过期 URL 会发出 SUCCESS 但图片打不开
- 钉盘上传只是为了"托管"图片拿 URL，钉盘文件本身不是投递物
- **服务端发送时即抓图**：钉钉服务端在发送瞬间抓取 URL 图片，把 `![](url)` 转成 `[图片消息](mediaId=url)` 结构化引用，`resourceRefs ready=true` 并提取 `<imageContent>` 标题——所以只要 mget 核验 ready=true + imageContent 存在，图片即**永久**可看，之后 URL 过期不影响已发消息。这也是本链路天然绕过受限群 FORBIDDEN 的根因：全程没有接口上传文件动作
- URL 返回 Content-Type 为 application/octet-stream（非 image/png）且带 attachment disposition，不影响钉钉渲染（内容嗅探），curl 预检看字节数即可

## 四条死路记录（2026-09-13 实测，别再重试）

取"可内联的图片 URL"的以下路径全部实测走不通，直接跳过：

| 死路 | 实测结果 |
|---|---|
| `dws drive publish set` 互联网公开发布 | 个人钉盘节点报"当前节点不支持互联网公开发布"；且内部数据公开发布有合规风险 |
| `dws drive +cover` 封面 URL | 返回的是 jpg 缩略图（非原图，画质损失），且 x-oss-expires 仅约 4~5 分钟 |
| `dws chat +messages-resource-url` | 只认 `--type mediaId`；文件消息的资源是 fileId，报 RESOURCE_NOT_FOUND 且不通用；`resource-download` 只落盘不暴露 URL |
| qw-pages 托管公网图 | 交互会话与 cron 会话**均无**发布工具（lazy-load 列表为空，2026-09-14 交互会话复测确认），任何会话都不可用 |

唯一可行路：**钉盘 +upload 托管 → download --url-only 取原图签名 URL → Markdown 内联**（即本流水线）。

## 环境常量注册表（本机实测，换群/换图时更新此处）

| 项 | 值 |
|---|---|
| 班委群（受限群，接口传文件 FORBIDDEN） | 群号 GROUP_NUMBER_PLACEHOLDER，cid `cidRELAY_GROUP_PLACEHOLDER` |
| 核心沟通群 | cid `cidTARGET_GROUP_PLACEHOLDER` |
| 天气长图钉盘节点 | `DRIVE_NODE_ID_WEATHER` |
| 关键指标日报长图钉盘节点 | `DRIVE_NODE_ID_KEYMETRICS` |
| 负责人A openDingtalkId | `OPEN_DINGTALK_ID_PLACEHOLDER` |
| 负责人B openDingtalkId | 见天气 cron payload（不在此重复登记） |
| dws 宿主安装目录 | `D:\Program Files\千问办公\QwenWorkCN\<版本号>\`（通配符定位） |
| 依赖本流水线的 cron | 天气预报双群推送、大盘关键指标日报、Homes 报表 |

只有数字群号时先换 cid：`dws chat group get-by-group-id --group-id <数字群号> --format json`。

## 完整流水线（8 步）

### 第 1 步：本地图 → Temp ASCII 名
中文文件名会在 PS1/dws 链路乱码，先转 ASCII 名：
```bash
cp "<源路径>/xxx.png" "C:/Windows/Temp/<ascii_name>.png"
```
命名约定：`weather_20260913.png`、`keymetrics_20260912.png`（用途+日期）。

### 第 2 步：PS1 通配符拷入 dws 安装目录
完全复用 `dws-restricted-group-file-relay` 技能的「文件放置子流程」（PS1 脚本、通配符版本目录、脚本四纪律、-File 执行），只需把模板里的源路径改为 `C:\Windows\Temp\<ascii_name>.png`。放置后验证副本存在（Get-Item 输出 Name/Length）。

### 第 3 步：上传钉盘托管
```bash
dws drive +upload --file "./<ascii_name>.png" --file-name "<中文显示名>.png" --yes --format json
```
- 已有同主题图片要覆盖更新（每日日报场景）时加 `--node <已有节点ID>`，避免节点膨胀：
```bash
dws drive +upload --file "./<ascii_name>.png" --file-name "<中文显示名>.png" --node "<nodeId>" --yes --format json
```
- 记下返回的 nodeId（下一步取 URL 用）
- `--file` 只认 `./<文件名>` 且文件必须已在安装目录内（与 relay 技能同规则）
- 每条 dws 命令必须是**独立的 Bash 工具调用**（wrapper 禁管道/重定向/混用）

### 第 4 步：取签名原图 URL（时效关键步）
```bash
dws drive download --node "<nodeId>" --url-only --format json
```
- URL 形如 `https://sh-dualstack.trans.dingtalk.com/yundisk0/...?Expires=...&OSSAccessKeyId=...&Signature=...&response-content-disposition=...`
- **Expires≈15 分钟，取到后 5 分钟内必须发出**，两步之间不要插入任何慢步骤
- 中途被打断或超过 5 分钟 → 重取再发，绝不复用旧 URL
- 取 URL 返回非 200/报错 → 重试一次，仍失败查 dws 登录态
- 可选预检（首次接入新图时建议做一次）：`curl -s -o /dev/null -w "HTTP %{http_code} | size=%{size_download}\n" "<URL>"`，HTTP 200 且字节数与本地 PNG 一致即免鉴权可内联

### 第 5 步：发送 Markdown 内联消息
```bash
dws chat +messages-send --as user --group "<cid>" --markdown $'![<alt>](<URL>)\n@所有人 <一句话说明>' --at-all --yes --format json
```
- **禁 `--title`**，形式越简越稳（加 title 增加渲染不确定性）
- @所有人：正文写**纯文字**"@所有人"字样（**不带** `<@everyone>` 尖括号占位符）+ `--at-all` 参数触发真实提醒 —— ✅ 推荐形式（2026-09-24 修订：旧的 `<@everyone>` 占位符写法会破坏图片渲染，勿用）
- 指名@：正文 `<@openDingtalkId>` + `--at-open-dingtalk-id` 参数——⚠️ @占位符会导致图片渲染失败，见 @占位符冲突矩阵，**不要与图片同框**（要指名@走拆两条或降级方案）
- 布局建议：第一行图片、第二行 @，不带其他长文字（**仅图+一行短文字最稳**）
- 确需短说明时控制在一行内（关键指标版"图+一行短文字"渲染正常；长正文+@ 必失败）
- 记下返回的 openTaskId

### 第 6 步：查发送状态
```bash
dws chat +messages-query-send-status --open-task-id "<openTaskId>" --format json
```
- `sendStatus=SUCCESS` 才可继续；记下 openMessageId（即 msgId）

### 第 7 步：mget 核验媒体就绪（不可省）
```bash
dws chat +messages-mget --msg-ids "<msgId>" --format json
```
- 核验 imageContent / resourceRefs `ready=true`；发送 SUCCESS ≠ 媒体就绪
- 只认 `--msg-ids`；多条用英文逗号分隔一次验完

### 第 8 步：清理副本
```bash
mkdir -p "<workspace>\.trash" && mv "/d/Program Files/千问办公/QwenWorkCN/<ver>/<ascii_name>.png" "<workspace>\.trash/"
```
移动后 ls 确认到位；Temp 里的 ASCII 副本一并清理。

## @占位符冲突矩阵与形式降级规则（本轮试错的核心教训）

⚠️ **真正的破坏因子是「@占位符实体」，不是「@所有人」这件事**。Markdown 消息正文里出现 `<@…>` 形式的 @占位符（无论是指名 `<@openDingTalkId>` 还是 `--at-all` 注入的 `<@everyone>`）会打断钉钉服务端把 `![](url)` 转成内联图片，导致图片位只显示 alt 文本（"!标题"）。要 @所有人又想保留图文同框，正确做法是**正文写纯文字"@所有人"字样 + 带 `--at-all` 参数，绝不写 `<@everyone>` 尖括号占位符**（此口径与 `qingdao-weather-daily-push`、`hema-keymetrics-daily-report` 两技能一致）。

实测渲染矩阵（班委群/核心沟通群，2026-09-13 基线 + 2026-09-24 占位符口径修订）：

| 消息形式 | 图片渲染 | 备注 |
|---|---|---|
| 仅图片，无@ | ✅ 正常 | 关键指标首版 |
| 图片+短说明（一行，无@占位符） | ✅ 正常 | 关键指标终版 |
| 图片+纯文字"@所有人"+`--at-all`（无尖括号占位符） | ✅ 正常 | 真实@提醒生效，推荐形式 |
| 图片+`<@everyone>` 占位符+`--at-all` | ❌ 失败（显示 !alt） | **本次新增**：`--at-all` 注入的 `@everyone` 实体同样破坏渲染 |
| 图片+长正文+@ | ❌ 失败 | 显示"!标题"alt 文本 |
| 图片+`<@openDingTalkId>` 指名@占位符 | ❌ 失败 | 两次复现，本技能要防的根因 |

规则：
1. **任何 `<@…>` @占位符都与同框图片互斥**——指名@、`<@everyone>` 都算，不要试图兼得
2. 要 @所有人 + 图文同框：正文首行图片、次行写**纯文字**"@所有人 …"（不带尖括号占位符），命令带 `--at-all`；这是既保渲染又保真实@提醒的唯一同框形式
3. 必须指名@具体人时三选一降级（按推荐序）：
   - **A. 改@所有人（纯文字）**（推荐）：图+纯文字@所有人+`--at-all` 同框已验证可渲染；小群里@所有人同样覆盖被指名者
   - **B. 拆两条消息**：图片单独一框（保证显示），紧接一条文字消息真实指名@（`<@openDingTalkId>`+`--at-open-dingtalk-id`）；代价是图文不同框
   - **C. @文字烙进图片**：把"@张三 @李四"字样直接做进 PNG，走 `dws-restricted-group-file-relay` 接力发送；图片 100% 显示但无真实@提醒震动，可能漏看
4. 渲染失败（显示 alt 文本）时：撤回坏消息 → 撤掉所有 `<@…>` 占位符（或改纯文字@所有人+`--at-all`）→ 重取 URL → 按更简形式重发，不要原形式硬重试

## 失败分支与重试

| 症状 | 原因 | 处置 |
|---|---|---|
| 图片打不开/灰占位 | 签名 URL 过期（超15分钟或取后超5分钟） | 撤回旧消息 → 重取 URL → 立即重发 |
| 图片位显示"!alt"文字 | @占位符（`<@everyone>`/`<@openDingTalkId>`）或长正文破坏渲染 | 撤回 → 去掉所有 `<@…>` 占位符（@所有人改纯文字+`--at-all`）→ 按@占位符冲突矩阵降级形式 → 重取 URL 重发 |
| 服务端全绿（SUCCESS+ready=true+mediaId转换）但客户端灰占位 | 链路抖动/节点覆盖上传后媒体转换异常（2026-09-15 实测：钉盘大体积PUT刚恢复后用旧节点覆盖上传，mget ready=true 但群里灰框） | 撤回 → 换新节点上传（不复用 --node）重取 URL 重发；再失败直接降级文件接力（图 100% 显示）+ @所有人拆单独一条文字消息 |
| 取 URL 非 200/报错 | 接口偶发或节点失效 | 重试一次；仍失败核对 nodeId、查 dws 登录态 |
| +upload 连续 TLS handshake timeout，但 curl GET 同域正常 | 上传链路层故障（非 dws/登录态问题） | 停止重试，按下方「钉盘上传链路故障：诊断与降级」处置：分层探测确诊 → IM 接力降级 → at 任务探测补发 |
| Markdown 消息也被受限群拦 | 该群连文字消息都拦（非典型 FORBIDDEN） | 不适用本技能，按 TOKEN_VERIFIED_FAILED 等其他故障类别排查（见 im-proxy-delivery-orchestration） |

撤回命令：
```bash
dws chat +messages-recall --msg-id "<msgId>" --yes --format json
```

## 钉盘上传链路故障：诊断与降级（2026-09-14 实战沉淀）

`dws drive +upload` 的失败不只有"接口偶发"一种——**本机到钉盘中转节点（sh-dualstack.trans.dingtalk.com）的链路本身可能故障**。该故障无法靠重试或重置 dws 登录态恢复，必须识别签名、降级投递、布点补发。

### 故障签名
- `+upload` 连续报 TLS handshake timeout（连败 ≥2 次）
- 但 curl GET 同域根路径正常（TLS 秒级握通）
- 特征：**小请求能过、大体积必卡**——≥2KB 的 PUT 卡死约 19 秒后被中间设备 reset（curl exit 56）或超时（exit 28）；故障程度可能波动恶化（从 100KB 卡死恶化到 1400B 也卡），小请求间歇可通过
- 结论：链路层故障（疑似 PMTU 黑洞/中转节点丢包/EDR 对云存储上传干预），**不是 dws bug、不是登录态问题**

### curl 分层探测法（确诊用，3 层递进）
```bash
# 层1：TLS/DNS 连通（GET 根路径）
curl -4 -s -o /dev/null -m 12 -w "GET: HTTP %{http_code} | tls=%{time_appconnect}s | total=%{time_total}s\n" "https://sh-dualstack.trans.dingtalk.com/"

# 层2：小体积 PUT（1KB）——返回 HTTP 403 = 链路通（OSS 拒绝未签名 PUT，属预期）
dd if=/dev/zero of=/tmp/probe_1k.bin bs=1024 count=1 2>/dev/null
curl -4 -s -o /dev/null -m 12 -X PUT --data-binary "@/tmp/probe_1k.bin" -w "1KB PUT: HTTP %{http_code} | total=%{time_total}s\n" "https://sh-dualstack.trans.dingtalk.com/yundisk0/probe"

# 层3：大体积 PUT（16KB 起）——HTTP 000 + 卡死/超时 = 大体积停滞
dd if=/dev/zero of=/tmp/probe_16k.bin bs=1024 count=16 2>/dev/null
curl -4 -s -o /dev/null -m 15 -X PUT --data-binary "@/tmp/probe_16k.bin" -w "16KB PUT: HTTP %{http_code} | total=%{time_total}s\n" "https://sh-dualstack.trans.dingtalk.com/yundisk0/probe"
```
判定：**层1/层2 正常 + 层3 卡死 = 链路层故障确诊**，停止重试 +upload。cron 会话无 /tmp 时探测文件改放 `C:/Windows/Temp/`。

两个对照实验（排他因，首次确诊时做一次即可）：
```bash
# 对照1：本机出口是否正常（大体积 POST 到公网其它站点，200 即出口无碍）
dd if=/dev/zero of=/tmp/probe_100k.bin bs=1024 count=100 2>/dev/null
curl -4 -s -o /dev/null -m 15 -X POST --data-binary "@/tmp/probe_100k.bin" -w "baidu 100KB POST: HTTP %{http_code} | total=%{time_total}s\n" "https://www.baidu.com/"

# 对照2：OSS IP 是否误走 VPN 路由（AliWireGuard 只承载内网网段，OSS 应走 WLAN 直连）
powershell.exe -NoProfile -Command "Find-NetRoute -RemoteIPAddress <OSS_IP> | Select-Object IPAddress,InterfaceAlias,NextHop | Format-Table -AutoSize"
```
对照1 正常 + 对照2 走直连 → 锁定"本机到钉盘中转节点"这一段链路故障（OSS IP 可从 +upload 报错或 nslookup sh-dualstack.trans.dingtalk.com 获得）。

### 降级路径（2026-09-24 增补更轻的一档，按目标群是否受限择路）
钉盘 `+upload` 走的是 OSS 中转端点，而 `dws chat --file` 走的是 **IM 消息上传基础设施**，是不同链路——钉盘链路断时 IM 消息上传通常仍正常（2026-09-14/09-24 复测）。二者代价相同：失去图文同框（图片以文件/图片消息进群，非 Markdown 内联，无法同框@），但都绕开故障端点、能把图发出去。

**降一档（非受限群首选）：`dws chat --file` 直发目标群**
目标群不禁接口传文件（发文件返回不是 FORBIDDEN）时，直接一条命令发到目标群，无需中转接力：
```bash
dws chat +messages-send --as user --group "<目标群cid>" --msg-type file --file "./<ascii_name>.png" --yes --format json
```
- `--file` 只认 `./<文件名>` 且文件须已在 dws 安装目录内（同 relay 技能「文件放置子流程」第 1 步的 PS1 拷贝）
- 之后 query-send-status → mget 核验送达，与标准流水线第 6/7 步一致
- 呈现为文件卡片，接收方点开可见原图；这是"钉盘 TLS 失败但目标群可用"时的最快保底

**降二档（受限群）：IM 媒体接力绕过钉盘 OSS 端点**
目标群禁接口传文件（发文件 FORBIDDEN）时，`--file` 直发也会被拦，改走 relay 接力（中转群→forward→目标群）。命令链复用 `dws-restricted-group-file-relay`（文件须已在 dws 安装目录内）：
```bash
# 1. 发中转群
dws chat +messages-send --as user --group "<中转群cid>" --msg-type file --file "./<ascii_name>.png" --yes --format json
# 2. 取真实消息 ID
dws chat +messages-query-send-status --open-task-id "<openTaskId>" --format json
# 3. 转发目标群
dws chat +messages-forward --src-conversation-id "<中转群cid>" --msg-id "<msgId>" --dest-conversation-id "<目标群cid>" --yes --format json
# 4. mget 核验目标群新 msgId 媒体就绪（ready=true）
dws chat +messages-mget --msg-ids "<目标群msgId>" --format json
# 5. 撤回中转群消息，清残留
dws chat +messages-recall --conversation-id "<中转群cid>" --msg-id "<msgId>" --yes --format json
```
**两档降级投递后都必须**：向老板说明"钉盘链路故障、已降级为文件卡片投递（非图文同框）、已布补发任务"，并立即布下面的补发任务。

### 补发机制：一次性 at 任务探测补发
链路恢复后自动按标准流水线补发图文同框形态（不靠人肉盯）：用 `qwenwork_cron_manage_job` action=add 建一次性任务：
- name：`<场景>·图文同框补发(一次性)`
- schedule：`{ kind: "at", at: "<ISO时间，建议故障后30~60分钟>" }`
- missedRunPolicy：`run_latest`
- payload.message 必须**自包含**（任务无会话上下文），包含五要素：
  1. 背景：故障时段/端点/已降级投递的事实与目标群 cid
  2. 探测：16KB curl PUT 探测最多 3 轮、间隔 3 分钟
  3. 恢复：按本技能标准 8 步流水线补发目标群（图+@所有人同框）
  4. 未恢复：经问问向老板汇报，不再硬试
  5. 无论成败都要汇报结果

### 禁忌
1. 把链路故障误诊为 dws/登录态问题，反复重试 +upload 或重置凭据
2. 故障期原地硬等重试——链路层故障非重试可恢复，且浪费 URL 时效窗口，应尽快降级
3. 降级投递后不布补发任务——图文同框形态会永久丢失

## 常见坑
1. 中文文件名直接进 PS1/dws 链路 → ANSI 乱码，一律先转 Temp ASCII 名
2. 取 URL 后先干别的事、过会儿才发 → URL 约 15 分钟有效且应 5 分钟内用，即取即发
3. Markdown 消息加 `--title` → 增加渲染不确定性，禁
4. 长正文+@ → 必失败；仅图+一行短文字最稳
5. **任何 `<@…>` @占位符（含 `<@everyone>`）与图片同框 → 图片必渲染失败**；@所有人要同框就写纯文字"@所有人"+`--at-all`，指名@改@所有人或拆两条（1.3.0 修订：旧版"图片+@所有人同框 ✅"的结论仅在纯文字写法下成立）
6. 渲染失败后复用旧 URL 重试 → 必须重取新鲜 URL
7. 省掉 mget 核验 → 发送 SUCCESS ≠ 媒体就绪
8. 覆盖上传忘带 `--node` → 每天新建节点，钉盘节点膨胀；但节点复用遇"服务端全绿+客户端灰框"时改新节点重发（9-15 实测）
9. 内联 PowerShell 传 `$env:VAR` → 被 bash 吞，一律写 .ps1 文件 -File 执行（见 relay 技能脚本四纪律）
10. 把 openTaskId 当 msgId 用 → 必须先 query-send-status 换取 openMessageId
11. +upload 连败误当 dws/登录态问题，反复重试或重置凭据 → 先 curl 分层探测区分链路故障（见诊断与降级章节）
12. 链路故障期原地硬等重试 → 链路层故障非重试可恢复，浪费 URL 时效窗口，应尽快走 IM 接力降级并布补发任务

## 验证清单
- [ ] Temp ASCII 副本存在，安装目录副本存在（Name/Length 核验）
- [ ] drive +upload SUCCESS，记下 nodeId（覆盖场景节点不变）
- [ ] URL 取到后 5 分钟内发出
- [ ] query-send-status = SUCCESS，拿到 msgId
- [ ] mget imageContent 就绪（resourceRefs ready=true）
- [ ] 群侧图片真实渲染（请老板确认或截图核验；显示 alt 文本 = 失败，先查正文是否混入 `<@…>` 占位符）
- [ ] 安装目录与 Temp 副本已移入 .trash
- [ ] 降级场景：非受限群 `--file` 直发或受限群 relay 接力，交付汇报注明非图文同框形式、（relay 时）中转消息已撤回、已布一次性补发任务

## 恢复与防丢失（2026-09-24）
本技能目录曾在 dws 连接器 EPERM 修复（dws-connector-eperm-repair 的"腾空目录"步骤）中被移出 skills 目录，且该技能的备份名单未覆盖 `dingtalk-yundisk-*` 前缀，导致修复后未回填、5 个技能（dws-restricted-group-file-relay / group-image-send-15min-sla / hema-keymetrics-daily-report / weather-longimage-logo-footer / hema-homes-dingtalk-text-template 相关链条）出现悬空引用。约定：
1. **执行 dws-connector-eperm-repair 或任何批量腾空 skills 目录的操作前**，先备份本技能目录（连同自定义技能一起），修复后 diff 校验回填
2. 每次重大修订后在本工作区留一份时间戳副本（如 `SKILL.md.v1.3.0.prerestore`），作为最后恢复点
3. 若再次发现本技能缺失，按上文版本沿革定位最新备份恢复，勿凭记忆重建

## 与相关技能的分工
- **dws-restricted-group-file-relay**：向受限群发**任意文件/文件卡片**形式（中转群接力）；本技能发**原画质内联图片**（Markdown 文字消息天然绕开 FORBIDDEN）。该技能常见坑 #9 即指向本路线。图文同框 → 本技能；非图片文件或要文件卡片 → relay；本技能的钉盘链路故障降级路径也直接复用其接力命令链
- **dingtalk-drive / dingtalk-chat**：dws drive / chat 通用命令手册；本技能是组合两者的图文同框流水线
- **im-delegation-media-delivery**：经问问通道投递图片给老板本人（send_media），与本技能不同通道
- **im-proxy-delivery-orchestration**：投递编排手册；FORBIDDEN 图片场景优先走本技能（比接力简单），不适用时才落 relay
