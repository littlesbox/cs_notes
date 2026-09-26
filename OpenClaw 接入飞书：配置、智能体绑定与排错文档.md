# OpenClaw 接入飞书：配置、智能体绑定与排错文档

> 适用版本：OpenClaw 2026.6.11+  
> 适用场景：飞书企业自建应用 + OpenClaw 机器人 + 专用智能体  
> 说明：以下命令均在服务器终端执行。

---

## 1. 接入飞书

### 1.1 前置准备

- 确认 OpenClaw 版本不低于 `2026.5.29`。
- 检查版本：

```bash
openclaw --version
```

- 如版本过低，升级：

```bash
openclaw update
```

### 1.2 创建飞书应用

1. 访问飞书开放平台：
   
   - 国内版：https://open.feishu.cn
   
   - 国际版：https://open.larksuite.com
2. 创建一个 **企业自建应用**。
3. 为应用添加 **机器人** 能力。
4. 在“凭证与基础信息”页面记录：
   
   - **App ID**
   
   - **App Secret**

### 1.3 配置权限与事件

在飞书开放平台完成以下配置：

#### 权限管理

至少开通以下权限：

| 权限标识                     | 用途             |
| ------------------------ | -------------- |
| `im:message`             | 获取与发送单聊、群组消息   |
| `im:message:send_as_bot` | 以应用身份发送消息      |
| `cardkit:card:write`     | 创建和更新消息卡片，推荐开通 |

根据日志提示，可能还会涉及：

- `im:message.reactions:write_only`
- `im:message:send`

#### 事件订阅

- 添加事件：`im.message.receive_v1`
- 推荐使用 **WebSocket 长连接** 模式，无需公网 URL。
- 如使用 Webhook 模式，需要填写公网可访问的回调地址。

#### 发布应用

权限配置完成后，必须在“版本管理与发布”中：

1. 创建新版本。
2. 提交发布。
3. 等待审核通过。

> 注意：权限变更后，必须发布新版本才会正式生效。

### 1.4 在 OpenClaw 中配置飞书频道

推荐使用配置向导：

```bash
openclaw channels login --channel feishu
```

可选择：

- **Manual setup**：手动粘贴 App ID 和 App Secret。
- **QR setup**：扫码创建机器人。若国内飞书 App 无反应，改用手动设置。

也可手动配置：

```bash
openclaw config set channels.feishu.enabled true --json
openclaw config set channels.feishu.appId "你的AppID"
openclaw config set channels.feishu.appSecret "你的AppSecret"
```

配置完成后重启 Gateway：

```bash
openclaw gateway restart
```

### 1.5 验证

```bash
openclaw channels status --probe
```

在飞书中向机器人发私信，或将其拉入群聊并 @提及，测试是否正常回复。

---

## 2. 创建飞书专用智能体

### 2.1 创建智能体

为飞书渠道创建独立工作区：

```bash
openclaw agents add feishu-agent-lei \
  --workspace ~/.openclaw/workspace-feishu-agent-lei
```

- `feishu-agent-lei`：智能体名称，可自定义。
- `--workspace`：独立工作区目录，用于隔离记忆、配置和工具。

### 2.2 绑定飞书渠道

将飞书渠道路由到该智能体：

```bash
openclaw agents bind --agent feishu-agent-lei --bind feishu
```

如需匹配所有飞书账户：

```bash
openclaw agents bind --agent feishu-agent-lei --bind feishu:*
```

如需只匹配默认账户：

```bash
openclaw agents bind --agent feishu-agent-lei --bind feishu:default
```

### 2.3 验证绑定

```bash
openclaw agents list --bindings
```

应能看到类似：

```text
- feishu-agent-lei
  Routing rules: 1
  Routing rules:
    - feishu
```

### 2.4 设置智能体身份

方式一：命令行设置

```bash
openclaw agents set-identity --agent feishu-agent-lei \
  --name "飞书助手" \
  --emoji "🚀" \
  --avatar avatars/feishu-assistant.png
```

方式二：通过 `IDENTITY.md`

