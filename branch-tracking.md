# 分支列表上游追踪状态协作流程

本文档梳理 lazygit 中分支列表里上游（upstream）追踪状态从加载、标记到展示的完整协作链路。

## 一、整体架构概览

整个流程分为三层，协作关系如下：

```
┌────────────────────┐     数据流向      ┌────────────────────┐    UI 刷新    ┌────────────────────┐
│  数据加载层         │ ──────────────▶  │  状态标记层         │ ──────────▶  │  展示渲染层         │
│  (branch_loader)   │                   │  (models.Branch)   │               │  (presentation)    │
└────────────────────┘                   └────────────────────┘               └────────────────────┘
         ▲                                                                          ▲
         │                                                                          │
         └─────────────── 刷新触发层 (refresh_helper) ───────────────────────────────┘
```

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
6. 可选：**调度**异步计算 BehindBaseBranch（相对基准分支落后数）

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

**注意**：两次调用都传入同一个 `upstreamName`（即 `%(upstream:short)` 的值），但传入不同的 `track` 字符串。这意味着：
- 当 `upstreamName` 为空时，两次调用都返回 `"?"`
- 当 `upstreamName` 非空但 `track` 为 `""`（即与上游同步，Git 不输出 track 信息时），两次调用都返回 `("0", "0", false)`

对于 `pushTrack`，第三次返回值（gone 标记）被丢弃，因为 push 分支不存在不算 "gone"。

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

**原因**：`git for-each-ref` 的 `%(upstream:short)` 只在本地缓存了远端引用时才有值，而 git config 始终保存追踪配置，两者互为补充。

### 3.5 BehindBaseBranch 的异步调度与计算

