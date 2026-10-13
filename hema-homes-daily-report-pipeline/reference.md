# reference.md — Homes 每日导出流水线常量速查表

> 配套 SKILL.md 使用。执行细节以 `D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\RUNBOOK.md` 为准；本表是常量与口径的速查底稿。

## 1. 报表与令牌
- reportId=**REPORT_ID**（FBI/Quark「店长大盘」）
- 入口 URL：`https://report-portal.example.com/rbac/quark/?reportId=REPORT_ID&ts=1`
- 自动重定向到 `quark.alibaba.com/dashboard/view/report.htm?...&QUARK_PARAMS=<签名令牌>`
- QUARK_PARAMS 从 `location.href` 正则提取（`/QUARK_PARAMS=([^&]*)/`）；取不到或标题含「登录」= SSO 失效（失败处理-A，勿反复重试）
- 令牌每次打开页面刷新；两个周期（mtd/1d）复用同一个已打开的标签页即可

## 2. WidgetAction API 重放
- 端点：`https://quark.alibaba.com/.dox?action=WidgetAction&event_submit_doGetWidgetData=true&callback=<cb>&_ts_=auto_<widgetId>`
- 方法：POST，`Content-Type: application/x-www-form-urlencoded`，`credentials: 'include'`
- body：`param=` + `encodeURIComponent(JSON.stringify({conditionList, groupNo}))` + `&groupNo=<gn>&widgetId=<wid>&encryptWidgetId=<enc>&externalFlag=true&reportId=REPORT_ID&env=pro&requestType=view&dtsJobId=&QUARK_PARAMS=<QP>`
- 响应为 JSONP 包裹文本：取首个 `(` 与末个 `)` 之间 JSON.parse；`returnCode===0` 才算成功；数据在 `returnValue.datas.datas`

## 3. Widget 映射表
| widgetId | encryptWidgetId | groupNo | 看板 | 用途 |
|---|---|---|---|---|
| WIDGET_CAT_ID | ENC_WIDGET_CAT | 14336961 | 品类看板-部门 | 各部门 未税营收/成交/app/pos 明细 |
| WIDGET_GOODS_ID | ENC_WIDGET_GOODS | 14336961 | 商品管理-部门 | 损耗率/管损率 |
| WIDGET_STORE_ID | ENC_WIDGET_STORE | 14336959 | 门店看板-整体表现 | 总计行 成交/用户/客单价 |
| WIDGET_CHANNEL_ID | ENC_WIDGET_CHANNEL | 14336959 | 门店看板-分渠道成交 | 总计行 app/pos |

## 4. conditionId 映射表
**cat 条件组**（品类/商品管理 widget：WIDGET_CAT_ID、WIDGET_GOODS_ID）：

| conditionId | 含义 | 取值 |
|---|---|---|
| 494366 | 开始日期 | v1=DATE, v2=DATE |
| 494367 | 结束日期 | v1=DATE, v2=DATE |
| 473800 | 周期 | v1=[PERIOD]（mtd/1d） |
| 473513 | （空） | v1=[] |
| 473514 | 门店 | v1=['STORE_CODE'] |
| 473515 | （空） | v1=[] |
| 473597 | （空） | v1=[] |
| 473598 | （空） | v1=[] |
| 473516 | （空） | v1='' |

**st 条件组**（门店看板 widget：WIDGET_STORE_ID、WIDGET_CHANNEL_ID）：

| conditionId | 含义 | 取值 |
|---|---|---|
| 473566 | 周期 | v1=[PERIOD] |
| 494364 | 日期 | v1=DATE, v2=DATE |
| 494365 | 日期 | v1=DATE, v2=DATE |
| 473509 | （空） | v1=[] |
| 473510 | 门店 | v1=['STORE_CODE'] |
| 473511 | （空） | v1='' |

DATE 格式 YYYYMMDD；门店值 STORE_CODE = 示例门店（勿改）。

## 5. 端口与本地服务器
- 端口分配：**MTD=8790、单日=8791**（两周期分开跑，勿混用）
- 启动（后台）：`python homes_server.py --date <YYYYMMDD> --port <端口> --timeout 300`
- 服务器监听 127.0.0.1，取数脚本 POST 到 `/payload`；落盘 `raw/homes_<period>_<date>.json` 后调 fill_template.py
- 服务器 stdout 输出一行 JSON（含 checks）：校验 `status=="ok"` 且 `date_echo_ok==true`
- 退出码：0=成功、2=等待超时、3=payload 解析失败、4=日期不符、5=填表失败

