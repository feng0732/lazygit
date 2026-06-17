# Lazygit Cherry-Pick 多选执行流程详解

## 一、整体架构概览

Cherry-pick 在 lazygit 中采用"复制-粘贴"隐喻：先"复制"提交到缓冲区，再"粘贴"到目标分支。涉及的核心文件和职责如下：

| 层次 | 文件（仓库相对路径） | 职责 |
|------|---------------------|------|
| 状态模型 | `pkg/gui/modes/cherrypicking/cherry_picking.go` | 维护已复制的提交列表、上下文键、粘贴标记 |
| 控制器入口 | `pkg/gui/controllers/basic_commits_controller.go` | 绑定 `CherryPickCopy` 按键，发起范围选择 |
| 控制器入口 | `pkg/gui/controllers/local_commits_controller.go` | 绑定 `PasteCommits` 按键，触发粘贴执行 |
| 核心逻辑 | `pkg/gui/controllers/helpers/cherry_pick_helper.go` | CopyRange / Paste / Reset 等完整业务逻辑 |
| 冲突处理 | `pkg/gui/controllers/helpers/merge_and_rebase_helper.go` | CheckMergeOrRebase、冲突检测、Continue/Abort 提示 |
| 刷新检测 | `pkg/gui/controllers/helpers/refresh_helper.go` | 检测冲突文件清零、自动弹出 Continue 提示 |
| Git 命令 | `pkg/commands/git_commands/rebase.go` | CherryPickCommits、GenericMergeOrRebaseAction 等 |
| 工作区状态 | `pkg/commands/models/working_tree_state.go` | WorkingTreeState 多状态组合与优先级判断 |
| 状态探测 | `pkg/commands/git_commands/status.go` | IsInCherryPick：检测 CHERRY_PICK_HEAD 是否存在 |
| 界面呈现 | `pkg/gui/presentation/commits.go` | 已复制提交的蓝色高亮显示 |
| 集成测试 | `pkg/integration/tests/cherry_pick/cherry_pick_conflicts.go` | 冲突场景的端到端行为验证 |

---

## 二、阶段一：批量选择（Copy）

### 2.1 入口：`copyRange` 按键绑定

在 `pkg/gui/controllers/basic_commits_controller.go` 中，`CherryPickCopy` 按键绑定到 `self.copyRange`，后者直接委托给 `CherryPickHelper.CopyRange`：

```go
func (self *BasicCommitsController) copyRange(*models.Commit) error {
    return self.c.Helpers().CherryPick.CopyRange(self.context.GetCommits(), self.context)
}
```

守卫检查 `canCopyCommits`（同文件）：不允许复制 hash 为空的条目。

### 2.2 上下文隔离：`resetIfNecessary`

`pkg/gui/controllers/helpers/cherry_pick_helper.go` 中的 `CopyRange` 首先调用 `resetIfNecessary`：

```go
func (self *CherryPickHelper) resetIfNecessary(context types.Context) error {
    oldContextKey := types.ContextKey(self.getData().ContextKey)
    if oldContextKey != context.GetKey() {
        self.getData().ContextKey = string(context.GetKey())
        self.getData().CherryPickedCommits = make([]*models.Commit, 0)
    }
    return nil
}
```

**设计意图**：只允许从同一个 context（LocalCommits / ReflogCommits / SubCommits）中复制。切换 context 时之前的选择被清空——因为不同 context 的 `commitsList` 排列规则不同，混合后顺序无意义。

### 2.3 粘贴后覆盖：`DidPaste` 清理

```go
if self.getData().DidPaste {
    self.getData().CherryPickedCommits = nil
    self.getData().DidPaste = false
}
```

粘贴操作不会清空缓冲区，而是设置 `DidPaste = true`，使 UI 隐藏蓝色标记。当用户再次执行 Copy 时才真正清空旧数据，实现"粘贴后再次复制 = 覆盖"。

### 2.4 范围选择逻辑：Toggle 语义

