# Cherry-Pick Sequencer 文件结构与状态演变详解

> **本文专注于 cherry-pick 机制，不涉及 rebase 的 done/orig-head 等文件。**
>
> cherry-pick 与 rebase 虽然共享部分底层代码（sequencer 框架），但它们使用不同的目录结构和文件命名：cherry-pick 使用 `.git/sequencer/`，rebase 使用 `.git/rebase-merge/`。请不要将两者混淆。

---

## 一、Cherry-Pick 涉及的所有 .git 文件

执行批量 cherry-pick（≥2 个提交）时，Git 会在 `.git` 目录下创建以下文件：

```
.git/
 ├── CHERRY_PICK_HEAD          ← 当前正在应用（或冲突暂停）的提交 hash（全量 40 位）
 ├── MERGE_MSG                 ← 预填的提交信息（冲突解决后 continue 时自动带入）
 └── sequencer/
      ├── todo                 ← 剩余待处理的提交队列（从旧到新，pick 命令格式）
      ├── head                 ← cherry-pick 启动时的 HEAD commit hash
      └── abort-safety         ← 防止跨 worktree 误 abort 的防护标记
```

**注意**：
- cherry-pick 的 sequencer 目录 **不包含** `done` 和 `orig-head` 文件（那是 rebase 的）。
- 单提交 cherry-pick **完全不创建 `sequencer/` 目录**，只写 `CHERRY_PICK_HEAD` 和 `MERGE_MSG`。

下面逐个文件详细说明。

---

## 二、逐文件详解

### 2.1 `.git/CHERRY_PICK_HEAD` —— 当前操作提交

**存在时机**：所有 cherry-pick 过程中（无论单提交还是批量，无论正常进行还是冲突暂停）。

**内容**：一行 40 位完整 SHA-1 hash，末尾可能带或不带换行符。

```
abc1234def567890abc1234def567890abc1234d
```

**语义**：指向"正在被 cherry-pick 的那个提交"。
- 在成功应用的过程中，它短暂指向当前正在 apply 的提交
- 在冲突暂停时，它指向"因冲突而失败的那个提交"
- 在 `--continue` 成功提交之后、下一个提交开始之前，它被立即更新为下一个目标提交

**代码对应**：`pkg/commands/git_commands/status.go` 第 49-79 行 `IsInCherryPick` 通过该文件是否存在来判断是否处于 cherry-pick 状态：

```go
func (self *StatusCommands) IsInCherryPick() (bool, error) {
    exists, _ := self.os.FileExists(
        filepath.Join(self.repoPaths.WorktreeGitDirPath(), "CHERRY_PICK_HEAD"))
    if !exists { return false, nil }
    // ...与 rebase-merge/stopped-sha 交叉比对，排除 rebase 残留...
    return true, nil
}
```

以及 `pkg/commands/git_commands/commit_loader.go` 第 515-542 行 `getConflictedSequencerCommit`，在 sequencer/todo 不存在（即单提交 cherry-pick）时，直接从该文件合成冲突提交的 UI 显示：

```go
func (self *CommitLoader) getConflictedSequencerCommit(
    hashPool *utils.StringPool,
    workingTreeState models.WorkingTreeState,
) *models.Commit {
    shaFile = "CHERRY_PICK_HEAD"
    bytesContent, _ := self.readFile(
        filepath.Join(self.repoPaths.WorktreeGitDirPath(), shaFile))
    lines := strings.Split(string(bytesContent), "\n")
    return models.NewCommit(hashPool, models.NewCommitOpts{
        Hash:   lines[0],
        Status: models.StatusConflicted,
        Action: todo.Pick,
    })
}
```

### 2.2 `.git/sequencer/todo` —— 剩余待处理队列

**存在时机**：批量 cherry-pick（≥2 提交）期间。单提交时不存在。

**格式**：每行一条 `pick` 指令，顺序即执行顺序（旧→新）：

```
pick abc1234 commit subject for A
pick def5678 commit subject for B
pick 9f8e7d6 commit subject for C
```

格式由 `git-todo-parser/todo` 库解析，结构为：

```go
type Todo struct {
    Command   TodoCommand  // pick / revert / ...
    Commit    string       // 缩写 hash（够唯一即可，通常 7 位）
    Flag      string       // 如 "-C" 用于 fixup -C
    Msg       string       // 提交标题（可含空格）
    Ref       string       // update-ref 用，cherry-pick 中为空
}
```

