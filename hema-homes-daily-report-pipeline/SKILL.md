---
name: hema-homes-daily-report-pipeline
description: 盒马Homes店长大盘营收对比表（MTD+单日双份）每日自动导出全链路：浏览器重放Quark报表WidgetAction数据接口（reportId=REPORT_ID，QUARK_PARAMS令牌）取纯JSON→本地HTTP服务接收→openpyxl自动填表（含预算达成口径）→渲染六区块报表长图→经问问通道投递PNG+钉钉核心沟通群转发文字版（红色标注+真实@负责人）。当cron代理执行每日Homes导出任务、用户要求补跑/重跑Homes营收对比表、或维护该流水线（取数失败、口径疑问、预算文件更新、widget变更、投递通道调整、定时任务失败排查）时使用。触发词：Homes营收对比表、Homes日报、每日自动导出、MTD报表、单日报表、店长大盘取数、店长大盘导出、预算达成率、预算分解接入、预算覆盖审计、跨月预算缺口、QUARK_PARAMS、WidgetAction、问问发图、钉钉群转发、红色标注、真实@、文字版增强规则、管损最差部门、前场负责人名单。注意防误路由：「店长大盘看板指标图片」「店长大盘各看板指标的图片」「门店大盘关键指标日报」指 大盘取数看板_公共资产 的 gen_keymetrics_image.py/gen_keymetrics_png.py 生成的竖版长图，不属于本技能的营收对比表，此类请求不要触发本技能。
version: 1.3.0
---

# Homes 营收对比表 · 每日自动导出流水线（MTD + 单日双份）

一条已稳定运行的每日自动化流水线：从盒马 Homes「店长大盘」（FBI/Quark 报表 reportId=REPORT_ID）直接重放 WidgetAction 数据接口取数（不抓 DOM），生成【MTD 月度累计】+【单日】两份营收对比表 Excel，按部门预算算占比/达成率，各渲染成一张六区块报表长图 PNG，经「问问」IM 通道投递（图片为主，Excel 存档 D 盘）。由 cron 每日 09:00 触发，端到端试跑通过、多轮迭代稳定。

