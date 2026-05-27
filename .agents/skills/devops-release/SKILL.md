---
name: devops-release
description: >
  封装量潮DevOps发布规范为可执行技能，指导 AI Agent 完成 Git 标签发布与
  GitHub Release 全流程。当用户提出发布、release、发版、打标签、版本号升级
  或退役版本时使用该技能。
compatibility: Requires git and gh (GitHub CLI).
metadata:
  version: "1.0.0"
  author: 量潮科技
  category: DevOps
  tags: release, git, github, ci/cd, versioning
  source: https://github.com/quanttide/qtcloud-devops
  cli: qtcloud-devops
  dependencies: git, gh, cargo (optional), pip (optional)
---

# Devops Release

## 状态机

发布版本的生命周期共 4 种状态，状态转换方向固定：

```
Staged ────→ Published ────→ Retired
  ↑               │
  └──── Cancelled ┘
```

| 状态 | 含义 | 可操作 |
|------|------|--------|
| **Staged** | 版本已标记，等待 CI 验证 | `publish` 继续、`cancel` 作废 |
| **Published** | 正式上线（tag + GitHub Release 已创建） | `retire` 退役 |
| **Cancelled** | 已取消（审计用途，不清理 tag） | 无（终态） |
| **Retired** | 已退役 | 无（终态） |

---

## 角色分工

AI Agent 在执行发布流程时，需要区分哪些步骤可自动化、哪些需要等待用户确认。

| 标识 | 含义 | 说明 |
|:----:|------|------|
| 🤖 规则 | 可自动化 | 静态规则或 CLI 命令即可执行，无需 AI 判断 |
| 🧠 AI | AI 自主决策 | AI 分析信息后自主执行，无需用户确认 |
| 🙋 人工确认 | 需用户参与 | AI 向用户展示信息并等待确认，或处理异常决策 |

---

## 流程

### Step 1：了解当前状态 — 🧠 AI

```bash
# 查看发布状态
qtcloud-devops release status

# 查看 CHANGELOG
cat CHANGELOG.md

# 查看待办与缺陷
cat ROADMAP.md
cat BUGS.md
```

AI 应向用户输出摘要：

- 当前版本号（从 `Cargo.toml` / `pyproject.toml` 读取）
- 待发布版本列表
- 最新发布记录
- 已知阻塞项

### Step 2：版本号升级 — 🙋 人工确认

> CLI 暂未提供自动 bump 能力，AI 手动编辑后必须输出摘要交用户确认。

所有语言清单文件与 CHANGELOG 的版本号**必须一致**（按项目实际语言检查对应文件）：

| 语言 | 文件 | 字段/格式 |
|:----:|------|-----------|
| Rust | `Cargo.toml` | `version = "X.Y.Z"` |
| Python | `pyproject.toml` | `version = "X.Y.Z"` |
| Dart | `pubspec.yaml` | `version: X.Y.Z` |
| Go | `go.mod` / `VERSION` | `module ...` / 文件内容 `X.Y.Z` |
| — | `CHANGELOG.md` | `## [X.Y.Z] - YYYY-MM-DD` |

CHANGELOG 格式：

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added
- 新功能 A

### Fixed
- 修复问题 B

### Changed
- 变更 C
```

AI 完成编辑后，输出版本一致性摘要供用户确认：

```text
Cargo.toml:    version = "X.Y.Z"
pyproject.toml: version = "X.Y.Z"
CHANGELOG.md:  ## [X.Y.Z] - YYYY-MM-DD
```

用户确认一致后，继续执行 Step 3。

### Step 3：提交变更 — 🤖 规则

Step 2 中修改的三个文件需提交并推送，确保工作区干净：

```bash
# 提交变更并推送
git add -A && git commit -m "chore: bump to vX.Y.Z" && git push origin
```

### Step 4：预检查 — 🤖 规则

```bash
# 使用 CLI 预检查（dry-run 仅检查不执行）
qtcloud-devops release stage -v cli/vX.Y.Z-rc.1     # 预发布走 stage

# 或运行本地预验证脚本
./scripts/preflight.sh
```

预检查自动包括：

- ✅ 版本号格式（semver + scope 前缀）
- ✅ CHANGELOG.md 存在且包含目标版本记录
- ✅ 标签是否已存在（幂等跳过）
- ✅ 工作区是否干净
- ✅ 是否在可发布分支

### Step 5：预发布（可选）— 🙋 人工确认

对正式版本，建议先走预发布流程触发 CI 验证。AI 输出待执行命令供用户确认 tag 无误：

```text
即将执行: qtcloud-devops release stage -v cli/vX.Y.Z-rc.1
tag: cli/vX.Y.Z-rc.1
确认执行? (y/N):
```

用户确认后 AI 执行：

```bash
# 标记预发布版本（仅接受 -rc.N / -alpha.N / -beta.N 后缀）
qtcloud-devops release stage -v cli/vX.Y.Z-rc.1

