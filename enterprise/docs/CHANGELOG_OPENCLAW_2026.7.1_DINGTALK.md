# OpenClaw 2026.7.1-2 升级与钉钉绑定变更记录

- 日期：2026-08-14
- 状态：已上线验证
- 适用范围：OpenClaw Enterprise
- 当前镜像：`openclaw-2026.7.1-2-fix1`

## 1. 变更目标

本次变更解决以下问题：

1. 将企业版 OpenClaw 从 `2026.3.24` 升级到稳定版本 `2026.7.1-2`。
2. 固定 Node.js、ClawHub 和 Amazon Bedrock provider 版本，保证构建可复现。
3. 修复 AgentCore 容器中 Gateway 无法启动以及 UI 返回 `LLM request failed.`。
4. 兼容 OpenClaw 2026.7 的新消息结构，使钉钉 `/start TOKEN` 能正确进入企业绑定流程。
5. 保留版本化镜像和旧镜像，支持生产回滚。

## 2. 固定版本

| 组件 | 版本 |
|---|---|
| OpenClaw | `2026.7.1-2` |
| Node.js | `24.15.0` |
| ClawHub | `0.23.3` |
| Amazon Bedrock provider | `2026.7.1` |
| 生产镜像 tag | `openclaw-2026.7.1-2-fix1` |
| 回滚镜像 tag | `openclaw-2026.7.1-2` |

所有生产依赖均使用精确版本，不使用 `latest`。

## 3. OpenClaw 升级改动

### 3.1 版本化、可回滚部署

`enterprise/deploy.sh` 现在支持：

- `bash enterprise/deploy.sh`：初次部署或完整更新。
- `bash enterprise/deploy.sh --build-only`：只构建、验证和推送候选镜像，不切换生产 Runtime。
- `bash enterprise/deploy.sh --skip-build --skip-seed`：复用 ECR 镜像更新现有 Stack、Runtime 和 EC2 服务，不重置种子数据。
- `IMAGE_TAG=<tag>`：显式选择部署或回滚版本。

构建候选镜像时不会自动修改 AgentCore Runtime、ECS 或 EC2 Gateway。镜像通过 smoke test 后才推送到 ECR。

### 3.2 AgentCore 容器修复

升级后 UI 聊天曾返回 `LLM request failed.`。调用链检查确认 Portal、Tenant Router 和 AgentCore API 均正常，错误来自容器内 OpenClaw。

根因：

- OpenClaw 2026.7 将 Amazon Bedrock 支持作为独立 provider 加载。
- 最终 runtime 镜像未包含 `@openclaw/amazon-bedrock-provider`。
- runtime 镜像中的 npm 链接无效，Gateway 无法自动安装 provider。
- Gateway 启动失败后，CLI 仍能返回 JSON，但回复内容是 `LLM request failed.`。

修复：

- 固定安装 `@openclaw/amazon-bedrock-provider@2026.7.1`。
- 将 OpenClaw 受管 provider npm project 复制到最终镜像。
- 在 runtime stage 重建 npm、npx 和 OpenClaw CLI 符号链接。
- 增加 `smoke-test.sh`，验证 Node、OpenClaw、provider 和 Gateway ready endpoint。
- Gateway 启动失败时输出经过筛选和脱敏的诊断信息。
- OpenClaw stderr 只记录安全错误分类，不记录用户消息、工具 payload 或凭据。

### 3.3 升级前后的运行架构

升级前后，企业版对外 API 和租户路由保持不变，但容器内部的 OpenClaw 运行方式发生了关键变化。

升级后的 serverless 调用链：

```text
Portal / IM
  → EC2 Tenant Router
  → Amazon Bedrock AgentCore Runtime
  → /app/server.py
  → openclaw agent --session-id <tenant> --message <text> --json
  → 本容器 OpenClaw Gateway (127.0.0.1:18789)
  → @openclaw/amazon-bedrock-provider
  → Amazon Bedrock Converse API
```

各组件职责：

