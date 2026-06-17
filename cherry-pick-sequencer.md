# Cherry-Pick 冲突后剩余提交的记录机制与 Sequencer 详解

## 一、核心问题：冲突后 Git 如何记住"还要拣哪些提交"

当执行批量 cherry-pick 并遇到冲突时，Git 暂停执行。此时有两个关键问题：

1. **剩余待拣的提交列表保存在哪里？**
2. **当前冲突的是哪一个提交？**
3. **`--continue` 恢复时怎么知道从哪里继续？**

答案就在 `.git/sequencer/` 目录和 `.git/CHERRY_PICK_HEAD` 文件中。本文从 Git 内部文件结构、lazygit 代码实现、状态呈现三个层面逐层解析。

---

## 二、.git 目录文件总览

批量 cherry-pick（≥2 个提交）与单提交 cherry-pick 使用完全不同的记录机制：

| 机制 | 文件路径 | 作用 | 何时存在 |
|------|---------|------|---------|
| **Sequencer** | `.git/sequencer/todo` | 待处理的提交队列（从旧到新） | 批量 cherry-pick / revert 期间 |
| | `.git/sequencer/done` | 已成功应用的提交记录 | 同上 |
| | `.git/sequencer/head` | cherry-pick 开始时的 HEAD hash | 同上 |
| | `.git/sequencer/abort-safety` | 防止跨 worktree 误操作 | 同上 |
| | `.git/sequencer/orig-head` | 起始 HEAD（供 `--abort` 回退） | 同上 |
| **HEAD 标记** | `.git/CHERRY_PICK_HEAD` | 当前正在拣的提交 hash（全量 40 位） | **所有** cherry-pick 期间（含单提交） |
| | `.git/MERGE_MSG` | 预填的提交信息（冲突解决后继续提交用） | 冲突暂停期间 |
| | `.git/REVERT_HEAD` | 当前正在 revert 的提交 hash | revert 期间 |
| **Rebase（对比参考）** | `.git/rebase-merge/git-rebase-todo` | rebase 待办 | interactive rebase 期间 |
| | `.git/rebase-merge/done` | rebase 已完成 | 同上 |
| | `.git/rebase-merge/stopped-sha` | rebase 冲突时暂停处 | 同上 |

### 2.1 为什么单提交不使用 sequencer

代码中的注释（`pkg/commands/git_commands/commit_loader.go` 第 265-267 行）给出了原因：

> For single-commit cherry-picks and reverts, git apparently doesn't use the sequencer; in that case, CHERRY_PICK_HEAD or REVERT_HEAD is our conflicting commit.

Git 对单提交 cherry-pick 做了"轻量处理"——因为只涉及一个提交，只需一个 `CHERRY_PICK_HEAD` 就能完整描述"暂停状态"，无需完整的 sequencer 队列。

只有当 cherry-pick ≥2 个提交时，Git 才创建 `.git/sequencer/` 目录记录完整队列。

---

## 三、Sequencer 文件格式详解

### 3.1 `.git/sequencer/todo` —— 剩余待办队列

**格式**（由 `git-todo-parser/todo` 库解析）：

```
pick <hash> <subject>
pick <hash> <subject>
...
```

每行一条指令，命令类型包括 `pick`、`revert` 等（cherry-pick 场景下都是 `pick`）。hash 是**缩写形式**（通常 7 位，够唯一即可），subject 是提交标题（可含空格）。

**关键性质**：
- **从上到下即执行顺序**：最上面一行是**下一个**要应用的提交
- **冲突时，当前提交是 todo 中的第一行**（Git 还没来得及把它移到 done 就停了）
- lazygit 展示时需要 `utils.Prepend` 反转顺序，使其显示为"新→旧"，与日志列表方向一致

对应代码 `pkg/commands/git_commands/commit_loader.go` 第 483-513 行 `getSequencerCommits`：

```go
func (self *CommitLoader) getSequencerCommits(hashPool *utils.StringPool) []*models.Commit {
    bytesContent, _ := self.readFile(filepath.Join(self.repoPaths.WorktreeGitDirPath(), "sequencer/todo"))
    todos, _ := todo.Parse(bytes.NewBuffer(bytesContent), self.config.GetCoreCommentChar())

    commits := []*models.Commit{}
    for _, t := range todos {
        commits = utils.Prepend(commits, models.NewCommit(hashPool, models.NewCommitOpts{
            Hash:   t.Commit,
            Name:   t.Msg,
            Status: models.StatusCherryPickingOrReverting,
            Action: t.Command,
        }))
    }
    return commits
}
```

