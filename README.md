# wechat-bridge

**把微信接到你的个人 AI 助手。**

微信消息经腾讯官方 [ClawBot / iLink](https://ilinkai.weixin.qq.com/) 进入本机
`wechat-bridge`，转成 JSON 落盘到 `inbox/`；你的 Agent 读取后，用 `send` 发回微信。

```
+--------+   iLink getupdates   +--------------+   inbox/ JSON   +---------+
|        | -------------------> |              | --------------> |         |
| WeChat |                      | wechat-bridge|                 |  Agent  |
|        | <------------------- |              | <-------------- |         |
+--------+   iLink sendmessage  +--------------+  outbox/pending +---------+
```

## 功能

- **双向收发**：收文本、图片、语音、文件、视频；回发文本、图片、文件、视频。语音附带微信转写文本。
- **只出站**：走 iLink 长轮询，不对外开端口。
- **媒体加解密**：微信 CDN 媒体 AES-128-ECB 解密下载；发送时加密上传。
- **指数退避**：getupdates 失败 2 秒重试，连续 3 次失败后退避到 30 秒。
- **文本分片**：超 2000 字自动切多条发出。
- **二维码自动刷新**：`wait-login` 遇到过期自动重取，最多 3 次。

## 环境要求

- Python 3.8 及以上（建议 3.10+），Linux 或 macOS
- 能访问公网（只需出站 HTTPS，无需开放入站端口）
- 一部装有微信的手机（用于扫码登录）

## 安装

```bash
# 1. 建议用虚拟环境
python3 -m venv .venv
source .venv/bin/activate

# 2. 安装依赖并安装本包
pip install qrcode pycryptodome
pip install -e .

# 3. 跑通加解密单元测试
python3 tests/test_crypto.py
```

数据目录默认在 `<你的工作区>/wechat-bridge`（例如 `~/workspace/wechat-bridge`），
可用环境变量 `WECHAT_BRIDGE_DIR` 覆盖。**注意：该目录下会生成含登录凭据和
聊天记录的文件，绝不要把它提交进 git**（本仓库自带 `.gitignore` 已排除）。

## 登录（只需做一次）

```bash
wechat-bridge login
```

终端会告诉你二维码图片保存在哪里（例如 `<数据目录>/qrcode.png`）。
**二维码有效期只有约 2 分钟**，建议先拿起手机打开微信扫一扫、说"准备好了"
再运行 `login`。

然后在另一个终端启动扫码轮询：

```bash
wechat-bridge wait-login
```

手机上点"确认登录"后，会话（含 `bot_token` 等凭据）会以 600 权限保存在
`<数据目录>/state/session.json`。登录成功后原二维码文件会自动删除。

## 启动常驻服务

```bash
# 先在微信里给 Bot 发一条消息（官方限制：必须用户先发消息，
# Bot 拿到 context_token 之后才能回复）
nohup <项目目录>/.venv/bin/wechat-bridge serve >> <数据目录>/state/serve.log 2>&1 &
```

检查状态：

```bash
wechat-bridge status
```

`status` 会显示登录态、serve 是否运行、inbox 消息数、待发回复数。
`bot_token` 只显示掩码，绝不会完整打印。

## 发送测试

```bash
# 发文本（收件人默认取最近一次会话的缓存，可用 --to 覆盖）
wechat-bridge send --text "你好，这是测试消息"

# 发图片 / 文件 / 视频（--text 会作为配文先发）
wechat-bridge send --image ./a.jpg --text "见图"
wechat-bridge send --file ./report.pdf
wechat-bridge send --video ./b.mp4
```

## 看门狗（可选但推荐）

`watchdog.sh` 每 5 分钟检查一次：serve 挂了就拉起来；检测到登录失效
（`state/NEEDS_RELOGIN`）只通知一次。用 cron 或 systemd 定时调用：

```bash
# cron 示例：每 5 分钟跑一次
*/5 * * * * /bin/bash <项目目录>/watchdog.sh >> <数据目录>/state/watchdog.log 2>&1
```

看门狗只做这三件事，不读聊天内容、不发消息：

1. `state/session.json` 不存在 → 静默退出（视为未登录）
2. `NEEDS_RELOGIN` 存在 → 只告警一次，等你重新扫码
3. serve 进程不存在 → 用 `.venv/bin/wechat-bridge serve` 拉起，并把真实
   Python 进程的 pid 写入 `state/serve.pid`

## 给 AI 助手用的消息流

`serve` 把收到的每条消息写成一个 JSON 到 `inbox/<message_id>.json`：

```json
{
  "message_id": "<消息唯一 ID>",
  "text": "消息文本（图片/视频/文件会记为 [图片]/[视频]/[文件]）",
  "from_user_id": "<发信人微信 ID>",
  "context_token": "<回这条消息必须带上的上下文 token>",
  "create_time_ms": 1234567890000,
  "media_paths": ["<数据目录>/media/<日期>/<message_id>_image.jpg"]
}
```

AI 助手要回复时，把回复写成 JSON 放进 `outbox/pending/<message_id>.json`
（**文件名用 message_id，保证幂等**）：

```json
{
  "to_user_id": "<收件人微信 ID>",
  "text": "回复文本",
  "context_token": "<来自 inbox 那条消息的 context_token>",
  "message_id": "<所回复消息的 message_id>"
}
```

发图片/文件时再加 `"media_path": "<本地文件绝对路径>"`，`text` 会作为配文先发。
`serve` 每轮循环会把 `pending/` 里的任务调 `sendmessage` 发出，
成功后移到 `outbox/sent/`。

完整约定见 [SKILL.md](SKILL.md)（去重、防重复回复、限流注意事项）。

## 目录结构

```
<数据目录>/
  state/            # session.json(600 权限)、serve.pid、NEEDS_RELOGIN、日志
  inbox/            # 收到的消息，每条一个 JSON
  outbox/pending/   # 待发送的回复任务
  outbox/sent/      # 已发送的回复任务
  media/<日期>/     # 下载的图片/视频/文件
```

## 限制（微信官方）

- 仅一对一私聊，不支持群聊
- 助手不能主动发起会话，用户必须先发过消息
- 连续约 10 条不回复会被限流，用户回一句重置
- 二维码有效期约 2 分钟；`wait-login` 最多自动刷新 3 次

## 常见问题

**Q: `serve` 日志里偶尔出现 `getupdates 出错：Remote end closed connection`？**
A: 微信服务端偶发直接断开连接，属正常现象，2 秒后会自动重试；
连续失败 3 次后退避到 30 秒。

**Q: `status` 显示需要重新扫码（NEEDS_RELOGIN）？**
A: `bot_token` 失效且无刷新机制。重新跑一遍 `login` /
`wait-login` 即可，标记会自动清除。

**Q: `send` 没报错但对方收不到？**
A: 检查三点：`sendmessage` 的 `from_user_id` 必须传空字符串；
`context_token` 必须用对方那条消息里的值原样回传；
顶层必须带 `base_info{channel_version, bot_agent}`。本项目的
`ilink.py` 已处理这些细节，不要自己拼请求。

**Q: `ret` / `errcode` 返回 -14？**
A: 可能是限流而非 token 过期。`serve` 会先暂停 5 分钟重试一次，
仍失败才写 `NEEDS_RELOGIN` 标记。

**Q: 收到消息但 `inbox/` 里没有？**
A: 群消息（带 `group_id`）和非机主微信 ID 发来的消息会被忽略，
只处理登录时绑定的那个微信号。

**Q: 媒体解密失败？**
A: 视频必须用 `encrypt_query_param` 拼的 CDN 地址下载，`full_url`
的数据解不开；`aes_key` 有 4 种格式，`crypto.py` 会严格解析，
实在解不开时自动尝试多种密钥派生组合。

## 安全须知

- `state/session.json` 含 `bot_token`，权限 600，绝不进 git、
  绝不贴到聊天记录或发给任何人
- 日志和 `status` 输出里 token 均为脱敏显示
- `media/`、`inbox/`、`outbox/` 都是你的私人聊天数据，
  备份时注意妥善保管

## 致谢

媒体收发与分层思路参考 [lawrencewzen/wx-bot](https://github.com/lawrencewzen/wx-bot)
（Go 实现）；协议文档见
[tencent/openclaw-weixin](https://github.com/tencent/openclaw-weixin/blob/HEAD/docs/protocol_zh_CN.md)。

## License

MIT License，见 [LICENSE](LICENSE)。
