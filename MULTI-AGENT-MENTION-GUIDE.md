# OpenClaw 多 Agent 群聊互相呼叫指南

## 问题

Telegram Bot 之间**无法互相 @**。Bot A 在群里 `@BotB` 发消息，Bot B 收不到。这是 Telegram 平台限制，不是 OpenClaw 的 bug。

**只有人类用户 @ Bot 才有效，Bot 之间 @ 无效。**

## 解决方案：mentionPatterns + groupPolicy

用**关键词触发**替代 @提及。当消息中包含指定名字时，对应 Bot 自动收到并响应。

---

## 配置修改（3 步）

### 第 1 步：给每个子 Agent 添加 mentionPatterns

在 `openclaw.json` 的 `agents.list` 中，给每个子 Agent 加上 `groupChat.mentionPatterns`：

```json
{
  "agents": {
    "list": [
      { "id": "main" },
      {
        "id": "graham",
        "name": "graham",
        "workspace": "/path/to/agents/graham",
        "groupChat": {
          "mentionPatterns": ["保罗·格雷厄姆"]
        }
      },
      {
        "id": "page",
        "name": "page",
        "workspace": "/path/to/agents/page",
        "groupChat": {
          "mentionPatterns": ["拉里·佩奇"]
        }
      }
    ]
  }
}
```

**规则：**
- `mentionPatterns` 是字符串数组，支持多个触发词
- 消息中只要包含数组中任意一个词，对应 Agent 就会收到
- 建议用**完整名字**，避免简称导致误触发（比如 "佩奇" 可能在聊小猪佩奇时误触）

### 第 2 步：子 Bot 的 groupPolicy 改为 open

在 `channels.telegram.accounts` 中，把子 Bot 的 `groupPolicy` 从 `"allowlist"` 改为 `"open"`，并移除 `requireMention`：

```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "graham": {
          "groups": {
            "-100xxxxxxxxxx": {}
          },
          "groupPolicy": "open",
          "groupAllowFrom": ["你的用户ID"]
        },
        "page": {
          "groups": {
            "-100xxxxxxxxxx": {}
          },
          "groupPolicy": "open",
          "groupAllowFrom": ["你的用户ID"]
        }
      }
    }
  }
}
```

**关键变化：**
- `groupPolicy`: `"allowlist"` → `"open"`（让 Bot 能收到群里所有消息）
- `groups.{groupId}`: 移除 `"requireMention": true`（改为空对象 `{}`）
- `mentionPatterns` 会在 Agent 层面做过滤，只响应包含触发词的消息

### 第 3 步：重启 Gateway 建议手动操作

```bash
openclaw gateway restart
```

---

## 前置条件（Telegram 端）

在修改配置**之前**，确保：

1. **BotFather → /setprivacy → Disable**
   - 对每个子 Bot 都要做
   - 关闭后需要把 Bot 从群里移除再重新添加才生效

2. **把所有 Bot 拉进目标群聊**
   - 主 Bot + 所有子 Bot 都要在群里

3. **确认 Bot 在线**
   - `openclaw gateway status` 检查
   - 查日志：`grep "starting provider" /tmp/openclaw/openclaw-$(date +%F).log`

---

## AGENTS.md 配套规则

在主 Agent 的 AGENTS.md 里加上提及规则，防止主 Agent 用 @ 去叫子 Agent：

```markdown
### 🤖 跨 Agent 提及规则

不要用 @bot_username 来提及其他 Agent（Bot 之间 @ 无效）。
要让某个 Agent 收到消息，直接在消息中写出完整名字：

| Agent | 触发名字 | 说明 |
|---|---|---|
| 保罗·格雷厄姆 | `保罗·格雷厄姆` | YC 创始人人格 |
| 拉里·佩奇 | `拉里·佩奇` | Google 创始人人格 |

名字必须写完整、准确。
```

---

## 工作原理

```
用户发消息 "保罗·格雷厄姆，你怎么看？"
        │
        ▼
  Telegram 把消息发给所有 Bot（因为 privacy=off）
        │
        ├── 主 Bot 收到 → groupPolicy: allowlist → 检查发送者 → 通过 → 正常处理
        ├── Graham Bot 收到 → groupPolicy: open → mentionPatterns 匹配 "保罗·格雷厄姆" → ✅ 触发响应
        └── Page Bot 收到 → groupPolicy: open → mentionPatterns 不匹配 → ❌ 静默忽略
```

---

## 常见问题

**Q: 子 Bot 还是没反应？**
- 检查 BotFather privacy 是否关闭
- 检查 Bot 是否在群里
- 检查 groupPolicy 是否为 open
- 检查 mentionPatterns 拼写是否和消息中完全一致

**Q: 会不会所有消息都触发子 Bot？**
- 不会。虽然 groupPolicy 是 open（Bot 能收到所有消息），但 mentionPatterns 会过滤，只有包含触发词的消息才会让 Agent 响应。

**Q: 主 Bot 能不能通过内部调用让子 Bot 说话？**
- 可以，用 `sessions_send` 跨 Agent 通信（需要 `tools.agentToAgent.enabled: true`）。但这是内部通道，不经过 Telegram。

**Q: mentionPatterns 支持正则吗？**

- 支持。但建议用纯文本，正则容易误匹配。

---

## 配置 Diff 汇总

```diff
# agents.list 中的子 Agent
+ "groupChat": {
+   "mentionPatterns": ["保罗·格雷厄姆"]
+ }

# channels.telegram.accounts 中的子 Bot
- "requireMention": true
- "groupPolicy": "allowlist"
+ "groupPolicy": "open"
```

三处改动，restart 生效。
