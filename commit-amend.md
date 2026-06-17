# Commit 编辑与 Amend 实现思路

本文按代码执行顺序拆解 lazygit 中 commit 消息编辑、参数组织（git 命令构建）以及完成后的状态刷新三大流程。

---

## 一、消息编辑流程

涉及消息编辑的操作场景有三种：**Reword（重命名提交）**和**CreateAmendCommit（创建 amend! 提交）**共用同一套消息面板机制；而**Amend（修改提交）**不经过消息编辑面板，用户确认后直接保留原消息并合并暂存区内容。

### 1.1 入口：触发编辑操作

所有操作都从 [local_commits_controller.go](pkg/gui/controllers/local_commits_controller.go) 中的按键绑定开始：

- **Reword（面板内编辑）**：调用 `reword(commit)` [L424-L449](pkg/gui/controllers/local_commits_controller.go#L424-L449)
- **Reword（外部编辑器）**：调用 `rewordEditor(commit)` [L516-L523](pkg/gui/controllers/local_commits_controller.go#L516-L523)
- **Amend**：调用 `amendTo(commit)` [L813-L844](pkg/gui/controllers/local_commits_controller.go#L813-L844)
- **CreateFixupCommit / CreateAmendCommit**：调用 `createFixupCommit(commit)` [L989-L1044](pkg/gui/controllers/local_commits_controller.go#L989-L1044)，弹出菜单后根据选项调用 `createAmendCommit(commit, includeFileChanges)` [L1090-L1127](pkg/gui/controllers/local_commits_controller.go#L1090-L1127)

### 1.2 获取原始提交消息

以 `reword` 为例，第一步通过 git 命令获取已有消息：

```go
// local_commits_controller.go L429
commitMessage, err := self.c.Git().Commit.GetCommitMessage(commit.Hash())
```

实现在 [commit.go:157-165](pkg/commands/git_commands/commit.go#L157-L165)：

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

`createAmendCommit` 同样在打开面板前调用 `GetCommitMessage` 获取原提交消息，另外还保存原始 subject（`originalSubject`）用于 amend! 头的生成。

### 1.3 自动换行预处理

如果用户配置了 `Git.Commit.AutoWrapCommitMessage`，会调用 [commits_helper.go:83-103](pkg/gui/controllers/helpers/commits_helper.go#L83-L103) 中的 `TryRemoveHardLineBreaks` 把硬换行转成软换行（空格），让消息在编辑面板内自然流动。

### 1.4 打开 Commit Message Panel

通过 `CommitsHelper.OpenCommitMessagePanel` [commits_helper.go:138-176](pkg/gui/controllers/helpers/commits_helper.go#L138-L176) 打开编辑面板。核心步骤：

1. **包装确认回调**：用户确认时先关闭面板，再调用业务层 `OnConfirm`
2. **消息保留（PreserveMessage）**：如果开启了消息保留且没有初始消息，从保留文件中恢复
3. **设置面板状态**：调用 `CommitMessageContext.SetPanelState` 传入：
   - `CommitIndex`：提交索引
   - `SummaryTitle` / `DescriptionTitle`：面板标题
   - `PreserveMessage`：是否保留消息（CreateAmendCommit / Reword 传 false，普通 commit 传 true）
   - `initialMessage`：初始消息文本
   - `onConfirm`：确认回调函数
   - `OnSwitchToEditor`：切换到外部编辑器的回调（CreateAmendCommit 传 nil，不支持切换）
4. **填充视图**：`SetMessageAndDescriptionInView` 将消息拆分为 summary（第一行）和 description（其余部分）分别填入两个视图
5. **推入上下文栈**：`Context().Push(CommitMessageContext)`

### 1.5 消息拆分与合并逻辑

在 [commits_helper.go:43-81](pkg/gui/controllers/helpers/commits_helper.go#L43-L81)：

- **`SplitCommitMessageAndDescription`**：按第一个 `\n` 切割，前面是 summary，后面去掉开头的 `\n` 是 description
- **`JoinCommitMessageAndUnwrappedDescription`**：如果 description 为空只返回 summary，否则用 `\n` 连接

### 1.6 面板内确认提交流程

用户按提交键触发 `CommitMessageController.confirm()` [commit_message_controller.go:176-192](pkg/gui/controllers/commit_message_controller.go#L176-L192)：

1. 特殊处理：如果正在粘贴文本且回车是默认提交键，则跳到 description 面板而非提交
2. 调用 `CommitsHelper.HandleCommitConfirm()` [commits_helper.go:182-195](pkg/gui/controllers/helpers/commits_helper.go#L182-L195)：
   - 校验 summary 非空
   - 获取当前 summary 和 description
   - 调用之前注册的 `OnConfirm` 回调执行业务逻辑

#### 1.6.1 Reword 的确认回调（handleReword）

`handleReword` [local_commits_controller.go:478-L494](pkg/gui/controllers/local_commits_controller.go#L478-L494) 根据是否是 HEAD 走不同分支：
- HEAD → 用 GpgHelper 封装执行 `RewordLastCommit`
- 非 HEAD → `WithWaitingStatus` 中调用 `Rebase.RewordCommit` 完成后 `Refresh(ASYNC)`

#### 1.6.2 Amend 的确认流程（不走消息面板）

普通 Amend 操作（`amendTo`）不打开消息编辑面板，而是弹出确认对话框 [local_commits_controller.go:838-L843](pkg/gui/controllers/local_commits_controller.go#L838-L843)：

```go
return self.c.ConfirmIf(!self.c.UserConfig().Gui.SkipAmendWarning,
    types.ConfirmOpts{
        Title:         self.c.Tr.AmendCommitTitle,
        Prompt:        self.c.Tr.AmendCommitPrompt,
        HandleConfirm: handleCommit,
    })
```

用户确认后直接执行 `handleCommit`，保留原提交消息不变（`--no-edit`），只将暂存区内容合并到目标提交。具体路径见第二章 2.3 和 2.4 节。

#### 1.6.3 CreateAmendCommit 的确认回调（OnConfirm 闭包）

`createAmendCommit` 的 OnConfirm 在 [local_commits_controller.go:1106-L1121](pkg/gui/controllers/local_commits_controller.go#L1106-L1121) 内联定义，执行步骤：

```
CreateAmendCommit 的 OnConfirm(summary, description):
  1. 调用 Commit.CreateAmendCommit(originalSubject, summary, description, includeFileChanges)
     → 执行 git commit -m "amend! ..." -m ...
  2. 调用 moveFixupCommitToOwnerStackedBranch(commit)
     → 前置条件检查（Git 版本、rebase 状态、merged 状态、rebase.updateRefs 配置）
     → 找到目标 commit 所在 stacked branch 的 head
     → 调用 Rebase.MoveFixupCommitDown 将新创建的 amend! 提交移到 branch head 上方
  3. self.context().MoveSelectedLine(1)
     → 选择下移一行（回到原选中的 target commit，因为 amend! 被挪走了）
  4. Refresh(SYNC)
     → 同步模式刷新，确保所有 UI 状态与磁盘一致
```

对应的 Fixup 菜单项（非 Amend 模式）流程相同，只是调用 `Commit.CreateFixupCommit`，也同样执行 `moveFixupCommitToOwnerStackedBranch` + `MoveSelectedLine(1)` + `Refresh(SYNC)` [local_commits_controller.go:1007-L1018](pkg/gui/controllers/local_commits_controller.go#L1007-L1018)。

### 1.7 切换到外部编辑器

通过 `CommitsHelper.SwitchToEditor()` [commits_helper.go:105-118](pkg/gui/controllers/helpers/commits_helper.go#L105-L118)：

1. 拼接 summary + description 为完整消息
2. 写入临时文件：`{tempDir}/{repoName}/{timestamp}.msg`
3. 关闭面板
4. 调用 `CommitMessageContext.SwitchToEditor(filepath)` 触发业务层注册的 `OnSwitchToEditor` 回调

Reword 的 `OnSwitchToEditor` = `switchFromCommitMessagePanelToEditor` [local_commits_controller.go:451-L476](pkg/gui/controllers/local_commits_controller.go#L451-L476)：
- HEAD → `RunSubprocessAndRefresh(RewordLastCommitInEditorWithMessageFileCmdObj(filepath))`
- 非 HEAD → BeginInteractiveRebaseForCommit → RunSubprocessAndRefresh amend → ContinueRebase → Refresh(ASYNC)

---

## 二、参数组织（Git 命令构建）

根据操作类型和目标提交位置（HEAD 或非 HEAD），走不同的 git 命令构建路径。

### 2.1 Reword Last Commit（HEAD 提交重命名）

#### 面板内编辑

入口在 `handleReword` [local_commits_controller.go:478-L494](pkg/gui/controllers/local_commits_controller.go#L478-L494)，是 HEAD 提交时：

```go
return self.c.Helpers().GPG.WithGpgHandling(
    self.c.Git().Commit.RewordLastCommit(summary, description),
    git_commands.CommitGpgSign,
    self.c.Tr.RewordingStatus, nil, nil)
```

`RewordLastCommit` 在 [commit.go:120-129](pkg/commands/git_commands/commit.go#L120-L129)：

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

`commitMessageArgs` [commit.go:131-139](pkg/commands/git_commands/commit.go#L131-L139) 负责把 summary/description 转成 `-m` 参数：
- summary → 第一个 `-m`
- description 非空 → 追加第二个 `-m`

最终命令形如：`git commit --allow-empty --amend --only -m "summary" -m "description"`

#### 外部编辑器编辑

`RewordLastCommitInEditorCmdObj` [commit.go:101-103](pkg/commands/git_commands/commit.go#L101-L103)：

```go
return self.cmd.New(NewGitCmd("commit").Arg("--allow-empty", "--amend", "--only").ToArgv())
```

命令：`git commit --allow-empty --amend --only`（不加 `-m`，git 会启动编辑器）

如果是从面板切换到编辑器（有临时消息文件），用 `RewordLastCommitInEditorWithMessageFileCmdObj` [commit.go:105-108](pkg/commands/git_commands/commit.go#L105-L108)：

```go
return self.cmd.New(NewGitCmd("commit").
    Arg("--allow-empty", "--amend", "--only", "--edit", "--file="+tmpMessageFile).ToArgv())
```

命令：`git commit --allow-empty --amend --only --edit --file=<tmpfile>`

### 2.2 Reword 非 HEAD 提交

需要通过交互式 rebase 实现。入口同样是 `handleReword`，但走 else 分支 [local_commits_controller.go:486-L493](pkg/gui/controllers/local_commits_controller.go#L486-L493)：

```go
return self.c.WithWaitingStatus(self.c.Tr.RewordingStatus, func(gocui.Task) error {
    err := self.c.Git().Rebase.RewordCommit(
        self.c.Model().Commits,
        self.c.Contexts().LocalCommits.GetSelectedLineIdx(),
        summary, description)
    if err != nil { return err }
    self.c.Refresh(types.RefreshOptions{Mode: types.ASYNC})
    return nil
})
```

`RebaseCommands.RewordCommit` 在 [rebase.go:37-56](pkg/commands/git_commands/rebase.go#L37-L56)：

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

如果使用外部编辑器 reword 非 HEAD 提交，走 `RewordCommitInEditor` [rebase.go:58-69](pkg/commands/git_commands/rebase.go#L58-L69)：

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

`amendTo` [local_commits_controller.go:816-L825](pkg/gui/controllers/local_commits_controller.go#L816-L825) 判断是 HEAD 提交时：

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

`AmendHelper.AmendHead` [amend_helper.go:20-23](pkg/gui/controllers/helpers/amend_helper.go#L20-L23)：

```go
func (self *AmendHelper) AmendHead() error {
    cmdObj := self.c.Git().Commit.AmendHeadCmdObj()
    self.c.LogAction(self.c.Tr.Actions.AmendCommit)
    return self.gpg.WithGpgHandling(cmdObj, git_commands.CommitGpgSign, self.c.Tr.AmendingStatus, nil, nil)
}
```

`AmendHeadCmdObj` [commit.go:235-241](pkg/commands/git_commands/commit.go#L235-L241)：

```go
cmdArgs := NewGitCmd("commit").
    Arg("--amend", "--no-edit", "--allow-empty", "--allow-empty-message").
    ToArgv()
```

命令：`git commit --amend --no-edit --allow-empty --allow-empty-message`

关键点：使用 `--no-edit`，不打开编辑器，直接用暂存区内容 amend 上一次提交，保留原消息。

### 2.4 Amend 非 HEAD 提交

`amendTo` 的 else 分支 [local_commits_controller.go:826-L835](pkg/gui/controllers/local_commits_controller.go#L826-L835)：

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

`RebaseCommands.AmendTo` [rebase.go:298-315](pkg/commands/git_commands/rebase.go#L298-L315)：

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

`CreateFixupCommit` [commit.go:290-294](pkg/commands/git_commands/commit.go#L290-L294)：

```go
cmdArgs := NewGitCmd("commit").Arg("--fixup=" + hash).ToArgv()
```

命令：`git commit --fixup=<targetHash>`

### 2.5 CreateFixupCommit 与 CreateAmendCommit（fixup!/amend! 提交 + stacked branch 移动）

两者均从 `createFixupCommit` 菜单弹出，成功创建提交后都会执行 stacked branch 移动与选择同步。

#### 2.5.1 CreateFixupCommit（Fixup 菜单项）

[local_commits_controller.go:1004-L1019](pkg/gui/controllers/local_commits_controller.go#L1004-L1019)：

```
WithEnsureCommittableFiles
  → WithWaitingStatusSync("Creating fixup commit")  // 同步等待
    1. Commit.CreateFixupCommit(targetHash)
       → git commit --fixup=<targetHash>
    2. moveFixupCommitToOwnerStackedBranch(targetCommit)
       → 5 项前置检查 → Rebase.MoveFixupCommitDown
    3. MoveSelectedLine(1)                          // 选择下移一行
    4. Refresh(SYNC)                                 // 同步刷新所有 UI
```

#### 2.5.2 CreateAmendCommit（两个 Amend 菜单项）

`createAmendCommit` 函数 [local_commits_controller.go:1090-L1127](pkg/gui/controllers/local_commits_controller.go#L1090-L1127)：

**面板打开阶段**：
1. 获取原提交消息 `GetCommitMessage(commit.Hash())`
2. 自动换行预处理
3. 切取原始 subject：`originalSubject, _, _ := strings.Cut(commitMessage, "\n")`
4. `OpenCommitMessagePanel`，设置 `OnSwitchToEditor = nil`（不支持切换编辑器）

**OnConfirm 回调阶段**（与 Fixup 模式相同的后处理）：

```
OnConfirm(summary, description):
  → WithWaitingStatusSync("Creating fixup commit")
    1. Commit.CreateAmendCommit(originalSubject, summary, description, includeFileChanges)
       → 构建命令参数见下文
    2. moveFixupCommitToOwnerStackedBranch(targetCommit)
       → 前置检查 + Rebase.MoveFixupCommitDown
    3. MoveSelectedLine(1)
    4. Refresh(SYNC)
```

#### 2.5.3 Commit.CreateAmendCommit 参数组织

[commit.go:297-309](pkg/commands/git_commands/commit.go#L297-L309)：

```go
func (self *CommitCommands) CreateAmendCommit(originalSubject, newSubject, newDescription string, includeFileChanges bool) error {
    description := newSubject
    if newDescription != "" {
        description += "\n\n" + newDescription
    }
    cmdArgs := NewGitCmd("commit").
        Arg("-m", "amend! "+originalSubject).  // 第一个 -m：git 识别用的 amend! 头
        Arg("-m", description).                  // 第二个 -m：实际的新消息（subject + body）
        ArgIf(!includeFileChanges, "--only", "--allow-empty").
        ToArgv()
    return self.cmd.New(cmdArgs).Run()
}
```

命令示例（无文件变更时）：
```
git commit -m "amend! original subject text" -m "new subject\n\nnew description" --only --allow-empty
```

`includeFileChanges=false` 时加 `--only --allow-empty` 创建空消息提交，后续通过 autosquash 合并时只替换消息内容。

#### 2.5.4 moveFixupCommitToOwnerStackedBranch 前置条件判断

[local_commits_controller.go:1046-L1088](pkg/gui/controllers/local_commits_controller.go#L1046-L1088)：

```go
func (self *LocalCommitsController) moveFixupCommitToOwnerStackedBranch(targetCommit *models.Commit) error {
    // 1. Git 版本 >= 2.38.0（引入 rebase.updateRefs）
    if self.c.Git().Version.IsOlderThan(2, 38, 0) { return nil }
    // 2. 当前不在 rebase/merge/cherry-pick 等中间状态
    if self.c.Git().Status.WorkingTreeState().Any() { return nil }
    // 3. 目标 commit 未进入 main 分支（Status != Merged）
    if targetCommit.Status == models.StatusMerged { return nil }
    // 4. 用户开启了 rebase.updateRefs 配置
    if !self.c.Git().Config.GetRebaseUpdateRefs() { return nil }
    // 5. 从当前选中位置向上找到有对应 branch head 的 commit（stacked branch head）
    headOfOwnerBranchIdx := -1
    for i := self.context().GetSelectedLineIdx(); i > 0; i-- {
        if lo.SomeBy(self.c.Model().Branches, func(b *models.Branch) bool {
            return b.CommitHash == self.c.Model().Commits[i].Hash()
        }) {
            headOfOwnerBranchIdx = i
            break
        }
    }
    if headOfOwnerBranchIdx == -1 { return nil }
    // 通过以上检查，执行移动
    return self.c.Git().Rebase.MoveFixupCommitDown(self.c.Model().Commits, headOfOwnerBranchIdx)
}
```

`Rebase.MoveFixupCommitDown` [rebase.go:317-L328](pkg/commands/git_commands/rebase.go#L317-L328)：

```go
func (self *RebaseCommands) MoveFixupCommitDown(commits []*models.Commit, targetCommitIndex int) error {
    fixupHash, err := self.getHashOfLastCommitMade()  // HEAD = 刚创建的 fixup/amend! 提交
    if err != nil { return err }
    return self.PrepareInteractiveRebaseCommand(PrepareInteractiveRebaseCommandOpts{
        baseHashOrRoot: getBaseHashOrRoot(commits, targetCommitIndex+1),
        overrideEditor: true,
        // 最后一个参数 applyAutosquash=false：只移动不 squash（等用户手动执行 SquashAboveCommits）
        instruction:    daemon.NewMoveFixupCommitDownInstruction(commits[targetCommitIndex].Hash(), fixupHash, false),
    }).Run()
}
```

注意这里第 3 个参数 `applyAutosquash=false`，区别于 `AmendTo` 的 `true`：
- AmendTo：移动 + 立即 autosquash 一步到位合并
- CreateFixupCommit / CreateAmendCommit：只移动位置，保留 fixup!/amend! 标记，等用户后续手动 squash

### 2.6 Amend Commit 属性（作者、Co-author）

通过 `GenericAmend` [rebase.go:89-112](pkg/commands/git_commands/rebase.go#L89-L112) 统一处理：

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

- **ResetAuthor** [commit.go:24-30](pkg/commands/git_commands/commit.go#L24-L30)：
  `git commit --allow-empty --allow-empty-message --only --no-edit --amend --reset-author`

- **SetAuthor** [commit.go:33-39](pkg/commands/git_commands/commit.go#L33-L39)：
  `git commit --allow-empty --allow-empty-message --only --no-edit --amend --author="Name <Email>"`

- **AddCoAuthor** [commit.go:42-55](pkg/commands/git_commands/commit.go#L42-L55)：
  先获取原消息 → 在末尾追加 `Co-authored-by:` 行 → 执行 `git commit --allow-empty --amend --only -m <newMessage>`

### 2.7 GPG 签名处理封装

`GpgHelper.WithGpgHandling` [gpg_helper.go:26-41](pkg/gui/controllers/helpers/gpg_helper.go#L26-L41) 根据配置决定执行方式：

- **需要 GPG 子进程**（如需要输入密码）：用 `RunSubprocess` 挂起 lazygit 让 git 与终端交互，成功后调 `onSuccess` 回调，最后 `Refresh(ASYNC)`
- **不需要**：用 `runAndStream` 包装在 `WithWaitingStatus` 中，流式输出命令日志，成功后同样 `Refresh(ASYNC)`

---

## 三、完成后的状态刷新

### 3.1 刷新触发的几种方式

根据上下文不同，刷新有以下几种触发入口：

| 场景 | 触发方式 | 代码位置 |
|------|---------|---------|
| Amend HEAD 完成 | `Refresh(ASYNC)` | local_commits_controller.go L822 |
| Reword 非 HEAD 完成 | `Refresh(ASYNC)` | local_commits_controller.go L491 |
| Reword HEAD 面板编辑 | GpgHelper 内部 `Refresh(ASYNC)` | gpg_helper.go L58 |
| Reword 外部编辑器（含非 HEAD）| `RunSubprocessAndRefresh` 自动刷新 | gui_common.go L37-L39 |
| 通过 MergeAndRebaseHelper（AmendTo 等）| `CheckMergeOrRebase` → `Refresh(ASYNC)` | merge_and_rebase_helper.go L168-L170 |
| Amend 属性（ResetAuthor/SetAuthor/AddCoAuthor）| `Refresh(ASYNC)` | local_commits_controller.go L893, L909, L928 |
| CreateFixupCommit 菜单 Fixup 项 | `WithWaitingStatusSync` 内：`MoveSelectedLine(1)` + `Refresh(SYNC)` | local_commits_controller.go L1016-L1017 |
| CreateAmendCommit（两个 Amend 菜单项）| OnConfirm 内：`MoveSelectedLine(1)` + `Refresh(SYNC)` | local_commits_controller.go L1117-L1118 |
| 更新 rebase TODO 文件（rebase 中 edit/squash/fixup）| `Refresh(SYNC, Scope: [REBASE_COMMITS])` | local_commits_controller.go L727-L729 |
| Squash/Fixup/Drop 非 HEAD 提交（非 rebase 中）| `CheckMergeOrRebase` → `Refresh(ASYNC)` | local_commits_controller.go L712 |

### 3.2 两种关键刷新模式：ASYNC vs SYNC

#### WithWaitingStatus（异步）

用于 AmendTo、Reword 非 HEAD、Amend 属性等场景。包装的函数在后台 goroutine 中执行（显示 Loading 遮罩但不阻塞所有 UI 操作），完成后通常调用 `Refresh(ASYNC)`。

#### WithWaitingStatusSync（同步）

专门用于 CreateFixupCommit / CreateAmendCommit。内部使用 `Refresh(SYNC)` 而非 `ASYNC`，语义区别：

- **SYNC**：刷新任务在 goroutine 中并发执行，但通过 `sync.WaitGroup` 等待所有 Scope 刷新完毕才返回。对 CreateFixupCommit/CreateAmendCommit 很重要，因为 `WithWaitingStatusSync` 完成后用户会继续操作，必须保证所有模型数据（Commits、Branches、Files 等）和 UI 都是最新的。
- **ASYNC**：刷新任务提交到 worker 队列后立即返回，不等待完成。适合不需要立即获得最新状态的场景，如 Amend HEAD（只是 amend 最后一次提交，用户不会马上做依赖于新状态的操作）。

### 3.3 选择状态同步（MoveSelectedLine）

CreateFixupCommit 和 CreateAmendCommit 完成后会执行 `self.context().MoveSelectedLine(1)` [local_commits_controller.go L1016, L1117](pkg/gui/controllers/local_commits_controller.go#L1016)。

原因与效果：
- 执行前，用户选中的是目标 commit（要被 fixup/amend 的那个）
- `CreateFixupCommit` / `CreateAmendCommit` 创建的新提交出现在 HEAD 位置（列表最上方）
- 新 commit 随后被 `moveFixupCommitToOwnerStackedBranch` 挪到 stacked branch 的 head 上方，从当前选中位置上方移走
- 此时如果不移动选择，光标会停在刚被挪走的 commit 原来的位置上，指向另一个无关 commit
- `MoveSelectedLine(1)` 把选择向下移动一行，让用户回到之前选中的目标 commit 上，操作体验连贯

注意：`MoveSelectedLine(1)` 在 `Refresh(SYNC)` 之前调用，刷新后模型数据更新，选择位置在新的提交数组中仍然正确指向目标 commit（因为 commit 的 hash 没变，只是列表顺序变了）。

### 3.4 Refresh 核心流程

`RefreshHelper.Refresh` [refresh_helper.go:63-237](pkg/gui/controllers/helpers/refresh_helper.go#L63-L237) 是刷新的总入口：

#### 步骤 1：确定刷新范围（Scope）

如果没传 Scope，默认刷新：
`COMMITS, BRANCHES, FILES, STASH, REFLOG, TAGS, REMOTES, WORKTREES, STATUS, BISECT_INFO, STAGING, PULL_REQUESTS`

如果传了 Scope（如只刷新 `REBASE_COMMITS`），则只刷新指定项。

#### 步骤 2：根据模式决定执行方式

- **ASYNC**：在 worker goroutine 中执行（`c.OnWorker`），不阻塞 UI，也不等待完成
- **SYNC**：在 goroutine 中并发执行，通过 `sync.WaitGroup` 等待全部完成后返回
- **BLOCK_UI**：在 UI 线程同步执行（会冻结界面），仅用于 Then 回调需要在 UI 线程执行的场景

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

`refreshCommitsAndCommitFiles` [refresh_helper.go:306-323](pkg/gui/controllers/helpers/refresh_helper.go#L306-L323)：

1. 调用 `refreshCommitsWithLimit` 重新加载提交列表
2. 如果当前在查看某个 commit 的文件列表，同步刷新 commit files context

`refreshCommitsWithLimit` [refresh_helper.go:351-383](pkg/gui/controllers/helpers/refresh_helper.go#L351-L383)：

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

对 stacked branch 场景，`rebase.updateRefs` 配置的作用体现在两个环节：

1. **移动前置条件**：`moveFixupCommitToOwnerStackedBranch` [local_commits_controller.go:1066-L1069](pkg/gui/controllers/local_commits_controller.go#L1066-L1069) 检查 `GetRebaseUpdateRefs()` 是否开启，只有开启时才允许将 fixup!/amend! 提交移到 owner stacked branch。如果未开启，移动操作会跳过，因为此时 rebase 不会自动更新下游分支指针，移动 commit 会破坏 stacked branch 的结构。

2. **分支头展示**：`refreshCommitsWithLimit` 固定传入 `IncludeRebaseCommits: true`，使得 `CommitLoader.GetCommits` 在加载提交列表时调用 `MergeRebasingCommits`，将 rebase TODO 文件中的条目（包括 `update-ref` 行）与已完成的 commit 合并展示。当 `rebase.updateRefs` 开启时，交互式 rebase 的 TODO 中会包含 `update-ref` 指令，`MergeRebasingCommits` 将这些指令作为虚拟提交项插入列表，使各 stacked branch 的分支头位置在界面上正确标注。刚被 `MoveFixupCommitDown` 移动的 fixup!/amend! 提交因此能在正确的 stacked branch 位置显示。

#### 步骤 5：Rebase Commits 单独刷新

如果 Scope 只有 `REBASE_COMMITS`（如修改 TODO 文件后），走 `refreshRebaseCommits` [refresh_helper.go:446-459](pkg/gui/controllers/helpers/refresh_helper.go#L446-L459)：

```go
updatedCommits, err := self.c.Git().Loaders.CommitLoader.MergeRebasingCommits(
    self.c.Model().HashPool, self.c.Model().Commits)
self.c.Model().Commits = updatedCommits
self.c.Model().WorkingTreeStateAtLastCommitRefresh = self.c.Git().Status.WorkingTreeState()
self.refreshView(self.c.Contexts().LocalCommits)
```

只合并 rebase TODO 信息，不重新加载全部提交，性能更好。

#### 步骤 6：视图更新（refreshView）

`refreshView` [refresh_helper.go:785-809](pkg/gui/controllers/helpers/refresh_helper.go#L785-L809) 将数据同步到 UI：

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

如果 `RefreshOptions.Then` 不为空，所有刷新完成后在当前线程执行该回调（仅支持 SYNC 和 BLOCK_UI 模式）。典型场景：启动 rebase 后通过 `Then` 重新恢复选中的 commit 范围，因为 rebase TODO 文件中可能插入 update-ref 行导致 commit 列表位置变化 [local_commits_controller.go:597-L600](pkg/gui/controllers/local_commits_controller.go#L597-L600)。

### 3.5 Merge/Rebase 结果检查

`MergeAndRebaseHelper.CheckMergeOrRebase` [merge_and_rebase_helper.go:152-170](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L152-L170) 在刷新后处理特殊情况：

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
    → WithWaitingStatus("Rewording")
      → cmdObj.StreamOutput().Run()                 // git commit --amend --only -m ...
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
        → PrepareInteractiveRebaseCommand + MoveFixupCommitDown(applyAutosquash=true)
          // 启动 rebase，自动移动 fixup commit 并 autosquash 合并
      → MergeAndRebaseHelper.CheckMergeOrRebase(err)
        → Refresh(ASYNC)
        → 检查冲突 / 空提交等特殊情况
```

### 场景 C：CreateAmendCommit（菜单 Amend 项，含 stacked branch 移动）

```
用户按 'c' 键打开 CreateFixupCommit 菜单 → 选 'a' AmendWithChanges
  → WorkingTree.WithEnsureCommittableFiles
    → LocalCommitsController.createAmendCommit(commit, true)
      → Commit.GetCommitMessage(commit.Hash())      // 获取原消息
      → 自动换行预处理
      → originalSubject = 第一行 subject
      → OpenCommitMessagePanel(opts)
        → OnSwitchToEditor = nil
        → OnConfirm = createAmendCommit 内部闭包

用户编辑消息后按 Enter
  → CommitMessageController.confirm()
    → CommitsHelper.HandleCommitConfirm()
      → 调用 OnConfirm(summary, description)
        → WithWaitingStatusSync("Creating fixup commit")
          1. Commit.CreateAmendCommit(originalSubject, summary, description, true)
             // git commit -m "amend! ..." -m ...
          2. moveFixupCommitToOwnerStackedBranch(commit)
             // 检查 Git 版本、rebase 状态、merged 状态、updateRefs 配置
             // 找到 stacked branch head → Rebase.MoveFixupCommitDown
             //   → PrepareInteractiveRebaseCommand(applyAutosquash=false)
          3. context.MoveSelectedLine(1)            // 选择回到目标 commit
          4. Refresh(SYNC)                          // 同步等待所有刷新完成
             // WaitGroup 等待 Commits/Branches/Files 等全部刷新完毕
             // Commits 数组重新加载，新 amend! 提交在 stacked branch 上方显示
```

---

## 五、核心文件索引

| 文件 | 职责 |
|------|------|
| [local_commits_controller.go](pkg/gui/controllers/local_commits_controller.go) | 按键绑定、操作入口、HEAD/非 HEAD 分支判断、createFixupCommit 菜单、createAmendCommit 面板回调、moveFixupCommitToOwnerStackedBranch 前置检查、MoveSelectedLine 选择同步 |
| [commit_message_controller.go](pkg/gui/controllers/commit_message_controller.go) | 消息面板的键盘/鼠标事件处理、粘贴拦截、历史消息切换 |
| [commits_helper.go](pkg/gui/controllers/helpers/commits_helper.go) | 消息面板打开/关闭、消息拆分合并、自动换行处理、切换外部编辑器（写临时文件）、commit 菜单（AddCoAuthor/Paste）|
| [amend_helper.go](pkg/gui/controllers/helpers/amend_helper.go) | Amend HEAD 的 GPG 封装入口 |
| [gpg_helper.go](pkg/gui/controllers/helpers/gpg_helper.go) | GPG 签名分支处理（子进程 vs 流式输出）、统一触发 ASYNC Refresh |
| [merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go) | rebase/merge 结果检查、空提交自动 skip、冲突检测与处理菜单、continue/abort/skip、冲突解决后自动提示 continue |
| [refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go) | 统一刷新入口、Scope 拆分并发刷新、WaitGroup 同步等待、ASYNC/SYNC/BLOCK_UI 模式调度、视图 PostRefreshUpdate |
| [commit.go](pkg/commands/git_commands/commit.go) | 构建 commit/amend/reword/fixup/CreateAmendCommit 的 git 命令参数、GetCommitMessage 获取原消息、AddCoAuthorToMessage 拼接 Co-author |
| [rebase.go](pkg/commands/git_commands/rebase.go) | 非 HEAD 提交修改：BeginInteractiveRebaseForCommit、RewordCommit（三步法）、GenericAmend（循环 rebase）、AmendTo（fixup + autosquash）、MoveFixupCommitDown（stacked branch 移动）|
| [gui_common.go](pkg/gui/gui_common.go) | RunSubprocessAndRefresh：执行子进程挂起 lazygit，恢复后自动触发刷新 |
