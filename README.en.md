# super-agent

[中文](README.md) | **English**

Test repository for the [dsh-superagent](https://github.com/deepseek-ai/dsh-superagent-prototype)
research prototype, running as an independent profile of the local
[DeepSeek Harness](https://www.deepseek.com/en/harness/) (DSH) client.

The agent uses the local DSH home and model credentials, but runs in its own `superagent`
profile with its own process and state directory. It can restart and upgrade without
closing the operator's Sessions, and the `desktop` profile is left untouched. The
prototype is research code, not a production-ready product.

| Item | Value |
| --- | --- |
| Agent account | `@arvin00000001` (comments, branches and pull requests are written under this account) |
| Operator | `@arvin-xiao` |
| Repository | `arvin-xiao/super-agent` |
| DSH profile | `superagent`, a separate copy of the Web profile |
| Panel | `http://127.0.0.1:3082` (the startup URL carries a token) |
| State directory | `~/.dsh/state/superagent` |

Work arrives when an account on the allowed list mentions the agent in an Issue or comment:

```text
@arvin00000001 add a file called HELLO.md with the text Hello Super-agent
```

Within about a minute the request appears as a Ticket in the **Superagent** panel. The
agent works in its own checkout and answers in the original conversation, usually with a
pull request.

## Set up from the local DSH client

These steps reproduce this installation on a local DSH client. The prototype's
[deployment guide](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md)
is the full reference for every setting, and its
[agent-driven setup](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/SETUP.md)
is an alternative that asks one question at a time.

### Prerequisites

- Node.js 22.19+ or 24+, pnpm, git 2.32+, Python 3.8+ and GitHub CLI (`gh`) 2.48+
- A DSH client installation with a model provider configured; the new profile reuses the
  model credentials in the local DSH home
- `tmux` only if you use the bundled `scripts/dsh-superagent` service instead of running
  `dsh` directly
- A github.com repository, and a dedicated GitHub agent account with write access to it.
  GitHub Enterprise Server is not supported.
- Optional: a Feishu/Lark app with a bot and `lark-cli` for chat requests

### 1. Install the DSH CLI

The `dsh` command creates and boots the profile. Install a version within the prototype's
`engines.dsh` range (`^0.2.1-alpha.2`); this installation uses `0.2.1-rc.3`:

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
dsh --version
```

The CLI uses `~/.dsh` (set `DSH_HOME` to use another directory), the same home as the
desktop client.

### 2. Build the prototype

```sh
git clone https://github.com/deepseek-ai/dsh-superagent-prototype.git ~/src/dsh-superagent
cd ~/src/dsh-superagent
pnpm install
pnpm build
```

`pnpm check` runs the typecheck, the tests and the build if you want to verify the
checkout.

### 3. Create the `superagent` profile

```sh
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

The first command creates `~/.dsh/profiles/superagent` from the shipped Web template and
prints the composed configuration. The second adds this checkout as the profile's
`dsh-superagent` bundle. Run the first command only for a new installation: it replaces an
existing profile and deletes its `cordis.patch.yml`.

The profile is separate because the bundle changes process-wide `PATH` and git settings
for its Sessions, and it needs independent restarts and upgrades.

### 4. Configure the profile

Replace `~/.dsh/profiles/superagent/cordis.patch.yml` with:

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

- Replace `/Users/me` with your home directory. `stateDir` must be an absolute path; it
  holds the database, the worker command shims and git configuration, and by default the
  Ticket checkouts. Back it up before upgrades.
- Change `port` if 3082 is taken, and pass the matching URL to the service script,
  for example `--url http://127.0.0.1:3083`.
- The copied Web profile uses the model credentials in the local DSH home and the Web
  template's default model. To choose another provider or model for this profile only,
  add an `agent-default-model` entry with `provider`, `model` and optional
  `reasoningEffort` keys.

### 5. Sign in the agent's GitHub CLI

The agent acts as its own GitHub account, never the operator's. Create a dedicated
account, invite it as a collaborator with write access to the repository, accept the
invitation while signed in as the agent, and give it a `gh` configuration separate from
yours:

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

### 6. Start the panel

From the checkout, boot the new profile with the local DSH client:

```sh
dsh --profile superagent --no-open
```

The process prints `dsh web: http://127.0.0.1:3082/?token=…`. Open that URL in the browser
(a new token is generated on each start) and choose **Superagent** in the sidebar. Stop
the process with Ctrl-C.

For a managed background service, use the bundled script; it requires `tmux`:

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` runs `dsh --profile superagent` in tmux session `dsh-superagent`, waits for the
runtime to answer, and prints the log file with the panel URL. `status` exits 0 when the
service is healthy, 1 when it is down, duplicated, unreachable or reporting a runtime
error, 2 when a component such as a Source reports an error, and 3 when setup still waits
for a person. `stop` and `restart` shut down gracefully.

### 7. Connect the repository in the panel

Until these settings are saved, setup reports a problem on Overview and no request can
become a Ticket:

1. **Settings → Sources → GitHub**: repository `owner/name`, here
   `arvin-xiao/super-agent`, and the agent CLI path, for example
   `/Users/me/.local/bin/gh-agent`.
2. **Settings → Eval → On each Source**: the GitHub logins that may ask for work, here
   `arvin-xiao`. The list starts empty, and no request is accepted while it is empty.
3. **Settings → Execution**: a directory root for Ticket checkouts, for example
   `~/src/super-agent-tickets`, and a prepare command. This installation gives each
   Ticket a fresh clone:

   ```sh
   git clone "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "$DSH_SUPERAGENT_DIR"
   ```

   The host sets `$DSH_SUPERAGENT_GITHUB_REPOSITORY` and `$DSH_SUPERAGENT_DIR`. Use
   absolute paths in the directory root. For a large repository, fetch into one shared
   clone and add a worktree instead; the deployment guide has that command.

4. **Settings → Prompts**: the team sections, including the commit author, the GitHub
   disclosure line and the auto-merge condition. Keep auto-merge off until the agent's
   pull requests have been right for a while.
5. Optional Feishu: configure the bot CLI and chats as described in the deployment guide,
   then enable the `dsh-superagent-feishu` entry in the profile configuration.

Some saves restart the application; wait until the panel answers again before the next
change. The Overview setup card links to whatever is still missing.

### 8. Verify the setup

```sh
scripts/dsh-superagent status   # exits 0 when setup is complete
```

Then, from an allowed account, comment on an Issue:

```text
@arvin00000001 add a file called HELLO.md with the text Hello Super-agent
```

Within a polling interval (60 seconds by default) the Ticket appears in the panel, and the
agent acknowledges it in the thread.

## Operate

- **Panel**: open the startup URL; **Superagent** is in the sidebar. The management API
  answers under `http://127.0.0.1:3082/orchestrator/api/` when the
  `x-orchestrator-request: 1` header is present.
- **Logs**: `scripts/dsh-superagent` writes `.artifacts/dsh-superagent-<profile>-<date>-<time>-<pid>.log`
  and keeps the newest 20. When running `dsh` directly, the process prints the same
  information to its terminal.
- **Upgrade**: wait until no Executor is in the middle of a turn, back up
  `<stateDir>/orchestrator.sqlite`, run `git pull && pnpm install && pnpm build`, then
  restart and inspect `status`. A database migrated by a newer version cannot be opened
  by an older one.
- **Security**: Sessions run with `danger-full-access` as the operator's OS user and can
  read the home directory. For isolation, run the service as a dedicated OS account that
  holds only the agent's credentials.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `The GitHub Source names no repository.` | Set `owner/name` on Settings → Sources → GitHub |
| The agent GitHub CLI was not found or cannot read its account | Check the wrapper path; run `gh-agent auth login` |
| `@… cannot push to …` | Give the agent account write access to the repository |
| A Ticket fails with `The prepare command failed` | The error output names the cause; fix the command and retry the Ticket |
| `status` exits 3 | The service runs and waits for setup; Overview names the missing setting |

The deployment guide's Troubleshooting section covers the remaining setup and runtime
errors.

## Uninstall

Stop the service first, then remove the bundle and the profile's state:

```sh
dsh plugin --profile superagent remove dsh-superagent
rm -rf ~/.dsh/profiles/superagent ~/.dsh/state/superagent
```

`npm uninstall -g @deepseek-ai/dsh` removes the CLI only when nothing else uses it.

## Documentation

- [Deployment](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md):
  every setting, setup check and troubleshooting entry
- [Panel](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/005-panel.md):
  the pages and what they show
- [Philosophy](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/000-philosophy.md):
  the attention principle the agent follows
- [Agent-driven setup](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/SETUP.md):
  the setup procedure the prototype asks a coding agent to run
