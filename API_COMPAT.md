# 青龙接口对照表（对齐 v2.21.0）

本文档逐条对照**本安卓端实现**与**青龙面板 v2.21.0** 的 API 契约。

所有端点均以源码核实为准，参考路径：
`github.com/whyour/qinglong` tag `v2.21.0`，`back/api/*.ts`。

---

## 通用约定

### 响应结构

青龙（`back/data/index.ts`）：

```ts
export type ResponseType<T> = { code: number; data?: T; message?: string };
```

| 场景 | 响应 |
|---|---|
| 成功 | `{ "code": 200, "data": ... }` |
| 业务失败 | `{ "code": 400, "message": "…" }` |
| 未认证 | `{ "code": 401, "message": "Token 已失效" }`（HTTP 401） |
| 未找到 | `{ "code": 404, "message": "数据不存在" }` |

> 注意：青龙的业务失败也用 HTTP 200 + body 内的 code，本实现保持一致。

### 认证

对齐 `back/loaders/express.ts`：

- `POST /api/user/login` 返回 JWT
- 请求头 `Authorization: Bearer <token>`（**仅此一种**；查询参数 `?token=` 已移除，防 token 泄露到日志/Referer）
- `/api/open/apps` 签发的长期 token 同样被接受（带 scopes）
**本安卓端收紧**：**所有来源（含本机回环、局域网私网）都需鉴权**，不再按来源放行。
原因：用户可能做内网穿透，把 127.0.0.1 映射到公网，此时「本机来源」在服务端看来与任意请求无异，
来源免鉴权等于门户大开。网页面板首次访问需用 App 内设置的账号密码登录。

判定来源时使用 `call.request.origin.remoteAddress`（对端字面 IP），**不要**用 `origin.remoteHost`：
Ktor 3.x 的 CIO 引擎里 `remoteHost` 走 `InetSocketAddress.getHostName()`，会做反向 DNS；
一旦解析成主机名，私网正则匹配失败，本机 / 局域网请求会被误判为不可信并返回 `Token 已失效`。

### 命名风格

青龙的 sequelize **未启用** `underscored`，API 直接输出模型字段名，
因此 JSON 中蛇形与驼峰**混用**。本实现逐字对齐：

| 蛇形 | 驼峰 |
|---|---|
| `log_path` | `isDisabled` |
| `last_running_time` | `isPinned` |
| `last_execution_time` | `isSystem` |
| `extra_schedules` | `allowMultipleInstances` |
| `sub_id` | — |

### 状态枚举

青龙 `back/data/cron.ts`：

```ts
export enum CrontabStatus { 'running' = 0, 'idle' = 1, 'disabled' = 2, 'queued' = 3 }
```

> 注意：0 是「运行中」，不是「禁用」。直觉假设会出错。

青龙 `back/data/env.ts`：

```ts
export enum EnvStatus { 'normal', 'disabled' }   // 0 / 1
```

---

## 定时任务 `/api/crons`

### 列表响应：`data` 是**裸数组**

```json
{ "code": 200, "data": [ { "id": 0, "name": "...", "command": "task x.py", ... } ] }
```

**不要包成 `{data:[...], total:N}`。**

> 排查记录：青龙 `back/services/cron.ts` 的 `Crontabs()` 返回
> `{data, total}`，但那是**服务层**的返回值，**路由层只取数组部分**发给客户端。
> 早期实现照搬了服务层结构，导致第三方客户端的 `data` 拿到对象而非数组，
> 按数组解析时抛异常（客户端表现为「无法解析 json」，列表空着）。


| 方法 | 路径 | 说明 | 状态 |
|---|---|---|---|
| GET | `/` | 列表（支持 `searchValue` / `limit` / `offset`） | ✅ |
| POST | `/` | 新建 | ✅ |
| PUT | `/` | 更新 | ✅ |
| DELETE | `/` | 删除（见下方「批量端点的 body 形态」） | ✅ |
| PUT | `/enable` | 批量启用 | ✅ |
| PUT | `/disable` | 批量禁用 | ✅ |
| PUT | `/run` | 批量运行（**本 App 扩展：`data` 返回 `{started, skipped}`**，见下） | ✅ |
| PUT | `/stop` | 批量停止 | ✅ |
| PUT | `/pin` | 置顶 | ✅ |
| PUT | `/unpin` | 取消置顶 | ✅ |
| PUT | `/status` | 状态统计 | ✅ |
| GET | `/labels` | 标签列表 | ✅ |
| POST | `/import` | 导入任务 | ✅ |
| GET | `/:id` | 详情 | ✅ |
| GET | `/:id/log` | 最新日志 | ✅ |
| GET | `/:id/logs` | 历史日志 | ✅ |
| GET | `/:id/instances` | 运行实例 | ✅ |
| PUT | `/:id/instances/:instanceId/stop` | 停止实例 | ✅ |

### 批量端点的 body 形态

`DELETE /api/crons`、`PUT /api/crons/{enable,disable,run,stop,pin,unpin}`
（以及 `/api/envs` 的同名端点）**同时接受**以下任一形态：