**与 CHERRY_PICK_HEAD 的一致性**：
- 冲突暂停时，`sequencer/todo` **第一行**的 hash（扩展为 40 位后）应当等于 `CHERRY_PICK_HEAD` 文件的内容
- 这是 Git 保证的：todo 首行 = "正在尝试但尚未完成"的那个提交

**代码对应**：`pkg/commands/git_commands/commit_loader.go` 第 483-513 行 `getSequencerCommits`：

```go
func (self *CommitLoader) getSequencerCommits(hashPool *utils.StringPool) []*models.Commit {
    bytesContent, _ := self.readFile(
        filepath.Join(self.repoPaths.WorktreeGitDirPath(), "sequencer/todo"))
    todos, _ := todo.Parse(bytes.NewBuffer(bytesContent), self.config.GetCoreCommentChar())

    commits := []*models.Commit{}
    for _, t := range todos {
        // 用 Prepend 逐行反转：todo 首行 → 列表末尾（显示在最顶）
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

**冲突标记定位**：`getHydratedSequencerCommits`（同文件第 258-275 行）把反转后列表的**最后一项**（即原 todo 第一行）标记为冲突：

```go
if len(commits) > 0 {
    // sequencer/todo 非空 → 最后一个（显示在最顶）是冲突的
    commits[len(commits)-1].Status = models.StatusConflicted
}
```

### 2.3 `.git/sequencer/head` —— 起点 HEAD

**存在时机**：批量 cherry-pick 期间。

**内容**：一行 40 位 SHA-1，即 cherry-pick 命令执行**前** `HEAD` 指向的 commit hash。

**作用**：
1. 记录"本次批量 cherry-pick 是基于哪个 commit 开始的"
2. 用于恢复工作区和 index 到 cherry-pick 启动前的状态（虽然 abort 时具体读的是其他机制）
3. 用于检测某些"操作被中断过"的异常情况

**与 CHERRY_PICK_HEAD 的区别**：

| 文件 | 指向 | 何时变化 |
|------|------|---------|
| `sequencer/head` | cherry-pick **启动时**的 HEAD | 整个批量 cherry-pick 期间**不变** |
| `CHERRY_PICK_HEAD` | 当前**正在拣**的源提交 | 每成功应用一个就变一次（指向 todo 下一条） |

举例：当前分支 HEAD=X，要 cherry-pick A、B、C 三个提交。
- `sequencer/head` 始终是 X 的 hash
- `CHERRY_PICK_HEAD` 依次指向 A → B → C

### 2.4 `.git/sequencer/abort-safety` —— 跨 Worktree 防护

**存在时机**：批量 cherry-pick 期间。

**内容**：通常是一行文本，记录 worktree 的路径或一个唯一标识。

**作用**：防止用户在 worktree A 中启动 cherry-pick，然后在 worktree B（同一个 repo 的另一个工作树）中执行 `git cherry-pick --abort`。如果没有这个防护，B 的 abort 会错误地清除 A 正在进行的 cherry-pick 状态。

当执行 `git cherry-pick --abort` 时，Git 会：
1. 读取 `abort-safety` 文件
2. 检查它是否与当前 worktree 匹配
3. 如果不匹配 → 拒绝 abort 并报错

**lazygit 中的直接引用**：lazygit 代码中**没有直接读写该文件**，所有校验都由 Git 本身在执行 `git cherry-pick --abort` 时完成。lazygit 只需调用 `git cherry-pick --abort`，如果跨 worktree 误用，Git 会返回错误，lazygit 再通过 `CheckMergeOrRebase` 将错误呈现给用户。

---

## 三、MERGE_MSG：不属 sequencer 目录但不可或缺

**路径**：`.git/MERGE_MSG`

**存在时机**：cherry-pick 冲突暂停时。

**内容**：当前冲突提交的完整 commit message（含 subject 和 body），外加一些注释行（以 `#` 开头）：

```
commit B's subject line

Here is the detailed body of commit B explaining
why this change was made.

# Conflicts:
#   file.txt
```