### 3.2 `.git/sequencer/done` —— 已完成记录

格式与 todo 完全相同。每成功应用一个提交，Git 就把它从 todo 第一行移除，追加到 done 文件末尾。

**lazygit 的用法**：在 rebase 场景下，`getConflictedCommit` 函数（`commit_loader.go` 第 386-403 行）通过比较 `done` 的最后一条与 `todo` 的第一条，判断冲突提交是否已被"重新排入 todo"（Git 在某些错误下会把失败的提交再塞回 todo 开头）。

### 3.3 `.git/CHERRY_PICK_HEAD` —— 当前冲突提交

这是一个**单行文件**，保存 40 位的完整 commit hash（无换行或带换行都可能）。它始终指向"因冲突而暂停的那个提交"。

注意它与 sequencer/todo 的关系：
- 对批量 cherry-pick：`CHERRY_PICK_HEAD` 的 hash **应等于** `sequencer/todo` 第一行的缩写 hash（扩展后一致）
- 对单提交 cherry-pick：没有 `sequencer/`，`CHERRY_PICK_HEAD` 是**唯一**的冲突来源标记

对应代码 `pkg/commands/git_commands/status.go` 第 49-79 行 `IsInCherryPick`：

```go
func (self *StatusCommands) IsInCherryPick() (bool, error) {
    exists, _ := self.os.FileExists(filepath.Join(self.repoPaths.WorktreeGitDirPath(), "CHERRY_PICK_HEAD"))
    if !exists { return false, nil }

    cherryPickHead, _ := os.ReadFile(...)
    stoppedSha, err := os.ReadFile(filepath.Join(self.repoPaths.WorktreeGitDirPath(), "rebase-merge", "stopped-sha"))
    if err != nil { return true, nil } // 无 stopped-sha → 确属 cherry-pick

    if strings.HasPrefix(strings.TrimSpace(string(cherryPickHead)),
                          strings.TrimSpace(string(stoppedSha))) {
        return false, nil // hash 一致 → 这是 rebase 留下的"幽灵"
    }
    return true, nil
}
```

**设计原因**：Git 的 rebase 内部实现为一系列 cherry-pick，也可能残留 `CHERRY_PICK_HEAD`。因此需要与 `rebase-merge/stopped-sha` 交叉比对。只有 hash 不一致时才判定为"独立 cherry-pick 状态"。

### 3.4 `.git/sequencer/head` 与 `.git/sequencer/orig-head`

- `head`：cherry-pick 启动时的 HEAD commit hash（用于状态判断）
- `orig-head`：执行 `git cherry-pick --abort` 时 Git 应该回退到的 HEAD（与 head 通常相同，但语义不同）

---

## 四、Continue / Abort / Skip 与 Sequencer 的交互

### 4.1 入口：GenericMergeOrRebaseAction

文件 `pkg/commands/git_commands/rebase.go`：

```go
func (self *RebaseCommands) GenericMergeOrRebaseAction(commandType string, command string) error {
    cmdArgs := NewGitCmd(commandType).Arg("--" + command).ToArgv()
    return self.runSkipEditorCommand(self.cmd.New(cmdArgs))
}
```

根据 `WorkingTreeState` 动态生成：

| 当前状态 | commandType | command | 实际 Git 命令 |
|---------|------------|---------|--------------|
| CherryPicking | `cherry-pick` | `continue` | `git cherry-pick --continue` |
| CherryPicking | `cherry-pick` | `abort` | `git cherry-pick --abort` |
| CherryPicking | `cherry-pick` | `skip` | `git cherry-pick --skip` |
| Rebasing | `rebase` | `continue` | `git rebase --continue` |
| Merging | `merge` | `continue` | `git merge --continue` |

### 4.2 Continue 的内部行为

```
git cherry-pick --continue

  1. 读取 MERGE_MSG 和 CHERRY_PICK_HEAD
  2. 如果已无冲突 → 创建提交（当前阶段提交）
  3. 从 sequencer/todo 中删除第一行 → 追加到 sequencer/done
  4. 如果 todo 已空 → 删除 sequencer/ 目录、删除 CHERRY_PICK_HEAD → 结束
  5. 如果 todo 还有内容 → 尝试 apply todo 下一行的提交：
        ├─ 成功 → 回到步骤 3（循环直到空或冲突）
        └─ 冲突 → 把冲突写入文件，停止并返回非零退出码
                → CHERRY_PICK_HEAD 更新为新的冲突 commit
                → sequencer/todo 保留当前未应用的提交在第一行
```