```
[1,2]                    青龙原生（Joi.array().items(Joi.number())）
{"ids":[1,2]}            本端既有形态
{"id":1}                 单个数字
{"id":"1"}               字符串 id（第三方客户端形态）
{"id":"1","enable":true} 带多余字段（被忽略）
1 / "1"                  裸标量
```

**为什么宽松**：第三方客户端（如 `work.master.qinglongapp`）的 Dart 侧 `id`
属性是 **String 类型**，且**逐条调用**（不发数组）。它发的是 `{"id":"1"}`，
若只按青龙原生的数字数组解析会静默失败。

**非数字 id 会被拒绝**：若传入本端从未签发过的 id（如青龙原生的 UUID），
服务端**记日志后返回 400**，不做任何猜测匹配 —— 猜测可能命中别的实体，
导致 `/disable` 误停任务、`/delete` 误删数据。

### `PUT /api/crons/run` 的两点扩展

**① 手动运行不受「禁用」限制**，与 App 内点「启动」一致 ——
命令走 `CronScheduler.fireManually` 而非 `fire`（后者对 `isDisabled=1`
直接 return）。跑完状态回到 `disabled(2)` 而非 `idle(1)`。

> ⚠️ **不要改回 `fire`**。曾经就是用它，导致禁用任务被静默跳过，而面板
> 仍弹「已提交运行」—— 用户点了毫无反应，也查不到原因（日志为空）。

**② `data` 返回 `{started, skipped}`**（青龙原版这里是 `data: null`）：

```json
{ "code": 200, "data": { "started": 2, "skipped": 1 } }
```

调用方可用它给出准确提示。`skipped` 的原因只有两种：任务不存在，
或正在运行且 `allowMultipleInstances != 1`。

---

## 环境变量 `/api/envs`

| 方法 | 路径 | 说明 | 状态 |
|---|---|---|---|
| GET | `/` | 列表（按 position 升序） | ✅ |
| POST | `/` | 新建 | ✅ |
| PUT | `/` | 更新 | ✅ |
| DELETE | `/` | 删除 | ✅ |
| GET | `/:id` | 详情 | ✅ |
| PUT | `/:id` | 更新 | ✅ |
| DELETE | `/:id` | 删除 | ✅ |
| PUT | `/:id/move` | 拖拽排序 | ✅ |
| PUT | `/enable` | 批量启用 | ✅ |
| PUT | `/disable` | 批量禁用 | ✅ |
| PUT | `/pin` `/unpin` | 置顶 | ✅ |
| GET | `/name` | 按名查询 | ✅ |
| POST | `/upload` | 批量导入 | ✅ |
| GET | `/labels` | 标签 | ✅ |

**排序语义**：青龙用 `position` 实现拖拽排序
（`initPosition=4500000000000000`，`stepPosition=10000000000`）。
本实现采用「取相邻中点」，中点碰撞时退化为整表重排。

---

## 脚本管理 `/api/scripts`

| 方法 | 路径 | 说明 | 状态 |
|---|---|---|---|
| GET | `/` | 目录树 / 列表（`?path=` 指定子目录） | ✅ |
| GET | `/:file` | 读取文件 | ✅ |
| POST | `/` | 新建 | ✅ |
| PUT | `/` | 保存 | ✅ |
| DELETE | `/` | 删除 | ✅ |
| POST | `/run` | 运行（**异步**，返回 `{executionId, logFile, status}`） | ✅ |
| GET | `/run/status` | 查询运行状态（`?executionId=` → `running/done/failed/none`） | ➕ |
| PUT | `/stop` | 停止（body `{executionId}` 停单个；省略则停全部） | ✅ |
| PUT | `/rename` | 重命名 | ✅ |
| GET | `/detail` | 详情 | ✅ |
| GET | `/download` | 下载 | ✅ |

**安全**：`GET /:file` 接受相对路径，实现中会过滤 `..` 并校验解析后的
规范路径仍在 `scripts/` 内，防止目录穿越。

### 脚本树节点的字段

基准是青龙 `v2.21.0` 的 `IFile`（`back/config/util.ts`）：

```
{ title, key, type, parent, createTime, size?, children? }
```

本端**额外输出**几个字段供第三方客户端兜底（经确认不破坏按 `IFile` 解析的客户端）：

| 字段 | 说明 |
|---|---|
| `path` | 与 `key` 同值。第三方客户端的脚本模型有此属性（非青龙字段） |
| `created` | `createTime` 的别名，同值 |
| `isDirectory` | 布尔，等价于 `type == "directory"` |
| `extension` | 文件扩展名（小写、不含点）；目录为 `null` |

### `GET /api/scripts/files`（非青龙端点，为第三方客户端补）

**不是青龙原生端点**（`v2.21.0` 的 `back/api/script.ts` 没有它），但实测
第三方客户端 `work.master.qinglongapp` **拉脚本列表时请求的正是它**
（`GET /api/scripts/files?t=<时间戳>`），而非 `GET /api/scripts`。

返回结构与 `GET /api/scripts` 相同（IFile 树）。

> 排查记录：此前没有这个路由，请求被 `GET /{file...}` 通配接住，把 `files`
> 当成文件名去读 → 返回「脚本不存在」，客户端界面上完全显示不出脚本。
> 定义时必须排在 `/{file...}` **之前**，否则仍会被通配抢先匹配。

