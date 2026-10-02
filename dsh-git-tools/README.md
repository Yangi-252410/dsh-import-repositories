> 本插件创建于 2026年10月2日 10:33，配套的快捷命令让你能直接联通 GitHub 仓库。

# dsh-git-tools

给 DeepSeek Harness 的 agent 用的 Git 工具，git 在**宿主进程**中执行。

## 为什么需要它

agent 的 shell 运行在 `workspace-write` 文件沙箱里，实测该环境**无法访问网络**：
受限进程拿不到 TLS 凭据句柄，任何 `https://` 请求都会失败（包括 GitHub、gitee、百度）。
同时 **git 不读 Windows 系统代理（WinINET）**，所以即使 FLClash 开着系统代理，
沙箱内的 `git fetch` / `pull` / `push` 依然失败。

本插件通过 `ctx.subprocess` 在宿主进程里 spawn git。宿主进程不受 agent 沙箱约束，
因此这些联网操作可以正常工作。这与官方 `dsh-workspace-changes` 插件执行
`git diff-tree` 的方式一致。

## 提供的工具

| 工具 | 作用 | 是否联网 |
|---|---|---|
| `git_status` | 分支、upstream、ahead/behind、逐文件状态 | 否 |
| `git_diff` | 工作区 / 暂存区 / 指定 rev 的 diff | 否 |
| `git_log` | 最近提交列表 | 否 |
| `git_stage` | 暂存或取消暂存（`all` 或指定路径） | 否 |
| `git_commit` | 创建提交（支持 `all`、`amend`） | 否 |
| `git_fetch` | 抓取远端并报告 ahead/behind | **是** |
| `git_pull` | 抓取并集成（支持 `rebase`） | **是** |
| `git_push` | 推送到远端（支持 `set-upstream`） | **是** |

所有工具都接受可选 `cwd`，默认使用当前会话的工作目录。

## 提供的斜杠命令（人类使用）

在输入框直接输入即可，**不经过模型**，在宿主进程中执行 git：

| 命令 | 选项 | 是否联网 |
|---|---|---|
| `/git` | — | 否 |
| `/git-status` | `cwd=<dir>` | 否 |
| `/git-diff` | `--staged` `rev=<rev>` `cwd=<dir>` | 否 |
| `/git-log` | `n` `cwd=<dir>` | 否 |
| `/git-commit` | `--all` `--amend` `<message>` `cwd=<dir>` | 否 |
| `/git-fetch` | `--prune` `remote=<name>` `cwd=<dir>` | **是** |
| `/git-pull` | `--rebase` `remote=<name>` `branch=<name>` `cwd=<dir>` | **是** |
| `/git-push` | `--set-upstream` `remote=<name>` `branch=<name>` `cwd=<dir>` | **是** |

**每一个命令都支持 `cwd=<dir>`**，用于指定仓库所在目录（见下方「关于仓库定位」）。

常用形式与选项：

```
/git-status
/git-commit 修复登录跳转
/git-commit --all 批量更新文档
/git-push --set-upstream
/git-pull --rebase
/git-diff --staged
```

也支持 `key=value` 形式的选项，用于覆盖默认值：

```
/git-status cwd=D:\some\other\repo
/git-push remote=upstream branch=release
/git-diff rev=origin/main...HEAD
```

**关于仓库定位**：命令默认以「会话工作目录」为仓库根。如果该目录本身不是仓库
（例如工作区指向 `D:\Githubrep` 而仓库在其子目录），命令会返回一条明确提示，
此时用 `cwd=<路径>` 指向真正的仓库，或直接把工作区改到仓库根目录。

## 提供的 agent 工具

与斜杠命令并列，模型也可以自主调用同名能力（`git_status`、`git_commit` 等）。
两者的区别只是触发者：命令由**人**输入 `/` 触发，工具由**模型**按需调用。

## 设计说明

- **不做 force push**：`git_push` 只使用 git 默认的非强制推送，没有提供 force 参数。
- **输出与语言环境无关**：使用 `--porcelain`、`-z`、显式 `--pretty=format`，
  不依赖 `LC_ALL`。
- **凭据**：`GIT_TERMINAL_PROMPT=0` 保证缺少凭据时快速失败而不是挂起。
  Windows 上 `credential.helper=manager` 会从凭据管理器取票，无需交互。
- **超时**：本地操作 120 秒，联网操作 300 秒。
- **`--no-color` 只用于接受它的子命令**：实测（git 2.55）`git diff` 与 `git log`
  **接受** `--no-color`，而 `git commit`、`git fetch`、`git pull`、`git push`
  **拒绝**它并直接以 `error: unknown option 'no-color'` 失败。切勿给后者添加该参数。
- **沙箱边界不变**：agent 自己的 shell 仍受 `workspace-write` 限制，
  本插件没有放宽它，只是把 git 放到了宿主侧执行。

## 安装

用 DSH GUI 侧边栏的「插件」页，选择「添加插件」，填入本目录的绝对路径：

```
D:\Githubrep\dsh-git-tools
```

或由具备 `plugin_manager` 工具的会话执行 `install_bundle`。

## 已知限制

- 仅宿主插件，暂无 Web UI 面板（面板是下一步）。
- `git_push` 依赖已保存在 Windows 凭据管理器中的凭据；没有缓存凭据时会失败并给出 git 的诊断信息。
- 冲突的 `git_pull` 会报错并把冲突留给用户解决，不会自动处理。
