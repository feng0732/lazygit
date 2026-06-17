# 分支列表上游追踪状态协作流程

本文档梳理 lazygit 中分支列表里上游（upstream）追踪状态从加载、标记到展示的完整协作链路。

## 一、线程模型概览

### 1.1 三种执行上下文

lazygit 的分支刷新涉及三种执行上下文：

| 执行上下文 | 创建者 | 典型职责 |
|----------|-------|--------|
| **MainLoop 线程** | gocui `MainLoop()` | 事件循环：消费 userEvents 队列，执行 View buffer 写入，flush 渲染屏幕 |
| **刷新 goroutine** | `refresh()` 内 `go Safe()` 或 `OnWorker()` 内 `go func()` | 调 git 命令、写 Model 数据、调度 UI 更新 |

**刷新 goroutine** 是统称，包含 `refresh()` 创建的裸 goroutine 和 `OnWorker` 创建的带 Task goroutine。两者的区别仅在于是否注册 Task（决定是否显示 loading），对 View 写入的线程规则完全相同。

### 1.2 View 写入的两条路径

所有 UI 更新最终都要写 View buffer（`view.SetContent` 或 `view.writeMutex` 保护的内部写方法），但有两条路径：

**路径 A：经 OnUIThread 投递到 MainLoop 执行**

```
刷新 goroutine
  │  refreshView(ctx)  或  OnUIThread(f)
  ▼
g.userEvents <- userEvent{f, task}
  │
  ▼
MainLoop 线程 processEvent() 消费
  │  执行 f → HandleRender() → view.SetContent()
  ▼
flush() → view.draw() → 渲染到屏幕
```