**目录的 `size` 为 `null`**（青龙只在文件节点设 `size`）。注意 [ApiJson] 的
`explicitNulls` 是开的，故目录会输出 `"size":null` 而非省略该键 ——
**判断节点类型请用 `type`，不要看 `size`**。

**运行语义**：`POST /run` 是**异步**的——立即返回 `executionId` 与日志文件
绝对路径，脚本在后台执行。调用方轮询 `GET /run/status?executionId=` 判断
是否结束（`running` → `done`/`failed`），并用 `GET /logs/file?path=<logFile>`
读取实时输出。这样长脚本不会撞上客户端的请求超时。
`GET /run/status` 与 `/stop` 的 `executionId` 是 Web 面板扩展，非青龙原生。

---

## 系统 `/api/system`

| 方法 | 路径 | 说明 | 状态 |
|---|---|---|---|
| GET | `/` | 系统信息 | ✅ |
| PUT | `/config` | 保存配置 | ✅ |
| PUT | `/reload` | 重载调度 | ✅ |
| GET | `/log` | 面板日志 | ✅ |
| PUT | `/log-remove` | 日志清理 | ✅（本端扩展） |
| GET | `/data/export` | 数据导出：返回**完整备份 tar.gz**（脚本/任务/变量/订阅/令牌/配置） | ✅ |
| POST | `/data/import` | 数据导入：multipart 上传（`?mode=skip\|overwrite`）或 JSON 粘贴 | ✅ |

**备份格式**：App 自有 tar.gz（`manifest.json` + `scripts/`），**非青龙格式**
（青龙表名 `crontabs`/列名 snake_case 与本端不兼容，故不做互通）。
导入冲突策略由 `?mode=` 决定，默认 `skip`（跳过重复，安全）。
账号密码不在备份内（导入后需在新机重设）。

---

## 配置 `/api/config`

| 方法 | 路径 | 对应青龙配置项 |
|---|---|---|
| GET | `/` | 返回完整 `SystemConfigInfo` |
| PUT | `/save` | 批量保存 |
| PUT | `/lang` | `lang` |
| PUT | `/timezone` | `timezone` |
| PUT | `/panel-title` | `panelTitle` |
| PUT | `/node-mirror` | `nodeMirror` |
| PUT | `/python-mirror` | `pythonMirror` |
| PUT | `/linux-mirror` | `linuxMirror` |
| PUT | `/dependence-proxy` | `dependenceProxy` |
| PUT | `/global-ssh-key` | `globalSshKey` |
| PUT | `/cron-concurrency` | `cronConcurrency`（并发上限） |
| PUT | `/log-remove-frequency` | `logRemoveFrequency`（日志保留天数） |
| PUT | `/{key}` | 通用单项写入 |

### 安卓扩展配置项

| 键 | 说明 | 默认 |
|---|---|---|
| `apiEnabled` | 本地 API 服务开关 | `true` |
| `apiPort` | 本地 API 端口 | `5700` |
| `autoStart` | 开机自启 | `true` |

---

## 用户与授权

| 方法 | 路径 | 说明 | 状态 |
|---|---|---|---|
| POST | `/api/user/init` | 初始化（首次设置账号） | ✅ |
| POST | `/api/user/login` | 登录，返回 `{token, tokenType, expiration}` | ✅ |
| POST | `/api/user/logout` | 登出 | ✅ |
| GET | `/api/user` | 当前用户信息 | ✅ |
| GET | `/api/user/login-log` | 登录日志 | ✅ |
| GET | `/api/open/apps` | 应用列表 | ✅ |
| POST | `/api/open/apps` | 创建应用（签发 token + scopes） | ✅ |
| PUT | `/api/open/apps` | 更新 | ✅ |
| DELETE | `/api/open/apps` | 删除 | ✅ |

**scopes**（青龙 `back/data/open.ts`）：
`envs` `crons` `configs` `scripts` `logs` `system` `dashboard`

---

## 本端扩展端点

这些路径**不在青龙 v2.21.0 的契约内**，是 B-Panel 为网页面板与 App 新增的能力。
第三方青龙客户端不需要它们；但改动这些路由时**不受「字节兼容」约束**，
只需保证 App 与网页面板两侧一致。

### 基础

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/health` | 健康检查，返回 `{code:200,data:{status:"ok"}}` |

### 登录日志（审计）

登录日志的表结构与 `status` 取值见 `LoginLogEntity` 的 KDoc；此处只列端点。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/user/login-log` | 最近 200 条，时间倒序。`status`：`0` 成功 / `1` 失败 / `2` 被防爆破锁定（`2` 为本端扩展） |
| DELETE | `/user/login-log` | 清空（青龙的审计只读，此为扩展） |

**取 IP 只认 `remoteAddress`**，不解析 `X-Forwarded-For` / `X-Real-IP` —— 那些头
客户端可随意填写，采信会让审计价值归零。

### IP 黑名单 `/api/user/blocked-ips`

