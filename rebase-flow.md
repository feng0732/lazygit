# 交互式 Rebase 代码流程梳理

## 概述

Lazygit 的交互式 rebase 系统围绕 **待办列表 (Todo List)**、**命令触发**、**冲突回退** 三个核心概念构建。整个系统通过 Git 的 `GIT_SEQUENCE_EDITOR` 机制切入，使用 lazygit 自身作为 daemon 进程来编辑 todo 文件，从而实现程序化的 rebase 操作。

---

## 一、核心文件索引

> 以下路径均为相对于仓库根目录的相对路径。

| 模块 | 文件 | 职责 |
|------|------|------|
| Git 命令层 | [pkg/commands/git_commands/rebase.go](pkg/commands/git_commands/rebase.go) | 构建 rebase 命令、封装 continue/abort/skip |
| 待办列表工具 | [pkg/utils/rebase_todo.go](pkg/utils/rebase_todo.go) | 读写 git-rebase-todo 文件、移动/删除 todo 项 |
| Daemon 层 | [pkg/app/daemon/daemon.go](pkg/app/daemon/daemon.go) + [pkg/app/daemon/rebase.go](pkg/app/daemon/rebase.go) | 作为 Git 的编辑器被调用，执行具体的 todo 修改指令 |
| GUI 控制层 | [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go) | rebase 选项菜单、冲突检测与处理、continue/abort 调度 |
| 提交加载器 | [pkg/commands/git_commands/commit_loader.go](pkg/commands/git_commands/commit_loader.go) | 从 git-rebase-todo 加载待办项并与真实提交合并展示 |
| 工作树状态 | [pkg/commands/models/working_tree_state.go](pkg/commands/models/working_tree_state.go) + [pkg/commands/git_commands/status.go](pkg/commands/git_commands/status.go) | 检测当前是否处于 rebase/merge/cherry-pick 状态 |
| 提交模型 | [pkg/commands/models/commit.go](pkg/commands/models/commit.go) | Commit 数据结构，包含 Action/Status 等 todo 相关字段 |

---

## 二、待办列表 (Todo List) 编排

### 2.1 Todo 文件的位置与格式

Git 在交互式 rebase 进行中会在 `.git/rebase-merge/` 目录下维护一组状态文件：

| 文件 | 作用 |
|------|------|
| `git-rebase-todo` | **待执行**的 todo 列表 |
| `done` | **已执行完成**的 todo 列表 |
| `stopped-sha` | 当前停止处的提交 SHA（缩写） |
| `amend` | edit 模式标记（存在表示 edit 命令已开始） |
| `message` | edit/reword 模式的提交信息文件 |
| `head-name` | 正在被 rebase 的分支全名 |

Todo 格式由第三方库 `git-todo-parser/todo` 解析，每条记录包含：

- `Command`：操作类型（pick / reword / edit / squash / fixup / exec / break / label / reset / merge / update-ref 等）
- `Commit`：提交哈希
- `Flag`：如 `-C`（fixup -C 保留提交信息）
- `Msg`：提交信息或 exec 命令内容
- `Ref`：update-ref 的目标引用
- `Label`：label/reset 的标签名
- `ExecCommand`：exec 命令内容

### 2.2 Todo 列表的加载与展示

在 [commit_loader.go](pkg/commands/git_commands/commit_loader.go) 的 `MergeRebasingCommits` 方法（约第 155 行）中，todo 列表与真实提交被融合展示：

1. **过滤旧 TODO**：先移除已有的 TODO 类型提交，避免重复
2. **检测状态**：检查 `WorkingTreeState`：
   - CherryPicking / Reverting 状态：从 `.git/sequencer/todo` 加载
   - Rebasing 状态：从 `.git/rebase-merge/git-rebase-todo` 加载
3. **水化 (Hydrate)**：todo 文件中只有短哈希和简要信息，通过 `git show` 获取提交的完整信息（作者、日期、完整信息等）
4. **倒序插入**：由于 git log 是"新的在上、旧的在下"，而 todo 文件是"旧的在上、新的在下"，因此 todo 项需要**倒序**插入到真实提交列表的**前面**

