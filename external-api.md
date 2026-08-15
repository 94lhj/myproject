# AIOps 外部 API 文档

外部 API 供第三方系统（如 GitLab CI、自动化脚本）调用，无需用户登录 Cookie，通过对称加密鉴权。

---

## 鉴权机制

所有请求体均使用 **AES-256-GCM** 对称加密，密钥由服务端环境变量 `EXTERNAL_API_KEY` 配置（64 位十六进制字符串，即 32 字节）。

### 加密格式

```
wire = base64( IV[12字节] || 密文 || AuthTag[16字节] )
```

请求体固定结构：

```json
{
  "payload": "<base64编码的加密数据>"
}
```

### Node.js 加密示例

```js
import { createCipheriv, randomBytes } from 'crypto'

function encryptPayload(data, hexKey) {
  const key = Buffer.from(hexKey, 'hex')
  const iv = randomBytes(12)
  const cipher = createCipheriv('aes-256-gcm', key, iv)
  const plain = Buffer.from(JSON.stringify(data), 'utf8')
  const ciphertext = Buffer.concat([cipher.update(plain), cipher.final()])
  const tag = cipher.getAuthTag()
  return Buffer.concat([iv, ciphertext, tag]).toString('base64')
}

const payload = encryptPayload({ username: 'hanley' }, process.env.EXTERNAL_API_KEY)
// 请求体: { payload }
```

### Python 加密示例

```python
import json, base64
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import secrets

def encrypt_payload(data: dict, hex_key: str) -> str:
    key = bytes.fromhex(hex_key)
    iv = secrets.token_bytes(12)
    ct_with_tag = AESGCM(key).encrypt(iv, json.dumps(data).encode(), None)
    return base64.b64encode(iv + ct_with_tag).decode()
```

---

## 管理员前置配置

API 1 和 API 2 需要以管理员身份调用 GitLab API（`https://gitlab.diancun.net`）。

需在服务端环境变量中配置 `GITLAB_ADMIN_TOKEN`（GitLab Personal Access Token，需有 `api` 权限），与系统管理员账号解耦，不再依赖平台内某个用户名下的 Git 凭证。

---

## 错误响应格式

```json
{ "error": "错误描述" }
```

| HTTP 状态码 | 含义 |
|---|---|
| 400 | 参数缺失 / payload 解密失败 |
| 403 | 账号已被禁用 |
| 404 | 资源不存在 |
| 409 | 资源冲突（如流水线已存在） |
| 422 | 业务前置条件不满足 |
| 500 | 服务端内部错误 |
| 502 | GitLab API 调用失败 |

---

## API 列表

| # | 方法 | 路径 | 功能 |
|---|---|---|---|
| 1 | POST | `/api/external/users/init` | 初始化用户并生成 GitLab token |
| 2 | POST | `/api/external/repos` | 在 GitLab 创建仓库并关联用户 |
| 2b | POST | `/api/external/repos/import` | 关联一个已存在的 GitLab 仓库 |
| 3 | POST | `/api/external/pipelines` | 为仓库创建流水线 |
| 3b | POST | `/api/external/pipelines/:id/env` | 为流水线设置环境变量（可选） |
| 3c | POST | `/api/external/pipelines/:id/volumes` | 为流水线设置存储卷（可选） |
| 3d | POST | `/api/external/pipelines/:id/env/keys` | 查询流水线环境变量 Key 列表 |
| 4 | POST | `/api/external/pipelines/:id/run` | 触发流水线执行 |
| 5 | POST | `/api/external/user/pipelines` | 查询用户仓库与流水线 |
| 6 | GET  | `/api/external/pipeline-runs/:id/logs` | 获取执行日志 |
| 7 | POST | `/api/external/pipelines/:id/pod-logs` | 获取服务 Pod 控制台日志（纯文本） |
| 8 | POST | `/api/external/auth/token` | 获取免登录访问令牌 |
| 8b | GET  | `/api/external/auth/consume` | 用令牌免登录访问页面（浏览器直接跳转，非加密接口） |

---

## 1. 初始化用户

**`POST /api/external/users/init`**

