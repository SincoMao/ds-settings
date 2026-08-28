# ds-settings

DeepSeek-V4 × Claude Code（Windows CLI）——面向**正确性、完备性、可靠性**的极致化配置。成本与速度不纳入决策考量；模型策略为**主线单旗舰 + 多模态专线**：一切文本推理（含后台辅助任务与全部常规子代理）仅运行 **DeepSeek-V4-Pro**，无备选、无回退；图片输入为 **DeepSeek-V4-Flash-Vision-Exp** 独占能力，经专用 vision 子代理极致承载（图片直读、录屏 ffmpeg 抽帧取证、扫描件按页读、数字原生 PDF 文字层提取走主线），与主线物理隔离。

| 文件 | 说明 |
| --- | --- |
| [`settings.json`](./settings.json) | 部署至 `%USERPROFILE%\.claude\settings.json`，仅需将 `ANTHROPIC_AUTH_TOKEN` 替换为实际的 DeepSeek API Key（已通过官方 JSON Schema 严格校验） |
| [`agents/vision.md`](./agents/vision.md) | 部署至 `%USERPROFILE%\.claude\agents\vision.md`——vision 子代理，多模态链路唯一通道（debug 截图诊断、录屏/视频抽帧取证、图纸理解、K 线/图表读数、扫描件阅读），frontmatter 直指 `deepseek-v4-flash-vision-exp` |
| [`GUIDE.md`](./GUIDE.md) | 配置与作业手册：三类需求（BMC 级超大代码体系开发、A 股持续深度行情分析、多模态证据链）的需求规格与追溯矩阵；参数逐项详解（附权威来源链接）；多模态子代理路由架构与作业手册（图片直读/视频抽帧/PDF 分流）；无人值守双轨制；余额与并发运维；升级 SOP；漂移源清单；两份 CLAUDE.md 模板 |
| [`d-ds.bat`](./d-ds.bat) | **一键部署入口**：终端输入 `d-ds`（需本目录在 PATH）或到本目录双击运行，自动从 `DS_API_KEY` 环境变量回填密钥并部署整套配置，详见下文「一键部署」 |
| [`deploy.ps1`](./deploy.ps1) | 部署逻辑本体（`d-ds.bat` 仅是薄封装；也可带参数直接运行，支持 `-KeyEnvName` / `-ProfileLabel` / `-AgentAction deploy|remove|none`） |

核心设定：effort 锁定 `max`（DeepSeek 映射下 `xhigh` 仅落 `high`，`max` 才是最深档，为此接受 ultracode 自动编排关闭的知情取舍，交互关键词触发的单任务 workflow 不受影响）、1M 上下文全程可用（`[1m]` 后缀 + 窗口预算互补设计：压缩基准 786432 + 输出上限 262144 = 1048576）、多模态经 vision 子代理路由（不占模型别名位、**不设 `CLAUDE_CODE_SUBAGENT_MODEL`** 以保 frontmatter 路由生效——主线粘贴图片会 400，图片/视频/扫描件一律落盘后点名 `@vision`：视频自动抽帧、扫描件按页读、数字原生 PDF 文字层走主线）、内置 WebSearch 原生可用（DeepSeek 服务端执行，零 MCP 依赖）、工具输出与子代理回传按官方上限放宽不截断、三层流式看门狗 + 无人值守重试 + headless 后台无限等待、默认 `bypassPermissions` 全自动（16 条 allow 清单同时是 workflow 子代理的免批通行证）、fullscreen 渲染 + 终端铃声完成通知（无人值守闭环）、`minimumVersion=2.1.246` 将版本门槛固化为启动检查（凭证隔离安全修复）。

## 一键部署（d-ds）

终端输入 `d-ds`（需本目录在 PATH）或双击 `d-ds.bat`，一条命令完成"回填密钥 + 部署 settings.json + 部署 vision 子代理 + 备份原配置"，无需手改文件、无需粘贴密钥。

一次性准备（设完永久生效；也可在「编辑账户的环境变量」界面手动添加）：

```powershell
setx DS_API_KEY "sk-你的 DeepSeek API Key"   # platform.deepseek.com/api_keys

# 将本仓库目录加入用户 PATH（按实际 clone 位置调整，不加 PATH 双击 .bat 亦可）
[Environment]::SetEnvironmentVariable('Path',
  [Environment]::GetEnvironmentVariable('Path','User') + ';D:\Source\ds-settings',
  'User')
```

日常部署：任意终端输入 `d-ds` 回车即可（PATH 变更需新开终端生效）。`deploy.ps1` 自动完成：

1. 读取 `DS_API_KEY`（当前进程 → 用户级注册表，`setx` 设完无需重开终端）；
2. 回填 `ANTHROPIC_AUTH_TOKEN` 占位符，JSON 校验后以 UTF-8 无 BOM 写出，覆盖前自动将现有配置备份为 `.bak-<时间戳>`；
3. 部署 vision 子代理 `agents\vision.md` → `%USERPROFILE%\.claude\agents\vision.md`；
4. 打印端点 / 模型 / 密钥掩码摘要。

部署目标为 `%USERPROFILE%\.claude\settings.json`。已运行的会话不热加载 settings.json，部署后请新开 Claude Code 会话。