- `tenant_router.py`：根据员工、岗位和部署模式选择 AgentCore Runtime 或 Fargate endpoint。
- `server.py`：完成 workspace 装配、Skill 加载、Guardrail、OpenClaw CLI 调用、usage/audit/memory 写入。
- OpenClaw CLI：使用固定 `session-id` 调用本容器 Gateway，保留 OpenClaw 原生 session、memory compaction 和 tool orchestration。
- OpenClaw Gateway：加载模型 provider、workspace、Skills 和工具，并负责实际 agent run。
- Bedrock provider：把 OpenClaw 的模型请求转换为 Bedrock Converse/ConverseStream 调用。

没有改成自研 WebSocket 客户端，仍使用官方 `openclaw agent --json` CLI 作为企业适配边界，从而降低与 OpenClaw 内部 Gateway protocol 耦合。

### 3.4 Node.js 与 OpenClaw engine 兼容

OpenClaw `2026.7.1-2` 对 Node.js engine 有明确要求。可兼容的主版本范围包括：

```text
>=22.22.3 <23
>=24.15.0 <25
>=25.9.0
```

本项目选择 `Node.js 24.15.0`，原因是：

- 满足 OpenClaw engine 下限。
- 使用 LTS 主版本线，避免直接采用 Node 25。
- 可使用 Node 的 V8 compile cache。
- ARM64 二进制可直接运行在 Graviton EC2、AgentCore 和 Fargate 镜像中。

Docker 构建不依赖发行版自带 Node 包，而是下载精确版本 tarball，并在构建时执行：

```bash
node --version
openclaw --version
openclaw agent --help
clawhub --cli-version
```

任何版本不匹配都会中止构建。

### 3.5 Bedrock provider 加载机制变化

OpenClaw 2026.7 不再仅依靠主包内置逻辑处理 Bedrock。Gateway 会发现并加载独立插件：

```text
@openclaw/amazon-bedrock-provider@2026.7.1
```

OpenClaw 将其安装到受管 npm project，而不是普通全局模块：

```text
/root/.openclaw/npm/projects/
  openclaw-amazon-bedrock-provider-<managed-id>/
    package.json
    package-lock.json
    node_modules/@openclaw/amazon-bedrock-provider/
```

该 project 还包含 OpenClaw 管理的依赖 overrides，用于约束 AWS SDK、OpenTelemetry、HTTP 和压缩相关依赖版本。因此不能只把 provider package 随意复制到全局 `node_modules`；镜像构建阶段按固定版本创建受管 project，并将整个 project 复制到 runtime stage。

最终镜像仍保留可用的 `npm`/`npx`，用于 OpenClaw runtime 的受管插件机制，但生产启动不依赖在线下载 provider。

### 3.6 多阶段 Docker 镜像设计

镜像使用 builder/runtime 两阶段：

#### Builder stage

- 下载并安装 Node.js `24.15.0`。
- 全局安装 OpenClaw `2026.7.1-2` 和 ClawHub `0.23.3`。
- 创建受管 npm project 并精确安装 Bedrock provider `2026.7.1`。
- 预装企业内置 Skills。
- 安装 Python requirements。
- 预热 V8 compile cache。

#### Runtime stage

- 保留 Python 3.12、AWS CLI、boto3、requests、curl 和 jq。
- 从 builder 复制 Node binary、全局 Node modules、受管 provider project、Skills、模板和 compile cache。
- 显式重建以下符号链接：

```text
/usr/local/bin/npm      → /usr/lib/node_modules/npm/bin/npm-cli.js
/usr/local/bin/npx      → /usr/lib/node_modules/npm/bin/npx-cli.js
/usr/local/bin/openclaw → /usr/lib/node_modules/openclaw/openclaw.mjs
```

这里必须重建链接，因为 Docker `COPY` 会解引用源符号链接。曾经直接复制 `/usr/bin/npm`，结果 runtime 中 npm 以普通脚本存在，其相对引用 `../lib/cli.js` 指向错误位置，smoke test 报 `Cannot find module '../lib/cli.js'`。

