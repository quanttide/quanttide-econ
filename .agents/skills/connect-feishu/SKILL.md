---
name: connect-feishu
description: >
  连接飞书 IM，指导 AI Agent 使用 lark-cli 完成飞书即时通讯操作，包括
  搜索用户与群聊、发送与回复消息、查看历史消息、管理群组。当用户要求
  在飞书发消息、找人、查群、查聊天记录时使用该技能。
compatibility: Requires lark-cli with valid OAuth credentials.
metadata:
  version: "1.0.0"
  author: 量潮科技
  category: Communication
  tags: feishu, lark, im, messaging, contacts
  cli: lark-cli
  dependencies: lark-cli, OAuth credentials
---

# Connect Feishu

## 角色分工

| 标识 | 含义 | 说明 |
|:----:|------|------|
| 🤖 规则 | 可自动化 | 命令直接执行，无需 AI 判断 |
| 🧠 AI | AI 自主决策 | AI 分析信息后自主执行 |
| 🙋 人工确认 | 需用户参与 | AI 展示信息并等待确认 |

## 通用约定

操作前优先通过查询命令获取 ID（用户 open_id、群聊 chat_id），不要假设 ID 是已知的。

### 输出格式

所有查询命令默认输出 JSON。使用 `--jq` 提取关键字段，或用 `--format pretty` 输出可读表格。

### 身份标识

| 标识类型 | 前缀 | 说明 |
|----------|------|------|
| `chat_id` | `oc_` | 群聊 ID |
| `open_id` | `ou_` | 用户 ID |
| `message_id` | `om_` | 消息 ID |

---

## 流程

### Step 1：搜索用户 — 🤖 规则

发送消息前，先通过关键词搜索用户以获取其 `open_id`：

```bash
lark-cli contact +search-user --query "<姓名/关键词>" --format pretty
```

限定范围提高精准度：

```bash
# 只搜聊过天的
lark-cli contact +search-user --query "<姓名>" --has-chatted --format pretty

# 批量查多个用户
lark-cli contact +search-user --queries "张三,李四,王五" --format pretty

# 通过 open_id 查用户信息
lark-cli contact +search-user --user-ids "ou_xxx,ou_yyy" --format pretty
```

### Step 2：搜索群聊 — 🤖 规则

发送消息前，先通过群名搜索群聊以获取其 `chat_id`：

```bash
# 按群名搜索
lark-cli im +chat-search --query "<群名关键词>" --format pretty

# 只看自己管理的群
lark-cli im +chat-search --query "<群名>" --is-manager --format pretty

# 列出所有群聊
lark-cli im +chat-list --format pretty
```

### Step 3：发送消息 🙋 人工确认

AI 组装命令后展示给用户确认，确认后执行。

发送文本消息：

```bash
lark-cli im +messages-send --chat-id "oc_xxx" --text "消息内容"
```

发送 Markdown 消息（自动渲染为富文本卡片）：

```bash
lark-cli im +messages-send --chat-id "oc_xxx" --markdown "**标题**\n- 列表项1\n- 列表项2"
```

发送给个人（使用 `--user-id` 替代 `--chat-id`）：

```bash
lark-cli im +messages-send --user-id "ou_xxx" --text "消息内容"
```

防止重复发送（幂等键）：

```bash
lark-cli im +messages-send --chat-id "oc_xxx" --text "消息内容" --idempotency-key "$(uuidgen)"
```

### Step 4：回复消息 🙋 人工确认

```bash
# 回复消息（主评论）
lark-cli im +messages-reply --message-id "om_xxx" --text "回复内容"

# 回复到子线程
lark-cli im +messages-reply --message-id "om_xxx" --text "回复内容" --reply-in-thread
```

### Step 5：查看消息 — 🤖 规则

查看群聊中的历史消息：