#### 冲突提交的识别

[getConflictedCommit](pkg/commands/git_commands/commit_loader.go)（约第 386 行）负责识别当前正在冲突的提交：

- 解析 `rebase-merge/done` 文件，取最后一条记录
- 如果最后一条是 `break` / `exec` / `reword` → 不是冲突，是正常停止
- 如果 done 的最后一条 == todo 的第一条 → 命令被 Git 重新调度（rescheduled），不算冲突
- 对于 `edit` 命令：如果 `amend` 文件存在 → edit 成功执行，不是冲突
- 其他有 commit hash 的 todo → 认为发生了冲突，标记为 `StatusConflicted`

> **Rescheduled 机制**：当一个命令因冲突失败时，Git 会把它重新放回 todo 列表的开头，等待重试。这时候 done 的最后一条和 todo 的第一条是相同的。代码中需要检测这种情况，避免把同一个提交既显示在"已完成"又显示在"待执行"。

### 2.3 Todo 的修改操作

[rebase_todo.go](pkg/utils/rebase_todo.go) 提供了多种原子操作：

| 函数 | 用途 | 触发时机 |
|------|------|----------|
| `EditRebaseTodo` | 修改指定 commit 的 action（pick→edit 等） | reword / edit / squash / fixup / drop |
| `MoveTodosUp` / `MoveTodosDown` | 移动 todo 项（上/下） | 用户按上下键移动提交 |
| `DeleteTodos` | 删除 todo 项 | 删除 update-ref todo |
| `MoveFixupCommitDown` | 移动 fixup 提交到目标提交下方 | amend / fixup 操作 |
| `DropMergeCommit` | 删除合并提交的 todo 项 | 删除 merge commit |
| `PrependStrToTodoFile` | 在 todo 文件开头插入内容 | insert break |

#### 移动逻辑的关键细节

- **方向反转**：Git 的 todo 文件自上而下执行（旧的在上），但 Lazygit 提交视图自下而上展示（新的在上）。因此 UI 上的"向下移动"对应 todo 列表中向**起始方向**移动。
- **跳过不可见项**：移动时只在 `isRenderedTodo` 之间跳跃，label / reset / comment 等不可见项会被自动跳过。
- **批量移动**：`moveTodosUp` 会逐个移动每个 todo 项，保持它们之间的相对顺序。

### 2.4 已渲染 Todo 的判定

[isRenderedTodo](pkg/utils/rebase_todo.go)（约第 292 行）决定哪些 todo 项会显示在 UI 中：

```go
func isRenderedTodo(t todo.Todo, isInRebase bool) bool {
    return t.Commit != "" || (isInRebase && (t.Command == todo.UpdateRef || t.Command == todo.Exec))
}
```

即：
- 有 commit hash 的 → 显示（pick / reword / edit / squash / fixup / drop / merge 等）
- 在 rebase 中且是 `update-ref` → 显示
- 在 rebase 中且是 `exec` → 显示
- 其他（label / reset / comment / break）→ 不显示

---

## 三、命令触发机制

### 3.1 Daemon 模式：Lazygit 作为 Git 的编辑器

Lazygit 通过将自身设置为 `GIT_SEQUENCE_EDITOR` 和 `GIT_EDITOR` 环境变量，在 Git 需要编辑 todo 文件或提交信息时被回调调用。这就是所谓的 "daemon 模式" —— 虽然名字叫 daemon，但实际上是一次性的子进程调用。

**指令传递方式**：通过环境变量传递 JSON 序列化的指令
- `LAZYGIT_DAEMON_KIND`：daemon 类型枚举（整数）
- `LAZYGIT_DAEMON_INSTRUCTION`：JSON 格式的指令参数

[daemon.go](pkg/app/daemon/daemon.go) 中定义了 9 种 daemon 类型：