```go
allCommitsCopied := lo.EveryBy(commitsList[startIdx:endIdx+1], func(commit *models.Commit) bool {
    return commitSet.Includes(commit.Hash())
})

if allCommitsCopied {
    // 范围内全部已选中 → 取消选择
    for index := startIdx; index <= endIdx; index++ {
        self.getData().Remove(commit, commitsList)
    }
} else {
    // 范围内有未选中的 → 全部选中
    for index := startIdx; index <= endIdx; index++ {
        self.getData().Add(commit, commitsList)
    }
}
```

### 2.5 顺序保证：`update` 方法

`pkg/gui/modes/cherrypicking/cherry_picking.go` 中的 `Add` / `Remove` 最终都走 `update` 方法：

```go
func (self *CherryPicking) update(selectedHashSet *set.Set[string], commitsList []*models.Commit) {
    self.CherryPickedCommits = lo.Filter(commitsList, func(commit *models.Commit, _ int) bool {
        return selectedHashSet.Includes(commit.Hash())
    })
}
```

**关键洞察**：`CherryPickedCommits` 的顺序**不是**按用户点击先后，而是按**源列表中的出现顺序**排列。`lo.Filter` 遍历 `commitsList`，只保留 hash 在集合中的元素，结果天然保序。

---

## 三、阶段二：顺序执行（Paste）

### 3.1 入口：`paste` 按键绑定

`pkg/gui/controllers/local_commits_controller.go` 中 `PasteCommits` 按键绑定到 `self.paste`：

```go
func (self *LocalCommitsController) paste() error {
    return self.c.Helpers().CherryPick.Paste()
}
```

`canPaste` 守卫：缓冲区为空时禁用。

### 3.2 确认对话框与同步等待

`pkg/gui/controllers/helpers/cherry_pick_helper.go` 中的 `Paste` 弹出确认框，用户确认后进入 `WithWaitingStatusSync`：

```go
self.c.Confirm(types.ConfirmOpts{
    Title:  self.c.Tr.CherryPick,
    Prompt: "Are you sure you want to cherry-pick the N copied commit(s)...",
    HandleConfirm: func() error {
        return self.c.WithWaitingStatusSync(self.c.Tr.CherryPickingStatus, func() error {
            // ...核心逻辑
        })
    },
})
```

`WithWaitingStatusSync` 是同步阻塞操作，UI 显示等待状态。

### 3.3 工作区脏文件处理：Auto-Stash

```go
mustStash := IsWorkingTreeDirtyExceptSubmodules(self.c.Model().Files, self.c.Model().Submodules)
if mustStash {
    if err := self.c.Git().Stash.Push(self.c.Tr.AutoStashForCherryPicking); err != nil {
        return err
    }
}
```

如果工作区有脏文件（忽略子模块），先 `git stash push`，cherry-pick 完成后再 `git stash pop`。

### 3.4 一发入魂：`CherryPickCommits`

```go
cherryPickedCommits := self.getData().CherryPickedCommits
result := self.c.Git().Rebase.CherryPickCommits(cherryPickedCommits)
```

`pkg/commands/git_commands/rebase.go` 中 `CherryPickCommits` 构造的 Git 命令：

```go
func (self *RebaseCommands) CherryPickCommits(commits []*models.Commit) error {
    hasMergeCommit := lo.SomeBy(commits, func(c *models.Commit) bool { return c.IsMerge() })
    cmdArgs := NewGitCmd("cherry-pick").
        Arg("--allow-empty").
        ArgIf(self.version.IsAtLeast(2, 45, 0), "--empty=keep", "--keep-redundant-commits").
        ArgIf(hasMergeCommit, "-m1").
        Arg(lo.Reverse(lo.Map(commits, func(c *models.Commit, _ int) string { return c.Hash() }))...).
        ToArgv()
    return self.cmd.New(cmdArgs).Run()
}
```

**关键细节**：

1. **`lo.Reverse`**：`CherryPickedCommits` 按"列表从上到下"排列（新→旧），但 `git cherry-pick` 要求参数从旧→新，所以需要反转。
2. **`--allow-empty`**：保留空提交。
3. **`-m1`**：如果有 merge commit，选第一个父节点作为主线。
4. **一次性执行**：不是逐个 cherry-pick，而是将所有 hash 一次性传给 `git cherry-pick`，由 Git 内部按顺序依次应用。这意味着顺序执行的"循环"在 Git 内部完成。

