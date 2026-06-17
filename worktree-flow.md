# Worktree 多树管理代码流程分析

本文档按代码路径追踪 lazygit 中 worktree 的**创建、切换、清理**三大流程，以及**界面层与 git 状态层的串联机制**。

---

## 0. 核心文件概览

| 层级 | 文件 | 职责 |
|------|------|------|
| 数据模型 | [worktree.go](pkg/commands/models/worktree.go) | `Worktree` 结构体定义 |
| Git 命令封装 | [worktree.go](pkg/commands/git_commands/worktree.go) | `New / Delete / Detach` 等 git 命令构造 |
| Git 状态加载 | [worktree_loader.go](pkg/commands/git_commands/worktree_loader.go) | 解析 `git worktree list --porcelain`，构建 Worktree 模型数组 |
| UI 控制器 | [worktrees_controller.go](pkg/gui/controllers/worktrees_controller.go) | Worktrees 面板的按键绑定、主视图渲染 |
| 选项控制器 | [worktree_options_controller.go](pkg/gui/controllers/worktree_options_controller.go) | 在其他面板（如分支面板）触发 worktree 选项菜单 |
| 业务辅助层 | [worktree_helper.go](pkg/gui/controllers/helpers/worktree_helper.go) | 创建/切换/删除/分离等核心业务流程编排 |
| Repo 切换层 | [repos_helper.go](pkg/gui/controllers/helpers/repos_helper.go) | `DispatchSwitchTo` 切库/切 worktree 的公共切换逻辑 |
| 刷新编排 | [refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go) | `loadWorktrees / refreshWorktrees` 数据重加载与视图刷新 |
| 上下文/数据绑定 | [worktrees_context.go](pkg/gui/context/worktrees_context.go) | 列表视图模型与 `Model().Worktrees` 的绑定 |
| 视图渲染 | [worktrees.go](pkg/gui/presentation/worktrees.go) | 将 Worktree 模型转为显示字符串 |
| GUI 入口 | [gui.go](pkg/gui/gui.go) | `onNewRepo` 切换仓库时重建整个 GUI 状态 |
| 分支联动 | [branches_controller.go](pkg/gui/controllers/branches_controller.go) | 分支检出时检查 worktree 占用、跨 worktree 快进 |

---

## 1. 数据模型（Worktree Struct）