[branch_loader.go](pkg/commands/git_commands/branch_loader.go#L139-L330)

在 `BranchLoader.Load` 的第 139-143 行：

```go
if loadBehindCounts && self.UserConfig().Gui.ShowDivergenceFromBaseBranch != "none" {
    onWorker(func() error {
        return self.GetBehindBaseBranchValuesForAllBranches(branches, mainBranches, renderFunc)
    })
}
```

**关键机制**：
- `loadBehindCounts=true` 表示**调度**异步计算任务，不是同步计算
- 实际计算通过 `onWorker` 抛到后台 worker goroutine
- 同步返回的分支列表继承旧的 BehindBaseBranch 值（避免闪烁）
- 计算完成后通过 `renderFunc()` 切到 UI 线程触发 `HandleRender()` 和 `refreshStatus()`

根据 Git 版本有两条计算路径：

| 路径 | Git 版本要求 | 方式 |
|------|-------------|------|
| **Fast** | ≥ 2.41 | 一次 `git for-each-ref --format="%(ahead-behind:<base>)" refs/heads` 批量获取 |
| **Legacy** | < 2.41 | 对每个分支单独 `git merge-base` + `git rev-list --left-right --count` |

两条路径最终都调用 `renderFunc()` 在 UI 线程触发重绘，实现渐进式渲染。

## 四、排序配置对加载流程的影响

### 4.1 三种排序模式

[user_config.go](pkg/config/user_config.go#L331-L334) 定义了 `git.localBranchSortOrder`：

| 配置值 | for-each-ref 排序参数 | 对 Recency 字段的影响 | 对刷新路径的影响 |
|--------|----------------------|----------------------|----------------|
| `date`（默认） | `--sort=-committerdate` | 直接用提交时间戳算 Recency | 无需 reflog，直接调用 `refreshBranches` |
| `recency` | `--sort=-committerdate` | 从 reflog 中提取最近检出时间 | 需先加载 reflog，走 `refreshReflogAndBranches` |
| `alphabetical` | `--sort=refname` | 直接用提交时间戳算 Recency | 无需 reflog，直接调用 `refreshBranches` |

### 4.2 排序对数据加载步骤的差异

[branch_loader.go](pkg/commands/git_commands/branch_loader.go#L74-L119)

**`date` / `alphabetical` 模式**：
```
obtainBranches()
  → git for-each-ref (排序参数生效)
  → obtainBranch(): storeCommitDateAsRecency = true
  → 直接用 committerdate:unix 计算 Recency

跳过 reflog 合并步骤
```

**`recency` 模式**：
```
obtainBranches()
  → git for-each-ref --sort=-committerdate
  → obtainBranch(): storeCommitDateAsRecency = false
  → Recency 暂时为空

obtainReflogBranches(reflogCommits)
  → 遍历 reflog 中的 checkout 记录
  → 按 checkout 时间给分支赋予 Recency

合并：按 reflog 顺序重排分支（最近 checkout 的排前面）
  → branchesWithRecency 在前
  → 其余分支按字母序排在后面
  → 合并结果 = branchesWithRecency + 按字母序的其余分支
```

### 4.3 刷新路径差异

[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L131-L152)

```go
if self.c.UserConfig().Git.LocalBranchSortOrder == "recency" {
    // 需要先有 reflog 数据才能排序
    refresh("reflog and branches", func() {
        self.refreshReflogAndBranches(...)
    })
} else {
    // 不依赖 reflog，可以并行
    refresh("branches", func() {
        self.refreshBranches(..., true)  // loadBehindCounts = true，硬编码
    })
    refresh("reflog", func() { ... })    // reflog 独立加载
}
```

**影响**：`recency` 模式下，分支加载必须走 `refreshReflogAndBranches` 串行链路；`date` / `alphabetical` 模式下分支与 reflog 可以并行加载。

### 4.4 启动阶段：recency 模式的两次刷新机制

[refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L283-L304)

为解决 recency 排序下 reflog 加载瓶颈，系统设计了 **INITIAL/COMPLETE 两阶段** + **两次 refreshBranches 调用**机制。

#### 4.4.1 startupStage 的初始值

[common.go](pkg/gui/types/common.go#L406-L407) 定义：
```go
INITIAL StartupStage = iota  // 默认值 0
COMPLETE                     // 值 1
```

`GuiRepoState` 初始化时 `StartupStage` 默认为 `INITIAL`（Go struct 零值机制）。

#### 4.4.2 recency 模式启动时序（初始状态 startupStage = INITIAL）

**`refreshReflogAndBranches` 内部执行顺序**：

```
第 299 行: loadBehindCounts := startupStage == COMPLETE
           → 当前是 INITIAL → loadBehindCounts = false

第 301 行: 调用 refreshReflogCommitsConsideringStartup()
           → 匹配 INITIAL case
           → 把以下工作抛到 worker goroutine，函数立即返回:
               1. refreshReflogCommits()           // 加载 reflog
               2. refreshBranches(false, true, true)  // loadBehindCounts = true
               3. SetStartupStage(COMPLETE)

第 303 行: 调用 refreshBranches(..., loadBehindCounts=false)
           → 同步执行，BehindBaseBranch 不调度计算
```

**关键时序**（按执行顺序）：

| 步骤 | 执行位置 | 操作 | loadBehindCounts | startupStage 状态 |
|------|---------|------|-----------------|------------------|
| 1 | UI 线程 | `refreshReflogAndBranches` 计算 `loadBehindCounts=false` | — | INITIAL |
| 2 | UI 线程 | `refreshReflogCommitsConsideringStartup` 调度异步任务，**立即返回** | — | INITIAL |
| 3 | UI 线程 | 第一次 `refreshBranches(false)` 执行，**立即返回分支列表** | false | INITIAL |
| 4 | 后台 worker | `refreshReflogCommits()` 加载 reflog | — | INITIAL |
| 5 | 后台 worker | **第二次 `refreshBranches(true)` 执行**，调度 BehindBaseBranch 计算 | true | **还是 INITIAL** |
| 6 | 后台 worker | `SetStartupStage(COMPLETE)` | — | → COMPLETE |

**重要结论**：
1. **recency 模式启动时会调用两次 refreshBranches**
   - 第一次（同步，UI 线程）：`loadBehindCounts=false` → 快速展示分支列表（无 BehindBaseBranch 计算）
   - 第二次（异步，worker 中）：`loadBehindCounts=true` → reflog 加载完后再次刷新，调度 BehindBaseBranch 计算
2. 第二次 `refreshBranches(true)` 发生在 `SetStartupStage(COMPLETE)` **之前**，不是等进入 COMPLETE 阶段后的"后续刷新"
3. 两次调用间隔是 reflog 加载的耗时（通常几十到几百毫秒）

#### 4.4.3 COMPLETE 阶段后的行为

当 startupStage = COMPLETE 后，后续刷新 `refreshReflogAndBranches` 的行为：

```
第 299 行: loadBehindCounts := startupStage == COMPLETE → true

第 301 行: refreshReflogCommitsConsideringStartup()
           → 匹配 COMPLETE case
           → 同步执行 refreshReflogCommits()

第 303 行: refreshBranches(..., true)  → loadBehindCounts=true
```

此时是完整的同步流程：先同步加载 reflog，再同步刷新分支（调度 BehindBaseBranch 异步计算）。

#### 4.4.4 date/alphabetical 模式的行为

**没有两阶段机制**。启动第一次刷新就直接调用：

```go
self.refreshBranches(includeWorktreesWithBranches, options.KeepBranchSelectionIndex, true)
```

`loadBehindCounts` **硬编码为 true**，无需等待 reflog，BehindBaseBranch 计算在第一次刷新时就会被调度。

#### 4.4.5 loadBehindCounts 参数传递全景

| 排序模式 | 启动阶段 | 调用者 | loadBehindCounts 值 |
|---------|---------|-------|--------------------|
| recency | 初始 INITIAL | 第 303 行 sync call | false（第 299 行计算） |
| recency | 初始 INITIAL | 第 288 行 async call | true（硬编码） |
| recency | 后续 COMPLETE | 第 303 行 sync call | true（第 299 行计算） |
| date/alphabetical | 任何阶段 | 第 148 行 sync call | true（硬编码） |

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

**注意**：优先级 5-7 的数字来自 `BehindForPull` / `AheadForPull`，不是 `*ForPush`。

### 6.3 基准分支分歧：divergenceStr

[branches.go](pkg/gui/presentation/branches.go#L247-L266)

在分支名**右侧**显示相对基准分支的落后数（由配置 `gui.showDivergenceFromBaseBranch` 控制）：

- `none`：不显示
- `onlyArrow`：显示 `↓`
- `arrowAndNumber`：显示 `↓N`

此指标与 BranchStatus 是**独立的两个维度**：BranchStatus 反映与 upstream 的关系，divergenceStr 反映与 base branch（如 main/master）的关系。

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

## 七、完整调用链路图

```
触发刷新 (Refresh)
  │
  ├─ recency 模式 ────────────────────────────────────────────────────┐
  │   refreshReflogAndBranches()                                       │
  │     ├─ 第 299 行: loadBehindCounts := startupStage == COMPLETE     │
  │     ├─ 第 301 行: refreshReflogCommitsConsideringStartup()         │
  │     │   ├─ INITIAL: OnWorker 调度异步任务，立即返回                 │
  │     │   │   ├─ 后台: refreshReflogCommits()                        │
  │     │   │   ├─ 后台: refreshBranches(false, true, true)  ◀── 第二次刷新，true
  │     │   │   └─ 后台: SetStartupStage(COMPLETE)                     │
  │     │   └─ COMPLETE: 同步 refreshReflogCommits()                   │
  │     │                                                              │
  │     └─ 第 303 行: refreshBranches(..., loadBehindCounts)  ◀── 第一次刷新，false(INITIAL)/true(COMPLETE)
  │                                                                    │
  └─ date/alphabetical 模式 ───────────────────────────────────────────┤
      refreshBranches(..., true)  ← 硬编码 true，与 refreshReflog 并行 │
                                                                      │
  ┌────────────────────────────────────────────────────────────────────┘
  │
  ▼
refreshBranches()
  │
  ├─ BranchLoader.Load(reflogCommits, mainBranches, oldBranches, loadBehindCounts, onWorker, renderFunc)
  │    │
  │    ├─ obtainBranches()
  │    │    ├─ getRawBranches()
  │    │    │   └─ git for-each-ref --sort=<sortOrder> refs/heads
  │    │    │       ├─ %(upstream:short)  ── 本地是否有远端引用
  │    │    │       ├─ %(upstream:track)  ── ahead/behind/gone
  │    │    │       └─ %(push:track)      ── push 目标的 ahead/behind
  │    │    │
  │    │    └─ obtainBranch(split)
  │    │         ├─ parseUpstreamInfo(upstreamName, track)
  │    │         │   → AheadForPull, BehindForPull, UpstreamGone
  │    │         └─ parseUpstreamInfo(upstreamName, pushTrack)
  │    │             → AheadForPush, BehindForPush (gone 丢弃)
  │    │
  │    ├─ [recency 模式] obtainReflogBranches() + 合并重排
  │    │
  │    ├─ config.Branches(cmd) → 补 UpstreamRemote, UpstreamBranch
  │    │
  │    ├─ 继承旧 BehindBaseBranch 值（减少闪烁）
  │    │
  │    └─ loadBehindCounts && showDivergence != "none"?
  │         └─ 是 → onWorker(GetBehindBaseBranchValuesForAllBranches)
  │                  ├─ git ≥ 2.41: for-each-ref %(ahead-behind:<base>)
  │                  └─ git < 2.41: per-branch merge-base + rev-list
  │                    → 全部完成后 renderFunc() → UI 线程重绘
  │
  ├─ Model.Branches = branches
  ├─ refreshView(Branches)  ← 第一次渲染（BehindBaseBranch 用继承值）
  │
  └─ [BehindBaseBranch 计算完成后] renderFunc()
       → OnUIThread: HandleRender + refreshStatus  ← 第二次渲染（显示正确 ↓N）
       │
       ▼
presentation.GetBranchListDisplayStrings()
  │
  ├─ BranchStatus()
  │   ├─ itemOperation  → "Pushing |" (优先)
  │   ├─ UpstreamGone   → "upstream gone" (红色)
  │   ├─ MatchesUpstream() → "✓" (绿色)
  │   ├─ RemoteBranchNotStoredLocally() → "?" (品红)
  │   ├─ IsBehindForPull() && IsAheadForPull() → "↓N↑M" (黄色)
  │   ├─ IsBehindForPull() → "↓N" (黄色)
  │   └─ IsAheadForPull() → "↑N" (黄色)
  │       ↑ 仅用 *ForPull，不用 *ForPush
  │
  └─ divergenceStr()
      └─ BehindBaseBranch.Load() → "↓N" (青色，右对齐)
          ↑ 与 BranchStatus 独立，反映对 base branch 而非 upstream
       │
       ▼
ListRenderer 渲染到 Branches View
```

## 八、关键设计要点

1. **双来源互补**：追踪状态来自两处——`git for-each-ref` 的 `%(upstream:track)` 提供 ahead/behind 计数和 [gone] 状态，`git config` 提供 remote 和 merge 名称。前者依赖本地远端引用缓存，后者始终可用。

2. **"?" 的语义**：`AheadForPull == "?"` 并非异常，而是表示"本地没有对应远端分支的引用，无法计算差异"。常见于配置了 upstream 但从未执行过 fetch。

3. **Push 计数隐藏设计**：`AheadForPush` / `BehindForPush` 在 UI 上不可见，仅在 push 操作时用于判断是否需要 force push 确认。这是三角工作流（pull from origin, push to fork）的必要支持，但对普通用户来说是无感的安全保障。

4. **排序影响刷新路径**：`recency` 模式下必须走 `refreshReflogAndBranches` 串行链路，而 `date` / `alphabetical` 模式下分支与 reflog 可以并行加载。

5. **recency 模式的两次刷新策略**：启动时先用 `loadBehindCounts=false` 快速展示分支列表，reflog 加载完后再用 `loadBehindCounts=true` 刷新一次并调度 BehindBaseBranch 计算。两次刷新之间状态仍为 INITIAL，第二次刷新完成后才切换到 COMPLETE。

6. **date/alphabetical 模式无两阶段**：与 recency 模式不同，`date` / `alphabetical` 排序下启动第一次刷新就硬编码 `loadBehindCounts=true`，BehindBaseBranch 计算立即调度，无需等 reflog。

7. **渐进式渲染三层机制**：① 从旧分支列表继承 BehindBaseBranch 减少闪烁；② `loadBehindCounts=true` 仅调度不阻塞；③ 计算完成后通过 `renderFunc()` 切 UI 线程局部重绘。

8. **状态优先级**：展示时 `itemOperation`（正在 push/pull 等）优先级最高，覆盖所有追踪状态显示；其次是 `UpstreamGone`，最后才是常规 ahead/behind 状态。

9. **两个独立维度**：BranchStatus（`✓`/`↓N↑M` 等）反映与 **upstream** 的同步关系，divergenceStr（右对齐 `↓N`）反映与 **base branch** 的落后关系，两者在视觉位置和语义上都做了区分。