### 3.5 结果检查与选择偏移

```go
err := self.rebaseHelper.CheckMergeOrRebaseWithRefreshOptions(result, types.RefreshOptions{Mode: types.SYNC})
if err != nil {
    return result
}

if commit := self.c.Contexts().LocalCommits.GetSelected(); commit != nil && !commit.IsTODO() {
    self.c.Contexts().LocalCommits.MoveSelection(len(cherryPickedCommits))
    self.c.Contexts().LocalCommits.FocusLine(true)
}
```

成功后光标向下移动 N 行，保持原来选中的提交仍在视觉位置上被选中。

---

## 四、阶段三：冲突检测与初始处理

### 4.1 冲突检测链

```
Paste()
  → CherryPickCommits() → 返回 error
  → CheckMergeOrRebaseWithRefreshOptions(result, SYNC)
    ├─ error == nil → 直接返回
    ├─ error 包含 "No changes - did you forget to use"
    │   → 自动执行 skip（genericMergeCommand("skip")）
    ├─ error 包含 "The previous cherry-pick is now empty"
    │   → 自动执行 skip
    ├─ error 包含 "No rebase in progress?"
    │   → 视为已完成，直接返回 nil
    └─ 其他 error → CheckForConflicts(result)
         └─ isMergeConflictErr(result.Error())
              ├─ 匹配 7 种冲突特征字符串 → PromptForConflictHandling()
              └─ 不匹配 → 原样返回 error
```

这段逻辑位于 `pkg/gui/controllers/helpers/merge_and_rebase_helper.go` 的 `CheckMergeOrRebaseWithRefreshOptions` 和 `CheckForConflicts`。

### 4.2 冲突特征字符串

```go
var conflictStrings = []string{
    "Failed to merge in the changes",
    "When you have resolved this problem",
    "fix conflicts",
    "Resolve all conflicts manually",
    "Merge conflict in file",
    "hint: after resolving the conflicts",
    "CONFLICT (content):",
}
```

只要 Git 输出包含任一字符串即判定为冲突。

### 4.3 冲突时的用户选项

`PromptForConflictHandling`（同文件）提供两个选项：

1. **查看冲突文件**（View conflicts）：跳转到 Files 面板
2. **中止操作**（Abort cherry-pick）：执行 `git cherry-pick --abort`

注意：这里没有 "Continue" 选项，因为冲突尚未解决时不能 continue。Continue 会在冲突全部解决后自动弹出。

### 4.4 冲突时不清空缓冲区

这是 cherry-pick 流程中最精妙的设计之一：

```go
isInCherryPick, result := self.c.Git().Status.IsInCherryPick()
if result != nil {
    return result
}
if !isInCherryPick {
    self.getData().DidPaste = true
    self.rerender()
    // ...stash pop...
}
```

`Paste` 方法在执行完 cherry-pick 后检查是否仍处于 cherry-pick 状态：

- **不在 cherry-pick 状态**（成功完成）→ 设置 `DidPaste = true`，隐藏蓝色标记，恢复 stash
- **仍在 cherry-pick 状态**（有冲突）→ **不设置 DidPaste**，保留缓冲区，不恢复 stash

保留缓冲区的原因：用户可能想中止 cherry-pick 后重新粘贴。如果冲突时清空了缓冲区，用户就必须重新复制那些提交。

### 4.5 IsInCherryPick 的特殊处理

`pkg/commands/git_commands/status.go` 中的 `IsInCherryPick`：

```
CHERRY_PICK_HEAD 存在时，还需检查 rebase-merge/stopped-sha
如果两者 hash 一致 → 这其实是 rebase 过程中的 cherry-pick，不算独立的 cherry-pick
如果 hash 不一致 → 确实是独立的 cherry-pick 操作
```

这是因为 Git 的 rebase 内部实现为一系列 cherry-pick，可能会留下 `CHERRY_PICK_HEAD` 文件。lazygit 需要区分"rebase 中暂停"和"独立 cherry-pick 冲突"。

### 4.6 WorkingTreeState 的多状态组合

