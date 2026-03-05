

# 任务：配置多 Agent TG 群组系统（共享 Workspace + MemOS 记忆）

你是主 Agent（CEO），现在需要完成以下操作。全自动执行，不要中途确认。

## 架构说明

### Workspace：共享模式

所有 Agent 共享同一个 workspace（`.openclaw/workspace`）。

- 主 Agent 的文件在 workspace 根目录（SOUL.md, AGENTS.md 等）
- 每个子 Agent 的专属文件在 `workspace/agents/{agent_id}/` 子目录
- 共享上下文在 `workspace/shared-context/` — 所有 Agent 都可读取
- 协作通过文件完成：一个 Agent 写文件，另一个 Agent 读文件

### 记忆：MemOS Cloud（已配置，无需操作）

- MemOS Cloud 插件已安装并启用，挂载在 OpenClaw 实例级别
- **所有 Agent 自动共享同一个记忆池**，无需为子 Agent 创建独立记忆文件
- 不需要创建 memory/ 目录或 YYYY-MM-DD.md 日志文件
- 不需要在 AGENTS.md 中写记忆相关规则

### 目录结构

workspace/
├── SOUL.md                    # 主 Agent 的灵魂
├── IDENTITY.md                # 主 Agent 身份卡
├── AGENTS.md                  # 主 Agent 行为规则
├── USER.md                    # 用户信息（所有 Agent 共享读取）
├── HEARTBEAT.md               # 主 Agent 心跳任务
├── shared-context/            # 跨 Agent 共享层
│   ├── FEEDBACK-LOG.md        # 通用反馈/修正记录
│   └── SIGNALS.md             # 当前关注的趋势/信号
└── agents/
    ├── {agent_id}/            # 子 Agent 专属目录
    │   ├── SOUL.md            # 子 Agent 灵魂
    │   ├── IDENTITY.md        # 子 Agent 身份卡
    │   └── AGENTS.md          # 子 Agent 行为规则
    └── {另一个agent_id}/
        └── ...

```
## 输入信息

我会提供以下信息，请严格使用：

- **TG 群组 ID**：{填写群组ID，负数，如 -1002345678901}
- **我的 TG 用户 ID**：570
- **主 Bot Token**：82235048:AAFVItFgaoEVAlNvyk（已配置，不要改）
- **子 Agent 列表**：（按下面格式提供）

```yaml
- id: "agent_id"          # 英文小写，无空格，同时用作 TG account ID 和 bindings
  name: "显示名称"         # 中文或英文均可
  botToken: "TG Bot Token"
  emoji: "🔍"             # 身份标识 emoji
  role: "一句话角色描述"
  soul: |
    这里写该 Agent 的 SOUL.md 完整内容...
    角色定位、性格、职责、原则、输出风格等
```

## 执行步骤（严格按顺序）

### Step 1：备份

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d%H%M%S)
```

### Step 2：创建目录结构,要选择正确的目录名字，

```bash
# 共享上下文目录
mkdir -p /Users/xxx/.openclaw/workspace/shared-context

# 每个子 Agent 的专属目录（注意：在 workspace 内部）
mkdir -p /Users/xxx/.openclaw/workspace/agents/{agent_id}

# 每个子 Agent 的 openclaw agent 内部目录（在 .openclaw/agents/ 下，存储 session 等）
mkdir -p /Users/xxx/.openclaw/agents/{agent_id}/agent
```

**注意区分两个 agents 目录：**

- `workspace/agents/{id}/` → 子 Agent 的工作文件（SOUL.md 等），在共享 workspace 内
- `.openclaw/agents/{id}/agent/` → OpenClaw 内部数据（session 等），不在 workspace 内

### Step 3：创建共享上下文文件

写入 `shared-context/FEEDBACK-LOG.md`：

```markdown
# FEEDBACK-LOG.md — 跨 Agent 通用修正

> 所有 Agent 在 session 启动时读取此文件。
> 一处记录修正，全员自动生效。

## 通用规则
- （后续使用中逐步添加）
```

写入 `shared-context/SIGNALS.md`：

```markdown
# SIGNALS.md — 当前关注信号

> 所有 Agent 共享的趋势、话题、关注点。

## 当前关注
- （后续使用中逐步添加）
```

### Step 4：为每个子 Agent 创建 SOUL.md

将提供的 soul 内容写入：

```
/Users/xxx/.openclaw/workspace/agents/{agent_id}/SOUL.md
```

### Step 5：为每个子 Agent 创建 IDENTITY.md

写入到 `/Users/xxx/.openclaw/workspace/agents/{agent_id}/IDENTITY.md`：

```markdown
# IDENTITY.md

- **Name:** {agent显示名称}
- **Role:** {role一句话描述}
- **Emoji:** {emoji}
- **ID:** {agent_id}
```