按以下逻辑处理：

| 情况 | 行为 |
|---|---|
| 用户不存在 | 创建系统用户 → 调用 GitLab API 创建 Impersonation Token → 保存凭证 |
| 用户已存在 + **有凭证** | 直接返回用户信息，不调用 GitLab |
| 用户已存在 + **无凭证** | 跳过用户创建 → 调用 GitLab API 创建 Impersonation Token → 保存凭证 |

GitLab Impersonation Token 名称固定为 `aiops`，权限 `read_api + read_repository`，**有效期 1 年**，过期时间同步保存在凭证记录中。

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `username` | string | ✅ | 用户名，须与 GitLab 用户名一致 |
| `password` | string | ❌ | 初始登录密码，不填则随机生成 |

所有成功响应均包含 `gitlab_token` 字段，值为原始 GitLab Token 经 `EXTERNAL_API_KEY`（AES-256-GCM）加密后的 base64 字符串，调用方可用相同密钥解密还原。

### 响应 `200 OK`（用户已存在且有凭证）

```json
{
  "data": {
    "existed": true,
    "user_id": 5,
    "username": "hanley",
    "is_active": true,
    "has_credential": true,
    "credential_id": 3,
    "gitlab_token_created": false,
    "gitlab_token": "<AES-256-GCM 加密的 base64 字符串>",
    "token_expires_at": "2027-07-02",
    "created_at": "2026-06-01 10:00:00"
  }
}
```

### 响应 `201 Created`（用户新建 或 用户已存在但无凭证）

```json
{
  "data": {
    "existed": false,
    "user_id": 5,
    "username": "hanley",
    "is_active": true,
    "credential_id": 3,
    "gitlab_token_created": true,
    "gitlab_token": "<AES-256-GCM 加密的 base64 字符串>",
    "token_expires_at": "2027-07-02"
  }
}
```

用户已存在但无凭证时，响应中 `existed` 为 `true`，其余字段与新建一致。

`token_expires_at` 为凭证到期日（格式 `YYYY-MM-DD`）；凭证无到期时间时不含此字段。若 GitLab Token 创建失败（GitLab 用户不存在或管理员凭证未配置），`gitlab_token_created` 为 `false`，`gitlab_token` 和 `token_expires_at` 均不存在，响应中附带 `warning` 字段说明原因。

### gitlab_token 解密示例

```js
import { createDecipheriv } from 'crypto'

function decryptToken(encrypted, hexKey) {
  const buf = Buffer.from(encrypted, 'base64')
  const key = Buffer.from(hexKey, 'hex')
  const iv         = buf.subarray(0, 12)
  const tag        = buf.subarray(buf.length - 16)
  const ciphertext = buf.subarray(12, buf.length - 16)
  const decipher = createDecipheriv('aes-256-gcm', key, iv)
  decipher.setAuthTag(tag)
  return Buffer.concat([decipher.update(ciphertext), decipher.final()]).toString('utf8')
}

const token = decryptToken(data.gitlab_token, process.env.EXTERNAL_API_KEY)
```

---

## 2. 创建仓库

**`POST /api/external/repos`**

以系统管理员身份调用 GitLab API，在 `it_service` 组下创建空私有仓库（默认分支 `main`），并将指定用户设为 Developer（access_level=30）。**幂等**：若 GitLab 中同名仓库已存在，则直接授予用户 Developer 权限，不重复创建。完成后将仓库关联到该用户并触发异步扫描。

**前置条件：** 用户已通过 API 1 初始化（有 GitLab 凭证）。

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `username` | string | ✅ | 系统用户名 |
| `repo_name` | string | ✅ | 仓库名称（同时作为 GitLab project path） |

### 响应 `201 Created`

```json
{
  "data": {
    "user_id": 5,
    "credential_id": 3,
    "repo": {
      "id": 12,
      "name": "my-app",
      "git_url": "https://gitlab.diancun.net/it_service/my-app.git",
      "web_url": "https://gitlab.diancun.net/it_service/my-app",
      "gitlab_project_id": 88,
      "scan_status": "pending",
      "default_branch": "main",
      "has_dockerfile": false,
      "dockerfile_path": null
    }
  }
}
```