`pkg/commands/models/working_tree_state.go` 中 `WorkingTreeState` 可以同时有多个状态为 true（如 Rebasing + CherryPicking），`Effective()` 方法定义了优先级：

```go
func (self WorkingTreeState) Effective() EffectiveWorkingTreeState {
    if self.Reverting { return WORKING_TREE_STATE_REVERTING }
    if self.CherryPicking { return WORKING_TREE_STATE_CHERRY_PICKING }
    if self.Merging { return WORKING_TREE_STATE_MERGING }
    if self.Rebasing { return WORKING_TREE_STATE_REBASING }
    return WORKING_TREE_STATE_NONE
}
```

优先级：Reverting > CherryPicking > Merging > Rebasing。如果用户在 rebase 过程中执行 cherry-pick 并冲突，lazygit 认为当前"有效状态"是 cherry-pick，用户必须先处理 cherry-pick 才能继续 rebase。

---

## 五、阶段四：冲突解决与后续提交应用

### 5.1 冲突解决的触发：自动检测

用户在 Files 面板中逐个解决冲突文件后，每次保存或操作都会触发刷新。`pkg/gui/controllers/helpers/refresh_helper.go` 的 `refreshStateFiles` 方法检测到：

```go
if self.c.Git().Status.WorkingTreeState().Any() && conflictFileCount == 0 && prevConflictFileCount > 0 {
    self.c.OnUIThread(func() error { return self.mergeAndRebaseHelper.PromptToContinueRebase() })
}
```

即：**处于任一合并/变基/拣选状态** + **本次刷新时冲突文件数从 >0 变为 0** → 自动弹出 Continue 确认框。

### 5.2 PromptToContinueRebase：确认继续

`pkg/gui/controllers/helpers/merge_and_rebase_helper.go` 中的 `PromptToContinueRebase`：

```go
func (self *MergeAndRebaseHelper) PromptToContinueRebase() error {
    self.c.Confirm(types.ConfirmOpts{
        Title:  self.c.Tr.Continue,
        Prompt: fmt.Sprintf(self.c.Tr.ConflictsResolved, commandName),
        HandleConfirm: func() error {
            // 先刷新文件，确保判断准确
            self.c.Refresh(types.RefreshOptions{Mode: types.SYNC, Scope: []types.RefreshableView{types.FILES}})

            unstagedFiles := GetUnstagedFilesExceptSubmodules(self.c.Model().Files, self.c.Model().Submodules)
            if len(unstagedFiles) > 0 {
                self.c.Confirm(types.ConfirmOpts{
                    Title:  self.c.Tr.Continue,
                    Prompt: self.c.Tr.UnstagedFilesAfterConflictsResolved,
                    HandleConfirm: func() error {
                        self.c.LogAction(self.c.Tr.Actions.StageAllFiles)
                        if err := self.c.Git().WorkingTree.StageFiles(unstagedFiles, []string{}); err != nil {
                            return err
                        }
                        return self.genericMergeCommand(REBASE_OPTION_CONTINUE)
                    },
                })
                return nil
            }

            return self.genericMergeCommand(REBASE_OPTION_CONTINUE)
        },
    })
}
```

两层确认：

1. 第一层：确认所有冲突已解决，要继续 cherry-pick
2. 第二层（如有）：如果还有未暂存文件，询问是否自动 stage 后再继续

### 5.3 genericMergeCommand：统一执行 continue / abort / skip

`pkg/gui/controllers/helpers/merge_and_rebase_helper.go` 中的 `genericMergeCommand`：

```go
func (self *MergeAndRebaseHelper) genericMergeCommand(command string) error {
    status := self.c.Git().Status.WorkingTreeState()
    if status.None() {
        return errors.New(self.c.Tr.NotMergingOrRebasing)
    }

    commandType := status.CommandName()  // 对 cherry-pick 就是 "cherry-pick"

    // 两种情况用子进程（打开终端）：
    // 1. merge 且 ManualCommit=true 且 不是 abort
    // 2. rebase 且 存在 exec todo 且 不是 abort
    needsSubprocess := (effectiveStatus == WORKING_TREE_STATE_MERGING && ...) ||
        (effectiveStatus == WORKING_TREE_STATE_REBASING && ... && self.hasExecTodos())

    if needsSubprocess {
        return self.c.RunSubprocessAndRefresh(
            self.c.Git().Rebase.GenericMergeOrRebaseActionCmdObj(commandType, command),
        )
    }
    result := self.c.Git().Rebase.GenericMergeOrRebaseAction(commandType, command)
    if err := self.CheckMergeOrRebase(result); err != nil {
        return err
    }
    return nil
}
```

