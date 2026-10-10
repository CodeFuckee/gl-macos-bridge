# gl-macos-bridge

把自建 GitLab（`home.chenkaidi.top:509`）上只能在 macOS 执行的 CI 作业，
桥接到 **GitHub Actions 托管 macOS runner** 上执行。一台常断网的 MacBook 不再是瓶颈。

## 工作原理

```
GitLab 作业（macos-bridge-*，跑在 linux-x64 shell runner 上）
   │ ci/bridge.sh：POST repository_dispatch（脚本 b64 + 上下文）
   ▼
本仓库 gl-macos.yml（runs-on: macos-15，一次性 VM）
   │ ① resolve：PROJECTS allowlist 路由 + runner 标签白名单校验
   │ ② 按 SHA 从 GitLab 克隆源码（GL_PAT_<id> secret）
   │ ③ 执行 payload 里的作业脚本（注入白名单 CI_* 变量）
   │ ④ dist/* 回传 GitLab generic package registry
   ▼
GitLab ci/bridge.sh 轮询 run 终态 → 镜像为作业退出码（流水线门禁原生生效）
```

- **关联键**：`correlation_id = gl<项目ID>-p<流水线ID>j<作业ID>`，同时是 run-name 后缀。
- **取消传播**：GitLab 取消流水线 → bridge.sh 的 trap 调 cancel API；同项目+分支+作业的
  新 run 也会因 `concurrency.cancel-in-progress` 取代旧 run。
- 详细方案：见 work-review 仓库 `docs/ci/macos-bridge.md`。

## 秘密与凭据（四段式）

| 凭据 | 值 | 存放 | 权限 |
| --- | --- | --- | --- |
| GitHub 触发 PAT | （建议 fine-grained，仅本仓库） | GitLab 项目 CI 变量 `GH_BRIDGE_TOKEN`（masked+protected） | contents: read/write（dispatch 必需） |
| GitLab 项目 token | `GL_PAT_131`（项目 131 的 project access token，Developer 级，api scope） | 本仓库 Actions secret | 克隆源码 + 上传 package registry |
| 作业内容 | script_b64 | payload（无凭据） | — |

- payload 视为**半公开数据**：只放脚本与元数据，绝不放 token。
- `GL_PAT_131` 当前到期 **2027-01-08**，到期前在 GitLab 重建并更新本 secret。
- 本仓库若保持 public：GitHub 标准托管 runner（含 macOS）**免费**；代价是 dispatch
  payload 任何能读本仓库的人可见（脚本与 work-review 公开镜像内容同源，可接受）。
  若未来桥接含私有信息的作业，改 private（macOS 分钟按 10× 计费）。

## 新接入一个 GitLab 仓库

1. GitLab 侧：项目里建 project access token（Developer 级，`api` scope，≤1 年期）。
2. 本仓库：Settings → Secrets → Actions 添加 `GL_PAT_<项目ID>`。
3. 编辑 `.github/workflows/gl-macos.yml` 顶部 `PROJECTS` JSON 加一行（默认分支生效）。
4. GitLab 仓库里放 `ci/bridge.sh` + 目标脚本，作业 `extends` 桥接模式（见 work-review）。

## 安全红线

- 不添加 `pull_request` / `pull_request_target` 触发器（fork 可读 secrets）。
- 不引入第三方 action（当前纯 run 步骤，保持零依赖）。
- payload 字段永远经 `env:` 间接进入 shell，禁止 `${{ }}` 直插。
- `runs-on` 只消费 allowlist 白名单校验后的值。

## 当前接入项目

| GitLab project_id | 项目 | runner | 备注 |
| --- | --- | --- | --- |
| 131 | chenkaidi/work-review | macos-15 | 试点：macos-bridge-smoke |
