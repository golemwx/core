# Golem Core 使用说明


Golem Core 是一个**协议服务端程序，启动后通过 HTTP 接口提供扫码登录、收发消息、联系人与群聊管理等能力，并可将新消息实时推送给你的业务系统。

- 单文件二进制，无需安装运行环境，不依赖 MySQL、Redis、消息队列
- 一个进程对应一个账号，多账号请在不同目录各启动一个进程
- 所有数据保存在程序所在工作目录的 `data` 文件夹内
- 完整接口文档：<https://golem.apifox.cn>


> [!WARNING]
> **免责声明**
>
> 1. 本项目为个人研究学习项目，仅供技术研究与学习交流，请遵守所在地法律法规，勿用于任何商业用途或非法行为。
> 2. 本项目与任何第三方公司及其产品不存在隶属、合作或授权关系，涉及的名称与商标归各自权利人所有。
> 3. 使用本项目可能违反第三方平台的用户协议，由此产生的账号限制、封禁、数据丢失等后果由使用者自行承担。
> 4. 禁止将本项目用于发送骚扰或垃圾信息、批量营销、侵犯他人隐私、诈骗等损害他人权益的行为。
> 5. 本项目按“现状”提供，不附带任何明示或暗示的保证，作者不对使用或无法使用本项目造成的任何直接或间接损失承担责任。
> 6. 作者保留随时修改、停止更新或停止分发本项目的权利，无需另行通知。
> 7. 下载、运行或以任何方式使用本项目，即视为您已阅读并同意以上全部内容，并自行承担使用带来的风险与后果；如不同意，请立即停止使用并删除。

> [!IMPORTANT]
> 使用本项目存在封号风险，请勿使用大号直接登录。
>
> 使用小号登录后，先养养号，风控期间不要进行高危操作。


## 下载

