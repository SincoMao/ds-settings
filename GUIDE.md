# DeepSeek-V4 × Claude Code（Windows）配置与作业手册

> **定位**：本手册面向两个生产场景——**① BMC 级超大复杂代码体系开发**（从嵌入式底层到上层 Web 的全栈工程）与 **② A 股市场持续深度行情分析**（数据值绝对可靠、考虑问题全面、可长期稳定复跑）——并将 **③ 多模态证据链**（报错截图诊断、录屏取证、图纸理解、图表读数、扫描件阅读）作为两场景共用的第一类需求纳入体系。
>
> **总原则**：成本与速度不纳入决策考量。所有决策仅对三项指标负责：**正确性、完备性、可靠性**。凡"节省 token / 提升速度"与"更准确 / 更完整 / 更稳定"相冲突之处，一律选择后者。模型策略为**主线单旗舰 + 视觉专线**：一切文本推理（含后台辅助任务与全部常规子代理）仅运行 **DeepSeek-V4-Pro**，无备选、无回退；图片输入是 DeepSeek 系列中 **DeepSeek-V4-Flash-Vision-Exp** 的独占能力，通过专用 vision 子代理承接，与主线物理隔离。
>
> **适用环境**：Windows 原生 Claude Code CLI（2026-08 时点 v2.1.247）+ DeepSeek 开放平台 API（充值按量计费）。
> **文档快照日期**：2026-08-27。全部机制性结论以两个官方文档站为准：[Claude Code Docs](https://code.claude.com/docs/en/overview)、[DeepSeek API Docs（中文）](https://api-docs.deepseek.com/zh-cn/)；两大领域的作业规程（第 9、10 章）为专家经验层，独立于上述文档。

---

## 目录

- **第一部分：需求定调**
  - 1. 场景需求规格（由三类需求反推的硬性需求）
  - 2. 对"高缓存命中"的定调
  - 3. 模型策略：主线单旗舰 + 多模态专线
  - 4. 需求 → 配置追溯矩阵
- **第二部分：配置**
  - 5. 工作原理速览
  - 6. 配置文件位置、生效时机与验证
  - 7. 参数逐项详解（附权威来源）
  - 8. 有意不设置的参数及原因
- **第三部分：作业手册**
  - 9. 场景 ①：BMC 级超大代码体系开发（含 CLAUDE.md 模板）
  - 10. 场景 ②：A 股深度行情分析（含数据铁律与 CLAUDE.md 模板）
  - 11. 多模态证据作业手册（vision 子代理）
- **第四部分：运维**
  - 12. 余额、并发与峰谷时段（可用性机制，而非节省手段）
  - 13. 升级 SOP
  - 14. 一致性管理：任务中途的行为漂移源清单
  - 15. 部署后验证清单
  - 16. 故障排查速查表
- **第五部分：附录**
  - 17. DeepSeek-V4 版本时间线与版本门槛
  - 18. 参考链接汇总

---

# 第一部分：需求定调

## 1. 场景需求规格

最深思考、最完整上下文、最少截断、最高缓存命中是方向性目标，并非可直接执行的需求。本章将需求分解为**可验证、可映射到配置项**的条目（编号 A1–A6 / B1–B6 / C1–C3，第 4 章给出每条需求到具体参数的追溯）。

### 1.1 场景 ①：超大复杂代码体系（以 BMC 为例）

BMC（基板管理控制器）开发是"单任务跨越整个技术栈"的典型代表：Yocto/bitbake 构建体系、u-boot、内核驱动（I2C/IPMB/KCS/MCTP）、设备树、D-Bus 服务群（OpenBMC 的 phosphor-* 仓库族）、IPMI / Redfish（DMTF DSP0266）协议服务、直至 WebUI 前端。由此反推的硬需求：

| 编号 | 需求 | 判定标准 |
| --- | --- | --- |
| **A1 跨栈上下文** | 单个任务的推理必须同时容纳多仓库、多语言（C / C++ / DTS / Python / JS）的真实代码与工程约定 | 1M 窗口全程可用；多目录挂载时各仓库的 CLAUDE.md 全部载入；巨型源码树的检索不超时 |
| **A2 长周期执行不中断** | 小时级构建、跨多轮的多阶段迁移中，任何一环的超时/断流/限流都不得终结任务 | 请求超时、流式看门狗、命令超时、子代理失速阈值、限流重试全部按最坏情况配置 |
| **A3 证据先行** | 协议字段、寄存器位、D-Bus 接口签名等**必须来自读到的真实代码/规范**，不允许凭模型记忆书写 | 工具输出（文件读取、命令输出、子代理回传）在文档允许的上限内不截断；上游开源仓库可被直接检索与全文读取 |
| **A4 规范硬约束** | 编码规范、内存约束、安全启动链、许可证边界、"禁改区"必须被强制执行而非口头遵守 | CLAUDE.md 硬规则 + 构建/静态检查/测试的验证闭环，必要时以 deny 规则或 hooks 兜底 |
| **A5 大规模并行不互踩** | 代码审计、批量迁移需要几十上百个子代理并行，且互不污染工作区 | 子代理并行能力、后台任务能力、worktree 隔离、agent teams 全部启用且引导上限放开 |
| **A6 行为一致性** | 同一任务内模型行为不得漂移（换模型、换推理档、换系统提示都会改变行为） | 模型/effort 固定为单一配置；升级只发生在任务间隙（见第 13、14 章） |

### 1.2 场景 ②：A 股深度行情分析

A 股的特殊性在于：交易规则密集（T+1、分板块涨跌停、ST 制度、停复牌）、口径陷阱密集（复权、单位、报告期 vs 公告日）、信息面变化迅速（政策、公告、资金）。"绝对可靠的数据值"在 LLM 语境下只有一种实现方式：**模型的每一个数字都来自当轮工具调用的返回值，并可溯源**。硬需求：

| 编号 | 需求 | 判定标准 |
| --- | --- | --- |
| **B1 数据值绝对可靠** | 禁止模型凭参数记忆给出任何行情/财务数字；关键数字须双源交叉验证 | 每个数值可追溯到具体工具调用（接口名/URL + 获取时间）；检索链路（内置 WebSearch）与原始页面抓取链路（Bash 直抓）全程可用且结果不被截断 |
| **B2 口径先于计算** | 任何计算前先锁定：复权方式、单位（元/万元/亿元）、币种、时区、报告期与公告日、交易日历 | CLAUDE.md 中的口径声明规程强制执行（见 10.3 模板） |
| **B3 时点正确性** | 回看历史必须用时点数据（point-in-time），防幸存者偏差、防以"当前已知信息"解释历史 | 作业规程强制声明数据截止时间；停牌/退市/新股按规程单独处理 |
| **B4 考虑问题全面** | 多空两面、宏观—行业—个股—资金—情绪—估值多维检查，必须主动搜索反驳证据 | 深思考锁定 `reasoning effort = max`（DeepSeek 的最深档）；并行子代理/fork 分头核查；检查清单进 CLAUDE.md（见 10.4） |
| **B5 持续稳定复跑** | 盘后例行分析可无人值守运行，结论产物落盘存档、可审计 | bypassPermissions 全自动；无人值守重试；结论文件化；会话记录长期保留 |
| **B6 推理可审** | 深度结论的中间推理必须可见、可复查 | 思考过程全程开启且界面可展开审阅 |

### 1.3 多模态证据链（两场景共用）

debug 工作对截图的依赖是刚性的：客户报障截图、CI 界面报错、示波器/仪表截图、前端还原度比对、复现录屏——这些证据无法要求提供方转成文本。A 股侧同样存在 K 线形态、研报扫描件、数据平台图表等只以视觉形态存在的证据。DeepSeek 系列中**只有 `deepseek-v4-flash-vision-exp` 接受图片输入，其他模型收到图片直接返回 400 错误**；官方多模态边界为**仅图片**（视频与文档输入官方未见/不支持），因此"极致压榨多模态模型"的形态是：图片直读、视频抽帧后读、扫描件按图读，全部视觉证据统一收敛为图片输入这一条通道，全部由 vision-exp 的视觉理解处理；数字原生 PDF 的文字层则提取为文本走主线 Pro（两模型各压榨各的）。由此反推：

| 编号 | 需求 | 判定标准 |
| --- | --- | --- |
| **C1 视觉证据可被机器读取** | 报错截图、UI 缺陷图、架构图/原理图、K 线图、录屏视频必须被模型直接读取而非人工转述 | 存在稳定的视觉模型通道；图片经 Read 直读、视频经 ffmpeg 抽帧收敛为图片进入该通道 |
| **C2 主线代码能力不妥协** | 引入视觉能力不得以降低主线代码能力为代价（vision-exp 纯文本能力仅为 Flash 级） | 视觉模型不进入任何模型别名位与 `/model` 选单；只在专用子代理内运行 |
| **C3 视觉证据可靠性** | 视觉结论须区分"图中可见"与"由此推断"，不得以视觉估读冒充精确数据 | vision 子代理系统提示固化转录先行/不确定性标注/抽帧与裁剪保全细节的规程（见第 11 章） |

- 来源：[DeepSeek – 图像理解（使用限制："仅视觉模型接受图片，其他模型会返回 400 错误"；三种图片传入方式；无视频/文档输入）](https://api-docs.deepseek.com/zh-cn/guides/vision)、[DeepSeek – 使用 Anthropic API（image 内容块 source 支持 base64/url/file）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)、[DeepSeek – Vision-Exp 发布公告（纯文本能力与 Flash 持平、多模态 API 三格式）](https://api-docs.deepseek.com/zh-cn/news/news260821)

## 2. 对"高缓存命中"的定调

本手册对该机制的定调如下：

1. **缓存是无损机制，与准确性零冲突**。DeepSeek 上下文硬盘缓存对所有用户默认开启、无需任何配置；官方明确"硬盘缓存只匹配到用户输入的前缀部分，**输出仍然是通过计算推理得到的**……其输出效果与不使用硬盘缓存相同"。它不需要被刻意追求，也不需要被防范。
2. **DeepSeek 缓存的匹配单元与传统前缀缓存不同**。受 Sliding Window Attention 机制影响，缓存以"**缓存前缀单元**"为完整匹配单位（请求结束位置落盘、公共前缀检测落盘、固定 token 间隔落盘三种时机）。这只影响命中率的形成方式，不影响正确性；命中情况可在响应 `usage` 的 `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` 观察。
3. **本配置没有任何为缓存牺牲能力的设定**。唯一与缓存相关的参数 `CLAUDE_CODE_ATTRIBUTION_HEADER=0` 的作用是"从系统提示词开头移除客户端版本指纹块"——这是减少与任务无关的提示噪音，缓存收益属于附带效果。Claude Code 请求中的 `cache_control` 标记被 DeepSeek 端点忽略（兼容性表），无需处理。
4. **第 14 章"行为漂移源清单"的真正价值是一致性，而非费用**。中途切换模型、切换 effort、升级版本——这些操作的要害是**改变了模型的行为条件**，缓存重建只是其可观测的影子。该清单服务于 A6/B5，与费用无关。
5. 客户端侧前提：Claude Code 自 v2.1.237 起修复了"经 LLM 网关 / 自定义 base URL 会话的提示缓存"问题，本配置的版本门槛（7.11）已覆盖。

- 来源：[DeepSeek – 上下文硬盘缓存](https://api-docs.deepseek.com/zh-cn/guides/kv_cache)、[DeepSeek – 使用 Anthropic API（`cache_control` 忽略）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)、[Claude Code – CHANGELOG v2.1.237](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)、[Claude Code – env-vars（`CLAUDE_CODE_ATTRIBUTION_HEADER`）](https://code.claude.com/docs/en/env-vars)

## 3. 模型策略：主线单旗舰 + 多模态专线

DeepSeek API 当前提供三个模型：`deepseek-v4-pro`（DeepSeek-V4-Pro-0813）、`deepseek-v4-flash`（DeepSeek-V4-Flash-0731）、`deepseek-v4-flash-vision-exp`（实验性，额外支持图片输入）。三者均为 1M 上下文 / 最大 384K 输出 / 支持思考模式与 Anthropic API。

**主线：全部文本推理仅用 V4-Pro。** "复杂代码开发只能使用代码能力最强的模型"有官方基准的硬数据支撑（均为官方更新日志口径，DeepSeek Harness 极简模式 + max 档测得）：

| Code Agent 基准 | V4-Pro-0813 | V4-Flash-0731 | Vision-Exp |
| --- | --- | --- | --- |
| Terminal Bench 2.1 | **87.9** | 82.7 | 83.9 |
| DeepSWE | **62.7** | 54.4 | 59.3 |
| NL2Repo | **61.5** | 54.2 | 57.7 |
| DSBench-Hard | **67.2** | 59.6 | 63.6 |
| Cybergym | **83.3** | 76.7 | — |

Pro 在全部 Code Agent 基准上领先，官方发布公告亦明确 Flash "在简单任务上与 V4-Pro 旗鼓相当，但在高难度任务上仍有差距"；DeepSeek 官方的 Claude Code 接入指南同样将主模型位设为 `deepseek-v4-pro[1m]`。因此：**四个别名位全部 V4-Pro，`/model` 选单不设任何备选**；后台辅助功能（Haiku 位：会话标题、WebFetch 内容抽取）同样采用 Pro——辅助环节不使用次级模型；`fallbackModel` 保持不设置，过载时按第 7.6 节看门狗重试同一模型，不允许静默降级出活。

**多模态专线：vision-exp 只存在于 vision 子代理。** 图片输入是 vision-exp 的独占能力（其他模型收图直接 400），多模态需求使第二个模型不可避免——但它的纯文本能力仅为 Flash 级（官方："与 DeepSeek-V4-Flash 正式版持平"），绝不允许其承接任何代码/分析主线工作。专线设计原则与主线对称——**极致压榨 vision-exp 的多模态承载**：官方多模态边界为仅图片（视频/文档输入未见与不支持），故图片直读、视频 ffmpeg 抽帧、扫描件按图读，一切视觉证据收敛为图片输入这一条官方明确支持的通道（Anthropic 端点 image 块官方兼容表背书，无需网关验证）；数字原生 PDF 文字层提取为文本走主线 Pro。本配置的隔离手段是结构性的：

- vision-exp **不进入任何 `ANTHROPIC_DEFAULT_*_MODEL` 别名位**，`/model` 选单中不存在，主线不可能误切换到它；
- 它仅出现在 vision 子代理定义（`agents/vision.md`）的 `model` frontmatter 中——视觉任务在子代理的独立上下文中完成，文本结论回传主线（Pro）；
- **`CLAUDE_CODE_SUBAGENT_MODEL` 必须保持不设置**（第 8 章）：该变量的优先级高于子代理定义的 `model` frontmatter，一旦设置，vision 子代理会被强制拉回文本模型、读图必然 400。不设置时，常规子代理按官方默认 `inherit` 继承主会话的 Pro，vision 子代理按 frontmatter 命中 vision-exp——两条线各得其所。

`deepseek-v4-flash` 在本配置中**零席位**：成本与速度不纳入考量，任何 Flash 位都是纯降级。

模型矩阵：

| 位置 | 值 | 承担的工作 |
| --- | --- | --- |
| Opus 位 | `deepseek-v4-pro[1m]` | 主会话（默认 `model: "opus"`） |
| Sonnet 位 | `deepseek-v4-pro[1m]` | 任何解析到 sonnet 别名的路径（含 `opusplan` 执行阶段等） |
| Fable 位 | `deepseek-v4-pro[1m]` | 误触 `/model fable` / `best` 时的兜底 |
| Haiku 位 | `deepseek-v4-pro` | 后台辅助功能（标题、WebFetch 抽取等）。`[1m]` 后缀官方仅对 Opus/Sonnet 位提供文档保证，后台任务极短，用裸模型名 |
| 常规子代理 | （不设 `CLAUDE_CODE_SUBAGENT_MODEL`，默认 `inherit`） | Explore、fork、workflow 代理、agent teams 队友全部继承主会话 = V4-Pro（1M） |
| vision 子代理 | frontmatter `model: deepseek-v4-flash-vision-exp` | 一切视觉证据（图片/视频抽帧/扫描件）的唯一通道（第 7.13、11 章） |

- 来源：[DeepSeek – 更新日志（三模型全部基准数字）](https://api-docs.deepseek.com/zh-cn/updates)、[DeepSeek – V4 发布公告（Flash 与 Pro 的定位差异）](https://api-docs.deepseek.com/zh-cn/news/news260424)、[DeepSeek – 接入 Claude Code（官方模型位推荐）](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)、[DeepSeek – 图像理解（仅视觉模型接受图片）](https://api-docs.deepseek.com/zh-cn/guides/vision)、[Claude Code – Sub-agents（`model` frontmatter 与解析优先级：环境变量 > 调用参数 > frontmatter > 继承）](https://code.claude.com/docs/en/sub-agents)、[Claude Code – Model configuration（别名机制、Fallback model chains）](https://code.claude.com/docs/en/model-config)

## 4. 需求 → 配置追溯矩阵

| 需求 | 对应配置/机制 | 详解章节 |
| --- | --- | --- |
| A1 跨栈上下文 | `deepseek-v4-pro[1m]` 全别名位、`CLAUDE_CODE_MAX_CONTEXT_TOKENS=1048576`、`CLAUDE_CODE_AUTO_COMPACT_WINDOW=786432`、`CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`、`CLAUDE_CODE_GLOB_TIMEOUT_SECONDS=60` | 7.2 / 7.4 |
| A2 长周期执行 | `API_TIMEOUT_MS=3000000`、`API_FORCE_IDLE_TIMEOUT=0`、`CLAUDE_STREAM_IDLE_TIMEOUT_MS=1800000`、`CLAUDE_CODE_RETRY_WATCHDOG=1`、`BASH_MAX_TIMEOUT_MS=3600000`、`CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS=3600000` | 7.6 |
| A3 证据先行 | `BASH_MAX_OUTPUT_LENGTH=150000`、`TASK_MAX_OUTPUT_LENGTH=160000`、`CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS=65536`、内置 WebSearch + Bash 直抓取证 | 7.5 / 7.8 |
| A4 规范硬约束 | CLAUDE.md 模板、验证闭环规程、（可选）项目级 deny/hooks | 9.2–9.4 |
| A5 大规模并行 | `workflowSizeGuideline=unrestricted`、`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`、`CLAUDE_CODE_FORK_SUBAGENT=1`、worktree、workflow 显式触发通道 | 7.10 / 9.5 |
| A6 / B5 行为一致性 | 主线全 Pro 单模型、effort 环境变量锁定 `max`、不设 fallback、升级 SOP、漂移源清单 | 3 / 7.3 / 13 / 14 |
| B1 数据可靠 | 内置 WebSearch（DeepSeek 服务端原生支持）+ `skipWebFetchPreflight`、Bash 原始页面抓取、数据铁律规程、WebFetch 抽取模型升至 Pro | 7.8 / 10.2 |
| B2/B3 口径与时点 | A 股 CLAUDE.md 模板（口径声明块、时点规程） | 10.3 |
| B4 全面性 | `reasoning effort = max` 锁定、fork/子代理并行核查、检查清单 | 7.3 / 10.4 |
| B5 持续复跑 | bypassPermissions、无人值守重试、headless 例行任务、`preferredNotifChannel=terminal_bell`、`minimumVersion=2.1.246` | 7.9 / 7.11 / 7.12 / 10.5 |
| B6 推理可审 | `alwaysThinkingEnabled`、`showThinkingSummaries`、fullscreen 渲染（压缩后全量回滚 + `Ctrl+O` 全文检索） | 7.3 / 7.12 |
| C1 视觉证据可读 | vision 子代理（`model: deepseek-v4-flash-vision-exp`）+ Read 图片直读 + ffmpeg 视频抽帧 | 7.13 / 11 |
| C2 主线不妥协 | vision-exp 零别名位、不设 `CLAUDE_CODE_SUBAGENT_MODEL` | 3 / 7.13 / 8 |
| C3 视觉可靠性 | vision 子代理系统提示（转录先行/裁剪保全/不确定性标注） | 11 |

---

# 第二部分：配置

## 5. 工作原理速览

### 5.1 网关模式

Claude Code 通过 `ANTHROPIC_BASE_URL` 将全部模型请求改发至兼容 Anthropic Messages 协议的端点；该变量仅改变"请求发往何处"，不改变"哪个模型回答"。DeepSeek 的 Anthropic 协议端点为 `https://api.deepseek.com/anthropic`。

**DeepSeek 端点的一个关键行为**：传入不支持的模型名时**不报错**，后端自动映射——`claude-opus` 开头映射到 `deepseek-v4-pro`，`claude-haiku`/`claude-sonnet` 开头映射到 `deepseek-v4-flash`，**其余不认识的名字一律静默映射到 `deepseek-v4-flash`**。这意味着模型名拼写错误的后果不是可见的报错，而是**静默降级到 Flash**——比报错更危险。本配置四个别名位显式写死 `deepseek-v4-pro`（`[1m]` 后缀由 Claude Code 在发送前剥除），部署后按第 15 章清单核对，平时以平台用量页的按模型计量分布交叉验证。

- 来源：[DeepSeek – 使用 Anthropic API（端点、自动映射规则）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)、[Claude Code – Model configuration](https://code.claude.com/docs/en/model-config)

### 5.2 别名映射与 effort 映射

模型别名（opus/sonnet/haiku/fable）由 `ANTHROPIC_DEFAULT_*_MODEL` 解析（见第 3 章矩阵）。Claude Code 的 effort 档位以 `output_config.effort` 发出（DeepSeek Anthropic 端点对 `output_config` 仅支持该字段），经 DeepSeek 端映射为模型思考强度：

| Claude Code effort | DeepSeek 实际 effort |
| --- | --- |
| `low` | `low` |
| `medium` / `high` / **`xhigh`** | `high` |
| `max` | **`max`**（DeepSeek 的最深推理档） |

**注意 `xhigh` 在 DeepSeek 侧只映射到 `high`**——这与 GLM（xhigh 与 max 同映射到最深档）相反，是两家端点最重要的机制差异，直接决定了第 7.3 节锁定 `max` 而非 `xhigh` 的选择。DeepSeek 思考模式默认打开且 effort 默认 `high`；官方对"高度复杂的任务场景"的推荐即 `max`。

- 来源：[DeepSeek – 思考模式（开关、三档强度、映射表、默认值）](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode)、[DeepSeek – 使用 Anthropic API（`output_config` 仅支持 `effort`、`thinking` 支持）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)、[DeepSeek – V4-Pro 正式版公告（复杂场景用 max）](https://api-docs.deepseek.com/zh-cn/news/news260813)

### 5.3 两层缓存（背景知识，非优化目标）

Claude Code 侧为提示缓存（v2.1.237 起在自定义 base URL 下正常工作）；DeepSeek 侧为默认开启的上下文硬盘缓存（`usage.prompt_cache_hit_tokens` 可见，"缓存前缀单元"完整匹配语义见第 2 章）。定调见第 2 章，漂移源清单见第 14 章。

- 来源：[Claude Code – Prompt caching](https://code.claude.com/docs/en/prompt-caching)、[DeepSeek – 上下文硬盘缓存](https://api-docs.deepseek.com/zh-cn/guides/kv_cache)

## 6. 配置文件位置、生效时机与验证

| 作用域 | Windows 路径 | 影响范围 |
| --- | --- | --- |
| **用户级（本配置使用）** | `%USERPROFILE%\.claude\settings.json` | 当前用户、所有项目 |
| **用户级子代理（本配置使用）** | `%USERPROFILE%\.claude\agents\vision.md` | 当前用户、所有项目 |
| 项目级（共享） | `<项目>\.claude\settings.json` | 该仓库所有协作者 |
| 项目级（本地） | `<项目>\.claude\settings.local.json` | 当前用户、仅该仓库 |
| 托管策略 | `C:\Program Files\ClaudeCode\managed-settings.json` 等 | 全机器，最高优先级 |

要点：

- 优先级：托管 > 命令行 > 项目本地 > 项目共享 > 用户级；**数组类设置（如 `permissions.allow`）跨作用域合并**而非覆盖。
- 环境变量与设置字段并存时**环境变量优先**；同一变量在 shell 与 settings 的 `env` 块并存时 **`env` 块优先**（启动时写入进程环境覆盖 shell 值）——因此一切变量集中配置于 `env` 块。**部署前须清理 shell 中残留的 DeepSeek 官方指南环境变量**（`ANTHROPIC_MODEL`、`CLAUDE_CODE_SUBAGENT_MODEL` 等——后者若残留会杀死 vision 子代理的模型路由，见第 8 章）。
- settings 文件被监视并**热加载**大部分键；`model`、`outputStyle` 等在下次启动/`/model` 时生效。
- 验证：`/status`（Setting sources 与 Model）、`/context`（窗口占用）、`/tasks`（子代理实际运行的模型与 effort，v2.1.243+）、`claude doctor`（健康检查）。

部署步骤：

1. 将仓库根目录 [`settings.json`](./settings.json) 部署至 `%USERPROFILE%\.claude\settings.json`，并将 `ANTHROPIC_AUTH_TOKEN` 的占位值替换为实际的 DeepSeek API Key（[DeepSeek 开放平台 → API keys](https://platform.deepseek.com/api_keys) 创建；平台按量计费，需先充值）。
2. 将 [`agents/vision.md`](./agents/vision.md) 部署至 `%USERPROFILE%\.claude\agents\vision.md`（vision 子代理，视觉链路的唯一通道）。
3. 关闭全部既有会话窗口后重新启动 `claude`，首次提示 "Do you want to use this API key?" 时选择 Yes。

安全提示：settings.json 中包含明文 API Key。将该文件提供给任何第三方、进行截图分享或提交至公开仓库之前，必须先移除 Key；已暴露的 Key 应立即在平台删除重建。Claude Code v2.1.246 起修复了"第三方网关 Key 被携带到 Anthropic 遥测请求"的问题（凭证只发往其所属 host），这是本配置版本门槛的组成部分（7.11）。

- 来源：[Claude Code – Settings（scopes / precedence / 热加载 / 验证）](https://code.claude.com/docs/en/settings)、[Claude Code – env-vars（Precedence）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Sub-agents（用户级 agents 目录）](https://code.claude.com/docs/en/sub-agents)、[DeepSeek – 接入 Claude Code](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)、[Claude Code – CHANGELOG v2.1.246](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

## 7. 参数逐项详解

除特别说明外，`env` 值一律为**字符串**。每项给出：名称、值、含义与理由、来源。

### 7.1 接入与认证

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `env.ANTHROPIC_BASE_URL` | `https://api.deepseek.com/anthropic` | DeepSeek 的 Anthropic 协议端点。结尾不带 `/`。 |
| `env.ANTHROPIC_AUTH_TOKEN` | 使用者的 DeepSeek API Key | 以 `Authorization: Bearer` 发送（DeepSeek 端 `x-api-key` 同样完全支持）。**系统中不得残留 `ANTHROPIC_API_KEY`**，避免双认证冲突。 |

- 来源：[DeepSeek – 接入 Claude Code（两变量的官方值）](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)、[DeepSeek – 使用 Anthropic API（HTTP Header 兼容表）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)、[Claude Code – env-vars（两变量条目）](https://code.claude.com/docs/en/env-vars)

### 7.2 模型矩阵（主线全 Pro）

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `env.ANTHROPIC_DEFAULT_OPUS_MODEL` | `deepseek-v4-pro[1m]` | 主力位。`[1m]` 使 Claude Code 按 1M 窗口管理会话（非 `claude-` 开头且含 `[1m]` 的 ID 直接按 1M 处理），发送前自动剥除后缀。官方接入指南的推荐值。 |
| `env.ANTHROPIC_DEFAULT_SONNET_MODEL` | `deepseek-v4-pro[1m]` | 覆盖一切解析到 sonnet 的路径。官方指南同样将此位设为 Pro。 |
| `env.ANTHROPIC_DEFAULT_FABLE_MODEL` | `deepseek-v4-pro[1m]` | 覆盖 `fable`/`best` 别名路径。 |
| `env.ANTHROPIC_DEFAULT_HAIKU_MODEL` | `deepseek-v4-pro` | **后台辅助功能同样采用旗舰**（第 3 章）。官方指南此位推荐 `deepseek-v4-flash`（经济考量）；本配置原则为成本不入决策，采用 Pro——WebFetch 的"小模型抽取"环节升至旗舰后信息损失进一步减小。`[1m]` 后缀在 Haiku 位无官方文档保证且后台任务极短，故用裸模型名。 |
| `model`（顶层） | `opus` | 会话启动模型显式声明，杜绝默认值漂移。 |
| `env.ANTHROPIC_DEFAULT_*_MODEL_NAME` / `_DESCRIPTION` | `DeepSeek-V4-Pro (1M)` 等 | `/model` 选择器显示名（`_NAME`/`_DESCRIPTION` 在网关模式生效）。 |
| `env.ANTHROPIC_DEFAULT_*_MODEL_SUPPORTED_CAPABILITIES` | `effort,xhigh_effort,max_effort,thinking,adaptive_thinking,interleaved_thinking`（四个位一致） | 向 Claude Code 声明自定义模型 ID 支持的能力：解锁 effort 滑杆与 max 档、思考渲染、工具间思考。`deepseek-v4-pro` 不匹配 Claude Code 的内置模型模式，不声明则这些能力可能整体关闭；官方文档明确该变量用于第三方 provider 的自定义 ID，网关场景以 `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` 为主保险（7.3），此处为无副作用的冗余保险。DeepSeek 侧真实支持情况：思考模式（默认开、可关）、effort 三档、思考中多轮工具调用——与声明一致。 |
| （**不设**）`env.CLAUDE_CODE_SUBAGENT_MODEL` | — | **本配置与官方接入指南的最重要分歧点**，理由见第 8 章表首行：设置它会以最高优先级覆盖 vision 子代理的 frontmatter 模型指定，多模态链路即被杀死。不设时常规子代理默认 `inherit` 继承主会话（= Pro 1M），效果与"显式设为 Pro"完全相同且不伤及多模态专线。 |

- 来源：[DeepSeek – 接入 Claude Code（官方推荐位）](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)、[Claude Code – Model configuration（`[1m]` 对自定义 ID 的窗口语义、`_NAME`/`_SUPPORTED_CAPABILITIES`）](https://code.claude.com/docs/en/model-config)、[Claude Code – Sub-agents（`CLAUDE_CODE_SUBAGENT_MODEL` > frontmatter 的优先级、默认 inherit）](https://code.claude.com/docs/en/sub-agents)、[DeepSeek – 思考模式](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode)

### 7.3 思考深度：锁定 reasoning effort = max（A6 / B4 / B6）

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `env.CLAUDE_CODE_EFFORT_LEVEL` | `max` | **锁定档，DeepSeek 官方接入指南的推荐值**。环境变量是 effort 的最高优先级来源（高于 `/effort` 与 `effortLevel` 设置），也是让 `max` 跨会话持久的**唯一**途径（官方：不经此变量设置时 `max` 仅对当前会话生效；`effortLevel` 设置键不接受 `max`）。为什么必须是 `max` 而非 GLM 方案的 `xhigh`：DeepSeek 的映射表中 **`xhigh` 只落到 `high`**，`max` 才是最深档——在 GLM 侧"锁 xhigh = 最深推理 + 保留 ultracode 编排"的严格占优选择，在 DeepSeek 侧不存在。锁 `max` 的已知代价：官方明确"`CLAUDE_CODE_EFFORT_LEVEL` 设为 `xhigh` 以外的档位时，ultracode 的 workflow 自动编排保持关闭"——即放弃"每任务自动编排"模式。按总原则裁决：单查询推理深度（每一次回答的正确性）优先于编排自动化（便利层）；workflow 本身并未失去——交互会话中人工输入 `ultracode` 关键词或"用 workflow"的自然语言请求仍触发单任务 workflow，且 run 按会话档位（max）执行（见 7.10、9.5）。 |
| `effortLevel`（顶层） | `xhigh` | 次优兜底层：`effortLevel` 的合法值到 `xhigh` 为止（不接受 `max`）。若上述环境变量意外缺失，会话落在 xhigh→经 DeepSeek 映射为 `high`（模型的 API 默认档），**深度有损**——因此这一层只是"缺失时不至于跌到更低档"的兜底，链路健康的判据是 `/effort` 显示被环境变量接管（第 15 章第 5 条）。 |
| `env.CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | `1` | Claude Code 默认仅对其识别的 Claude 模型 ID 发送 effort 参数；`deepseek-v4-pro` 是自定义 ID，不设此项 effort 可能根本不随请求发出（DeepSeek 端将采用其默认 `high`，而非 `max`）。官方原文："Use this when routing through an LLM gateway or third-party provider that serves models under custom identifiers." |
| `alwaysThinkingEnabled`（顶层） | `true` | 所有会话默认扩展思考。DeepSeek 思考模式本就默认打开，此项保证客户端不发出关闭思考的请求形态（Anthropic 协议 `thinking` 字段受支持，`budget_tokens` 被端点忽略）。 |
| `showThinkingSummaries`（顶层） | `true` | 思考内容在界面可见（`Ctrl+O` 展开）——B6"推理可审"的界面开关。A 股结论必须回看推理链；BMC 大改动前亦应审查其方案推演。 |

效果链路：`CLAUDE_CODE_EFFORT_LEVEL=max` + `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT=1` → 请求携带 `output_config.effort=max` → DeepSeek 端 `max→max` → V4-Pro 最深推理。vision 子代理同样受益：子代理 effort 默认继承会话（= max），vision-exp 支持相同三档。

- 来源：[DeepSeek – 接入 Claude Code（`CLAUDE_CODE_EFFORT_LEVEL=max` 官方推荐）](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)、[DeepSeek – 思考模式（映射表：xhigh→high、max→max；默认 high）](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode)、[Claude Code – Model configuration（max 持久化仅经环境变量、非 xhigh 锁定下 ultracode 编排关闭、优先级顺序）](https://code.claude.com/docs/en/model-config)、[Claude Code – Settings reference（`effortLevel` 枚举至 xhigh、`alwaysThinkingEnabled`、`showThinkingSummaries`）](https://code.claude.com/docs/en/settings-reference)、[Claude Code – env-vars（`CLAUDE_CODE_ALWAYS_ENABLE_EFFORT`）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Workflows（关键词触发独立于会话档位）](https://code.claude.com/docs/en/workflows)

### 7.4 上下文：真 1M 与窗口预算的互补设计（A1）

DeepSeek V4 全系上下文 1M、最大输出 384K（含思维链）。本配置的三个窗口参数是一组**互补设计**：

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `env.CLAUDE_CODE_AUTO_COMPACT_WINDOW` | `786432` | 自动压缩的容量基准（合法域 100000–1000000），**DeepSeek 官方接入指南的推荐值** = 0.75 × 1048576。对话占用逼近该值时触发压缩，其与真实窗口之间的 262144（256K）即为单轮输出（含思维链）的预留空间。**提前压缩 = 证据被摘要替换**（A1/A3 的首要威胁），但压缩基准也不能吃满窗口——`max` 档的思维链计入输出 token，输出预算必须宽裕，此为两者的平衡点且有官方推荐值背书。 |
| `env.CLAUDE_CODE_MAX_OUTPUT_TOKENS` | `262144` | 单请求输出上限，与压缩基准精确互补（786432 + 262144 = 1048576 = 1M）。**必须显式设置**：Claude Code 对不识别的模型 ID 默认仅给 32000——max 档长思维链 + 整文件重写场景下，32K 输出预算是最隐蔽的截断源。不取满 DeepSeek 的 384K 硬上限（393216）：官方明示"提高该值会减少触发 auto-compaction 前的可用上下文"，384K 会把压缩基准压至 ~640K，牺牲 146K 输入空间换取几乎用不到的输出余量，非帕累托改进。 |
| `env.CLAUDE_CODE_MAX_CONTEXT_TOKENS` | `1048576` | 为**裸模型名**声明真实窗口。`deepseek-v4-pro[1m]` 经 `[1m]` 通道直接按 1M 管理（该变量对含 `[1m]` 的 ID 不单独起作用）；此变量服务于两处裸名：Haiku 位 `deepseek-v4-pro` 与 vision 子代理的 `deepseek-v4-flash-vision-exp`（非 `claude-` 开头且不含 `[1m]` 的 ID，变量直接生效）。 |
| `env.CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | `1` | **多仓库工作区关键项**：默认情况下 `--add-dir`/`/add-dir` 挂载的目录**不加载**其 CLAUDE.md/规则文件；置 1 后各仓库的 `CLAUDE.md`、`.claude/rules/*.md`、`CLAUDE.local.md` 全部载入。BMC 场景将 u-boot、kernel、openbmc、webui 各自的工程约定同时载入上下文（A1/A4）。 |
| `env.CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | `60` | Glob 文件发现超时从默认 20 秒提高到 60 秒（官方在 WSL 上的默认值）。Yocto 全量源码树等超大目录的检索不因超时遗漏文件（A1/A3）。 |

**不设** `CLAUDE_CODE_DISABLE_1M_CONTEXT`（会使 `[1m]` ID 按 200K 处理）与 `DISABLE_AUTO_COMPACT`/`DISABLE_COMPACT`（窗口占满时压缩是避免请求直接失败的最后防线；配合第 9/10 章"关键证据落盘"规程，压缩造成的损失可控）。

- 来源：[DeepSeek – 模型 & 价格（1M 上下文 / 最大 384K 输出）](https://api-docs.deepseek.com/zh-cn/quick_start/pricing)、[DeepSeek – 接入 Claude Code（`CLAUDE_CODE_AUTO_COMPACT_WINDOW=786432` 官方推荐）](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)、[Claude Code – env-vars（`CLAUDE_CODE_AUTO_COMPACT_WINDOW` 域与语义、`CLAUDE_CODE_MAX_OUTPUT_TOKENS` 未识别 ID 默认 32000 与压缩空间挤占、`CLAUDE_CODE_MAX_CONTEXT_TOKENS`）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Model configuration（`[1m]` 与 `MAX_CONTEXT_TOKENS` 对三类 ID 的分别语义）](https://code.claude.com/docs/en/model-config)

### 7.5 证据不截断（A3 / B1）

| 参数 | 值 | 默认 | 含义与理由 |
| --- | --- | --- | --- |
| `env.CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | `65536` | 内置较小值 | 单次文件读取 token 上限。长驱动源文件、长财报文本尽量整读入上下文而非分页。（可继续上调，但单文件占用窗口比例过大反而挤压推理空间，65536≈250KB 文本是证据完整性与推理余量的平衡点。） |
| `env.BASH_MAX_OUTPUT_LENGTH` | `150000` | 30000 | 命令输出直接进入上下文的字符上限，**150000 为官方硬上限**。构建日志、回测输出超限时转存文件并提供路径，模型可继续检索。 |
| `env.TASK_MAX_OUTPUT_LENGTH` | `160000` | 32000 | 子代理回传上限，**160000 为官方最大值**。子代理的调研结论是证据链核心载体——vision 子代理的长转录（整页报错、表格）同样经此通道回传主线，设置为最大值。 |
| `env.MAX_MCP_OUTPUT_TOKENS` | `50000` | 25000 | MCP 响应 token 上限。本配置无官方 MCP 依赖（DeepSeek 无套餐 MCP，检索走内置 WebSearch），该值为**自备 MCP 的预置余量**——A 股场景若接入本地行情数据 MCP（10.4），长行情序列不被默认值截断。 |
| `env.BASH_DEFAULT_TIMEOUT_MS` | `600000` | 120000 | 命令默认超时 10 分钟。 |
| `env.BASH_MAX_TIMEOUT_MS` | `3600000` | 600000 | 模型可请求的命令超时上限 1 小时（bitbake 单包构建、长回测）。超时不等于被终止：到时命令转入后台继续执行。 |

- 来源：[Claude Code – env-vars（各变量条目）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Tools reference（Bash 输出/超时/自动转后台行为）](https://code.claude.com/docs/en/tools-reference)

### 7.6 传输可靠性（A2 / B5）

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `env.API_TIMEOUT_MS` | `3000000` | 单请求总超时 50 分钟（默认仅 10 分钟）。max 档深思考 + 1M 预填充的最坏情况兜底。DeepSeek 侧配套机制：请求等待期间持续返回保活内容（非流式空行 / 流式 SSE keep-alive 注释），若 **10 分钟后仍未开始推理则服务器主动关闭连接**——被关闭的请求进入下方重试层，客户端超时须长于服务端行为，避免两头抢跑。 |
| `env.API_FORCE_IDLE_TIMEOUT` | `0` | 关闭"流式 5 分钟无字节即中止"的定时器（非 Anthropic 直连时默认开启）。官方明确 `0` 用于块间停顿超过 5 分钟的网关/模型场景——深思考停顿被误中断正是"中途截断"的典型来源。DeepSeek 的 keep-alive 注释通常会持续产生字节，此项是对"注释间隔异常拉长"的兜底。 |
| `env.CLAUDE_STREAM_IDLE_TIMEOUT_MS` | `1800000` | 流式空闲看门狗放宽到 30 分钟（显式设置的官方下限为 5 分钟）。保留最终兜底：确认失效的连接在 30 分钟后仍会被切断并进入重试，不会无限挂起。 |
| `env.CLAUDE_ENABLE_BYTE_WATCHDOG` | `1` | **字节级**流式看门狗。与事件级看门狗互补：事件级监测"是否解析出 SSE 事件"，字节级监测"线路上是否存在任何字节"。网关连接下字节级默认**关闭**（官方：仅 Anthropic 直连等默认开启），显式开启补上该防护层；其阈值复用 `CLAUDE_STREAM_IDLE_TIMEOUT_MS`（30 分钟），不会误中断深思考停顿。 |
| `env.CLAUDE_CODE_RETRY_WATCHDOG` | `1` | 无人值守级重试：429/529 容量类错误无限重试（按限额重置时间或最长 5 分钟退避），其余瞬时错误默认重试提高到 300 次（约 3 小时退避）。v2.1.239 起"支出上限/额度耗尽"类 429 改为**立即失败**——对无人值守是正确行为。注意 DeepSeek 的余额耗尽是 **402** 而非 429：重试不解决 402，处置见第 12 章（充值是唯一路径）。官方推荐用于 unattended sessions——B5 盘后无人值守的基石。 |
| `env.CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` | `1` | 流式中断时禁止回退为非流式重发——官方指出该回退经代理/网关时可能**重复执行工具**。对金融数据写入、git 操作，重复执行属于正确性事故；禁用后错误统一进入上述重试层。 |
| `env.MCP_TIMEOUT` / `env.MCP_TOOL_TIMEOUT` / `env.CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT` | `120000` / `1800000` / `600000` | 本配置无官方 MCP，此三项为**自备 MCP 的预置基线**（本地行情数据 MCP 等，见 10.4）：启动连接 2 分钟（npx 首启拉包）、单请求/执行上限 30 分钟（HTTP 型 MCP 默认单请求仅 60 秒，长取数必须放宽）、空闲判定 10 分钟。无 MCP 部署时三项不产生任何行为。 |
| `env.CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS` | `0` | 关闭"MCP 调用超过 2 分钟自动转后台"（默认 120000）。理由是**证据顺序**：取数调用被静默转入后台后，模型可能在数据到达前继续推进论证——违背 B1"先有证据、后有结论"的时序。关闭后取数调用同步完成、结果按序进入推理链。BMC 的长构建走 Bash 后台任务，不受此项影响。 |

- 来源：[Claude Code – env-vars（各变量条目，含 `CLAUDE_ENABLE_BYTE_WATCHDOG` 复用 `CLAUDE_STREAM_IDLE_TIMEOUT_MS`、`MCP_TOOL_TIMEOUT` 60 秒单请求条款、RETRY_WATCHDOG 的 v2.1.239 语义）](https://code.claude.com/docs/en/env-vars)、[DeepSeek – 限速与隔离（请求保活机制、10 分钟未开始推理关闭连接）](https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit)、[DeepSeek – 错误码（402 余额不足）](https://api-docs.deepseek.com/zh-cn/quick_start/error_codes)

### 7.7 提示词纯净（附带缓存收益）

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `env.CLAUDE_CODE_ATTRIBUTION_HEADER` | `0` | 移除系统提示词开头的客户端版本/指纹归因块：减少与任务无关的提示噪音；官方另注明此项提升网关场景缓存命中——按第 2 章定调，属附带收益。 |

- 来源：[Claude Code – env-vars（`CLAUDE_CODE_ATTRIBUTION_HEADER`）](https://code.claude.com/docs/en/env-vars)

### 7.8 网络适配：内置 WebSearch 原生可用（B1）

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| （无需配置）内置 `WebSearch` | 默认可用 | **与 GLM 方案的根本性差异**：DeepSeek API **原生支持 Claude Code 的 Web Search 功能**——模型判断需要检索时调用 WebSearch 工具，搜索由 **DeepSeek 提供的 API 在服务端执行**（Anthropic 兼容表中 `server_tool_use` / `web_search_tool_result` 均支持），对获取的搜索内容进行总结会产生额外模型 token 费用。GLM 场景"内置 WebSearch 不可用、必须部署 Web Search MCP"的结论**不适用于本配置**；检索链路零 MCP 依赖。会话内 WebSearch 上限默认 200 次（`CLAUDE_CODE_MAX_WEBSEARCH_USES`，见第 8 章保持默认的理由）。 |
| `env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `1` | 关闭全部非必要外联：后台自动更新、遥测、错误上报、`/feedback`、发行说明、模型发现刷新、可用性检查。在无法访问 Anthropic 的日常环境中，这些请求仅产生超时报错并拖慢会话。手动 `claude update` 不受影响。 |
| `env.DISABLE_ERROR_REPORTING` / `env.DISABLE_TELEMETRY` | `1` / `1` | 显式双重保障；`DISABLE_TELEMETRY` 使特性开关采用代码默认值，行为更确定。 |
| `skipWebFetchPreflight`（顶层） | `true` | **关键项**：WebFetch 默认对每个 URL 先向 `api.anthropic.com` 发起域名安全预检；本环境无法访问 Anthropic，不跳过则 WebFetch 整体不可用（直接破坏 B1 数据链路）。官方明确此项用于 "environments that block traffic to Anthropic"。 |
| `autoUpdatesChannel`（顶层） | `latest` | 手动升级时跟随最新通道，配合主动持续跟踪升级的策略。 |

补充说明：WebFetch 为"抽取式"设计（页面内容经 Haiku 位模型抽取后进入上下文，本配置已将抽取模型升至 Pro，但仍是单遍有损抽取）——取证级网页读取的规程见 10.2（Bash 直抓原始页面）。claude.ai 登录态功能（Artifacts、Remote Control、Routines 等）在纯 API Token 模式下不可用，属预期行为。

- 来源：[DeepSeek – 接入 Claude Code（"DeepSeek API 原生支持 Claude Code 中的 Web Search 功能"及计费说明）](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)、[DeepSeek – 使用 Anthropic API（`server_tool_use`/`web_search_tool_result` 支持）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)、[Claude Code – env-vars（`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`CLAUDE_CODE_MAX_WEBSEARCH_USES`）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Settings（`skipWebFetchPreflight`、`autoUpdatesChannel`）](https://code.claude.com/docs/en/settings)、[Claude Code – Tools reference（WebSearch/WebFetch tool behavior）](https://code.claude.com/docs/en/tools-reference)

### 7.9 Windows 专项与自动化权限

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `env.CLAUDE_CODE_USE_POWERSHELL_TOOL` | `1` | 启用原生 PowerShell 工具（安装了 Git Bash 的机器上该工具默认"逐步放量"，显式置 1 立即启用）。自动优先选择 `pwsh.exe`，回退 `powershell.exe`，进程级 `-ExecutionPolicy Bypass` 保证 `.ps1` 可执行。Bash 工具保留用于 POSIX 脚本。 |
| `defaultShell`（顶层） | `powershell` | 输入框 `!` 手动执行的命令走 PowerShell。 |
| `permissions.defaultMode` | `bypassPermissions` | 默认全自动。残留提示仅包括：显式 ask 规则（未设置）、标记 `requiresUserInteraction` 的 MCP 工具、`rm -rf /` 类断路器。 |
| `skipDangerousModePermissionPrompt`（顶层） | `true` | 跳过进入 bypass 模式的确认弹窗（用户级设置有效，项目级会被忽略）。 |
| `permissions.allow` | 16 条 | Shift+Tab 切换到 manual/acceptEdits/plan 模式时的免批清单，见下文。 |
| `permissions.deny` | `[]` | deny 在**一切模式（含 bypass）**下生效，与全自动目标冲突，故保持为空。涉及真实资金的实盘接口防线见 10.6（项目级窄 deny 是官方提供的唯一硬闸）。 |

允许列表设计（工具名以 [tools reference](https://code.claude.com/docs/en/tools-reference) 的权威名为准）：

- `Bash`、`PowerShell`：任意命令；`Read`、`Glob`、`Grep`、`LSP`：读取与搜索（放行后**工作目录之外**的读取同样免批——跨盘符数据目录、图片素材目录等 Windows 常见场景，vision 子代理读图直接受益）；`Edit`、`Write`、`NotebookEdit`：全部文件修改；`WebFetch`、`WebSearch`：网页抽取与检索（WebSearch 由 DeepSeek 服务端执行，见 7.8）；`Agent`、`Skill`、`Monitor`：子代理（含 vision）、技能、后台监视；`EnterWorktree`、`Workflow`：worktree 进入与动态工作流编排。
- 与 GLM 配置的差异：无 `mcp__*` 条目——本配置零 MCP 依赖。自备 MCP 部署后按 10.4 的规程将 `mcp__<server>` 规则加入**项目级** settings，不进全局。
- **allow 清单的第三重身份**：workflow 调度的子代理**强制运行在 `acceptEdits` 模式并继承此 allowlist，不继承主会话的 bypass**（官方原文："The subagents the workflow spawns always run in `acceptEdits` mode and inherit your tool allowlist, regardless of your session's mode"）。因此即便常驻 bypass，这 16 条也并非冗余——它们是 workflow 大规模并行时的免批通行证；headless 模式下不在清单内的调用会被直接拒绝。
- **有意不放行** `ExitPlanMode`：计划审批保留人工闸门——BMC 级大改动先出计划、经确认后执行，是 A4 的低成本高收益环节。
- Shift+Tab 循环为 `manual → acceptEdits → plan → bypassPermissions`（bypass 因 defaultMode 声明而进入循环）；auto 模式依赖 Anthropic 侧分类器，DeepSeek 网关下不可用、不会出现在循环中。

- 来源：[Claude Code – Tools reference（PowerShell tool、权威工具名）](https://code.claude.com/docs/en/tools-reference)、[Claude Code – Permissions（规则语法、deny 全模式生效、断路器）](https://code.claude.com/docs/en/permissions)、[Claude Code – Permission modes（模式循环与 bypass 进入循环条件、auto 可用性要求）](https://code.claude.com/docs/en/permission-modes)、[Claude Code – Workflows（workflow 子代理 acceptEdits + allowlist 继承）](https://code.claude.com/docs/en/workflows)

### 7.10 大规模并行与长任务编排（A5）

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `workflowSizeGuideline`（顶层） | `unrestricted` | 动态 workflow 的规模引导。**默认 `medium` 会引导 Claude 只写 <15 个代理的 workflow**——直接压制 A5 全仓审计、B4 多角度核查所需的规模；`unrestricted` 即"无引导、按任务定规模"，与 workflow 单 run 1000 代理的硬上限对齐。官方明确 guideline 只是发给 Claude 的建议而非强制上限。 |
| `env.CLAUDE_CODE_ENABLE_TODO_TOOLS` | `1` | 任务清单工具（TaskCreate/TaskGet/TaskUpdate/TaskList）显式锁定。v2.1.233 起较新模型默认**不再提供**这些工具（官方以模型判定为准，自定义 ID 的判定结果不受本方控制）；BMC 级长任务、批量迁移依赖任务清单做进度跟踪与断点续查，显式置 1 保证任何判定变化下工具都在。 |
| `env.CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS` | `3600000` | 后台子代理"失速"判定从默认 10 分钟放宽到 1 小时——子代理内部单条长构建/长回测命令期间可能没有流式进度事件，默认值会将其误判为失败（A2）。超过 1 小时的构建应改用 `run_in_background` 的后台命令而非阻塞式子代理。 |
| `env.CLAUDE_CODE_FORK_SUBAGENT` | `1` | 启用 fork 型子代理：**继承完整对话上下文**开分支，而非空白起步。金融场景用它并行核查对立假设（多/空论证各 fork 一支，证据不丢失）；工程场景用它在不污染主线的情况下试探激进方案。**v2.1.232 起 fork 在交互会话默认启用，但 `claude -p`/SDK 仍默认关闭**——本配置的无人值守轨道（10.5）经 `-p` 启动，置 1 是 headless 下 fork 可用的必要开关。 |
| `env.CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` | `0` | headless（`claude -p`）模式默认仅等待后台子代理与 workflow **10 分钟**，超时后"remaining background tasks are terminated and the process exits"——无人值守的深度并行任务会在 10 分钟被终止，这是 B5 场景最隐蔽的截断源。官方明确 `0` = 无限等待。交互式会话不受此变量影响。 |
| `env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | `1` | 启用实验性 agent teams（主管+队友多实例协作）。v2.1.234 起队友恒用 leader 的模型（官方已移除 Default teammate model 选项）——队友必然落在 V4-Pro，主线单旗舰不因多实例协作而漂移。实验性特性，异常时删除此行即可。 |

workflow 在本配置下的可用边界（与 7.3 的锁 max 决策联动）：

- **可用**：交互会话中人工输入 `ultracode` 关键词或"用 workflow / run a workflow"的自然语言请求，触发**单任务 workflow**——官方明确该触发"不改变会话的 effort 档位"，run 内请求仍按锁定的 `max` 执行。规模引导已放开（unrestricted）。
- **不可用**：`/effort ultracode` 与 `claude --effort ultracode` 的"每任务自动编排"模式——环境变量锁定 `max`（非 xhigh）时官方语义为编排保持关闭，选择 ultracode 会提示被环境变量覆盖。这是 7.3 论证过的知情取舍：DeepSeek 映射下 ultracode 模式的请求档位是 xhigh（→ 仅 high），"自动编排 + 次深推理"劣于"显式触发 + 最深推理"。
- 无人值守场景的 workflow 替代方案见 10.5（`-p` 提示词不触发关键词是官方防注入设计，与档位无关）。

子代理规模的现行约束：**并发**运行数（`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`，默认 20）、**嵌套深度**（默认 3）与 workflow 单 run 1000 代理/16 并发；单会话总量上限已于 v2.1.224 移除。对全仓审计这意味着"超出并发的 spawn 排队等待，而非被拒绝"。两个并发约束保持默认的理由见第 8 章（调高只会把 429 打向 DeepSeek 的并发限额）。

- 来源：[Claude Code – Workflows（size guideline、单 run 上限、关键词触发语义、`--effort ultracode`）](https://code.claude.com/docs/en/workflows)、[Claude Code – Model configuration（非 xhigh 锁定下 ultracode 编排关闭）](https://code.claude.com/docs/en/model-config)、[Claude Code – env-vars（`CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS`、`CLAUDE_CODE_FORK_SUBAGENT`、`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`、`CLAUDE_CODE_ENABLE_TODO_TOOLS`、`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`、并发/深度变量）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Settings reference（`workflowSizeGuideline`）](https://code.claude.com/docs/en/settings-reference)

### 7.11 留痕与其他

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `cleanupPeriodDays` | `365` | 会话转录保留一年（默认 30 天清理）。分析会话 = 审计底稿（B5）；配合 `/resume` 回溯。可按需继续加大。 |
| `attribution.commit`/`attribution.pr` | `""`/`""` | 关闭提交/PR 中的 Claude 署名（新版 `attribution` 键取代已废弃的 `includeCoAuthoredBy`）。 |
| `language` | `chinese-simplified` | 默认中文回复（亦影响自动会话标题语言）。 |
| `$schema` | schemastore 地址 | 编辑器内自动补全与校验。 |
| `minimumVersion`（顶层） | `2.1.246` | 低于该版本不运行：把 17.2 版本门槛表从"备查"升级为**硬执行**。锚点选择 v2.1.246 的理由：该版本修复了"第三方网关（`ANTHROPIC_BASE_URL`）配置的 API Key 被携带到发往 Anthropic 的遥测/计量请求"的凭证泄漏问题（凭证只发往其所属 host）——对 DeepSeek Key 属安全必需；同时覆盖 v2.1.237 的网关提示缓存修复与 v2.1.243 的 `/tasks` 子代理模型显示（vision 路由的观测手段）。配合 `autoUpdatesChannel: "latest"` 防止切换发布通道时降级到门槛以下（旧版会静默回退多个关键变量到默认行为）。 |

- 来源：[Claude Code – Settings（`cleanupPeriodDays`、Attribution settings、`language`、`$schema`、`minimumVersion`）](https://code.claude.com/docs/en/settings)、[Claude Code – CHANGELOG（v2.1.246 凭证隔离、v2.1.243 `/tasks` 模型显示、v2.1.237 网关缓存）](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

### 7.12 终端渲染（fullscreen）与完成通知

| 参数 | 值 | 含义与理由 |
| --- | --- | --- |
| `tui`（顶层） | `fullscreen` | 全屏 alt-screen 渲染器（`/tui fullscreen` / `/tui default` 随时切换，会话完整保留）。对本配置三项实质收益：① **内存恒定**——只渲染可见消息，1M 窗口超长会话不随转录增长吃内存；② **压缩后全量回滚**——auto-compact 之后 `Ctrl+Home` 仍可滚回会话开头查看原始证据（官方明确 fullscreen scrollback 跨多次压缩保留全部早期消息），直接缓解 9.4 落盘规程要对冲的"压缩丢证据"压力；③ **大输出可交互**——折叠的工具输出点击展开（`BASH_MAX_OUTPUT_LENGTH=150000` 的大段结果可优雅回看）、`Ctrl+O` 进入 transcript 模式后 `/` 全文检索、`{`/`}` 逐 prompt 跳转、`v` 甩进外部编辑器。 |
| `env.CLAUDE_CODE_ALT_SCREEN_FULL_REPAINT` | `1` | Windows 专属配套：Windows Terminal 等 ConPTY 主机在增量重绘下会残留错位碎片（官方排障条目"Stale or misplaced text on screen"），修法即全帧重绘。官方已对后台会话与 agent view 在 Windows 上自动启用，**交互式全屏会话需显式设置**。若全屏滚动出现整块空白，可再叠加 `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL=1`。 |
| `preferredNotifChannel`（顶层） | `terminal_bell` | 完成通知闭环。官方默认仅 iTerm2/Ghostty/Kitty 发桌面通知，**Windows Terminal 默认零通知**——无人值守长任务（10.5 双轨）完成后终端静默无声。设为 `terminal_bell` 后，任务完成或等待输入时终端响铃；配合 Windows Terminal 设置 `bellStyle`（声音/任务栏闪烁/窗口注意）形成完整提醒链。B5"持续复跑"的人机协议：听到铃声 = 产物已落盘待验收。 |

- 来源：[Claude Code – Fullscreen rendering（收益、ConPTY 全帧重绘、transcript 检索、压缩后回滚）](https://code.claude.com/docs/en/fullscreen)、[Claude Code – Terminal configuration（terminal bell / `preferredNotifChannel`）](https://code.claude.com/docs/en/terminal-config)、[Claude Code – env-vars（`CLAUDE_CODE_ALT_SCREEN_FULL_REPAINT`、`CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL`）](https://code.claude.com/docs/en/env-vars)

### 7.13 多模态子代理：vision 路由架构（C1 / C2）

多模态不是一个参数，而是一条**路由**。本配置的实现（部署物为 [`agents/vision.md`](./agents/vision.md)，用法与作业规程见第 11 章）：

| 环节 | 机制 | 依据 |
| --- | --- | --- |
| 模型指定 | 子代理 frontmatter `model: deepseek-v4-flash-vision-exp`。官方允许 frontmatter 直接写完整模型 ID（"a full model ID… Accepts the same values as the `--model` flag"），无需占用任何别名位 | [Sub-agents – Choose a model](https://code.claude.com/docs/en/sub-agents) |
| 路由优先级 | 子代理模型解析顺序：`CLAUDE_CODE_SUBAGENT_MODEL` 环境变量 > 调用时 model 参数 > **frontmatter** > 继承主会话。本配置不设前者，frontmatter 稳定命中 | 同上 |
| 多模态进入 | 一切视觉证据收敛为 **image 内容块**这一条通道：图片——子代理 Read 直读（"returned as visual content that Claude can see"）；视频——子代理 ffmpeg 抽帧为图片再读（官方无视频输入，抽帧即对视频内容的最大化利用）；扫描版 PDF/文档——按图片逐页读（官方 document 块为"不支持"，页转图是唯一合规路径；数字原生 PDF 的文字层则提取文本走主线 Pro）。DeepSeek Anthropic 端点对 image 块的支持为**官方兼容表明确背书**（`source.type` 可为 base64（jpeg/png/gif/webp）/url/file）——与网关未声明环节不同，无需额外验证；OpenAI 格式的 `detail` 参数（low=512×512 提速省 token / original=保留原图）在 Anthropic 格式无对应物，Claude Code 路径不可控，精度控制由裁剪规程承担 | [Tools reference – Read](https://code.claude.com/docs/en/tools-reference)、[DeepSeek – 图像理解](https://api-docs.deepseek.com/zh-cn/guides/vision)、[DeepSeek – 使用 Anthropic API（image 块兼容表）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api) |
| 客户端压缩 | Claude Code 发送前对大图缩放重压缩；缩放后仍超 500KB 的图片降质重编码为 JPEG。官方建议错过像素级细节时先裁剪感兴趣区域再读 | [Tools reference – Read（Images）](https://code.claude.com/docs/en/tools-reference) |
| 服务端换算 | 图片按尺寸换算 token：小于约 384×384 的图放大，更大的图缩至约 800×800 当量，**单图至多 384 token**；与文本 token 一并按 Flash 价计费 | [DeepSeek – 图像理解（Token 用量）](https://api-docs.deepseek.com/zh-cn/guides/vision) |
| 深度与窗口 | 子代理 effort 默认继承会话（= max，vision-exp 支持同三档）；thinking 配置继承（v2.1.198+）；窗口由 `CLAUDE_CODE_MAX_CONTEXT_TOKENS=1048576` 声明（裸名 ID 直接生效）；回传上限 `TASK_MAX_OUTPUT_LENGTH=160000` | [Sub-agents（effort/thinking 继承）](https://code.claude.com/docs/en/sub-agents)、7.4 / 7.5 |
| 观测 | `/tasks` 与代理详情对话框显示每个子代理实际运行的模型与 effort（v2.1.243+）——验证 vision 走的是 vision-exp 的直接手段 | [CHANGELOG v2.1.243](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |

**为什么不是别的方案**（设计决策记录）：

- ~~主会话直接吃图~~：主线是 V4-Pro，DeepSeek 官方明确"仅视觉模型接受图片，其他模型返回 400（This model does not support image）"。把主线换成 vision-exp 则违反 C2（纯文本能力仅 Flash 级）。
- ~~Haiku/Sonnet 位挂 vision-exp + 子代理走别名~~：可行但有代价——别名位会把视觉模型暴露进 `/model` 选单与后台任务路径，误操作即主线降级（违反第 3 章"选单无备选"原则）。frontmatter 直写模型 ID 零暴露。
- ~~Vision MCP~~：GLM 的方案（其套餐含官方 Vision MCP）。DeepSeek 无官方 Vision MCP，自建 MCP 引入自维护组件与第三方依赖，违背"仅官方机制"的可靠性要求；且子代理方案在深度（max 思考读图）、上下文（1M）、权限继承上全面优于工具式调用。
- ~~`/model` 手动切换读图~~：主线换模型属最高级别行为漂移（第 14 章），且切回后图片仍在上下文中、后续每轮请求都会再次发往不支持图片的 Pro——结构性错误。

**主会话纪律（必须写入各项目 CLAUDE.md，模板见 9.2/10.3）**：图片一律**不粘贴进对话**（粘贴的图片会随请求发给 Pro → 400）；图片以文件形式放入工作区，点名 vision 子代理分析（自动委派或 `@vision`）。

- 来源：见表内逐行标注

## 8. 有意不设置的参数及原因

| 参数 | 决定 | 原因 |
| --- | --- | --- |
| `CLAUDE_CODE_SUBAGENT_MODEL` | **绝不设置**（官方接入指南设为 flash，本配置显式偏离） | 子代理模型解析的最高优先级来源，设置后**覆盖 vision 子代理的 frontmatter**——视觉链路被强制拉回文本模型，读图必然 400（C1 归零）。官方指南设 flash 出于经济性（本配置不采）；追求"子代理全 Pro"也无需它：不设时默认 `inherit`，常规子代理天然继承主会话的 Pro (1M)。来源：[Sub-agents – model 解析顺序](https://code.claude.com/docs/en/sub-agents) |
| `fallbackModel` | **绝不设置** | 过载时自动切换模型 = 静默降级出活，违反 A6/B1。策略：`CLAUDE_CODE_RETRY_WATCHDOG` 持续重试同一模型。来源：[Model configuration – Fallback model chains](https://code.claude.com/docs/en/model-config) |
| `ANTHROPIC_MODEL` / `ANTHROPIC_DEFAULT_MODEL` | 不设置 | 官方接入脚本用 `ANTHROPIC_MODEL` 指定会话模型；本配置以顶层 `model: "opus"` + 四别名位达成同一效果且语义更清晰（`ANTHROPIC_MODEL` 是覆盖一切文件设置的最强来源，保留它反而是一个不可见的漂移源；`ANTHROPIC_DEFAULT_MODEL` 仅在无文件设置时生效，无增益）。部署时须清理 shell 中按官方指南 export 过的残留值（第 6 章）。来源：[Claude Code – Settings（环境变量与 model 键的优先关系）](https://code.claude.com/docs/en/settings)、[env-vars](https://code.claude.com/docs/en/env-vars) |
| `CLAUDE_CODE_EXTRA_BODY`（如注入 `temperature`/`top_p`） | 不设置 | DeepSeek 官方直接封死了这条路：**思考模式不支持 `temperature`、`top_p`、`presence_penalty`、`frequency_penalty`——设置不报错但不生效**。本配置全程思考模式，注入采样参数是纯粹的无效负载；数值的确定性由"工具算数"保证（10.2 铁律：所有计算落到代码执行）。来源：[DeepSeek – 思考模式（输入输出参数）](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode) |
| `MAX_THINKING_TOKENS` | 不设置 | 设 0 会关闭思考；非零值属"固定思考预算"机制——而 DeepSeek Anthropic 端点本就**忽略 `thinking.budget_tokens`**，思考深度的唯一有效控制面是 effort（已锁 max，7.3）。来源：[DeepSeek – 使用 Anthropic API（budget_tokens 忽略）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)、[Claude Code – env-vars](https://code.claude.com/docs/en/env-vars) |
| `DISABLE_PROMPT_CACHING` / `promptCacheTtl` / `ENABLE_PROMPT_CACHING_1H` | 绝不设置 / 不设置 | 缓存是无损机制（第 2 章），关闭无任何收益；TTL 类选项属 Anthropic 官方缓存计费机制，DeepSeek 硬盘缓存自动管理（几小时到几天自动清理）、`cache_control` 被忽略，客户端 TTL 声明无对应物。来源：[Prompt caching](https://code.claude.com/docs/en/prompt-caching)、[DeepSeek – 上下文硬盘缓存](https://api-docs.deepseek.com/zh-cn/guides/kv_cache) |
| `modelPicker` | 不设置 | v2.1.242+ 可定制 `/model` 选单。本配置四个别名位已全部钉死 V4-Pro，选单本就无备选项；再叠加 `replaceBuiltInOptions` 只是多一层要维护的机制，无增量收益。若日后需要在选单中固定展示多模型（不推荐），此键是官方正道。来源：[Settings reference – modelPicker](https://code.claude.com/docs/en/settings-reference) |
| `ENABLE_TOOL_SEARCH` | 保持默认 | 非第一方 host 默认预载全部工具（tool search 需端点支持 `tool_reference`，DeepSeek 未声明支持，强制开启会导致请求失败）。来源：[env-vars（`ENABLE_TOOL_SEARCH`）](https://code.claude.com/docs/en/env-vars) |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | 默认不设 | DeepSeek 对 `/messages` 的 `anthropic-beta` 头直接**忽略**（兼容表），不存在报错场景；仅当未来端点行为变化时临时置 1。来源：[DeepSeek – 使用 Anthropic API（HTTP Header）](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api) |
| `DISABLE_INTERLEAVED_THINKING` | 不设置 | DeepSeek 思考模式原生支持"多轮思考与工具调用交替"（携带 tools 时思维链跨请求回传并拼接）——"每个工具结果均被推理消化"正是 A3/B4 的要求。来源：[DeepSeek – 思考模式（工具调用）](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode) |
| `DISABLE_AUTO_COMPACT` / `DISABLE_COMPACT` | 不设置 | 窗口占满时压缩是避免请求直接失败的最后防线；损失由"关键证据落盘"规程（9.4/10.5）对冲。来源：[env-vars](https://code.claude.com/docs/en/env-vars) |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 绝不设置 | 会使 `[1m]` ID 按 200K 窗口处理。来源：[Model configuration](https://code.claude.com/docs/en/model-config) |
| `ANTHROPIC_API_KEY` | 不设置 | 本配置使用 `ANTHROPIC_AUTH_TOKEN`；系统中残留的 `ANTHROPIC_API_KEY` 必须删除。来源：[env-vars](https://code.claude.com/docs/en/env-vars) |
| `CLAUDE_CODE_MAX_WEBSEARCH_USES` | 保持默认 200 | 会话内 WebSearch 上限。200 次/会话对单任务检索绰绰有余；深挖类任务的检索分散在多个子代理中，各会话独立计数。异常触顶多半意味着检索策略退化（同义反复查询），应修正提示词而非放大上限。来源：[env-vars](https://code.claude.com/docs/en/env-vars) |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` / `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` / `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` | 保持默认 10 / 20 / 3 | 并发受 DeepSeek 账号侧并发限额约束（Pro 500 / Flash 与 Vision 各 2500，账号粒度）：客户端并发堆高只会把 429 打向端点、进入重试循环。大规模审计的正确形态是"总量排队"而非"并发堆高"；确有更高并发需求走官方扩容工单（免费，见第 12 章）。来源：[env-vars](https://code.claude.com/docs/en/env-vars)、[DeepSeek – 限速与隔离](https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit) |
| `sandbox.*` | 不设置 | 沙箱仅支持 macOS/Linux/WSL2；Windows 原生 PowerShell 工具不支持沙箱。来源：[Settings – Sandbox settings](https://code.claude.com/docs/en/settings) |
| `fastMode` | 不设置 | Anthropic 原生 Opus 的加速计费特性，DeepSeek 网关无对应通道；且本配置定调"速度不纳入决策考量"。来源：[Fast mode](https://code.claude.com/docs/en/fast-mode) |
| `verbose` | 不设置 | 显示层完整工具输出。"证据不截断"由 7.5 的传输层上限保证；显示层 fullscreen 已提供折叠 + 点击展开。来源：[Settings – verbose](https://code.claude.com/docs/en/settings) |
| `askUserQuestionTimeout` / `crossSessionInbound` / `keybindingFlavor` / `voice` / channels | 不设置 / 保持默认 | 与 GLM 手册同判：自动继续与人工闸门哲学冲突；跨会话入站默认 `hold` 是防注入的安全设计；键位属个人偏好；语音/channels 需 claude.ai 账号体系，纯 API Token 模式不可用。来源：[Settings reference](https://code.claude.com/docs/en/settings-reference) |
| `feedbackDrafts` | 保持默认 | v2.1.247 的 SendFeedback 工具草拟反馈经 `/feedback` 发送——该链路属 Anthropic 侧非必要流量，本环境已由 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` 整体关闭，无需单独处理。来源：[CHANGELOG v2.1.247](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |
| `experimental.cacheTtl`（agent frontmatter） | 不设置 | v2.1.248 新增的每代理 prompt 缓存 TTL（`"5m"`/`"1h"`）。属 Anthropic 官方缓存 TTL 计费机制；DeepSeek 为硬盘缓存自动管理、`cache_control` 被忽略（第 8 章 TTL 类同判），客户端 TTL 声明无对应物。vision 子代理 frontmatter 亦不添加。来源：[CHANGELOG v2.1.248](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |
| `desktopSessionCleanupPeriodDays` | 不设置 | v2.1.248 新增，为 Claude Desktop 写入的会话设置清理豁免上限——claude.ai/Desktop 会话体系专属，纯 API Token 网关模式下不存在该类会话。来源：[CHANGELOG v2.1.248](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |
| `CLAUDE_CODE_RESTRICTED=1`（`--restricted` 启动旗标） | **绝不使用** | v2.1.248 新增的受限模式：移除命令/代码执行类内置工具与 WebFetch、文件工具禁出工作目录、**拒绝 bypassPermissions、忽略用户级/项目级/本地 settings 文件**——与"无人值守全自动 + 用户级配置"的整套架构正相反。属给受限环境管理员提供的锁定开关，非本场景工具。来源：[CHANGELOG v2.1.248](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |

---

# 第三部分：作业手册

## 9. 场景 ①：BMC 级超大代码体系开发

### 9.1 工作区形态

推荐"主仓启动 + 多目录挂载"：在主工作仓（如 openbmc 或自定义元层仓库）启动 `claude`，用 `/add-dir` 挂载 u-boot、kernel、webui 等兄弟仓库。本配置已开启 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`，各仓库自身的 CLAUDE.md/规则会一并载入（A1/A4）。超大单树（monorepo）参见官方专文：[Set up Claude Code in a monorepo or large codebase](https://code.claude.com/docs/en/large-codebases)。

### 9.2 根 CLAUDE.md 模板（置于主仓根目录，按实际项目改写）

模板遵循官方写法要求：单文件控制在 200 行内、规则具体到可验证（"执行 `<测试命令>`"而非"做好测试"）、按主题拆分时使用 `.claude/rules/*.md` 或 `@path` 导入。规则是"上下文"而非强制执行——需硬拦截的动作应使用 deny 规则或 PreToolUse hook（见 10.6）。

- 来源：[Claude Code – Memory（<200 行、具体可验证、rules 目录）](https://code.claude.com/docs/en/memory)

```markdown
# 工程语境
- 本工作区为 BMC 固件全栈：Yocto/bitbake 元层、u-boot、Linux 内核（I2C/IPMB/KCS/MCTP 驱动）、
  phosphor-* D-Bus 服务、bmcweb（Redfish）、webui-vue 前端。
- 目标机型/SoC：<填写>；交叉工具链：<填写>；构建入口：<bitbake 命令/脚本>。

# 硬性约束（违反即返工）
1. 协议字段、寄存器位、D-Bus 接口签名：必须先读到对应规范文本或现有代码定义再落笔，
   禁止凭记忆书写。Redfish 资源以 DSP0266 及 bmcweb 现有 schema 为准；IPMI 命令以规范文档
   和现有 handler 为准。引用时注明文件路径与行号。
2. 内存与实时性：守护进程禁止在热路径引入动态分配与阻塞 I/O；新增依赖必须评估镜像体积。
3. 禁改区：vendor blob、签名/安全启动密钥与流程、许可证头。涉及许可证边界的复制必须停下说明。
4. 每次代码改动后必须依次执行：<格式化命令>、<静态检查命令>、<单元测试命令>；
   跨组件改动必须完成 <QEMU/仿真运行命令> 冒烟。任何一步失败不得宣布完成。
5. commit 边界：一个逻辑变更一个 commit；禁止顺手重构无关代码。

# 视觉证据规程
- 一切视觉证据（报错截图、原理图、时序图、面板照片、复现录屏、扫描版 PDF）不粘贴进对话——
  主线模型不接受图片输入。文件放入 docs/evidence/ 后点名 vision 子代理分析
  （"用 vision 分析 docs/evidence/xxx.png，提取完整报错栈"；视频自动抽帧、扫描件按页读）；
  vision 回传的转录文本即为证据，引用时注明文件路径。
- 数字原生 PDF（有文字层）优先提取文本直读（pdftotext），文字证据不绕视觉通道。

# 长任务规程
- 阶段性结论（架构判断、影响面清单、风险点）写入 docs/worklog/<日期>-<主题>.md 后再继续，
  确保上下文压缩后证据仍在磁盘上。
- 小时级构建用后台方式运行，期间继续推进可并行的工作。

# 检索优先级
上游/开源参考：优先 WebSearch 检索 + WebFetch 读页定位；取证级阅读（协议规范原文、上游源码）
用 Bash 拉取原始内容（git clone/curl）后 Read 全文，不依赖抽取式摘要。
```

### 9.3 任务下达范式

先"技术审计"（架构图、模块职责、API 契约、数据流、调用链、技术债、约束清单），再下达带边界的改造任务。每个任务的下达遵循四要素结构——**Goal（要做成什么）/ Context（涉及哪些文件、报错、文档）/ Constraints（工程约束）/ Done when（怎样算完成）**。长期性规则进 CLAUDE.md，临时性指令进提示词，二者不混放。涉及截图证据的任务，Context 中直接给出图片路径并声明"先经 vision 转录"。

### 9.4 证据落盘规程（对冲压缩损失）

1M 窗口极难占满，但多日任务终会触顶（压缩基准 786432，见 7.4）。规程：**阶段结论必须写入文件**（worklog、影响面清单、决策记录）。压缩摘要会替换对话史，但磁盘文件永久保留，模型可随时重读——证据的持久层在文件系统，不在对话窗口。vision 子代理的关键转录（报错栈全文、图纸要素清单）同样落盘至 `docs/evidence/*.md`。

### 9.5 并行编排

- **计划先行**：大改动先 Shift+Tab 进入 plan 模式出计划，`ExitPlanMode` 未放行，计划必经人工审批（A4 人工闸门）。
- **worktree 隔离**：并行修改多分支时让 Claude 使用 worktree（默认 `.claude/worktrees/` 路径下创建/进入不弹窗），互不污染。
- **批量审计/迁移使用动态工作流（交互触发）**：workflow 是"计划写入脚本"的重型编排——阶段化、可并行、可对抗互验、可保存复跑，单次可并行调度几十上百个子代理（引导已放开至 unrestricted）。本配置下的触发方式：在交互输入框人工输入 `ultracode` 关键词（如 `ultracode: 审计全部 phosphor-* 服务的 XX 用法`）或直接说"用 workflow 并行审计……"——run 内请求按锁定的 max 档执行（7.10）。`/effort ultracode` 的自动编排模式已被 max 锁定关闭，属 7.3 的知情取舍。
- **fork 型子代理**用于"带完整上下文的方案对比"：激进重构方案先 fork 一支推演，主线不受污染。
- **多实例协作**：agent teams 已启用，适合"一人主控 + 多队友分模块"的大迁移；队友恒用 leader 模型（= Pro）。
- **深度代码审查用 `/code-review`**：`high`/`xhigh`/`max` 档在后台 agent 运行、不阻塞主线；云端档位（ultra）需 claude.ai 通道，网关下不可用——本地档位不受影响。
- **长任务托管的第三形态——后台会话（agent view）**：`claude --bg "..."`、会话内 `/bg`、或总控台 `claude agents` 直接派发。会话由 supervisor 进程托管，关闭终端后仍在运行；`claude attach <id>` 随时进入查看，后台会话默认 worktree 隔离。与 `-p` headless 的分工：`-p` 是"射后不管 + 产物落盘"，`--bg` 是"长任务托管 + 中途可介入"。
- **研究型工作流**：内置 `/deep-research` 依赖内置 WebSearch——**本配置下可用**（DeepSeek 服务端原生支持 WebSearch，与 GLM 场景不同），读页环节由 WebFetch 承担；取证级页面仍按 9.2 检索优先级用 Bash 直抓。

- 来源：[Claude Code – Workflows（定位、规模、adversarial review、触发语义）](https://code.claude.com/docs/en/workflows)、[Claude Code – Agent view](https://code.claude.com/docs/en/agent-view)、[Claude Code – Tools reference（EnterWorktree/Agent/Workflow 行为）](https://code.claude.com/docs/en/tools-reference)、[DeepSeek – 接入 Claude Code（WebSearch 原生支持）](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)

## 10. 场景 ②：A 股深度行情分析

### 10.1 总原则

LLM 在金融数据上的唯一可靠形态是**编排器**而非数据库：它负责取数、编写代码计算、交叉验证、组织论证；数字本身必须全部来自工具调用。以下规程将这一点制度化。

### 10.2 数据铁律（写入 CLAUDE.md，同时也是验收结论的标准）

1. **零记忆数字**：任何行情/财务/宏观数字，禁止直接引用模型记忆；必须来自当轮工具调用（本地数据接口或检索链路），并标注「来源（接口名或 URL）+ 获取时间」。
2. **计算全部落代码**：均线、波动率、估值分位、组合归因等一律由 Python 代码算出（Bash/PowerShell 工具执行），模型不做心算——采样随机性不影响 `pandas` 的输出，这才是"数值绝对可靠"的机制保证（也是第 8 章不注入采样参数的底气：DeepSeek 思考模式本就不接受采样参数）。
3. **双源交叉**：进入结论的关键数字（收盘价、涨跌幅、成交额、财务科目）至少两个独立来源核对；不一致时必须报告差异与取舍理由。
4. **来源分级**：交易所与法定披露渠道（上交所/深交所官网、巨潮资讯网、中国结算、央行/统计局/证监会）> 结构化数据接口（如 akshare/tushare/baostock 等本地库）> 财经媒体/聚合站。级别低的来源只能佐证，不能独立支撑结论。
5. **有损工具不入账**：WebFetch 是抽取式设计（页面经模型单遍抽取，本配置已将抽取模型升至 Pro，但仍有损）——关键数字取证使用 Bash 中 `curl`/`python requests` 获取**原始页面/接口返回**后 Read 全文，WebFetch 仅用于泛读定位。
6. **检索走内置 WebSearch**：DeepSeek 服务端原生执行（7.8），时效信息（政策、公告、突发）以 WebSearch 定位 + 原始页面直抓取证，形成"检索定位 → 原文取证"两段式。
7. **图表证据走 vision 子代理**：K 线形态、研报截图、数据平台图表由 vision 转录读数（标注估读精度）；**精确数值仍以数据接口为准**，视觉读数只用于形态判断与线索定位，不得直接进入计算（C3）。

- 来源：[Claude Code – Tools reference（WebFetch 抽取式设计、WebSearch）](https://code.claude.com/docs/en/tools-reference)、[DeepSeek – 图像理解](https://api-docs.deepseek.com/zh-cn/guides/vision)

### 10.3 A 股 CLAUDE.md 模板（置于分析工作区根目录）

```markdown
# 角色与目标
你是 A 股市场分析工程师：负责取数、验证、计算、组织多空论证，产出可审计的分析报告。

# 数据铁律（不可违反）
1. 任何数字必须来自本轮工具调用，标注来源与获取时间；禁止凭记忆给数。
2. 一切计算写 Python 执行（pandas），禁止心算；脚本与中间数据落盘到 data/ 与 scripts/。
3. 关键数字双源交叉验证；不一致必须披露差异及取舍理由。
4. 数据不可得就明说"取不到"，禁止估计值冒充实际值。
5. 图表/截图证据：文件放入 charts/ 后交 vision 子代理转录（"用 vision 读 charts/xxx.png 的
   轴刻度与形态"）；视觉读数标注估读精度，只用于形态判断，精确数值一律回到数据接口取数。

# 口径声明（每份报告开头必须给出）
- 数据截止时间（Asia/Shanghai）；行情快照 vs 收盘数据。
- 复权方式：前复权/后复权/不复权（涉及跨期比较必须声明）。
- 单位统一声明：元/万元/亿元；成交额与市值单位分开写清。
- 财务数据：报告期（如 2026Q1）与公告日期分开标注；同比/环比基期写明。
- 指数/板块口径：指数代码与编制方（中证/国证/申万行业版本）。

# 交易规则常识（计算与回测必须内建）
- T+1；交易日历以交易所公布为准（节假日、临时休市）。
- 涨跌停：主板 ±10%，ST/*ST ±5%，创业板/科创板 ±20%，北交所 ±30%；
  新股/次新与退市整理期规则特殊，单独核对后再用。
- 停复牌、退市、ST 变更影响可交易性与连续性：回看历史用时点数据（point-in-time），
  防幸存者偏差；禁止用后来发生的事解释之前的价格。

# 全面性检查清单（深度分析必须逐项过）
宏观与政策 / 行业景气与产业链上下游 / 公司基本面（财报三表+关键科目异动）/
资金面（成交、换手、大宗、股东户数、北向及融资融券如可得）/ 估值（绝对+相对+历史分位）/
情绪与预期差 / 风险与反方论证（必须主动检索做空/看空观点并回应）/ 结论的可证伪条件。

# 输出规范
- 报告落盘 reports/<日期>-<主题>.md；附数据来源清单表（数值|来源|获取时间|校验源）。
- 明确区分「事实（带来源）」「推断（带逻辑链）」「观点（带置信度与前提）」。
- 本工作区只做研究分析，不接实盘下单接口。
```

### 10.4 深度与全面性的执行手段

- `reasoning effort = max` 已锁定（7.3），无需在提示词中要求"深度思考"。
- 全面性依靠**结构**而非"多想想"：要求模型按 10.3 检查清单逐项核对，并使用 fork/子代理并行核查——例如"fork 两支分别构建多头与空头最强论证，主线汇总裁决"（fork 继承完整上下文，证据不丢失，见 7.10）。
- **重大专题使用对抗互验工作流（交互触发）**：workflow 的官方设计即包含 "independent agents adversarially review each other's findings before they're reported"——对"某行业景气拐点是否成立"这类命题，在交互输入框下达"用 workflow：多角度分头取证 → 交叉互验 → 汇总裁决；检索用 WebSearch、取证按数据铁律直抓原文"。
- 每份报告强制设"反方论证"章节：主动检索看空/风险观点并逐条回应，纠正单边倾向。
- **自备金融数据 MCP 置于项目级**：若接入行情/资讯类 MCP（如 akshare、wind 的本地封装），其 allow 规则（`"mcp__<server>"`）放入**分析工作区的** `.claude/settings.json`，不进入全局用户级——部署后才放行、重装不残留；全局层 7.5/7.6 的 MCP 上限与超时参数已为此预置。
- 来源：[Claude Code – Workflows（adversarial review 模式）](https://code.claude.com/docs/en/workflows)、[Claude Code – Permissions（作用域合并）](https://code.claude.com/docs/en/permissions)

### 10.5 持续稳定复跑（盘后例行）——双轨制

无人值守采用 **Windows 任务计划程序 + headless 模式**，按任务性质分两轨：

```powershell
# 轨道一：每日例行（结构稳定、可比对、可复跑——B5 主力）
claude -p "执行每日例行分析：按 CLAUDE.md 铁律更新自选池数据、运行 scripts/daily.py、生成 reports/$(Get-Date -Format yyyy-MM-dd)-daily.md，全部数字附来源清单"

# 轨道二：重大专题（季度策略复盘、行业景气拐点判定——多路并行深挖 + 对抗互验）
claude -p "对 <标的/行业> 做深度多空核查：并行派出多个子代理分别负责宏观政策、行业景气、公司基本面、资金面、反方论证，各自独立取证后回传，主线交叉互验并汇总裁决；检索用 WebSearch、关键数字按数据铁律双源直抓取证；产出 reports/$(Get-Date -Format yyyy-MM-dd)-deep.md，全部数字附来源清单"
```

双轨的划分依据是**产物性质**：每日例行的价值在于结构稳定、日间可比、异常可归因——固定脚本 + 单线程 max 是其正确形态；轨道二的多路并行深挖用于重大专题。

headless 下的三个关键机制（均已在本配置内置或说明）：

1. **`-p` 提示词不触发 workflow**（官方：`ultracode` 关键词与自然语言 workflow 请求仅对人工输入生效，防注入设计），且本配置锁定 max 后 `--effort ultracode` 启动参数也被环境变量覆盖（7.3/7.10）——因此轨道二的无人值守形态是**显式指令化的并行子代理**（Agent 工具不受触发语义限制，fork 已开启 headless 支持）：并行结构写死在提示词中，反而带来日间可比性。需要 workflow 级"对抗互验编排"的专题，改用交互会话人工触发（10.4），或 `claude --bg` 托管后 attach 输入。
2. **`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS=0`（settings 已内置）**：headless 默认仅等待后台子代理 10 分钟，超时直接终止剩余任务并退出——轨道二的并行深挖极易超过 10 分钟，不设 `0` 即为最隐蔽的"无人值守被截断"。
3. 子代理在 headless 下无人可询问：不在 allowlist 的调用被直接拒绝（而非弹窗）——本配置 16 条 allow 已覆盖（7.9）。

其余要点：`-p` 沿用同一份用户级 settings（bypass、重试看门狗、模型矩阵全部生效）；产物落盘 = 审计底稿；`cleanupPeriodDays=365` 保留转录一年；例行任务天然运行于空闲时段（北京时间高峰为工作日 9:00–12:00、14:00–18:00，盘后与夜间均为空闲段，价格减半——按总原则这只是附带效果，非决策依据）。

需要中途跟进的重大专题可改用**后台会话形态**：`claude --bg "..."` 托管（supervisor 进程管理，关闭终端不中断），`claude agents` 总控台随时 attach 查看进度，完成/待输入时终端铃声提醒（7.12）。

- 来源：[Claude Code – Workflows（触发语义仅限人工输入、acceptEdits 语义）](https://code.claude.com/docs/en/workflows)、[Claude Code – env-vars（`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`、`CLAUDE_CODE_FORK_SUBAGENT`）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Agent view](https://code.claude.com/docs/en/agent-view)、[DeepSeek – 模型 & 价格（峰谷时段定义）](https://api-docs.deepseek.com/zh-cn/quick_start/pricing)

### 10.6 实盘边界（唯一建议保留的硬闸）

分析与执行必须物理分离：实盘下单脚本置于独立仓库，在**该仓库**的 `.claude/settings.local.json` 中添加窄 deny（如 `"Bash(python live_trade.py*)"`、`"Edit(//C/trading/keys/**)"`）。deny 规则在包括 bypass 在内的一切模式下生效，是官方提供的唯一硬闸；分析工作区保持零 deny 全自动。

- 来源：[Claude Code – Permissions（deny 全模式生效、路径规则语法）](https://code.claude.com/docs/en/permissions)

## 11. 多模态证据作业手册（vision 子代理）

架构与机制见 7.13；本章为使用规程。vision 子代理是**两场景共用**的多模态证据通道：BMC 侧的报错截图、原理图、时序图、面板照片、复现录屏，A 股侧的 K 线图、研报扫描件、数据平台图表——图片直读、视频抽帧、扫描件按页读，全部视觉证据由 vision-exp 的多模态承载处理。

### 11.1 部署与触发

- 部署：[`agents/vision.md`](./agents/vision.md) → `%USERPROFILE%\.claude\agents\vision.md`（用户级，全项目可用；亦可放入 `<项目>\.claude\agents\` 做项目级覆写）。
- 触发：主线依据子代理 `description` **自动委派**（其中已声明"任何涉及图片或视频文件的任务必须委派本代理"），亦可 `@vision` 显式点名。可靠的下达格式：**文件路径 + 要回答的问题**——"用 vision 分析 `logs/crash-0827.png`：提取完整报错栈与可见的环境信息"、"用 vision 分析 `docs/evidence/repro-0827.mp4`：找出报错弹窗出现的时刻与全文"。
- 观测：`/tasks` 或代理详情应显示该子代理运行于 `deepseek-v4-flash-vision-exp`（部署后按第 15 章第 8 条验证一次）。

### 11.2 多模态规程（防 400 与防细节丢失）

1. **视觉证据永远是文件，不是粘贴物**：粘贴进主会话输入框的图片会随请求发给 V4-Pro → `400: This model does not support image`。规程：截图/录屏/扫描件落盘到工作区（建议 `docs/evidence/` 或 `charts/`），对话中只出现路径，点名 vision 处理。
2. **图片直读**：JPEG/PNG/GIF/WebP（按文件实际内容判断，与扩展名无关），Read 工具路径。大图先裁剪——两级压缩会吃掉细节（客户端对超 500KB 的图降质重编码；DeepSeek 端把大图缩到约 800×800 当量、单图至多 384 token）。vision 子代理的系统提示已内置裁剪规程（对高分屏截图的局部报错、密集小字、需精确读数的图表，先用 Python Pillow/ImageMagick 裁出感兴趣区域再读，必要时分块）；下达任务时主动指出关注区域（"右下角的报错弹窗"）能显著提高转录质量。
3. **视频走抽帧**（官方无视频输入，抽帧是对视频证据的最大化利用）：子代理用 ffmpeg 抽帧后逐帧读——通览 `fps=1`、抓闪现报错/弹窗 `fps=2` 或全关键帧；长视频先低密度通览、再对可疑片段高密度重抽；产出带时间点（"t=00:14 处出现红色弹窗，文字为……"）。ffmpeg 缺失时 `winget install Gyan.FFmpeg`。
4. **PDF/扫描件分流**：数字原生 PDF（有文字层）优先 `pdftotext`/PyPDF 提取文本走主线 Pro——文字证据由旗舰直读，更准且不占视觉通道；扫描版/图片型 PDF 由 vision 按页转为图片读取（官方 document 块为"不支持"，页转图是唯一合规路径）。
5. **计费口径**：图片按尺寸换算 token 与文本一并计费（vision-exp 单价与 Flash 一致）；单图 token 上限 384，尺寸再大也不会超过。
6. **转录即证据**：vision 回传的文本是主线唯一可见的证据载体（主线看不到原始文件）。关键转录按 9.4 落盘；报告中引用时注明「文件路径 + vision 转录 + 获取时间」。
7. **视觉读数不进计算**：图表估读只用于形态判断与线索定位；进入计算与结论的数值必须回到数据接口/原文取证（10.2 第 7 条）。

### 11.3 典型任务模板

```text
# debug 截图（BMC/CI 报错）
用 vision 分析 docs/evidence/ci-fail-0827.png：逐字提取报错消息、错误码、栈帧与文件行号；
记录截图中可见的环境线索（终端标题/路径/版本号/时间戳）；报错被截断处明确标注。

# UI 走查（前端还原度）
用 vision 对比 docs/evidence/design.png 与 docs/evidence/actual.png：
按区块列出布局/配色/字号/间距差异，标注各差异在截图中的位置；不确定处标注"不可辨认"。

# 架构图/时序图理解
用 vision 读 docs/evidence/ipmb-timing.png：转录全部节点/信号名与连线关系，
输出时序步骤清单；图中文字不可辨认处逐项列出，不得脑补。

# K 线/图表读数（A 股）
用 vision 读 charts/601398-daily.png：先锁定口径（坐标轴含义、时间范围、复权标注、成交量单位），
再描述形态（趋势、跳空、放量位置），读数一律带估读精度区间；提醒主线：精确数值回数据接口取。

# 复现录屏/视频取证（BMC、UI 回归）
用 vision 分析 docs/evidence/repro-0827.mp4：抽帧找出报错弹窗/崩溃画面出现的全部时刻，
逐字转录弹窗文字与可见的终端输出；描述触发前的操作序列。

# PDF 规范/研报阅读
数字原生 PDF：主线先 pdftotext 提取全文直读（文字走旗舰）。
扫描版研报：用 vision 逐页读 docs/evidence/report-0827.pdf 的关键页，
逐字转录财务表格的科目/数值列，跨页与截断处明确标注。
```

### 11.4 已知边界

- vision-exp 为**实验性模型**（官方 `-exp` 后缀），存在迭代与调整的可能——纳入第 14 章漂移源清单监控；其纯文本能力为 Flash 级，**任何非视觉任务不得委派给它**（子代理 description 与本手册规程双重约束）。
- **多模态官方边界为仅图片**：视频输入官方文档未见、PDF/document 块官方明确"不支持"——视频抽帧与扫描件页转图不是权宜，而是该边界下对视觉证据的最大化利用形态；数字原生 PDF 的文字层走主线 Pro（11.2 第 4 条），两模型各压榨各的。
- 图片仅允许出现在 user 消息中（system/assistant 携带图片返回 400）——Claude Code 的 Read 工具路径天然合规，无需处理。
- 单请求限制三层：单图 32 MiB（经 Files API `file_id` 引用时放宽到 64 MiB）、编码后计入 48 MiB 请求体上限、每请求至多 600 张；**单请求 ≥15 张图时单边像素上限从 8192 降为 4096**——批量图片任务（整目录截图审查）让 vision 分批读取（每批 <15 张，保住像素上限即保住可辨认度）。Files API（`file_id` 复用、单图最大 64 MiB）是 API 直调场景的机制，Claude Code 的 Read 路径用不到，不配置。

- 来源：[DeepSeek – 图像理解（全部限制与计费）](https://api-docs.deepseek.com/zh-cn/guides/vision)、[DeepSeek – Vision-Exp 发布公告](https://api-docs.deepseek.com/zh-cn/news/news260821)、[Claude Code – Tools reference（Read 图片压缩行为）](https://code.claude.com/docs/en/tools-reference)、[Claude Code – Sub-agents（自动委派依据 description）](https://code.claude.com/docs/en/sub-agents)

---

# 第四部分：运维

## 12. 余额、并发与峰谷时段（可用性机制）

DeepSeek 为**充值按量计费**（无订阅套餐），可用性约束与 GLM Coding Plan 的"5 小时窗口/周配额"完全不同——不存在配额窗口，多账号轮换机制**不再需要**；取而代之的两个约束是**余额**与**并发**：

1. **余额（402）**：余额不足时 API 返回 `402`，重试不解决（`CLAUDE_CODE_RETRY_WATCHDOG` 对计费类错误也是立即失败语义）——**充值是唯一路径**。纪律：无人值守任务开跑前核对余额（[平台用量页](https://platform.deepseek.com/usage)，或 API `GET /user/balance` 脚本化预检并纳入任务计划程序的前置步骤）；赠送余额与充值余额并存时优先扣减赠送余额。
2. **并发（429）**：账号粒度并发限制——**V4-Pro 500 / V4-Flash 2500 / Vision-Exp 2500**；超限收到 429，由看门狗按退避重试自动吸收。单人多会话 + 数十子代理并行距 500 并发尚有巨大余量；确需更高并发（多机并行审计），**提交账号扩容申请工单，扩容不增加额外费用**——这是官方提供的正道，优于任何客户端并发调参（第 8 章）。
3. **峰谷定价**：高峰为北京时间周一至周五 9:00–12:00、14:00–18:00，其余时段价格减半。按总原则成本不入决策；客观效果是盘后例行分析（10.5）与夜间长审计天然落在空闲段。
4. **请求保活**：请求发出后未开始推理期间，服务端持续返回保活内容（非流式空行/流式 SSE 注释）；**10 分钟仍未开始推理则服务器关闭连接**，由客户端重试层接管（7.6）。
5. **user_id 隔离**（备查）：多业务共用账号时可经 Anthropic 协议的 `metadata.user_id` 做 KVCache 与调度隔离；本配置单人使用，不设置。

- 来源：[DeepSeek – 限速与隔离（并发表、扩容工单、user_id、保活）](https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit)、[DeepSeek – 错误码（402/429）](https://api-docs.deepseek.com/zh-cn/quick_start/error_codes)、[DeepSeek – 模型 & 价格（峰谷时段、扣费规则）](https://api-docs.deepseek.com/zh-cn/quick_start/pricing)、[Claude Code – env-vars（RETRY_WATCHDOG 计费类立即失败）](https://code.claude.com/docs/en/env-vars)

## 13. 升级 SOP

自动更新已被 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` 关闭，升级完全由使用者主动执行：

1. **确认无任务运行**（升级 = 系统提示词变化 = 行为漂移 + 缓存冷启动，只能发生在任务间隙——A6）。
2. 连接 VPN → `claude --version` 记录当前版本。
3. 升级（按安装方式）：原生安装 `claude update`；WinGet `winget upgrade Anthropic.ClaudeCode`；npm `npm install -g @anthropic-ai/claude-code@latest`。
4. `claude --version` + `claude doctor` 验证（不得低于 `minimumVersion: 2.1.246`）。
5. 断开 VPN，正常使用（模型流量仅走 `api.deepseek.com`）。
6. 升级后**从新会话开始新任务**；避免立即 `--resume` 长会话（升级后恢复会话将无缓存重放全部历史，且新系统提示下的行为与原任务存在漂移）。

DeepSeek 侧无需升级动作（服务端热更新，见第 14 章的监控对策）。

- 来源：[Claude Code – Overview（各安装方式更新行为）](https://code.claude.com/docs/en/overview)、[Claude Code – Prompt caching（Upgrading Claude Code）](https://code.claude.com/docs/en/prompt-caching)

## 14. 一致性管理：任务中途的行为漂移源清单

以下操作会在任务中途改变"回答者的条件"（缓存重建只是其可观测的影子），**任务进行中一律避免**，统一移至任务边界执行：

| 漂移源 | 后果 | 对策 |
| --- | --- | --- |
| `/model` 切换模型 | 不同模型 = 不同回答者 | 主线全 Pro 单模型，选单中已无备选；不得手动指定其他模型 |
| `/effort` 切换档位 | 推理深度改变 | 已用环境变量锁定 max，`/effort` 实际失效（显示被接管） |
| **DeepSeek 服务端模型热更新** | `deepseek-v4-pro` 恒指向最新版本（如 0731→0813 类升级由服务端推送），**无版本钉住机制**——这是按量 API 与"模型名即版本"体系的结构性差异，使用方无法锁定 | 无法消除，只能管理：关注[官方更新日志](https://api-docs.deepseek.com/zh-cn/updates)与公告；重大长任务避开官方已预告的模型更新窗口；版本更新后按第 15 章清单回归验证一遍 |
| **vision-exp 实验性迭代** | `-exp` 后缀模型可能调整、更名或转正 | 同上；若模型名变更，同步修改 `agents/vision.md` 的 frontmatter 一处即可（架构上仅此一处引用） |
| 升级 Claude Code | 系统提示词/工具定义变化 | 第 13 章 SOP：仅在任务间隙升级 |
| MCP 服务器中途连接/断开 | 工具定义集合变化 | 本配置零常驻 MCP，此源天然消除；自备 MCP（10.4）部署后不再变动 |
| 以裸工具名 deny 工具 | 工具集合变化 | deny 保持为空（7.9） |
| `/compact` 压缩 | 对话史被摘要替换 | 手动压缩安排在任务之间；证据按 9.4/10.5 规程落盘 |
| 中途修改 CLAUDE.md | 本会话不生效（下次启动才生效），易产生"以为已修改"的错觉 | 修改规则后重启会话再开工 |
| 中途修改 `agents/vision.md` | 同上（子代理定义会话启动时装载） | 同上 |

无漂移风险、可随时执行：编辑仓库文件、Shift+Tab 切换权限模式、调用技能/命令、`/recap`、`/rewind`、派生子代理（含 vision）。

观测手段（按需使用、无常驻组件）：`/status` 查看模型与设置来源、`/context` 查看窗口占用与压缩水位、`/effort` 确认档位被环境变量接管、`/tasks` 查看子代理实际模型；DeepSeek 侧缓存命中在响应 `usage.prompt_cache_hit_tokens` 观察，各模型调用分布在[平台用量页](https://platform.deepseek.com/usage)核对（兼作"静默映射到 flash"的巡检，见 5.1）。

- 来源：[DeepSeek – 更新日志（模型版本随模型名滚动）](https://api-docs.deepseek.com/zh-cn/updates)、[Claude Code – Prompt caching](https://code.claude.com/docs/en/prompt-caching)、[DeepSeek – 上下文硬盘缓存（usage 字段）](https://api-docs.deepseek.com/zh-cn/guides/kv_cache)

## 15. 部署后验证清单

| # | 动作 | 期望结果 |
| --- | --- | --- |
| 1 | `claude --version` | ≥ v2.1.246（低于 `minimumVersion` 会拒绝启动；全部版本门槛见 17.2） |
| 2 | 启动 `claude` | 无认证报错；`⏵⏵ bypass permissions on` 徽标 |
| 3 | `/status` | Setting sources 含 User settings；Model 为 `deepseek-v4-pro[1m]`（显示名 DeepSeek-V4-Pro (1M)） |
| 4 | `/model` | 各别名位均显示 DeepSeek-V4-Pro；**无** flash / vision 备选项 |
| 5 | `/effort` | 提示档位被环境变量接管（max） |
| 6 | `/context` | 窗口按 1M 计，压缩基准 786432 |
| 7 | 复杂提问 | 思考过程可见（`Ctrl+O`），流式稳定 |
| 8 | 项目放置一张报错截图，"用 vision 分析 error.png，提取完整报错栈" | 主线委派 vision 子代理；`/tasks`/代理详情显示其模型为 `deepseek-v4-flash-vision-exp`；回传逐字转录 |
| 8a | 放一段复现录屏 + 一份扫描版 PDF，各点名 vision 分析 | 录屏自动 ffmpeg 抽帧并按时间点转录；扫描件按页读。若 PDF 直读报 400（document 块不支持）：按 11.2 第 4 条分流——数字原生走 pdftotext 主线直读，扫描版页转图后读 |
| 9 | 同一张图直接粘贴进主会话输入框（**边界验证，做一次即可**） | 请求报 400（This model does not support image）——印证 11.2 规程"图片永远是文件" |
| 10 | "搜索今天沪深两市成交额并给来源" | 内置 WebSearch 工具执行成功（DeepSeek 服务端），数字带来源与时间 |
| 11 | WebFetch 拉取任意 https 页面 | 可用（Anthropic 预检已跳过） |
| 12 | 交互输入 `ultracode: 审计 <目录> 的 XX 用法` | 触发 workflow 编排，`/workflows` 可见并行子代理，代理数不被压在 15 以内 |
| 13 | Shift+Tab | manual → acceptEdits → plan → bypass 循环 |
| 14 | 日常断开 Anthropic 访问 | 无升级/遥测报错刷屏；WebFetch/WebSearch 正常 |
| 15 | `/tui`（不带参数） | Current renderer: fullscreen |
| 16 | 长任务运行期间切换到其他窗口 | 完成或等待输入时终端铃声（Windows Terminal 需为 bellStyle 启用声音/任务栏闪烁至少一项） |
| 17 | 下达含多步清单的任务 | 任务清单工具可用（TaskCreate/TaskUpdate 正常被调用） |
| 18 | [平台用量页](https://platform.deepseek.com/usage)核对当日调用分布 | 主线调用计在 `deepseek-v4-pro`，vision 调用计在 `deepseek-v4-flash-vision-exp`；**`deepseek-v4-flash` 应为零**（出现即说明存在拼写错误引发的静默映射，见 5.1/16） |

## 16. 故障排查速查表

| 症状 | 原因 | 处置 |
| --- | --- | --- |
| `401 认证失败` | API Key 错误/已删除 | 核对 [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)；确认无 `ANTHROPIC_API_KEY` 残留 |
| `402 余额不足` | 账户余额耗尽（重试看门狗对计费类错误立即失败，属正确行为） | [充值](https://platform.deepseek.com/top_up)；无人值守任务按第 12 章加余额预检 |
| 主线能力明显变弱/用量页出现 flash 调用 | 模型名拼写错误，或 CLI 版本过旧不识别 `[1m]` 后缀（把 `deepseek-v4-pro[1m]` 整串发往端点）→ DeepSeek 端对不认识的名字**静默映射到 flash**（不报错） | 逐字核对 `env` 四个模型位的 `deepseek-v4-pro[1m]`/`deepseek-v4-pro`；`claude --version` ≥ 2.1.246；按 15.18 巡检 |
| `400 This model does not support image` | 图片被发给了非视觉模型：粘贴进主会话，或 vision 子代理被改到文本模型 | 按 11.2 图片落盘走 vision；确认**未设置** `CLAUDE_CODE_SUBAGENT_MODEL`（含 shell 残留），`/tasks` 核对 vision 实际模型 |
| vision 子代理未被自动委派 | 主线未识别任务的视觉属性 | `@vision` 显式点名；任务描述中给出图片路径；核对 `agents/vision.md` 部署路径与 frontmatter `description` |
| vision 读 PDF/视频直接报错 | 官方边界：document 块不支持、无视频输入 | 预期内——按 11.2 规程：视频先 ffmpeg 抽帧、PDF 先判文字层（原生走 pdftotext 主线读，扫描版页转图）；这不是故障，是专线设计的工作形态 |
| 抽帧/批量读图后细节丢失 | 单请求 ≥15 张时单边像素上限降为 4096 | 分批读取（每批 <15 张）；关注区域先裁剪再读 |
| vision 转录细节缺失/读数模糊 | 两级压缩（客户端 500KB 重编码 + 服务端 800×800 缩放） | 按 11.2 先裁剪感兴趣区域再读；下达任务时指明关注区域 |
| `429` 频发 | 并发超限（Pro 500 并发，账号粒度）或 TPM/RPM 触顶 | 看门狗自动退避；持续不足提交扩容工单（免费，第 12 章）；勿调高客户端并发变量（第 8 章） |
| 修改 settings 不生效 | JSON 语法错误/文件位置错误/旧窗口 | 全部关闭重开；`/status` 查看 sources；`claude doctor` |
| 响应中途中断 | 链路抖动；或请求 10 分钟未开始推理被服务端关闭（保活机制） | 已含三层看门狗与重试（7.6）；频发时检查本地代理是否拦截 `api.deepseek.com` |
| 请求在思考停顿处被切断 | 空闲定时器误判 | 确认 `API_FORCE_IDLE_TIMEOUT=0`、`CLAUDE_STREAM_IDLE_TIMEOUT_MS=1800000` 生效 |
| "忘记前文" | 提前压缩或窗口误判 | `/context` 核对（应为 1M / 基准 786432）；确认 `[1m]` 后缀与 `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 生效 |
| 接近满窗口时请求 400（输入+输出超窗） | 极端场景：786432 输入 + 262144 输出预算逼近窗口 | 属预期外情况（压缩应先触发）；确需处理时将 `CLAUDE_CODE_MAX_OUTPUT_TOKENS` 降至 196608 |
| WebFetch 全部失败 | 未跳过 Anthropic 预检 | `skipWebFetchPreflight: true`（7.8） |
| WebSearch 报错 | DeepSeek 服务端搜索异常或端点配置错误 | 核对 base_url；稍后重试；临时改用 Bash 直抓 + 已知信源 |
| `/effort ultracode` 提示被环境变量覆盖 | max 锁定下 ultracode 自动编排关闭 | **预期行为**（7.3 知情取舍）；用交互关键词触发单任务 workflow（9.5） |
| `-p` 提示词中写了 ultracode 却没有 workflow | 关键词仅对人工输入生效（防注入） | 无人值守按 10.5 轨道二用显式并行子代理；或改交互/`--bg` 形态 |
| workflow 代理数总在十几个封顶 | `workflowSizeGuideline` 被项目级设置覆盖回 medium | 确认 unrestricted 生效（`/status` 查来源） |
| 无人值守深度分析 10 分钟被终止 | headless 后台等待上限默认 10 分钟 | 本配置已设 `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS=0`；核对设置来源 |
| 全屏渲染残影/错位碎片 | ConPTY 增量重绘缺陷 | 确认 `CLAUDE_CODE_ALT_SCREEN_FULL_REPAINT=1`；滚动空白再叠加 `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL=1`；临时回退 `/tui default` |
| 任务清单工具不可用 | 新版对较新模型默认移除 | 确认 `CLAUDE_CODE_ENABLE_TODO_TOOLS=1` 生效 |
| 后台子代理被误标 "failed" | 长命令期间无进度事件被判失速 | 已放宽到 1h（`CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS`）；超 1h 的构建改用后台命令 |

---

# 第五部分：附录

## 17. DeepSeek-V4 版本时间线与版本门槛

### 17.1 V4 系列时间线（模型侧）

| 日期 | 事件 | 与本配置的关系 |
| --- | --- | --- |
| 2026-04-24 | **DeepSeek-V4 预览版**发布并开源：1M 上下文标配（token 维度压缩 + DSA 稀疏注意力）、针对 Claude Code 等 Agent 产品专项优化；旧模型名 `deepseek-chat`/`deepseek-reasoner` 宣布 2026-07-24 停用 | 1M 窗口与 Agent 适配的机制基础 |
| 2026-07-31 | **V4-Flash 正式版（0731）**：Agent 能力大幅增强，原生支持 Responses API | vision-exp 的文本能力基线 |
| 2026-08-13 | **V4-Pro 正式版（0813）**：Code Agent 基准全面领先（第 3 章表）；思考强度三档 low/high/max；峰谷定价 8-17 生效 | 本配置主线模型的现行版本 |
| 2026-08-21 | **V4-Flash-Vision-Exp** 上线：DeepSeek API 开启多模态服务，图片输入独占；多模态 Agent 能力接近 Opus-4.8、纯文本与 Flash 持平；Files API 同步开放（免费） | 本配置多模态专线模型 |

要点：**模型名即滚动版本**（`deepseek-v4-pro` 恒指向最新版），服务端热更新、无钉版机制——已列入第 14 章漂移源清单管理。模型侧"升级"无需使用方操作，更新后按第 15 章回归验证。

- 来源：[DeepSeek – 更新日志](https://api-docs.deepseek.com/zh-cn/updates)、[DeepSeek – V4 发布公告](https://api-docs.deepseek.com/zh-cn/news/news260424)、[DeepSeek – V4-Pro 正式版公告](https://api-docs.deepseek.com/zh-cn/news/news260813)、[DeepSeek – Vision-Exp 公告](https://api-docs.deepseek.com/zh-cn/news/news260821)

### 17.2 版本门槛备查表（Claude Code 侧）

| 变量/机制（本配置依赖） | 最低版本 | 旧版行为 |
| --- | --- | --- |
| 第三方网关凭证隔离（DeepSeek Key 不发往 Anthropic 遥测） | **v2.1.246（门槛锚点）** | Key 可能随遥测/计量请求发往 Anthropic host |
| `/tasks` 显示子代理实际模型与 effort（vision 路由观测） | v2.1.243 | 无法直接核对 vision 子代理所用模型 |
| 网关/自定义 base URL 下提示缓存修复 | v2.1.237 | 网关会话缓存可能失效（无损但每轮冷算） |
| `CLAUDE_CODE_ENABLE_TODO_TOOLS=1` | v2.1.233 | 无此变量；较新模型判定下任务清单工具缺失 |
| `CLAUDE_CODE_FORK_SUBAGENT=1` 在 `-p` 下生效 | v2.1.232（交互默认启用同版本） | 旧版语义一致，仍需显式置 1 |
| 子代理 frontmatter `model` 指定完整模型 ID + 解析优先级现行语义 | v2.1.196+（`SUBAGENT_MODEL=inherit` 等价不设） | 更旧版本 inherit 语义不同 |
| `workflowSizeGuideline=unrestricted`（设置键） | v2.1.219 | 旧版忽略该键 |
| `CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS=0` | v2.1.212 | 静默回退 2 分钟自动转后台 |
| `CLAUDE_CODE_RETRY_WATCHDOG=1`（含计费类 429 立即失败语义） | v2.1.186（语义细化 v2.1.239） | 旧版对支出类 429 无限等待 |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 对自定义模型名直接生效 | v2.1.193 | 需配合其他条件 |

`minimumVersion: "2.1.246"`（7.11）已把门槛固化为启动检查——切换发布通道降级到门槛以下会直接拒绝运行，而非静默回退。使用者当前环境为 v2.1.250，全部满足（v2.1.247–250 区间配置零改动：无新采用键，新增键均判不适用，取舍见第 8 章；2.1.249 未发布，2.1.250 整版仅可靠性修复）。

- 来源：[Claude Code – CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)、[Claude Code – env-vars（各变量版本注记）](https://code.claude.com/docs/en/env-vars)、[Claude Code – Sub-agents](https://code.claude.com/docs/en/sub-agents)

## 18. 参考链接汇总

**Claude Code 官方文档**

- 总览与安装：<https://code.claude.com/docs/en/overview>
- 设置（作用域/优先级/热加载）：<https://code.claude.com/docs/en/settings>
- 设置全参考（settings-reference，含 `modelPicker`/`effortLevel` 枚举）：<https://code.claude.com/docs/en/settings-reference>
- 环境变量全参考：<https://code.claude.com/docs/en/env-vars>
- 模型配置（别名/[1m]/effort/能力声明/回退链/窗口修正）：<https://code.claude.com/docs/en/model-config>
- 子代理（frontmatter model/解析优先级/自动委派）：<https://code.claude.com/docs/en/sub-agents>
- 权限系统：<https://code.claude.com/docs/en/permissions>
- 权限模式（Shift+Tab）：<https://code.claude.com/docs/en/permission-modes>
- 工具参考（权威工具名/Read 图片行为/WebSearch/WebFetch）：<https://code.claude.com/docs/en/tools-reference>
- 提示缓存机制：<https://code.claude.com/docs/en/prompt-caching>
- 动态工作流（workflow/ultracode 触发语义）：<https://code.claude.com/docs/en/workflows>
- Agent view / 后台会话：<https://code.claude.com/docs/en/agent-view>
- 全屏渲染：<https://code.claude.com/docs/en/fullscreen>
- 终端配置（铃声通知）：<https://code.claude.com/docs/en/terminal-config>
- 记忆机制（CLAUDE.md/规则）：<https://code.claude.com/docs/en/memory>
- 超大代码库/monorepo 专文：<https://code.claude.com/docs/en/large-codebases>
- CHANGELOG（版本门槛核对）：<https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md>

**DeepSeek 官方文档（中文）**

- 文档首页 / 首次调用：<https://api-docs.deepseek.com/zh-cn/>
- 模型 & 价格（1M/384K/峰谷/并发）：<https://api-docs.deepseek.com/zh-cn/quick_start/pricing>
- 接入 Claude Code（官方推荐配置）：<https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code>
- 使用 Anthropic API（端点/兼容性细表/自动映射）：<https://api-docs.deepseek.com/zh-cn/guides/anthropic_api>
- 思考模式（开关/三档/映射表/采样参数限制）：<https://api-docs.deepseek.com/zh-cn/guides/thinking_mode>
- 图像理解（vision-exp/三种传图/token 换算/限制）：<https://api-docs.deepseek.com/zh-cn/guides/vision>
- 上下文硬盘缓存：<https://api-docs.deepseek.com/zh-cn/guides/kv_cache>
- Files API：<https://api-docs.deepseek.com/zh-cn/guides/files_api>
- 工具调用（思考模式下的多轮调用）：<https://api-docs.deepseek.com/zh-cn/guides/tool_calls>
- 限速与隔离（并发/扩容/保活/user_id）：<https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit>
- 错误码：<https://api-docs.deepseek.com/zh-cn/quick_start/error_codes>
- Token 用量计算（含图片计算器）：<https://api-docs.deepseek.com/zh-cn/quick_start/token_usage>
- 更新日志（模型版本与基准）：<https://api-docs.deepseek.com/zh-cn/updates>
- V4 发布公告：<https://api-docs.deepseek.com/zh-cn/news/news260424>
- V4-Pro 正式版公告：<https://api-docs.deepseek.com/zh-cn/news/news260813>
- Vision-Exp 发布公告：<https://api-docs.deepseek.com/zh-cn/news/news260821>
- 开放平台（API Key/充值/用量）：<https://platform.deepseek.com>
- DeepSeek-V4 技术报告：<https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf>

> 领域作业规程说明：第 9、10 章中 BMC 工程实践与 A 股交易规则/口径清单（T+1、分板块涨跌停幅度、复权、报告期 vs 公告日、point-in-time 等）为领域专家经验层，不出自上述两个文档站；使用前请按实际标的与最新交易所规则校验（新股、退市整理期等特殊证券的规则随监管调整）。