| 类型 | 用途 |
|------|------|
| `ExitImmediately` | 直接退出（跳过编辑器，无需编辑） |
| `RemoveUpdateRefsForCopiedBranch` | 移除复制分支末尾的 update-ref todo |
| `ChangeTodoActions` | 批量修改 todo 的 action |
| `DropMergeCommit` | 删除合并提交的 todo |
| `MoveFixupCommitDown` | 移动 fixup 提交到目标下方 |
| `MoveTodosUp` / `MoveTodosDown` | 移动 todo 项 |
| `InsertBreak` | 在 todo 列表开头插入 break |
| `WriteRebaseTodo` | 写入完整的 todo 文件内容 |

### 3.2 启动交互式 Rebase 的完整流程

以 `InteractiveRebase`（修改多个提交的 action）为例，完整调用链路：

```
用户操作 (local_commits_controller)
    ↓
interactiveRebase(action, startIdx, endIdx)
    ↓
RebaseCommands.InteractiveRebase  [rebase.go]
    ├─ 计算 baseHashOrRoot
    │   (endIdx+1 作为基准，squash/fixup 则 +2，
    │    因为要 squash 到下一个提交，需要多包含一个)
    ├─ 构造 ChangeTodoAction 列表
    │   (每个选中的 commit 对应一个 action 变更)
    └─ PrepareInteractiveRebaseCommand
        ├─ 构造 git rebase 命令参数
        │   ├─ --interactive
        │   ├─ --autostash           (自动暂存改动)
        │   ├─ --keep-empty          (保留空提交)
        │   ├─ --no-autosquash       (不自动处理 fixup)
        │   └─ --rebase-merges       (保留合并提交)
        ├─ 设置 GIT_SEQUENCE_EDITOR = lazygit 路径
        ├─ 设置 GIT_EDITOR = lazygit (overrideEditor 时)
        ├─ 通过环境变量注入 daemon instruction
        └─ 执行 git 命令
            ↓
Git 启动交互式 rebase
    ↓
Git 调用 $GIT_SEQUENCE_EDITOR（即 lazygit daemon）
    ↓
daemon.Handle() 被触发  [daemon.go]
    ├─ 从环境变量解析 instruction
    └─ instruction.run()
        └─ handleInteractiveRebase  [daemon/rebase.go]
            ├─ 检查命令行参数是不是 git-rebase-todo 文件
            ├─ 执行具体的 todo 修改逻辑
            └─ daemon 进程退出
                ↓
Git 读取修改后的 todo 文件，继续执行 rebase
```

### 3.3 Rebase 进行中修改 Todo

当 rebase 已经在进行中（stopped 状态，比如遇到冲突或 edit 命令），可以直接修改 `git-rebase-todo` 文件而无需重新启动 rebase。

相关方法在 [rebase.go](pkg/commands/git_commands/rebase.go) 约第 338-387 行：

- `EditRebaseTodo`：直接修改 todo 文件中指定提交的 action
- `MoveTodosUp` / `MoveTodosDown`：直接移动 todo 项
- `DeleteUpdateRefTodos`：删除 update-ref todo（需要通过 `git rebase --edit-todo` 让 Git 做内部 housekeeping）

> 注意：删除 update-ref todo 不能直接改文件，必须走 `git rebase --edit-todo` 命令，让 Git 处理相关的内部状态更新。

### 3.4 Continue / Abort / Skip

[GenericMergeOrRebaseAction](pkg/commands/git_commands/rebase.go)（约第 482 行）是 continue/abort/skip 的统一入口：

```go
continue → git rebase --continue
abort    → git rebase --abort
skip     → git rebase --skip
```

#### onSuccessfulContinue 回调机制

有些操作需要在 rebase 中分多步执行，典型场景是 `GenericAmend` 批量修改多个提交：

1. 启动交互式 rebase，将多个提交改为 edit
2. 第一个提交停下 → amend → continue
3. 第二个提交停下 → amend → continue
4. ...