**作用**：当用户解决完冲突，执行 `git cherry-pick --continue` 时，Git 自动读取该文件作为新提交的 commit message，用户无需重新输入。

如果该文件不存在或为空，`--continue` 会要求用户提供提交信息（通常通过打开编辑器）。

---

## 四、完整状态演变：3 提交 Cherry-Pick 的生命周期

设定场景：当前分支 HEAD = X（基础提交），要依次 cherry-pick A（旧）→ B（中）→ C（新）三个提交。其中 B 会发生冲突。

### 阶段 0：执行前

```
.git 目录：
  （无上述任何文件）

lazygit CherryPicking 缓冲区：
  [A, B, C]   ContextKey=<来源 context>   DidPaste=false
```

### 阶段 1：`git cherry-pick A B C` 刚启动

```
.git/
 ├── CHERRY_PICK_HEAD    → <hash of A>  （正在应用 A）
 ├── MERGE_MSG           → subject & body of A
 └── sequencer/
      ├── todo           → pick A / pick B / pick C  （全部三条都在）
      ├── head           → <hash of X>  （启动时的 HEAD，全程不变）
      └── abort-safety   → <当前 worktree 标识>
```

A 可以干净应用，Git 继续内部流程，不暂停。

### 阶段 2：A 应用成功，开始应用 B，B 冲突 → 暂停

```
.git/
 ├── CHERRY_PICK_HEAD    → <hash of B>  （冲突的是 B）
 ├── MERGE_MSG           → subject & body of B  （B 的提交信息）
 └── sequencer/
      ├── todo           → pick B / pick C  （A 已被移除）
      ├── head           → <hash of X>  （不变）
      └── abort-safety   → <标识>        （不变）

HEAD 指向 A'（A 在目标分支上的副本）。

lazygit 展示：
  pick ... B → StatusConflicted  <-- Conflict ---  （红色）
  pick ... C → StatusCherryPickingOrReverting       （青色）
HEAD → A'
```

**关键观察**：
- A 已成功应用 → **从 todo 消失**（cherry-pick sequencer 没有 done 文件！）
- todo 首行 = B，与 CHERRY_PICK_HEAD 一致
- sequencer/head 仍是 X（全程不变）

### 阶段 3：用户解决冲突 → `git cherry-pick --continue`

Git 内部行为：

```
1. 读取 MERGE_MSG → 作为 B' 的提交信息
2. 创建提交 B'  → HEAD 指向 B'
3. 从 sequencer/todo 首行移除 pick B
4. CHERRY_PICK_HEAD 指向 C
5. MERGE_MSG 覆盖为 C 的提交信息
6. 尝试 apply C...
```

假设 C 也能干净应用：

```
7. 创建提交 C'  → HEAD 指向 C'
8. 从 sequencer/todo 首行移除 pick C
9. todo 变空 → 触发"完成"清理：
     删除 sequencer/ 整个目录
     删除 CHERRY_PICK_HEAD
     删除 MERGE_MSG
```

**最终状态（阶段 3 后）**：

```
.git 目录：
  sequencer/          （已删除）
  CHERRY_PICK_HEAD    （已删除）
  MERGE_MSG           （已删除）

HEAD → C'

lazygit CherryPicking 缓冲区：
  [A, B, C]   ContextKey=<来源>   DidPaste=false
    ↑
    注意：DidPaste 仍然是 false！因为 continue 路径不经过 Paste() 方法中的
    那段 if !isInCherryPick { DidPaste=true } 判断。这是已知的设计取舍。
```

### 阶段 3'：替代分支 —— 用户点击 Abort

```
git cherry-pick --abort

1. 读取 sequencer/head（即 X）
2. 重置 index 和工作区到 X 的状态
3. HEAD 回到 X
4. 清理文件：
     删除 sequencer/ 整个目录
     删除 CHERRY_PICK_HEAD
     删除 MERGE_MSG
```

**Abort 后状态**：

```
.git 目录：
  sequencer/          （已删除）
  CHERRY_PICK_HEAD    （已删除）
  MERGE_MSG           （已删除）

HEAD → X   （回到 cherry-pick 启动前）

lazygit CherryPicking 缓冲区：
  [A, B, C]   ContextKey=<来源>   DidPaste=false
    ↑
    完全不变！因为 abort 只动 Git 的磁盘文件，不碰 lazygit 的内存结构。
    用户可以立即再次按 Paste 重试，无需重新 Copy。
```