## 6. 脚本用法速查
| 脚本 | 用法 |
|---|---|
| fetch_homes.js | 报表页上下文执行；替换占位符 `__DATE__`/`__PORT__`/`__PERIOD__`；成功返回 `POSTED ...B -> RECEIVED`，失败 `ERR ...` |
| homes_server.py | `python homes_server.py --date <DATE> --port <PORT> --timeout 300`（run_in_background） |
| fill_template.py | `python fill_template.py <raw.json> <模板.xlsx> <out.xlsx> <period:mtd\|1d> [预算.xlsx]`（预算缺省用目录内 预算分解.xlsx） |
| budget_lib.py | 库模块 BudgetReader（daily_pretax/mtd_pretax/daily_tax/mtd_tax/min_date/max_date）；CLI：`python budget_lib.py <预算.xlsx> <YYYYMMDD>` |
| render_report_image.py | `python render_report_image.py <in.xlsx> <out.html>`；`render()` 可被导入复用 |
| xlsx_to_image.py | `python xlsx_to_image.py <in.xlsx> <out.png>`；成功输出 `IMAGE_OK ...png size=...x... bytes=...` |

## 7. 路径速查
- 脚本目录：`D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\`
- 产物目录：`D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\out\`
  - `盒马Homes自动生成MTD营收对比表_<DATE>.xlsx` / `盒马Homes自动生成单日营收对比表_<DATE>.xlsx`
  - `盒马MTD营收对比表_<DATE>.png` / `盒马单日营收对比表_<DATE>.png`
- 原始数据存档：`raw\homes_<period>_<date>.json`（period=mtd/1d；历史文件 homes_<date>.json 无前缀）
- 模板：`C:\Users\<USER>\Desktop\盒马Homes自动生成MTD营收对比表.xlsx`（仅部门行序+格式；预算不读模板）
- 预算文件：`D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\预算分解.xlsx`（当前覆盖 2026-09-01 ~ 10-07）
- Python：`C:\Users\<USER>\AppData\Local\Programs\Python\Python312\python.exe`
- Chrome：`C:\Program Files\Google\Chrome\Application\chrome.exe`
- 截图临时目录：`C:\Windows\Temp`（headless 直写工作区被拒 0x5，须先落这里再拷贝）

## 8. checks 字段词汇表（fill_template.py 输出）
| 字段 | 含义 | 判读 |
|---|---|---|
| date_echo_ok | API 返回统计日期与请求一致 | **必须 true**（防拿旧数据） |
| budget_date_ok / budget_covered | 统计日期在预算文件覆盖期内 | covered=false → 报表照出但预算列缺失，投递时提醒用户更新预算文件 |
| online_plus_pos_eq_1 | 线上占比+POS占比≈1 | 渠道拆分自洽 |
| diff_N18_vs_sumDept | 门店级总成交 − 部门Σ成交 | 差约几千元属正常（门店级含未分摊部分） |
| missing_in_api / new_in_api | 模板部门与 API 部门差集 | 列出当日/当月无数据部门或新增部门 |
| E18 / J18_budget / K18 / AQ18 | 总未税营收/总预算/总整体达成率/总POS达成率回显 | 达成率>200% 或营收=0 → 提示用户复核 |

任何一项异常都要在处理结果里向用户说明，不得静默跳过。

## 9. 预算文件结构与口径公式
- Sheets：`说明` / `9月(9.1-9.30)` / `10月(10.1-10.7)` / `部门月预算(未税)` / `日参数`
- 日预算表列结构：日期/星期/数据类型 + 13 部门_线上 + 线上小计 + 13 部门_POS + POS小计 + 日合计 + 同期参考列
- **末行是「合计」行，读取时必须跳过**（误加会恰好 2 倍——曾踩坑）
- 日预算为【含税成交】口径；`日参数` sheet 有逐日税比（门店级、逐日不同）
- 部门当日未税预算 = (线上+POS) ÷ 当日税比；POS 未税预算 = POS ÷ 税比
- 单日预算 = 当日未税；MTD 预算 = 本月 1 日至统计日逐日累计未税
- 占比 = 部门预算/总预算；整体达成率 K = E/J；POS 达成率 AQ = (AF含税 ÷ 税比) / AP未税预算
- 税比选择：单日用 `daily_tax(date)`，MTD 用 `mtd_tax(date)`（混合税比 = Σ含税 ÷ Σ未税）
- 勾稽记录：9 月逐部门勾稽差异 0.0%；全店日预算Σ(含税) ÷ 月度未税预算 = 1.1336 ≈ 税比，勾稽成立
- 历史注：旧版 POS 达成率曾用 ÷1.1 近似去税，已被真实税比取代

## 10. 单日口径（已实测验证）
- **单日环比 = 比上周同日**（7 天前同星期），不是比前一天
  - 验证：标准品部 9/7(周一) 83,292 vs 8/31(上周一) 101,125 → −17.6%，与 API 返回完全吻合；而 9/6 单日 83,318（若比前一天应为 −0.03%，不符）
- **单日无同比**（API 返回 "-"），单日报表不含同比列
- 日均 = 当日值；days=1
- 当日无经营部门（社区服务/其他部门等）→ 清零占位，且单日不保留预算占比列
- 单版图抬头自动标注「单日 (1d)·环比=比上周同日 · 单日无同比」

## 11. cron 任务与投递
- jobId：`CRON_JOB_ID_PLACEHOLDER`（任务名「Homes MTD营收对比表每日自动导出」）
- 每日 09:00（Asia/Shanghai）；missedRunPolicy=**run_latest**（错过在开机/唤醒/联网后自动补跑一次，数据仍为 T-1，不会过期）
- 依赖：开机 + 未休眠 + 联网 + Chrome 运行（带 QwenWork 插件）+ Homes 登录态有效；缺一即失败，按失败处理通知用户
- payload 结构：保持"先完整 Read RUNBOOK.md 再逐步执行"，手册细节不进 payload（手册是唯一事实源）
- 投递通道：问问 (xiaoq) IM——`qwenwork_channel_list_conversations` 查找，`qwenwork_channel_delegate_to_im` 发送；参考 chatId `efb27773-3d60-445f-8910-79a12ba94254`（以实时查询为准）
- 投递要求：两张长图作为图片附件（明确要求"直接发图，不要只发文字路径"）+ 摘要（统计日期/MTD 总未税营收/预算达成率/总成交/单日数据并注明环比=比上周同日）+ 异常提示
- 钉钉现状：两条通路均被组织权限拦截（dws 报"该组织尚未开启 CLI 数据访问权限"；QwenWork 钉钉通道 CORP_VALIDATION_FAILED）；待打通后改为直发钉钉

## 12. 失败处理速查
- **A 登录态失效**（标题含「登录」/ NO_QP）：不要反复重试。经问问通道发：「Homes 登录态已失效，今日营收对比表未生成。请打开盒马 Homes 店长大盘完成登录后告诉我重跑。」
- **B 数据/服务异常**：重试一次对应周期链路（第 1~4 步）。仍失败则发消息附具体错误。若 MTD 成功、单日失败（或反之），先把成功的那份发出，并说明另一份失败原因。
- 红线：不伪造数据、不跳过校验硬交付。

## 13. 长图渲染要点
- 57 列太宽 → 按 预算达成/整体成交/盒马APP/盒马POS/线上汇总/渠道占比 六区块纵向堆叠，宽约 1100px
- 负值标红、总计行加粗、hero 带店名+统计日期+周期类型
- Chrome headless：`--headless --disable-gpu --no-sandbox --hide-scrollbars --force-device-scale-factor=2 --virtual-time-budget=5000 --window-size=<宽>,<超高>`
- 裁边：按页面实际背景色 ±4 容差自底向上扫描内容底边，裁掉空白（防截断页脚）
- 验收：切段（约 1400px/段）逐段读图，核对数字与 Excel 一致

## 14. cron 子任务 token 限流与重试
- 现象：定时任务（agentTurn）偶发启动即失败，报 "SDK result failed without a classified error code"，任务未真正执行
- 原因：模型/token 瞬时限流，属瞬时故障，与流水线脚本无关
- 处置：用 `qwenwork_cron_manage_job` action=run 手动重触发一次即可恢复；不要因此修改脚本或 RUNBOOK
- 鉴别：若重触发后仍失败且报具体错误（登录态/端口/取数），转失败处理 A/B

## 15. bash 中文路径与 shell 引号陷阱
- `D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出` 等含中文/特殊字符路径，在 bash 命令里**必须整体加双引号**，否则拆词报错或操作到错误路径
- Git Bash 两种写法均可：`"/d/WORKSPACE_PLACEHOLDER/Homes_MTD自动导出/..."`（POSIX）或 `"D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\..."`（Windows 反斜杠）；传给 python 作文件参数时用后者
- Python 字符串字面量里反斜杠路径建议用 raw string（r"D:\..."）或双反斜杠，避免 `\x`/`\U` 转义错误（中文用户名路径尤其注意）
- DWS（钉钉 CLI）命令必须直接执行，勿接管道/重定向/命令替换，真实结果由宿主侧执行返回

## 16. 复用到其它 Quark 报表（改造清单）
1. 入口 URL 与请求 body 的 reportId 改为目标报表（入口模板 `https://report-portal.example.com/rbac/quark/?reportId=<ID>&ts=1`）
2. 打开目标报表页，F12 Network 过滤 `WidgetAction`，逐个 widget 重抓 widgetId/encryptWidgetId/groupNo 与完整 conditionList
3. conditionList 结构可套用本流水线：日期条件 v1=v2=DATE、周期条件 v1=[PERIOD]、门店过滤 v1=['<门店编码>']、其余留空
4. JSONP 剥壳、`returnCode===0` 判定、`returnValue.datas.datas` 取数逻辑通用，无需改
5. homes_server.py 接收框架通用；fill_template.py / render_report_image.py 需按新报表字段与模板重写
6. 新报表同样先做一次「真实数据逐格验证」再投入定时运行