实现方式：
- 通过 `self.onSuccessfulContinue` 字段注册下一步回调函数
- 每次成功执行 `rebase --continue` 后，自动调用回调
- `rebase --abort` 时清空回调

这样就实现了"多步骤 rebase 操作链"，用户看起来是一个操作，实际上由多次 continue 串联完成。

---

## 四、Todo/Done 状态与 Continue/Abort 的关系

### 4.1 状态文件的角色分工

在 rebase 进行中，`git-rebase-todo` 和 `done` 两个文件构成了经典的"待办/已办"双队列模型：

```
  ┌─────────────────────┐       ┌─────────────────────┐
  │   git-rebase-todo   │       │        done         │
  │   (待执行队列)       │──────▶│   (已完成队列)       │
  │   顶部 = 下一个      │ 执行  │   末尾 = 最近完成    │
  └─────────────────────┘       └─────────────────────┘
         ↑
    stopped-sha = 正在处理中的提交
```

- **git-rebase-todo**：FIFO 队列，顶部（文件开头）是下一个要执行的命令
- **done**：按执行顺序追加，末尾是最近完成的命令
- **stopped-sha**：当前停下来的提交 SHA（只有 stopped 状态时有意义）

### 4.2 Continue 时的状态流转

执行 `git rebase --continue` 后，Git 内部的状态变化：

```
停止状态 (stopped)
    │
    │  用户执行 continue
    ▼
1. 提交当前改动（如果有 staged 的内容）
   - 如果是 edit 模式 → git commit --amend
   - 如果是冲突 → git commit (完成冲突合并)
    │
    ▼
2. 把刚完成的命令追加到 done 文件尾部
    │
    ▼
3. 从 git-rebase-todo 顶部取下一条命令
    │
    ▼
4. 执行该命令
   ├─ 成功 → 回到步骤 2（继续下一条）
   ├─ 失败（冲突）→ 停止，更新 stopped-sha，返回错误
   └─ 遇到 break/edit/reword → 停止，等待用户操作
    │
    ▼
全部完成 → 删除 rebase-merge 目录，rebase 结束
```

从代码层面看，`ContinueRebase()` 只是简单调用 `git rebase --continue`，但背后 Git 完成了一系列状态文件更新。Lazygit 通过刷新后重新读取状态文件来感知变化。

### 4.3 Abort 时的状态流转

执行 `git rebase --abort` 后：

```
停止状态 (stopped)
    │
    │  用户执行 abort
    ▼
1. 重置工作区和暂存区到 rebase 开始前的状态
   - 丢弃所有已应用的提交改动
   - 恢复原始 HEAD
    │
    ▼
2. 清空并删除 rebase-merge 目录
   (git-rebase-todo / done / stopped-sha / amend / message 等全部消失)
    │
    ▼
3. WorkingTreeState.Rebasing 变为 false
    │
    ▼
回退到 rebase 开始前的状态
```

Lazygit 代码层面的联动：
- `onSuccessfulContinue` 回调被清空（因为 abort 后不需要继续了）
- 下次刷新时，CommitLoader 检测不到 rebase-merge 目录，就不再加载 todo 项

### 4.4 Skip 时的状态流转

执行 `git rebase --skip`：

```
停止状态 (stopped)
    │
    │  用户执行 skip
    ▼
1. 放弃当前提交的改动
    │
    ▼
2. 把当前命令标记为已执行（追加到 done？还是丢弃？）
   → 实际上 skip 会跳过该提交，不将其加入 done
    │
    ▼
3. 从 todo 取下一条命令继续
    │
    ▼
... 后续同上
```

> Skip 的典型场景：空提交（"No changes"）时，Lazygit 会自动调用 skip，避免用户手动处理。

### 4.5 冲突状态的检测与判定

Lazygit 通过两层机制判断"是否在冲突"：

**第一层：文件存在性检测**（粗粒度）
- `IsInRebase()` → rebase-merge 目录存在吗？
- 存在 → 处于 rebase 中，但不一定在冲突（可能是 edit/break 正常停下）