### 阶段 3''：替代分支 —— B 是空提交，自动 Skip

`pkg/gui/controllers/helpers/merge_and_rebase_helper.go` 中 `CheckMergeOrRebaseWithRefreshOptions` 检测到特定错误信息后自动触发：

```go
if strings.Contains(result.Error(), "The previous cherry-pick is now empty") {
    return self.genericMergeCommand("skip")
}
```

```
git cherry-pick --skip

1. 丢弃 B 的工作区合并结果
2. 重置 index
3. 从 sequencer/todo 首行移除 pick B  （注意：没有 done 文件，就直接丢弃）
4. CHERRY_PICK_HEAD 指向 C
5. 继续执行后续的 C（回到阶段 2-3 的循环）
```

Skip 后 todo 只有 pick C，B 被永久跳过。

---

## 五、各文件在关键操作下的变化一览

| 操作 | sequencer/todo | sequencer/head | abort-safety | CHERRY_PICK_HEAD | MERGE_MSG | HEAD |
|------|---|---|---|---|---|---|
| 启动批量 cp | 写入全部 A/B/C | 写入 X | 写入标识 | 写入 A | 写入 A 的 msg | X |
| A 成功 | 删首行 → B/C | 不变 | 不变 | → B | → B 的 msg | → A' |
| B 冲突暂停 | B/C（B 在首行） | 不变 | 不变 | → B | → B 的 msg | → A' |
| 解决后 continue | 删首行 → C；然后 apply C 成功 → 空 | 不变 | 不变 | → C；然后被删除 | → C 的 msg；然后被删除 | → B' → C' |
| 全部完成 | （文件随目录删除） | （删除） | （删除） | （删除） | （删除） | C' |
| 冲突时 abort | （删除） | （删除） | （删除） | （删除） | （删除） | → X |
| 冲突时 skip B | 删首行 → C | 不变 | 不变 | → C | → C 的 msg | → A'（不变） |
| 单提交 cp 冲突 | （不存在） | （不存在） | （不存在） | 写入该提交 | 写入该提交 msg | 当前 HEAD |

---

## 六、Lazygit 如何消费这些文件

### 6.1 判断是否处于 cherry-pick 状态

只看 `CHERRY_PICK_HEAD` 是否存在（`pkg/commands/git_commands/status.go` 第 49-79 行）。但为了排除 rebase 残留的"幽灵"CHERRY_PICK_HEAD，还要交叉检查 `rebase-merge/stopped-sha`：

```
CHERRY_PICK_HEAD 存在？
  ├─ 不存在 → 不是 cherry-pick
  └─ 存在 → 读 rebase-merge/stopped-sha：
       ├─ 文件不存在 → 是独立的 cherry-pick
       └─ 文件存在 → 比较 hash：
            ├─ 相同 → 是 rebase 的残留，不是 cherry-pick
            └─ 不同 → 是独立的 cherry-pick
```

### 6.2 合成待办提交的 UI 显示

`pkg/commands/git_commands/commit_loader.go`：

```
sequencer/todo 文件存在？
  ├─ 存在 → 逐行解析，用 Prepend 反转顺序 → 最后一条标记 StatusConflicted
  └─ 不存在 → 是单提交 cherry-pick：
       └─ 读 CHERRY_PICK_HEAD → 合成一条 StatusConflicted 的虚拟提交
```

水合（hydration）：sequencer/todo 里只有缩写 hash 和标题，需要用 `git show` 查出完整元信息（作者、时间戳、标签、完整 40 位 hash）。由于 hash 是缩写，匹配时必须用 `strings.HasPrefix`（`todoFileHasShortHashes=true` 分支）。

### 6.3 Continue / Abort / Skip 的执行

`pkg/gui/controllers/helpers/merge_and_rebase_helper.go` 的 `genericMergeCommand` 根据 `WorkingTreeState.CommandName()` 动态决定命令前缀是 `cherry-pick` 还是 `rebase` 等，然后：

```go
func (self *RebaseCommands) GenericMergeOrRebaseAction(commandType string, command string) error {
    // 对 cherry-pick：commandType="cherry-pick"，command="continue"|"abort"|"skip"
    cmdArgs := NewGitCmd(commandType).Arg("--" + command).ToArgv()
    return self.runSkipEditorCommand(self.cmd.New(cmdArgs))
}
```