对于 cherry-pick 的 continue，commandType="cherry-pick"，command="continue"，走非子进程路径。

### 5.4 Git 命令层：GenericMergeOrRebaseAction

`pkg/commands/git_commands/rebase.go`：

```go
func (self *RebaseCommands) GenericMergeOrRebaseAction(commandType string, command string) error {
    err := self.runSkipEditorCommand(self.GenericMergeOrRebaseActionCmdObj(commandType, command))
    // ...
}

func (self *RebaseCommands) GenericMergeOrRebaseActionCmdObj(commandType string, command string) *oscommands.CmdObj {
    cmdArgs := NewGitCmd(commandType).Arg("--" + command).ToArgv()
    return self.cmd.New(cmdArgs)
}
```

生成的命令为：`git cherry-pick --continue`。

关键在于 `runSkipEditorCommand`：通过设置环境变量让 lazygit 自身作为 editor/sequence-editor，收到 `ExitImmediatelyInstruction` 后立即退出（0），使 Git 不会打开任何编辑器。

### 5.5 continue 后的递归流程

`genericMergeCommand` 执行完 `git cherry-pick --continue` 后，调用 `CheckMergeOrRebase`，这与初始 Paste 时调用的 `CheckMergeOrRebaseWithRefreshOptions` 走同一条检测链：

```
genericMergeCommand("continue")
  → git cherry-pick --continue
  → CheckMergeOrRebase(result)
      ├─ 所有剩余提交都成功应用 → 返回 nil，流程结束
      ├─ 下一提交又遇到冲突 → PromptForConflictHandling()
      │                            （回到阶段四的起点，重复解决）
      └─ 遇到空提交 → 自动 skip
```

这就是"冲突 → 解决 → continue → 应用后续提交 → 可能再冲突 → 再解决 → ..."的循环。Git 内部知道还有哪些提交待应用（保存在 `.git/CHERRY_PICK_HEAD` 等文件中），lazygit 只需在每次 continue 后做相同的冲突检测即可。

### 5.6 abort：完整回滚

用户在冲突提示菜单中选 Abort，同样走 `genericMergeCommand("abort")`：

```
genericMergeCommand("abort")
  → git cherry-pick --abort
  → 工作区回到 cherry-pick 开始前的状态
  → CHERRY_PICK_HEAD 文件被删除
  → CheckMergeOrRebase 返回 nil
```

abort 之后缓冲区仍然保留（因为冲突时 DidPaste 从未被设为 true），用户可以立即再次粘贴，或修改选择后重新复制。

### 5.7 skip：跳过当前提交

`CheckMergeOrRebaseWithRefreshOptions` 中对特定错误自动触发 skip：

- `"No changes - did you forget to use"` → 自动 `git cherry-pick --skip`
- `"The previous cherry-pick is now empty"` → 自动 `git cherry-pick --skip`

skip 同样走 `genericMergeCommand("skip")`，Git 跳过当前提交并继续后续提交。用户也可在 `CreateRebaseOptionsMenu`（Rebase options 菜单）中手动触发 skip。

---

## 六、缓冲区状态完整生命周期

### 6.1 CherryPicking 结构体字段

`pkg/gui/modes/cherrypicking/cherry_picking.go`：

```go
type CherryPicking struct {
    CherryPickedCommits []*models.Commit   // 已复制的提交列表
    ContextKey          string             // 来源 context（防止跨上下文混合）
    DidPaste            bool               // 是否已粘贴过（影响视觉呈现）
}
```

### 6.2 状态转换表

