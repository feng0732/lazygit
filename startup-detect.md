# Lazygit 启动与首次仓库检测流程分析

## 整体流程概览

```
main()
  ↓
app.Start() ── 解析 CLI 参数、环境变量、构建信息
  ↓
app.Run() ── 创建 App 实例，验证 Git 版本，检测仓库
  ↓
NewApp()
  ├─ validateGitVersion()  ── Git 版本校验
  ├─ GetRepoPaths()        ── 获取仓库路径（可能失败）
  ├─ setupRepo()           ── 仓库设置（非仓库/裸仓库/最近仓库）
  └─ gui.NewGui()          ── 创建 GUI 实例
  ↓
app.Run(startArgs)
  ↓
gui.RunAndHandleError()
  ↓
gui.Run()
  ├─ initGocui()           ── 初始化 gocui 终端 UI 库
  ├─ createAllViews()      ── 创建所有视图
  ├─ onNewRepo()           ── 新仓库初始化（状态、控制器、快捷键）
  └─ MainLoop()            ── 进入事件主循环
```

---

## 1. 程序入口：main.go

入口文件为 [main.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/main.go)，非常简洁：

- 接收构建时注入的 `commit`、`date`、`version`、`buildSource` 变量（通过 LDFLAGS）
- 构造 `BuildInfo` 结构体
- 调用 `app.Start(ldFlagsBuildInfo, nil)` 启动应用