用于挡住持续爆破的**来源 IP**。防爆破锁定只针对账号维度，一个攻击 IP 打满
5 次会把**所有人**一起锁住；要按来源拒绝，需要这张黑名单。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/user/blocked-ips` | 列表 `{rows:[{id,ip,reason,createdAt}], hint}` |
| POST | `/user/blocked-ips` | 封禁，body `{ip, reason?}`。重复封返回 **400**（前端据此提示「已在黑名单中」） |
| POST | `/user/blocked-ips/remove` | 解封，body `{ip}`。用 POST 而非 DELETE：面板发 JSON body 更顺手 |
| DELETE | `/user/blocked-ips` | 清空黑名单 |

**拦截语义**：黑名单在 `authPlugin` 里**先于 `isPublicPath` 判定** —— 被封 IP 连
`/api/user/login`、`/panel` 这些公开路径也进不去，否则封禁形同虚设。

**取 IP 只认 `remoteAddress`**，理由同上。

**回环（`127.x` / `::1`）豁免且不可封** —— 防自锁：若本机被误封，
App 内的黑名单管理界面（走同一个 HTTP 服务）也会打不开，等于没有补救入口。

**表名是 `blocked_ips_v2`**：本工程早期有过一张同名旧表 `blocked_ips`
（因一次 schema 事故被移除），新名彻底避开与残留表的冲突。

### 使用者（AI 权限）`/api/agent/users`

把 AI 给**非管理员**用时，「谁能用」的总闸。陌生人通过机器人发消息会被
自动登记，但 `aiEnabled` / `skillEnabled` **默认关闭**，用不了任何功能。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/agent/users` | 使用者列表（管理员排前面） |
| PUT | `/agent/users` | 无 `id` = **手动添加**（`{platform, externalId, nickname?, role?, aiEnabled?, skillEnabled?, dailyQuota?}`）；带 `id` = 按字段更新（**只改提及的字段**） |
| DELETE | `/agent/users` | 删除，body `{id}` |

**权限公式**（唯一真相在 `AgentUserRepository.permissionFor`）：

```
isAdmin      = role == 1
aiAllowed    = isAdmin || aiEnabled == 1
skillAllowed = isAdmin || skillEnabled == 1
allowWrite   = isAdmin && bot.allowWrite == 1     ← 普通用户恒 false
```

**管理员来源**：`PUT` 不带 `id` 的「手动添加」。这是体系里**唯一**的管理员入口 ——
不设任何自动或自助升级途径（否则「第一个发消息的人」就成了主人）。

**工具可见性不做每用户白名单**（粗粒度）：沿用机器人自身的 `allowedTools`，
用户维度只决定「能不能用」「能不能写」「额度」。

### AI 工具集（MCP 与 Agent 共用，共 48 个）

| 前缀 | 数量 | 覆盖 |
|---|---|---|
| `cron_` | 11 | 任务的增删改查、启停、运行、停止、置顶、标签、日志 |
| `env_` | 7 | 环境变量（凭据默认掩码，要明文须显式 `reveal=true`） |
| `script_` | 10 | 脚本读写、运行、停止、重命名、建目录 |
| `subscription_` | 3 | 订阅列表 / 拉取 / 启停 |
| `config_` | 2 | 读全部配置 / 写单项 |
| `log_` | 1 | 按路径读日志 |
| `bucket_` | 4 | 数据桶（**按用户隔离**，见下） |
| `plugin_` | 5 | 插件列表 / 详情+源码 / 启停 / 创建 / 写入 |
| `user_` | 5 | 使用者列表 / 角色 / AI / Skill / 额度 |

**全表无 `delete`**（`bucket_delete` 是唯一例外，见数据桶一节）。理由见
`McpToolRegistry` 类 KDoc 第 3 条：删除不可逆，AI 一次误判就没了。
要删请在 App 里手动操作（那里有二次确认）。

#### 两条特别的安全约束

**1. 插件写入 = 任意代码执行**
`plugin_write` / `plugin_create` 写的是**会被执行**的代码（消息触发时在
host 进程里跑）。与 `script_write` 同一类能力，故都标 `write = true`。
而 `plugin_write` 覆盖已有文件**必须显式传 `overwrite=true`** ——
插件内容一旦被覆盖就没了，不该由「一次手滑的调用」决定。

**2. 使用者权限的两条保护**

| 保护 | 为什么 |
|---|---|
| **禁止改自己** | 否则「帮我把自己设成管理员」就绕过了整个权限模型 |
| **禁止降级最后一个管理员** | 否则系统锁死，没人能再进「使用者」页改权限 |

「改自己」按 `ToolCallContext` 的 `(platform, externalId)` 比对目标行；
MCP 直连（无用户概念）时跳过 —— 那种场景调用者是本机 AI 客户端（已持有效令牌）。

### 数据桶 `/api/buckets`

