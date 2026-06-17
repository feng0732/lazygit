# Lazygit 远端同步（Push/Pull）触发链路代码分析

## 一、整体架构概览

Lazygit 的远端同步功能采用**分层架构**，从用户交互到底层命令执行分为以下几层：

```
┌───────────────────────────────────────────────────────────┐
│  用户交互层 (GUI)                                          │
│  SyncController - 按键绑定、前置检查、用户确认弹窗           │
├───────────────────────────────────────────────────────────┤
│  辅助控制层 (Helpers)                                      │
│  InlineStatusHelper - 内联加载动画                         │
│  MergeAndRebaseHelper - 冲突检测与处理                     │
│  UpstreamHelper - 上游分支设置提示                         │
│  RefreshHelper - 异步状态刷新                              │
├───────────────────────────────────────────────────────────┤
│  命令构建层 (Git Commands)                                 │
│  SyncCommands - push/pull/fetch 命令构建与执行             │
│  GitCommandBuilder - git 命令参数链式构建器                 │
├───────────────────────────────────────────────────────────┤
│  命令执行层 (OS Commands)                                  │
│  CmdObj - 命令对象封装（凭证策略、PTY、日志等）              │
│  gitCmdObjRunner - 带重试逻辑的 runner                     │
└───────────────────────────────────────────────────────────┘
```

---

## 二、Push 触发链路详解

### 2.1 入口：按键绑定与前置检查

