# dsh-git-tools

给 DeepSeek Harness 的 agent 用的 Git 工具，git 在**宿主进程**中执行。

## 版本与测试环境

| 项目 | 版本 |
|---|---|
| **DSH 运行时** | **`0.1.7-rc.2`** |
| Node.js | `v24.9.0`（DSH 随包自带） |
| Git | `2.55.0.windows.3` |
| 操作系统 | Windows 11（build 10.0.26200） |

**说明：**

- 上面的 DSH 版本是**本插件实际测试通过的版本**。`0.1.7-rc.2` 是 `@deepseek-ai/dsh`、
  `@deepseek-ai/dsh-tools`、`@deepseek-ai/dsh-base` 三者一致的版本号。
- 声明在 `package.json` 的 `peerDependencies` 是 `@deepseek-ai/cordis@^4.0.1` 与
  `@deepseek-ai/dsh-tools@^0.1.7-rc.2`；这两个包由 DSH 运行时注入，**不需要 pnpm 安装**，
  所以安装时若出现 `missing peer` 警告可以忽略。
- 插件依赖两个宿主服务：`ctx.subprocess`（执行 git）与 `ctx.commands`（注册斜杠命令）。
  两者都由 DSH 随包提供，在 `0.1.7-rc.2` 上均已验证可用。
- **换用其他 DSH 版本时请自行验证。** 尤其是斜杠命令的注册契约
  （`ctx.commands.register({ definitionId, name, description, handler })`）与
  命令名规则（`/^[a-z][a-z0-9_-]*$/`）属于宿主内部约定，跨版本可能变化。

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
| `/git-name-set` | `name=<name>` `email=<email>` `remote=<url>` `cwd=<dir>` | 否 |
| `/git-tag` | `cwd=<dir>` | 否 |
| `/git-show` | `<version>` `cwd=<dir>` | 否 |
| `/git-tag-create` | `<version>` `message=<text>` `rev=<rev>` `cwd=<dir>` | 否 |
| `/git-tag-push` | `[version]` `remote=<name>` `cwd=<dir>` | **是** |

**每一个命令都支持 `cwd=<dir>`**，用于指定仓库所在目录（见下方「关于仓库定位」）。

### 参数类型约定

| 记号 | 含义 | 例子 |
|---|---|---|
| `<x>` | 必填值 | `<message>` |
| `[x]` | 可选 | `[cwd=<dir>]` |
| `--flag` | 开关，写了就生效，**不带值** | `--staged` |
| `key=<v>` | 键值对，**必须带 `=`** | `cwd=D:\repo` |

⚠️ **值参数漏掉 `=` 会失效**：`cwd=D:\repo` ✅；`cwd D:\repo` ❌（被当成两个无关的词）。

## 两个关键参数：`cwd` 与 `remote`

这两个是**寻址参数**——它们不提供新能力，只负责把命令指向正确的目标。

### `cwd=<dir>`：指定操作哪个仓库

**所有 9 个命令都支持**，这是最常用的参数。

| 项目 | 说明 |
|---|---|
| 指定什么 | git 在**哪个目录**执行，也就是操作哪个仓库 |
| 默认值 | 省略时使用**当前会话的工作目录** |
| 写法 | `cwd=D:\path\to\repo` 或 `cwd=D:/path/to/repo`（两种斜杠都行） |
| 必须是仓库 | 目录不是 git 仓库时会明确报错，**不会静默失败** |
| 路径含空格 | 目前**不支持**（解析按空格分词），请避免 |

**什么情况下必须写 `cwd=`：**

会话工作目录本身不是 git 仓库时。典型情形是工作区指向一个"容器目录"，
真正的仓库在它的子目录里：

```
工作区根      D:\Githubrep                              ← 不是仓库
真正的仓库    D:\Githubrep\skills-introduction-to-github  ← 仓库在这里
```

此时每个命令都要带 `cwd=`：

```
/git-status cwd=D:\Githubrep\skills-introduction-to-github
/git-commit cwd=D:\Githubrep\skills-introduction-to-github "改了什么"
/git-push   cwd=D:\Githubrep\skills-introduction-to-github
```