插件 / Skill / 规则脚本的通用键值存储（`桶 → 键 → JSON 值`）。
进程内插件直接调 `BucketRepository`，不走 HTTP；这里只暴露管理能力。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/buckets` | 桶列表（含条数与启用状态） |
| POST | `/buckets` | 新建桶，body `{name, remark?}` |
| GET | `/buckets/{name}` | 桶内**全部**条目（含已禁用 —— 管理界面需要能重新启用） |
| PUT | `/buckets/{name}` | 写一条，body `{key, value, disabled?}`。`value` 是任意 JSON |
| DELETE | `/buckets/{name}` | 删一条，body `{key}` |

**桶名 / 键名白名单**：只允许 `[A-Za-z0-9_.-]`、长度 1..64、且不得以 `sys_` 开头。
理由：插件会拿桶名拼存储路径，放任任意字符等于把路径穿越（`../`）装进去。

**禁用语义**：桶禁用 → 其下条目对**读取方**不可见；条目禁用 → 单条不可见。
但管理接口（`GET /buckets/{name}`）能看到全部 —— 否则用户禁用后再也找不回来。

**按用户隔离**：插件存用户数据时用 `BucketRepository.userBucket(platform, id)`
统一命名（`user_<platform>_<id>`），**不要自行拼接** —— 一处拼错就是数据串号。

**AI 也能操作数据桶**（工具 `bucket_list` / `bucket_get` / `bucket_set` / `bucket_delete`）：

| 场景 | 落到哪个桶 |
|---|---|
| 聊天机器人里**不传** `bucket` | 调用者自己的隔离桶 `user_<platform>_<id>` |
| 聊天机器人里**传** `bucket` | 仅**管理员**放行；普通用户拒绝 |
| MCP 直连（Claude Desktop 等） | 共享桶 `mcp_shared`（无用户概念） |

隔离靠 `ToolCallContext`（`platform` / `externalId` / `isAdmin`）实现 —— 它由
`AgentService` 从消息来源填、经 `AgentToolBridge` 传给工具 handler。
MCP 路径没有发送者，传 `ToolCallContext.MCP`（`hasUser=false`）。

`bucket_delete` 是本工具集**唯一**的删除类工具（原则见 `McpToolRegistry` 类 KDoc
第 3 条「不暴露删除类」）。理由：它删的是插件/AI 的键值状态，重写即可恢复，
不属「不可逆操作」；且没有它，AI 存错数据就永远清不掉。

### 插件 `/api/plugins`（管理）与 `/api/plugin/sock`（回连）

按 atm 规则实现的文件式插件（`.py` / `.js`，放在 `data/plugins/`）。

**插件头**（写进文件顶部注释，两种风格都认）：

```
# [title:插件名]                    /  /** @title: 插件名 */
# [class: 工具类]                   /  @class:
# [platform: qq,wx,tg]             /  @platform:
# [rule: ^zdm(.*)$]                /  @rule:            ← 可多行 = 多条规则
# [description: ...]               /  @description:
# [admin: true]                    /  @admin:
# [priority: 0]                    /  @priority:
# [version: 1.0.0]                 /  @version:
# [imType:qq,wx]                   /  @imType:          ← 白名单
# [param: {"required":true,"key":"桶.键","bool":false,"placeholder":"x","name":"x","desc":"x"}]
```

**也认 B-BOT 的模块属性写法**（参考项目 `plugins/` 下大量使用）：

```python
__version__ = "1.0.0"
__description__ = "说明"
__admin__ = True
__pattern__ = r"^你好$"          # 也可 ["a", "b"]
__rule_type__ = "keyword"        # regex / keyword / fullmatch
__priority__ = 5
__param__ = {"key": "桶.键", ...}  # **可重复多行**，每行一条
```

> 两处都写时**头注释优先**（atm 是主线，模块属性是适配层）——
> 否则用户改头注释却不生效，很难排查。
>
> `__param__` 多行赋值必须**扫源码**收集：Python 语义下连续赋值只留最后一条。
> `__pattern__` 的列表形式按**顶层逗号**切分，否则 `\d{1,3}` 会被切成两半。

#### 插件形态（三种都支持）

| 形态 | 判据 | 入口 |
|---|---|---|
| atm 模块级脚本 | 默认 | `runpy` 跑 `__main__` |
| B-BOT `register()` | 顶层有 `register` | 调 `register(mw)`，再跑注册的 handler |
| atm_context | `__main__` + 从 `get_current_context()` 取 `(message, middleware)` | 先设 context 再 `runpy` |

探测用 `ast` **静态读源码，不 import**（import 会真的执行模块顶层代码）。
`register` 优先于 `__main__`。

`middleware` 是**包**（不是单文件）—— 因为 `from middleware.atm_context import ...`
要求如此。包内：`_impl.py`（atm 协议，原样）+ `atm_context.py` + `bbot.py`（B-BOT 薄适配）。

#### 匹配方式三态

| `rule_type` | 语义 | 说明 |
|---|---|---|
| `regex`（默认） | `containsMatchIn` | |
| `keyword` | 子串包含（大小写不敏感） | **绝不走正则** —— 用户填的 `1.5` 不该匹配 `1x5` |
| `fullmatch`（别名 `exact`） | 完全相等（忽略首尾空白） | 同上 |

#### 手动规则（不写文件）

命中即回固定文本，存数据桶 `rule_engine` / `manual_rules`（与参考项目同构）。
字段：`name` / `pattern` / `reply` / `type` / `priority` / `description` / `isAdmin`。

**匹配顺序：文件插件先于手动规则** —— 插件能力更强、用户写文件时意图更明确。
手动规则是「没有插件管这条消息」时的兜底。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/plugins` | 列表。**规则/描述/参数一律现读文件**（元数据不落库） |
| POST | `/plugins` | 新建（服务端生成带完整插件头的模板），body `{path, lang}` |
| PUT | `/plugins` | 启停，body `{path, enabled}` |
| DELETE | `/plugins` | 删除，body `{path}` |
| GET | `/plugins/{path}/params` | 该插件参数的当前值（值存在数据桶里） |