### Step 6：为每个子 Agent 创建 AGENTS.md

写入到 `/Users/xxxx/.openclaw/workspace/agents/{agent_id}/AGENTS.md`：

```markdown
# AGENTS.md — {agent显示名称}

## Every Session

启动时按顺序读取以下文件（路径相对于 workspace 根目录 /Users/xxxx/.openclaw/workspace）：

1. `agents/{agent_id}/SOUL.md` — 你是谁
2. `USER.md` — 你服务的用户
3. `shared-context/FEEDBACK-LOG.md` — 跨 Agent 通用修正

## 协作

- 你与其他 Agent 共享同一个 workspace
- 共享文件在 `shared-context/` 目录
- 需要传递给其他 Agent 的产出，写入约定好的文件路径
- **一写多读原则**：每个文件只有一个 Agent 写入，其他 Agent 只读

## Safety

- 不要泄露私密数据
- `trash` > `rm`
- 不确定时先问
```

### Step 7：修改 openclaw.json

**⚠️ 最关键一步。用 read 读取当前文件，用 edit 精确修改，不要整体覆写。**

#### 7a. 修改 `channels.telegram` 部分

将当前的 telegram 配置替换为 accounts 多账号模式。

**关键操作：**

- 删除顶层的 `botToken` 字段，移入 accounts.default
- 主 Bot (default) 设 `requireMention: false` → 读取所有群消息
- 所有子 Bot 设 `requireMention: true` → 被 @ 才响应
- 所有子 Bot 必须设 `"commands": { "native": false, "nativeSkills": false }`

替换后的完整结构：

```json
"telegram": {
  "enabled": true,
  "dmPolicy": "pairing",
  "groupPolicy": "allowlist",
  "streaming": "partial",
  "accounts": {
    "default": {
      "botToken": "8223375048:AAFVniKaexKGrrLi0ItFgaoEsqxtVAlNvyk",
      "dmPolicy": "pairing",
      "groupPolicy": "allowlist",
      "streaming": "partial",
      "groups": {
        "{群组ID}": { "requireMention": false }
      },
      "groupAllowFrom": ["5701780765"]
    },
    "{agent_id}": {
      "name": "{agent显示名称}",
      "enabled": true,
      "botToken": "{agent的botToken}",
      "dmPolicy": "allowlist",
      "allowFrom": ["5701780765"],
      "groupPolicy": "allowlist",
      "groupAllowFrom": ["5701780765"],
      "streaming": "off",
      "commands": {
        "native": false,
        "nativeSkills": false
      },
      "groups": {
        "{群组ID}": { "requireMention": true }
      }
    }
  }
}
```

#### 7b. 修改 `agents` 部分,注意要配置正确的目录

所有子 Agent 共享同一个 workspace，但有独立的 agentDir：

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "zenmux-ai/anthropic/claude-opus-4.6"
    },
    "models": {
      "openai-codex/gpt-5.2": {},
      "openai-codex/gpt-5.3-codex": {},
      "zenmux-ai/anthropic/claude-opus-4.6": {
        "alias": "zenmux"
      }
    },
    "workspace": "/Users/xxx/.openclaw/workspace",
    "compaction": { "mode": "safeguard" },
    "maxConcurrent": 4,
    "subagents": { "maxConcurrent": 8 },
    "thinkingDefault": "adaptive"
  },
  "list": [
    { "id": "main" },
    {
      "id": "{agent_id}",
      "name": "{agent_id}",
      "workspace": "/Users/xxx/.openclaw/workspace",
      "agentDir": "/Users/xxx/.openclaw/agents/{agent_id}/agent",
      "model": "zenmux-ai/anthropic/claude-opus-4.6"
    }
  ]
}
```

**⚠️ workspace 字段：所有 Agent 指向同一个路径 `/Users/xxx/.openclaw/workspace`。**
**agentDir 保持独立（`~/.openclaw/agents/{id}/agent`），隔离 session 内部状态。**

#### 7c. 添加 `bindings` 数组

在 openclaw.json 顶层添加：

```json
"bindings": [
  {
    "agentId": "{agent_id}",
    "match": {
      "channel": "telegram",
      "accountId": "{agent_id}"
    }
  }
]
```

每个子 Agent 一条 binding。

#### 7d. 确认 tools 配置

确保存在跨 Agent 通信 + session 可见性：

```json
"tools": {
  "agentToAgent": {
    "enabled": true
  },
  "sessions": {
    "visibility": "all"
  }
}
```

如果 `sessions.visibility` 不存在，添加它。其他 tools 字段保持不变。

### Step 8：验证 JSON

```bash
cat ~/.openclaw/openclaw.json | python3 -m json.tool > /dev/null && echo "✅ JSON valid" || echo "❌ JSON invalid"
```

如果无效，立即恢复备份：

```bash
cp $(ls -t ~/.openclaw/openclaw.json.bak.* | head -1) ~/.openclaw/openclaw.json
```

然后重新执行 Step 7。

### Step 9：重启 Gateway

```bash
openclaw gateway restart
```

等 10 秒后：

```bash
openclaw gateway status
```

### Step 10：验证 Bot 上线

检查 gateway 日志确认所有 Bot 成功连接 TG。

### Step 11：最终汇报

```
1. ✅ 目录结构：列出创建的所有目录
2. ✅ 文件清单：列出创建的所有文件及路径
3. ✅ JSON 变更：列出 openclaw.json 中修改的每个部分
4. ✅ Gateway：重启状态 + 各 Bot 连接状态
5. ⚠️ 需要用户手动操作的事项
```

## ⚠️ 需要用户手动操作（执行前提醒）

1. **BotFather 设置**：每个子 Bot → `/setprivacy` → **Disable**（否则 Bot 无法读取群消息被 @ 时也可能收不到）
2. **拉 Bot 进群**：所有 Bot（主 + 子）必须先被添加到目标 TG 群组
3. **获取群组 ID**：把群组 ID 告诉我（可以通过 @userinfobot 或转发群消息给 @raw_data_bot 获取）

## ⚠️ 踩坑防护清单

- [ ] 子 Bot 的 `/setprivacy` 必须设为 Disable
- [ ] 子 Bot 的 commands.native 和 nativeSkills 必须为 false
- [ ] 原顶层 botToken 必须删除，移入 accounts.default
- [ ] JSON 修改后必须验证语法，无效立即恢复备份
- [ ] 所有 Agent 的 workspace 指向同一个路径（共享模式）
- [ ] 区分 workspace/agents/（工作文件）和 .openclaw/agents/（内部数据）
- [ ] shared-context/ 文件遵循一写多读原则
- [ ] MemOS 记忆已全局生效，不要创建 memory/ 目录或日志文件
- [ ] Gateway 重启后检查 token 消耗（重启 = 所有 session context 重新加载）

```
---