**一劳永逸的替代方案**：把 DSH 工作区直接指向仓库根目录。
之后所有命令零参数，也不再需要 `cwd=`。

### `remote=<...>`：指定远端——**注意语义有两种**

`remote=` 在两类命令里含义**完全不同**，这是最容易搞混的地方。
（下表只列出**涉及远端**的命令，不是命令全集；全集见上面的命令表。）

| 命令 | `remote=` 填什么 | 例子 | 效果 |
|---|---|---|---|
| `/git-push`<br>`/git-pull`<br>`/git-fetch` | **远端名** | `remote=origin` | 对哪个已配置的远端操作 |
| `/git-name-set` | **URL** | `remote=https://github.com/you/repo.git` | 把这个仓库的 origin **改指向**新地址 |

**为什么容易错**：`remote=upstream` 在 push/pull/fetch 里是合法的（"名叫 upstream 的远端"），
但在 `/git-name-set` 里会被**拒绝**，因为它期待一个 URL。

```
/git-push remote=upstream                    ✅ 远端名，推到名为 upstream 的远端
/git-name-set name=X email=x@y.com remote=upstream          ❌ 被拒绝：这需要 URL
/git-name-set name=X email=x@y.com remote=https://...git    ✅ URL，改写 origin
```

**基本信息：**

| 项目 | 说明 |
|---|---|
| 适用命令 | 只有 `/git-push`、`/git-pull`、`/git-fetch`（其余命令不涉及远端） |
| 默认值 | `origin`——git 克隆仓库时自动创建的默认远端 |
| 什么时候需要改 | 见下方「多个远端的场景」 |

**多个远端的场景**（默认 `origin` 不够用时）：

```powershell
# 先在终端里添加一个远端（插件没有添加远端的命令）
git remote add gitee https://gitee.com/you/repo.git
git remote add upstream https://github.com/original/repo.git
```

之后就能用 `remote=` 区分：

```
/git-push remote=gitee          推到 Gitee 而不是 GitHub
/git-fetch remote=upstream      从原始仓库（你 fork 的来源）取更新
/git-pull remote=upstream branch=main   把原始仓库的 main 合并进来
```

**查看当前有哪些远端**：插件没有专门命令，用终端 `git remote -v`，
或让 agent 执行。

### `/git-name-set`：配置提交身份与仓库指向

一次性设置**提交身份**，并可选地把当前仓库指向另一个远端地址：

```
/git-name-set name=ZhangSan email=zhangsan@example.com
/git-name-set name=ZhangSan email=zhangsan@example.com remote=https://github.com/zhangsan/repo.git
```

| 参数 | 必填 | 写入位置 | 作用范围 |
|---|---|---|---|
| `name=<name>` | ✅ | `git config --global user.name` | **全机器所有仓库** |
| `email=<email>` | ✅ | `git config --global user.email` | **全机器所有仓库** |
| `remote=<url>` | — | 目标仓库的 `origin`（`remote set-url`） | **仅该仓库** |
| `cwd=<dir>` | — | — | 指定要改远端的仓库 |

**两点必须理解清楚：**

1. **身份是全局的，不是临时的。** `git config --global` 写入 `~/.gitconfig`，
   之后本机所有仓库的提交都用这个名字和邮箱。想只影响单个仓库，请手动用
   `git config --local`。
2. **`remote=` 必须填 URL，不能填远端名。** 填 `remote=upstream` 会被拒绝，
   因为这是"远端名"而非地址。它的作用是**改变这个仓库推送的目标地址**，
   而不是新建一个远端。

所有校验都在写入之前完成：**参数有误时不会有任何副作用**（不会写配置、不会改 remote）。

示例：