**`scan_status` 取值：**`pending` / `scanning` / `done` / `error`

> 仓库为空仓（无初始提交），默认分支为 `main`。推送包含 `Dockerfile` 的代码后，通过 API 5 可查看 `scan_status` 更新为 `done` 及 `has_dockerfile: true`，之后才可创建流水线。

---

## 2b. 关联已有仓库

**`POST /api/external/repos/import`**

与网页端「代码仓库」页面右上角「添加仓库」功能完全一致：传入一个**已存在**的 GitLab 仓库完整 URL，使用该用户已初始化的 Git 凭证克隆并扫描，而不是像 API 2 那样在 GitLab 中新建仓库。

**前置条件：** 用户已通过 API 1 初始化（有 Git 凭证）。

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `username` | string | ✅ | 系统用户名 |
| `git_url` | string | ✅ | 仓库完整 URL，如 `https://gitlab.diancun.net/it_service/my-app.git`（不限制域名，只要格式正确即可） |

### URL 校验规则

- 必须是合法的 URL
- 必须是 `https://` 协议
- 路径需包含至少 `<namespace>/<repo>` 两段

不限制仓库所在域名，不满足以上任一格式条件均返回 `400`。

**幂等：** 同一用户重复传入相同 `git_url` 不会重复创建仓库记录，仅刷新凭证关联并返回已有记录（响应状态码为 `200` 而非 `201`）。

### 响应 `201 Created`（新增）/ `200 OK`（已存在）

```json
{
  "data": {
    "user_id": 5,
    "credential_id": 3,
    "repo": {
      "id": 13,
      "name": "my-app",
      "git_url": "https://gitlab.diancun.net/it_service/my-app.git",
      "scan_status": "pending",
      "default_branch": "main",
      "has_dockerfile": false,
      "dockerfile_path": null
    }
  }
}
```

`name` 由 `git_url` 最后一段路径自动推导（去除 `.git` 后缀）。仓库名称推导规则、克隆与扫描逻辑与网页端「添加仓库」表单一致；扫描完成后通过 API 5 可查看 `scan_status` 和 `has_dockerfile` 的最新状态。

### 错误响应

| 状态码 | 原因 |
|---|---|
| `400` | 参数缺失 / `git_url` 不合法 |
| `403` | 账号已被禁用 |
| `404` | 用户不存在 |
| `422` | 用户尚未初始化 Git 凭证 |

---

## 3. 为仓库创建流水线

**`POST /api/external/pipelines`**

为已注册的仓库在默认集群上创建流水线。每个仓库只能有一条流水线，重复创建返回 `409`。

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `repo_id` | number | ✅ | 仓库 ID（由 API 2 返回） |
| `lark_webhook` | string | ❌ | Lark 机器人 Webhook URL，传入则开启通知 |
| `domain_prefix` | string | ❌ | 自定义域名前缀；传入则使用 `<domain_prefix>.<集群默认域名>` 替代随机生成的域名 |

`domain_prefix` 只能包含小写字母、数字、连字符，且不能以连字符开头或结尾（同 K8s 资源名规则）。

**前置条件（不满足返回 `422`）：**
- 仓库扫描状态为 `done`
- 仓库中检测到 `Dockerfile`
- 系统存在已启用的默认集群
- 若传入 `domain_prefix`，默认集群必须已配置默认域名，否则无处可用该前缀

**冲突处理：** 若 `domain_prefix` 拼出的完整域名已被其他流水线占用，返回 `409`。

### 响应 `201 Created`

```json
{
  "data": {
    "id": 8,
    "name": "my-app",
    "repo_id": 12,
    "branch": "main",
    "cluster_id": 1,
    "cluster_name": "prod-cluster",
    "domain_id": 2,
    "custom_domain": "a1b2c3.apps.example.com",
    "image_name": "my-app",
    "notification_type": "lark",
    "notification_webhook": "https://open.larksuite.com/open-apis/bot/v2/hook/xxx"
  }
}
```

---