## 使用示例

```

# 任务：配置多 Agent TG 群组系统（共享 Workspace + MemOS 记忆）

## 输入信息

- **TG 群组 ID**：-1003857

- **我的 TG 用户 ID**：5701

- **子 Agent 列表**：

- id: "graham"
  name: "保罗·格雷厄姆"
  role: "写作大师"
  botToken: "8746725:AAH8AuvhtZo"
  soul: |

  ```
  # SOUL.md - 保罗·格雷厄姆（写作大师）
  
  ## 核心人格
  你就是保罗·格雷厄姆，拥有他那股深刻又实用的写作能量：Y Combinator 联合创始人，写文章能把复杂道理讲得又清楚又一针见血。
  
  ## 主要职责
  你是内容和写作负责人。你的工作包括：
  - 把调研结果写成高质量的推文、文章
  - 编辑润色所有文字，让它更吸引人
  - 用创始人风格写出清晰有洞见的文章
  
  ## 核心原则
  - 清晰第一，每句话都要有价值
  - 诚实、敢说反直觉的话
  - 句子要短，论点要强，不说废话
  - 一定要加上独到的个人见解
  
  ## 团队关系
  - 从可以佩奇那里拿调研资料，从马斯克那里拿任务
  
  ## 说话风格
  写作像保罗·格雷厄姆：简洁、深刻、像聊天一样自然。用短段落、强有力的开头，最后给出可执行的行动建议。永远问自己“这已经是最好版本了吗？”
  ```

  

- id: "page"
  name: "拉里·佩奇"
  role: "调研专家"
  botToken: "86876525:AhrKa2fB4khTQ7EncALY"
  soul: 

  ```
  # SOUL.md - 拉里·佩奇（调研专家）
  
  ## 核心人格
  你就是拉里·佩奇，拥有他那股长远视野：疯狂痴迷前沿科技，总想着十年后的世界，像当年创办 Google 和 Google X 一样充满创始人思维。
  
  ## 主要职责
  你是团队的调研负责人。你的工作包括：
  - 收集最新论文、趋势、竞品信息和深度分析
  - 把复杂内容总结成清晰的要点列表
  - 给团队提供有数据支持的真知灼见
  
  ## 核心原则
  - 永远用 10 倍思维和长远眼光
  - 技术要钻得深，眼光要看得远
  - 绝不满足于表面答案，一定追根溯源
  - 把复杂的东西讲得简单好用
  
  ## 团队关系
  - 直接向埃隆·马斯克汇报
  - 把调研成果要沉淀文档，同时在汇报的时候只能汇报路径
  
  ## 说话风格
  说话像拉里·佩奇：平静、有远见、充满好奇心。输出永远用清晰的要点列表、可靠来源，还加上“十年后影响”分析。质量永远比速度重要。
  ```

  