```
/git 列出全部命令与用法
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

## 版本管理（标签）

### 先理解：版本是什么

git 里**没有自动的版本号**。每 `git commit` 一次就产生一个提交，但提交只由哈希
（如 `9f6af9d`）标识，无法用 `v1.0.0` 这样称呼它。

**标签（tag）就是给某个提交起的名字**，这才是"版本"：

```
提交 b237987 ──▶ C1
提交 8985e21 ──▶ C2
提交 816bced ──▶ C3  ◀── 标签 v1.0.0 指向这里
```

关键性质：

- **标签指向一个提交**，所以任何版本都能被永久取回（只要该提交存在）
- **标签是本地对象**，`git tag` 只写进你的本地仓库
- **必须推送到远端**（`git-tag-push`），GitHub 才会在 **Tags** 和 **Releases** 页显示它
- **删掉标签不影响提交**；提交本身不会被标签"绑定"

### 四个命令

| 命令 | 作用 | 联网 |
|---|---|---|
| `/git-tag` | 列出所有版本（最新在前，含哈希、日期、主题） | 否 |
| `/git-show <version>` | 查看某个版本：标签信息 + 改动的文件 | 否 |
| `/git-tag-create <version>` | 给某个提交打标签（版本名 + 可选说明） | 否 |
| `/git-tag-push [version]` | 把一个（或全部）标签推到 GitHub | **是** |

### 完整工作流

```
# 1. 确认当前状态，想清楚给哪个提交打版本
/git-status cwd=D:\Githubrep\skills-introduction-to-github
/git-log 5 cwd=D:\Githubrep\skills-introduction-to-github

# 2. 打版本标签（附注标签，带说明）
/git-tag-create v1.0.0 message=首个可用版本 cwd=D:\Githubrep\skills-introduction-to-github

# 3. 本地确认
/git-tag cwd=D:\Githubrep\skills-introduction-to-github

# 4. 推到 GitHub —— 之后 Tags/Releases 页才会出现这个版本
/git-tag-push v1.0.0 cwd=D:\Githubrep\skills-introduction-to-github

# 5. 随时回顾某个版本改了什么
/git-show v1.0.0 cwd=D:\Githubrep\skills-introduction-to-github
```

### 参数细节

**`/git-tag-create`**

| 参数 | 必填 | 说明 |
|---|---|---|
| `<version>` | ✅ | 版本名，如 `v1.0.0`。位置参数 |
| `message=<text>` | — | 给出则创建**附注标签**（带说明与打标签者信息）；省略则创建**轻量标签** |
| `rev=<rev>` | — | 给指定提交打标签；省略则给当前 `HEAD` 打 |
| `cwd=<dir>` | — | 仓库目录 |

**版本名规则**（同时用内置正则与 `git check-ref-format` 双重校验）：

| 规则 | 例 |
|---|---|
| 不能含空格 | ❌ `v1 0` |
| 不能含 `~ ^ : ? * [ \ |` | ❌ `v1:0` |
| 不能含连续两个点 | ❌ `v1..0` |
| 不能以 `.` 或 `-` 开头 | ❌ `-v1` |
| 不能以 `.` 或 `.lock` 结尾 | ❌ `v1.0.0.lock` |
| 允许斜杠（可做分层命名） | ✅ `release/v1.0.0` |

**`/git-tag-push`**

| 参数 | 必填 | 说明 |
|---|---|---|
| `[version]` | — | **省略则推送全部本地标签**（`push --tags`） |
| `remote=<name>` | — | 远端名，默认 `origin` |
| `cwd=<dir>` | — | 仓库目录 |

### 与 GitHub Releases 的关系

推送标签后，GitHub 会**自动生成** Tags 页：

```
https://github.com/<用户>/<仓库>/tags
```

而 **Releases 页**需要额外一步（在标签基础上附加发布说明、二进制包）：

```
https://github.com/<用户>/<仓库>/releases
```

两种做法：

1. **在 GitHub 网页上创建 Release**（推荐）——打开 Tags 页，点标签右侧的
   "Create release"，填标题和说明即可
2. **等本插件后续支持** —— 创建 Releases 需要调用 GitHub API，目前**未实现**；
   本插件的标签命令只负责 git 侧的标签，不涉及 GitHub Releases API

### 已知限制

- **没有删除标签的命令**。要删请用终端 `git tag -d <版本>`（本地）或
  `git push origin --delete <版本>`（远端）
- **不能检出/切换到某个版本**。查看用 `/git-show`，真要切过去需要
  `git switch --detach <版本>`
- **不支持签名标签**（`git tag -s`）
- **一次 `/git-tag-push` 不带参数会推送全部标签**，注意目标仓库是否需要这么多版本

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