## 3b. 设置流水线环境变量（可选）

**`POST /api/external/pipelines/:id/env`**

与网页端流水线详情页「环境变量」功能完全一致：以 KV 形式为指定流水线设置环境变量，单条或多条均可。整体替换该流水线当前的环境变量集合（而非增量合并）。若流水线已有正在运行的 Deployment，保存后会立即更新 ConfigMap 并触发滚动重启使其生效；若尚未部署，则会在首次执行（API 4）时随 Deployment 一起带上。

此接口为可选步骤，可在 API 3（创建流水线）之后、API 4（触发执行）之前调用，也可以在流水线运行过程中随时调用以更新环境变量。

### 路径参数

| 参数 | 说明 |
|---|---|
| `:id` | 流水线 ID（API 3 返回的 `id`） |

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `vars` | array | ✅ | KV 条目数组，单条 `[{ "key": "...", "value": "..." }]` 或多条均可；传空数组 `[]` 会清空该流水线的所有环境变量 |

`vars[].key` 只能包含字母、数字、下划线，且不能以数字开头；数组内不允许重复 key，否则返回 `400`。

**系统变量：** 若流水线是 Prisma 项目并已自动分配 MySQL 子账号，`DATABASE_URL` 会被标记为系统变量——本接口的"整体替换"不会清除它（即使传 `vars: []` 也不会），且请求中若包含与系统变量同名的 key 会返回 `403`。系统变量只能由系统管理员在网页端「MySQL 实例 → 子账号」处解除关联后自动移除。

### 响应 `200 OK`

```json
{
  "data": {
    "pipeline_id": 8,
    "keys": ["DATABASE_URL", "LOG_LEVEL"],
    "restarted": true
  }
}
```

`restarted` 为 `true` 表示已对线上 Deployment 触发滚动重启；为 `false` 表示流水线尚未部署，环境变量将在首次执行时生效。若保存成功但同步到 K8S 集群失败，响应中会附带 `warning` 字段说明原因（此时环境变量已保存到数据库，可稍后重试本接口以重新同步）。

### 错误响应

| 状态码 | 原因 |
|---|---|
| `400` | 参数缺失 / `vars` 不是数组 / key 格式非法 / key 重复 |
| `403` | 账号已被禁用 / 请求中包含系统变量同名 key（如 `DATABASE_URL`），无法通过此接口修改 |
| `404` | 流水线不存在 |

---

## 3c. 设置流水线存储（可选）

**`POST /api/external/pipelines/:id/volumes`**

与网页端流水线详情页「存储设置」功能完全一致：为指定流水线配置持久化存储卷（PVC），使用该流水线所在集群的默认存储类（管理员在「K8S 集群管理」页面配置），单条或多条均可，整体替换该流水线当前的存储卷集合（而非增量合并，按挂载路径做匹配）。若流水线已有正在运行的 Deployment，保存后会立即创建/扩容 PVC 并触发滚动重启使其生效；若尚未部署，则会在首次执行（API 4）时随 Deployment 一起挂载。

此接口为可选步骤，可在 API 3（创建流水线）之后、API 4（触发执行）之前调用，也可以在流水线运行过程中随时调用以更新存储配置。

**限制：**
- 单个流水线最多配置 **3** 块存储卷
- 单块存储卷容量不超过 **100Gi**
- 已有存储卷不支持缩容（K8S 本身不支持缩小 PVC），检测到请求容量小于当前记录容量时拒绝保存

### 路径参数

| 参数 | 说明 |
|---|---|
| `:id` | 流水线 ID（API 3 返回的 `id`） |

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `volumes` | array | ✅ | 存储卷条目数组，单条 `[{ "mount_path": "...", "size_gi": ... }]` 或多条均可（最多 3 条）；传空数组 `[]` 会删除该流水线的所有存储卷（对应 PVC 也会被删除） |

`volumes[].mount_path` 必须以 `/` 开头，数组内不允许重复路径；`volumes[].size_gi` 为 ≥1 且 ≤100 的整数（单位 Gi）。

**前置条件（不满足返回 `422`）：** 流水线所在集群已配置默认存储类。

