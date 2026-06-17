# Commit 编辑与 Amend 实现思路

本文按代码执行顺序拆解 lazygit 中 commit 消息编辑、参数组织（git 命令构建）以及完成后的状态刷新三大流程。

---

## 一、消息编辑流程

消息编辑涉及三种主要操作场景：**Reword（重命名提交）**、**Amend（修改提交）**、以及**CreateAmendCommit（创建 amend! 提交）**。它们都共用同一套消息面板机制。

### 1.1 入口：触发编辑操作

所有操作都从 [local_commits_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go) 中的按键绑定开始：

- **Reword（面板内编辑）**：调用 `reword(commit)` [L424-L449](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go#L424-L449)
- **Reword（外部编辑器）**：调用 `rewordEditor(commit)` [L516-L523](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go#L516-L523)
- **Amend**：调用 `amendTo(commit)` [L813-L844](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go#L813-L844)
- **CreateAmendCommit**：调用 `createAmendCommit(commit, includeFileChanges)` [L1090-L1127](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go#L1090-L1127)

### 1.2 获取原始提交消息

以 `reword` 为例，第一步通过 git 命令获取已有消息：

```go
// local_commits_controller.go L429
commitMessage, err := self.c.Git().Commit.GetCommitMessage(commit.Hash())
```

实现在 [commit.go:157-165](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L157-L165)：

```go
func (self *CommitCommands) GetCommitMessage(commitHash string) (string, error) {
    cmdArgs := NewGitCmd("log").
        Arg("--format=%B", "--max-count=1", commitHash).
        Config("log.showsignature=false").
        ToArgv()
    message, err := self.cmd.New(cmdArgs).DontLog().RunWithOutput()
    return strings.ReplaceAll(strings.TrimSpace(message), "\r\n", "\n"), err
}
```

执行的 git 命令：`git log --format=%B --max-count=1 <hash>`

### 1.3 自动换行预处理

如果用户配置了 `Git.Commit.AutoWrapCommitMessage`，会调用 [commits_helper.go:83-103](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/commits_helper.go#L83-L103) 中的 `TryRemoveHardLineBreaks` 把硬换行转成软换行（空格），让消息在编辑面板内自然流动。

### 1.4 打开 Commit Message Panel

通过 `CommitsHelper.OpenCommitMessagePanel` [commits_helper.go:138-176](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/commits_helper.go#L138-L176) 打开编辑面板。核心步骤：

1. **包装确认回调**：用户确认时先关闭面板，再调用业务层 `OnConfirm`
2. **消息保留（PreserveMessage）**：如果开启了消息保留且没有初始消息，从保留文件中恢复
3. **设置面板状态**：调用 `CommitMessageContext.SetPanelState` 传入：
   - `CommitIndex`：提交索引
   - `SummaryTitle` / `DescriptionTitle`：面板标题
   - `PreserveMessage`：是否保留消息
   - `initialMessage`：初始消息文本
   - `onConfirm`：确认回调函数
   - `OnSwitchToEditor`：切换到外部编辑器的回调
4. **填充视图**：`SetMessageAndDescriptionInView` 将消息拆分为 summary（第一行）和 description（其余部分）分别填入两个视图
5. **推入上下文栈**：`Context().Push(CommitMessageContext)`

### 1.5 消息拆分与合并逻辑

在 [commits_helper.go:43-81](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/commits_helper.go#L43-L81)：

- **`SplitCommitMessageAndDescription`**：按第一个 `\n` 切割，前面是 summary，后面去掉开头的 `\n` 是 description
- **`JoinCommitMessageAndUnwrappedDescription`**：如果 description 为空只返回 summary，否则用 `\n` 连接

### 1.6 面板内确认提交流程

用户按提交键触发 `CommitMessageController.confirm()` [commit_message_controller.go:176-192](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/commit_message_controller.go#L176-L192)：

1. 特殊处理：如果正在粘贴文本且回车是默认提交键，则跳到 description 面板而非提交
2. 调用 `CommitsHelper.HandleCommitConfirm()` [commits_helper.go:182-195](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/commits_helper.go#L182-L195)：
   - 校验 summary 非空
   - 获取当前 summary 和 description
   - 调用之前注册的 `OnConfirm` 回调执行业务逻辑

### 1.7 切换到外部编辑器

通过 `CommitsHelper.SwitchToEditor()` [commits_helper.go:105-118](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/commits_helper.go#L105-L118)：

1. 拼接 summary + description 为完整消息
2. 写入临时文件：`{tempDir}/{repoName}/{timestamp}.msg`
3. 关闭面板
4. 调用 `CommitMessageContext.SwitchToEditor(filepath)` 触发业务层注册的 `OnSwitchToEditor` 回调

---

## 二、参数组织（Git 命令构建）

根据操作类型和目标提交位置（HEAD 或非 HEAD），走不同的 git 命令构建路径。

### 2.1 Reword Last Commit（HEAD 提交重命名）

#### 面板内编辑

入口在 `handleReword` [local_commits_controller.go:478-L494](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go#L478-L494)，是 HEAD 提交时：

```go
return self.c.Helpers().GPG.WithGpgHandling(
    self.c.Git().Commit.RewordLastCommit(summary, description),
    git_commands.CommitGpgSign,
    self.c.Tr.RewordingStatus, nil, nil)
```

`RewordLastCommit` 在 [commit.go:120-129](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L120-L129)：

```go
func (self *CommitCommands) RewordLastCommit(summary string, description string) *oscommands.CmdObj {
    messageArgs := self.commitMessageArgs(summary, description)
    cmdArgs := NewGitCmd("commit").
        Arg("--allow-empty", "--amend", "--only").
        Arg(messageArgs...).
        ToArgv()
    return self.cmd.New(cmdArgs)
}
```

`commitMessageArgs` [commit.go:131-139](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L131-L139) 负责把 summary/description 转成 `-m` 参数：
- summary → 第一个 `-m`
- description 非空 → 追加第二个 `-m`

最终命令形如：`git commit --allow-empty --amend --only -m "summary" -m "description"`

#### 外部编辑器编辑

`RewordLastCommitInEditorCmdObj` [commit.go:101-103](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L101-L103)：

```go
return self.cmd.New(NewGitCmd("commit").Arg("--allow-empty", "--amend", "--only").ToArgv())
```

命令：`git commit --allow-empty --amend --only`（不加 `-m`，git 会启动编辑器）

如果是从面板切换到编辑器（有临时消息文件），用 `RewordLastCommitInEditorWithMessageFileCmdObj` [commit.go:105-108](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L105-L108)：

```go
return self.cmd.New(NewGitCmd("commit").
    Arg("--allow-empty", "--amend", "--only", "--edit", "--file="+tmpMessageFile).ToArgv())
```

命令：`git commit --allow-empty --amend --only --edit --file=<tmpfile>`

### 2.2 Reword 非 HEAD 提交

需要通过交互式 rebase 实现。入口同样是 `handleReword`，但走 else 分支：

```go
// local_commits_controller.go L486-L493
return self.c.WithWaitingStatus(self.c.Tr.RewordingStatus, func(gocui.Task) error {
    err := self.c.Git().Rebase.RewordCommit(
        self.c.Model().Commits,
        self.c.Contexts().LocalCommits.GetSelectedLineIdx(),
        summary, description)
    if err != nil {
        return err
    }
    self.c.Refresh(types.RefreshOptions{Mode: types.ASYNC})
    return nil
})
```

`RebaseCommands.RewordCommit` 在 [rebase.go:37-56](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/rebase.go#L37-L56)：

```go
func (self *RebaseCommands) RewordCommit(commits []*models.Commit, index int, summary string, description string) error {
    // 1. 启动交互式 rebase，停在目标 commit（将其设为 edit 动作）
    err := self.BeginInteractiveRebaseForCommit(commits, index, false)
    if err != nil { return err }
    // 2. 此时目标 commit 已成为 HEAD，直接 amend 消息
    err = self.commit.RewordLastCommit(summary, description).Run()
    if err != nil { return err }
    // 3. 继续 rebase 完成后续提交
    return self.ContinueRebase()
}
```

如果使用外部编辑器 reword 非 HEAD 提交，走 `RewordCommitInEditor` [rebase.go:58-69](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/rebase.go#L58-L69)：

```go
changes := []daemon.ChangeTodoAction{{
    Hash:      commits[index].Hash(),
    NewAction: todo.Reword,  // 直接设为 reword 动作，git 会自动调编辑器
}}
return self.PrepareInteractiveRebaseCommand(PrepareInteractiveRebaseCommandOpts{
    baseHashOrRoot: getBaseHashOrRoot(commits, index+1),
    instruction:    daemon.NewChangeTodoActionsInstruction(changes),
}), nil
```

### 2.3 Amend HEAD 提交

`amendTo` [local_commits_controller.go:816-825](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go#L816-L825) 判断是 HEAD 提交时：

```go
handleCommit = func() error {
    return self.c.Helpers().WorkingTree.WithEnsureCommittableFiles(func() error {
        if err := self.c.Helpers().AmendHelper.AmendHead(); err != nil {
            return err
        }
        self.c.Refresh(types.RefreshOptions{Mode: types.ASYNC})
        return nil
    })
}
```

`AmendHelper.AmendHead` [amend_helper.go:20-23](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/amend_helper.go#L20-L23)：

```go
func (self *AmendHelper) AmendHead() error {
    cmdObj := self.c.Git().Commit.AmendHeadCmdObj()
    self.c.LogAction(self.c.Tr.Actions.AmendCommit)
    return self.gpg.WithGpgHandling(cmdObj, git_commands.CommitGpgSign, self.c.Tr.AmendingStatus, nil, nil)
}
```

`AmendHeadCmdObj` [commit.go:235-241](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L235-L241)：

```go
cmdArgs := NewGitCmd("commit").
    Arg("--amend", "--no-edit", "--allow-empty", "--allow-empty-message").
    ToArgv()
```

命令：`git commit --amend --no-edit --allow-empty --allow-empty-message`

关键点：使用 `--no-edit`，不打开编辑器，直接用暂存区内容 amend 上一次提交，保留原消息。

### 2.4 Amend 非 HEAD 提交

`amendTo` 的 else 分支 [local_commits_controller.go:826-835](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go#L826-L835)：

```go
handleCommit = func() error {
    return self.c.Helpers().WorkingTree.WithEnsureCommittableFiles(func() error {
        return self.c.WithWaitingStatus(self.c.Tr.AmendingStatus, func(gocui.Task) error {
            self.c.LogAction(self.c.Tr.Actions.AmendCommit)
            err := self.c.Git().Rebase.AmendTo(
                self.c.Model().Commits,
                self.context().GetView().SelectedLineIdx())
            return self.c.Helpers().MergeAndRebase.CheckMergeOrRebase(err)
        })
    })
}
```

`RebaseCommands.AmendTo` [rebase.go:298-315](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/rebase.go#L298-L315)：

```go
func (self *RebaseCommands) AmendTo(commits []*models.Commit, commitIndex int) error {
    commit := commits[commitIndex]
    // 1. 先用暂存区内容创建一个 fixup! 提交
    if err := self.commit.CreateFixupCommit(commit.Hash()); err != nil {
        return err
    }
    // 2. 获取刚创建的 fixup commit 的 hash
    fixupHash, err := self.getHashOfLastCommitMade()
    if err != nil { return err }
    // 3. 启动交互式 rebase，自动把 fixup commit 移到目标 commit 下方并 autosquash
    return self.PrepareInteractiveRebaseCommand(PrepareInteractiveRebaseCommandOpts{
        baseHashOrRoot: getBaseHashOrRoot(commits, commitIndex+1),
        overrideEditor: true,
        instruction:    daemon.NewMoveFixupCommitDownInstruction(commit.Hash(), fixupHash, true),
    }).Run()
}
```

`CreateFixupCommit` [commit.go:290-294](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L290-L294)：

```go
cmdArgs := NewGitCmd("commit").Arg("--fixup=" + hash).ToArgv()
```

命令：`git commit --fixup=<targetHash>`

### 2.5 CreateAmendCommit（创建 amend! 提交）

不同于 AmendTo 直接合并，这个操作创建一个独立的 `amend!` 提交，后续通过 autosquash 合并。

`CreateAmendCommit` [commit.go:297-309](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L297-L309)：

```go
func (self *CommitCommands) CreateAmendCommit(originalSubject, newSubject, newDescription string, includeFileChanges bool) error {
    description := newSubject
    if newDescription != "" {
        description += "\n\n" + newDescription
    }
    cmdArgs := NewGitCmd("commit").
        Arg("-m", "amend! "+originalSubject).  // 第一行：amend! + 原提交标题
        Arg("-m", description).                  // 第二行：新消息内容
        ArgIf(!includeFileChanges, "--only", "--allow-empty").
        ToArgv()
    return self.cmd.New(cmdArgs).Run()
}
```

如果不包含文件变更（`includeFileChanges=false`），加上 `--only --allow-empty` 创建空提交。

### 2.6 Amend Commit 属性（作者、Co-author）

通过 `GenericAmend` [rebase.go:89-112](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/rebase.go#L89-L112) 统一处理：

```go
func (self *RebaseCommands) GenericAmend(commits []*models.Commit, start, end int, f func(commit *models.Commit) error) error {
    // HEAD 提交直接执行，无需 rebase
    if start == end && models.IsHeadCommit(commits, start) {
        return f(commits[start])
    }
    // 非 HEAD：启动交互式 rebase 停在该 commit
    err := self.BeginInteractiveRebaseForCommitRange(commits, start, end, false)
    // 倒序遍历每个 commit，执行 amend 操作后继续 rebase
    for commitIndex := end; commitIndex >= start; commitIndex-- {
        err = f(commits[commitIndex])
        if err := self.ContinueRebase(); err != nil { return err }
    }
    return nil
}
```

具体属性修改的命令：

- **ResetAuthor** [commit.go:24-30](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L24-L30)：
  `git commit --allow-empty --allow-empty-message --only --no-edit --amend --reset-author`

- **SetAuthor** [commit.go:33-39](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L33-L39)：
  `git commit --allow-empty --allow-empty-message --only --no-edit --amend --author="Name <Email>"`

- **AddCoAuthor** [commit.go:42-55](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go#L42-L55)：
  先获取原消息 → 在末尾追加 `Co-authored-by:` 行 → 执行 `git commit --allow-empty --amend --only -m <newMessage>`

### 2.7 GPG 签名处理封装

`GpgHelper.WithGpgHandling` [gpg_helper.go:26-41](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/gpg_helper.go#L26-L41) 根据配置决定执行方式：

- **需要 GPG 子进程**（如需要输入密码）：用 `RunSubprocess` 挂起 lazygit 让 git 与终端交互，成功后调 `onSuccess` 回调，最后 `Refresh(ASYNC)`
- **不需要**：用 `runAndStream` 包装在 `WithWaitingStatus` 中，流式输出命令日志，成功后同样 `Refresh(ASYNC)`

---

## 三、完成后的状态刷新

### 3.1 刷新触发的几种方式

根据上下文不同，刷新有以下几种触发入口：

| 场景 | 触发方式 | 代码位置 |
|------|---------|---------|
| Amend HEAD 完成 | 直接调用 `self.c.Refresh(ASYNC)` | local_commits_controller.go L822 |
| Reword 非 HEAD 完成 | `self.c.Refresh(ASYNC)` | local_commits_controller.go L491 |
| 通过 MergeAndRebaseHelper | `CheckMergeOrRebase` → `Refresh(ASYNC)` | merge_and_rebase_helper.go L168-L170 |
| 通过 GpgHelper | 内部统一调用 `Refresh(ASYNC)` | gpg_helper.go L35, L46, L58 |
| 通过 RunSubprocessAndRefresh | 子进程结束后自动刷新 | gui_common.go L37-L39 |
| 更新 rebase TODO 文件 | `Refresh(SYNC, Scope: [REBASE_COMMITS])` | local_commits_controller.go L727-L729 |

### 3.2 Refresh 核心流程

`RefreshHelper.Refresh` [refresh_helper.go:63-237](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L63-L237) 是刷新的总入口：

#### 步骤 1：确定刷新范围（Scope）

如果没传 Scope，默认刷新：
`COMMITS, BRANCHES, FILES, STASH, REFLOG, TAGS, REMOTES, WORKTREES, STATUS, BISECT_INFO, STAGING, PULL_REQUESTS`

如果传了 Scope（如只刷新 `REBASE_COMMITS`），则只刷新指定项。

#### 步骤 2：根据模式决定执行方式

- **ASYNC**：在 worker goroutine 中执行（`c.OnWorker`），不阻塞 UI
- **SYNC**：在 goroutine 中执行但通过 WaitGroup 等待全部完成
- **BLOCK_UI**：在 UI 线程同步执行（会冻结界面）

#### 步骤 3：并行执行各 Scope 的刷新

通过 `sync.WaitGroup` 并发刷新：

```go
refresh := func(name string, f func()) {
    if !self.c.InDemo() && options.Mode == types.ASYNC {
        self.c.OnWorker(func(t gocui.Task) error { f(); return nil })
    } else {
        wg.Add(1)
        go utils.Safe(func() {
            defer wg.Done()
            f()
        })
    }
}
```

关键刷新任务之间的依赖关系：
- COMMITS 与 BRANCHES 互相依赖，放在一起刷新
- FILES 刷新完成后才刷新 STAGING
- BRANCHES + REMOTES 刷新完成后才刷新 PULL_REQUESTS
- MERGE_CONFLICTS 随 FILES 一起刷新

#### 步骤 4：Commits 刷新细节

`refreshCommitsAndCommitFiles` [refresh_helper.go:306-323](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L306-L323)：

1. 调用 `refreshCommitsWithLimit` 重新加载提交列表
2. 如果当前在查看某个 commit 的文件列表，同步刷新 commit files context

`refreshCommitsWithLimit` [refresh_helper.go:351-383](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L351-L383)：

```go
func (self *RefreshHelper) refreshCommitsWithLimit() error {
    self.c.Mutexes().LocalCommitsMutex.Lock()
    defer self.c.Mutexes().LocalCommitsMutex.Unlock()

    checkedOutRef := self.determineCheckedOutRef()
    commits, err := self.c.Git().Loaders.CommitLoader.GetCommits(...)
    self.c.Model().Commits = commits
    self.RefreshAuthors(commits)
    self.c.Model().WorkingTreeStateAtLastCommitRefresh = self.c.Git().Status.WorkingTreeState()
    self.c.Model().CheckedOutBranch = checkedOutRef.RefName()
    self.refreshView(self.c.Contexts().LocalCommits)
    return nil
}
```

要点：
- 加 `LocalCommitsMutex` 锁保护并发访问
- 通过 `determineCheckedOutRef` 正确处理 rebase/bisect 中的 detached HEAD
- 刷新作者缓存
- 更新工作树状态（是否在 rebase/merge 中）

#### 步骤 5：Rebase Commits 单独刷新

如果 Scope 只有 `REBASE_COMMITS`（如修改 TODO 文件后），走 `refreshRebaseCommits` [refresh_helper.go:446-459](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L446-L459)：

```go
updatedCommits, err := self.c.Git().Loaders.CommitLoader.MergeRebasingCommits(
    self.c.Model().HashPool, self.c.Model().Commits)
self.c.Model().Commits = updatedCommits
self.c.Model().WorkingTreeStateAtLastCommitRefresh = self.c.Git().Status.WorkingTreeState()
self.refreshView(self.c.Contexts().LocalCommits)
```

只合并 rebase TODO 信息，不重新加载全部提交，性能更好。

#### 步骤 6：视图更新（refreshView）

`refreshView` [refresh_helper.go:785-809](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L785-L809) 将数据同步到 UI：

```go
func (self *RefreshHelper) refreshView(context types.Context) {
    self.c.OnUIThread(func() error {
        self.searchHelper.ReApplyFilter(context)   // 重应用过滤器
        self.c.PostRefreshUpdate(context)          // 更新视图内容
        self.c.AfterLayout(func() error {
            self.searchHelper.ReApplySearch(context) // 重应用搜索高亮
            return nil
        })
        return nil
    })
}
```

所有 UI 更新必须通过 `OnUIThread` 切到主线程执行，保证线程安全。

#### 步骤 7：执行 Then 回调

如果 `RefreshOptions.Then` 不为空，所有刷新完成后在当前线程执行该回调（仅支持 SYNC 和 BLOCK_UI 模式）。

### 3.3 Merge/Rebase 结果检查

`MergeAndRebaseHelper.CheckMergeOrRebase` [merge_and_rebase_helper.go:152-170](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L152-L170) 在刷新后处理特殊情况：

```go
func (self *MergeAndRebaseHelper) CheckMergeOrRebaseWithRefreshOptions(result error, refreshOptions types.RefreshOptions) error {
    self.c.Refresh(refreshOptions)
    if result == nil { return nil }
    // "No changes" → 自动 skip 该 commit
    if strings.Contains(result.Error(), "No changes - did you forget to use") {
        return self.genericMergeCommand(REBASE_OPTION_SKIP)
    }
    // "cherry-pick is now empty" → 自动 skip
    if strings.Contains(result.Error(), "The previous cherry-pick is now empty") {
        return self.genericMergeCommand(REBASE_OPTION_SKIP)
    }
    // "No rebase in progress" → 视为成功
    if strings.Contains(result.Error(), "No rebase in progress?") {
        return nil
    }
    // 冲突 → 弹出冲突处理菜单
    return self.CheckForConflicts(result)
}
```

检测到冲突时调用 `PromptForConflictHandling` 弹出菜单，让用户选择查看冲突文件或 abort 操作。

---

## 四、完整执行链路示例

### 场景 A：Reword HEAD 提交（面板内编辑）

```
用户按 'r' 键
  → LocalCommitsController.reword(commit)
    → Commit.GetCommitMessage(hash)                 // git log --format=%B
    → CommitsHelper.OpenCommitMessagePanel(opts)    // 打开编辑面板
      → CommitMessageContext.SetPanelState(...)
      → SetMessageAndDescriptionInView(message)     // 拆分 summary/description
      → Context.Push(CommitMessageContext)          // 切换到编辑上下文

用户编辑后按 Enter
  → CommitMessageController.confirm()
    → CommitsHelper.HandleCommitConfirm()
      → 校验 summary 非空
      → 调用 OnConfirm 回调 = LocalCommitsController.handleReword

handleReword（HEAD 分支）
  → GpgHelper.WithGpgHandling(RewordLastCommitCmdObj)
    → cmdObj.StreamOutput().Run()                   // git commit --amend --only -m ...
    → Refresh(ASYNC)
      → RefreshHelper.Refresh
        → refreshCommitsAndCommitFiles
        → refreshBranches / refreshFiles / ...
        → refreshView → PostRefreshUpdate
```

### 场景 B：Amend 非 HEAD 提交

```
用户按 'A' 键
  → LocalCommitsController.amendTo(commit)
    → Confirm 确认对话框
    → WorkingTree.WithEnsureCommittableFiles        // 确保有可提交文件
    → WithWaitingStatus("Amending")
      → RebaseCommands.AmendTo(commits, index)
        → Commit.CreateFixupCommit(targetHash)      // git commit --fixup=<hash>
        → getHashOfLastCommitMade()
        → PrepareInteractiveRebaseCommand + MoveFixupCommitDown
          // 启动 rebase，自动移动 fixup commit 并 autosquash
      → MergeAndRebaseHelper.CheckMergeOrRebase(err)
        → Refresh(ASYNC)
        → 检查冲突 / 空提交等特殊情况
```

---

## 五、核心文件索引

| 文件 | 职责 |
|------|------|
| [local_commits_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/local_commits_controller.go) | 按键绑定、操作入口、HEAD/非 HEAD 分支判断 |
| [commit_message_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/commit_message_controller.go) | 消息面板的键盘/鼠标事件处理 |
| [commits_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/commits_helper.go) | 消息面板打开/关闭、消息拆分合并、编辑器切换 |
| [amend_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/amend_helper.go) | Amend HEAD 的 GPG 封装 |
| [gpg_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/gpg_helper.go) | GPG 签名分支处理（子进程 vs 流式输出） |
| [merge_and_rebase_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go) | rebase/merge 结果检查、冲突处理、continue/abort/skip |
| [refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/gui/controllers/helpers/refresh_helper.go) | 统一刷新入口、并行刷新各 Scope、视图更新 |
| [commit.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/commit.go) | 构建 commit/amend/reword/fixup 的 git 命令参数 |
| [rebase.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-lazygit/pkg/commands/git_commands/rebase.go) | 非 HEAD 提交修改：交互式 rebase 编排、GenericAmend、AmendTo |