**代码位置**：[sync_controller.go:32-51](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/sync_controller.go#L32-L51)

```go
// 按键绑定
{
    Keys:              opts.GetKeys(opts.Config.Universal.Push),  // 默认 'P'
    Handler:           opts.Guards.NoPopupPanel(self.HandlePush),
    GetDisabledReason: self.getDisabledReasonForPushOrPull,
    Description:       self.c.Tr.Push,
}
```

**禁用检查** `getDisabledReasonForPushOrPull`：[sync_controller.go:65-75](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/sync_controller.go#L65-L75)

- 通过 `GetItemOperation(currentBranch)` 检查该分支是否**正在进行** push/pull 操作
- 防止同一分支重复触发同步操作

### 2.2 核心流程：push() 方法

**代码位置**：[sync_controller.go:89-116](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/sync_controller.go#L89-L116)

```
HandlePush()
    └─> branchCheckedOut(push)()   // 确保当前分支已检出
        └─> push(currentBranch)
            │
            ├─ 情况1: 已跟踪远程分支 (IsTrackingRemote() == true)
            │   ├─ IsBehindForPush() == true  →  requestToForcePush()
            │   └─ IsBehindForPush() == false →  pushAux(opts)
            │
            ├─ 情况2: pushToCurrent 配置开启
            │   └─ pushAux(opts{setUpstream: true})
            │
            └─ 情况3: 未跟踪远程
                └─ PromptForUpstreamWithInitialContent()  →  pushAux()
```

**分支状态判断关键方法**（[branch.go:92-125](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/models/branch.go#L92-L125)）：

| 方法 | 判断逻辑 | 作用 |
|------|----------|------|
| `IsTrackingRemote()` | `UpstreamRemote != ""` | 是否已配置上游 |
| `RemoteBranchStoredLocally()` | 已跟踪 + Ahead/Behind 非 "?" | 本地是否有远程分支信息 |
| `IsBehindForPush()` | 远程分支已缓存 + `BehindForPush != "0"` | 是否需要 force-push |

### 2.3 命令准备：PushCmdObj

**代码位置**：[sync.go:22-46](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/git_commands/sync.go#L22-L46)

`PushOpts` 参数结构：
```go
type PushOpts struct {
    Force          bool   // --force
    ForceWithLease bool   // --force-with-lease
    CurrentBranch  string
    UpstreamRemote string
    UpstreamBranch string
    SetUpstream    bool   // --set-upstream
}
```

**命令构建过程**：
```go
cmdArgs := NewGitCmd("push").
    ArgIf(opts.Force, "--force").
    ArgIf(opts.ForceWithLease, "--force-with-lease").
    ArgIf(opts.SetUpstream, "--set-upstream").
    ArgIf(opts.UpstreamRemote != "", opts.UpstreamRemote).
    ArgIf(opts.UpstreamBranch != "", 
        fmt.Sprintf("refs/heads/%s:%s", opts.CurrentBranch, opts.UpstreamBranch)).
    ToArgv()

cmdObj := self.cmd.New(cmdArgs).PromptOnCredentialRequest(task)
```

**关键参数解释**：
- 当指定 `UpstreamBranch` 时，使用完整 refspec `refs/heads/<local>:<remote>` 避免歧义
- `PromptOnCredentialRequest(task)` 设置凭证策略为 `PROMPT`，并启用 PTY

### 2.4 内联状态显示：WithInlineStatus

**代码位置**：[inline_status_helper.go:65-87](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/inline_status_helper.go#L65-L87)

```
pushAux()
    └─> WithInlineStatus(branch, ItemOperationPushing, LOCAL_BRANCHES_CONTEXT_KEY,
            func(task) error {
                // 执行 git push
                err := Sync.Push(task, opts)
                // 错误处理...
                Refresh(ASYNC)
            })
```

**工作原理**：

1. **可见性判断**：如果分支项在视图中可见，启动内联 spinner 动画
2. **设置操作状态**：`SetItemOperation(item, ItemOperationPushing)` 
3. **启动 ticker**：按 `Gui.Spinner.Rate` 间隔周期性重绘上下文
4. **包装 Task**：`inlineStatusHelperTask` 在 Pause/Continue 时同步隐藏/显示 spinner
5. **defer 清理**：操作完成后 `stop()` 清除状态，停止 ticker

**ItemOperation 存储结构**（[gui.go:202-221](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/gui.go#L202-L221)）：
- `itemOperations` map：`map[string]ItemOperation`，key 为 item 的 `URN()`
- 受 `itemOperationsMutex` 互斥锁保护

### 2.5 Push 错误处理

**代码位置**：[sync_controller.go:195-235](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/sync_controller.go#L195-L235)

错误处理决策树：

```
git push 返回 err
    │
    ├─ 非 force 模式 && 错误信息包含 "Updates were rejected"
    │   │
    │   ├─ remoteBranchStoredLocally == true
    │   │   └─ 返回 "UpdatesRejected" 提示用户先 fetch
    │   │
    │   └─ remoteBranchStoredLocally == false
    │       ├─ DisableForcePushing 配置开启 → 返回禁用提示
    │       └─ 弹 Confirm 框询问是否 force push
    │           └─ 确认 → 递归调用 pushAux(force=true)
    │
    └─ 其他错误 → 直接返回 err（全局错误处理器弹窗）
```

**提前检测落后场景**：`requestToForcePush()` [sync_controller.go:237-253](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/sync_controller.go#L237-L253)
- 在执行 push 前就已知本地落后
- 直接弹框询问，使用更安全的 `--force-with-lease` 而非 `--force`

---

## 三、Pull 触发链路详解

### 3.1 入口与前置检查

与 Push 共享相同的按键绑定机制和禁用检查。

**代码位置**：[sync_controller.go:118-133](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/sync_controller.go#L118-L133)

### 3.2 Pull 流程核心逻辑

```
HandlePull()
    └─> branchCheckedOut(pull)()
        └─> pull(currentBranch)
            │
            ├─ 未跟踪远程分支
            │   ├─ PromptForUpstreamWithInitialContent()
            │   ├─ setCurrentBranchUpstream()   // 设置上游
            │   └─ PullAux()
            │
            └─ 已跟踪远程分支
                └─ PullAux()
```

**设置上游的错误处理** `setCurrentBranchUpstream()`：
- 如果上游不存在（错误包含 "does not exist"），给出明确提示：
  - 提示用户 fetch (`f`) 或 push (`shift+P`)
- 其他错误直接返回

### 3.3 Pull 命令准备

**代码位置**：[sync.go:90-111](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/git_commands/sync.go#L90-L111)

```go
cmdArgs := NewGitCmd("pull").
    Arg("--no-edit").                                    // 不打开编辑器编辑 merge message
    ArgIf(opts.FastForwardOnly, "--ff-only").
    ArgIf(opts.RemoteName != "", opts.RemoteName).
    ArgIf(opts.BranchName != "", "refs/heads/"+opts.BranchName).
    GitDirIf(opts.WorktreeGitDir != "", opts.WorktreeGitDir).
    WorktreePathIf(opts.WorktreePath != "", opts.WorktreePath).
    ToArgv()

// 关键：绕过交互式 rebase 编辑器
return self.cmd.New(cmdArgs).
    AddEnvVars("GIT_SEQUENCE_EDITOR=:").   // ':' 表示什么都不做直接退出
    PromptOnCredentialRequest(task).
    Run()
```

**重要设计细节**：
- `--no-edit`：避免自动弹出编辑器
- `GIT_SEQUENCE_EDITOR=:`：覆盖用户 `pull.rebase = interactive` 配置，防止挂起
- 支持 worktree 场景（`GitDirIf`/`WorktreePathIf`）

### 3.4 Pull 错误处理：CheckMergeOrRebase

**代码位置**：[merge_and_rebase_helper.go:152-182](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L152-L182)

这是 Pull（以及 Merge、Rebase、Cherry-pick）共享的冲突处理核心。

**关键纠正**：`CheckMergeOrRebase` 调用的是 **ASYNC** 刷新，不是 SYNC！

```go
func (self *MergeAndRebaseHelper) CheckMergeOrRebase(result error) error {
    return self.CheckMergeOrRebaseWithRefreshOptions(result, 
        types.RefreshOptions{Mode: types.ASYNC})  // ✅ ASYNC
}
```

**为什么可以用 ASYNC？**
冲突检测分两层：第一层基于 git 命令的错误字符串（即时），第二层在刷新过程中基于 Model.Files（持续）。

```
Pull 返回 err
    └─> CheckMergeOrRebase(err)
        ├─ Refresh(ASYNC)   // 立即返回，后台开始刷新（非阻塞）
        │
        ├─ err == nil → 成功，返回 nil
        │
        ├─ 特殊空提交处理
        │   ├─ "No changes - did you forget to use" → skip (rebase)
        │   └─ "The previous cherry-pick is now empty" → skip
        │
        ├─ "No rebase in progress?" → 认为已完成，返回 nil
        │
        └─ CheckForConflicts(err)  // ✅ 基于 err 字符串，不依赖 Model.Files
            ├─ 匹配冲突关键词 → PromptForConflictHandling()
            └─ 不匹配 → 返回原错误

    （后台 ASYNC 刷新同时进行，负责第二层冲突处理）
    └─ refreshStateFiles()
        ├─ 自动 stage 已解决的内联冲突
        ├─ 冲突 N→0 检测 → PromptToContinueRebase
        └─ 文件过滤自动切换
```

**冲突关键词检测** `isMergeConflictErr()`：
```
"Failed to merge in the changes"
"When you have resolved this problem"
"fix conflicts"
"Resolve all conflicts manually"
"Merge conflict in file"
"hint: after resolving the conflicts"
"CONFLICT (content):"
```

**冲突处理菜单** `PromptForConflictHandling()`：
1. **查看冲突文件**：跳转到 Files 上下文（自动过滤冲突文件）
2. **Abort**：执行 `git merge --abort` 或 `git rebase --abort`

---

## 四、核心机制详解

### 4.1 凭证请求处理：PromptOnCredentialRequest

**代码位置**：[cmd_obj.go:42-55,204-217](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/oscommands/cmd_obj.go#L42-L55)

三种凭证策略：

| 策略 | 值 | 行为 | 使用场景 |
|------|----|------|----------|
| `NONE` | 0 | 不处理凭证，命令可能挂起 | 本地操作 |
| `PROMPT` | 1 | 启用 PTY，检测到用户名/密码提示时弹框输入用户 | push/pull/fetch |
| `FAIL` | 2 | 启用 PTY，检测凭证请求时发送换行让其失败 | 后台 fetch |

**设置代码**：
```go
func (self *CmdObj) PromptOnCredentialRequest(task gocui.Task) *CmdObj {
    self.credentialStrategy = PROMPT
    self.usePty = true          // 必须使用 PTY 才能与提示交互
    self.task = task            // 关联 Task 用于暂停/继续
    return self
}
```

### 4.2 命令执行重试机制

**代码位置**：[git_cmd_obj_runner.go:13-76](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/git_cmd_obj_runner.go#L13-L76)

`gitCmdObjRunner` 包装了基础 runner，针对 Git 锁文件问题提供**自动重试**：

```go
RetryCount = 5
WaitTime   = 50 * time.Millisecond
```

**可重试错误检测** `isRetryableError()`：
- `.git/index.lock` 存在
- `cannot lock ref`（ref 锁冲突）

### 4.3 Refresh 刷新机制（完整流程）

**代码位置**：[refresh_helper.go:63-237](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L63-L237)

#### ❗ 核心纠正：三种刷新模式的真实区别

之前的理解有误，**ASYNC 和 SYNC 都是并行，区别在于 Refresh() 是否等待所有 scope 完成**。

关键代码在 `refresh()` 包装函数 [refresh_helper.go:110-128](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L110-L128)：

```go
refresh := func(name string, f func()) {
    if !self.c.InDemo() && options.Mode == types.ASYNC {
        // ASYNC: 每次调 OnWorker 都启动一个独立 goroutine
        self.c.OnWorker(func(t gocui.Task) error {
            f()
            return nil
        })
    } else {
        // SYNC / BLOCK_UI: 每次也启动一个独立 goroutine，但参与 wg
        wg.Add(1)
        go utils.Safe(func() {
            defer wg.Done()
            f()
        })
    }
}
```

**OnWorker 的底层实现** [gui.go:650-656](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gocui/gui.go#L650-L656)：
```go
func (g *Gui) OnWorker(f func(Task) error) {
    task := g.NewTask()
    go func() {          // ← 每次调 OnWorker 都启动一个独立 goroutine！
        g.onWorkerAux(f, task)
        task.Done()
    }()
}
```

**ASYNC 和 SYNC 的真正区别不是"串行 vs 并行"，而是"fire-and-forget vs 等待完成"：**

- **ASYNC**：每个 scope 启动独立 goroutine → `wg.Wait()` 无事可等（scope 不调 `wg.Add`）→ `Refresh()` 立即返回
- **SYNC**：每个 scope 启动独立 goroutine + `wg.Add(1)` → `wg.Wait()` 阻塞到所有 scope 完成 → `Refresh()` 在所有 scope 完成后才返回
- **BLOCK_UI**：`f()` 在 UI 线程上执行（scope 内部仍是 goroutine 并行），`wg.Wait()` 阻塞 UI 线程

三种模式对比：

| 模式 | scope 启动方式 | scope 间关系 | `Refresh()` 是否等待 | 能否立即读 `Model.*` | 典型场景 |
|------|---------------|-------------|---------------------|---------------------|----------|
| **ASYNC** | `OnWorker` → `go func()` | **并行**（各自独立 goroutine） | ❌ 立即返回 | ❌ 不能 | push/pull 成功后 |
| **SYNC** | `go utils.Safe()` + `wg.Add` | **并行**（各自独立 goroutine） | ✅ `wg.Wait()` 阻塞 | ✅ 能（所有 scope 完成后） | PromptToContinueRebase |
| **BLOCK_UI** | `OnUIThread` 上执行 `f()`，scope 内仍是 `go func()` | **并行** | ✅ 阻塞 UI 线程等 wg | ✅ 能 | 切换分支 |

**ASYNC 下 `wg.Wait()` 为什么空转？** 因为 ASYNC 分支不调 `wg.Add(1)`，所以 `wg` 计数器始终为 0，`wg.Wait()` 立即返回。这就是 ASYNC + `Then` 组合会 panic 的原因 [refresh_helper.go:64-66](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L64-L66)——Then 本应在所有 scope 完成后执行，但 wg 不跟踪 scope，Then 无法保证时机。

**ASYNC 下 `Refresh()` 的 `f()` 何时执行？** 

[refresh_helper.go:228-236](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L228-L236)：
```go
if options.Mode == types.BLOCK_UI {
    self.c.OnUIThread(func() error { f(); return nil })
    return
}
f()   // ← ASYNC 和 SYNC 都在调用线程直接执行 f()
```

`f()` 包含两件事：
1. 逐个调 `refresh()` 提交 scope（ASYNC 下每次 `OnWorker` 启动新 goroutine）
2. 调 `refreshStatus()` 和 `wg.Wait()`

所以 ASYNC 模式下，`f()` 在调用线程**同步执行**，但 scope 的实际工作（如 `refreshBranches()`）已经在各自的 goroutine 中**并行**开始了。`f()` 执行到 `wg.Wait()` 时因为 wg 为空立即返回，`Refresh()` 结束。

#### ❗ CheckMergeOrRebase 实际用 ASYNC，不是 SYNC

**代码事实** [merge_and_rebase_helper.go:168-170](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L168-L170)：
```go
func (self *MergeAndRebaseHelper) CheckMergeOrRebase(result error) error {
    return self.CheckMergeOrRebaseWithRefreshOptions(result, types.RefreshOptions{Mode: types.ASYNC})
}
```

**为什么 ASYNC 也能正确检测冲突？**

因为冲突检测分**两层**：
- **第一层（即时）**：`CheckForConflicts()` 基于 **git 命令返回的错误字符串**（如 "CONFLICT (content):"），不是 `Model.Files`
- **第二层（持续）**：`refreshStateFiles()` 在刷新过程中基于 `Model.Files` 做自动处理

完整的 Pull 冲突流程：
```
git pull 返回 err (含 "CONFLICT (content): ...")
    │
    ▼
CheckMergeOrRebase(err)
    ├─ Refresh(ASYNC)  // 立即返回，后台开始刷新
    │
    ├─ 空提交处理（读 err 字符串）
    │
    └─ CheckForConflicts(err)
        └─ isMergeConflictErr(err.Error())  // ✅ 基于 err 字符串，不依赖 Model.Files
            └─ PromptForConflictHandling()  // 立即弹冲突菜单
                ├─ ViewConflicts → 跳 Files 面板
                └─ Abort → git merge --abort

    （同时，后台 ASYNC 刷新在进行）
    ├─ refreshBranches() → 更新 ahead/behind → Branches 重绘
    └─ refreshFilesAndSubmodules()
        ├─ refreshStateFiles()  // 第二层检测
        │   ├─ 自动 stage 已解决的内联冲突文件
        │   ├─ 冲突 N→0 检测 → PromptToContinueRebase
        │   └─ 文件过滤自动切换
        └─ refreshView(Files) → Files 面板显示冲突文件
```

**只有 PromptToContinueRebase 用 SYNC** [merge_and_rebase_helper.go:236-238](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L236-L238)：
```go
// 这里用 SYNC 是因为后面要立即读 Model.Files
self.c.Refresh(types.RefreshOptions{
    Mode: types.SYNC, Scope: []types.RefreshableView{types.FILES},
})
unstagedFiles := GetUnstagedFilesExceptSubmodules(self.c.Model().Files, ...)
```

#### Push/Pull 触发的刷新 Scope（默认全量）：

```
COMMITS, BRANCHES, FILES, STASH, REFLOG, TAGS, REMOTES,
WORKTREES, STATUS, BISECT_INFO, STAGING, PULL_REQUESTS
```

#### 并行执行策略与依赖关系：

```
Refresh() 入口
    │
    ├─ commits and commit files
    │   └─ refreshCommitsAndCommitFiles()
    │       ├─ refreshCommitsWithLimit()  →  Model.Commits
    │       ├─ 计算 CheckedOutBranch
    │       ├─ refreshCommitFilesContext()
    │       └─ refreshView(LocalCommits)  // OnUIThread
    │
    ├─ branchesAndRemotesWg (WaitGroup)
    │   ├─ branches (+ reflog)
    │   │   └─ refreshBranches() / refreshReflogAndBranches()
    │   │       ├─ BranchLoader.Load()  →  Model.Branches
    │   │       ├─ 异步加载 BehindBaseBranch (onWorker 回调)
    │   │       ├─ 恢复选中分支索引
    │   │       ├─ refreshView(Branches)  // OnUIThread
    │   │       └─ refreshStatus()
    │   │
    │   └─ remotes
    │       └─ refreshRemotes()
    │           ├─ RemoteLoader.GetRemotes()  →  Model.Remotes
    │           │   ├─ 并行: getRemoteBranchesByRemoteName()
    │           │   └─ getRemotesFromConfig()
    │           ├─ rebuildPullRequestsMap()
    │           ├─ refreshView(Remotes)
    │           └─ refreshView(RemoteBranches)
    │
    ├─ fileWg (WaitGroup)
    │   └─ refreshFilesAndSubmodules()
    │       ├─ refreshStateSubmoduleConfigs()
    │       ├─ refreshStateFiles()  // 第二层冲突处理
    │       │   ├─ FileLoader.GetStatusFiles()  →  Model.Files
    │       │   ├─ 自动 stage 已解决的内联冲突
    │       │   ├─ 检测冲突从有到无 → PromptToContinueRebase
    │       │   └─ 文件树过滤自动切换
    │       ├─ FileTreeViewModel.SetTree()
    │       └─ OnUIThread:
    │           ├─ refreshView(Submodules)
    │           └─ refreshView(Files)
    │
    ├─ stash 线程 (独立)
    ├─ tags 线程 (独立)
    ├─ worktrees 线程 (独立，可选)
    ├─ reflog 线程 (独立，可选)
    ├─ sub_commits 线程 (独立，可选)
    │
    ├─ PULL_REQUESTS (依赖 branchesAndRemotesWg)
    │   └─ refreshGithubPullRequests()
    │
    ├─ STAGING (依赖 fileWg)
    │   └─ StagingHelper.RefreshStagingPanel()
    │
    ├─ MERGE_CONFLICTS / FILES  ❗ **独立 goroutine，不等待任何 wg！**
    │   └─ mergeConflictsHelper.RefreshMergeState()
    │       └─ 当前上下文不是 MergeConflicts 时直接 return nil
    │
    └─ refreshStatus()  ❗ 调度完所有 scope **立即**在当前线程同步执行
        ├─ GetCheckedOutRef() 为 nil 时（branches 还没跑完）直接 return
        └─ 否则 FormatStatus() → SetViewContent(Status)
```

#### Scope 间的 WaitGroup 依赖（**代码事实**）：

```
commitsAndCommitFiles ─┐
branchesAndRemotes    ─┤
files/submodules      ─┼── sync.WaitGroup (wg) 等待  (仅 SYNC/BLOCK_UI 模式下生效)
stash                 ─┤
tags                  ─┤
worktrees             ─┘
                           │
                           ▼
                  PULL_REQUESTS scope ──▶ 内部 branchesAndRemotesWg.Wait()
                  STAGING scope       ──▶ 内部 fileWg.Wait()
                  MERGE_CONFLICTS scope ──▶ ❌ 不 Wait，和 FILES 完全并行
```

**补充事实**：
- `ASYNC` 模式下所有 scope 都通过 `OnWorker` 提交，**不参与 wg.Add/Done**，所以 `wg.Wait()` 几乎立即返回，空转
- 因此 ASYNC + `Then` 组合会 **panic** [refresh_helper.go:64-66](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L64-L66)
- `MERGE_CONFLICTS` 触发的是 `RefreshMergeState()`，它通过 `Current().GetKey() == MERGE_CONFLICTS_CONTEXT_KEY` 决定要不要干活，而不是通过 scope 顺序保证文件已就绪

#### ASYNC vs SYNC 完整时序对比

**ASYNC 模式（push/pull 成功后）：**
```
调用方线程 (Worker goroutine，执行 push/pull 回调)
├─ Refresh(ASYNC)
│   ├─ f() 在当前线程同步执行:
│   │   ├─ refresh("commits", ...) → OnWorker → go goroutine_A  ─┐
│   │   ├─ refresh("branches", ...) → OnWorker → go goroutine_B  │ 同时并行！
│   │   ├─ refresh("files", ...) → OnWorker → go goroutine_C     │
│   │   ├─ ...                                                    │
│   │   ├─ refreshStatus()   // branches 还没完成，大概率 early return
│   │   └─ wg.Wait()         // wg 为空，立即返回
│   └─ Refresh() 立即返回 → 调用方继续执行（InlineStatus stop 等）
│
├────── 各 scope goroutine 并行执行 ──────
│
│  goroutine_A: refreshCommitsAndCommitFiles()
│    └─ refreshView(LocalCommits) → OnUIThread → 渲染
│
│  goroutine_B: refreshBranches()
│    ├─ refreshView(Branches) → OnUIThread → 渲染（✓ 出现）
│    └─ refreshStatus()  // 状态栏真正更新的地方
│
│  goroutine_C: refreshFilesAndSubmodules()
│    ├─ refreshStateFiles()  // 冲突自动处理
│    └─ OnUIThread: refreshView(Files) → 渲染
│
│  ... 其他 scope goroutine 并行执行
```

**SYNC 模式（PromptToContinueRebase）：**
```
调用方线程 (UI 线程，弹 Continue 确认框回调)
├─ Refresh(SYNC, Scope: [FILES])
│   ├─ f() 在当前线程同步执行:
│   │   ├─ refresh("files", ...)
│   │   │   └─ wg.Add(1) + go goroutine_X {
│   │   │              refreshFilesAndSubmodules()  // 独立 goroutine 并行
│   │   │              wg.Done()
│   │   │          }
│   │   │
│   │   └─ wg.Wait()  // ⚠️ 阻塞当前线程，直到 goroutine_X 完成
│   │
│   └─ Refresh() 返回 → Model.Files 已是最新 ✅
│
│  // 现在可以安全地读 Model.Files
│  unstagedFiles := GetUnstagedFilesExceptSubmodules(self.c.Model().Files, ...)
│  if len(unstagedFiles) > 0 {
│      // 弹确认框询问是否 auto-stage
│  }
```

**注意**：BLOCK_UI 模式与 SYNC 类似，但 `f()` 在 UI 线程上执行（通过 `OnUIThread` 提交到 `userEvents` channel，在主循环中被取出执行），scope 内部仍用 `go func()` 并行。UI 线程被 `wg.Wait()` 阻塞直到所有 scope 完成，期间不处理按键和渲染。

#### refreshView 总是 OnUIThread 异步渲染

**代码位置** [refresh_helper.go:785-805](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L785-L805)：

```go
func (self *RefreshHelper) refreshView(context types.Context) {
    // 注释明确说明：从 worker goroutine 调用，所以 bounce 到 UI 线程
    self.c.OnUIThread(func() error {
        self.searchHelper.ReApplyFilter(context)       // 1. 重新应用过滤
        self.c.PostRefreshUpdate(context)              // 2. HandleRender + 焦点
        self.c.AfterLayout(func() error {              // 3. 布局后重应用搜索
            self.searchHelper.ReApplySearch(context)
            return nil
        })
        return nil
    })
}
```

每个 scope 的 refresh 函数**末尾**调用 `refreshView()`：
- `refreshBranches()` → `self.refreshView(self.c.Contexts().Branches)`
- `refreshCommitsAndCommitFiles()` → `self.refreshView(self.c.Contexts().LocalCommits)`
- `refreshFilesAndSubmodules()` → OnUIThread 中调用 `refreshView(Files)`

---

### 4.4 分支计数（Ahead/Behind）加载详解

**代码位置**：[branch_loader.go:66-492](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/git_commands/branch_loader.go#L66-L492)

#### 数据源：`git for-each-ref` 单次命令

Lazygit 通过一次 `git for-each-ref` 调用获取所有分支信息，字段包括：

```go
var branchFields = []string{
    "HEAD",              // 是否为当前分支 "*"
    "refname:short",     // 分支名
    "upstream:short",    // 上游分支名 (如 origin/main)
    "upstream:track",    // 上游 track 信息 (ahead N, behind M)
    "push:track",        // push remote track 信息
    "subject",           // commit subject
    "objectname",        // commit hash
    "committerdate:unix" // commit 时间戳
}
```

#### 解析流程 `parseUpstreamInfo()`：[branch_loader.go:466-491](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/git_commands/branch_loader.go#L466-L491)

```
upstreamName == ""
   ├─ 是 → 返回 ("?", "?", false)  // 远程分支未在本地缓存
   └─ 否
       ├─ track == "[gone]" → 返回 ("?", "?", true)  // 上游已删除
       └─ 正则解析 track 字符串
           ├─ `ahead (\d+)`  → AheadForPull
           └─ `behind (\d+)` → BehindForPull
```

**注意**：Ahead/Behind 分两组：
- `AheadForPull` / `BehindForPull`：基于 `upstream:track`（fetch 后的 origin/xxx 比较）
- `AheadForPush` / `BehindForPush`：基于 `push:track`（push remote 比较）

#### 渲染显示 `BranchStatus()`：[branches.go:215-245](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/presentation/branches.go#L215-L245)

```go
ItemOperation != None  →  "Pushing ⠋"  或 "Pulling ⠋" (带 spinner)
UpstreamGone           →  "(gone)" (红色)
MatchesUpstream()      →  "✓" (绿色)
RemoteBranchNotStored  →  "?" (品红)
Ahead + Behind         →  "↓N↑M" (黄色)
仅 Behind              →  "↓N" (黄色)
仅 Ahead               →  "↑N" (黄色)
```

#### 基分支落后计数 `BehindBaseBranch`（异步加载）

push/pull 后 `BranchLoader.Load()` 会通过 `onWorker` 回调异步计算所有分支相对主分支（main/master）的落后数：

- **Git ≥ 2.41**：一次 `for-each-ref --format=%(ahead-behind:<base>)` 批量获取
- **Git < 2.41**：每个分支单独 `rev-list --left-right --count <branch>...<base>`，errgroup 并发

每计算完一个分支会调用 `renderFunc()` 触发 `OnUIThread` 重绘分支上下文。

---

### 4.5 远程信息（Remotes）刷新详解

**代码位置**：[remote_loader.go:31-164](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/git_commands/remote_loader.go#L31-L164)

#### 并行加载策略 `GetRemotes()`：

```
GetRemotes()
    ├─ goroutine A: getRemoteBranchesByRemoteName()
    │   └─ git for-each-ref --format=%(refname) refs/remotes
    │       └─ 解析每行 refs/remotes/<name>/<branch>
    │           └─ 构建 map[string][]*RemoteBranch
    │
    └─ 主线程: getRemotesFromConfig()
        └─ git config --local --get-regexp ^remote\.[^.]+\.(url|pushurl)$
            └─ 解析 remote name + urls + pushUrls
                └─ 构建 []*Remote (不含 Branches)

    └─ wg.Wait()  → 合并: remote.Branches = map[name]
    └─ 排序: origin 优先，其余按字母序
```

#### 刷新后的联动：`refreshRemotes()` [refresh_helper.go:689-720](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L689-L720)

1. 更新 `Model.Remotes`
2. `rebuildPullRequestsMap()`：根据远程分支重建 GitHub PR 映射
3. 保持选中的 remote，同步更新 `Model.RemoteBranches`
4. 刷新 Remotes 视图 + RemoteBranches 视图
5. 如果 PR 映射从空变非空，额外刷新 Branches 视图（显示 PR 图标）

---

### 4.6 文件状态刷新与冲突检测联动

**代码位置**：[file_loader.go:41-215](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/git_commands/file_loader.go#L41-L215)

#### 核心命令 `git status --porcelain -z`：

```go
cmdArgs := NewGitCmd("status").
    Arg("--untracked-files=all").
    Arg("--porcelain").
    Arg("-z").  // NUL 分隔，支持含空格的文件名
    Arg("--find-renames=50%").
    ToArgv()
```

#### 状态字段解析 `SetStatusFields()` / `deriveStatusFields()`：[file.go:132-164](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/commands/models/file.go#L132-L164)

每个文件的 2 字符状态码（XY）被解析为结构化字段：

| 状态码 | 含义 | HasMergeConflicts | HasInlineMergeConflicts |
|--------|------|-------------------|-------------------------|
| `UU` | 双方都修改 (unmerged) | ✅ | ✅ |
| `AA` | 双方都添加 | ✅ | ✅ |
| `DD` | 双方都删除 | ✅ | ❌ |
| `AU` | 我们添加，他们修改 | ✅ | ❌ |
| `UA` | 我们修改，他们添加 | ✅ | ❌ |
| `UD` | 我们修改，他们删除 | ✅ | ❌ |
| `DU` | 我们删除，他们修改 | ✅ | ❌ |
| `??` | 未跟踪 | ❌ | ❌ |
| `M ` | 已暂存修改 | ❌ | ❌ |
| ` M` | 未暂存修改 | ❌ | ❌ |

#### 刷新期间的自动处理 `refreshStateFiles()`：[refresh_helper.go:570-639](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L570-L639)

```
refreshStateFiles()
    │
    ├─ 自动 Stage 已解决的内联冲突文件 (AutoStageResolvedConflicts)
    │   └─ HasInlineMergeConflicts == true 的文件
    │       └─ 检查实际文件是否还有冲突标记
    │           └─ 无冲突 → 加入 pathsToStage → git add
    │
    ├─ 冲突消失检测 (从有到无)
    │   └─ WorkingTreeState.Any() && conflictFileCount == 0 && prevConflictFileCount > 0
    │       └─ OnUIThread → PromptToContinueRebase()
    │           └─ 弹 "Conflicts resolved, continue?" 确认框
    │
    └─ 文件过滤自动切换
        ├─ 冲突从 0 → N: DisplayAll → DisplayConflicted
        └─ 冲突从 N → 0: DisplayConflicted → DisplayAll
```

#### 文件渲染：[files.go:22-348](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/presentation/files.go#L22-L348)

- 绿色字符 = 暂存区状态（X）
- 红色字符 = 工作区状态（Y）
- 工作树目录特殊显示（检测到 worktree path 时去尾斜杠）
- 支持文件树（展开/折叠）和扁平两种展示模式

---

### 4.7 界面渲染顺序详解

从数据模型更新到最终屏幕显示，经过以下调用链：

#### 调用链：`Refresh → refreshView → postRefreshUpdate → HandleRender`

```
RefreshHelper.Refresh(ASYNC)
    │
    ├─ [Worker goroutine] 各 scope 加载数据 → 更新 Model.*
    │
    └─ refreshView(context)  [每个 scope 完成后触发]
        │
        └─ OnUIThread (切到 UI 线程)
            │
            ├─ ReApplyFilter(context)  // 重新应用搜索过滤
            │
            ├─ PostRefreshUpdate(context)
            │   │
            │   └─ Gui.postRefreshUpdate()  [view_helpers.go:127-163]
            │       │
            │       ├─ 1. c.HandleRender()
            │       │   │
            │       │   ├─ SimpleContext: 调用 handleRenderFunc()
            │       │   │
            │       │   └─ ListContextTrait.HandleRender()
            │       │       ├─ ClampSelection()  // 限制选中范围
            │       │       ├─ renderLines()     // 调用 presentation.* 生成字符串
            │       │       │   └─ getDisplayStrings()
            │       │       │       └─ e.g. GetBranchListDisplayStrings()
            │       │       │           ├─ 读 Model.Branches
            │       │       │           ├─ 读 State.itemOperations (for spinner)
            │       │       │           └─ BranchStatus() → ✓/↓N/↑M/?/(gone)
            │       │       ├─ SetContent() / SetViewPortContent()  // 写入 gocui.View 缓冲区
            │       │       └─ setFooter()  // "N of M"
            │       │
            │       ├─ 2. 焦点处理
            │       │   ├─ 当前视图 == c → HandleFocus() → FocusLine(true)
            │       │   └─ 否则 → FocusLine(false)
            │       │
            │       └─ 3. 主视图刷新
            │           ├─ 当前在主视图且非搜索中 → 侧面板 context.HandleRenderToMain()
            │           └─ 当前是弹框且 c 是静态上下文 → HandleRenderToMain()
            │
            └─ AfterLayout
                └─ ReApplySearch(context)  // 重新应用搜索高亮
```

#### 状态栏的特殊刷新 `refreshStatus()`：[refresh_helper.go:748-767](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L748-L767)

状态栏有**两个调用点**，不是通过 postRefreshUpdate：

**调用点 1（Refresh 主流程末尾）** [refresh_helper.go:219](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L219)：
- 所有 scope 调度完毕后**立即同步执行**（在调用 Refresh 的线程里）
- ASYNC 模式下此时 branches 大概率还没跑完，`GetCheckedOutRef()` 返回 nil → **直接 return，什么都不做**

```go
// Refresh() 主流程末尾
self.refreshStatus()   // 第 219 行

// refreshStatus 内部
currentBranch := self.refsHelper.GetCheckedOutRef()
if currentBranch == nil {
    // need to wait for branches to refresh
    return   // ← ASYNC 模式下大部分走这里
}
```

**调用点 2（refreshBranches 内部）** [refresh_helper.go:542](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L542)：
- branches scope 完成后一定会调用，此时 `Model.Branches` 已就绪
- 这是 ASYNC 模式下状态栏真正更新的时刻

状态栏渲染：
```go
status := presentation.FormatStatus(
    repoName,
    currentBranch,
    types.ItemOperationNone,      // ❗ 状态栏不显示 spinner（spinner 在分支行内显示）
    linkedWorktreeName,
    workingTreeState,              // 显示 "(rebasing)" / "(merging)" 等
    tr,
    userConfig,
)
self.c.SetViewContent(self.c.Views().Status, status)  // 直接写 View，不走 HandleRender
```

#### ❗ BehindBaseBranch 的独立二次渲染

在 `BranchLoader.Load()` 内部，基分支落后计数是通过 `onWorker` 回调**异步**计算的，**不受 Refresh 的 wg 控制**：

```go
// BranchLoader.Load 内部 [branch_loader.go:139-143]
if loadBehindCounts && self.UserConfig().Gui.ShowDivergenceFromBaseBranch != "none" {
    onWorker(func() error {
        return self.GetBehindBaseBranchValuesForAllBranches(branches, mainBranches, renderFunc)
    })
}
// renderFunc 的实际内容 [refresh_helper.go:500-506]
func() {
    self.c.OnUIThread(func() error {
        self.c.Contexts().Branches.HandleRender()  // 二次渲染：只渲染分支列表
        self.refreshStatus()                       // 二次渲染：状态栏
        return nil
    })
}
```

**效果**：ASYNC 模式下，push/pull 成功后屏幕会出现三次渲染：
1. branches scope 完成 → 第一次渲染（显示 `✓`，但 BehindBaseBranch 还是旧值）
2. BehindBaseBranch 异步计算完 → 第二次渲染（`↓N` 出现或更新）
3. files、remotes 等其他 scope 各自完成 → 各自面板独立渲染

#### 完整刷新时序（Push 成功后 ASYNC 模式，**代码事实修正版**）：

```
时间轴 →
│
│  git push 命令执行完毕 (在 WithInlineStatus 的 OnWorker goroutine 内)
│  │
│  ├─ InlineStatusHelper.stop()
│  │   ├─ ClearItemOperation(branch)
│  │   └─ 非 Demo 模式：不立即重绘
│  │
│  ▼
│  RefreshHelper.Refresh(ASYNC)  ← 在当前 goroutine 同步执行 f()
│  │
│  ├─ f() 同步执行调度逻辑:
│  │   ├─ refresh("commits")      → OnWorker → go goroutine_A  ─┐
│  │   ├─ refresh("branches")     → OnWorker → go goroutine_B  │ 各自独立 goroutine
│  │   ├─ refresh("files")        → OnWorker → go goroutine_C  │ 同时并行开始
│  │   ├─ refresh("remotes")      → OnWorker → go goroutine_D  │
│  │   ├─ ...                                                    │
│  │   ├─ refreshStatus()    // branches 还没完成 → early return
│  │   └─ wg.Wait()          // wg 为空 → 立即返回
│  │
│  ▼
│  Refresh() 返回 → pushAux 继续
│
├────── 以下各 goroutine 并行执行，谁先完成取决于 git 命令耗时 ──────
│
│  goroutine_A: refreshCommitsAndCommitFiles()
│    ├─ Model.Commits = newCommits
│    └─ refreshView(LocalCommits) → OnUIThread(userEvents channel) → UI 主循环取出执行
│
│  goroutine_B: refreshBranches()
│    ├─ BranchLoader.Load() → Model.Branches = newBranches
│    │   └─ 异步提交 BehindBaseBranch 计算（OnWorker → 又一个独立 goroutine）
│    ├─ refreshView(Branches) → OnUIThread → 渲染（✓ 出现，BehindBaseBranch 可能旧值）
│    └─ refreshStatus() → 状态栏刷新 ✅ （这才是状态栏真正更新的地方）
│
│  goroutine_C: refreshFilesAndSubmodules()
│    ├─ refreshStateFiles()
│    │   ├─ 自动 stage 已解决冲突
│    │   ├─ 冲突消失检测 → OnUIThread: PromptToContinueRebase()
│    │   └─ 文件过滤自动切换
│    └─ OnUIThread: refreshView(Files) + refreshView(Submodules)
│
│  goroutine_D: refreshRemotes()
│    ├─ Model.Remotes = newRemotes
│    └─ refreshView(Remotes) + refreshView(RemoteBranches)
│
│  ... 其他 scope goroutine 并行
│
│  BehindBaseBranch goroutine: （稍后完成）
│    └─ OnUIThread:
│        ├─ Branches.HandleRender()  → ↓N 出现或更新
│        └─ refreshStatus()
│
▼  屏幕最终稳定显示新状态
```

---

### 4.8 Upstream 提示机制

**代码位置**：[upstream_helper.go:51-56](file:///d:/fz/0601-2/solo-dogfeeding/code/25-lazygit/pkg/gui/controllers/helpers/upstream_helper.go#L51-L56)

```go
func PromptForUpstreamWithInitialContent(currentBranch, onConfirm) error {
    suggestedRemote := GetSuggestedRemote()  // 优先 "origin"
    initialContent := suggestedRemote + " " + currentBranch.Name
    return promptForUpstream(initialContent, onConfirm)
}
```

Prompt 特点：
- 自动填充建议值（`<remote> <branch>`）
- `FindSuggestionsFunc` 提供远程分支自动补全
- 用户确认后解析为 `upstreamRemote` 和 `upstreamBranch`

---

## 五、完整调用链总结

### 5.1 Push 完整调用链

```
用户按 'P' 键
  │
  ▼
SyncController.HandlePush()
  ├─ getDisabledReasonForPushOrPull()  // 检查是否已在操作
  ├─ branchCheckedOut(push)()          // 获取当前分支
  │
  ▼
SyncController.push(currentBranch)
  ├─ 已跟踪远程?
  │   ├─ 是 → IsBehindForPush()?
  │   │       ├─ 是 → requestToForcePush() 弹框 → force-with-lease
  │   │       └─ 否 → 正常 push
  │   ├─ pushToCurrent? → --set-upstream
  │   └─ 否 → PromptForUpstream 弹输入框
  │
  ▼
SyncController.pushAux(currentBranch, opts)
  │
  ▼
guiCommon.WithInlineStatus(branch, PUSHING, BRANCHES, callback)
  │
  ▼
InlineStatusHelper.WithInlineStatus()
  ├─ SetItemOperation(branch, PUSHING)   // 标记操作中
  ├─ 启动 spinner ticker (定时重绘)
  ├─ OnWorker(task) 中执行 callback
  │   │
  │   ▼
  │   SyncCommands.Push(task, opts)
  │     ├─ PushCmdObj() 构建命令参数
  │     │   └─ NewGitCmd("push").ArgIf(...)...PromptOnCredentialRequest()
  │     └─ cmdObj.Run()
  │         └─ gitCmdObjRunner.Run()  // 含重试逻辑
  │             └─ innerRunner (PTY + 凭证处理)
  │
  │   错误处理:
  │   ├─ "Updates were rejected" → 弹 force push 确认框
  │   ├─ 其他 err → 全局 ErrorHandler
  │   └─ 成功 → Refresh(ASYNC)
  │
  ▼
InlineStatusHelper.stop()
  ├─ ClearItemOperation(branch)  // 清除操作标记
  └─ 停止 spinner ticker
```

### 5.2 Pull 完整调用链

```
用户按 'p' 键
  │
  ▼
SyncController.HandlePull()
  ├─ getDisabledReasonForPushOrPull()
  ├─ branchCheckedOut(pull)()
  │
  ▼
SyncController.pull(currentBranch)
  ├─ 未跟踪远程?
  │   ├─ 是 → PromptForUpstream → setCurrentBranchUpstream()
  │   └─ 否 → 直接 PullAux
  │
  ▼
SyncController.PullAux(currentBranch, opts)
  │
  ▼
WithInlineStatus(branch, PULLING, BRANCHES, callback)
  │
  ▼
SyncCommands.Pull(task, PullOptions)
  ├─ NewGitCmd("pull").
  │   Arg("--no-edit").
  │   AddEnvVars("GIT_SEQUENCE_EDITOR=:").   // 防交互式挂起
  │   PromptOnCredentialRequest(task)
  └─ Run()
  │
  ▼
MergeAndRebaseHelper.CheckMergeOrRebase(err)
  ├─ Refresh(ASYNC)  // 先刷新文件状态
  ├─ 空提交特殊处理 → auto-skip
  ├─ 冲突检测 → PromptForConflictHandling() 菜单
  │   ├─ 查看冲突 (跳 Files 面板)
  │   └─ Abort 操作
  └─ 非冲突错误 → 返回原错误
  │
  ▼
InlineStatusHelper.stop()
  └─ ClearItemOperation + 停止 spinner
```

---

## 六、关键设计亮点与易错点

### 6.1 设计亮点

1. **防重复操作保护**：`ItemOperation` 状态 + `getDisabledReasonForPushOrPull`
2. **Force-push 分层策略**：
   - 已知落后 → `--force-with-lease`（安全）
   - 未知（远程分支未缓存）→ `--force` 弹框二次确认
3. **GIT_SEQUENCE_EDITOR 绕过**：防止交互式 rebase 配置导致 pull 挂起
4. **锁文件自动重试**：解决 `.git/index.lock` 瞬时冲突问题
5. **后台 fetch 凭证策略**：`FAIL` 策略（发换行）防止无提示挂起
6. **刷新并发设计**：
   - branchesAndRemotesWg / fileWg 等依赖管理
   - 各 scope 独立 goroutine + 各自 OnUIThread 渲染，避免大锁阻塞
   - BehindBaseBranch 异步渐进加载，先显示基本信息再补充基分支差距
7. **冲突解决闭环**：refreshStateFiles 自动 stage 已解决冲突 + 冲突消失自动提示 Continue Rebase

### 6.2 易错点 / 注意事项（基于代码事实修正）

1. **错误消息语言依赖**：冲突检测、落后提示基于英文错误字符串（如 "Updates were rejected"），非英文 Git 环境可能失效
2. **远程分支信息缺失**：`AheadForPull == "?"` 时无法提前判断是否需要 force-push，只能先尝试普通 push
3. **ASYNC 和 SYNC 都是并行执行 scope**，区别在于 `Refresh()` 是否等待：ASYNC 下 `wg` 为空，`wg.Wait()` 立即返回，`Refresh()` fire-and-forget；SYNC 下 `wg` 跟踪所有 scope，`wg.Wait()` 阻塞到全部完成
4. **ASYNC + Then 会 panic**：因为 ASYNC 不参与 wg，`wg.Wait()` 不等任何 scope，Then 无法在所有 scope 完成后执行，代码直接 panic
5. **状态栏有两个刷新入口**，ASYNC 模式下 Refresh 主流程末尾的 refreshStatus() 大概率因为 branches 还没完成而 early return，真正更新在 refreshBranches() 内部
6. **MERGE_CONFLICTS scope 和 FILES scope 是并行的**：不保证谁先完成，RefreshMergeState 靠"当前上下文是不是 MergeConflicts"判断要不要干活，不依赖 fileWg
7. **BehindBaseBranch 独立二次渲染**：在 BranchLoader.Load 内部通过 `onWorker` 异步提交，排在所有 Refresh scope 之后，是独立的 OnWorker task，不受 Refresh wg 控制
8. **CheckMergeOrRebase 默认用 ASYNC**，冲突检测靠 git 命令返回的错误字符串（第一层），不是靠 Model.Files；Model.Files 上的冲突自动处理是第二层（持续刷新时触发）
9. **Refresh 时序**：push/pull 成功后的渲染是分多批次出现的
   - 第一批：commits 面板
   - 第二批：branches 面板 + 状态栏（显示 ✓，BehindBaseBranch 仍为旧值）
   - 第三批：files、remotes 等其他面板
   - 第四批：BehindBaseBranch 计算完 → branches 面板再次渲染（显示 ↓N）
10. **Demo 模式特殊处理**：InlineStatus stop 时会额外渲染，因为 demo 中 ASYNC 被强制降级为 SYNC
11. **线程安全**：
    - `itemOperations` map 有独立 mutex 保护
    - `Model.Branches` / `Model.Files` 等在 Worker 线程写、UI 线程读，靠整体结构替换（非原地修改）保证可见性
12. **过滤状态丢失**：每次 `ReApplyFilter(context)` 会重新创建过滤列表，如果用户在搜索中刷新，搜索高亮会在 `AfterLayout` 的 `ReApplySearch` 才恢复