**第二层：done 文件分析**（细粒度）
- 读取 done 文件最后一条记录
- 检查它是不是 break/exec/reword（这些是正常停止，不是冲突）
- 检查它有没有被 rescheduled（done 最后一条 == todo 第一条）
- 对于 edit，检查 amend 文件是否存在
- 都排除了 → 认为是冲突停止

这就是 [getConflictedCommit](pkg/commands/git_commands/commit_loader.go) 的核心逻辑。

### 4.6 命令执行的成功路径 vs 冲突路径

以一次 pick 命令为例，对比两种路径：

**成功路径**：
```
pick abc123 "some commit"  (在 todo 顶部)
    ↓
git apply 成功
    ↓
todo 顶部移除该命令
    ↓
追加到 done 文件末尾
    ↓
继续下一条 todo
```

**冲突路径**：
```
pick abc123 "some commit"  (在 todo 顶部)
    ↓
git apply 失败 → 冲突
    ↓
工作区出现冲突标记 (UU 状态文件)
    ↓
stopped-sha 被设置为 abc123
    ↓
该命令被重新放回 todo 顶部 (rescheduled)
    ↓
done 文件末尾也追加了该命令？
→ 不一定，取决于 Git 版本和具体情况
→ 代码中用 "done最后一条 == todo第一条" 来检测 rescheduled
    ↓
rebase 停止，返回错误
```

---

## 五、冲突检测与回退机制

### 5.1 工作树状态检测

[WorkingTreeState](pkg/commands/models/working_tree_state.go) 有四个独立的布尔状态，可以同时存在多个：

- `Rebasing`：检测 `rebase-merge` 或 `rebase-apply` 目录是否存在
- `Merging`：检测 `MERGE_HEAD` 文件是否存在
- `CherryPicking`：检测 `CHERRY_PICK_HEAD` 文件是否存在（需排除 rebase 过程中的残留）
- `Reverting`：检测 `REVERT_HEAD` 文件是否存在

**Effective 状态**（优先级从高到低）：
```
Reverting > CherryPicking > Merging > Rebasing > None
```

Effective 状态决定了 UI 显示哪种状态的标题和操作菜单。例如：在 rebase 中执行 cherry-pick 发生冲突时，effective 状态是 cherry-picking，用户只能先处理 cherry-pick 的 continue/abort，然后才能继续 rebase。

### 5.2 冲突检测流程

冲突检测发生在执行 rebase 命令之后，由 [CheckMergeOrRebase](pkg/gui/controllers/helpers/merge_and_rebase_helper.go) 处理：

1. **刷新 UI**：先调用 `c.Refresh()` 刷新视图状态
2. **成功判断**：如果命令返回 `nil`（成功），直接返回
3. **特殊情况处理**：
   - "No changes - did you forget to use" → 空提交，自动 skip
   - "The previous cherry-pick is now empty" → 空 cherry-pick，自动 skip
   - "No rebase in progress?" → 认为已经完成了
4. **冲突检测**：调用 `CheckForConflicts`，通过错误字符串匹配判断

[isMergeConflictErr](pkg/gui/controllers/helpers/merge_and_rebase_helper.go) 匹配 7 种冲突关键字：
- "Failed to merge in the changes"
- "When you have resolved this problem"
- "fix conflicts"
- "Resolve all conflicts manually"
- "Merge conflict in file"
- "hint: after resolving the conflicts"
- "CONFLICT (content):"

### 5.3 冲突处理流程

检测到冲突后，弹出 [PromptForConflictHandling](pkg/gui/controllers/helpers/merge_and_rebase_helper.go) 菜单：

```
┌─ 发现冲突 ──────────────┐
│  查看冲突               │  → 跳转到 Files 视图
│  中止 rebase (a)       │  → 执行 rebase --abort
└─────────────────────────┘
```