在 [Releases](https://github.com/golemwx/core/releases) 页面按系统和架构选择文件，文件名格式为 `golem-core-<版本>-<系统>-<架构>`：

| 系统 | 架构 | 文件名示例 |
| --- | --- | --- |
| Windows | amd64 / arm64 | `golem-core-v1.0.0-windows-amd64.exe` |
| Linux | amd64 / arm64 | `golem-core-v1.0.0-linux-amd64` |
| macOS | amd64 / arm64 | `golem-core-v1.0.0-darwin-arm64` |
| Android | amd64 / arm64 | `golem-core-v1.0.0-android-arm64` |

下载后可用同页面的 `checksums.txt` 校验文件完整性：

```bash
sha256sum -c checksums.txt --ignore-missing
```

## 快速开始

### 1. 启动

建议为程序单独建一个目录，`data` 文件夹会生成在启动时所在的目录下。

Windows：

```powershell
.\golem-core-v1.0.0-windows-amd64.exe
```

Linux / macOS / Android：

```bash
chmod +x golem-core-v1.0.0-linux-amd64
./golem-core-v1.0.0-linux-amd64
```

程序没有命令行参数。首次启动会自动生成默认配置 `data/config.toml` 并直接运行，看到下面的日志即启动成功：

```text
web 服务启动 addr=http://<本机IP>:8080
Golem Core 初始化完成
```

按 `Ctrl+C` 可安全退出。

> 首次启动前可以设置环境变量 `GOLEM_SERVER_TOKEN`，它会被写入配置作为接口访问令牌。该变量只在首次生成配置文件时读取，之后请直接修改配置。

### 2. 扫码登录

请求登录接口，它以 SSE 事件流的形式持续返回登录进度：

```bash
curl -N http://127.0.0.1:8080/api/login
```

收到 `qrcode` 事件后，把其中的链接用浏览器打开即可看到二维码，用手机客户端扫码并确认：

| 事件 | 含义 |
| --- | --- |
| `qrcode` | 二维码图片链接，有效期约 4 分钟 |
| `status` | 扫码状态：`pending` 等待扫码、`scanned` 已扫码、`confirmed` 已确认、`timeout` 超时、`canceled` 已取消、`device_verify` / `pin_verify` 需要安全验证 |
| `verify_url` | 需要人工完成安全验证，用浏览器打开该链接按提示操作 |
| `login` | 登录结果，`success` 表示登录完成 |
| `error` | 登录失败原因 |

登录成功后会话保存在本地，之后重启程序会自动恢复登录，无需重新扫码。

查询登录状态：

```bash
curl http://127.0.0.1:8080/api/login/status
```

退出登录：

```bash
curl -X DELETE http://127.0.0.1:8080/api/login
```

### 3. 发送第一条消息

```bash
curl -X POST http://127.0.0.1:8080/api/message/text \
  -H "Content-Type: application/json" \
  -d '{"receiver":"<接收者ID>","content":"Hello, World!"}'
```

`receiver` 填好友的用户名或群 ID（以 `@chatroom` 结尾）；群内 @ 成员时用 `remind` 传成员 ID，多个以逗号分隔。

其余接口见[接口文档](https://golem.apifox.cn)。

## 接口约定

### 鉴权

配置了 `server.token` 后，所有 `/api` 请求必须携带请求头，否则返回 HTTP 401：

```text
Authorization: Bearer <你的token>
```

未配置时 `/api` 不做鉴权，任何能访问端口的人都可以操作你的账号。**对外网开放端口时务必设置 token**，或把 `server.host` 改为 `127.0.0.1` 只允许本机访问。

### 响应格式

业务接口的 HTTP 状态码恒为 200，结果以 `code` 区分：

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

| code | 含义 |
| --- | --- |
| `0` | 成功 |
| `-1` | 失败，`message` 为失败原因 |
| 其他 | 协议返回的错误码，`message` 以 `协议响应异常：` 开头 |

### 请求体大小上限

| 类型 | 上限 |
| --- | --- |
| 普通 JSON 请求 | 8 MB |
| 语音 | 16 MB |
| 图片、表情、头像 | 32 MB |
| 图文动态（`/api/moments`） | 300 MB |
| 视频、文件 | 2 GB |

超过上限返回 HTTP 413。

## 接收消息

在配置中启用推送渠道后，新消息、联系人变更等事件会实时推送出来。三种渠道可以任意组合：

| 渠道 | 方向 | 用法 |
| --- | --- | --- |
| `webhook` | 程序主动 POST 到你的地址 | 配置 `webhook_url`，可选 `webhook_token` |
| `websocket` | 你来连接，或程序主动连接你 | 连接 `GET /api/message/sync`；或配置 `websocket_url` 让程序主动连出 |
| `sse` | 你来连接 | 连接 `GET /api/message/sync` |

配置示例：

```toml
[push]
channels = ["webhook", "sse"]
webhook_url = "https://example.com/golem/events"
webhook_token = "webhook-secret"
```

用 SSE 订阅：

```bash
curl -N -H "Authorization: Bearer <你的token>" http://127.0.0.1:8080/api/message/sync
```

所有渠道的数据格式一致：

```json
{
  "type": "sync::new_message",
  "data": {}
}
```

| type | 含义 |
| --- | --- |
| `sync::new_message` | 新消息 |
| `sync::modify_contact` | 联系人或群信息变更 |
| `sync::delete_contact` | 联系人删除 |
| `sync::modify_user_info` | 本人资料变更 |
| `sync::modify_user_avatar` | 本人头像变更 |
| `sync::function_switch` | 功能开关变更 |
| `sync::user_info_extend` | 本人扩展资料 |
| `sync::unknown_command` | 未识别的同步指令 |

使用时请注意：

- 订阅接口需要已登录；没有启用对应渠道时返回 HTTP 503
- Webhook 只把 2xx 视为成功，超时时间 10 秒，失败不重试，不跟随重定向
- 程序不保存离线消息，断线期间的消息不会补发，也不保证恰好推送一次
- SSE 每 15 秒发送一次心跳；客户端消费过慢、积压超过 100 条时连接会被断开
- 浏览器原生的 `EventSource` 和 `WebSocket` 无法携带 `Authorization` 请求头，配置了 token 时请在服务端订阅
- 默认忽略 10 秒之前的历史消息，可通过 `core.ignore_history` 与 `core.ignore_timeout` 调整

## 配置

配置文件固定为 `data/config.toml`，有两种修改方式：

- **改文件**：先停止程序，修改后再启动
- **调接口**：通过管理接口在线修改，程序会自动重启生效，见[管理接口](#管理接口)

### [core] 核心

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `license` | 空 | 许可证，留空时自动获取 |
| `print_banner` | `true` | 启动时是否打印版本信息 |
| `ignore_history` | `true` | 是否忽略历史消息 |
| `ignore_timeout` | `10` | 早于多少秒的消息视为历史消息 |
| `random_delay` | `true` | 发送消息前是否加随机延时 |
| `random_delay_min` | `500` | 随机延时最小值，单位毫秒 |
| `random_delay_max` | `3000` | 随机延时最大值，单位毫秒 |
| `random_delay_strategy` | `pending` | `every` 每条消息都延时；`pending` 只在消息排队时延时 |
| `custom_info` | 空 | `golem::info` 回复末尾附带的自定义信息，最长 1024 字符 |
| `qrcode_api` | `https://api.qrtool.cn/?text=` | 生成登录二维码图片的服务地址 |

### [server] HTTP 服务

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `enable` | `true` | 是否启用 HTTP 服务 |
| `host` | `0.0.0.0` | 监听地址，仅本机访问可改为 `127.0.0.1` |
| `port` | `8080` | 监听端口 |
| `token` | 空 | 接口访问令牌，同时作为管理接口令牌 |

### [device] 设备

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `type` | `mac` | 模拟的设备类型，可选 `mac`、`win` |
| `id` | 自动生成 | 设备 ID，留空时沿用已有 ID 或自动生成 |
| `name` | 自动生成 | 设备名称，显示在手机客户端的登录设备列表中 |

> 修改设备配置后，原有的设备与会话文件会被改名备份，需要重新扫码登录。

### [proxy] 代理

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `url` | 空 | 代理地址，留空为直连 |

支持 `http`、`https`、`socks5`、`socks5h`，例如：

```toml
[proxy]
url = "socks5://127.0.0.1:1080"
```

- 需要认证时写成 `协议://用户名:密码@主机:端口`，用户名和密码中的特殊字符需做百分号编码
- HTTP 代理必须支持 `CONNECT` 方法
- 代理只作用于与服务器之间的连接和媒体文件传输；Webhook、WebSocket 推送与许可校验不走代理
- 代理地址无效时程序拒绝启动；代理连接失败时不会自动回退到直连
- 该代理配置仅作为异地登录时代理，最好是提供你所在地同市的代理，以降低风控风险

### [push] 推送

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `channels` | 空 | 启用的渠道，可选 `webhook`、`websocket`、`sse`，空数组为关闭 |
| `webhook_url` | 空 | Webhook 接收地址 |
| `webhook_token` | 空 | Webhook 请求携带的令牌 |
| `websocket_url` | 空 | 程序主动连接的 WebSocket 地址，`ws://` 或 `wss://` |
| `websocket_token` | 空 | 主动连接时携带的令牌 |

### [storage] 存储

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `encrypt` | `false` | 是否加密保存会话与设备数据 |
| `encrypt_key` | 空 | 加密密钥，`encrypt = true` 时必填 |

> 密钥丢失后已加密的数据无法恢复，只能重新扫码登录。

### [log] 日志

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `level` | `info` | 日志级别：`trace`、`debug`、`info`、`warn`、`error` |
| `output` | `both` | 输出位置：`console`、`file`、`both`、`none` |
| `ansi` | `true` | 控制台是否输出颜色 |
| `max_size` | `100` | 单个日志文件大小上限，单位 MB |
| `max_age` | `7` | 日志保留天数 |
| `max_backups` | `10` | 保留的历史日志文件数 |
| `compress` | `gzip` | 历史日志压缩方式：`none`、`gzip`、`zstd` |

日志文件位于 `data/logs/golem_core.log`。

## 管理接口

管理接口用于在线查看和修改配置，路径在 `/admin` 下，必须携带管理令牌：

```text
Authorization: Bearer <管理令牌>
```

管理令牌的来源：

- 配置了 `server.token`：管理令牌就是它
- 未配置：程序每次启动时生成一个临时令牌，登录后向该账号发送 `golem::admin` 获取。每次启动只回复一次，重启后重新生成

查看当前配置：

```bash
curl -H "Authorization: Bearer <管理令牌>" http://127.0.0.1:8080/admin/config
```

修改配置时只需提交要改的字段，其余保持不变：

```bash
curl -X POST http://127.0.0.1:8080/admin/config \
  -H "Authorization: Bearer <管理令牌>" \
  -H "Content-Type: application/json" \
  -d '{"push":{"channels":["sse"]},"log":{"level":"debug"}}'
```

配置有变化时返回 `data.restarting = true`，程序随即自动重启并应用新配置。

## 内置指令

登录后，向机器人账号发送以下文本消息会得到自动回复，可用来确认程序是否在线：

| 指令 | 回复内容 | 谁可以触发 |
| --- | --- | --- |
| `golem::ping` | `pong` | 任何人 |
| `golem::info` | 运行状态、启动与登录时间、自定义信息 | 任何人 |
| `golem::about` | 版本、构建时间、许可等级与到期时间 | 任何人 |
| `golem::admin` | 临时管理令牌 | 任何人 |

> `golem::info` 任何人都能触发，自定义信息里不要放敏感内容。
> 
> `golem::admin` 任何人都能触发，但每次启动只回复一次随机token，回复之后或配置server.token之后不再响应，请及时获取临时token并修改。

## 许可证

- 许可证绑定登录账号。`core.license` 留空时，程序会自动获取并写入配置，通常无需手动填写
- 本机时间与许可服务时间相差超过 1 分钟时程序拒绝启动，请保持系统时间准确
- 许可按等级开放功能模块，调用未开放的模块会返回“无权限”错误。当前账号的等级与到期时间可发送 `golem::about` 查看
- 程序在启动时和每天 0 点检查许可：到期前 5 天内自动尝试续期；已过期时拉取一次新许可后退出，重新启动即可

## 数据与备份

`data` 目录的内容：

| 路径 | 内容 |
| --- | --- |
| `data/config.toml` | 配置文件 |
| `data/session.json` | 登录会话 |
| `data/device.json` | 设备信息 |
| `data/logs/` | 日志 |

启用存储加密后，会话与设备文件的后缀为 `.info`。

- **迁移或备份**：停止程序后整体复制 `data` 目录，在新位置启动后会尝试恢复登录
- **不要泄露**：会话文件等同于你的登录凭证，不要发送给他人或上传到公开位置
- **不要多开**：同一份 `data` 目录不要同时被多个进程使用

## 可选依赖

发送视频时如果系统中装有 FFmpeg（`ffmpeg` 与 `ffprobe` 在 `PATH` 中），程序会优先用它解析视频信息，兼容性更好；未安装时使用内置解析，不影响基本使用。

## 常见问题

**启动时提示“本地时间与许可服务时间差超过1分钟”**

系统时间不准，开启系统的自动对时后重试。

**启动时提示“web 服务监听失败”**

端口被占用，修改 `server.port` 或关闭占用该端口的程序。

**启动时提示“代理配置无效”**

检查 `proxy.url` 的协议和格式，或清空它改为直连。

**接口返回 HTTP 401**

配置了 `server.token` 但请求没有携带，或携带的值不一致。请求头格式为 `Authorization: Bearer <token>`。

**接口返回“没登录呢~”**

尚未登录或登录已失效，重新走一遍扫码登录。

**获取二维码时提示“只支持单账号，不能再登录了哦~”**

当前进程已有账号在线。要换号请先退出登录；要多账号请在另一个目录再启动一个进程，并改用不同的端口。

**提示“用户 [xxx] 无许可权限”或“无权限: 某模块”**

当前账号没有许可，或许可等级不包含该功能模块。

**订阅 `/api/message/sync` 返回 HTTP 503**

没有启用对应的推送渠道，在 `push.channels` 中加入 `sse` 或 `websocket`。

**macOS 提示无法打开或已损坏**

二进制未经过苹果签名，执行以下命令解除隔离后再启动：

```bash
xattr -d com.apple.quarantine ./golem-core-v1.0.0-darwin-arm64
```

**忘记了 `server.token`**

停止程序，打开 `data/config.toml` 查看或修改 `[server]` 下的 `token`。