```bash
# 查看最近消息（默认按时间降序）
lark-cli im +chat-messages-list --chat-id "oc_xxx" --format pretty

# 查看指定时间范围的消息
lark-cli im +chat-messages-list --chat-id "oc_xxx" --start "2026-05-01T00:00:00+08:00" --end "2026-05-27T23:59:59+08:00" --format pretty

# 按时间升序排列
lark-cli im +chat-messages-list --chat-id "oc_xxx" --sort asc --format pretty
```

跨群搜索消息：

```bash
# 按关键词搜索
lark-cli im +messages-search --query "关键词" --page-all --format pretty

# 限定群聊范围
lark-cli im +messages-search --query "关键词" --chat-id "oc_xxx" --page-all --format pretty

# 限定发送者和时间
lark-cli im +messages-search --query "关键词" --sender "ou_xxx" --start "2026-05-01T00:00:00+08:00" --end "2026-05-27T23:59:59+08:00" --page-all --format pretty
```

### Step 6：管理群聊 🙋 人工确认

创建群聊：

```bash
# 创建私有群
lark-cli im +chat-create --name "群名" --users "ou_xxx,ou_yyy"

# 创建公开群（需要群名）
lark-cli im +chat-create --name "群名" --type public --users "ou_xxx,ou_yyy" --description "群描述"

# 创建主题群
lark-cli im +chat-create --name "群名" --chat-mode topic --users "ou_xxx,ou_yyy"
```

更新群聊信息：

```bash
# 改群名
lark-cli im +chat-update --chat-id "oc_xxx" --name "新群名"

# 改群描述
lark-cli im +chat-update --chat-id "oc_xxx" --description "新描述"
```

---

## 常见工作流

### 给某人发消息

```
搜索用户 → 获取 open_id → 展示发送命令给用户确认 → 执行
```

```bash
# Step 1: 搜索用户
lark-cli contact +search-user --query "张三" --format pretty

# Step 2: 发送（假设 open_id 为 ou_abc）
lark-cli im +messages-send --user-id "ou_abc" --markdown "你好 **张三**，这是测试消息"
```

### 往群聊发通知

```
搜索群聊 → 获取 chat_id → 展示发送命令给用户确认 → 执行
```

```bash
# Step 1: 搜索群
lark-cli im +chat-search --query "产品实验室" --format pretty

# Step 2: 发送 Markdown 通知
lark-cli im +messages-send --chat-id "oc_xxx" --markdown "# 周报提醒\n- 请各位在周五前提交周报"
```

### 查看未读/近期消息

```
定位群聊或用户 → 拉取消息列表 → 用 --jq 过滤关键信息
```

```bash
# 最近 100 条消息的发送人与摘要
lark-cli im +chat-messages-list --chat-id "oc_xxx" --page-size 50 --format json --jq '.data.items[] | {sender: .sender.name, text: .body.content}'
```

---

## 常见错误

| 场景 | 原因 | 处理 |
|------|------|------|
| `chat_id` 无效 | ID 拼写错误或没有该群权限 | 先用 `+chat-search --query` 确认正确的 chat_id |
| `user_id` 无效 | open_id 格式错误 | 先用 `+search-user --query` 确认正确的 open_id |
| `message_id` 无效 | 消息 ID 过期或错误 | 先用 `+chat-messages-list` 获取最新的 message_id |
| 命令返回空结果 | 搜索词不精确或权限不足 | 缩小关键词或使用 `--page-all` 遍历 |
| 消息发送失败 | 机器人无群权限或被禁言 | 检查群权限，联系群管理员 |

---

## 工作纪律

1. **先查后发** — 发送消息前必须先搜索确认接收方（用户或群聊）存在，并获取正确 ID
2. **AI 不能直接发** — 任何发送/回复/建群操作前，AI 必须展示完整命令供用户确认
3. **搜人用 `--has-chatted`** — 在目标明确时加此参数缩小结果范围
4. **搜群用 `--is-manager`** — 只有管理员才能改群信息，非管理员搜到也操作不了
5. **Markdown 优先** — 正式消息使用 `--markdown` 而非 `--text`，获得更好的阅读体验
6. **长查询用 `--page-all`** — 搜索结果可能分页，加此参数自动遍历所有页