关键：Continue 不是"一步"，而是**一路冲到下一个冲突或末尾**。所以一次 Continue 可能实际应用了多个后续提交。

### 4.3 Skip 的内部行为

```
git cherry-pick --skip

  1. 丢弃当前 CHERRY_PICK_HEAD 的工作区合并结果
  2. 重置 index
  3. 从 sequencer/todo 第一行移除该提交 → 不追加到 done（跳过，不计入完成）
  4. 回到 Continue 的步骤 4，继续处理下一条 todo
```

Skip 意味着"承认这个提交不要了"，既不应用，也不记录完成。

### 4.4 Abort 的内部行为

```
git cherry-pick --abort

  1. 重置 index 和工作区到 sequencer/orig-head 状态
  2. 删除整个 sequencer/ 目录
  3. 删除 CHERRY_PICK_HEAD 文件
  4. HEAD 指向 cherry-pick 开始前的位置
```

Abort 之后 Git 恢复到 cherry-pick 未执行过的状态。sequencer 文件被整体清除。

### 4.5 runSkipEditorCommand：不打开编辑器的技巧

`pkg/commands/git_commands/rebase.go` 中：

```go
func (self *RebaseCommands) runSkipEditorCommand(cmdObj oscommands.ICmdObj) error {
    return self.setupGitEditorCommand(cmdObj).
        AddEnv(fmt.Sprintf("%s=true", env.ExitImmediatelyEnvKey)).
        Run()
}
```

通过把 lazygit 自身设为 `GIT_EDITOR` 和 `GIT_SEQUENCE_EDITOR`，并设置 `LAZYGIT_EXIT_IMMEDIATELY=true`，Git 每次要打开编辑器时会调起 lazygit（daemon 模式），而 lazygit daemon 识别出该环境变量后立即以 0 退出，模拟"用户什么都没改，保存关闭"。

对应 `pkg/app/daemon/rebase.go`：

```go
func handleInteractiveRebase(common *common.Common, f func(path string) error) error {
    if exitImmediately := env.GetExitImmediately(); exitImmediately {
        common.Log.Info("Lazygit invoked as editor, exiting immediately")
        return nil  // 直接退出 0，Git 认为编辑完成
    }
    // ...正常处理 todo 编辑...
}
```

---

## 五、Lazygit 如何呈现 Sequencer 状态

### 5.1 加载流程：CommitLoader.GetCommits

`pkg/commands/git_commands/commit_loader.go` 第 159-184 行：

```go
// 先去掉所有已有的 TODO 提交（防止刷新后重复）
for i := 0; i < len(commits); i++ {
    if !commit.IsTODO() {
        result = append(result, commits[i:]...)
        break
    }
}

workingTreeState := self.getWorkingTreeState()

addConflictedRebasingCommit := true
if workingTreeState.CherryPicking || workingTreeState.Reverting {
    sequencerCommits, err := self.getHydratedSequencerCommits(hashPool, workingTreeState)
    result = append(sequencerCommits, result...)
    addConflictedRebasingCommit = false  // sequencer 已自带冲突指示
}

if workingTreeState.Rebasing {
    rebasingCommits, _ := self.getHydratedRebasingCommits(hashPool, addConflictedRebasingCommit)
    if len(rebasingCommits) > 0 {
        result = append(rebasingCommits, result...)
    }
}
```

**加载顺序**：
1. 先读 git log 得到正常的本地提交（非 TODO）
2. 若处于 CherryPicking/Reverting → 从 `sequencer/todo` 或 `CHERRY_PICK_HEAD` 合成虚拟提交，**拼到最前面**
3. 若处于 Rebasing → 再把 rebase todo 拼到最前面（这样 rebase 时的 cherry-pick conflict 不会显示两次）

### 5.2 `getHydratedSequencerCommits`：合成虚拟提交

`pkg/commands/git_commands/commit_loader.go` 第 258-275 行：

