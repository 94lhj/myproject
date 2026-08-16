# ba-deploy

面向非技术用户的**一键发布工具**，集成 GitLab + 自研运维服务，让任何人都能在 Claude Code 中用自然语言完成项目部署。

## 仓库结构

```
ba-deploy/
├── bap/           # CLI 工具源码（Bun + TypeScript）
└── ba-publish/    # Claude Code skill 分发包
```

---

## bap — CLI 工具

`bap`（Build and Publish）是编译为单二进制的命令行工具，负责与公司 GitLab 和内部运维服务通信。

### 命令

| 命令 | 说明 |
|------|------|
| `bap login` | 浏览器登录 GitLab，配置 SSH 密钥 |
| `bap login-status` | 检查当前登录状态 |
| `bap deploy` | 发布当前项目（阻塞，等待构建完成） |
| `bap deploy --name <名称>` | 指定应用名称（默认取目录名） |
| `bap deploy --no-wait` | 触发构建后立即返回，输出 `run_id` 和 `stream_token` |
| `bap logs <run_id> <stream_token>` | 实时订阅构建日志（SSE 流） |
| `bap list` | 列出所有已发布的应用 |
| `bap status [app_name]` | 查看最近一次构建状态 |
| `bap version` | 查看版本号 |

### 发布流程

1. **前置检查**：检测 git 是否已安装（`GIT_NOT_FOUND`）；检测 Dockerfile 是否存在（`DOCKERFILE_MISSING`），不存在由 skill 负责生成
2. **判断依据**：只看本地是否已有 `git remote origin` —— 有则去运维服务资源里匹配（匹配不上报 `REMOTE_MISMATCH`），无则视为全新项目
3. **首次部署**：运维服务创建空仓库并把当前用户设为 Owner，本地推送代码，等待仓库扫描，创建流水线
4. **后续部署**：同步代码变更，触发已有流水线（缺流水线则补建）
4. **状态追踪**：通过 SSE 实时订阅日志，成功后返回访问地址

支持的项目类型：Next.js、React、Node.js（自动检测）。

### 技术架构

```
用户 / Claude Code
      │  自然语言
      ▼
 ba-publish skill          ← Claude Code 读取 SKILL.md，理解意图
      │  shell 调用
      ▼
   bap CLI                 ← 单二进制，无运行时依赖
      │
      ├─► GitLab API       ← 创建仓库、推送代码（OAuth token）
      │
      └─► 运维服务 API     ← 注册仓库、创建流水线、触发构建、拉取日志
               │  AES-256-GCM 加密通信
               ▼
    {app}.internal.company.com
```

**数据流：**

1. `bap deploy` 检测 Dockerfile；若缺失，输出 `DOCKERFILE_MISSING` 交由 skill 生成后重试
2. 调用 GitLab API 创建/定位仓库，通过 OAuth HTTP URL 推代码
3. 调用运维服务注册仓库（异步轮询 `scan_status`），创建流水线，触发构建
4. 触发后返回 `run_id` + `stream_token`；CLI 或 `bap logs` 通过 SSE 实时接收日志
5. 构建完成后，`bap list/status` 通过运维服务查询应用信息和访问地址

**配置存储：**

- 用户配置（GitLab token）保存在 `~/.bap/config.json`，权限 600
- 项目元数据（应用名、仓库路径、流水线 ID）保存在项目根目录的 `.bap` 文件（已加入 `.gitignore`）

**安全：**

- 所有运维服务请求使用 AES-256-GCM 对称加密，密钥由 `OPS_AES_KEY_HEX` 环境变量注入
- OAuth token 每次请求前自动刷新（`getValidAccessToken`）

### 开发

技术栈：**Bun + TypeScript**，编译为独立二进制，无需用户安装任何运行时。

```bash
# 安装依赖
bun install

# 本地运行
bun run src/index.ts deploy

# 编译多平台二进制（darwin-x64 / darwin-arm64）
bun run build

# 类型检查
bun run lint
```

编译产物：
- `bap/dist/bap-darwin-x64`
- `bap/dist/bap-darwin-arm64`

构建脚本还会：
- 将二进制同步到 `ba-publish/bin/`
- 将 `ba-publish/SKILL.md` 复制为 `ba-publish/.claude/commands/bap.md`（Claude Code slash command 分发格式）

本地测试请直接运行 `bun run src/index.ts <command>`（见上方"本地运行"），不再自动创建全局 `/usr/local/bin/bap` 软链接，以免掩盖 `ba-publish` skill 端到端分发流程中的问题。