# 等待 CI 构建通过后，如失败则递增 rc 序号
qtcloud-devops release stage -v cli/vX.Y.Z-rc.2
```

`stage` 会自动完成：

1. 写 journal 记录到 `.quanttide/devops/release-journal.jsonl`
2. `git tag` 创建本地标签
3. `git push origin` 推送标签
4. `gh release create` 创建 GitHub Release
5. **回滚**：步骤 3 失败时删本地 tag；步骤 4 失败时删本地+远程 tag

### Step 6：正式发布 — 🙋 人工确认

AI 输出待执行命令供用户确认 tag 无误：

```text
即将执行: qtcloud-devops release publish -v cli/vX.Y.Z
tag: cli/vX.Y.Z
确认执行? (y/N):
```

用户确认后 AI 执行：

```bash
# 正式发布（不需要先 stage）
qtcloud-devops release publish -v cli/vX.Y.Z

# 或带注册源指定
qtcloud-devops release publish -v cli/vX.Y.Z --registry crates
```

CLI 会再次输出发布摘要并等待用户输入 `y/yes` 二次确认（传 `-y` 跳过，仅 CI/自动化场景使用）：

```text
发布版本: cli/vX.Y.Z

确认发布? (y/N):
```

`publish` 会自动完成：

1. 预检查（同 stage）
2. `git tag` 创建本地标签（幂等）
3. `git push origin` 推送标签
4. `gh release create` 创建 GitHub Release
5. 写 journal 更新状态为 Published
6. **回滚**：步骤 3 失败时删本地 tag；步骤 4 失败时删本地+远程 tag

### Step 7：验证发布 — 🤖 规则

```bash
# 验证发布状态
qtcloud-devops release status

# 验证 GitHub Release
gh release view cli/vX.Y.Z --repo quanttide/quanttide-devops

# 验证注册源（如有）
cargo search qtcloud-devops-cli --registry crates-io
pip install qtcloud-devops-cli==X.Y.Z
```

预期输出：

```
✓ Release vX.Y.Z 创建成功
  标签: vX.Y.Z
  URL: https://github.com/quanttide/quanttide-devops/releases/tag/vX.Y.Z
```

### Step 8：版本退役（按需）— 🙋 人工确认

```bash
# 退役已发布的旧版本
qtcloud-devops release retire -v v0.3.0
```

---

## 一次完整的发布流程示例

以发布 `cli/v0.4.1` 为例：

```bash
# 1. 确认当前状态
qtcloud-devops release status

# 2. 编辑版本号 + CHANGELOG（三个文件一致）
#    Cargo.toml:    version = "0.4.1"
#    pyproject.toml: version = "0.4.1"
#    CHANGELOG.md:  ## [0.4.1] - 2026-05-25

# 3. 提交流
git add -A && git commit -m "chore: bump to v0.4.1" && git push origin main

# 4. 预发布验证（可选）
qtcloud-devops release stage -v cli/v0.4.1-rc.1
#   → CI 触发构建验证，通过后继续

# 5. 正式发布
qtcloud-devops release publish -v cli/v0.4.1