```go
func (self *CommitLoader) getHydratedSequencerCommits(hashPool *utils.StringPool, workingTreeState models.WorkingTreeState) ([]*models.Commit, error) {
    commits := self.getSequencerCommits(hashPool) // 读 sequencer/todo
    if len(commits) > 0 {
        commits[len(commits)-1].Status = models.StatusConflicted // 最后一个（显示在最顶）是冲突的
    } else {
        // 单提交情况，无 sequencer：从 CHERRY_PICK_HEAD 合成
        conflictedCommit := self.getConflictedSequencerCommit(hashPool, workingTreeState)
        if conflictedCommit != nil {
            commits = append(commits, conflictedCommit)
        }
    }
    return self.getHydratedTodoCommits(hashPool, commits, true)
}
```

**为什么最后一个是冲突的**：因为 `getSequencerCommits` 中用了 `utils.Prepend` 逐行反转。原始 todo 文件第一行（下一条要执行的，即冲突那一条）经反转后落在**切片末尾**，也就是显示在**列表最顶部**（离 HEAD 最近的位置）。

### 5.3 `getHydratedTodoCommits`：水合提交信息

`pkg/commands/git_commands/commit_loader.go` 第 277-334 行：

由于 `sequencer/todo` 中只有缩写 hash 和标题，lazygit 要补充完整元信息（作者、时间戳、标签、完整 40 位 hash）：

```go
// 1. 提取所有 hash
commitHashes := lo.FilterMap(todoCommits, func(commit *models.Commit, _ int) (string, bool) {
    return commit.Hash(), commit.Hash() != ""
})

// 2. 用 git show 一次性查出所有完整信息
cmdObj := self.cmd.New(
    NewGitCmd("show").
        Config("log.showSignature=false").
        Arg("--no-patch", "--oneline", "--abbrev=20", prettyFormat).
        Arg(commitHashes...).
    ToArgv(),
)

// 3. 构造 hash → fullCommit 映射
fullCommits := map[string]*models.Commit{}
cmdObj.RunAndProcessLines(func(line string) (bool, error) {
    commit := self.extractCommitFromLine(hashPool, line[1:], false)
    fullCommits[commit.Hash()] = commit
    return false, nil
})

// 4. 合并：保留 Action/Status，但用完整 commit 的其他字段
for _, rebasingCommit := range todoCommits {
    if commit := findFullCommit(rebasingCommit.Hash()); commit != nil {
        commit.Action = rebasingCommit.Action
        commit.Status = rebasingCommit.Status
        hydratedCommits = append(hydratedCommits, commit)
    }
}
```

**todoFileHasShortHashes=true 的含义**：sequencer/todo 里的 hash 是缩写，需要 `strings.HasPrefix` 匹配；而 rebase 的 `git-rebase-todo` 中可能是完整 hash。

### 5.4 呈现层：presentation/commits.go

虚拟提交的 UI 元素由以下字段决定：

| 字段 | 视觉效果 |
|------|---------|
| `Action != ActionNone` | `IsTODO()` 返回 `true` → 显示 `action` 前缀列（如 cyan 色 `pick`） |
| `Status == StatusCherryPickingOrReverting` | hash 显示为蓝色（`style.FgBlue`） |
| `Status == StatusConflicted` | 行末追加红色 `"<-- Conflict ---"` 标记；action 文字变红色 |
| `commits[0]` 位置 | 放在列表最上方，紧接 HEAD 之后 |

`pkg/gui/presentation/commits.go` 第 425-427 行冲突标记：

```go
if commit.Status == models.StatusConflicted {
    youAreHere := style.FgRed.Sprintf("<-- %s ---", common.Tr.ConflictLabel)
    mark = fmt.Sprintf("%s ", youAreHere)
}
```

以及第 387-393 行 action 列：

```go
if commit.Action != models.ActionNone {
    actionStr := commit.Action.String()
    actionString = actionColorMap(commit.Action, commit.Status).Sprint(actionStr)
}
```

`actionColorMap`（第 515-531 行）定义颜色：

```go
func actionColorMap(action todo.TodoCommand, status models.CommitStatus) style.TextStyle {
    if status == models.StatusConflicted { return style.FgRed }
    switch action {
    case todo.Pick:   return style.FgCyan   // cherry-pick 的 pick 显示为青色
    case todo.Drop:   return style.FgRed
    case todo.Edit:   return style.FgGreen
    case todo.Fixup:  return style.FgMagenta
    default:          return style.FgYellow
    }
}
```

### 5.5 视觉示例

当 cherry-pick 3 个提交（A → B → C，从旧到新），B 发生冲突时，列表顶部显示：

```
  <-- Conflict --- pick  abc1234  B's subject        (青色→红色，冲突标记)
                     pick  def5678  C's subject        (青色，尚未处理)
  HEAD →              pick  9f8e7d6  A's subject        (已完成，在 HEAD)
                     [...]
```