当用户解决冲突后（文件状态从 UU 变为已暂存），通过 [PromptToContinueRebase](pkg/gui/controllers/helpers/merge_and_rebase_helper.go) 提示继续：

1. 刷新文件状态（SYNC 模式，确保最新）
2. 如果还有未暂存文件，询问是否自动暂存
3. 执行 `rebase --continue`
4. 根据结果继续循环（可能下一个提交又冲突）

### 5.4 回退（Abort）机制详解

Abort 的完整调用路径：

```
用户点击 Abort 菜单
    ↓
MergeAndRebaseHelper.genericMergeCommand("abort")
    ↓
RebaseCommands.GenericMergeOrRebaseAction("rebase", "abort")
    ↓
runSkipEditorCommand(cmdObj)   ← 注入跳过编辑器的环境变量
    ↓
执行 git rebase --abort
    ↓
Git 内部：
  1. 重置索引和工作树
  2. 检出原始 HEAD
  3. 删除 rebase-merge 目录
    ↓
代码层面：
  - onSuccessfulContinue = nil  ← 清空回调链
  - 返回错误被 CheckMergeOrRebase 处理
    ↓
刷新后：
  - WorkingTreeState.Rebasing = false
  - CommitLoader 不再加载 todo 项
  - UI 回到正常提交列表视图
```

Abort 可以在**任何时候**执行，无论当前是冲突停止、edit 停止、还是 break 停止。执行后工作树完全回到 rebase 开始前的状态。

---

## 六、关键数据结构关系图

```
┌───────────────────────────────────────────────────────┐
│                    GUI 层                              │
│  local_commits_controller ─→ merge_and_rebase_helper   │
└──────────────────────┬────────────────────────────────┘
                       │ 调用
┌──────────────────────▼────────────────────────────────┐
│               Git Commands 层                         │
│  RebaseCommands ─→ PrepareInteractiveRebaseCommand    │
│                    (设置 GIT_SEQUENCE_EDITOR)         │
└──────────────────────┬────────────────────────────────┘
                       │ 执行 git 命令
┌──────────────────────▼────────────────────────────────┐
│                      Git                               │
│  git rebase -i → 调用 $GIT_SEQUENCE_EDITOR             │
└──────────────────────┬────────────────────────────────┘
                       │ 启动 lazygit daemon 子进程
┌──────────────────────▼────────────────────────────────┐
│                  Daemon 层                             │
│  daemon.Handle → instruction.run()                     │
│                   └→ utils/rebase_todo.go              │
│                      (读写 .git/rebase-merge/...)       │
└──────────────────────┬────────────────────────────────┘
                       │ 修改文件
┌──────────────────────▼────────────────────────────────┐
│              .git/rebase-merge/ 目录                   │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │git-rebase-todo│  │     done     │                  │
│  │  (待执行)    │  │  (已完成)    │                  │
│  └──────┬───────┘  └──────┬───────┘                   │
│         │ continue/abort   │                            │
│         ▼                  ▼                            │
│  stopped-sha / amend / message  (停止时的状态文件)     │
└───────────────────────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────┐
│                Commit Loader 层                        │
│  MergeRebasingCommits → 水化 → 融合到真实提交列表       │
│  getConflictedCommit  → 识别冲突提交                   │
└───────────────────────────────────────────────────────┘
```

---

## 七、典型场景：Edit + Amend 完整流程

以"修改历史中第 3 个提交"为例，完整走一遍状态流转：

### 阶段 1：启动 Rebase

用户选中第 3 个提交，按 `e`（edit）：
1. `BeginInteractiveRebaseForCommit` 被调用
2. 构造 `ChangeTodoAction`：将该提交的 action 从 pick 改为 edit
3. `PrepareInteractiveRebaseCommand` 构建 `git rebase -i <base>` 命令
4. 设置 daemon 指令为 `ChangeTodoActions`
5. 启动 git 命令