在工作区根目录创建 `IDENTITY.md`，然后导入：

```bash
openclaw agents set-identity \
  --workspace ~/.openclaw/workspace-feishu-agent-lei \
  --from-identity
```

### 2.5 精细路由

如只允许特定群组或私聊路由到该智能体：

```bash
openclaw agents bind --agent feishu-agent-lei \
  --bind feishu:default \
  --match '{"peer": {"kind": "group", "id": "oc_xxxxxxxxxxxxxxxx"}}'
```

`peer.kind` 可取：

- `direct`：私聊
- `group`：群聊
- `channel`：频道

> 绑定不等于授权。路由绑定只决定由哪个智能体处理消息，不会绕过飞书渠道自身的配对、白名单等访问控制。

---

## 3. 发消息收不到回复的排查

按以下顺序排查。

### 3.1 检查飞书开放平台

- 应用是否为企业自建应用。
- 应用是否已发布并审核通过。
- 是否已添加 `im.message.receive_v1` 事件。
- 是否使用 WebSocket 长连接，或 Webhook 地址是否公网可达。
- 是否已开通 `im:message`、`im:message:send_as_bot`、`cardkit:card:write` 等权限。
- OpenClaw 中的 App ID、App Secret 是否与飞书开放平台一致。

### 3.2 检查 OpenClaw 渠道状态

```bash
openclaw gateway status
openclaw gateway restart
openclaw logs --follow
```

发一条测试消息，观察日志是否有新记录。

### 3.3 检查智能体绑定与路由

```bash
openclaw agents list --bindings
```

确认目标智能体已绑定到 `feishu` 或 `feishu:default`。

### 3.4 检查访问控制策略

查看待配对请求：

```bash
openclaw pairing list feishu
```

批准配对：

```bash
openclaw pairing approve feishu <CODE>
```

临时开放私信策略用于测试：

```bash
openclaw config set channels.feishu.dmPolicy open
openclaw gateway restart
```

群聊默认需要 @提及机器人。如使用白名单，需要将群聊 ID 加入 `groupAllowFrom`。

### 3.5 其他检查

- OpenClaw 与飞书插件是否为最新版本。
- 服务器时钟是否与网络时间同步。
- Webhook 模式下，回调 URL 是否公网可访问。

---

## 4. 查看日志确认消息是否到达

### 4.1 基础日志

```bash
openclaw logs --follow
```

### 4.2 过滤飞书日志

```bash
openclaw logs | grep -i -E "feishu|webhook|event|callback"
```

### 4.3 多账号场景

```bash
openclaw logs --channel feishu --account main --follow
```

### 4.4 判断消息是否到达

- 有日志产生，尤其是出现 `im.message.receive_v1`、`received message`，说明消息已到达 OpenClaw。
- 完全没有日志，说明问题在飞书侧或网络连通性。

---

## 5. 典型日志与错误处理

### 5.1 消息已到达

示例：

```text
feishu[default]: received message from ou_*** in oc_*** (p2p)
```

说明：

- 飞书消息已成功到达 OpenClaw。
- 问题不在“飞书 → OpenClaw”接收链路。

### 5.2 消息被路由到智能体

示例：

```text
feishu[default]: dispatching to agent
(session=agent:feishu-agent-lei:feishu:direct:ou_***)
```

说明消息已进入智能体处理流程。

### 5.3 权限错误 99991672

典型错误：

```text
code: 99991672
msg: Access denied. One of the following scopes is required:
[im:message, im:message.reactions:write_only]
```

或：

```text
[im:message:send, im:message, im:message:send_as_bot]
```

原因：飞书应用缺少发送消息或卡片所需权限。

解决：

1. 在飞书开放平台开通：
   
   - `im:message`
   
   - `im:message:send_as_bot`
   
   - `cardkit:card:write`
2. 创建新版本并发布。
3. 重启 Gateway：

```bash
openclaw gateway restart
```

### 5.4 流式卡片失败与降级

日志示例：

```text
streaming start failed; using non-streaming card fallback
```

说明：