`runSkipEditorCommand` 通过将 lazygit 自身设为 `GIT_EDITOR` / `GIT_SEQUENCE_EDITOR` 并设置 `LAZYGIT_EXIT_IMMEDIATELY=true`，使 Git 在需要打开编辑器时立即"空保存退出"，无需用户交互。

### 6.4 自动提示 Continue

`pkg/gui/controllers/helpers/refresh_helper.go` 的 `refreshStateFiles` 在每次刷新文件列表时检测：

```
冲突文件数 从 >0 变为 0  &&  WorkingTreeState.Any()
  → 自动弹出 PromptToContinueRebase()
```

该提示里还会检查是否有未暂存文件，如有则提示用户是否自动 stage，然后调用 `genericMergeCommand("continue")`。

---

## 七、两个"列表"的解耦设计

lazygit 中有**两份独立的 cherry-pick 提交列表**，它们的生命周期完全不同：

| | `CherryPicking.CherryPickedCommits`（内存） | `.git/sequencer/todo`（磁盘） |
|---|---|---|
| 归属 | lazygit 内存状态 | Git 磁盘状态 |
| 内容 | 用户复制的**全部**提交 | Git 剩余**待处理**的提交 |
| 创建于 | 用户按 Copy | 用户按 Paste 并触发 Git 命令 |
| 冲突时 | **完全不变** | 只保留未处理的提交（首行是冲突的） |
| Continue 全部成功 | **仍保留**（DidPaste 仍为 false，已知取舍） | 随 sequencer 目录删除 |
| Abort 后 | **仍保留**，可立即再次 Paste | 被删除 |
| 用户按 Esc | 清空 | 不变 |

**为什么要分两份**：
- 内存那份代表用户的**选择意图**——即便 Git 操作失败，也不需要重新选择
- 磁盘那份代表 Git 的**执行进度**——Git 根据它恢复 `--continue` 的起点

这种解耦实现了 **"abort 后无需重新 Copy 即可再次 Paste"** 的无缝体验。集成测试 `cherry_pick_conflicts.go` 第 59-61 行明确验证了此行为：

```go
// cherry pick selection is not cleared when there are conflicts, so that the user
// is able to abort and try again without having to re-copy the commits
t.Views().Information().Content(Contains("2 commits copied"))
```

---

## 八、代码位置速查（仅 cherry-pick 相关）

| 功能 | 文件（仓库相对路径） | 关键函数 |
|------|---------------------|---------|
| 判断是否处于 cherry-pick 状态 | `pkg/commands/git_commands/status.go` | `IsInCherryPick` |
| 从 CHERRY_PICK_HEAD 合成单提交冲突显示 | `pkg/commands/git_commands/commit_loader.go` | `getConflictedSequencerCommit` |
| 读取 sequencer/todo 合成批量显示 | 同上 | `getSequencerCommits` |
| 水合 todo 提交（git show 补全信息） | 同上 | `getHydratedTodoCommits` / `getHydratedSequencerCommits` |
| 启动 cherry-pick 批量命令 | `pkg/commands/git_commands/rebase.go` | `CherryPickCommits` |
| 执行 --continue/--abort/--skip | 同上 | `GenericMergeOrRebaseAction` / `runSkipEditorCommand` |
| 检测冲突清零后提示 Continue | `pkg/gui/controllers/helpers/refresh_helper.go` | `refreshStateFiles` 内联逻辑 |
| 统一 Continue/Abort/Skip 分发 | `pkg/gui/controllers/helpers/merge_and_rebase_helper.go` | `genericMergeCommand` / `CheckMergeOrRebaseWithRefreshOptions` |
| Paste 入口（含 Auto-Stash 和 DidPaste 设置） | `pkg/gui/controllers/helpers/cherry_pick_helper.go` | `Paste` |
| 呈现 TODO 的 pick 前缀和冲突标记 | `pkg/gui/presentation/commits.go` | `displayCommit` / `actionColorMap` |
| 内存缓冲区管理 | `pkg/gui/modes/cherrypicking/cherry_picking.go` | `Add` / `Remove` / `update` / `Active` |