#### 回连端点 `/api/plugin/sock/{action}`

插件是**独立进程**，通过它回复消息、读写数据桶。**这是本功能安全上最要紧的一处**
——它等价于进程内控制通道。三道闸：

1. **必须来自回环**（`remoteAddress` 是 `127.x` / `::1`），否则 403
2. **必须带 `X-Plugin-Token`** —— 每次执行随机生成的一次性令牌，
   执行结束即失效；只存内存、不落库、常量时间比较
3. `isPublicPath` 里放行该路径（插件带不了 JWT），**但第 1、2 道在路由内自校验**

action 全量对齐 atm 的 `_atm_framework_dispatch`：`sendText` / `sendImage` /
`sendVoice` / `sendVideo` / `sendFile` / `at_user` / `at_all` / `input` /
`listen` / `recallMessage` / `getUserID` / `getUserName` / `getUserAvatarUrl` /
`getChatID` / `getChatName` / `getImtype` / `isAdmin` / `getMessage` /
`getMessageID` / `bucketGet` / `bucketSet` / `bucketDel` / `bucketAll` /
`bucketAllKeys` / `bucketKeys` / `get` / `set` / `delete` / `push` /
`notifyMasters` / `getActiveImtypes`。

**数据桶按发送者隔离**（对齐 atm）：`bucketGet/Set/Del` 的键会加 `发送者id:` 前缀，
避免「A 存的东西 B 看到」。读时带兜底：先试带前缀的键，找不到再试裸键
（让「用裸键存全局配置」的写法也能工作）。

#### `input` / `listen` 的两个参数

| 参数 | 语义 | 默认 |
|---|---|---|
| `forGroup` | true = 群内**任意成员**发言都算回答；false = 只接受**发起者本人** | false |
| `recallDuration` | 收到输入后延迟该毫秒数**撤回那条消息**（阅后即焚，适合密码/验证码） | 0（不撤回） |

两者由 host 随等待状态存下来（`PendingInput`），投递时按 `forGroup` 过滤、
把 `recallDuration` 透给 `AgentBotManager` 去延迟撤回。

> `forGroup=false` 时**只认发起者**这一点很重要：否则群里别人随便说句话
> 就会被当成插件要的答案，用户的输入被「截胡」且毫无提示。
>
> 撤回靠 `BotChannel.recall()`，**默认实现返回 false**（渠道不支持）——
> 与 `sendRich` 的降级策略一致：渠道能力限制不该让插件整体失败。
> Echo（本地测试渠道）实现了它，便于验证这条链路。

**支付类四个 action**（`create_payment` / `query_order` /
`process_payment_notification` / `get_payment_config`）返回 **501** ——
本端未接入支付通道，如实占位而不是假装成功。

#### 插件运行记录

`PluginRunLog`（**内存**环形缓冲，每个插件留 20 条）记录每次执行的
时间/触发消息/发送者/退出码/耗时/错误，供 App 的插件展开面板展示。
与 `AppLogRecorder`（按天归档的**文本**日志）各司其职：后者面向完整排查，
前者面向「这个插件最近怎么样」的结构化速览。**不落库** —— 重启即失效，
那是运行态诊断信息。

#### 消息分派顺序

```
① 该会话是否正挂在插件的「等待输入」上？ → 交给插件
② 插件按 rule 正则匹配（priority 降序；adminOnly / imTypes 过滤）→ 命中即执行
③ AI：对话中（自动续接）或带前缀触发
④ 都不匹配 → 不回复
```

**AI 是前缀式会话**：发 `<前缀> 内容` 进入对话，之后**自动续接**（不必每次带前缀），
发 `q` 或空闲超时（默认 5 分钟）退出。

### 通知渠道 `/api/notify/channels`

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/notify/channels` | 列表，**同时返回渠道定义 `definitions`**（含 `fields[]`，前端据此数据驱动生成表单） |
| POST | `/notify/channels` | 新建（`channel` 必须是已知渠道，否则 400） |
| PUT | `/notify/channels` | 编辑。**`channel` 不可改**；缺 `config` 时保留原值 |
| DELETE | `/notify/channels` | 删除（body 为 **id 数组** `[1,2]`） |
| PUT | `/notify/channels/enable` | 批量启用（body 为 id 数组） |
| PUT | `/notify/channels/disable` | 批量禁用（body 为 id 数组） |

**为什么返回 `definitions`**：新增渠道时若 web 端硬编码一份字段表，两边迟早漂移。
返回定义让 App 的编辑表单与 web 表单**共用同一份 `NotifyChannels.ALL`**。

### MCP `/api/mcp`
| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/mcp/status` | `{enabled: bool}` |
| PUT | `/mcp/enabled` | body `{"value":"true"/"false"}` |