### 3.7 容器启动顺序

`entrypoint.sh` 的核心启动顺序：

1. 设置 Node compile cache 和 IPv4-first DNS。
2. 选择 S3 workspace 或 EFS workspace。
3. 生成 `/root/.openclaw/openclaw.json`，替换 region 和 model ID。
4. 对 always-on 场景预装配 workspace。
5. 注入可选 IM channel 配置。
6. 启动 `openclaw gateway --port 18789`。
7. 等待 Gateway control endpoint ready。
8. 启动 AgentCore HTTP server `/app/server.py`。
9. 后台执行 S3 同步、workspace assembler 和 Skill loader。

Gateway ready 检查使用：

```text
http://127.0.0.1:18789/__openclaw/control-ui-config.json
```

AgentCore 模式最多等待 30 秒；Fargate 为降低健康检查启动时间只等待 5 秒，Gateway 继续在后台启动。

如果 Gateway 未 ready，日志会记录：

- Gateway PID 是否仍存活。
- 提前退出时的 exit code。
- 最后若干条经过关键词筛选和脱敏的启动错误。

日志会过滤 URL，并遮蔽 token、secret、password、authorization 和 API key 等常见敏感字段。

### 3.8 OpenClaw CLI 响应兼容

`server.py` 继续调用：

```bash
openclaw agent \
  --session-id <tenant_id> \
  --message <message> \
  --json \
  --timeout <seconds>
```

升级后的 CLI 可能返回两种 JSON 结构：

```json
{"payloads": [{"text": "..."}], "meta": {}}
```

或 Gateway wrapper：

```json
{"runId": "...", "result": {"payloads": [{"text": "..."}], "meta": {}}}
```

服务端会先解包 `result`，再提取 `payloads[].text`，同时兼容顶层 `text`。OpenClaw 的信息和 warning 可能写入 stderr，因此实现不会把“stderr 非空”直接当作失败；只有无可解析 JSON、超时或 subprocess 异常才进入 retry/error 分支。

为防止敏感数据泄漏，stderr 不直接写入日志，只记录：

- OpenClaw 版本。
- subprocess exit code。
- stdout/stderr 字节数。
- 安全错误分类，例如 `runtime_npm_missing`、`bedrock_provider_error`、`bedrock_access_denied`、`bedrock_validation_error` 或 `provider_timeout`。

### 3.9 镜像 smoke test

候选镜像推送 ECR 前必须通过 `/app/smoke-test.sh`：

1. 校验 Node 精确版本。
2. 校验 OpenClaw 精确版本。
3. 确认 `openclaw agent` 仍支持 `--json`。
4. 确认 npm 可执行。
5. 从受管 project 中读取 provider `package.json`，校验 provider 精确版本。
6. 生成最小 OpenClaw 配置。
7. 实际启动 Gateway。
8. 在 45 秒内轮询 Gateway ready endpoint。

CLI `--help` 通过但 Gateway 无法启动的镜像不会被推送。这一检查正是本次发现 runtime npm 链接问题的关键。

### 3.10 AgentCore Runtime 更新机制

镜像通过验证并推送后，部署脚本调用 `update-agent-runtime`，更新：

- `containerUri`：指向版本化 ECR tag。
- execution role。
- network mode。
- protocol configuration。
- lifecycle configuration。
- Runtime environment variables。

关键 lifecycle 参数：

```json
{
  "idleRuntimeSessionTimeout": 300,
  "maxLifetime": 3600
}
```

更新附加 Runtime 时，脚本先读取 Runtime role，只处理角色名属于当前 Stack 前缀的资源，避免误更新同账号其他系统的 Runtime。

镜像更新只影响新 microVM。已有 AgentCore session 可能继续运行旧镜像，因此配置或镜像切换后需要调用 `stop_runtime_session`。本次上线停止了员工旧 session，下一次请求重新创建 microVM 并拉取 `fix1`。

### 3.11 EC2 Gateway 与 AgentCore 镜像的差异