**所有脚本与 RUNBOOK 都在 `D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\`，是固定资产，勿迁移。**

## 何时使用
- cron 代理执行每日任务「Homes MTD营收对比表每日自动导出」（jobId `CRON_JOB_ID_PLACEHOLDER`，每日 09:00 Asia/Shanghai）
- 用户提到 Homes 日报 / 店长大盘 / 营收对比表 / MTD / 单日报表，要求补跑/重跑指定日期
- 维护该流水线：取数失败、口径疑问、预算文件更新、换门店、widget 变更、投递通道调整
- checks 校验异常排查、登录态失效处理
- 回答单日环比/同比口径、预算达成率计算的问题
- 用户问预算文件覆盖到哪个月、跨月预算怎么取值、下月预算是否有缺口 → 跑「预算月份覆盖审计」（见同名小节）

**边界**：生成/更新预算分解文件本身用 `hema-dept-budget-decomposition`（本技能上游）；Homes 手工导出 Excel 做交互取数看板用 `hema-dapan-flex-dashboard`；FBI ChatBI API 通道取数用 `fbi-dashboard-extractor`（不同 API 通道）。本技能只管「WidgetAction 重放取数 → 对比表 Excel/长图 → 定时投递」。「店长大盘看板指标图片 / 门店大盘关键指标日报」（手机竖版长图，含 KPI 速览 + 店长大盘/渠道成交/物流O2O/运力构成/用户大盘月/会员成交月等板块）由 `D:\WORKSPACE_PLACEHOLDER\大盘取数看板_公共资产\gen_keymetrics_image.py` / `gen_keymetrics_png.py` 生成，不属于本技能；用户说"按照最新的店长大盘各看板指标的图片格式"或"门店大盘关键指标日报"时，去公共资产目录找该脚本，不要跑本技能的营收对比表流水线。

## 执行入口（最重要）
**执行流水线前，先用 Read 完整读取 `D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\RUNBOOK.md`，严格按手册逐步执行。** RUNBOOK 是执行细节的唯一权威源（每步命令、占位符替换、checks 判读、失败处理话术）。本技能提供背景、常量速查、口径与陷阱清单；常量细节见同目录 `reference.md`。

## 每日执行流程概览（细节以 RUNBOOK 为准）
0. **定统计日期**：Bash `date` 取今天，统计日期=昨天 T-1（YYYYMMDD）；跨月第一天取上月最后一天（即上月完整 MTD）。
1. **拿令牌**：`tabs_create` 新标签 navigate `https://report-portal.example.com/rbac/quark/?reportId=REPORT_ID&ts=1` → 自动重定向到 quark.alibaba.com 且 URL 带 `QUARK_PARAMS`。标题含「登录」或 NO_QP = SSO 失效，走失败处理-A，不要硬试。记住 tabId，两个周期复用同一标签。
2. **跑 MTD（端口 8790）**：Bash **run_in_background** 起 `homes_server.py --date <DATE> --port 8790 --timeout 300` → 读 fetch_homes.js 替换占位符（`__DATE__`/`__PORT__`=8790/`__PERIOD__`=mtd）整段作为 javascript_tool 执行 → 期望返回 `POSTED ...B -> RECEIVED` → 读后台输出的一行 JSON，校验 `status=="ok"` 且 `date_echo_ok==true`。
3. **跑单日（端口 8791）**：同第 2 步，仅端口改 **8791**、`__PERIOD__`=`1d`。
4. **出长图**：对两份 Excel 各跑一次 `xlsx_to_image.py <in.xlsx> <out.png>`，期望 `IMAGE_OK ...png size=...x... bytes=...`。
5. **投递**：`qwenwork_channel_list_conversations` 找「问问」(xiaoq) → `qwenwork_channel_delegate_to_im` 把**两张 PNG** 发给用户（明确写"直接发图，不要只发文字路径"），附摘要（统计日期、MTD 总未税营收/预算达成率/总成交、单日总未税营收/总成交并注明单日环比=比上周同日）及 checks 异常提示；告知 Excel 存档路径 `out\`。**同时**按「钉钉群转发」小节规则，将文字版发往核心沟通群（红色标注 + 真实 @ 负责人）。

## 钉钉群转发（核心沟通群）
日报文字版同时发往钉钉「示例目标群」（conversationId `cidTARGET_GROUP_PLACEHOLDER`），让部门负责人收到真实 @ 通知。

- **Markdown 红字标注**：最差指标行用 `<font color="#FF0000">...</font>` 包裹；**必须带 `--title` 参数**，否则钉钉不渲染 Markdown
- **真实 @**：正文用 `<@openDingTalkId>` 占位 + 命令行 `--at-open-dingtalk-ids` 传入对应账号；两处缺一不可
- **@负责人C规则**：负责人C（前场副店）只在文字末尾 @ 一次，不逐条重复（避免刷屏）
- **负责人账号**：取自同目录 `前场负责人名单.md`（含 6 位负责人的 openDingTalkId，2026-09-10 在群成员列表中实测匹配）
- **验证**：每条 `dws chat message send` 返回 open-task-id → 紧跟 `dws chat message query-send-status --open-task-id <值>` 确认 `status==SUCCESS`
- **dws 故障恢复**：
  - `Unknown tool: dws_bash` = 钉钉连接器被禁用 → `qw_action` enable `qwenwork.settings.connector.builtin.dws`，重试一次即可恢复
  - `TOKEN_VERIFIED_FAILED`（"该组织尚未开启 CLI 数据访问权限"）= 组织级/服务端临时拦截，本地不可修，重试勿超 1 次；通道恢复后补发并注明"补发"
  - 班委群（cidzQ6rcC...）禁止通过接口上传文件（FORBIDDEN）→ 只能发文字，图片走问问或请用户手动转发

## 文字版增强规则（已固化进两个 cron payload）
每日文字版投递文案不仅发汇总数据，还须包含以下增强内容（规则源自 2026-09-10 老板确认）：

- **只提 10 个前场营收部门**：标准品部、水果部、肉类禽蛋部、日配部、蔬菜部、中餐部、海鲜水产部、烘焙部、餐饮联营部、海鲜岛。行政部、社区服务、其他部门、卡部等非前场营收部门一律不提、不参与最差排名
- **管损（MTD + 单日）**：列出整体管损率 + 损耗率；列出管损率最高 2 部门（仅前场部门参与）
- **预算达成 <100% 时**：列出预算达成率最低 2 部门（仅前场部门参与）
- **整体环比 <0 时**：列出降幅最大 2 部门（仅前场部门参与）
- **最差 2 名 @ 负责人**：部门进入任一维度最差 2 名时，@该部门负责人 + 末尾 @负责人C一次
- **⚠️ 警告段**：最差指标行用红色标注（钉钉群用 `<font color="#FF0000">`，问问用 ⚠️ 前缀）
- **负责人名单**：见同目录 `前场负责人名单.md`

## 资产清单（均在 `D:\WORKSPACE_PLACEHOLDER\Homes_MTD自动导出\`，跨任务共享，勿依赖任务临时工作区）
| 文件 | 职责 |
|---|---|
| RUNBOOK.md | 权威执行手册（cron 代理逐步执行用） |
| hema_login.py | 自动登录第二兜底：Python 原子脚本（Win32 API 前台化+视口探测+SendInput 坐标点击+剪贴板粘贴+回车，免疫视口 0x0/surface 漂移；`--dry-run` 安全自检；用法见 RUNBOOK「第二兜底」） |
| clip_cred.py | 自动登录第一兜底辅助：把 credentials.json 的账号/密码送入剪贴板（`user|pass|clear`，零回显） |
| credentials.json | Homes 凭据（用户本人填写维护；内容严禁回显/入日志/作参数传递） |
| fetch_homes.js | 浏览器端取数：重放 4 个 widget API 并 POST 到本地服务；占位符 `__DATE__`/`__PORT__`/`__PERIOD__` |
| homes_server.py | 127.0.0.1 接收服务：落盘 raw JSON → 调 fill_template.py；输出一行 JSON 结果，退出码 0=成功 |
| fill_template.py | openpyxl 填模板 + 派生列口径 + 自动 checks |
| budget_lib.py | 读预算分解.xlsx：含税→未税（÷税比），单日/MTD 累计/混合税比；CLI 自检 `python budget_lib.py <预算.xlsx> <YYYYMMDD>` |
| render_report_image.py | xlsx → 六区块纵向长图 HTML（预算达成/整体成交/盒马APP/盒马POS/线上汇总/渠道占比；负值标红、总计加粗） |
| xlsx_to_image.py | HTML → Chrome headless 2x 截图 → 按内容底边裁切 |
| 预算分解.xlsx | 预算文件（用户维护，当前覆盖 2026-09-01 ~ 10-07） |
| out/、raw/ | 产物（xlsx+png）与原始 JSON 存档 |
| 前场负责人名单.md | 10 个前场营收部门负责人名单 + openDingTalkId（钉钉真实 @ 用） |

依赖：Python `C:\Users\<USER>\AppData\Local\Programs\Python\Python312\python.exe`（openpyxl/PIL 已装）；模板 `C:\Users\<USER>\Desktop\盒马Homes自动生成MTD营收对比表.xlsx`（仅提供部门行序与表格格式，预算不读模板）。

## 关键常量（勿擅改）
- reportId=**REPORT_ID**；门店过滤值 **STORE_CODE**（示例门店）；周期 mtd / 1d；端口 MTD=8790、单日=8791（不得冲突）
- 4 个 widget（widgetId / encryptWidgetId / groupNo）：
  - WIDGET_CAT_ID / ENC_WIDGET_CAT / 14336961 —— 品类看板-部门（各部门 未税营收/成交/app/pos 明细）
  - WIDGET_GOODS_ID / ENC_WIDGET_GOODS / 14336961 —— 商品管理-部门（损耗率/管损率）
  - WIDGET_STORE_ID / ENC_WIDGET_STORE / 14336959 —— 门店看板-整体表现（总计行 成交/用户/客单价）
  - WIDGET_CHANNEL_ID / ENC_WIDGET_CHANNEL / 14336959 —— 门店看板-分渠道成交（总计行 app/pos）
- API 重放报文、conditionId 映射、checks 词汇表、预算文件结构等速查表见同目录 `reference.md`。

## 口径规则（已用 8 月真实数据逐格验证、9 月预算逐部门勾稽差异 0.0%，日常不必重复核公式）
- **线上 = 整体 − POS**（部门 AR=N−AF；总计 AR18=N18−AF18）；线上占比+POS占比≈1
- 部门日均按**有效天数** days_eff=N/P（社区服务 26 天/其他部门 5 天等场景），不是日历天数；总计行日均=AR18/days，MTD 天数=int(date[6:8])，单日 days=1
- 上期值用 cur/(1+环比) 倒推；总计行环比/同比 F18/G18=ΣE/Σ(E/(1+率))−1；损耗率 L18/M18=营收加权平均
- **预算口径**：日预算表为【含税成交】，部门当日未税预算=(线上+POS)÷当日税比；单日预算=当日未税；**MTD 预算=本月 1 日至统计日逐日累计未税**（不是月预算按天均摊）；占比=部门/总预算；整体达成率 K=E/J；**POS 达成率 AQ=(AF含税÷税比)÷POS未税预算**（单日用 daily_tax、MTD 用 mtd_tax 混合税比）。历史注：旧版 POS 达成率曾用 ÷1.1 近似去税，已被真实税比取代。
- **单日口径（实测验证）**：环比=比上周同日（7 天前同星期，不是比前一天）；单日无同比（API 返回 "-"）；日均=当日值；单版图抬头自动注明「单日 (1d)·环比=比上周同日 · 单日无同比」
- 总计行 N18 取整体表现的门店级值（与 Σ部门 N 相差约几千元属正常，门店级含未分摊部分）
- 预算表末行是「合计」行，**累加必须跳过**（误加结果恰好 2 倍——曾踩坑）

## 内置 checks 审核（fill_template.py 输出；cron 代理必须逐项看，异常必须在投递文案说明，不得静默跳过）
- `date_echo_ok` 必须为 true（防拿到旧数据）
- `budget_date_ok` / `budget_covered`：covered=false = 统计日期超出预算覆盖期 → 报表照常出但预算列缺失，必须提醒用户更新预算分解.xlsx
- `online_plus_pos_eq_1`：渠道拆分自洽
- `diff_N18_vs_sumDept`：正常约几千元
- `missing_in_api` / `new_in_api`：模板部门与 API 部门差集（当日/当月无经营部门会列出，如社区服务/其他部门）
- `K18`/`AQ18`/`E18`/`J18_budget`：达成率>200% 或营收=0 时提示用户复核

## 已知坑（全部实战踩过）
1. **钉钉群转发已打通**（2026-09-10 实测通过），但需注意两种故障：① `Unknown tool: dws_bash` = 钉钉连接器被禁用 → 用 `qw_action` enable `qwenwork.settings.connector.builtin.dws` 重新启用；② `TOKEN_VERIFIED_FAILED`（"该组织尚未开启 CLI 数据访问权限"）= 组织级/服务端临时拦截，本地不可修，重试勿超 1 次，通道恢复后补发并注明"补发"。班委群（cidzQ6rcC...）禁止通过接口上传文件（FORBIDDEN），只能发文字，图片仍走问问或请用户手动转发
2. **问问打不开 xlsx**（只能当附件、无法内嵌预览）→ 投递以 PNG 长图为主，Excel 仅存档；文件本身没坏，别误判
3. Chrome headless `--screenshot` **直写工作区会被拒（0x5）**，必须先落 `C:\Windows\Temp`，裁切后再拷到目标路径；验收时别读 Temp 缓存（可能是上一周期的），从最终成品 PNG 切段读
4. `sys.stdout = io.TextIOWrapper(...)` **二次包装**会导致编码错误/输出丢失/"I/O operation on closed file"，每个进程只包一次；库模块（render_report_image.py 被 xlsx_to_image.py 导入）禁止模块级包装，只放 CLI main() 内
5. SSO 失效（标题含「登录」或 NO_QP）→ 不要反复重试，直接经问问通道通知用户登录后重跑
6. 端口 8790/8791 每周期独占；上一次服务未结束会端口冲突（timeout 300 兜底）
7. fetch_homes.js 是 IIFE 返回 Promise，须整段作为 javascript_tool 一次执行；返回 `ERR_NO_QUARK_PARAMS` = 页面不对或登录态失效
8. **cron 子任务 token 限流**：定时任务（agentTurn）偶发因模型/token 限流失败（报 "SDK result failed without a classified error code"），属瞬时故障 → 用 `qwenwork_cron_manage_job` action=run 重触发一次即可，勿盲目改脚本
9. **bash 中文路径引号**：`D:\WORKSPACE_PLACEHOLDER\...` 等含中文路径必须整体加双引号；Git Bash 也可用 POSIX 形式 `/d/WORKSPACE_PLACEHOLDER/...`；不加引号会被空格/特殊字符拆词导致命令失败；DWS 命令须直接执行，勿接管道/重定向
10. 绝不伪造数据、不跳过校验硬交付；MTD/单日一份成功一份失败时，先发出成功的那份并说明另一份失败原因
11. 行政部 POS 达成率经常畸高——POS 预算基数极小的正常小基数现象，不是数据错误
12. present_files 无法直接呈现 D 盘路径，需先复制到工作区 outputs 再呈现
13. **MCP 坐标自动登录的两种新失效模式**（2026-09-13 实锤）：① 视口 0x0——新建 tab innerWidth=0、iframe 负坐标、resize 无效，只发生在新建 tab，已有 tab 正常；② surface 漂移——多 Chrome 窗口时 computer 工具截图/点击落到 OS 前台窗口（截图是别的页面、粘贴不进）。出现任一症状立即切第二兜底 hema_login.py（见 RUNBOOK「第二兜底」），不要用 MCP 反复重试
14. **Win32 原子脚本三坑**（hema_login.py 制作时实测）：① 64 位 Python ctypes 必须给 GlobalAlloc/GlobalLock/SetClipboardData 设 argtypes/restype，否则报 OverflowError；② Chrome 窗口处于后台时渲染子窗口矩形不刷新（只得 (0,0) 幻影矩形、真实视口子窗口缺席），必须先 Alt trick 前台化再 EnumChildWindows；③ 视口原点不是固定值（随 Chrome UI 状态实测出现过 151/221 两种），每次必须实时读 Chrome_RenderWidgetHostHWND（top≥100 且面积最大者）矩形，严禁硬编码原点或点击坐标

## 维护操作
- **预算文件滚动**（用户职责）：新预算期开始后把新一期 预算分解.xlsx 覆盖到同路径（格式=日预算表 日期×部门_线上/_POS 含税 + 日参数税比 + 部门月预算(未税)），可用 `hema-dept-budget-decomposition` 技能生成；覆盖前后跑「预算月份覆盖审计」（见下节）确认覆盖期与缺口；覆盖后跑 `python budget_lib.py 预算分解.xlsx <YYYYMMDD>` 验证单日/MTD 读数；更新 RUNBOOK 覆盖期说明，budget_covered 校验自动生效
- **换门店**：同步改三处——① fetch_homes.js 的 473514 与 473510 中 ['STORE_CODE'] ② fill_template.py 的 STORE 常量 ③ RUNBOOK 关键常量
- **widget/encryptWidgetId 漂移**（接口 rc≠0 或结构变化）：在报表页网络请求里重抓 WidgetAction 请求体，更新 fetch_homes.js 与 RUNBOOK
- **模板结构变化**：部门行序读自模板第 3~17 行 D 列、总计行固定第 18 行；行序变化须同步改 fill_template.py 与 render_report_image.py
- **改定时任务**：`qwenwork_cron_manage_job` update jobId（见上），payload 保持"先完整 Read RUNBOOK.md 再逐步执行"的结构，勿把手册细节复制进 payload（手册是唯一事实源）；改了脚本/口径须同步更新 RUNBOOK 与 cron payload，三者保持一致
- **改投递渠道**：钉钉已打通（见「钉钉群转发」小节），改投递目标群或格式改 RUNBOOK 第 5 步即可
- **资产丢失重建**：按 reference.md 的常量与接口重建（widget/encryptWidgetId/groupNo/conditionList、脚本签名、RUNBOOK 全文备份）

## 预算月份覆盖审计（跨月缺口预警）

**何时跑**：每月下旬（25 号后）、用户问「预算覆盖到哪个月 / 跨月怎么取值 / 下月预算够不够」、或 checks 报 `budget_covered=false` 时。一条命令完成审计（在流水线目录下运行，输出 5 项：sheet 清单、预算日期范围、按月分布、缺税比日期、缺口预警日）：

```bash
cd "D:/WORKSPACE_PLACEHOLDER/Homes_MTD自动导出" && "C:/Users/<USER>/AppData/Local/Programs/Python/Python312/python.exe" -X utf8 -c "
from openpyxl import load_workbook
import collections, datetime
wb = load_workbook('预算分解.xlsx', data_only=True)
print('sheets:', wb.sheetnames)
from budget_lib import BudgetReader
br = BudgetReader('预算分解.xlsx')
print('预算日期范围:', br.min_date(), '->', br.max_date(), '共', len(br.daily), '天')
print('按月分布:', dict(collections.Counter(d[:6] for d in br.daily)))
miss = sorted(d for d in br.daily if d not in br.tax_ratio)
print('税比覆盖: 缺税比日期 =', miss or '无')
last = datetime.date(int(br.max_date()[:4]), int(br.max_date()[4:6]), int(br.max_date()[6:8]))
print('缺口预警: 统计日期 >= ' + (last+datetime.timedelta(days=1)).strftime('%Y%m%d') + ' 缺预算，即报表日 ' + (last+datetime.timedelta(days=2)).strftime('%m/%d') + ' 起')
"
```

**T-1 自动切月逻辑**（budget_lib.py 已内置，无需人工干预）：
- 预算按统计日期（T-1）的 YYYYMMDD 精确索引，与 sheet 名无关；MTD 累计按年月前缀 `date[:6]` 圈定，跨月自动清零重算，不会把两个月预算混算
- 10月1日出的报表 → 统计日期 9.30 → 仍取 9 月预算（MTD = 9 月全月累计）
- 10月2日出的报表 → 统计日期 10.1 → 自动切换 10 月预算，MTD 从 10.1 重新起算

**缺口预警日期算法**：
- 第一缺口统计日 = `max_date + 1`；第一缺口报表日 = `max_date + 2`（报表日 = 统计日 + 1）
- 实例（2026-09 实测）：预算覆盖至 20261007 → 统计日期 ≥ 10/8 缺预算 → **10/9 起**报表缺预算，checks 标 `budget_covered=false`，须提醒用户更新预算分解.xlsx
- 注意「月内部分覆盖」：按月分布 Counter 可能显示某月只有头几天（如 10 月仅 10.1–10.7 的国庆档期口径），不能只看 min/max 判断跨月
- 税比缺口同样要命：日期在日预算表但不在日参数（缺税比）时，budget_lib 会静默按含税值当未税用（口径错误）；审计发现 miss 非空必须提醒用户补日参数 sheet

## 失败处理
- **A 登录态失效**：两级兜底自动登录，都失败才通知用户。第一兜底=MCP 坐标法（RUNBOOK「自动登录机制」步骤 1~9）；出现**视口 0x0**（innerWidth=0、iframe 负坐标，新建 tab 特有）或 **surface 漂移**（多 Chrome 窗口时截图/点击落到前台窗口、粘贴不进去）任一症状，立即切第二兜底=`hema_login.py` Python 原子脚本（Win32 API 单进程完成窗口激活+坐标点击+剪贴板粘贴+回车，免疫两种失效模式；用法见 RUNBOOK「第二兜底」小节）。两级都失败（无凭据/密码错误/验证码）才经问问通道发「Homes 登录态已失效且自动登录未成功，今日营收对比表未生成。请打开盒马 Homes 店长大盘完成登录后告诉我重跑。」安全阀：同一天自动登录总尝试 ≤2 次。
- **B 数据/服务异常**：重试一次对应周期链路（RUNBOOK 第 1~4 步）。仍失败则发消息附具体错误。若 MTD 成功、单日失败（或反之），先把成功的那份发出并说明另一份失败原因。
- **C dws 连接器禁用**：报 `Unknown tool: dws_bash` → 钉钉连接器被禁用，用 `qw_action` enable `qwenwork.settings.connector.builtin.dws` 重新启用，重试一次即可恢复
- **D 钉钉组织级拦截**：报 `TOKEN_VERIFIED_FAILED`（"该组织尚未开启 CLI 数据访问权限"）= 服务端拦截，本地不可修，重试勿超 1 次；通道恢复后补发并注明"补发"
- **E 班委群禁止文件上传**：报 FORBIDDEN → 班委群只接受文字消息，图片走问问或请用户手动转发
- 任何情况下不伪造数据、不跳过校验硬交付。
- 运行前提：电脑开机未休眠、Chrome 运行且 QwenWork 浏览器插件可用、Homes SSO 登录态有效；缺一即失败，按失败处理通知用户。错过 09:00 靠 missedRunPolicy=run_latest 补跑（数据仍是 T-1，只是晚发）。

## 复用到其它 Quark 报表（改造指南）
1. 入口 URL 与请求 body 的 reportId 改为目标报表
2. 打开目标报表页，F12 Network 过滤 WidgetAction，重抓每个 widget 的 widgetId/encryptWidgetId/groupNo/conditionList
3. conditionList 结构可套用：日期条件 v1=v2=DATE、周期条件 v1=[PERIOD]、门店过滤 v1=['<门店编码>']、其余留空
4. JSONP 剥壳（取首个 `(` 与末个 `)` 之间 JSON）、`returnCode===0` 判定、`returnValue.datas.datas` 取数逻辑通用
5. 本地接收服务器框架（homes_server.py）通用；填表与渲染脚本需按新报表字段/模板重写

## 红线
- 不伪造数据、不跳过校验硬交付。
- 端口/widget ID/门店值 STORE_CODE 为固定常量，勿随意改动。
- 口径公式属一次性验证通过的固定资产；若要改口径，先勾稽验证再改，并同步更新 RUNBOOK 与本技能。

## 验证
1. 每周期服务器输出 `status=="ok"` 且 `date_echo_ok==true`，checks 逐项判读无异常（词汇表见 reference.md）
2. 两条 `IMAGE_OK ...png size=...x... bytes=...` 输出，长图可正常打开（六区块齐全、负值红色、总计加粗；单版图抬头有「单日 (1d)·环比=比上周同日」）
3. 异常项确认已写进投递文案；问问通道确认两张图片可渲染；抽查 Excel 总计行 E18/N18 量级正常
4. 部门级抽查：占比合计=100%、K18=E18/J18、单日预算=预算文件当日未税值
5. 长图验收：切段（约 1400px/段）逐段读图，核对数字与 Excel 一致、无截断
6. 钉钉群转发验证：文字版 `query-send-status` 返回 SUCCESS；红色标注已渲染；真实 @ 已送达（负责人收到通知）；负责人C仅末尾 @ 一次