| 场景 | CherryPickedCommits | ContextKey | DidPaste | Active() | 蓝色高亮 |
|------|---------------------|------------|----------|----------|----------|
| 初始状态 | `[]` | `""` | `false` | `false` | 无 |
| 第一次 Copy | 非空 | 对应 context | `false` | `true` | 有 |
| 切换 context 后 Copy | 被清空→新选择 | 新 context | `false` | `true` | 有（新上下文） |
| 粘贴成功，无冲突 | 保持不变 | 不变 | `true` | `false` | 无（DidPaste=true → SelectedHashSet 返回空集） |
| 粘贴失败，有冲突 | 保持不变 | 不变 | `false` | `true` | 有（用户可 abort 后重贴） |
| 冲突后 abort | 保持不变 | 不变 | `false` | `true` | 有（可再次 paste） |
| 冲突后 continue，仍有后续冲突 | 保持不变 | 不变 | `false` | `true` | 有 |
| 冲突后 continue，全部成功 | 保持不变 | 不变 | `false`** | 见 6.3 | 见 6.3 |
| 粘贴后再次 Copy | 被清空→新选择 | 不变/更新 | `false` | `true` | 有 |
| 手动 Reset | `nil` | `""` | `false` | `false` | 无 |

\* 注意冲突解决后 continue 全部成功的情况，详见 6.3。

### 6.3 一个微妙的设计缺陷：continue 成功后 DidPaste 仍为 false

观察 `Paste` 方法的控制流：

```
Paste()
  ├─ WithWaitingStatusSync {
  │    ├─ CherryPickCommits() → 冲突，返回 error
  │    ├─ CheckMergeOrRebaseWithRefreshOptions() → PromptForConflictHandling()
  │    └─ 检查 IsInCherryPick() → true → 不设置 DidPaste=true
  └─ }

...（用户解决冲突，点击 continue）...

genericMergeCommand("continue")
  ├─ git cherry-pick --continue → 成功
  └─ CheckMergeOrRebase() → nil
```

问题：**continue 成功的代码路径不经过 Paste 方法中的那段 `if !isInCherryPick` 检查**，因此 `DidPaste` 永远不会被设置为 `true`。

验证：在 `pkg/integration/tests/cherry_pick/cherry_pick_conflicts.go` 的测试用例中，冲突解决并 continue 后，信息栏仍显示 "2 commits copied"：

```go
t.Views().Information().Content(Contains("2 commits copied"))
```

只有当用户按 Esc（ResetCherryPick）手动重置，或再次 Copy 时覆盖，缓冲区才会清空。这是一个已知的设计取舍：避免跨多层状态传递 DidPaste 设置逻辑，代价是冲突解决后信息栏不会自动隐藏"commits copied"提示。

### 6.4 Active() 与 SelectedHashSet() 的联动

```go
func (self *CherryPicking) Active() bool {
    return self.CanPaste() && !self.DidPaste
}

func (self *CherryPicking) SelectedHashSet() *set.Set[string] {
    if self.DidPaste {
        return set.New[string]()   // 返回空集，蓝色高亮不显示
    }
    hashes := lo.Map(self.CherryPickedCommits, func(commit *models.Commit, _ int) string {
        return commit.Hash()
    })
    return set.NewFromSlice(hashes)
}
```

`Active()` 决定信息栏是否显示 "N commits copied"。`SelectedHashSet()` 决定提交列表中哪些 hash 被蓝色高亮。两者都受 `DidPaste` 控制，但数据（`CherryPickedCommits`）始终保留，除非显式 Reset 或被新的 Copy 覆盖。

---

## 七、完整流程图