企业部署中存在两类 OpenClaw：

| 位置 | 用途 | 安装方式 |
|---|---|---|
| EC2 Gateway | 接收 IM、运行钉钉等 channel plugin、转发到 H2 Proxy | NVM + 全局 npm |
| AgentCore/Fargate 容器 | 执行租户隔离的 Agent、Skills、Memory 和 Bedrock 调用 | 版本化 Docker 镜像 |

EC2 Gateway 可在持久磁盘中自动创建受管 provider project；AgentCore microVM 是短生命周期环境，不能依赖首次请求时动态安装。因此 provider 必须预装进镜像。

升级时两侧都固定 OpenClaw/Node 版本，但发布方式不同：

- 容器侧通过 ECR tag 和 AgentCore Runtime 更新。
- EC2 侧由 `ec2-setup.sh` 使用 NVM 安装指定版本，并重建 Gateway systemd user service。

Gateway service 会注入：

```text
AWS_ENDPOINT_URL_BEDROCK_RUNTIME=http://localhost:8091
```

因此 EC2 Gateway 的 Bedrock 请求进入 H2 Proxy，再由企业路由决定 AgentCore/Fargate 目标。

### 3.12 生产切换与验证

已将默认 AgentCore Runtime 更新到 `openclaw-2026.7.1-2-fix1`，并停止旧会话以确保新请求拉取新镜像。

验证结果：

- Runtime 状态：`READY`
- Portal → Tenant Router → AgentCore → OpenClaw → Bedrock 完整链路成功
- 最小请求返回：`OK`
- 模型：`zai.glm-5`
- OpenClaw 返回正常 token usage

## 4. 钉钉绑定修复

### 4.1 问题现象

员工在 Portal 生成绑定 token 后，通过钉钉发送：

```text
/start <TOKEN>
```

钉钉返回 `LLM request failed.`，命令没有进入绑定确认流程。

### 4.2 根因

OpenClaw 2026.7 在真实用户消息后追加一个 synthetic user block：

```text
OpenClaw runtime context for the immediately preceding user message.
<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>
...
<<<END_OPENCLAW_INTERNAL_CONTEXT>>>
```

旧 H2 Proxy 总是读取最后一个 user block，因此：

- 丢失真实 `/start TOKEN`。
- 将内部上下文误识别为 `channel=actions`。
- 请求错误进入 AgentCore/LLM，而不是企业绑定流程。

### 4.3 修复内容

`enterprise/gateway/bedrock_proxy_h2.js` 已更新：

- 从后向前查找最新的真实用户消息。
- 跳过 OpenClaw synthetic runtime context block。
- 扫描全部 JSON metadata block，不再假设第一个 JSON 就是 inbound metadata。
- 将 `dingtalk-connector` 规范化为 `dingtalk`。
- 从 trusted inbound metadata 提取 channel 和 sender ID。
- 禁止将内部 `actions` 识别为真实 channel/user。
- `/start TOKEN` 无论 token 是否有效，都不会再落入 LLM。
- token 无效、过期、绑定服务不可用或身份信息不完整时返回明确错误。
- 移除会记录完整 system prompt、用户消息或绑定 token 的 DEBUG 日志。

### 4.4 绑定流程

1. 员工在 Portal 生成一次性 token，有效期 15 分钟。
2. 员工在钉钉发送 `/start <TOKEN>`。
3. H2 Proxy 调用 `pair-pending` 校验 token 和钉钉账号。
4. Bot 返回绑定身份信息，并要求发送 `BIND`。
5. 员工发送 `BIND`。
6. H2 Proxy 调用 `pair-complete`。
7. 系统写入 DynamoDB 用户映射，并保留 SSM 兼容映射。
8. 后续钉钉消息解析到对应企业员工和 AgentCore session。

绑定取消命令为 `CANCEL`。当前确认词是 `BIND`，不是 `YES`。

### 4.5 验证结果

已使用与 OpenClaw 2026.7 相同的双 user-block payload 做回归测试：