# 6. 验证
qtcloud-devops release status
```

---

## 版本号格式

### 规范版本

```
vX.Y.Z[-pre.release]
scope/vX.Y.Z[-pre.release]
```

| 示例 | 说明 |
|------|------|
| `v0.4.1` | 正式版本 |
| `cli/v0.4.1` | 带 scope 前缀的正式版本 |
| `v1.0.0-rc.1` | 预发布候选版 |
| `cli/v0.4.1-alpha.2` | 带 scope 的 alpha 预发布 |
| `v2.0.0-beta.1` | beta 预发布 |

### 拒绝规则

版本号必须以 `v` 开头，后接语义化版本号。scope 前缀后必须跟 `/v`。

```
✅ 接受: v1.2.3, cli/v1.2.3, v1.2.3-rc.1
❌ 拒绝: 1.2.3（缺 v 前缀）, v1.2（缺 patch）, abc（非法格式）
```

### stage 的预发布限制

`stage` 命令**仅接受**预发布版本（含 `-rc.N`、`-alpha.N`、`-beta.N` 等后缀）：

```
✅ stage -v cli/v1.0.0-rc.1
✅ stage -v v2.0.0-alpha.1
❌ stage -v cli/v1.0.0     → 正式版请直接 publish
❌ stage -v v1.0.0          → 正式版请直接 publish
```

---

## 错误处理与回滚

| 阶段 | 错误场景 | 处理方式 | 自动回滚 | 角色 |
|------|----------|----------|:--------:|:----:|
| 预检查 | CHANGELOG 缺少版本记录 | 输出错误，提示更新 CHANGELOG | — | 🤖 |
| 预检查 | 版本号格式错误 | 输出错误，提示正确格式 | — | 🤖 |
| 预检查 | 标签已存在 | 输出错误，提示使用新版本或删除旧标签 | — | 🤖 |
| 预检查 | 工作区脏 | 输出错误，提示提交或暂存变更 | — | 🤖 |
| 预检查 | CHANGELOG 文件不存在 | 输出错误，提示创建 | — | 🤖 |
| 预检查 | 正式版调用 stage | 输出错误，提示 publish | — | 🤖 |
| 执行 | `create_tag` 失败 | 输出错误，终止 | — | 🤖 |
| 执行 | `push_tag` 失败 | 删除本地 tag | ✅ | 🧠 |
| 执行 | `create_release` 失败 | 删除本地+远程 tag | ✅ | 🧠 |
| 执行 | 已发布版本重复 stage | 输出错误，拒绝 | — | 🤖 |
| 执行 | 已退役版本重复 stage | 输出错误，拒绝 | — | 🤖 |
| 验证 | `gh release view` 失败 | 输出警告（tag 和 release 可能已存在） | — | 🤖 |

### 幂等保证

- `create_tag`：tag 已存在时静默跳过，不报错
- `create_release`：Release 已存在时静默跳过，不报错
- `push_tag`：无 remote 时静默跳过（本地 tag 已创建）
- `publish`：从 journal 还原已发布版本，不重复创建

---

## 架构与数据

### Journal 持久化

发布记录以 JSONL 格式追加写入 `.quanttide/devops/release-journal.jsonl`：

```json
{"id":"uuid","version":"cli/v0.4.1","status":"Staged","created_at":"1716543200"}
{"id":"uuid","version":"cli/v0.4.1","status":"Published","created_at":"1716543400"}
```

- 一行一条事件，**追加写**不回溯修改
- `Replay` 时按 version 聚合，取最后一条事件作为最新状态
- `created_at` 保留首次创建时间，`updated_at` 取最后事件的时间

### CLI 架构

```
qtcloud-devops
├── release            ← 发布管理
│   ├── stage          ← 标记预发布（tag + Release + journal）
│   ├── publish        ← 正式发布（tag + Release + journal）
│   ├── retire         ← 退役（journal 标记）
│   └── status         ← 查询（journal 回放）
└── code               ← 子模块管理（不在本技能范围）
```

### 仓库自动检测

`get_remote_repo()` 从 `git remote get-url origin` 解析 GitHub 仓库名，支持 HTTPS 和 SSH 两种格式：

```
https://github.com/owner/repo.git  → owner/repo
git@github.com:owner/repo.git      → owner/repo
```

---

## Workflow Cheatsheet

```bash
# ─── 常用命令 ───────────────────────────────────────────

# 查看状态
qtcloud-devops release status

# 预发布（CI 验证）
qtcloud-devops release stage -v cli/v0.4.1-rc.1

# 正式发布
qtcloud-devops release publish -v cli/v0.4.1

# 退役
qtcloud-devops release retire -v v0.3.0


# ─── 降级方案：纯 git + gh（仅 CLI 不可用时使用） ────

# 创建标签
git tag cli/v0.4.1 && git push origin cli/v0.4.1

# 创建 Release
gh release create cli/v0.4.1 --title "cli/v0.4.1" \
  --notes "$(cat CHANGELOG.md | awk '/## \[/{if(i)exit;i=1;next}i')" \
  --repo quanttide/quanttide-devops


# ─── 回滚 ───────────────────────────────────────────────

# 删除本地标签
git tag -d cli/v0.4.1

# 删除远程标签
git push origin --delete cli/v0.4.1

# 删除 GitHub Release
gh release delete cli/v0.4.1 --repo quanttide/quanttide-devops -y
```

---

## 工作纪律

1. **AI 禁止直接 publish** — git 操作止步于 `commit && push`。tag、stage、publish 需由人工执行或经确认后由 AI 在 CLI 中执行
2. **发布前预验证** — 运行 `./scripts/preflight.sh` 确保编译 + 测试 + dry-run 发布通过
3. **必须走 rc 流程** — 正式发布前至少经过一次 rc 预发布 + CI 验证
4. **版本号三文件一致** — `Cargo.toml` + `pyproject.toml` + `CHANGELOG.md` 版本号必须相同
5. **CHANGELOG 必须有对应版本** — 缺失对应版本记录时拒绝执行
6. **stage 只接受预发布** — 正式版本不要调用 stage，直接 publish