定义在 [worktree.go:L4-L28](pkg/commands/models/worktree.go#L4-L28)

```
Worktree
├── IsMain        bool     // 是否为主工作树（对应 git worktree 的主入口）
├── IsCurrent     bool     // 是否为当前所在的工作树（当前 lazygit 实例工作目录）
├── Path          string   // 工作树目录路径（用户文件所在目录）
├── IsPathMissing bool     // 路径是否已不存在（工作树目录被手动删除）
├── GitDir        string   // 对应的 .git 目录路径（linked worktree 为 <repo>/.git/worktrees/<name>）
├── Branch        string   // 当前检出的分支名（rebase/bisect 时也会补全）
├── Head          string   // HEAD SHA（detached 状态时用于显示）
└── Name          string   // 基于 Path 去重生成的显示名（非 git 内部名）
```

**关键设计**：
- `IsCurrent` 的判断：`path == worktreePath`（当前进程工作目录）
- `Branch` 即使在 rebase/bisect（detached HEAD）也会通过读取文件补全
- `Name` 通过路径层级比对，取能区分各 worktree 的最短后缀

---

## 2. 创建流程（New Worktree）

### 2.1 入口

有两条入口路径：

**路径 A：Worktrees 面板按 `n`（New）**
[worktrees_controller.go:L120-L122](pkg/gui/controllers/worktrees_controller.go#L120-L122)
```
add() ──► WorktreeHelper.NewWorktree()
```

**路径 B：Branches/Commits 等面板按 worktree 快捷键**
[worktree_options_controller.go:L49-L51](pkg/gui/controllers/worktree_options_controller.go#L49-L51)
```
viewWorktreeOptions(ref) ──► WorktreeHelper.ViewWorktreeOptions(ctx, ref)
  └─► ViewBranchWorktreeOptions(branchName, canCheckoutBase)
      └─► 菜单选择 → NewWorktreeCheckout(branchName, ...)
```

### 2.2 新建交互流程（NewWorktree → NewWorktreeCheckout）

[worktree_helper.go:L57-L155](pkg/gui/controllers/helpers/worktree_helper.go#L57-L155)

```
NewWorktree()
│
├─► 取当前分支名作为默认值
│
└─► 弹出菜单：
    ├─ "Create worktree from <ref>"           → detached=false
    └─ "Create worktree from <ref> (detached)" → detached=true
        │
        ▼
    Prompt: 输入 base ref（默认当前分支，带 refs 自动补全）
        │
        ▼
    NewWorktreeCheckout(base, canCheckoutBase, detached, contextKey)
        │
        ├─► Prompt: 输入新 worktree 的 Path
        │
        ├─► 若 detached=false:
        │   ├─ 若 canCheckoutBase=true:
        │   │   Prompt: 分支名（留空=直接 checkout base 分支）
        │   └─ 若 canCheckoutBase=false:
        │       Prompt: 分支名（必须填写，因为 base 已被其他 worktree 占用）
        │
        └─► 执行创建 + 切换
```

### 2.3 执行 git 命令 + 自动切换

[worktree_helper.go:L103-L112](pkg/gui/controllers/helpers/worktree_helper.go#L103-L112)

```
WithWaitingStatus("Adding worktree")
│
├─► LogAction("Add worktree")
│
├─► WorktreeCommands.New(opts)
│   └─► git worktree add [--detach] [-b <branch>] <path> <base>
│       [worktree.go:L32-L43](pkg/commands/git_commands/worktree.go#L32-L43)
│
└─► reposHelper.DispatchSwitchTo(opts.Path, ...)
    └─► 创建完成后立即切换到新 worktree
```

---

## 3. 切换流程（Switch Worktree）

切换的核心是**工作目录变更 + 整个 GUI 状态重建**，与切换子模块/切最近仓库走的是同一条 `DispatchSwitchTo` 通道。

### 3.1 入口

**路径 A：Worktrees 面板按 Enter**
[worktrees_controller.go:L140-L142](pkg/gui/controllers/worktrees_controller.go#L140-L142)
```
enter(worktree) ──► WorktreeHelper.Switch(worktree, WORKTREES_CONTEXT_KEY)
```

**路径 B：分支面板检出被占用的分支**
[branches_controller.go:L443-L455](pkg/gui/controllers/branches_controller.go#L443-L455)
```
press(branch)
└─► 若分支已被其他 worktree 检出:
    promptToCheckoutWorktree(worktree)
      └─► WorktreeHelper.Switch(worktree, LOCAL_BRANCHES_CONTEXT_KEY)
```

### 3.2 Switch → DispatchSwitchTo

[worktree_helper.go:L157-L165](pkg/gui/controllers/helpers/worktree_helper.go#L157-L165)
```go
func Switch(worktree, contextKey) {
    if worktree.IsCurrent → 报错 AlreadyInWorktree
    LogAction("Switch to worktree")
    return reposHelper.DispatchSwitchTo(worktree.Path, ErrWorktreeMovedOrRemoved, contextKey)
}
```

### 3.3 DispatchSwitchTo：公共切换逻辑

[repos_helper.go:L148-L197](pkg/gui/controllers/helpers/repos_helper.go#L148-L197)

```
DispatchSwitchTo(path, errMsg, contextKey)
│
├─ WithWaitingStatus("Switching")
│
├─ 1. env.UnsetGitLocationEnvVars()         // 清除 GIT_DIR/GIT_WORK_TREE 等
│
├─ 2. os.Chdir(path)                        // 切换进程工作目录
│   └─► 若失败 IsNotExist → 返回 errMsg
│
├─ 3. commands.VerifyInGitRepo()            // 确认新路径在 git 仓库内
│   └─► 失败则 os.Chdir(originalPath) 回退
│
├─ 4. direnv.Load()                         // 加载 .envrc（如有的话）
│
├─ 5. RecordCurrentDirectory()              // 记录到最近仓库
│
├─ 6. 锁 RefreshingFilesMutex
│
└─ 7. onNewRepo(StartArgs{}, contextKey)    // ★ 重建整个 GUI 状态
```

### 3.4 onNewRepo：GUI 全量重建

[gui.go:L320-L434](pkg/gui/gui.go#L320-L434)

```
onNewRepo(startArgs, contextKey)
│
├─ 1. NewGitCommand(Common, ...)          // ★ 全新的 GitCommand 对象（基于新 cwd）
│                                            重新计算 repoPath / worktreePath / gitDir
├─ 2. ReloadUserConfigForRepo()            // 加载仓库级 lazygit.yml
├─ 3. onUserConfigLoaded()
│
├─ 4. resetState(startArgs)                // 清空 Commits/Branches/Files/Worktrees 等 Model 数据
│                                            重置 RepoState（rebase/bisect 状态等）
├─ 5. resetHelpersAndControllers()         // 重建所有 helper 和 controller（它们持有旧 Model 引用）
├─ 6. resetKeybindings()                   // 按新配置重建键绑定
│
├─ 7. 设置 FocusHandler / HyperlinkHandler / SearchResultHandler
│
├─ 8. 若 contextKey != NO_CONTEXT:
│       取对应上下文，光标归零
│
├─ 9. Context().Push(contextToPush)        // 将上下文推入栈顶
│
└─ 10. render()                            // 首次渲染 → 触发首个 Refresh 全量加载
                                                （包括 Worktrees 的重新加载）
```

**关键点**：切换 worktree = 切仓库级别的全量重置。Worktree、Branch、Files 等所有 Model 数据都会被丢弃并重新从 git 读取。

---

## 4. 清理/删除流程（Remove Worktree）

### 4.1 入口

Worktrees 面板按 `d`（Remove）：
[worktrees_controller.go:L124-L134](pkg/gui/controllers/worktrees_controller.go#L124-L134)
```
remove(worktree)
├─ worktree.IsMain    → 报错 CantDeleteMainWorktree
├─ worktree.IsCurrent → 报错 CantDeleteCurrentWorktree
└─► WorktreeHelper.Remove(worktree, force=false)
```

### 4.2 删除流程

[worktree_helper.go:L167-L207](pkg/gui/controllers/helpers/worktree_helper.go#L167-L207)

```
Remove(worktree, force)
│
├─► 确认对话框（force 决定标题和提示文案）
│
└─► HandleConfirm:
    │
    └─ WithWaitingStatus("Removing worktree")
        │
        ├─ LogAction("Remove worktree")
        │
        ├─ WorktreeCommands.Delete(worktree.Path, force)
        │   └─► git worktree remove [-f] <path>
        │       [worktree.go:L45-L49](pkg/commands/git_commands/worktree.go#L45-L49)
        │
        ├─ 若报错且信息含 "--force" 或 submodule 相关:
        │   └─ force=false 时 → 递归调用 Remove(worktree, force=true) 重试
        │
        └─► Refresh(ASYNC, [WORKTREES, BRANCHES, FILES])
            └─► 不切库，只刷新当前三个面板
```

**补充：Detach 流程**（将 worktree 从分支上 detach，转为 detached HEAD）
[worktree_helper.go:L209-L220](pkg/gui/controllers/helpers/worktree_helper.go#L209-L220)
```
Detach(worktree)
└─► git checkout --detach --git-dir=<worktree.Path>/.git
    └─► Refresh(ASYNC, [WORKTREES, BRANCHES, FILES])
```

---

## 5. 数据加载与刷新串联机制（UI ↔ Git State）

### 5.1 全链路数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│                          UI 渲染层                                   │
│  ┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐  │
│  │ Worktrees    │     │ BranchesContext  │     │ Status Panel    │  │
│  │ Context      │     │ (显示 worktree   │     │ (显示当前       │  │
│  │ (侧边列表)   │     │  占用标记)       │     │  linked wt 名) │  │
│  └──────┬───────┘     └────────┬─────────┘     └────────┬────────┘  │
│         │ Model().Worktrees     │ Model().Worktrees     │           │
│         ▼                      ▼                       ▼           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     Model (内存状态)                          │  │
│  │  Worktrees: []*Worktree    Branches: []*Branch   Files: ...  │  │
│  └───────────────────────────────┬──────────────────────────────┘  │
│                                  │ Refresh(WORKTREES)              │
│                                  ▼                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  RefreshHelper                                │  │
│  │  refreshWorktrees()                                           │  │
│  │    ├─ loadWorktrees() → 写 Model().Worktrees                  │  │
│  │    ├─ 刷新 Branches 视图（worktree 会影响分支占用显示）        │  │
│  │    └─ 刷新 Worktrees 视图                                     │  │
│  └───────────────────────────────┬──────────────────────────────┘  │
│                                  │                                 │
└──────────────────────────────────┼─────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Git 命令层                                     │
│  WorktreeLoader.GetWorktrees()                                      │
│    ├─ git worktree list --porcelain                                 │
│    ├─ 逐行解析 → Worktree 切片                                       │
│    ├─ 并行: git rev-parse --absolute-git-dir（每个 wt）             │
│    ├─ 基于 Path 生成唯一 Name                                        │
│    ├─ 当前 worktree 置顶                                             │
│    └─ 补全 rebase/bisect 中的分支名                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 WorktreeLoader.GetWorktrees 详解

[worktree_loader.go:L24-L144](pkg/commands/git_commands/worktree_loader.go#L24-L144)

```
GetWorktrees()
│
├─► cmd: git worktree list --porcelain
│   输出格式（每个 worktree 多行，空行分隔）：
│     worktree /path/to/wt
│     HEAD abc123...
│     branch refs/heads/feature-x
│
├─► 逐行解析:
│   ├─ "worktree <path>"  → 创建 Worktree{IsMain/IsCurrent/IsPathMissing/Path}
│   ├─ "HEAD <sha>"      → worktree.Head = sha
│   ├─ "branch <ref>"    → worktree.Branch = 去掉 refs/heads/
│   └─ 空行 → 将当前 worktree 入数组，重置临时指针
│
├─► 并行 WaitGroup:
│   对每个路径存在的 worktree 调用:
│     git -C <path> rev-parse --absolute-git-dir
│     → 结果写入 worktree.GitDir
│
├─► getUniqueNamesFromPaths(paths)
│   递归按路径深度分组，取能区分各 worktree 的最短后缀
│   例: /a/b/wt1 和 /a/c/wt1 → 名称为 "b/wt1" 和 "c/wt1"
│
├─► 当前 worktree 置顶（交换到 [0]）
│
└─► 补全分支名（rebase/bisect 中 git list 不显示分支）:
    ├─ rebasedBranch(): 读 <GitDir>/rebase-merge/head-name 或 <GitDir>/rebase-apply/head-name
    └─ bisectedBranch(): 读 <GitDir>/BISECT_START
```

### 5.3 刷新触发链路（RefreshHelper）

[refresh_helper.go:L63-L237](pkg/gui/controllers/helpers/refresh_helper.go#L63-L237)

```
Refresh(options)
│
├─► 若无 Scope，默认包含 WORKTREES 在内的多数面板
│
├─► Scope 转为 Set: scopeSet
│
├─► ASYNC 模式: 每个 refresh() 包到 OnWorker（独立 goroutine）
│   SYNC/BLOCK_UI 模式: WaitGroup + go Safe() 并发
│
├─► 若 Scope 含 COMMITS || BRANCHES || REFLOG || BISECT_INFO:
│   │
│   ├─ includeWorktreesWithBranches = Scope 是否含 WORKTREES
│   │
│   └─► refreshBranches(includeWorktreesWithBranches, ...)
│       │
│       └─► refreshBranches 内部: 若 includeWorktreesWithBranches=true
│           ├─ loadWorktrees()              // ★ 连带加载 worktrees
│           └─ refreshView(Worktrees)       //    因为分支排序/显示需要 worktree 信息
│
└─► 否则，若 Scope 单独含 WORKTREES:
    └─► refresh("worktrees", refreshWorktrees)
        └─► loadWorktrees()
            ├─ refreshView(Branches)    // 分支面板显示占用标记
            └─ refreshView(Worktrees)   // worktrees 列表面板
```

**`loadWorktrees()`**：[refresh_helper.go:L722-L730](pkg/gui/controllers/helpers/refresh_helper.go#L722-L730)
```go
func loadWorktrees() {
    worktrees, err := Git().Loaders.Worktrees.GetWorktrees()
    if err { Model().Worktrees = [] } else { Model().Worktrees = worktrees }
}
```

### 5.4 Context 数据绑定

[worktrees_context.go:L9-L48](pkg/gui/context/worktrees_context.go#L9-L48)

```go
NewWorktreesContext():
    viewModel := NewFilteredListViewModel(
        func() []*Worktree { return c.Model().Worktrees },  // ★ 数据源：动态从 Model 取
        func(wt) []string { return []string{wt.Name} },      // 过滤匹配字段
    )
    getDisplayStrings := GetWorktreeDisplayStrings(tr, filteredList)
    //  → 由 presentation 层渲染成 [当前标记, 图标, 名称, 分支+主树标记]
```

Worktrees 面板的**渲染不持有数据副本**，每次 `HandleRender()` 都直接从 `Model().Worktrees` 取最新值，配合过滤/搜索。

### 5.5 Status 状态栏显示

[refresh_helper.go:L749-L767](pkg/gui/controllers/helpers/refresh_helper.go#L749-L767)

```
refreshStatus()
│
├─ currentBranch = refsHelper.GetCheckedOutRef()
├─ linkedWorktreeName = worktreeHelper.GetLinkedWorktreeName()
│   // 当前在 linked worktree → 返回名称；在主树 → 空串
│
└─► FormatStatus(repoName, currentBranch, op, linkedWorktreeName, state, ...)
```

**Status 中 linked worktree 的显示格式**（不是单独的 "wt: <name>" 标记，而是拼接到仓库名后）：

[status.go:L37-L46](pkg/gui/presentation/status.go#L37-L46)

```go
name := GetBranchTextStyle(currentBranch.Name).Sprint(currentBranch.Name)
if linkedWorktreeName != "" {
    icon := ""
    if icons.IsIconEnabled() {
        icon = icons.LINKED_WORKTREE_ICON + " "
    }
    repoName = fmt.Sprintf("%s(%s%s)", repoName, icon, style.FgCyan.Sprint(linkedWorktreeName))
}
status += fmt.Sprintf("%s → %s", repoName, name)
```

最终 Status 格式示例：
- 主树：`my-repo → main`
- Linked worktree（图标开启）：`my-repo(󰌹 feature-x) → feature-x`
- 图标：`LINKED_WORKTREE_ICON`，worktree 名用青色（FgCyan）显示

图标来源：与 worktree 列表面板的 worktree 图标完全相同——两者都调用 `IconForWorktree(false)` → `LINKED_WORKTREE_ICON`，定义见 [git_icons.go:L17-L18](pkg/gui/presentation/icons/git_icons.go#L17-L18)
- Nerd Font v3 模式：`\U000f0339` → `󰌹`
- 兼容 Nerd Font v2 模式：`\uf838` → ``

### 5.6 文件状态中 linked worktree 的识别与渲染

文件面板中，linked worktree 目录会被识别为特殊文件类型（区别于普通目录），并显示专用图标。整条链路分为 **识别层** 和 **渲染层** 两部分。

#### 5.6.1 识别层：File.IsWorktree 的标记流程

```
git status --porcelain
      │
      ▼
FileLoader.GetStatusFiles()
      │
      ├─ 第一步：解析 git status 输出 → []*File（此时 IsWorktree 全为 false）
      │
      └─ 第二步：标记 worktree 文件 [file_loader.go:L85-L103](pkg/commands/git_commands/file_loader.go#L85-L103)
          │
          ├─ linkedWortkreePaths(Fs, repoGitDirPath)
          │   │   // 注意：函数名有拼写错误（Wortkree=Worktree）
          │   │
          │   └─ [repo_paths.go:L142-L172](pkg/commands/git_commands/repo_paths.go#L142-L172)
          │       ├─ 遍历 <repo>/.git/worktrees/ 下的子目录
          │       ├─ 每个子目录中读 gitdir 文件 → 得到 worktree 的 .git 目录绝对路径
          │       └─ filepath.Dir(gitdir) → worktree 工作目录绝对路径列表
          │
          └─ 双重循环匹配：
              对每个 file，对每个 worktreePath：
                if filepath.Abs(file.Path) == worktreePath:
                    file.IsWorktree = true
                    file.Path = TrimSuffix(file.Path, "/")  // 去掉尾斜杠，避免被当普通目录渲染
                    break
```

**关键设计**：
- `IsWorktree` 标记的是**在当前仓库视角下**作为子目录存在的 linked worktree（即其他 worktree 路径嵌套在当前仓库工作区内的情况）
- 去掉尾斜杠是为了不让文件树把它渲染成普通文件夹（带展开箭头和 null file）

#### 5.6.2 渲染层：图标与样式

文件树渲染时，通过 `isLinkedWorktree` 标志选择专用图标：

[files.go:L159-L167](pkg/gui/presentation/files.go#L159-L167)

```go
isSubmodule := file != nil && file.IsSubmodule(submoduleConfigs)
isLinkedWorktree := file != nil && file.IsWorktree
isDirectory := file == nil

if showFileIcons {
    icon := icons.IconForFile(name, isSubmodule, isLinkedWorktree, isDirectory, customIconsConfig)
    paint := color.HEX(icon.Color, false)
    output += paint.Sprint(icon.Icon) + nameColor.Sprint(" ")
}
```

**IconForFile 优先级**（先匹配先返回）：
[file_icons.go:L769-L794](pkg/gui/presentation/icons/file_icons.go#L769-L794)

```
customIcons.Filenames → nameIconMap → customIcons.Extensions → extIconMap
    → isSubmodule → isLinkedWorktree → isDirectory → DEFAULT_FILE_ICON
```

Linked worktree 的图标配置：
- 与 Status 栏、Worktrees 面板共用同一个 `LINKED_WORKTREE_ICON`（三者均调用 `IconForWorktree(false)`）
- Nerd Font v3 模式：`\U000f0339` → `󰌹`；兼容 Nerd Font v2 模式：`\uf838` → ``
- 颜色：`#4E4E4E`（深灰）
- 定义在 [git_icons.go:L17-L18](pkg/gui/presentation/icons/git_icons.go#L17-L18)

**注意**：commit file tree（提交文件视图）中 `isLinkedWorktree` 恒为 false，因为提交记录里没有 worktree 概念。

---

## 6. 分支面板的 Worktree 联动

### 6.1 检出分支时的 worktree 占用检查

[branches_controller.go:L443-L485](pkg/gui/controllers/branches_controller.go#L443-L485)

```
press(selectedBranch)
│
├─► 已当前检出 → AlreadyCheckedOutBranch
│
├─► worktreeForBranch(branch)   // 遍历 Model().Worktrees，匹配 Branch 字段
│   │
│   └─ 找到 && !IsCurrent:
│       └─ promptToCheckoutWorktree(worktree)
│           ├─ 提示 "该分支已被 worktree <name> 检出，是否切换？"
│           └─ 确认 → WorktreeHelper.Switch(worktree, LOCAL_BRANCHES_CONTEXT_KEY)
│
└─► 否则: Refs.CheckoutRef(branch)   // 在当前 worktree 正常 checkout
```

### 6.2 跨 Worktree 快进

[branches_controller.go:L691-L739](pkg/gui/controllers/branches_controller.go#L691-L739)

```
fastForward(branch)
│
└─► worktreeForBranch(branch):
    ├─ 找到 worktree:
    │   ├─ 若非当前 → 传 WorktreeGitDir / WorktreePath 给 Pull
    │   └─ 若当前 → 不传（使用默认上下文）
    │   └─► Sync.Pull(FastForwardOnly=true, WorktreeGitDir=..., WorktreePath=...)
    │       // 通过 --git-dir / -C 让 pull 操作在指定 worktree 的上下文中执行
    │
    └─ 未找到:
        └─► Sync.FastForward(branch, upstream)
```

---

## 7. 关键设计小结

| 设计点 | 实现方式 | 所在位置 |
|--------|---------|---------|
| **切换等价于切库** | 所有数据（Model/Helpers/Controllers）全量重建，不做增量更新 | [gui.go:L320-L434](pkg/gui/gui.go#L320-L434) |
| **Worktree ↔ 分支 关联** | Worktree.Branch 字段，rebase/bisect 期间通过读文件补全 | [worktree_loader.go:L115-L141](pkg/commands/git_commands/worktree_loader.go#L115-L141) |
| **分支占用检查** | `WorktreeForBranch()` 在 `Model().Worktrees` 中线性查找 | [worktree.go:L57-L65](pkg/commands/git_commands/worktree.go#L57-L65) |
| **跨 worktree 执行 git** | 传 `--git-dir=<wt.GitDir>` 或 `-C <wt.Path>` 切换 git 上下文 | [worktree.go:L51-L55](pkg/commands/git_commands/worktree.go#L51-L55) （Detach 方法示例） |
| **刷新依赖** | refreshBranches 可选联动 loadWorktrees；refreshWorktrees 必联动 refreshView(Branches) | [refresh_helper.go:L486-L543](pkg/gui/controllers/helpers/refresh_helper.go#L486-L543) |
| **Context 切换上下文保留** | DispatchSwitchTo 接收 contextKey，onNewRepo 末尾按 key 定位并光标归零 | [gui.go:L418-L427](pkg/gui/gui.go#L418-L427) |
| **GitDir 并行加载** | 每个 worktree 的 rev-parse 调用独立 goroutine，WaitGroup 聚合 | [worktree_loader.go:L78-L96](pkg/commands/git_commands/worktree_loader.go#L78-L96) |
| **主树/当前树识别** | Path 与 `repoPaths.RepoPath()` / `WorktreePath()` 比较 | [worktree_loader.go:L55-L69](pkg/commands/git_commands/worktree_loader.go#L55-L69) |