对应 sequencer/todo 内容：
```
pick abc1234... B's subject
pick def5678... C's subject
```
（A 已经在 sequencer/done 中了）

---

## 六、完整状态流程：一次 3 提交 Cherry-Pick 的 Sequencer 生命史

**初始状态**：当前分支 HEAD=X，要 cherry-pick 从旧到新的 A、B、C 三个提交。

### 步骤 1：执行 `git cherry-pick A B C`

Git 创建 sequencer 目录：
```
.git/
 ├── CHERRY_PICK_HEAD  → <hash of A> （正在应用 A）
 └── sequencer/
      ├── todo         → pick A / pick B / pick C
      ├── done         → (空)
      ├── head         → <hash of X>
      └── orig-head    → <hash of X>
```

A 应用**成功**，Git：
1. 把 pick A 从 todo 首行移到 done 末行
2. CHERRY_PICK_HEAD 指向 B
3. 继续尝试 apply B...

### 步骤 2：B 冲突，Git 暂停

```
.git/
 ├── CHERRY_PICK_HEAD  → <hash of B>
 ├── MERGE_MSG         → B's commit message (预填)
 └── sequencer/
      ├── todo         → pick B / pick C  (B 留在首行)
      ├── done         → pick A
      ├── head         → <hash of X>
      └── orig-head    → <hash of X>
```

HEAD 此时指向 **A'**（A 在目标分支上的 cherry-picked 副本）。

Lazygit 展示（按 Prepend 反转顺序）：
```
  pick ... B  → StatusConflicted  <-- Conflict ---
  pick ... C  → StatusCherryPickingOrReverting
HEAD → A'    → StatusUnpushed
```

### 步骤 3：用户解决冲突，点击 Continue

Lazygit 执行 `genericMergeCommand("continue")` → `git cherry-pick --continue`：
1. Git 根据 MERGE_MSG 创建提交 B'
2. 把 pick B 移到 done
3. B 和 C 之间无冲突，C 也顺利应用为 C'
4. todo 变空 → 清除 sequencer 目录、删除 CHERRY_PICK_HEAD

此时 HEAD = C'，所有文件消失：
```
sequencer/         （被删除）
CHERRY_PICK_HEAD   （被删除）
MERGE_MSG          （被删除）
```

**注意**：lazygit 的 `DidPaste` 标记在此路径中**不会**被设置为 `true`。continue 的代码路径是 `merge_and_rebase_helper.go` 的 `genericMergeCommand`，完全不经过 `cherry_pick_helper.go` 中 `Paste` 方法的那段 `if !isInCherryPick` 判断。

### 步骤 3'（替代分支）：用户点击 Abort

Lazygit 执行 `genericMergeCommand("abort")` → `git cherry-pick --abort`：
1. HEAD 回退到 X（orig-head）
2. 工作区和 index 恢复到 X 的状态
3. sequencer/ 和 CHERRY_PICK_HEAD 全部删除

此时 HEAD = X，lazygit 的 CherryPicking 缓冲区仍保留 [A, B, C]（因为 DidPaste 从未设为 true），用户可立即再次粘贴。

### 步骤 3''（替代分支）：B 是空提交，自动 Skip

`CheckMergeOrRebaseWithRefreshOptions` 识别错误信息：
- `"The previous cherry-pick is now empty"` → 自动走 `genericMergeCommand("skip")` → B 被跳过

Git 行为：
1. B **不**加入 done，也不创建提交
2. 继续处理 C
3. 最终 sequencer/done 里只有 A，没有 B

---

## 七、与 Interactive Rebase 的 Todo 机制对比

| 维度 | Cherry-Pick Sequencer | Interactive Rebase |
|------|----------------------|--------------------|
| 目录 | `.git/sequencer/` | `.git/rebase-merge/` |
| Todo 文件 | `.git/sequencer/todo` | `rebase-merge/git-rebase-todo` |
| Done 文件 | `.git/sequencer/done` | `rebase-merge/done` |
| 当前暂停位置 | `CHERRY_PICK_HEAD`（独立文件） | `rebase-merge/stopped-sha` |
| 命令类型 | `pick`、`revert` 固定 | `pick`、`drop`、`reword`、`fixup`、`squash`、`edit`、`exec`、`update-ref`、`label`、`reset`、`merge` 等 |
| 中途编辑 todo | **不支持**（lazygit 中 cherry-pick 不能动态删改） | 支持（lazygit 可直接重排、drop 等） |
| 冲突提交的判断 | sequencer/todo 非空 → 最后一行是冲突；否则读 CHERRY_PICK_HEAD | done 最后一条 ≠ todo 第一条 → done 最后一条是冲突 |