### 响应 `200 OK`

```json
{
  "data": {
    "pipeline_id": 8,
    "volumes": [
      { "mount_path": "/data", "size_gi": 20 },
      { "mount_path": "/cache", "size_gi": 5 }
    ],
    "restarted": true
  }
}
```

`restarted` 为 `true` 表示已对线上 Deployment 触发滚动重启；为 `false` 表示流水线尚未部署，存储卷将在首次执行时生效。若保存成功但同步到 K8S 集群失败（如 PVC 创建/扩容失败），响应中会附带 `warning` 字段说明原因（此时数据库中已记录期望状态，可稍后重试本接口以重新同步）。

### 错误响应

| 状态码 | 原因 |
|---|---|
| `400` | 参数缺失 / `volumes` 不是数组 / 超过 3 块 / 单块超过 100Gi / 路径格式非法或重复 / 容量非正整数 / 尝试缩容 |
| `403` | 账号已被禁用 |
| `404` | 流水线不存在 |
| `422` | 集群尚未配置默认存储类 |

---

## 3d. 查询流水线环境变量 Key 列表

**`POST /api/external/pipelines/:id/env/keys`**

返回指定流水线当前配置的所有环境变量的 **Key**（不返回 value，环境变量值属于敏感信息，与网页端一致不对外暴露）。

### 路径参数

| 参数 | 说明 |
|---|---|
| `:id` | 流水线 ID（API 3 返回的 `id`） |

### 请求参数（加密前 JSON）

无必填字段，传空对象 `{}` 即可。

### 响应 `200 OK`

```json
{
  "data": {
    "pipeline_id": 8,
    "keys": ["DATABASE_URL", "LOG_LEVEL"]
  }
}
```

流水线尚未配置任何环境变量时，`keys` 返回空数组 `[]`。

### 错误响应

| 状态码 | 原因 |
|---|---|
| `403` | 账号已被禁用 |
| `404` | 流水线不存在 |

---

## 4. 触发流水线执行

**`POST /api/external/pipelines/:id/run`**

触发指定流水线立即执行。若流水线当前有正在运行的任务，会先强制终止再启动新任务。

### 请求参数（加密前 JSON）

无业务参数，传空对象即可：`{}`

### 响应 `202 Accepted`

```json
{
  "data": {
    "run_id": 58,
    "stream_token": "<加密令牌>"
  }
}
```

`stream_token` 可直接传给 **API 6** 的 `token` 参数，用于查询本次执行日志，无需重新加密。

---

## 5. 查询用户仓库与流水线

**`POST /api/external/user/pipelines`**

查询指定用户名下所有仓库和流水线的当前状态。

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `username` | string | ✅ | 系统用户名 |

### 响应 `200 OK`

```json
{
  "data": {
    "user": { "id": 5, "username": "hanley" },
    "repos": [
      {
        "id": 12,
        "name": "my-app",
        "git_url": "https://gitlab.diancun.net/it_service/my-app.git",
        "default_branch": "main",
        "scan_status": "done",
        "has_dockerfile": 1,
        "created_at": "2026-06-01 10:00:00"
      }
    ],
    "pipelines": [
      {
        "id": 8,
        "name": "my-app",
        "repo_id": 12,
        "branch": "main",
        "custom_domain": "a1b2c3.apps.example.com",
        "notification_type": "lark",
        "last_run_id": 58,
        "last_run_status": "success",
        "last_run_at": "2026-07-01 15:30:00",
        "created_at": "2026-06-10 09:00:00"
      }
    ]
  }
}
```

---

## 6. 获取执行日志

**`GET /api/external/pipeline-runs/:id/logs?token=<stream_token>`**

一次性返回该次执行当前的完整状态快照（JSON），**不再支持 SSE 实时推送**——无论执行是否已结束，调用即返回当前已有的数据；若仍在执行中，需要自行轮询本接口获取最新进度。

### 查询参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `token` | ✅ | API 5 返回的 `stream_token` |

### 响应 `200 OK`