- 正确识别 `channel=dingtalk`。
- 正确识别 sender ID。
- 无效测试 token 被绑定分支拦截。
- 测试请求未进入 LLM，也未创建绑定。
- 真实钉钉 `/start`、`BIND` 和后续对话均已验证成功。

## 5. 改动后的功能变化

### 5.1 Gateway 和模型调用

- AgentCore 容器可以正常启动 OpenClaw Gateway。
- Bedrock provider 在镜像构建时预装，不依赖 runtime 临时下载。
- UI 和 IM 请求继续使用 `zai.glm-5`。
- Portal 与钉钉最终映射到员工身份，保持企业权限、SOUL、Skills、Memory 和审计链路。

### 5.2 钉钉连续消息

当前钉钉插件版本为 `0.8.24`。同一会话连续发送多条消息时：

- 上一条仍在处理，新消息会进入插件队列。
- 插件立即显示“上一条仍在处理，新消息已排队”之类的提示。
- 轮到该消息后，提示卡片会更新为最终回复。

这是钉钉连接器自 `0.8.3` 起提供的队列反馈，不是 AgentCore 拒绝请求。企业链路通常需要 8–16 秒，因此比单机部署更容易触发该提示。保留串行队列可以避免同一 session 中上下文乱序、重复工具执行和记忆覆盖。

### 5.3 安全与可观测性

- Gateway 失败日志经过筛选和脱敏。
- OpenClaw stderr 只记录错误分类和字节数。
- H2 Proxy 不再记录完整 system prompt、用户消息或绑定 token。
- 绑定命令不会发送给模型。
- Runtime 更新限制为当前 Stack 所属 IAM role，避免修改其他 Stack 的 Runtime。

## 6. 主要修改文件

- `enterprise/deploy.sh`
- `enterprise/.env`
- `enterprise/.env.example`
- `enterprise/agent-container/Dockerfile`
- `enterprise/agent-container/entrypoint.sh`
- `enterprise/agent-container/server.py`
- `enterprise/agent-container/smoke-test.sh`
- `enterprise/gateway/bedrock_proxy_h2.js`
- `enterprise/ec2-setup.sh`

## 7. 后续部署

现有 Stack 正式同步仓库改动：

```bash
bash enterprise/deploy.sh --skip-build --skip-seed
```

不要使用 `--skip-services`，否则 EC2 H2 Proxy 改动不会同步。

完整部署或重建：

```bash
bash enterprise/deploy.sh
```

当前 `.env` 应包含：

```dotenv
OPENCLAW_VERSION=2026.7.1-2
NODE_VERSION=24.15.0
CLAWHUB_VERSION=0.23.3
BEDROCK_PROVIDER_VERSION=2026.7.1
IMAGE_TAG=openclaw-2026.7.1-2-fix1
```

## 8. 回滚

若 `fix1` 出现异常，可将 `.env` 中的 tag 改回：

```dotenv
IMAGE_TAG=openclaw-2026.7.1-2
```

然后执行：

```bash
bash enterprise/deploy.sh --skip-build --skip-seed
```

注意：旧镜像缺少本次 AgentCore Gateway/provider 修复，只建议用于紧急回滚和诊断。

EC2 H2 Proxy hotfix 部署时保留了上一版本备份：

```text
/home/ubuntu/bedrock_proxy_h2.js.pre-im-context-fix
```

恢复该文件后需要重启：

```bash
sudo systemctl restart bedrock-proxy-h2
```

## 9. 已知事项

- EC2 H2 Proxy fast-path 当前缺少 `@aws-sdk/client-bedrock-runtime`，会回退到 Tenant Router/AgentCore；不影响已验证的正常对话和绑定流程，但增加首次响应延迟。
- 钉钉 AI Card API 曾返回 403，连接器会自动降级为普通消息；这与模型调用和账号绑定无关。
- 钉钉连接器的 busy ACK 是插件设计行为。若需要调整，应保留串行队列，仅定制提示或关闭即时 ACK，不建议对同一员工 session 开启真实并发。