这也是为什么 lazygit 在 cherry-pick 模式下禁用了 TODO 条目编辑（`NotAllowedMidCherryPickOrRevert` 错误信息）：Git 的 sequencer 机制不支持交互式修改，不像 rebase 有 `--edit-todo` 子命令。

---

## 八、关键设计总结

### 8.1 两处"提交列表"的区别

| | CherryPicking.CherryPickedCommits（内存） | .git/sequencer/todo（磁盘） |
|---|--|--|
| 存在期 | 从用户按 Copy 到下一次覆盖/Reset | 从批量 cherry-pick 启动到完成/abort |
| 内容 | 全部选中的提交（无论是否已应用） | 剩余**待应用**的提交（已完成的在 done） |
| 顺序 | 按源列表从上到下（新→旧） | 按执行顺序从上到下（旧→新） |
| 冲突时 | 不变 | 第一行固定是冲突提交 |
| DidPaste=true 时 | 仍保留，但视觉不显示 | 不存在 |
| abort 后 | 仍保留，便于再次 paste | 被删除 |

**为什么需要两份**：
- 内存中那份是用户的"选择意图"——即便 Git 执行失败，用户也无需重新复制
- 磁盘上那份是 Git 的"执行进度"——Git 根据它恢复 continue 的起点

### 8.2 冲突提交定位的双重策略

Lazygit 对"当前冲突的是哪个提交"有三种定位方式，层层兜底：

```
有 .git/sequencer/todo 文件？
    ├─ 有 → todo 最后一行（经 Prepend 反转）就是冲突提交
    └─ 无 → 读 CHERRY_PICK_HEAD / REVERT_HEAD 合成单条
              └─ 也没有？→ 什么都不显示
```

rebase 用的是 done/todo 交叉比对法，更复杂（要处理 Git 重新排期的边缘情况）。

### 8.3 Abort 后再 Paste 的无缝体验

整个流程设计的核心体验目标：**abort 后用户无需重新 Copy 就能立即再次 Paste**。

实现机制就是 6.1 节的"双列表"——cherry-pick 缓冲区（内存）与 sequencer 文件（磁盘）完全解耦。Git 的 abort 只清除磁盘文件，不动 lazygit 的内存数据结构。只有当用户主动按 Esc（ResetCherryPick）、或切换 context 后再次 Copy 时，内存缓冲区才会被清空。

---

## 九、代码位置速查

| 功能 | 文件（仓库相对路径） | 关键函数 |
|------|---------------------|---------|
| 读取 sequencer/todo | `pkg/commands/git_commands/commit_loader.go` | `getSequencerCommits` |
| 读取 CHERRY_PICK_HEAD 合成冲突提交 | 同上 | `getConflictedSequencerCommit` |
| 水合 TODO 提交（git show 补全信息） | 同上 | `getHydratedTodoCommits` / `getHydratedSequencerCommits` |
| 判断 CHERRY_PICK_HEAD 是否属于独立 cherry-pick | `pkg/commands/git_commands/status.go` | `IsInCherryPick` |
| 判断 rebase 冲突提交 | `pkg/commands/git_commands/commit_loader.go` | `getConflictedCommit` / `getConflictedCommitImpl` |
| 执行 --continue/--abort/--skip | `pkg/commands/git_commands/rebase.go` | `GenericMergeOrRebaseAction` / `runSkipEditorCommand` |
| 统一调用上述动作 | `pkg/gui/controllers/helpers/merge_and_rebase_helper.go` | `genericMergeCommand` |
| 检测冲突清零后提示 Continue | `pkg/gui/controllers/helpers/refresh_helper.go` | `refreshStateFiles` 内联逻辑 |
| 显示 TODO 的 pick 前缀和冲突标记 | `pkg/gui/presentation/commits.go` | `displayCommit` / `actionColorMap` |
| 解析/写入 todo 文件格式 | `pkg/utils/rebase_todo.go` | `ReadRebaseTodoFile` / `WriteRebaseTodoFile` |
| 作为 GIT_SEQUENCE_EDITOR 被 Git 调起 | `pkg/app/daemon/rebase.go` | `handleInteractiveRebase` |
