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

文件：[branch.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/commands/models/branch.go#L10-L43)

上游追踪状态相关字段：

| 字段 | 含义 |
|------|------|
| `UpstreamRemote` | 追踪的远端名称，如 "origin"；来自 git config |
| `UpstreamBranch` | 追踪的远端分支名，如 "main"；来自 git config |
| `AheadForPull` | 相对 upstream 领先的提交数（可 pull 的反向），来自 `%(upstream:track)` |
| `BehindForPull` | 相对 upstream 落后的提交数（可 pull 的数量），来自 `%(upstream:track)` |
| `AheadForPush` | 相对 push 目标分支领先的提交数（三角工作流），来自 `%(push:track)` |
| `BehindForPush` | 相对 push 目标分支落后的提交数，来自 `%(push:track)` |
| `UpstreamGone` | 追踪的远端分支是否已被删除，track 为 `[gone]` 时为 true |
| `BehindBaseBranch` | 相对基准分支（main/develop 等）的落后提交数，`atomic.Int32` 异步计算 |

### 2.2 状态判断辅助方法

文件：[branch.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/commands/models/branch.go#L92-L125)

| 方法 | 判定条件 | 含义 |
|------|---------|------|
| `IsTrackingRemote()` | `UpstreamRemote != ""` | 是否配置了上游追踪 |
| `RemoteBranchStoredLocally()` | 已追踪 且 `AheadForPull != "?" && BehindForPull != "?"` | 远端分支引用是否已本地缓存（即 `upstream:short` 非空） |
| `RemoteBranchNotStoredLocally()` | 已追踪 且 `AheadForPull == "?" && BehindForPull == "?"` | 远端分支引用未在本地存储（配置了追踪但从未 fetch） |
| `MatchesUpstream()` | 已存储 且 ahead 和 behind 均为 "0" | 与上游完全同步 |
| `IsAheadForPull()` | 已存储 且 `AheadForPull != "0"` | 领先于上游（有可 push 的提交） |
| `IsBehindForPull()` | 已存储 且 `BehindForPull != "0"` | 落后于上游（有可 pull 的提交） |

**关键点**：当 AheadForPull/BehindForPull 为 `"?"` 时，表示本地没有对应远端分支引用，无法计算差异。

## 三、数据加载层：从 Git 获取原始数据

### 3.1 入口：BranchLoader.Load

文件：[branch_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/commands/git_commands/branch_loader.go#L67-L146)

加载流程：

1. `obtainBranches()` → 通过 `git for-each-ref` 获取所有本地分支及其追踪信息
2. 若按 recency 排序，通过 reflog 补充最近检出信息
3. 把 HEAD 分支移到列表首位
4. **从 git config 补充 UpstreamRemote 和 UpstreamBranch**（`for-each-ref` 没给）
5. 可选：异步计算 BehindBaseBranch（相对基准分支落后数）

### 3.2 获取原始分支数据：git for-each-ref

文件：[branch_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/commands/git_commands/branch_loader.go#L392-L428)

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
git for-each-ref --sort=-committerdate --format="%(HEAD)%00%(refname:short)%00%(upstream:short)%00%(upstream:track)%00%(push:track)%00%(subject)%00%(objectname)%00%(committerdate:unix)" refs/heads
```

### 3.3 解析追踪状态：parseUpstreamInfo

文件：[branch_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/commands/git_commands/branch_loader.go#L466-L491)

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

该函数被调用两次，分别处理 upstream:track 和 push:track。

### 3.4 从 git config 补全上游名称

文件：[config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/commands/git_commands/config.go#L80-L114)

调用：
```bash
git config --local --get-regexp ^branch\.
```

从输出中解析每行：
- `branch.<name>.remote <value>` → BranchConfig.Remote
- `branch.<name>.merge <value>` → BranchConfig.Merge（去除 `refs/heads/` 前缀）

然后在 `BranchLoader.Load` 中回填到每个 Branch 的 `UpstreamRemote` 和 `UpstreamBranch`。

**原因**：`git for-each-ref` 的 `%(upstream:short)` 只在本地缓存了远端引用时才有值，而 git config 始终保存追踪配置，两者互为补充。

## 四、刷新触发层：何时重新加载

### 4.1 刷新入口：RefreshHelper.Refresh

文件：[refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L63-L237)

当刷新范围包含 `BRANCHES` 或 `COMMITS` 时，会触发分支刷新：

- 按 recency 排序：`refreshReflogAndBranches()` → 先加载 reflog，再加载分支
- 其他排序：`refreshBranches()` → 直接加载分支

### 4.2 实际刷新：refreshBranches

文件：[refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L486-L543)

```go
branches, err := self.c.Git().Loaders.BranchLoader.Load(
    self.c.Model().ReflogCommits,      // reflog 提交（用于 recency 排序）
    self.c.Model().MainBranches,       // 基准分支列表
    self.c.Model().Branches,           // 旧分支列表（缓存 BehindBaseBranch）
    loadBehindCounts,                  // 是否异步计算 BehindBaseBranch
    onWorker,                          // 把任务调度到 worker goroutine
    renderFunc,                        // BehindBaseBranch 完成后触发 UI 重绘
)
```

BehindBaseBranch 的计算在异步 worker 中进行，完成后通过 `renderFunc` 在 UI 线程调用 `HandleRender()` 和 `refreshStatus()`，实现渐进式渲染。

## 五、展示渲染层：如何呈现给用户

### 5.1 展示入口：BranchesContext

文件：[branches_context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/gui/context/branches_context.go#L19-L61)

`getDisplayStrings` 调用 `presentation.GetBranchListDisplayStrings`，把 `[]*models.Branch` 转成 `[][]string` 供列表渲染。

### 5.2 追踪状态图标：BranchStatus

文件：[branches.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/gui/presentation/branches.go#L215-L245)

分支名称后面会附加状态标记，优先级如下：

| 状态 | 显示 | 颜色 | 判定条件 |
|------|------|------|---------|
| 正在操作中 | `Pushing \|` 等 + spinner | 青色 | 有 `itemOperation`（优先显示） |
| 上游已删除 | `upstream gone` | 红色 | `UpstreamGone == true` |
| 与上游同步 | `✓` | 绿色 | `MatchesUpstream()` |
| 远端分支未本地存储 | `?` | 品红 | `RemoteBranchNotStoredLocally()` |
| 既领先又落后 | `↓N↑M` | 黄色 | `IsBehindForPull() && IsAheadForPull()` |
| 仅落后 | `↓N` | 黄色 | `IsBehindForPull()` |
| 仅领先 | `↑N` | 黄色 | `IsAheadForPull()` |
| 无追踪 | （空） | - | `!IsTrackingRemote()` |

### 5.3 基准分支分歧：divergenceStr

文件：[branches.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/gui/presentation/branches.go#L247-L266)

在分支名**右侧**显示相对基准分支的落后数（由配置 `gui.showDivergenceFromBaseBranch` 控制）：

- `none`：不显示
- `onlyArrow`：显示 `↓`
- `arrowAndNumber`：显示 `↓N`

### 5.4 完整渲染：getBranchDisplayStrings

文件：[branches.go](file:///d:/fz/0601-2/solo-dogfeeding/code/24-lazygit/pkg/gui/presentation/branches.go#L47-L186)

每一行的列组成（按顺序）：

| 列 | 内容 | 说明 |
|----|------|------|
| 1 | `Recency` | 最近检出时间 / `  *`（HEAD 标记） |
| 2 | PR 图标 | 若开启 GitHub PR 集成 |
| 3 | Commit hash | `fullDescription` 或配置 `showBranchCommitHash` 开启时显示 |
| 4 | 分支名 + 状态 | 含 worktree 标记、BranchStatus、divergenceStr（右对齐） |
| 5 | 上游信息 | `fullDescription` 时显示：`UpstreamRemote UpstreamBranch` |
| 6 | Commit 标题 | `fullDescription` 时显示 |

## 六、完整调用链路图

```
触发刷新 (Refresh)
  │
  ▼
refresh_helper.refreshBranches()
  │
  ├─ git_commands.BranchLoader.Load()
  │    │
  │    ├─ obtainBranches()
  │    │    ├─ getRawBranches()
  │    │    │   └─ git for-each-ref refs/heads  ── 取 %(upstream:short), %(upstream:track), %(push:track)
  │    │    │
  │    │    └─ obtainBranch(split)
  │    │         └─ parseUpstreamInfo(upstreamName, track)
  │    │              ├─ upstream:short 空 → ("?","?",false)
  │    │              ├─ track="[gone]"   → ("?","?",true)
  │    │              └─ 正则提取 ahead/behind 数字
  │    │
  │    └─ config.Branches(cmd)
  │         └─ git config --local --get-regexp ^branch\.  ── 补 UpstreamRemote, UpstreamBranch
  │
  └─ 异步（可选）：BehindBaseBranch 计算完成 → renderFunc → HandleRender
       │
       ▼
presentation.GetBranchListDisplayStrings()
  │
  ├─ BranchStatus()  → ✓ / ↓N / ↑N / ↓N↑M / ? / upstream gone
  └─ divergenceStr() → ↓N (相对基准分支)
       │
       ▼
ListRenderer 渲染到 Branches View
```

## 七、关键设计要点

1. **双来源互补**：追踪状态来自两处——`git for-each-ref` 的 `%(upstream:track)` 提供 ahead/behind 计数和 [gone] 状态，`git config` 提供 remote 和 merge 名称。前者依赖本地远端引用缓存，后者始终可用。

2. **"?" 的语义**：`AheadForPull == "?"` 并非异常，而是表示"本地没有对应远端分支的引用，无法计算差异"。常见于配置了 upstream 但从未执行过 fetch。

3. **渐进式渲染**：`BehindBaseBranch` 使用 `atomic.Int32` 存储，在后台 worker 中计算，完成后触发局部 UI 刷新，避免阻塞。

4. **三角工作流支持**：同时维护 `*ForPull` 和 `*ForPush` 两套计数，分别对应 upstream 分支和 push 目标分支，两者可以不同。

5. **状态优先级**：展示时 `itemOperation`（正在 push/pull 等）优先级最高，其次是 `UpstreamGone`，最后才是常规 ahead/behind 状态。