关键代码位于 [main.go#L15-L23](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/main.go#L15-L23)。

---

## 2. 启动阶段：app.Start()

位于 [entry_point.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/entry_point.go) 的 `Start` 函数是启动的核心入口。

### 2.1 CLI 参数解析

`parseCliArgsAndEnvVars()` 解析命令行参数和环境变量，主要参数包括：

| 参数 | 缩写 | 说明 |
|------|------|------|
| `--path` | `-p` | Git 仓库路径 |
| `--filter` | `-f` | 过滤路径（git log -- <path>） |
| `--work-tree` | `-w` | 工作树路径（等效 GIT_WORK_TREE） |
| `--git-dir` | `-g` | Git 目录路径（等效 GIT_DIR） |
| `--version` | `-v` | 打印版本信息 |
| `--debug` | `-d` | 调试模式 |
| `--logs` | `-l` | 跟踪日志 |
| `--config` | `-c` | 打印默认配置 |
| `--print-config-dir` | `-cd` | 打印配置目录 |
| `--use-config-dir` | `-ucd` | 指定配置目录 |
| `--use-config-file` | `-ucf` | 指定自定义配置文件 |
| `--screen-mode` | `-sm` | 初始屏幕模式（normal/half/full） |
| `--profile` | - | 启动性能分析（pprof） |

位置参数 `git-arg` 用于指定启动时聚焦的面板（status/branch/log/stash）。

### 2.2 构建信息合并

`mergeBuildInfo()` 函数将 LDFLAGS 注入的版本信息与 Go 内置的 `debug.ReadBuildInfo()` 合并，优先级：LDFLAGS > Go 构建信息。

如果没有通过 LDFLAGS 设置版本，则使用 git commit hash 的短格式作为版本号。

### 2.3 仓库路径预处理

在进入主逻辑前，会根据 CLI 参数对仓库路径进行预处理：

#### 场景 A：指定了 --path 参数

```go
if cliArgs.RepoPath != "" {
    // 1. 检查与 --work-tree / --git-dir 的互斥性
    // 2. 转换为绝对路径
    // 3. 验证是否为有效 Git 仓库（检查 .git 目录是否存在）
    // 4. 设置 GitDir 为 <path>/.git
    // 5. 切换工作目录到该路径
}
```

见 [entry_point.go#L57-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/entry_point.go#L57-L76)。

**错误分支**：
- 如果 `--path` 与 `--work-tree` / `--git-dir` 同时使用 → `log.Fatal`
- 如果路径不是有效 Git 仓库 → `log.Fatal`
- 如果切换目录失败 → `log.Fatal`

#### 场景 B：指定了 --work-tree 参数

```go
} else if cliArgs.WorkTree != "" {
    env.SetWorkTreeEnv(cliArgs.WorkTree)
    os.Chdir(cliArgs.WorkTree)
}
```

设置 `GIT_WORK_TREE` 环境变量并切换工作目录。

### 2.4 环境变量设置

根据参数设置以下环境变量：
- `LG_CONFIG_FILE`：自定义配置文件
- `CONFIG_DIR`：配置目录
- `GIT_DIR`：Git 目录

### 2.5 早期退出分支

在继续启动前，会检查一些立即退出的选项：

| 选项 | 行为 |
|------|------|
| `--version` | 打印版本信息后 `os.Exit(0)` |
| `--config` | 打印默认配置后 `os.Exit(0)` |
| `--print-config-dir` | 打印配置目录后 `os.Exit(0)` |
| `--logs` | 启动日志跟踪后 `os.Exit(0)` |

### 2.6 临时目录与配置初始化

```go
tempDir, err := os.MkdirTemp(tempDirBase, "lazygit-*")
// ...
appConfig, err := config.NewAppConfig(...)
```

- 创建临时目录（用户隔离：`/tmp/lazygit-<uid>/`）
- 初始化应用配置（AppConfig）

### 2.7 守护进程模式

如果运行在 daemon 模式（通过环境变量检测），则调用 `daemon.Handle()` 直接返回，不启动 GUI。

### 2.8 性能分析

如果 `--profile` 参数被设置，在后台 goroutine 中启动 pprof HTTP 服务器（localhost:6060）。

最后调用 `Run(appConfig, common, startArgs)` 进入应用主逻辑。

---

## 3. 应用初始化：app.Run() 与 NewApp()

位于 [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/app.go)。

### 3.1 NewApp() 流程

`NewApp()` 是应用实例的构造函数，执行以下步骤：

#### 步骤 1：创建 OSCommand

```go
app.OSCommand = oscommands.NewOSCommand(common, config, platform, guiIO)
```

#### 步骤 2：创建更新器（Updater）

```go
updater, err := updates.NewUpdater(common, config, app.OSCommand)
```

#### 步骤 3：获取当前工作目录

```go
dirName, err := os.Getwd()
```

#### 步骤 4：验证 Git 版本

```go
gitVersion, err := app.validateGitVersion()
```

详见 [3.2 Git 版本验证](#32-git-版本验证)。

#### 步骤 5：获取仓库路径

```go
repoPaths, err := git_commands.GetRepoPaths(app.OSCommand.Cmd, gitVersion)
```

详见 [3.3 仓库路径检测](#33-仓库路径检测)。

**注意**：这里即使获取失败也不会立即终止，只是记录日志，后续会在 `setupRepo()` 中处理。

#### 步骤 6：设置仓库 setupRepo()

```go
showRecentRepos, err := app.setupRepo(repoPaths)
```

详见 [3.4 仓库设置 setupRepo()](#34-仓库设置-setuprepo)。

#### 步骤 7：创建 GUI 实例

```go
app.Gui, err = gui.NewGui(common, config, gitVersion, updater, showRecentRepos, dirName, test)
```

### 3.2 Git 版本验证

`validateGitVersion()` 位于 [app.go#L150-L164](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/app.go#L150-L164)。

最低要求版本：**2.32.0**（`minGitVersionStr` 常量）。

流程：
1. 调用 `git --version` 获取版本字符串
2. 用正则表达式解析出版本号（Major.Minor.Patch）
3. 比较版本，低于最低版本则返回错误

**错误分支**：
- 获取版本失败 → 返回最低版本错误
- 版本太低 → 返回最低版本错误
- 两者都返回相同的错误消息 `minGitVersionErrorMessage`

### 3.3 仓库路径检测

`GetRepoPaths()` 位于 [repo_paths.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/commands/git_commands/repo_paths.go)。

核心实现是调用 `git rev-parse` 命令获取多个路径信息：

```bash
git rev-parse --path-format=absolute \
  --show-toplevel \          # 工作树根目录
  --absolute-git-dir \       # Git 目录绝对路径
  --git-common-dir \         # Git 公共目录
  --is-bare-repository \     # 是否为裸仓库
  --show-superproject-working-tree  # 超级项目工作树（子模块场景）
```

返回的 `RepoPaths` 结构体包含：
- `worktreePath`：当前工作树路径
- `worktreeGitDirPath`：工作树的 .git 目录路径
- `repoPath`：仓库路径（子模块场景等于 worktreePath）
- `repoGitDirPath`：仓库的 Git 目录路径
- `repoName`：仓库名称（目录基名）
- `isBareRepo`：是否为裸仓库

**错误分支**：如果当前目录不在 Git 仓库中，`git rev-parse` 会失败，返回 `nil, err`。

### 3.4 仓库设置 setupRepo()

`setupRepo()` 位于 [app.go#L194-L282](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/app.go#L194-L282)，是仓库检测的核心逻辑。

#### 前置判断：GIT_DIR 环境变量

如果已设置 `GIT_DIR` 环境变量，直接跳过所有 setup，返回 `(false, nil)`。

#### 场景 1：不在 Git 仓库中（repoPaths == nil）

```go
if repoPaths == nil {
    // 1. 获取当前工作目录
    // 2. 再次确认不是 Git 仓库（isDirectoryAGitRepository 检查 .git 目录）
    // 3. 根据 NotARepository 配置处理
}
```

根据 `userConfig.NotARepository` 配置有不同行为：

| 配置值 | 行为 |
|--------|------|
| `"prompt"` | 提示用户是否初始化新仓库（y/n），可选设置初始分支名 |
| `"create"` | 直接创建 Git 仓库 |
| `"skip"` | 不创建，尝试打开最近仓库 |
| `"quit"` | 打印错误后 `os.Exit(1)` |
| 其他 | 打印配置错误后 `os.Exit(1)` |

如果用户选择创建仓库（`shouldInitRepo = true`）：
```go
args := []string{"git", "init"}
// 可选添加 --initial-branch=...
app.OSCommand.Cmd.New(args).Run()
```

**如果不创建仓库**，则调用 `openRecentRepo(app)` 尝试打开最近使用的仓库。

#### openRecentRepo() 逻辑

`openRecentRepo()` 位于 [app.go#L171-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/app.go#L171-L192)：

1. 遍历 `app.Config.GetAppState().RecentRepos` 列表（最近使用的仓库）
2. 检查每个目录是否为有效 Git 仓库（`.git` 目录存在）
3. 尝试 `os.Chdir()` 切换到该目录
4. 加载 direnv（如果有）
5. 返回 `true` 表示成功打开

如果所有最近仓库都无法打开，打印 "No recent repositories" 并 `os.Exit(1)`。

**返回值**：`(showRecentRepos bool, err error)`
- 当成功从最近仓库列表打开时，`showRecentRepos = true`（后续 GUI 会显示最近仓库菜单）
- 正常进入当前目录仓库时，`showRecentRepos = false`

#### 场景 2：是裸仓库（repoPaths.IsBareRepo()）

```go
if repoPaths.IsBareRepo() {
    // 提示用户是否要打开最近仓库
    fmt.Print(app.Tr.BareRepo)
    response, _ := bufio.NewReader(os.Stdin).ReadString('\n')
    
    if shouldOpenRecent := strings.Trim(response, " \r\n") == "y"; !shouldOpenRecent {
        os.Exit(0)  // 用户选 n，直接退出
    }
    
    // 尝试打开最近仓库
    if openRecentRepo(app) {
        return true, nil
    }
    
    // 没有可用的最近仓库
    fmt.Println(app.Tr.NoRecentRepositories)
    os.Exit(1)
}
```

**错误分支**：
- 用户输入非 "y" → `os.Exit(0)`
- 没有可用的最近仓库 → 打印消息后 `os.Exit(1)`

---

## 4. 错误处理：knownError 机制

`Run()` 函数中的错误处理位于 [app.go#L48-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/app.go#L48-L62)：

```go
if err != nil {
    if errorMessage, known := knownError(common.Tr, err); known {
        log.Fatal(errorMessage)  // 已知错误：友好提示
    }
    // 未知错误：打印堆栈跟踪
    newErr := errors.Wrap(err, 0)
    stackTrace := newErr.ErrorStack()
    app.Log.Error(stackTrace)
    log.Fatalf("%s: %s\n\n%s", common.Tr.ErrorOccurred, constants.Links.Issues, stackTrace)
}
```

### 已知错误映射

`knownError()` 位于 [errors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/errors.go)，识别以下已知错误：

| 原始错误包含 | 友好提示 |
|-------------|----------|
| 最低 Git 版本错误（完全匹配） | 同消息（多语言翻译） |
| `"fatal: not a git repository"` | `NotARepository`（不是 Git 仓库） |
| `"getwd: no such file or directory"` | `WorkingDirectoryDoesNotExist`（工作目录不存在） |

已知错误直接 `log.Fatal` 打印友好消息；未知错误则打印完整堆栈跟踪和 issue 链接。

---

## 5. GUI 启动流程

### 5.1 NewGui() 构造

`gui.NewGui()` 位于 [gui.go#L721-L797](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/gui.go#L721-L797)，主要做以下初始化：

- 设置 `showRecentRepos` 标志（来自 setupRepo）
- 初始化 `RepoStateMap`（仓库状态映射，支持多仓库/工作树切换）
- 创建 `PopupHandler`（弹窗处理器）
- 创建 `OSCommand`（带 GUI IO）
- 创建 `BackgroundRoutineMgr`（后台例程管理器）
- 创建 `PagerConfig`

### 5.2 Run() 执行

`gui.Run()` 位于 [gui.go#L882-L952](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/gui.go#L882-L952)：

```go
func (gui *Gui) Run(startArgs appTypes.StartArgs) error {
    // 1. 初始化 gocui
    g, err := gui.initGocui(Headless(), startArgs.IntegrationTest)
    
    // 2. 设置错误处理器
    g.ErrorHandler = gui.PopupHandler.ErrorHandler
    
    // 3. 设置布局管理器
    gui.g.SetManager(gocui.ManagerFunc(gui.layout))
    
    // 4. 创建所有视图
    if err := gui.createAllViews(); err != nil {
        return err
    }
    
    // 5. 新仓库初始化（必须在 SetManager 之后，因为 SetManager 会删除快捷键）
    if err := gui.onNewRepo(startArgs, context.NO_CONTEXT); err != nil {
        return err
    }
    
    // 6. 启动后台例程
    gui.BackgroundRoutineMgr.startBackgroundRoutines()
    
    // 7. 安装恢复信号处理器
    gui.Helpers().SuspendResume.InstallResumeSignalHandler()
    
    // 8. 进入主事件循环
    err = gui.g.MainLoop()
}
```

### 5.3 onNewRepo() - 新仓库初始化

`onNewRepo()` 位于 [gui.go#L320-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/gui.go#L320-L434)，每次切换到新仓库时调用：

1. **创建 GitCommand**：`commands.NewGitCommand()`
2. **重新加载仓库配置**：`Config.ReloadUserConfigForRepo()`
3. **用户配置加载后处理**：`onUserConfigLoaded()`
   - 加载翻译（语言设置）
   - 设置颜色主题
   - 配置视图属性
   - 设置搜索快捷键
   - 设置编辑快捷键
   - 配置图标（Nerd Fonts）
   - 配置分支颜色
4. **重置状态**：`resetState(startArgs)`
   - 从 `RepoStateMap` 复用已有状态，或创建新状态
   - 初始化 `Model`（提交、文件、分支等数据模型）
   - 初始化 `Modes`（过滤、樱桃拣选、diff 等模式）
   - 设置初始屏幕模式
   - 创建上下文树和上下文管理器
   - 设置初始上下文（根据 git-arg 参数决定聚焦哪个面板）
5. **重置控制器和快捷键**
6. **设置焦点处理器**：窗口获得焦点时刷新配置和数据
7. **设置超链接处理器**：支持 `lazygit-edit://` 协议
8. **设置搜索结果处理器**
9. **推入初始上下文**：`gui.c.Context().Push(contextToPush, ...)`
10. **首次渲染**：`gui.render()`

### 5.4 初始上下文选择

`initialContext()` 位于 [gui.go#L692-L713](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/gui.go#L692-L713)：

- 默认聚焦 `Files` 上下文
- 如果有 `--filter` 参数，聚焦 `LocalCommits`
- 根据 `git-arg` 参数决定：
  - `status` → Files
  - `branch` → Branches
  - `log` → LocalCommits
  - `stash` → Stash

---

## 6. 首次布局与界面进入

### 6.1 布局函数 layout()

首次布局时会触发 `onInitialViewsCreation()` 回调，位于 [layout.go#L255-L280](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/layout.go#L255-L280)：

```go
func (gui *Gui) onInitialViewsCreation() error {
    // 1. 启动弹窗（首次使用介绍 / 破坏性变更说明）
    if !gui.c.UserConfig().DisableStartupPopups {
        storedPopupVersion := gui.c.GetAppState().StartupPopupVersion
        if storedPopupVersion < StartupPopupVersion {
            gui.showIntroPopupMessage()  // 新手引导
        } else {
            gui.showBreakingChangesMessage()  // 版本更新说明
        }
    }
    
    // 2. 保存当前版本号
    gui.c.GetAppState().LastVersion = gui.Config.GetVersion()
    gui.c.SaveAppStateAndLogError()
    
    // 3. 显示最近仓库菜单（如果是从最近仓库打开的）
    if gui.showRecentRepos {
        gui.helpers.Repos.CreateRecentReposMenu()
        gui.showRecentRepos = false
    }
    
    // 4. 后台检查更新
    gui.helpers.Update.CheckForUpdateInBackground()
    
    // 5. 标记 intro 等待完成
    gui.waitForIntro.Done()
}
```

### 6.2 最近仓库菜单

当 `showRecentRepos = true` 时（即从最近仓库列表打开时），会显示最近仓库菜单，让用户可以快速切换到其他最近使用的仓库。

菜单创建由 `gui.helpers.Repos.CreateRecentReposMenu()` 完成。

---

## 7. 启动阶段：StartupStage

为了优化启动速度，Lazygit 采用两阶段启动策略，定义在 [common.go#L402-L408](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/types/common.go#L402-L408)：

```go
type StartupStage int

const (
    INITIAL StartupStage = iota  // 初始阶段：快速显示基础数据
    COMPLETE                     // 完成阶段：所有数据加载完毕
)
```

### 7.1 两阶段刷新策略

`refreshReflogCommitsConsideringStartup()` 位于 [refresh_helper.go#L283-L296](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L283-L296)：

**INITIAL 阶段**：
- 同步加载核心数据（文件、分支、提交等）
- 异步（OnWorker）加载 reflog 提交
- reflog 加载完成后刷新分支排序
- 将阶段设置为 COMPLETE

**COMPLETE 阶段**：
- 正常同步加载所有数据
- 加载 behind/ahead 计数等次要数据

这样设计的目的是：reflog 数据是启动时的性能瓶颈（用于分支按最近使用排序），但不是首屏必须的，所以延后加载。

---

## 8. 错误分支总结

### 8.1 启动前期错误（CLI 阶段）

这些错误通过 `log.Fatal` 直接终止程序，不会进入 GUI：

| 错误场景 | 来源 |
|---------|------|
| --path 与 --work-tree/--git-dir 互斥 | entry_point.go |
| 指定路径不是有效 Git 仓库 | entry_point.go |
| 切换目录失败 | entry_point.go |
| 无效 git-arg 参数 | entry_point.go |
| 临时目录创建失败（权限等） | entry_point.go |
| 配置初始化失败 | entry_point.go |

### 8.2 Git 相关错误

| 错误场景 | 处理方式 |
|---------|----------|
| Git 版本过低（< 2.32.0） | 已知错误，友好提示后退出 |
| 不是 Git 仓库 | 进入 setupRepo 逻辑（提示创建/打开最近仓库/退出） |
| 裸仓库 | 提示是否打开最近仓库，或退出 |
| 工作目录不存在 | 已知错误，友好提示后退出 |

### 8.3 GUI 启动错误

| 错误场景 | 处理方式 |
|---------|----------|
| gocui 初始化失败 | 返回错误，由 Run 函数处理 |
| 视图创建失败 | 返回错误，由 Run 函数处理 |
| onNewRepo 失败 | 返回错误，由 Run 函数处理 |
| 主循环异常 | 返回错误，记录堆栈后退出 |

### 8.4 错误处理流程

```
发生错误
  ↓
gui.RunAndHandleError()
  ↓
app.Run() 中的错误处理
  ├─ knownError? ──是──→ log.Fatal(友好消息)
  └─ 否 ──→ 记录堆栈 + log.Fatal(错误详情 + issue 链接)
```

---

## 9. 界面进入完整时序

1. **命令行启动** → `main()` → `app.Start()`
2. **参数解析** → CLI args、环境变量
3. **仓库路径预处理** → --path / --work-tree / --git-dir
4. **配置初始化** → AppConfig、临时目录
5. **App 实例化** → `NewApp()`
   - Git 版本验证
   - 仓库路径检测（git rev-parse）
   - 仓库设置（非仓库/裸仓库/最近仓库）
   - GUI 实例创建
6. **GUI 运行** → `gui.Run()`
   - gocui 初始化
   - 视图创建
   - 新仓库初始化（状态、控制器、快捷键）
   - 首次上下文推入
   - 首次渲染
7. **首次布局** → `onInitialViewsCreation()`
   - 启动弹窗（新手引导/更新说明）
   - 最近仓库菜单（如果适用）
   - 后台更新检查
8. **主事件循环** → `MainLoop()`
   - 接收用户输入
   - 数据刷新（两阶段启动）
   - 界面渲染

---

## 关键文件索引

| 文件 | 作用 |
|------|------|
| [main.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/main.go) | 程序入口 |
| [pkg/app/entry_point.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/entry_point.go) | 启动入口、CLI 解析 |
| [pkg/app/app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/app.go) | App 构造、仓库设置、版本验证 |
| [pkg/app/errors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/app/errors.go) | 已知错误映射 |
| [pkg/commands/git_commands/repo_paths.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/commands/git_commands/repo_paths.go) | 仓库路径检测 |
| [pkg/commands/git_commands/version.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/commands/git_commands/version.go) | Git 版本解析 |
| [pkg/gui/gui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/gui.go) | GUI 构造与运行 |
| [pkg/gui/layout.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/layout.go) | 布局与首次视图创建 |
| [pkg/gui/recent_repos_panel.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/recent_repos_panel.go) | 最近仓库管理 |
| [pkg/gui/types/common.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/types/common.go) | 启动阶段定义 |
| [pkg/gui/controllers/helpers/refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-lazygit/pkg/gui/controllers/helpers/refresh_helper.go) | 刷新与两阶段启动 |
