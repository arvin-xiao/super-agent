# super-agent

**中文** | [English](README.en.md)

这是 [dsh-superagent](https://github.com/deepseek-ai/dsh-superagent-prototype) 研究原型的测试仓库。该原型作为本机 [DeepSeek Harness](https://www.deepseek.com/en/harness/)（DSH）客户端的一个独立 profile 运行。

Agent 使用本机 DSH home 和模型凭据，但运行在独立的 `superagent` profile 中，拥有自己的进程和状态目录。它可以在不关闭操作者 Session 的情况下重启和升级，`desktop` profile 不受影响。该原型是研究代码，不是可投入生产的产品。

| 项 | 值 |
| --- | --- |
| Agent 账号 | `@arvin00000001`（评论、分支和 PR 都以该账号发出） |
| 操作者 | `@arvin-xiao` |
| 仓库 | `arvin-xiao/super-agent` |
| DSH profile | `superagent`，Web profile 的独立副本 |
| 面板 | `http://127.0.0.1:3082`（启动 URL 带 token） |
| 状态目录 | `~/.dsh/state/superagent` |

当允许列表中的账号在 Issue 或评论中 @ Agent 时，工作就会到来：

```text
@arvin00000001 add a file called HELLO.md with the text Hello Super-agent
```

大约一分钟后，请求会作为 Ticket 出现在 **Superagent** 面板中。Agent 在自己的 checkout 中工作，并在原对话中回复，通常是一个 PR。

## 基于本地 DSH 客户端搭建

以下步骤说明如何在本机 DSH 客户端上复现这套安装。原型仓库的[部署指南](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md)是各项设置的完整参考；它的[由 Agent 执行的安装流程](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/SETUP.md)是另一种选择，会一次问一个问题。

### 前置条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 GitHub CLI（`gh`）2.48+
- 已安装 DSH 客户端并配置模型提供方；新 profile 复用本机 DSH home 中的模型凭据
- 仅在使用仓库自带的 `scripts/dsh-superagent` 托管服务（而不是直接运行 `dsh`）时需要 `tmux`
- 一个 github.com 仓库，以及一个对该仓库有写权限的专用 GitHub Agent 账号。不支持 GitHub Enterprise Server。
- 可选：带机器人的 Feishu/Lark 应用和 `lark-cli`，用于处理聊天请求

### 1. 安装 DSH CLI

`dsh` 命令负责创建并启动 profile。请安装原型 `engines.dsh` 范围（`^0.2.1-alpha.2`）内的版本；本安装使用 `0.2.1-rc.3`：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
dsh --version
```

CLI 使用 `~/.dsh`（可用 `DSH_HOME` 指定其他目录），与桌面客户端共用同一个 home。

### 2. 构建原型

```sh
git clone https://github.com/deepseek-ai/dsh-superagent-prototype.git ~/src/dsh-superagent
cd ~/src/dsh-superagent
pnpm install
pnpm build
```

`pnpm check` 会运行类型检查、测试和构建，可用于验证 checkout。

### 3. 创建 `superagent` profile

```sh
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

第一条命令从内置 Web 模板创建 `~/.dsh/profiles/superagent`，并打印组合后的配置。第二条命令把当前 checkout 作为 profile 的 `dsh-superagent` bundle 安装进去。第一条命令只用于全新安装：它会替换已有 profile 并删除其 `cordis.patch.yml`。

单独使用一个 profile 是因为该 bundle 会改变其 Session 的进程级 `PATH` 和 git 设置，而且需要独立重启和升级。

### 4. 配置 profile

将 `~/.dsh/profiles/superagent/cordis.patch.yml` 替换为：

```yaml
# superagent profile: dsh-superagent panel configuration.
- id: webserver
  config:
    host: 127.0.0.1
    port: 3082
- id: dsh-superagent
  config:
    stateDir: /Users/me/.dsh/state/superagent
# Feishu is optional: uncomment after configuring the bot's lark-cli.
# - id: dsh-superagent-feishu
#   disabled: false
```

- 把 `/Users/me` 换成你的 home 目录。`stateDir` 必须是绝对路径；它保存数据库、worker 命令 shim 和 git 配置，默认还保存 Ticket checkout。升级前请备份。
- 如果 3082 被占用，请修改 `port`，并把对应 URL 传给服务脚本，例如 `--url http://127.0.0.1:3083`。
- 复制来的 Web profile 使用本机 DSH home 中的模型凭据和 Web 模板的默认模型。若只想为该 profile 选择其他 provider 或模型，可添加一个 `agent-default-model` 条目，包含 `provider`、`model` 和可选的 `reasoningEffort` 键。

### 5. 登录 Agent 的 GitHub CLI

Agent 使用自己的 GitHub 账号，绝不使用操作者的账号。创建一个专用账号，以写权限邀请它为仓库协作者，在登录 Agent 账号的状态下接受邀请，然后为它配置独立的 `gh` 配置：

```sh
mkdir -p ~/.local/bin
cat > ~/.local/bin/gh-agent <<'EOF'
#!/bin/sh
GH_CONFIG_DIR="$HOME/.config-gh-agent" exec gh "$@"
EOF
chmod +x ~/.local/bin/gh-agent
gh-agent auth login
gh-agent api user --jq .login   # must print the agent login
```

### 6. 启动面板

在 checkout 中，用本机 DSH 客户端启动新 profile：

```sh
dsh --profile superagent --no-open
```

进程会打印 `dsh web: http://127.0.0.1:3082/?token=…`。在浏览器中打开该 URL（每次启动都会生成新 token），然后在侧边栏选择 **Superagent**。按 Ctrl-C 停止进程。

如需后台托管服务，可使用仓库自带脚本；它需要 `tmux`：

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 会在 tmux 会话 `dsh-superagent` 中运行 `dsh --profile superagent`，等待 runtime 就绪，并打印带有面板 URL 的日志文件。`status` 的退出码：0 表示服务健康；1 表示服务未运行、重复运行、不可达或 runtime 报错；2 表示某个组件（例如 Source）报错；3 表示仍在等待人工完成设置。`stop` 和 `restart` 会优雅关闭。

### 7. 在面板中接入仓库

在保存这些设置之前，Overview 会显示设置问题，任何请求都无法成为 Ticket：

1. **Settings → Sources → GitHub**：仓库 `owner/name`（这里是 `arvin-xiao/super-agent`），以及 Agent CLI 路径，例如 `/Users/me/.local/bin/gh-agent`。
2. **Settings → Eval → On each Source**：允许发起工作的 GitHub 登录名，这里是 `arvin-xiao`。列表初始为空；列表为空时不会接受任何请求。
3. **Settings → Execution**：Ticket checkout 的目录根，例如 `~/src/super-agent-tickets`，以及 prepare 命令。本安装为每个 Ticket 新建一份 clone：

   ```sh
   git clone "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "$DSH_SUPERAGENT_DIR"
   ```

   Host 会设置 `$DSH_SUPERAGENT_GITHUB_REPOSITORY` 和 `$DSH_SUPERAGENT_DIR`。目录根请使用绝对路径。如果仓库很大，也可以 clone 一份共享仓库并添加 worktree；部署指南中有对应命令。

4. **Settings → Prompts**：团队提示词，包括提交作者、GitHub 披露行和 auto-merge 条件。在 Agent 的 PR 稳定可靠之前，请保持关闭 auto-merge。
5. 可选 Feishu：按部署指南配置 bot CLI 与聊天，然后在 profile 配置中启用 `dsh-superagent-feishu` 条目。

部分保存会重启应用；等面板恢复响应后再做下一次修改。Overview 的设置卡片会链接到仍缺少的配置。

### 8. 验证搭建

```sh
scripts/dsh-superagent status   # exits 0 when setup is complete
```

然后，用允许列表中的账号在 Issue 中评论：

```text
@arvin00000001 add a file called HELLO.md with the text Hello Super-agent
```

在一个轮询周期内（默认 60 秒），Ticket 会出现在面板中，Agent 会在该线程中确认收到。

## 运维

- **面板**：打开启动 URL；侧边栏中有 **Superagent**。管理 API 位于 `http://127.0.0.1:3082/orchestrator/api/`，请求需要带 `x-orchestrator-request: 1` 头。
- **日志**：`scripts/dsh-superagent` 会写入 `.artifacts/dsh-superagent-<profile>-<date>-<time>-<pid>.log`，并保留最新 20 个文件。直接运行 `dsh` 时，同样的信息会打印在终端。
- **升级**：等待没有 Executor 正在执行一轮工作，备份 `<stateDir>/orchestrator.sqlite`，运行 `git pull && pnpm install && pnpm build`，然后重启并检查 `status`。被新版本迁移过的数据库无法由旧版本打开。
- **安全**：Session 以操作者的 OS 用户身份、`danger-full-access` 权限运行，可以读取 home 目录。若需隔离，请用只保存 Agent 凭据的专用 OS 账号运行服务。

## 故障排查

| 现象 | 处理 |
| --- | --- |
| `The GitHub Source names no repository.` | 在 Settings → Sources → GitHub 设置 `owner/name` |
| 找不到 Agent 的 GitHub CLI，或无法读取其账号 | 检查 wrapper 路径；运行 `gh-agent auth login` |
| `@… cannot push to …` | 给 Agent 账号该仓库的写权限 |
| Ticket 失败并提示 `The prepare command failed` | 错误输出会说明原因；修正命令后重试 Ticket |
| `status` 退出码 3 | 服务正在运行但在等待设置；Overview 会指明缺少哪项设置 |

部署指南的 Troubleshooting 一节覆盖其余设置与运行时错误。

## 卸载

先停止服务，然后移除 bundle 和 profile 状态：

```sh
dsh plugin --profile superagent remove dsh-superagent
rm -rf ~/.dsh/profiles/superagent ~/.dsh/state/superagent
```

`npm uninstall -g @deepseek-ai/dsh` 只在没有其他用途时移除 CLI。

## 文档

- [部署](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md)：所有设置、启动检查与故障排查条目
- [面板](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/005-panel.md)：各个页面及其展示内容
- [理念](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/000-philosophy.md)：Agent 遵循的注意力原则
- [由 Agent 执行的安装流程](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/SETUP.md)：原型要求编码 Agent 执行的安装步骤