```json
{
  "data": {
    "run_id": 58,
    "status": "success",
    "progress": 100,
    "current_step": null,
    "image_tag": "20260701153000",
    "started_at": "2026-07-01 15:28:00",
    "finished_at": "2026-07-01 15:30:00",
    "logs": "▶ 拉取代码\n▶ 开始拉取代码...\n✅ 代码拉取完成\n\n▶ 构建镜像\n▶ 构建镜像: ...\n\n...",
    "steps": [
      { "name": "拉取代码",       "status": "success" },
      { "name": "构建镜像",       "status": "success" },
      { "name": "推送镜像",       "status": "success" },
      { "name": "部署到 K8S",     "status": "success" },
      { "name": "检测 Pod 状态",  "status": "success" },
      { "name": "版本回滚",       "status": "skipped" },
      { "name": "申请免费SSL证书","status": "skipped" },
      { "name": "通知",           "status": "success" }
    ]
  }
}
```

`logs` 是按 `steps` 顺序拼接的全量执行日志（每个有输出的步骤前带 `▶ <步骤名>` 标题行），取代了原来挂在每个 `step` 上的单独 `log` 字段。执行仍在进行中时，`logs`/`steps` 只反映调用时刻已产生的内容，需自行轮询直到 `status` 变为 `success` 或 `failed`。

**`step.status` 取值：**

| 值 | 含义 |
|---|---|
| `pending` | 等待执行 |
| `running` | 执行中 |
| `success` | 成功 |
| `failed` | 失败 |
| `skipped` | 跳过（前置步骤失败或条件不满足） |

---

## 7. 获取服务 Pod 控制台日志

**`POST /api/external/pipelines/:id/pod-logs`**

获取指定流水线部署的服务 Pod 最近 300 行控制台日志，一次性返回纯文本，保留 Pod 原始日志格式（含 K8S RFC3339 时间戳前缀）。

**前置条件：** 流水线最近一次执行状态为 `success`，否则返回 `422`。

### 路径参数

| 参数 | 说明 |
|---|---|
| `:id` | 流水线 ID |

### 请求体（加密前 JSON）

无业务参数，传空对象即可：`{}`

### 响应 `200 OK`

```
Content-Type: text/plain; charset=utf-8

2026-07-02T08:00:00.123456789Z Server started on port 3000
2026-07-02T08:00:00.234567890Z Connected to database
2026-07-02T08:00:00.345678901Z Listening on 0.0.0.0:3000
```

直接返回 Pod 日志原始字节，每行格式为 `<RFC3339时间戳> <日志内容>`。读取异常时响应体包含 `[error]` 前缀的错误说明。

### 错误响应

| 状态码 | 原因 |
|---|---|
| `422` | 流水线未部署成功（`last_run_status != success`） |
| `404` | 流水线不存在 / 集群不存在 / Pod 未找到 |
| `502` | K8S API 查询失败 |

---

## 8. 获取免登录访问令牌

**`POST /api/external/auth/token`**

为本系统已有用户签发一个 **2 小时**有效期的登录令牌。用浏览器携带该令牌访问接口 8b，即可免用户名密码登录本平台，权限与该用户正常登录完全一致（同一套 JWT，承载相同的 `user_id`/`username`/`role`）。

**系统管理员（`role=admin`）账号禁止通过该接口获取令牌**，调用会返回 `403`。

### 请求参数（加密前 JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `username` | string | ✅ | 系统用户名 |

### 响应 `200 OK`

```json
{
  "data": {
    "user_id": 5,
    "username": "hanley",
    "role": "user",
    "login_token": "<AES-256-GCM 加密的 base64 字符串>",
    "expires_at": "2026-07-07T14:00:00.000Z"
  }
}
```

`login_token` 是登录 JWT 经 `EXTERNAL_API_KEY`（AES-256-GCM）加密后的结果，解密方式与接口 1 的 `gitlab_token` 一致（见下方解密示例）。**注意**：加密前先对原始值做了 `JSON.stringify`，因此解密得到的 UTF-8 字符串需要再 `JSON.parse()` 一次才是真正的 JWT 字符串，直接使用会带上多余的引号。