Git 内部：
1. 启动交互式 rebase
2. 调用 GIT_SEQUENCE_EDITOR（lazygit daemon）
3. daemon 修改 git-rebase-todo 文件（pick → edit）
4. 继续执行 rebase，应用前 2 个提交
5. 遇到第 3 个提交的 edit 命令，停下

### 阶段 2：Edit 停止状态

此时 `.git/rebase-merge/` 中的状态：
- `git-rebase-todo`：剩下的提交（第 4 个及以后）
- `done`：前 2 个 pick + 1 个 edit？不一定，取决于 Git 实现
- `stopped-sha`：第 3 个提交的哈希
- `amend` 文件：存在吗？
  - 刚停下时 amend 不存在 → 表示 edit 命令"已到达但未开始修改"
  - 用户做了 commit --amend 后 amend 才会存在 → 表示 edit 已经在进行中

UI 表现：
- 第 3 个提交显示为 "You are here" 状态
- 下面的提交显示为 "Pending rebase todos"

### 阶段 3：用户修改并 Amend

用户修改文件、暂存，然后 amend：
1. `git commit --amend --no-edit`
2. 提交被修改（哈希变化）
3. `amend` 文件被创建

这一步不涉及 continue，rebase 仍然是 stopped 状态。

### 阶段 4：Continue

用户按 continue：
1. `git rebase --continue`
2. Git 发现 amend 文件存在 → 认为 edit 已完成
3. 把 edit 命令追加到 done
4. 从 todo 取下一条（第 4 个提交）
5. 如果下一个是 pick 且无冲突 → 继续执行...
6. 直到全部完成 → rebase-merge 目录被删除

如果有 onSuccessfulContinue 回调（比如批量 amend 多个提交）：
- 成功 continue 后自动执行回调
- 回调可能启动下一轮修改

### 阶段 5：如果中途冲突

假设第 4 个提交冲突：
1. Git 停止，返回错误
2. `stopped-sha` 指向第 4 个提交
3. 第 4 个提交被 rescheduled（回到 todo 顶部）
4. Lazygit 的 CheckForConflicts 检测到冲突
5. 弹出冲突处理菜单
6. 用户解决冲突 → 继续回到阶段 4

### 阶段 6：完成

所有 todo 执行完毕：
- `rebase-merge/` 目录被 Git 自动清理
- `WorkingTreeState.Rebasing` 变为 false
- CommitLoader 不再加载 todo 项
- UI 显示最终的提交列表（所有哈希可能都变了）

---

## 八、重要设计要点

### 1. Daemon 模式的巧思

利用 Git 的编辑器回调机制，让 lazygit 能以非侵入方式程序化控制交互式 rebase。相比手动调用 `git rebase --edit-todo` 或解析 git 输出，这种方式更稳定、更符合 Git 的设计哲学。

### 2. Todo 与真实提交的无缝融合

CommitLoader 将待办列表"水化"后插入真实提交列表前面，让用户在 UI 上看到连贯的视图 —— 仿佛 rebase 已经完成、只是还没"落实"。这种设计大大降低了用户的认知负担。

### 3. 多状态叠加与优先级

WorkingTreeState 支持四态叠加，Effective 状态确保用户永远处理最内层的操作。例如 rebase 中的 cherry-pick 冲突，用户必须先解决 cherry-pick，才能继续 rebase。

### 4. onSuccessfulContinue 回调链

通过注册回调函数的方式，支持多步骤的 rebase 操作（如批量 amend 多个 commit）。每步完成后自动触发下一步，对用户透明。

### 5. Rescheduled 检测的细腻处理

Git 在命令失败时会把命令重新放回 todo 队列顶部，导致 done 和 todo 出现重复项。代码中通过多种边界条件检测 rescheduled 状态，避免 UI 上出现重复或错误的冲突提示。

### 6. 移动时跳过非渲染项

移动 todo 时跳过 label/reset/comment 等不可见项，保证用户移动提交时的直觉正确 —— 按一下移动一个"可见提交"，而不是卡在看不见的内部命令上。