语义明确的端点（而非复用 `/config/mcpEnabled`），便于将来返回更多状态。

### AI 助手 `/api/agent`

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/agent/status` | `{agentEnabled, mcpEnabled}` |
| PUT | `/agent/enabled` · `/agent/mcp-enabled` | 开关，body `{"value":"true"}` |
| GET | `/agent/providers` | 模型账户列表，**`apiKey` 掩码返回** |
| POST · PUT · DELETE | `/agent/providers` | 增删改 |
| POST | `/agent/providers/models` | 拉模型列表 → `{models:[...]}` |
| POST | `/agent/providers/test` | 测试连接 → `{ok, reply, ms}` 或 `{ok:false, error}` |
| GET | `/agent/bots` | 机器人列表 |
| POST · PUT · DELETE | `/agent/bots` | 增删改（含 `triggerEnabled` / `allowWrite` / `allowedTools`） |
| POST | `/agent/scan/qr` | 取扫码二维码 → `{ticket, qrImage}`（**服务端渲染成 PNG 的 data URI**） |
| GET | `/agent/scan/status?ticket=…` | 轮询 → `{loggedIn, token?}` / `{loggedIn:false, waiting:true}` |

#### agent 扩展（知识库 / Skill / 计划任务 / 外部 MCP / Token 统计）

这些是**非青龙端点**，供面板的「agent 扩展」页使用。
容器未初始化时它们**不挂载**（契约测试不起 `AppContainer`），其余端点照常。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET · POST · DELETE | `/agent/knowledge` | 知识库条目增删改查 |
| GET · POST · PUT · DELETE | `/agent/skills` | Skill 列表 / 新建 / 启停 / 卸载 |
| POST | `/agent/skills/install` | 安装（`{url}` 或 `{name, content}`） |
| GET · PUT | `/agent/skills/file` | 读 / 写 skill 包内文件（`?id=&path=`） |
| GET · POST · DELETE | `/agent/plan-tasks` | agent 计划任务 |
| GET · POST · DELETE | `/agent/mcp-servers` | **外部** MCP 服务器（⚠️ 与本机对外提供的「MCP 服务」方向相反） |
| POST | `/agent/mcp-servers/refresh` | 手动重新拉取工具列表 |
| GET · DELETE | `/agent/token-stats` | token 消耗统计（按天 + 按模型）/ 清空 |

#### AI 可调的工具（与 MCP 工具集**同一份**）

工具集由 `McpToolRegistry.tools` 统一提供 —— MCP 直连与 AI 对话共用同一份，
不存在两个真相来源。除青龙兼容的 cron/env/script/log/config 等，
另有几类「agent 扩展」工具：

| 工具 | 作用 | 约束 |
|---|---|---|
| `http_request` | 发 HTTP 请求（curl 类） | 只允许 http(s)；超时 30s；响应截断 6000 字 |
| `file_list` / `file_read` / `file_write` / `file_delete` | 文件读写 | **限 `data/workspace/`**；拒 `..`、绝对路径、隐藏文件；不删目录 |
| `doc_generate` | 生成 Word / Excel / PPT | 同上沙箱；内容以**参数**传给 Python（不拼源码） |
| `skill_run_script` / `skill_<id>` | 跑 skill 包内脚本 / 读说明书 | 路径限该 skill 目录内 |
| `image_generate` | 按描述生成图片（用配好的画图模型账户） | 结果落 `workspace/images/` |

#### 多模态（图片收发）

**收图**：渠道把附件解析成 `IncomingAttachment` → `AttachmentStore` 下载到
`workspace/inbox/` → 转 base64 送模型。

| 渠道 | 能否拿到下载地址 |
|---|---|
| Telegram | ✅ 解析 `photo`/`voice`/`document`，调 `getFile` 换 `file_path` |
| QQ | ⚠️ `attachments[].url`（带时效签名，需尽快下载） |
| openclaw | ⚠️ 见 `WxClawChannel.extractItemUrl`（字段名因版本而异，尽力解析） |
| Echo | 不涉及 |

**模型是否支持视觉**：`VisionSupport.supportsVision(model, override)`
—— 模型名特征匹配（`gpt-4o` / `claud-*` / `qwen-vl` 等）+ 用户手动覆盖。
**未知模型默认判为不支持**（猜错的代价不对称：误判会让对话直接失败，
漏判只是用户手动开一下）。

**不支持时**：直接回绝并说明原因，**不发图、不烧 token**。

**覆盖值存桶**（不迁移）：`ProviderOverrides`，桶 `agent` / 键 `provider_overrides`，
形态 `{"<providerId>": true}`。**只存用户显式指定过的** —— 没存 = 自动识别，
避免模型清单更新后老记录僵在旧结论上。

**历史回放**：`AgentMessageEntity.toolPayload` 存**相对路径**
（`{"attachments":[{"path":"inbox/x.jpg","mime":"image/jpeg"}]}`），
回放时从磁盘重读 —— 不存 base64 是因为每行消息膨胀几十 KB 而每轮都要读最近 20 条。

**发图**：`ImageExtractor` 从模型回复里提取图片（markdown 语法 / 裸链接），
走渠道 `sendRich(OutgoingMessage.Image(url))`；提取后把 markdown 换成 `[图片]`，
裸链接保留（用户可能想复制）。

标 `skill = true` 的受「使用者页的 Skill 开关」管控（可给普通用户单独禁用）；
`http_request` / `file_write` / `file_delete` / `doc_generate` 另标 `write = true`。

> **Python 侧内置依赖**（构建期打进 APK —— 含 C 扩展的**只能**这样装）：
> `requests` / `httpx` / `pycryptodome` / `lxml` / `Pillow` /
> `python-docx` / `python-pptx` / `openpyxl` / `XlsxWriter`。
> 运行时只能装**纯 Python** wheel（`DependencyManager`），C 扩展包装不了。

#### 掩码 Key 不覆盖真 Key（**最要紧的一条**）

`GET /agent/providers` 返回的 `apiKey` 是掩码（`sk-••••3f2a`）。
但前端编辑时会把**整个对象回传**。若服务端不做判断，用户「只改个名字保存」
就会把真 key 覆盖成那串掩码 —— 之后所有 AI 调用都因鉴权失败而挂掉，用户却看不出原因。

故 `applyMaskedKey(submitted, existing)` 的判据是：

| 提交值 | 结果 |
|---|---|
| 空 / 全空白 | 保留原值 |
| **含 `••••`** | 保留原值 |
| 其它 | 覆盖（显式换 key 生效） |

用「含掩码符号」而非「等于掩码结果」——后者要求知道原值，而原值正是要保护的东西。
掩码规则与 App 卡片一致：长度 ≤10 显示全掩码 `••••`，否则 `前4 + •••• + 后4`。

#### 扫码为何在服务端渲染

openclaw 的扫码接口返回的是一段**文本**（不是图片）。App 端用项目自带的 zxing
（`QrCode.generateBitmap`）本地渲染；web 端若在 JS 里再造一个 QR 编码器是重复劳动，
故服务端直接生成 PNG 的 base64。会话是**无状态**的：服务端不存 ticket，
前端持有并回传。

---

## 与青龙的差异（必须知情）

### 1. 主键 `_id` 的取值

青龙的主键是**字符串 UUID**（sequelize 的 `_id`），本端是**自增 Long**。
两个字段都会输出：

| 字段 | 青龙 | 本端 |
|---|---|---|
| `_id` | UUID 字符串，如 `"a1b2c3d4-..."` | **数字的字符串形式**，如 `"1"` |
| `id` | 数字 | 数字 |

**第三方客户端读 `_id`**（它的 `id` 属性是 String 类型），再把它回传给动作端点。
本端输出数字字符串，故这条链路闭环自洽：

```
列表返回 {"_id":"1", "id":1}
  → 客户端取 _id = "1"
  → 回传 {"id":"1"}
  → 服务端宽松解析为数字 1 → 命中主键
