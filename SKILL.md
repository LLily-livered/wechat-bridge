# WeChat Bridge 助手技能

用 wechat-bridge 收发微信消息时，照这份约定做。桥接本身只做协议和落盘，
理解和回复由你完成。

## 收消息

- 轮询 `<数据目录>/inbox/` 目录，新出现的 `<message_id>.json` 就是一条新消息。
- 每条记录含 `message_id`、`text`、`from_user_id`、`context_token`、
  `create_time_ms`，媒体消息另有 `media_paths`（本地已解密文件）。
- 文本直接读 `text`。图片、视频、文件在 `text` 里只记 `[图片]`、`[视频]`、
  `[文件]`，用文件工具打开 `media_paths` 里的本地文件再回应。
- 语音消息如带微信转写文本，会以 `[语音转写]` 前缀出现在 `text` 里，按文本处理。
- 只处理发给你的私聊。桥接已过滤群消息和非机主 ID，你读到的默认可回。

## 发回复

- 把回复写成 JSON，放进 `<数据目录>/outbox/pending/<message_id>.json`。
  文件名必须用所回复消息的 `message_id`，不要用随机名，这是幂等的关键。
- 文本回复格式为 `{"to_user_id", "text", "context_token", "message_id"}`。
  `to_user_id` 和 `context_token` 照抄 inbox 里那条消息的值。
- 发图片、视频或文件时加 `"media_path"` 字段，填本地文件绝对路径。
  `text` 会作为配文先单独发出，再发出媒体。
- 写完 pending 文件就结束。`serve` 会在自己的循环里调 `sendmessage`
  发出，成功后把文件移到 `outbox/sent/`。不要自己调微信接口。
- 超长文本不用你分片，桥接按 2000 字自动切多条。

## 去重

- 维护 `<数据目录>/state/processed.json`，记下已处理过的 `message_id` 列表。
- 每次轮询先读它，跳过已在列表里的消息。处理完一条立刻把它的
  `message_id` 追加进去再去做别的事，避免崩溃重启后重复回复。
- 幂等文件名是第二道保险。同一 `message_id` 的 pending 文件写两次，
  也只会发出去一次。
- `serve` 自身也按 `message_id` 判重，`inbox/` 里不会出现重复文件。

## 限流与会话规则

- 同一轮唤醒里有多条未处理消息时，合并成一条回复，200 字以内。
  `context_token` 用最新那条的。
- 微信官方限制：连续约 10 条发出而用户不回，会被限流。用户回一句即重置。
  因此不要刷屏，能一句说完就一句。
- 用户发"嗯"、"好的"、"行"这类简短确认时，不要复读之前的内容，
  用自然的一句话接住。
- 回复前先读最近的收发记录，结合上下文理解用户最新消息的真实意图，
  不要只看最后一条。

## 故障信号

- `<数据目录>/state/NEEDS_RELOGIN` 文件存在表示 `bot_token` 失效。
  这时不要再写 pending 任务，通知用户重新扫码登录。
  用户重新登录成功后标记会自动清除。
- `wechat-bridge status` 可随时查看登录态和 serve 是否运行，
  `bot_token` 只显示掩码。
- 凭据只存在 `state/session.json`（600 权限）。绝不把 token、
  user_id、context_token 打印到聊天记录、日志或任何发给别人的内容里。