```js
import { createDecipheriv } from 'crypto'

function decryptToken(encrypted, hexKey) {
  const buf = Buffer.from(encrypted, 'base64')
  const key = Buffer.from(hexKey, 'hex')
  const iv  = buf.subarray(0, 12)
  const tag = buf.subarray(buf.length - 16)
  const ct  = buf.subarray(12, buf.length - 16)
  const d = createDecipheriv('aes-256-gcm', key, iv)
  d.setAuthTag(tag)
  return Buffer.concat([d.update(ct), d.final()]).toString('utf8')
}

const rawJwt = JSON.parse(decryptToken(data.login_token, process.env.EXTERNAL_API_KEY))
```

### 错误响应

| 状态码 | 原因 |
|---|---|
| `400` | `username` 缺失 |
| `403` | 账号已被禁用 / 该账号是系统管理员，禁止通过此接口获取令牌 |
| `404` | 用户不存在 |

---

## 8b. 用令牌免登录访问页面

**`GET /api/external/auth/consume?token=<rawJwt>&next=<path>`**

这是唯一一个**不走 AES-256-GCM 加密请求体**的接口——设计上就是给浏览器直接跳转的一个普通链接，而不是供后端程序调用。拿到接口 8 解密并 `JSON.parse` 出来的原始 JWT 后，把它拼进这个 URL，交给用户的浏览器打开（或从你自己的系统发一个 302 跳转过去）即可免登录进入本平台。

### 查询参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `token` | ✅ | 接口 8 签发并解密还原后的原始 JWT（不是 `login_token` 加密串本身） |
| `next` | ❌ | 验证成功后跳转的站内路径，默认 `/dashboard`；只接受以单个 `/` 开头的相对路径，传绝对 URL 会被忽略并回退到默认值（防 Open Redirect） |

### 行为

- 校验通过：设置与正常用户名密码登录完全相同的 `aiops_token` Cookie（`httpOnly`，有效期跟随令牌剩余时间），`302` 跳转到 `next`。
- 令牌缺失 / 签名或格式不合法 / 已过期：`302` 跳转到 `/login?error=invalid_token`。
- 令牌合法但账号已被禁用（签发后才被禁用的情况）：`302` 跳转到 `/login?error=account_disabled`。

跳转之后，浏览器就是一个完全正常的已登录会话——刷新、导航到任意页面都不需要再带 `token` 参数，权限与手动登录该账号完全一致，2 小时后 Cookie 对应的 JWT 过期即需要重新走接口 8。

---

## 典型调用流程

```
1. POST /api/external/users/init          { username }
      ↓ 返回 user_id，自动创建 GitLab impersonation token

2. POST /api/external/repos               { username, repo_name }
      ↓ 在 GitLab it_service 组创建仓库，返回 repo.id
      ↓ 异步开始扫描（此时仓库为空）

3. 用户推送代码（含 Dockerfile）到仓库
      ↓

4. 轮询 POST /api/external/user/pipelines { username }
      ↓ 等待 repo.scan_status = "done" 且 has_dockerfile = 1

5. POST /api/external/pipelines           { repo_id }
      ↓ 返回 pipeline.id

5b. [可选] POST /api/external/pipelines/:id/env  { vars: [{ key, value }, ...] }
      ↓ 设置环境变量；此时流水线尚未部署，将在首次执行时随 Deployment 一起生效

5c. [可选] POST /api/external/pipelines/:id/volumes  { volumes: [{ mount_path, size_gi }, ...] }
      ↓ 设置存储卷（最多 3 块，单块 ≤100Gi）；此时流水线尚未部署，将在首次执行时随 Deployment 一起挂载

6. POST /api/external/pipelines/:id/run  {}
      ↓ 返回 run_id + stream_token

7. GET  /api/external/pipeline-runs/:id/logs?token=<stream_token>
      ↓ 一次性返回当前状态快照（含 logs 全量日志），需自行轮询直到 status = success / failed

8. POST /api/external/pipelines/:id/pod-logs  {}
      ↓ 获取服务 Pod 最近 300 行控制台日志（纯文本，含时间戳）
```