```

客户端不会察觉与青龙的差异（它只关心中间的一致性，不关心值的形态）。

### 2. 调度精度

| | 青龙（服务器） | 本安卓端 |
|---|---|---|
| 进程存活 | 永远存活 | 可能被系统/用户杀死 |
| 最小精度 | 秒级 | 分钟级 |
| 进程被杀 | 不存在 | 依赖 AlarmManager 兜底（15 分钟周期） |

安卓的 Doze 模式、国产 ROM 的自启动拦截都会导致任务延迟或漏执行。
**这是平台限制，不是实现缺陷。** 设置页提供电池优化白名单引导。

### 3. 脚本执行方式

| | 青龙 | 本安卓端 |
|---|---|---|
| 机制 | `cross-spawn` 子进程 | 进程内嵌入运行时 |
| JS | 系统 node | nodejs-mobile（Node 18）常驻实例 + worker_threads |
| Python | 系统 python3 | Chaquopy（Python 3.11）进程内解释器 |

**停止语义差异**：
- JS：通过 `worker.terminate()` 可真正中断任意脚本（含死循环）
- Python：CPython 无法从外部安全中断，采用**协作式停止**
  （脚本可调用 `ql_should_stop()` 主动退出）；不配合的脚本需等超时

### 4. 环境差异

- 无 shell，故不提供 `/api/system/command-run`（该端点已移除，路径返回 404）
- 依赖可运行时安装：Python 装纯 Python wheel（py3-none-any），Node 装 npm tarball；
  含 C 扩展的 Python 包需构建期内置（常用包如 pycryptodome 已内置）
- 订阅已实现（GitHub / Gitee / GitLab 定时拉取，自动建任务）；无 SSH

---

## 验证方式

```bash
# 本仓库无 Gradle wrapper，用便携版 Gradle（见 CLAUDE.md）
# 契约测试（Robolectric + 内存 DAO，无需模拟器）
export JAVA_HOME="C:\Program Files\Java\jdk-17.0.2"
/d/tmp_build/gradle/bin/gradle.bat testDebugUnitTest

# 构建（镜像到 ASCII 路径 + 出包）
./build.sh           # debug
./build.sh release   # 签名混淆包
```

契约测试 `QlApiContractTest` 逐条断言上表中的路径、方法与响应结构。
