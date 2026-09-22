<div align="center">

# B-Panel

**运行在手机上的脚本任务管理面板**

[官网](https://241793.github.io/B-Panel/site/) · [下载](https://github.com/241793/B-Panel/releases) · [反馈](https://github.com/241793/B-Panel/issues)

在 Android 上运行 JavaScript / Python 脚本：定时调度、环境变量、订阅拉取、依赖管理，
并内置本地 HTTP API（与 [青龙面板](https://github.com/whyour/qinglong) v2.21.0 接口兼容，
第三方青龙客户端可直接接入）与网页管理面板。

还可以让 **AI 助手**接手操作 —— 对话即可查任务、读日志、改脚本。

</div>

---

## 功能

### 脚本与运行时
- **双运行时**：nodejs-mobile（Node 18）+ Chaquopy（Python 3.11），进程内执行
- **真并发**：多个脚本同时跑，Python 不再是「一次只能一个」，日志互不串台
- **代码编辑器**：语法高亮（CodeMirror 5，本地打包无 CDN）、行号、查找替换、
  固定高度内部滚动 + 自定义滚动条
- **调试控制台**：选 JS / Python，写临时代码立即运行，实时回显输出，可随时停止
- **脚本来源**：新建、URL 导入（自动经 GitHub 代理加速）、本地文件导入、外部「用本应用打开」
- **内置依赖**：`pycryptodome` 等含 C 扩展的常用包已在构建期打入，无需安装

### AI 助手
- **MCP 协议**：对外暴露 **34 个工具**（任务 / 脚本 / 变量 / 日志 / 配置），
  任何支持 MCP 的客户端（Claude Code、Claude Desktop 等）都能接入
- **模型账户**：OpenAI 兼容（DeepSeek / 智谱 / 月之暗面 / Ollama…）+ Anthropic 原生；
  支持拉取模型列表、测试连接
- **聊天机器人**：微信（openclaw）/ QQ 机器人 / 本地调试三种渠道，扫码即可登录；
  可设指令前缀、写权限开关、工具白名单
- **App 内试用**：直接对话验证配置是否可用
- 独立开关，默认关闭；与 MCP 可分别启停

### 通知渠道
- **Bark / Telegram / 企业微信机器人 / PushPlus** 四种渠道
- 可配多个账号、单独启停、加备注区分；任务级开关决定哪些任务推送

### 定时任务
- cron 调度（20s 轮询 + 15min 闹钟兜底）、错过执行 5 分钟宽限补跑
- 失败自动重试、任务级超时、并发数可配
- **运行中自动置顶**：点启动后浮到列表最前，跑完自动落回原位
- 缺依赖自动识别并尝试安装，结果写回日志
- 批量启停/删除、标签、置顶排序、左滑操作、任务级推送开关

### 环境变量
- 批量启停、值掩码显示、拖拽排序、备注、多账号 `&` 分隔并发语义

### 订阅
- GitHub / Gitee / GitLab 仓库定时拉取（zip 归档），GitHub 自动多代理源切换
- **单脚本 URL** 订阅：直接订一个脚本直链
- 白名单 / 黑名单 / 依赖文件筛选，自动创建/清理定时任务

### 网页面板（浏览器远程管理）
- 手机浏览器或同网段电脑打开 `http://<ip>:5700/panel`
- **13 个页签**，侧边栏分组可折叠：仪表盘 / 定时任务 / 日志（执行 + 系统）/
  环境变量 / 脚本管理（含依赖）/ 订阅 / API 令牌 / 通知渠道 / AI 助手（含 MCP）/ 设置
- **脚本下载**：单个下载，或勾选多个打包成 zip（保留目录结构）
- **API 令牌**：可签发带 MCP 权限的令牌供外部 AI 客户端使用
- **账号密码登录**（App 内设置），支持记住密码自动登录
- 日志弹窗查看（搜索 / 状态筛选 / 复制 / 下载）、脚本运行实时输出
- 完整备份导出与文件导入

### 数据备份与迁移
- 一键导出 **tar.gz 完整备份**：脚本文件 + 任务 + 变量 + 订阅 + API 令牌 + 配置
- 换机导入，重复项可选「跳过」或「覆盖」
- App 内分享备份文件 / 网页面板直接下载、上传

### 安全
- 本地 API 与面板全量 **JWT 登录鉴权**（含本机访问）
- **AI 工具集不含任何删除类操作** —— 即便开写权限也删不掉任务或脚本
- MCP 端点独立鉴权 + scope 校验，写操作需显式授权（`mcp:write`）
- 登录防爆破（连续失败锁定）、安全响应头（nosniff / XFO / no-referrer）
- 改密后旧令牌立即失效；备份上传限 64MB
- Release 包启用 **R8 混淆 + 资源收缩**

### 应用内更新
- 关于页检查 GitHub Releases，按设备架构匹配 APK，下载后拉起系统安装器
- 公告与更新历史从仓库 notify.json 拉取，失败显示原因明细

## 系统要求

- Android 8.0+（API 26）
- arm64-v8a / armeabi-v7a / x86_64

## 下载

从 [Releases](https://github.com/241793/B-Panel/releases) 下载对应架构的 APK：

| 文件 | 适用 |
|---|---|
| `B-Panel-arm64-<版本>.apk` | 绝大多数现代手机（推荐） |
| `B-Panel-armeabi-<版本>.apk` | 老旧 32 位设备 |
| `B-Panel-x86_64-<版本>.apk` | 模拟器 / x86 平板 |



## 文档

- [`docs/API_COMPAT.md`](docs/API_COMPAT.md) —— 本地 HTTP API 接口对照表
- App 内「侧边栏 → APP 使用文档」—— 功能说明 / 接口手册 / 常见问题

## 致谢

- 接口设计参考 [whyour/qinglong](https://github.com/whyour/qinglong)（GPL-3.0）
- 脚本运行基于 [nodejs-mobile](https://github.com/nodejs-mobile/nodejs-mobile) 与 [Chaquopy](https://chaquo.com/chaquopy/)

- 网页面板编辑器基于 [CodeMirror 5](https://codemirror.net/)