- OpenClaw 默认尝试流式卡片回复。
- 流式卡片失败后，会自动降级为非流式卡片。
- 如果最终收到回复，说明降级路径成功。

### 5.5 queuedFinal 会话阻塞

日志示例：

```text
dispatch complete (queuedFinal=true, replies=1)
```

说明：

- `replies=1` 表示有回复被发送。
- `queuedFinal=true` 表示可能存在未完成的最终回复，可能导致会话阻塞。
- 如出现长时间“已读不回”，重启 Gateway：

```bash
openclaw gateway restart
```

### 5.6 增加纯文本降级

为降低卡片发送失败影响，可配置：

```bash
openclaw config set channels.feishu.cardFallbackToText true
```

作用：卡片彻底失败时，尝试以纯文本发送。

---

## 6. 日志敏感信息与安全

### 6.1 通常不会出现的敏感信息

- 飞书 App Secret
- tenant_access_token / app_access_token
- DeepSeek 或其他模型 API Key
- 用户手机号、邮箱、真实姓名
- 系统密码、SSH 密钥

### 6.2 日志中可能包含的信息

| 内容          | 说明         | 敏感程度 |
| ----------- | ---------- | ---- |
| `ou_***`    | 用户 open_id | 中    |
| `oc_***`    | 会话 ID      | 中    |
| 消息正文        | 用户发送的内容    | 中    |
| `cardId`    | 飞书卡片 ID    | 低    |
| `messageId` | 飞书消息 ID    | 低    |
| `cli_***`   | 飞书 App ID  | 低    |

### 6.3 安全建议

- 不要公开粘贴完整原始日志。
- 分享前脱敏：替换 `ou_***`、`oc_***`、`om_***`、`cardId` 等。
- 非调试期降低日志级别，减少消息正文记录。
- 限制 OpenClaw 配置目录权限：

```bash
chmod 700 ~/.openclaw
```

- 如果 App Secret 曾在公开渠道出现，立即在飞书开放平台重置。

---

## 7. 命令速查表

| 操作            | 命令                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------- |
| 查看版本          | `openclaw --version`                                                                      |
| 升级            | `openclaw update`                                                                         |
| 配置飞书频道        | `openclaw channels login --channel feishu`                                                |
| 查看渠道状态        | `openclaw channels status --probe`                                                        |
| 查看 Gateway 状态 | `openclaw gateway status`                                                                 |
| 重启 Gateway    | `openclaw gateway restart`                                                                |
| 实时查看日志        | `openclaw logs --follow`                                                                  |
| 过滤飞书日志        | `openclaw logs \| grep -i -E "feishu\|webhook\|event\|callback"`                          |
| 创建智能体         | `openclaw agents add feishu-agent-lei --workspace ~/.openclaw/workspace-feishu-agent-lei` |
| 绑定飞书渠道        | `openclaw agents bind --agent feishu-agent-lei --bind feishu`                             |
| 查看绑定          | `openclaw agents list --bindings`                                                         |
| 查看配对请求        | `openclaw pairing list feishu`                                                            |
| 批准配对          | `openclaw pairing approve feishu <CODE>`                                                  |
| 开放私信策略        | `openclaw config set channels.feishu.dmPolicy open`                                       |
| 卡片失败降级为文本     | `openclaw config set channels.feishu.cardFallbackToText true`                             |

---

## 8. 总结

接入飞书并创建专用智能体的核心步骤是：

1. 在飞书开放平台创建企业自建应用、开通权限、订阅事件、发布版本。
2. 在 OpenClaw 中配置飞书频道。
3. 创建独立智能体并绑定飞书渠道。
4. 通过日志确认消息是否到达、是否路由到智能体、是否成功发送回复。
5. 遇到 `99991672` 错误时，优先检查飞书权限并发布新版本。
6. 日志中通常没有核心密钥，但包含 open_id、会话 ID 和消息正文，分享前应脱敏。

完成以上配置后，OpenClaw 即可通过飞书渠道与专用智能体稳定交互。

--- 

文档结束。