[refreshView](pkg/gui/controllers/helpers/refresh_helper.go#L785-L809) 内部注释明确说：

> refreshView is called from the worker goroutine that drives async refreshes, so bounce to the UI thread before mutating view content.

它调用 `OnUIThread` → `PostRefreshUpdate` → [HandleRender](pkg/gui/context/list_context_trait.go#L110-L128) → `view.SetContent`，整个链条在 MainLoop 上执行。

**路径 B：刷新 goroutine 中直接写 View buffer**

```
刷新 goroutine
  │  refreshStatus()
  ▼
RefreshingStatusMutex.Lock()
  │  SetViewContent(Status, status)
  │  → view.SetContent(cleanString(status))
RefreshingStatusMutex.Unlock()
```

[refreshStatus](pkg/gui/controllers/helpers/refresh_helper.go#L749-L767) **不走 OnUIThread**，直接在当前 goroutine 中调用 [SetViewContent](pkg/gui/view_helpers.go#L48-L50) → `view.SetContent`。

**线程安全性分析**：`view.SetContent` 内部持有 [view.writeMutex](pkg/gocui/view.go#L1049-L1055)，`view.draw`（MainLoop flush 时调用）也持有同一个 [writeMutex](pkg/gocui/view.go#L1215-L1217)。两者通过 per-View 的 `writeMutex` 互斥，所以路径 B 不会导致 buffer 并发损坏。`RefreshingStatusMutex` 只保护多次 `refreshStatus` 调用之间的互斥，真正的 buffer 安全由 `writeMutex` 保证。

**两条路径的实质区别**：不在于线程安全（`writeMutex` 已保证），而在于**执行时机**。路径 A 在 MainLoop 事件循环中执行，写完 buffer 后**同一个 MainLoop 周期**内就会 flush 到屏幕，读写紧凑。路径 B 写完 buffer 后，屏幕更新要等 MainLoop 下一次处理 userEvents 或定时 flush 时才生效，中间可能被其他 goroutine 再次写入同一 View，导致这次写入的内容"被覆盖"而非"被看到"。

### 1.3 refreshBranches 中的 View 写入点清单

[refreshBranches](pkg/gui/controllers/helpers/refresh_helper.go#L486-L543) 内部有 4 处 View 写入操作：

| # | 代码行 | 调用链 | 目标 View | 路径 | 实际执行线程 |
|---|-------|-------|----------|------|------------|
| ① | 第 500-506 行 `renderFunc` | `OnUIThread` → `HandleRender(Branches)` + `refreshStatus()` | Branches + Status | A | MainLoop |
| ② | 第 531 行 | `refreshView(Branches)` → `OnUIThread` → `PostRefreshUpdate` → `HandleRender` | Branches | A | MainLoop |
| ③ | 第 535-540 行 | `OnUIThread` → `HandleRender(Commits)` | Commits | A | MainLoop |
| ④ | 第 542 行 | `refreshStatus()` → `SetViewContent(Status)` → `view.SetContent` | Status | B | 刷新 goroutine |

**① 中的 `refreshStatus()`** 在 `OnUIThread` 回调内调用，所以实际在 MainLoop 上执行，走路径 A。

**④ 中的 `refreshStatus()`** 直接在刷新 goroutine 中调用，走路径 B。由于 ① 和 ④ 都写 Status View，且 ① 在 OnUIThread 中排队等待 MainLoop 消费，④ 在刷新 goroutine 中立即执行，**④ 的写入可能先于 ① 生效，但随后被 ① 的 MainLoop 执行覆盖**——最终屏幕上显示的是 ① 的内容。

### 1.4 三种 View 写入函数的线程规则总结

| 函数 | 是否经 OnUIThread | 写入路径 | 说明 |
|------|-----------------|---------|------|
| `refreshView(ctx)` | ✅ 是 | A | 包裹在 OnUIThread 中，View 写入在 MainLoop 上，写后即 flush |
| `OnUIThread(f)` 中调 `HandleRender` / `refreshStatus` | — （已在 MainLoop 上） | A | renderFunc 就是这种模式 |
| `refreshStatus()` | ❌ 否 | B | 直接写 Status View buffer，内容可能被后续路径 A 的写入覆盖 |

关键代码位置：
- `OnUIThread` 投递：[gui.go](pkg/gui/gui.go#L1185-L1189) → [gocui/gui.go](pkg/gocui/gui.go#L619-L642)
- MainLoop 消费 userEvents：[gocui/gui.go](pkg/gocui/gui.go#L756-L786)
- `refreshView` bounce 到 UI 线程：[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L785-L809)
- `refreshStatus` 直接写 buffer：[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L749-L767)
- `view.SetContent` 底层用 writeMutex 保护：[view.go](pkg/gocui/view.go#L1049-L1055)
- `view.draw` 读取时也用 writeMutex 保护：[view.go](pkg/gocui/view.go#L1215-L1217)

## 二、数据模型：状态的载体

### 2.1 Branch 结构体

[branch.go](pkg/commands/models/branch.go#L10-L43)

上游追踪状态相关字段：

| 字段 | 含义 | 来源 |
|------|------|------|
| `UpstreamRemote` | 追踪的远端名称，如 "origin" | git config `branch.<name>.remote` |
| `UpstreamBranch` | 追踪的远端分支名，如 "main" | git config `branch.<name>.merge` |
| `AheadForPull` | 相对 upstream 领先的提交数 | `%(upstream:track)` |
| `BehindForPull` | 相对 upstream 落后的提交数 | `%(upstream:track)` |
| `AheadForPush` | 相对 push 目标分支领先的提交数 | `%(push:track)` |
| `BehindForPush` | 相对 push 目标分支落后的提交数 | `%(push:track)` |
| `UpstreamGone` | 追踪的远端分支是否已被删除 | `%(upstream:track) == "[gone]"` |
| `BehindBaseBranch` | 相对基准分支的落后提交数 | `atomic.Int32`，异步计算 |

### 2.2 状态判断辅助方法

[branch.go](pkg/commands/models/branch.go#L92-L125)

| 方法 | 判定条件 | 含义 |
|------|---------|------|
| `IsTrackingRemote()` | `UpstreamRemote != ""` | 是否配置了上游追踪 |
| `RemoteBranchStoredLocally()` | 已追踪 且 `AheadForPull != "?" && BehindForPull != "?"` | 远端分支引用是否已本地缓存 |
| `RemoteBranchNotStoredLocally()` | 已追踪 且 `AheadForPull == "?" && BehindForPull == "?"` | 远端分支引用未在本地存储 |
| `MatchesUpstream()` | 已存储 且 ahead 和 behind 均为 "0" | 与上游完全同步 |
| `IsAheadForPull()` | 已存储 且 `AheadForPull != "0"` | 领先于上游 |
| `IsBehindForPull()` | 已存储 且 `BehindForPull != "0"` | 落后于上游 |
| `IsBehindForPush()` | 已存储 且 `BehindForPush != "0"` | 落后于 push 目标分支 |
| `IsRealBranch()` | `AheadForPull != "" && BehindForPull != ""` | 非 detached head 状态 |

**关键点**：当 AheadForPull/BehindForPull 为 `"?"` 时，表示本地没有对应远端分支引用，无法计算差异。当为 `""` 时表示 detached head 状态。

## 三、数据加载层：从 Git 获取原始数据

### 3.1 入口：BranchLoader.Load

[branch_loader.go](pkg/commands/git_commands/branch_loader.go#L67-L146)

加载流程：

1. `obtainBranches()` → 通过 `git for-each-ref` 获取所有本地分支及其追踪信息
2. 若按 recency 排序，通过 reflog 补充最近检出信息
3. 把 HEAD 分支移到列表首位
4. **从 git config 补充 UpstreamRemote 和 UpstreamBranch**
5. 从旧分支列表继承 BehindBaseBranch（减少闪烁）
6. 可选：**调度** BehindBaseBranch 异步计算（通过 onWorker）

### 3.2 获取原始分支数据：git for-each-ref

[branch_loader.go](pkg/commands/git_commands/branch_loader.go#L392-L428)

查询字段定义在 `branchFields`：

```go
var branchFields = []string{
    "HEAD",                // 是否当前分支（* 或空）
    "refname:short",       // 分支短名，如 heads/feature
    "upstream:short",      // 上游短名，如 origin/feature；空表示本地未存远端引用
    "upstream:track",      // 上游追踪状态，如 [ahead 3]、[behind 2, ahead 1]、[gone]
    "push:track",          // push 目标追踪状态（三角工作流）
    "subject",             // commit 标题
    "objectname",          // commit hash
    "committerdate:unix",  // 提交时间戳
}
```

Git 命令：
```bash
git for-each-ref --sort=<sortOrder> --format="%(HEAD)%00%(refname:short)%00%(upstream:short)%00%(upstream:track)%00%(push:track)%00%(subject)%00%(objectname)%00%(committerdate:unix)" refs/heads
```

### 3.3 解析追踪状态：parseUpstreamInfo

[branch_loader.go](pkg/commands/git_commands/branch_loader.go#L466-L491)

核心逻辑：

```
upstream:short 为空?
├─ 是 → ("?", "?", false)          // 本地无远端引用，无法计算差异
└─ 否
    ├─ track == "[gone]"?
    │   └─ 是 → ("?", "?", true)   // 远端分支已删除
    └─ 否 → 正则提取 ahead/behind 数字
            ├─ `ahead (\d+)` → AheadForPull
            └─ `behind (\d+)` → BehindForPull
            提取不到则默认 "0"
```

该函数在 [obtainBranch](pkg/commands/git_commands/branch_loader.go#L431-L464) 中被调用**两次**，分别处理不同的追踪来源：

```go
aheadForPull, behindForPull, gone := parseUpstreamInfo(upstreamName, track)       // %(upstream:track)
aheadForPush, behindForPush, _ := parseUpstreamInfo(upstreamName, pushTrack)      // %(push:track)
```

**注意**：两次调用都传入同一个 `upstreamName`（即 `%(upstream:short)` 的值），但传入不同的 `track` 字符串。对于 `pushTrack`，第三次返回值（gone 标记）被丢弃。

### 3.4 从 git config 补全上游名称

[config.go](pkg/commands/git_commands/config.go#L80-L114)

调用：
```bash
git config --local --get-regexp ^branch\.
```

从输出中解析每行：
- `branch.<name>.remote <value>` → BranchConfig.Remote
- `branch.<name>.merge <value>` → BranchConfig.Merge（去除 `refs/heads/` 前缀）

然后在 `BranchLoader.Load` 中回填到每个 Branch 的 `UpstreamRemote` 和 `UpstreamBranch`。

### 3.5 BehindBaseBranch 的异步调度与计算

[branch_loader.go](pkg/commands/git_commands/branch_loader.go#L139-L330)

在 `BranchLoader.Load` 的第 139-143 行（**在刷新 goroutine / OnWorker goroutine 中执行**）：

```go
if loadBehindCounts && self.UserConfig().Gui.ShowDivergenceFromBaseBranch != "none" {
    onWorker(func() error {
        return self.GetBehindBaseBranchValuesForAllBranches(branches, mainBranches, renderFunc)
    })
}
```

**关键机制（三层 goroutine 嵌套）**：

| 层级 | 执行上下文 | 代码位置 | 做什么 |
|------|----------|---------|-------|
| 第 1 层 | 刷新 goroutine | Load 第 139 行 | 判断条件，调用 `onWorker(f)` |
| 第 2 层 | 刷新 goroutine（OnWorker 创建的新 goroutine） | [gocui/gui.go](pkg/gocui/gui.go#L650-L656) | 执行 `GetBehindBaseBranchValuesForAllBranches()` → 调 git 命令算差异 |
| 第 3 层 | MainLoop 线程 | `renderFunc()` 内部 | `HandleRender()` 重绘分支列表（路径 A）+ `refreshStatus()` 重绘状态栏（也在 OnUIThread 回调内，路径 A） |

**`renderFunc()` 的完整定义**（传入 `BranchLoader.Load` 的第 6 个参数）在 [refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L500-L506)：

```go
func() {
    self.c.OnUIThread(func() error {       // ← 投递给 MainLoop
        self.c.Contexts().Branches.HandleRender()   // 路径 A：经 OnUIThread → MainLoop 写 Branches View
        self.refreshStatus()                       // 也在 OnUIThread 回调内 → 同样在 MainLoop 上，路径 A
        return nil
    })
}
```

这里 `refreshStatus()` 在 `OnUIThread` 回调**内部**调用，所以实际在 MainLoop 上执行，走路径 A。这和 `refreshBranches` 末尾直接调用 `refreshStatus()`（第 542 行，不走 OnUIThread，走路径 B）形成对比。

计算路径：
| 路径 | Git 版本要求 | 方式 |
|------|-------------|------|
| **Fast** | ≥ 2.41 | 一次 `git for-each-ref --format="%(ahead-behind:<base>)" refs/heads` |
| **Legacy** | < 2.41 | 对每个分支单独 `git merge-base` + `git rev-list --left-right --count` |

## 四、排序配置对加载流程的影响

### 4.1 三种排序模式

[user_config.go](pkg/config/user_config.go#L331-L334) 定义了 `git.localBranchSortOrder`：

| 配置值 | for-each-ref 排序参数 | 对 Recency 字段的影响 | 对刷新路径的影响 |
|--------|----------------------|----------------------|----------------|
| `date`（默认） | `--sort=-committerdate` | 直接用提交时间戳算 Recency | 直接调用 `refreshBranches` |
| `recency` | `--sort=-committerdate` | 从 reflog 中提取最近检出时间 | 走 `refreshReflogAndBranches` |
| `alphabetical` | `--sort=refname` | 直接用提交时间戳算 Recency | 直接调用 `refreshBranches` |

### 4.2 刷新调度入口：`refresh()` 闭包

[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L110-L128)

```go
refresh := func(name string, f func()) {
    if !self.c.InDemo() && options.Mode == types.ASYNC {
        // 走 OnWorker：显示 loader，注册 Task
        self.c.OnWorker(func(t gocui.Task) error {
            f()
            return nil
        })
    } else {
        // 走裸 goroutine：用 WaitGroup 等待，不注册 Task
        wg.Add(1)
        go utils.Safe(func() {
            t := time.Now()
            defer wg.Done()
            f()
            self.c.Log.Infof("refreshed %s in %s", name, time.Since(t))
        })
    }
}
```

**两种调度模式的差异**：

| 维度 | `ASYNC` 模式（走 OnWorker） | 其他模式（裸 goroutine + WaitGroup） |
|------|---------------------------|----------------------------------|
| 是否显示 loading | 是（NewTask 递增计数） | 否 |
| 等待机制 | 无（fire-and-forget） | `wg.Wait()` 等待所有 wg.Add 的任务完成 |
| 并发控制 | 与其他 Task 共享同一事件队列 | 完全独立并发 |

### 4.3 刷新路径差异（线程视角）

[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L139-L152)

**recency 模式**（单刷新 goroutine 串行执行）：

```
刷新 goroutine (由 refresh() 创建)
  └─ refreshReflogAndBranches()
       ├─ 第 299 行: 计算 loadBehindCounts
       ├─ 第 301 行: refreshReflogCommitsConsideringStartup()
       │    └─ 若 INITIAL: OnWorker 调度异步任务，立即返回
       └─ 第 303 行: refreshBranches(..., loadBehindCounts)
            └─ 串行执行 git for-each-ref、config、Model.Branches = branches
```

**date/alphabetical 模式**（两个刷新 goroutine 并发）：

```
刷新 goroutine A (branches)           刷新 goroutine B (reflog)
  └─ refreshBranches(..., true)         └─ refreshReflogCommits()
       └─ 并行执行，互不等待
```

### 4.4 启动阶段：recency 模式的完整线程时序

[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L283-L304)

为解决 recency 排序下 reflog 加载瓶颈，系统设计了 **INITIAL/COMPLETE 两阶段** + **两次 refreshBranches 调用**机制。

#### 4.4.1 startupStage 的初始值

[common.go](pkg/gui/types/common.go#L406-L407) 定义：
```go
INITIAL StartupStage = iota  // 默认值 0
COMPLETE                     // 值 1
```

`GuiRepoState` 初始化时 `StartupStage` 默认为 `INITIAL`（Go struct 零值机制）。

#### 4.4.2 recency 模式启动完整线程时序（初始状态 startupStage = INITIAL）

```
MainLoop 线程（用户触发 Refresh 或启动刷新）
  │
  │  调度 refresh("reflog and branches", refreshReflogAndBranches)
  ▼
┌─ 刷新 goroutine (裸 go Safe() / OnWorker goroutine)
│
│  第 299 行: loadBehindCounts := startupStage == COMPLETE
│             → 当前是 INITIAL → loadBehindCounts = false
│
│  第 301 行: refreshReflogCommitsConsideringStartup()
│    → 匹配 INITIAL case
│    └─ self.c.OnWorker(...)   ──────► 投递任务，立即返回
│                                     │
│                                     ▼
│                                   ┌─ OnWorker goroutine #1
│                                   │  步骤 4: refreshReflogCommits() → 读 .git/logs/HEAD
│                                   │  步骤 5: refreshBranches(false, true, true)
│                                   │          └─ BranchLoader.Load(...true...)
│                                   │              ├─ 同步：git for-each-ref + reflog 合并 + config
│                                   │              ├─ 写 Model.Branches = branches
│                                   │              ├─ refreshView(Branches) → OnUIThread 投递
│                                   │              └─ 第 139 行: loadBehindCounts=true
│                                   │                  └─ onWorker(computeBehindBaseBranch)
│                                   │                      └─ 另起 OnWorker goroutine #2 ──┐
│                                   │                                                         │
│                                   │  步骤 6: SetStartupStage(COMPLETE)                      │
│                                   │      (此时第二次 refreshBranches 已完成)                 │
│                                   └─────────────────────────────────────────────────        │
│                                                                                             │
│  第 303 行: refreshBranches(..., loadBehindCounts=false)  ◄── 步骤 3，立即执行              │
│      └─ 刷新 goroutine 内同步执行：                                                         │
│          步骤 3a: BranchLoader.Load(...false...)                                            │
│              ├─ git for-each-ref                                                            │
│              ├─ reflog 合并（reflogCommits 还没加载，分支按默认顺序）                         │
│              ├─ config.Branches(cmd)                                                        │
│              ├─ 继承旧 BehindBaseBranch 值                                                  │
│              ├─ 第 139 行: loadBehindCounts=false → 跳过调度计算                             │
│              └─ return branches                                                             │
│          步骤 3b: Model.Branches = branches                                                 │
│          步骤 3c: refreshView(Branches)                                                     │
│              └─ OnUIThread 投递 → 等待 MainLoop 消费                                        │
│          步骤 3d: refreshStatus()                                                           │
│              └─ SetViewContent(Status, status)  ⚠️ 路径 B：直接写 buffer，可能被路径 A 覆盖   │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                             │
┌─ OnWorker goroutine #2 (BehindBaseBranch 计算)  ◄──────────────────────────────────────────┘
│    步骤 N: git for-each-ref --format="%(ahead-behind:main)" 或 per-branch merge-base
│    → 写每个 branch.BehindBaseBranch.Store(N)
│    → renderFunc() → OnUIThread 投递 HandleRender + refreshStatus
│
└──────────────────────────────────────────────────────────────────────────────┐
                                                                              │
MainLoop 线程（持续消费 userEvents）  ◄────────────────────────────────────────┘
  ├─ 消费 refreshView 的 userEvent → PostRefreshUpdate → 写 View buffer → flush
  ├─ 消费 renderFunc 的 userEvent → HandleRender + refreshStatus → 写 View buffer → flush
  └─ ...
```

**按时间顺序的步骤表（带线程标签）**：

| 时间顺序 | 线程 | 步骤 | 代码位置 | 关键参数 | startupStage 状态 |
|---------|------|------|---------|---------|------------------|
| T0 | MainLoop | Refresh 调度，创建刷新 goroutine | [refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L110-L128) | — | INITIAL |
| T1 | 刷新 goroutine | 计算 `loadBehindCounts = false` | 第 299 行 | — | INITIAL |
| T2 | 刷新 goroutine | `refreshReflogCommitsConsideringStartup()` → 调度 OnWorker，**立即返回** | 第 301 行 → 第 285-291 行 | — | INITIAL |
| T3 | 刷新 goroutine | **第一次 refreshBranches(false)** → 同步返回分支列表（reflog 可能为空，分支顺序不准） | 第 303 行 → [第 486-543 行](pkg/gui/controllers/helpers/refresh_helper.go#L486-L543) | `loadBehindCounts=false` | INITIAL |
| T3a | 刷新 goroutine | 写 Model.Branches = branches | 第 513 行 | — | INITIAL |
| T3b | 刷新 goroutine | `refreshView(Branches)` → 投递 OnUIThread，**立即返回** | 第 531 行 → [第 785-809 行](pkg/gui/controllers/helpers/refresh_helper.go#L785-L809) | — | INITIAL |
| T3c | 刷新 goroutine | `refreshStatus()` → **路径 B：直接写 Status View buffer**（writeMutex 保护，无并发损坏风险，但内容可能被后续路径 A 的写入覆盖） | 第 542 行 → [第 749-767 行](pkg/gui/controllers/helpers/refresh_helper.go#L749-L767) | — | INITIAL |
| T4 | OnWorker goroutine #1 | `refreshReflogCommits()` → 实际加载 reflog | 第 287 行 | — | INITIAL |
| T5 | OnWorker goroutine #1 | **第二次 refreshBranches(true)** → reflog 已加载，分支按 recency 排序正确 | 第 288 行 | `loadBehindCounts=true` | INITIAL |
| T5a | OnWorker goroutine #1 | 调度 **OnWorker goroutine #2** 计算 BehindBaseBranch | `Load` 第 139-143 行 | — | INITIAL |
| T5b | OnWorker goroutine #1 | 写 Model.Branches + refreshView + refreshStatus | 第 513/531/542 行 | — | INITIAL |
| T6 | OnWorker goroutine #1 | `SetStartupStage(COMPLETE)` | 第 289 行 | — | → COMPLETE |
| T7 | OnWorker goroutine #2 | `GetBehindBaseBranchValuesForAllBranches()` 计算完成 | `branch_loader.go` 第 148 行起 | — | COMPLETE |
| T7a | OnWorker goroutine #2 | `renderFunc()` → 投递 OnUIThread (HandleRender + refreshStatus) | `refresh_helper.go` 第 500-506 行 | — | COMPLETE |
| Tn | MainLoop | 依次消费 T3b、T5b、T7a 的 userEvents → flush 屏幕 | `gocui/gui.go` 第 756-786 行 | — | COMPLETE |

#### 4.4.3 重要结论

1. **recency 模式启动时会调用两次 refreshBranches**
   - 第一次（刷新 goroutine 中，T3）：`loadBehindCounts=false`，reflog 可能未加载，分支排序不准，不调度 BehindBaseBranch
   - 第二次（OnWorker goroutine #1 中，T5）：`loadBehindCounts=true`，reflog 已加载，排序正确，调度 BehindBaseBranch 计算

2. **第二次 refreshBranches 发生在 SetStartupStage(COMPLETE) 之前**（T5 在 T6 之前），不是"进入 COMPLETE 后的后续刷新"

3. **三次 UI 重绘依次排队**（MainLoop 按投递顺序消费）：
   - 第一次重绘：T3b，分支可能排序不准，BehindBaseBranch 用继承值或 0
   - 第二次重绘：T5b，分支按 recency 正确排序，BehindBaseBranch 仍用继承值或 0
   - 第三次重绘：T7a，BehindBaseBranch 计算完成，显示正确的 ↓N

4. **refreshStatus 的两条写入路径**：
   - 路径 B（T3c/T5b）：刷新 goroutine 中直接调用 `SetViewContent()`，由 `view.writeMutex` 保护，无并发损坏风险
   - 路径 A（renderFunc 内）：`OnUIThread` 回调中调用，在 MainLoop 上执行
   - 两条路径都写 Status View，路径 B 的内容可能被后续路径 A 的 MainLoop 执行覆盖——最终屏幕上显示的是路径 A 的内容

#### 4.4.4 COMPLETE 阶段后的行为

startupStage = COMPLETE 后：
```
刷新 goroutine
  ├─ 第 299 行: loadBehindCounts := true
  ├─ 第 301 行: refreshReflogCommitsConsideringStartup() → COMPLETE case，同步加载 reflog
  └─ 第 303 行: refreshBranches(..., true)
       └─ BranchLoader.Load(...true...) → 调度 BehindBaseBranch 计算（又起一个 OnWorker goroutine）
```

此时是同步流程：先加载 reflog，再刷新分支（调度 BehindBaseBranch 异步计算）。

#### 4.4.5 date/alphabetical 模式的线程行为

**没有两阶段机制**。启动第一次刷新就硬编码 `loadBehindCounts=true`（[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L148)）：

```
MainLoop
  │
  ├─ refresh("branches", ...)  →  创建刷新 goroutine A
  └─ refresh("reflog", ...)    →  创建刷新 goroutine B（并发）

刷新 goroutine A
  └─ refreshBranches(..., true)
       ├─ BranchLoader.Load(...true...)
       ├─ refreshView → OnUIThread
       ├─ refreshStatus (直接写 buffer)
       └─ 第 139 行: loadBehindCounts=true
           └─ onWorker(...) → 创建 OnWorker goroutine #2 计算 BehindBaseBranch
```

BehindBaseBranch 计算在第一次刷新时就被调度。

#### 4.4.6 loadBehindCounts 参数传递全景（带线程信息）

| 排序模式 | 启动阶段 | 调用线程 | 代码位置 | loadBehindCounts 值 |
|---------|---------|---------|---------|--------------------|
| recency | 初始 INITIAL | 刷新 goroutine | 第 303 行 sync call | false（第 299 行计算） |
| recency | 初始 INITIAL | OnWorker goroutine #1 | 第 288 行 async call | true（硬编码） |
| recency | 后续 COMPLETE | 刷新 goroutine | 第 303 行 sync call | true（第 299 行计算） |
| date/alphabetical | 任何阶段 | 刷新 goroutine | 第 148 行 sync call | true（硬编码） |

## 五、推送计数对分支状态展示的影响

### 5.1 展示层不使用 Push 计数

[branches.go](pkg/gui/presentation/branches.go#L215-L245) 中的 `BranchStatus` 函数**仅使用 `*ForPull` 字段**：

```go
if branch.IsBehindForPull() && branch.IsAheadForPull() {
    result = style.FgYellow.Sprintf("↓%s↑%s", branch.BehindForPull, branch.AheadForPull)
} else if branch.IsBehindForPull() {
    result = style.FgYellow.Sprintf("↓%s", branch.BehindForPull)
} else if branch.IsAheadForPull() {
    result = style.FgYellow.Sprintf("↑%s", branch.AheadForPull)
}
```

`AheadForPush` 和 `BehindForPush` 在展示层**完全不出现**。分支列表中的 `↓N↑M` 仅反映与 upstream 分支（pull 方向）的差异。

### 5.2 Push 计数的实际使用场景

`IsBehindForPush()` 唯一的业务调用在 [sync_controller.go](pkg/gui/controllers/sync_controller.go#L89-L98)：

```go
func (self *SyncController) push(currentBranch *models.Branch) error {
    if currentBranch.IsTrackingRemote() {
        opts := pushOpts{remoteBranchStoredLocally: currentBranch.RemoteBranchStoredLocally()}
        if currentBranch.IsBehindForPush() {
            // push 目标分支有你没有的提交 → 需要确认是否 force push
            return self.requestToForcePush(currentBranch, opts)
        }
        return self.pushAux(currentBranch, opts)
    }
    ...
}
```

在**三角工作流**（triangular workflow）中：
- `branch.<name>.remote` = origin（pull 来源）
- `remote.pushDefault` 或 `branch.<name>.pushRemote` = fork（push 目标）
- `%(upstream:track)` 反映与 origin 的差异
- `%(push:track)` 反映与 fork 的差异

当 push 目标分支有本地没有的提交时，`IsBehindForPush()` 为 true，push 操作会触发 force push 确认对话框。

### 5.3 完整的状态来源与用途对照

| 字段 | 数据来源 | 展示用途 | 业务逻辑用途 |
|------|---------|---------|-------------|
| `AheadForPull` / `BehindForPull` | `%(upstream:track)` | BranchStatus 中的 `↑N` / `↓N` / `↓N↑M` | 判断是否可以 push/pull |
| `AheadForPush` / `BehindForPush` | `%(push:track)` | **不展示** | push 时判断是否需要 force push 确认 |
| `UpstreamGone` | `%(upstream:track) == "[gone]"` | BranchStatus 中的 `upstream gone`（红色） | 无额外用途 |
| `BehindBaseBranch` | 异步 `%(ahead-behind:<base>)` 或 `rev-list` | divergenceStr 中的 `↓N`（青色，右对齐） | 无额外用途 |

## 六、展示渲染层：如何呈现给用户

### 6.1 展示入口：BranchesContext

[branches_context.go](pkg/gui/context/branches_context.go#L19-L61)

`getDisplayStrings` 调用 `presentation.GetBranchListDisplayStrings`，把 `[]*models.Branch` 转成 `[][]string` 供列表渲染。

### 6.2 追踪状态图标：BranchStatus

[branches.go](pkg/gui/presentation/branches.go#L215-L245)

分支名称后面会附加状态标记，优先级如下：

| 优先级 | 状态 | 显示 | 颜色 | 判定条件 |
|--------|------|------|------|---------|
| 1（最高） | 正在操作中 | `Pushing |` 等 + spinner | 青色 | 有 `itemOperation` |
| 2 | 上游已删除 | `upstream gone` | 红色 | `UpstreamGone == true` |
| 3 | 与上游同步 | `✓` | 绿色 | `MatchesUpstream()` |
| 4 | 远端分支未本地存储 | `?` | 品红 | `RemoteBranchNotStoredLocally()` |
| 5 | 既领先又落后 | `↓N↑M` | 黄色 | `IsBehindForPull() && IsAheadForPull()` |
| 6 | 仅落后 | `↓N` | 黄色 | `IsBehindForPull()` |
| 7 | 仅领先 | `↑N` | 黄色 | `IsAheadForPull()` |
| — | 无追踪 | （空） | — | `!IsTrackingRemote()` |

### 6.3 基准分支分歧：divergenceStr

[branches.go](pkg/gui/presentation/branches.go#L247-L266)

在分支名**右侧**显示相对基准分支的落后数（由配置 `gui.showDivergenceFromBaseBranch` 控制）：

- `none`：不显示
- `onlyArrow`：显示 `↓`
- `arrowAndNumber`：显示 `↓N`

此指标与 BranchStatus 是**独立的两个维度**。

### 6.4 完整渲染：getBranchDisplayStrings

[branches.go](pkg/gui/presentation/branches.go#L47-L186)

每一行的列组成（按顺序）：

| 列 | 内容 | 说明 |
|----|------|------|
| 1 | `Recency` | 最近检出时间 / `  *`（HEAD 标记） |
| 2 | PR 图标 | 若开启 GitHub PR 集成 |
| 3 | Commit hash | `fullDescription` 或配置 `showBranchCommitHash` 开启时显示 |
| 4 | 分支名 + 状态 | 含 worktree 标记、BranchStatus、divergenceStr（右对齐） |
| 5 | 上游信息 | `fullDescription` 时显示：`UpstreamRemote UpstreamBranch` |
| 6 | Commit 标题 | `fullDescription` 时显示 |

## 七、完整线程调用链路图

```
MainLoop 线程 (键盘/启动触发 Refresh)
  │
  └─ refresh("reflog and branches", refreshReflogAndBranches)
     │
     ├─ ASYNC 模式: OnWorker(f)  ─────────────────────┐
     └─ 其他模式: go Safe(f) + wg.Add(1)  ───────────┐
                                                      │
┌─────────────────────────────────────────────────────┘
│  刷新 goroutine
│
│  ├─ recency 模式:
│  │   ├─ loadBehindCounts := startupStage == COMPLETE  (第 299 行)
│  │   ├─ refreshReflogCommitsConsideringStartup()
│  │   │   └─ INITIAL: OnWorker() 调度任务，立即返回
│  │   │       └─ 创建 OnWorker goroutine #1  ──────────┐
│  │   │                                                  │
│  │   └─ refreshBranches(..., loadBehindCounts)  ◄── 第 303 行，立即执行
│  │       ├─ BranchLoader.Load(loadBehindCounts=X)
│  │       │   ├─ git for-each-ref (sync, 当前 goroutine)
│  │       │   ├─ reflog 合并
│  │       │   ├─ git config (sync, 当前 goroutine)
│  │       │   ├─ 继承旧 BehindBaseBranch
│  │       │   └─ loadBehindCounts?
│  │       │       └─ 是 → onWorker(computeBehindBaseBranch)
│  │       │           └─ 创建 OnWorker goroutine #2  ───┐
│  │       │                                               │
│  │       ├─ Model.Branches = branches (写共享状态)
│  │       ├─ refreshView(Branches)
│  │       │   └─ OnUIThread → 投递 userEvents ──────────┐
│  │       │                                               │
│  │       └─ refreshStatus() (路径 B：直接写 Status View buffer，可能被路径 A 覆盖) │
│  │                                                       │
│  └─ date/alphabetical 模式:                              │
│      ├─ refreshBranches(..., true)                       │
│      └─ 与 refreshReflog 并发执行（另一个刷新 goroutine） │
│                                                          │
└──────────────────────────────────────────────────────────┘
                                                           │
┌─ OnWorker goroutine #1 (recency INITIAL 调度)  ◄─────────┘
│  ├─ refreshReflogCommits() → 读 .git/logs/HEAD
│  └─ refreshBranches(false, true, true)
│      └─ 同上结构：BranchLoader.Load(loadBehindCounts=true)
│          └─ 又调度一个 OnWorker goroutine (BehindBaseBranch)
│  └─ SetStartupStage(COMPLETE)
│
└──────────────────────────────────────────────────────────┐
                                                           │
┌─ OnWorker goroutine #2 (BehindBaseBranch 计算)  ◄────────┘
│  ├─ Fast: git for-each-ref %(ahead-behind:<base>)
│  └─ Legacy: per-branch merge-base + rev-list
│  → 写每个 branch.BehindBaseBranch.Store(N)
│  → renderFunc()
│      └─ OnUIThread → 投递 userEvents ───────────────────┐
│                                                          │
└──────────────────────────────────────────────────────────┘
                                                           │
MainLoop 线程 (持续消费 userEvents channel)  ◄─────────────┘
  │
  │  processEvent() / processRemainingEvents()
  │
  ├─ userEvent: refreshView → PostRefreshUpdate → 写 View buffer
  ├─ userEvent: renderFunc → HandleRender() + refreshStatus() → 写 View buffer
  ├─ ...
  │
  └─ flush() / flushContentOnly() → 实际渲染到终端屏幕
```

## 八、关键设计要点

1. **四线程协作模型**：MainLoop 线程负责事件循环和屏幕渲染；刷新 goroutine 负责 git 命令和模型更新；OnWorker goroutine 是注册为 Task 的后台任务；UI 线程回调是 MainLoop 消费 userEvents 时执行的闭包。

2. **三层 goroutine 嵌套（BehindBaseBranch）**：刷新 goroutine → OnWorker goroutine #1 计算差异 → 又通过 `OnUIThread` 投递到 MainLoop（路径 A）。renderFunc 中的 `HandleRender` 和 `refreshStatus` 都在 MainLoop 上执行。

3. **userEvents channel 串行化路径 A 的 UI 修改**：经 `OnUIThread()` → `g.userEvents <-` 投递的操作由 MainLoop 串行消费，写完 buffer 后同一周期内 flush。但路径 B（`refreshStatus` 直接写）不经过此队列，其内容可能被后续路径 A 覆盖。

4. **View buffer 的 writeMutex 保证并发安全**：`view.SetContent` 和 `view.draw` 都持有 per-View 的 `writeMutex`，即使路径 B 在非 MainLoop 线程写入，也不会与 flush 产生并发损坏。路径 A 和路径 B 的区别在于执行时机而非线程安全。

5. **recency 模式三次渐进重绘**：第一次重绘（分支排序可能不准）→ 第二次重绘（分支按 recency 排序正确）→ 第三次重绘（BehindBaseBranch 值正确）。三次依次排队到 userEvents channel，用户感知为渐进式更新。

6. **recency 模式两次 refreshBranches**：第一次在刷新 goroutine 中立即执行（`loadBehindCounts=false`），第二次在 OnWorker goroutine #1 中 reflog 加载完后执行（`loadBehindCounts=true`）。第二次执行时 startupStage 仍是 INITIAL。

7. **双调度模式差异**：ASYNC 模式走 OnWorker（显示 loading、注册 Task），其他模式走裸 goroutine + WaitGroup，并发度更高但不显示 loading。

8. **Push 计数隐藏设计**：`AheadForPush` / `BehindForPush` 在 UI 上不可见，仅在 push 操作时用于判断是否需要 force push 确认。这是三角工作流的安全保障。

9. **两个独立维度**：BranchStatus（`✓`/`↓N↑M` 等）反映与 **upstream** 的同步关系，divergenceStr（右对齐 `↓N`）反映与 **base branch** 的落后关系，分别来自不同的数据源和计算路径。

10. **线程安全的共享状态**：`BehindBaseBranch` 使用 `atomic.Int32` 存储，OnWorker goroutine 写、MainLoop 读，无需锁即可安全并发访问。`Model.Branches` 的读写通过 `RefreshingBranchesMutex` 保护。