**关键源文件：**

| 文件 | 说明 |
|------|------|
| `src/index.ts` | CLI 入口，命令路由与 flag 解析 |
| `src/commands/deploy.ts` | 核心发布逻辑（首次/增量部署） |
| `src/commands/logs.ts` | 实时日志订阅 |
| `src/commands/status.ts` | 查询最近构建状态 |
| `src/commands/list.ts` | 列出用户所有应用 |
| `src/lib/gitlab.ts` | GitLab API 封装 |
| `src/lib/ops.ts` | 运维服务 API 封装（含 AES 加密、SSE 日志流） |
| `src/lib/detect.ts` | 项目类型自动检测 |
| `src/lib/template.ts` | .gitignore 等模板注入 |
| `src/lib/git.ts` | Git 操作封装（init、push、sync、revert） |
| `src/lib/config.ts` | 配置读写、token 刷新 |

---

## ba-publish — Claude Code Skill

`ba-publish` 是打包好的 Claude Code skill，让用户直接用自然语言在 Claude Code 中触发部署，无需记忆命令。

### 安装

将 `ba-publish/` 目录复制到 `~/.claude/skills/ba-publish/`：

```bash
cp -r ba-publish ~/.claude/skills/ba-publish
```

Claude Code 会自动检测并注册该 skill。

### 使用

安装后，在 Claude Code 中直接说：

> 帮我发布这个项目

> 查看我的应用列表

> 回滚上一版本

Claude 会全程用中文引导，自动处理登录、Dockerfile 生成、构建、日志轮询等细节，无需接触任何技术术语。

**Skill 处理的关键场景：**

- `DOCKERFILE_MISSING`：读取项目结构，生成合适的 Dockerfile，用户确认后重新触发
- `GIT_CONFLICT`：告知用户需手动解决冲突
- 构建进度：解析 `run_id` + `stream_token`，优先用 Monitor 订阅 `bap logs` 实时流；异常时退回每 15 秒轮询 `bap status`

---

## 发布新版本

```bash
# 1. 修改 bap/src/ 中的代码
# 2. 编译并同步到 ba-publish
cd bap && bun run build

# 3. 提交 ba-publish/ 目录（包含新二进制）发布给用户
```





bap deploy
│
├─ [前置] 本地检查 Dockerfile
│    └─ 不存在 → 返回 DOCKERFILE_MISSING，退出（skill 接管生成）
│
├─ [前置] 本地检查 .gitignore
│    └─ 不存在 → 注入默认模板，继续
│
├─ [Token] 检查 token_expires_at
│    ├─ 未过期 → 继续
│    └─ 过期 → 文件锁
│              → refresh API
│              → 成功: 写 config + 调接口5B 同步到运维服务 + 释放锁
│              → 401: 抛 NOT_LOGGED_IN
│
├─ 调接口5A 获取用户所有 repo+pipeline
│
├─ 判断当前项目是否在列表中（匹配 gitlab_url）
│   │
│   ├─ 【已存在 → 增量更新流程】
│   │   ├─ 检查本地 git remote 是否已关联
│   │   │    └─ 未关联 → git remote add origin {gitlab_url}
│   │   ├─ git fetch origin
│   │   ├─ 比较本地 vs 远端 HEAD
│   │   │    ├─ 本地更新 → git push（直接推增量）
│   │   │    └─ 远端更新 → git pull --rebase
│   │   │                   ├─ 成功 → git push
│   │   │                   └─ 冲突 → 输出冲突文件列表，退出（人工处理）
│   │   └─ 调接口3 触发流水线
│   │
│   └─ 【不存在 → 首次部署流程】
│       ├─ GitLab API 在配置 group 下创建仓库
│       ├─ 调接口1 POST /api/external/repos（异步）
│       │    └─ 轮询 scan_status，每 3 秒一次
│       │         ├─ done → 继续
│       │         └─ error → 输出错误，退出
│       ├─ 调接口2 POST /api/external/pipelines 创建流水线（同步）
│       ├─ git init + add + commit
│       ├─ git push -u origin main
│       └─ 调接口3 触发流水线
│
└─ 调接口4 SSE 实时日志，输出到 stderr 供 skill 展示
     ├─ success → 输出 JSON 结果（含 app_url 来自接口2返回的 custom_domain）
     └─ failed  → 输出错误 JSON