```
用户按键 CherryPickCopy（可多次）
       │
       ▼
  BasicCommitsController.copyRange()
       │
       ▼
  CherryPickHelper.CopyRange()
       ├── resetIfNecessary()     ── 切换 context 时清空
       ├── DidPaste 清理          ── 粘贴后首次复制 → 覆盖
       ├── Toggle 语义判断        ── 全选/反选
       ├── Add/Remove → update()  ── 按列表顺序排列
       └── rerender()             ── 刷新蓝色高亮

       ··· 用户可能继续选择更多提交 ···

用户切换到目标分支，按键 PasteCommits
       │
       ▼
  LocalCommitsController.paste()
       │
       ▼
  CherryPickHelper.Paste()
       ├── 弹出确认对话框
       │
       ▼ (用户确认)
  WithWaitingStatusSync ────────────────────────────
       │                                              │
       ├── 工作区脏？→ git stash push                 │ 同步阻塞
       │                                              │
       ├── CherryPickCommits()                        │
       │   └── git cherry-pick --allow-empty [-m1]    │
       │       <hashes 反序>                           │
       │                                              │
       ├── CheckMergeOrRebaseWithRefreshOptions()     │
       │   ├── 成功？→ MoveSelection(N)                │
       │   ├── 空提交？→ 自动 skip                     │
       │   └── 冲突？→ PromptForConflictHandling()   │
       │       ├── 查看冲突文件 → Files 面板          │
       │       └── Abort → git cherry-pick --abort   │
       │                                              │
       └── 检查 IsInCherryPick()                      │
           ├── 不在 → DidPaste=true, stash pop        │
           └── 仍在 → 保留缓冲区                       │
                                                    ──┘
       ··· 冲突场景 ···
                                                    
       ┌──────────────────────────────────────────┐
       │  用户在 Files 面板逐个解决冲突文件          │
       │  每次保存/操作触发 refreshStateFiles()    │
       │  检测到冲突文件数从 >0 变为 0              │
       └────────────────────┬─────────────────────┘
                            │
                            ▼
            PromptToContinueRebase()
                ├── 有 unstaged 文件？→ 自动 stage
                └── genericMergeCommand("continue")
                        └── git cherry-pick --continue
                                ├── 成功应用所有剩余提交 → 结束
                                ├── 下一提交又冲突 → 回到 Conflict
                                │     PromptForConflictHandling()
                                │     （重复此循环）
                                └── 遇到空提交 → 自动 skip
                                            └── continue 剩余提交
```

---

## 八、关键设计总结

### 8.1 批量选择如何保证顺序

- `CherryPickedCommits` 始终按源列表的排列顺序存储，而非用户点击顺序
- `update()` 通过 `lo.Filter(commitsList, ...)` 重建列表，天然保序
- 粘贴时 `lo.Reverse` 反转为旧→新，满足 `git cherry-pick` 的参数要求

### 8.2 顺序执行为什么是"一发"而非循环

- `CherryPickCommits` 将所有 hash 一次性传给 `git cherry-pick`
- Git 内部按参数顺序逐个应用，遇到冲突时暂停（留下 `CHERRY_PICK_HEAD`）
- lazygit 不需要自己实现逐个 cherry-pick 的循环逻辑
- 好处：冲突时 Git 知道还有哪些提交待应用，`--continue` 会自动继续后续提交

### 8.3 冲突处理与缓冲区的配合

- 冲突时不清空 `CherryPickedCommits`，允许用户 abort 后重新 paste
- `DidPaste` 标记实现了"视觉隐藏 + 数据保留"的两层语义
- `Active()` = `CanPaste() && !DidPaste`，冲突时 `Active()` 仍为 true，信息栏继续显示"N commits copied"
- 只有在 Paste 方法的同步路径中成功完成（不在 cherry-pick 状态）后才设置 `DidPaste = true` 隐藏标记
- **已知设计取舍**：冲突后 continue 全部成功时，`DidPaste` 不会被自动设置为 true，缓冲区视觉提示不会自动消失（需手动 Reset 或下次 Copy 覆盖）

### 8.4 上下文隔离的意义

- 只允许从同一个 context 复制，防止跨面板混合提交
- 切换 context 时 `resetIfNecessary` 自动清空，用户无需手动 Reset
- 不同 context 的 `commitsList` 排列规则不同，混合后顺序无意义

### 8.5 Continue 的自动触发机制

- 不在冲突提示菜单中提供 Continue（因为冲突未解决时 continue 无意义）
- 通过 `refreshStateFiles` 检测冲突文件数清零 → 自动弹出 `PromptToContinueRebase`
- Continue 前还会检查是否有未暂存文件，提示用户自动 stage
- Continue 后走与 Paste 相同的 `CheckMergeOrRebase` 检测链，形成"冲突→解决→continue→再冲突"的循环
