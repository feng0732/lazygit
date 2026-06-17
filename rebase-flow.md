# 交互式 Rebase 代码流程梳理

## 概述

Lazygit 的交互式 rebase 系统围绕 **待办列表 (Todo)**、**已完成列表 (Done)**、**命令触发**、**冲突检测与回退** 四个核心概念构建。整个系统利用 Git 的 `GIT_SEQUENCE_EDITOR` 机制切入，将 lazygit 自身作为 daemon 进程来编辑 todo 文件，从而实现程序化的 rebase 操作。

---

## 一、核心文件索引

> 以下路径均为相对于仓库根目录的相对路径。

| 模块 | 文件 | 职责 |
|------|------|------|
| Git 命令层 | [pkg/commands/git_commands/rebase.go](pkg/commands/git_commands/rebase.go) | 构建 rebase 命令、封装 continue/abort/skip |
| 待办列表工具 | [pkg/utils/rebase_todo.go](pkg/utils/rebase_todo.go) | 读写 git-rebase-todo 文件、移动/删除 todo 项 |
| Daemon 层 | [pkg/app/daemon/daemon.go](pkg/app/daemon/daemon.go) + [pkg/app/daemon/rebase.go](pkg/app/daemon/rebase.go) | 作为 Git 的编辑器被调用，执行具体的 todo 修改指令 |
| GUI 控制层 | [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go) | rebase 选项菜单、冲突检测与处理、continue/abort 调度 |
| 提交加载器 | [pkg/commands/git_commands/commit_loader.go](pkg/commands/git_commands/commit_loader.go) | 从 git-rebase-todo 加载待办项，融合冲突提交与真实提交 |
| 工作树状态 | [pkg/commands/models/working_tree_state.go](pkg/commands/models/working_tree_state.go) + [pkg/commands/git_commands/status.go](pkg/commands/git_commands/status.go) | 检测当前是否处于 rebase/merge/cherry-pick 状态 |
| 提交模型 | [pkg/commands/models/commit.go](pkg/commands/models/commit.go) | Commit 数据结构，包含 Action / Status 等 todo 相关字段 |
| 测试用例 | [pkg/commands/git_commands/commit_loader_test.go](pkg/commands/git_commands/commit_loader_test.go) | `getConflictedCommitImpl` 的测试用例，清晰展示各状态组合 |

---

## 二、Todo/Done 双队列模型

### 2.1 状态文件总览

Git 在交互式 rebase 进行中会在 `.git/rebase-merge/` 目录下维护一组状态文件，构成一个经典的"待办/已办"双队列系统：

| 文件 | 角色 | 说明 |
|------|------|------|
| `git-rebase-todo` | **待执行队列** | 自上而下排列，顶部 = 下一个要执行的命令 |
| `done` | **已完成队列** | 按执行顺序追加，末尾 = 最近完成的命令 |
| `stopped-sha` | **当前停止点** | 当前停下处的提交 SHA（缩写），只有 stopped 状态时有意义 |
| `amend` | **edit 模式标记** | 文件存在表示用户已执行过 `git commit --amend` |
| `message` | **提交信息暂存** | 存放 edit/reword 命令的原始提交信息 |
| `head-name` | **原始分支名** | 正在被 rebase 的分支全名 |

### 2.2 双队列流转示意

```
git-rebase-todo (待执行)            done (已完成)
┌───────────────────────┐           ┌───────────────────────┐
│  pick C (下一个)       │──执行成功─▶│  pick A               │
│  pick D                │           │  pick B               │
│  ...                   │           │  ...                  │
│  旧的在下面             │           │  旧的在下面             │
└───────────────────────┘           └───────────────────────┘
         ▲
         │ 失败则 rescheduled（重新放回顶部）
         ▼
    stopped-sha = 当前卡住的提交
```

### 2.3 Lazygit 中的展示顺序

Git 的 todo/done 文件都是"旧的在上、新的在下"，但 Lazygit 的提交视图是"新的在上、旧的在下"。因此加载时需要做一次倒序转换。

在 `getRebasingCommits` 函数（[pkg/commands/git_commands/commit_loader.go](pkg/commands/git_commands/commit_loader.go) 第 341 行）中：

1. 先检测是否有冲突提交，如果有，追加到列表末尾
2. 然后遍历 todos，每一条都用 `Prepend` 插到列表前面
3. 最终实现倒序：新的 todo 在上，旧的在下

例如 todo 文件内容是：
```
pick commit-A (旧)
pick commit-B
pick commit-C (新，下一个执行)
```

倒序后展示为：
```
pick commit-C   ← 最上面，下一个要执行的
pick commit-B
pick commit-A   ← 最下面，最旧的
[冲突提交]      ← 如果有冲突，在 todo 和真实提交之间
真实提交...
```

---

## 三、冲突检测：getConflictedCommitImpl 详解

冲突提交的检测是整个状态系统中最复杂的部分，由 `getConflictedCommitImpl` 函数（[pkg/commands/git_commands/commit_loader.go](pkg/commands/git_commands/commit_loader.go) 第 405 行）实现。

### 3.1 函数签名与输入输出

```go
func getConflictedCommitImpl(
    hashPool *utils.StringPool,
    todos []todo.Todo,        // git-rebase-todo 解析结果
    doneTodos []todo.Todo,    // done 文件解析结果
    amendFileExists bool,     // amend 文件是否存在
    messageFileExists bool,   // message 文件是否存在
) *models.Commit
```

- **返回 nil**：当前不是冲突状态（是正常停止，或者状态无法判断）
- **返回 Commit 对象**：当前处于冲突状态，返回的 commit 状态为 `StatusConflicted`

### 3.2 完整判定流程

以下是完整的判定逻辑（按代码执行顺序）：

```
输入：todos, doneTodos, amendFileExists, messageFileExists
    │
    ▼
┌─ 1. done 为空？ ────────────────────┐
│  是 → 返回 nil（防御性检查）         │
└─────────────────────────────────────┘
    │
    ▼
┌─ 2. 最后一条 done 是 break/exec/reword？ ─┐
│  是 → 返回 nil（正常停止，不是冲突）      │
│                                            │
│  说明：                                    │
│   - break: 用户插入的断点                  │
│   - exec: 执行外部命令停下                 │
│   - reword: 等待用户编辑提交信息           │
└───────────────────────────────────────────┘
    │
    ▼
┌─ 3. Rescheduled 检测（标准情况） ────────────────┐
│  done 最后一条 == todo 第一条？                    │
│  是 → 返回 nil（命令被重新调度了）                 │
│                                                    │
│  说明：                                            │
│   当一个命令因冲突失败时，Git 会把它重新放回        │
│   todo 列表的顶部，这叫 "rescheduled"。            │
│   这时候 todo 和 done 各有一份相同的记录。          │
│   因为 todo 里已经有这条命令了，不需要额外加一个。  │
└───────────────────────────────────────────────────┘
    │
    ▼
┌─ 4. Rescheduled 检测（老版本 Git bug） ───────────┐
│  len(done) >= 3 且                                │
│  done 倒数第二条 == todo 第一条 且                  │
│  done 最后一条 == done 倒数第三条？                 │
│  是 → 返回 nil（也是 rescheduled）                 │
│                                                    │
│  说明：                                            │
│   老版本 Git 有个 bug：命令 rescheduled 时，       │
│   会把"上一个成功的命令"再追加一份到 done 末尾。    │
│   需要额外检测这种情况，避免误判。                 │
└───────────────────────────────────────────────────┘
    │
    ▼
┌─ 5. 如果最后一条 done 是 edit ────────────────────┐
│  ┌─ 5a. amend 文件存在？ ──┐                      │
│  │  是 → 返回 nil           │                      │
│  │  说明：edit 模式下，用户 │                      │
│  │  已执行了 amend，编辑正  │                      │
│  │  常进行中，不是冲突。    │                      │
│  └─────────────────────────┘                      │
│         │                                          │
│         ▼ 否                                      │
│  ┌─ 5b. message 文件不存在？ ─┐                    │
│  │  是 → 返回 nil             │                    │
│  │  说明：message 文件消失了， │                    │
│  │  可能是 cherry-pick/revert │                    │
│  │  等操作干扰了状态，不判为   │                    │
│  │  冲突。                     │                    │
│  └─────────────────────────┘                      │
│         │                                          │
│         ▼ 否（有 message 但没有 amend）            │
│    继续往下（可能是冲突）                          │
└───────────────────────────────────────────────────┘
    │
    ▼
┌─ 6. 最后一条 done 没有 commit hash？ ──────────┐
│  是 → 返回 nil（安全检查，没有提交 hash 无法显示） │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─ 7. 其他情况 → 判定为冲突 ──────────────────────┐
│  返回 StatusConflicted 的 Commit 对象            │
│                                                  │
│  包括的场景：                                    │
│   - pick / squash / fixup 等命令冲突             │
│   - edit 命令在应用阶段冲突（有 message 无 amend）│
└─────────────────────────────────────────────────┘
```

### 3.3 Rescheduled 的深入理解

**什么是 rescheduled？**

当一个 rebase 命令失败时（比如应用补丁时覆盖了未跟踪的文件，或者发生了冲突），Git 会把这条命令**重新放回 todo 列表的顶部**，等待用户处理后重试。这就是 "rescheduled"（重新调度）。

**为什么 rescheduled 就不需要额外添加冲突提交？**

因为命令已经在 todo 列表里了（作为第一条），虽然它的状态是 `StatusRebasing` 而不是 `StatusConflicted`，但用户可以通过文件视图中的冲突文件感知到冲突状态。

如果此时再额外添加一个冲突提交，就会出现重复显示。

**两种 rescheduled 检测的区别：**

| 检测方式 | 适用场景 | 判定条件 |
|----------|----------|----------|
| 标准检测 | 新版 Git | `done[last] == todo[first]` |
| Bug 检测 | 老版 Git | `done[last-1] == todo[first]` 且 `done[last] == done[last-2]` |

老版本 Git 的 bug 表现：当命令 rescheduled 时，会把"上一个成功的命令"重复追加到 done 末尾，导致 done 最后出现两条相同的记录。

### 3.4 Edit 停止状态的三种情况

Edit 命令是最特殊的，因为它有"正常停下"和"冲突停下"两种停止原因，需要通过辅助文件进一步区分。

从测试用例 `TestCommitLoader_getConflictedCommitImpl`（[pkg/commands/git_commands/commit_loader_test.go](pkg/commands/git_commands/commit_loader_test.go) 第 476-516 行）可以清晰看到三种情况：

| 场景 | amend 文件 | message 文件 | 判定结果 | 含义 |
|------|-----------|-------------|----------|------|
| edit + amend | 存在 | - | **不是冲突**（返回 nil） | 用户已执行 `git commit --amend`，edit 正常进行中 |
| edit + 无amend + 有message | 不存在 | 存在 | **是冲突**（返回 StatusConflicted） | edit 命令在应用阶段就冲突了 |
| edit + 无amend + 无message | 不存在 | 不存在 | **不是冲突**（返回 nil） | 状态被干扰（如 cherry-pick），不判定 |

**为什么"有 message 但没有 amend"是冲突？**

这需要理解 edit 命令的两阶段执行：

```
edit 命令执行流程：

  阶段 1：应用提交（pick 阶段）
     │
     ├─ 成功 → 进入阶段 2
     │
     └─ 失败（冲突）→ 停下
          → message 文件存在（Git 准备了原始消息）
          → amend 文件不存在（还没到那一步）
          → 这就是冲突状态

  阶段 2：等待用户编辑
     → message 文件存在
     → amend 文件不存在（用户还没 amend）
     → 这是正常停止，不是冲突

  阶段 3：用户执行了 commit --amend
     → message 文件存在
     → amend 文件被创建
     → 仍然是正常编辑中
```

等等，按照这个流程，"阶段 2"也是"有 message 无 amend"，但应该是正常停止，不是冲突啊？

**关键在于：edit 命令正常停下时，它在 done 列表里吗？**

答案是：**不在**。

当 edit 命令成功应用并正常停下时，这条 edit 命令**还没有被写入 done 文件**（因为 edit 命令还没"完成"，需要用户 continue 之后才算完成）。此时 done 的最后一条应该是上一个成功执行的命令（比如上一个 pick）。

只有当 edit 命令在**应用阶段就失败**（冲突）时，Git 才会把 edit 命令写入 done 文件（标记为"尝试过但失败了"）。这时候 done 的最后一条就是 edit，同时满足"有 message 无 amend"。

这就是为什么代码中 `lastTodo.Command == todo.Edit` 的情况下，还需要进一步用 amend 和 message 文件来判断：
- 如果 done 的最后一条是 edit，说明 edit 命令的执行出了问题
- 再结合 amend/message 文件判断具体原因
- 有 amend → 用户成功 amend 了，正常
- 没 amend 但有 message → 应用阶段冲突了
- 都没有 → 状态被干扰了

> **注意**：以上关于"edit 正常停下时不在 done 里"的推理是基于代码逻辑反推的。实际 Git 内部实现可能更复杂，但 lazygit 的判定逻辑是清晰的，可以通过测试用例验证。

---

## 四、Continue / Abort / Skip 与状态流转

### 4.1 统一入口

`GenericMergeOrRebaseAction`（[pkg/commands/git_commands/rebase.go](pkg/commands/git_commands/rebase.go) 第 482 行）是 continue / abort / skip 的统一入口：

```go
continue → git rebase --continue
abort    → git rebase --abort
skip     → git rebase --skip
```

### 4.2 Continue 时的状态流转

执行 `git rebase --continue` 后，Git 内部发生的状态文件变化：

```
当前状态：stopped（可能是冲突停下，也可能是 edit/break 正常停下）
    │
    │  用户执行 continue
    ▼
1. 提交当前改动
   - edit 模式且有 staged → git commit --amend
   - 冲突已解决 → git commit（完成合并）
   - 没有改动 → 直接继续
    │
    ▼
2. 把当前命令追加到 done 文件尾部
   （标记为已完成）
    │
    ▼
3. 从 git-rebase-todo 顶部取下一条命令
    │
    ▼
4. 执行该命令
   ├─ 成功 → 回到步骤 2，继续循环
   ├─ 失败（冲突）→ 停止，更新 stopped-sha
   │   → 该命令可能被 rescheduled（放回 todo 顶部）
   └─ 遇到 break/edit/reword → 停止，等待用户
    │
    ▼
全部执行完毕 → 删除 rebase-merge 目录，rebase 结束
```

从 Lazygit 代码层面看：
- `ContinueRebase()` 只是简单调用 `git rebase --continue`
- 不直接操作状态文件
- 通过下次刷新时重新读取 `.git/rebase-merge/` 下的文件来感知变化

### 4.3 onSuccessfulContinue 回调链

有些操作需要在 rebase 中分多步执行，典型场景是 `GenericAmend`（批量修改多个历史提交）：

1. 启动交互式 rebase，将多个提交改为 edit
2. 第一个提交停下 → amend → continue
3. 第二个提交停下 → amend → continue
4. ... 依次处理所有目标提交

实现方式：
- 通过 `self.onSuccessfulContinue` 字段注册下一步回调函数
- 每次成功执行 `rebase --continue` 后，自动调用回调
- `rebase --abort` 时清空回调

这样就实现了"多步骤 rebase 操作链"，用户看起来是一个操作，实际上由多次 continue 串联完成。

### 4.4 Abort 时的状态流转

执行 `git rebase --abort`：

```
停止状态 (stopped)
    │
    │  用户执行 abort
    ▼
1. 重置工作区和暂存区到 rebase 开始前的状态
   - 丢弃所有已应用的提交改动
   - 检出原始 HEAD
    │
    ▼
2. 清空并删除 rebase-merge 目录
   （git-rebase-todo / done / stopped-sha / amend / message 全部消失）
    │
    ▼
3. WorkingTreeState.Rebasing 变为 false
    │
    ▼
完全回到 rebase 开始前的状态
```

代码层面的联动：
- `onSuccessfulContinue` 回调被清空（abort 后不需要继续了）
- 下次刷新时，CommitLoader 检测不到 rebase-merge 目录，不再加载 todo 项
- UI 回到正常提交列表视图

### 4.5 Skip 时的状态流转

执行 `git rebase --skip`：

```
停止状态 (stopped)
    │
    │  用户执行 skip
    ▼
1. 放弃当前提交的所有改动
    │
    ▼
2. 跳过该提交，不将其加入 rebase 结果
    │
    ▼
3. 从 todo 取下一条命令继续执行
    │
    ▼
... 后续同 continue
```

**自动 skip 的场景**：
当 rebase 遇到空提交（"No changes" 错误）时，Lazygit 会自动调用 skip，避免用户手动处理。

---

## 五、命令触发机制

### 5.1 Daemon 模式：Lazygit 作为 Git 的编辑器

Lazygit 通过将自身设置为 `GIT_SEQUENCE_EDITOR` 和 `GIT_EDITOR` 环境变量，在 Git 需要编辑 todo 文件或提交信息时被回调调用。这就是所谓的 "daemon 模式" —— 虽然名字叫 daemon，但实际上是一次性的子进程调用。

**指令传递方式**：通过环境变量传递 JSON 序列化的指令
- `LAZYGIT_DAEMON_KIND`：daemon 类型枚举（整数）
- `LAZYGIT_DAEMON_INSTRUCTION`：JSON 格式的指令参数

### 5.2 九种 Daemon 类型

在 [pkg/app/daemon/daemon.go](pkg/app/daemon/daemon.go) 中定义了 9 种 daemon 类型：

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

### 5.3 启动交互式 Rebase 的完整链路

以 `InteractiveRebase`（修改多个提交的 action）为例：

```
用户操作 (local_commits_controller)
    ↓
interactiveRebase(action, startIdx, endIdx)
    ↓
RebaseCommands.InteractiveRebase  [rebase.go]
    ├─ 计算 baseHashOrRoot
    │   (endIdx+1 作为基准，squash/fixup 则 +2)
    ├─ 构造 ChangeTodoAction 列表
    └─ PrepareInteractiveRebaseCommand
        ├─ 构造 git rebase 命令参数
        │   ├─ --interactive
        │   ├─ --autostash
        │   ├─ --keep-empty
        │   ├─ --no-autosquash
        │   └─ --rebase-merges
        ├─ 设置 GIT_SEQUENCE_EDITOR = lazygit 路径
        ├─ 设置 GIT_EDITOR = lazygit (overrideEditor 时)
        ├─ 通过环境变量注入 daemon instruction
        └─ 执行 git 命令
            ↓
Git 启动交互式 rebase
    ↓
Git 调用 $GIT_SEQUENCE_EDITOR（lazygit daemon）
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

### 5.4 Rebase 进行中修改 Todo

当 rebase 已经在进行中（stopped 状态），可以直接修改 `git-rebase-todo` 文件而无需重新启动 rebase。

相关方法在 [pkg/commands/git_commands/rebase.go](pkg/commands/git_commands/rebase.go) 约第 338-387 行：

- `EditRebaseTodo`：直接修改 todo 文件中指定提交的 action
- `MoveTodosUp` / `MoveTodosDown`：直接移动 todo 项
- `DeleteUpdateRefTodos`：删除 update-ref todo（需要通过 `git rebase --edit-todo` 让 Git 做内部 housekeeping）

> 注意：删除 update-ref todo 不能直接改文件，必须走 `git rebase --edit-todo` 命令，让 Git 处理相关的内部状态更新。

---

## 六、Todo 编辑操作详解

### 6.1 原子操作列表

[pkg/utils/rebase_todo.go](pkg/utils/rebase_todo.go) 提供了多种原子操作：

| 函数 | 用途 | 触发场景 |
|------|------|----------|
| `EditRebaseTodo` | 修改指定 commit 的 action | reword / edit / squash / fixup / drop |
| `MoveTodosUp` / `MoveTodosDown` | 移动 todo 项（上/下） | 用户按方向键移动提交 |
| `DeleteTodos` | 删除 todo 项 | 删除 update-ref todo |
| `MoveFixupCommitDown` | 移动 fixup 提交到目标下方 | amend / fixup 操作 |
| `DropMergeCommit` | 删除合并提交的 todo | 删除 merge commit |
| `PrependStrToTodoFile` | 在 todo 开头插入内容 | insert break |

### 6.2 移动逻辑的关键细节

**方向反转**：
- Git 的 todo 文件：自上而下执行（旧的在上）
- Lazygit 提交视图：自下而上展示（新的在上）
- 因此 UI 上的"向下移动"对应 todo 列表中向**起始方向**移动

**跳过不可见项**：
移动时只在 `isRenderedTodo` 之间跳跃，label / reset / comment / break 等不可见项会被自动跳过，保证用户按一下移动一个"可见提交"。

### 6.3 已渲染 Todo 的判定

`isRenderedTodo` 函数（[pkg/utils/rebase_todo.go](pkg/utils/rebase_todo.go) 第 292 行）决定哪些 todo 项显示在 UI 中：

```go
func isRenderedTodo(t todo.Todo, isInRebase bool) bool {
    return t.Commit != "" || 
           (isInRebase && (t.Command == todo.UpdateRef || t.Command == todo.Exec))
}
```

**显示的项**：
- 有 commit hash 的命令（pick / reword / edit / squash / fixup / drop / merge 等）
- 在 rebase 中且是 `update-ref`
- 在 rebase 中且是 `exec`

**不显示的项**：
- label / reset / comment / break
- 在 cherry-pick/revert  sequencer 中的 update-ref / exec

---

## 七、工作树状态与多状态叠加

### 7.1 四种独立状态

[WorkingTreeState](pkg/commands/models/working_tree_state.go) 有四个独立的布尔状态，可以同时存在多个：

| 状态 | 检测方式 | 含义 |
|------|----------|------|
| `Rebasing` | `rebase-merge` 或 `rebase-apply` 目录存在 | 正在 rebase |
| `Merging` | `MERGE_HEAD` 文件存在 | 正在 merge |
| `CherryPicking` | `CHERRY_PICK_HEAD` 文件存在（排除 rebase 残留） | 正在 cherry-pick |
| `Reverting` | `REVERT_HEAD` 文件存在 | 正在 revert |

### 7.2 Effective 状态优先级

```
Reverting > CherryPicking > Merging > Rebasing > None
```

Effective 状态决定了 UI 显示哪种状态的标题和操作菜单。

**示例**：在 rebase 中执行 cherry-pick 发生冲突时，effective 状态是 cherry-picking。用户必须先处理 cherry-pick 的 continue/abort，然后才能继续 rebase。

### 7.3 冲突检测流程

冲突检测发生在执行 rebase 命令之后，由 `CheckMergeOrRebase`（[pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go)）处理：

1. 刷新 UI
2. 如果命令成功（返回 nil），直接返回
3. 处理特殊情况：
   - "No changes" → 自动 skip
   - "No rebase in progress?" → 视为已完成
4. 调用 `CheckForConflicts`，通过错误字符串匹配判断

`isMergeConflictErr` 匹配 7 种冲突关键字：
- "Failed to merge in the changes"
- "When you have resolved this problem"
- "fix conflicts"
- "Resolve all conflicts manually"
- "Merge conflict in file"
- "hint: after resolving the conflicts"
- "CONFLICT (content):"

---

## 八、典型场景：Edit + Amend 完整流程

以"修改历史中第 3 个提交"为例，完整走一遍状态流转：

### 阶段 1：启动 Rebase

用户选中第 3 个提交，按 `e`（edit）：
- `BeginInteractiveRebaseForCommit` 被调用
- 构造 `ChangeTodoAction`：将该提交从 pick 改为 edit
- `PrepareInteractiveRebaseCommand` 构建 `git rebase -i <base>` 命令
- 设置 daemon 指令为 `ChangeTodoActions`
- 启动 git 命令

Git 内部：
1. 启动交互式 rebase
2. 调用 GIT_SEQUENCE_EDITOR（lazygit daemon）
3. daemon 修改 git-rebase-todo 文件（pick → edit）
4. 继续执行 rebase，应用前 2 个提交
5. 遇到第 3 个提交的 edit 命令，应用成功 → 停下

### 阶段 2：Edit 正常停止状态

此时 `.git/rebase-merge/` 中的状态：
- `git-rebase-todo`：剩下的提交（第 4 个及以后）
- `done`：前 2 个 pick（注意：没有 edit，因为 edit 还没完成）
- `stopped-sha`：第 3 个提交的哈希
- `amend`：不存在（用户还没 amend）
- `message`：存在（原始提交信息）

**重点**：edit 正常停下时，edit 命令**不在 done 里**（因为还没完成）。所以 `getConflictedCommit` 的 lastTodo 是上一个 pick 命令，不会判定为冲突。

UI 表现：
- 第 3 个提交在哪里？它不是 todo（因为已经从 todo 里取出来了），也不是冲突提交
- 实际上，这个提交就是当前 HEAD，可以通过正常的提交视图看到
- 下方的 todo 列表显示为 "Pending rebase todos"

### 阶段 3：用户修改并 Amend

用户修改文件、暂存，然后执行 amend：
1. `git commit --amend --no-edit`
2. 提交被修改（哈希变化）
3. `amend` 文件被创建（标记 edit 正在进行中）

这一步不涉及 continue，rebase 仍然是 stopped 状态。

如果此时检查 done 文件，最后一条仍然是上一个 pick。edit 命令本身仍然不在 done 里。

### 阶段 4：Continue

用户按 continue：
1. `git rebase --continue`
2. Git 发现是 edit 模式 → 把 edit 命令追加到 done
3. 从 todo 取下一条（第 4 个提交）
4. 如果是 pick 且无冲突 → 追加到 done，继续下一条...
5. 直到全部完成 → 删除 rebase-merge 目录

如果有 onSuccessfulContinue 回调（比如批量 amend 多个提交）：
- 成功 continue 后自动执行回调
- 回调可能启动下一轮修改

### 阶段 5：如果中途冲突

假设第 4 个提交冲突：
1. Git 停止，返回错误
2. `stopped-sha` 指向第 4 个提交
3. 第 4 个提交可能被 rescheduled（回到 todo 顶部）
4. Lazygit 的 CheckForConflicts 检测到冲突
5. 弹出冲突处理菜单

此时有两种子情况：

**子情况 A：被 rescheduled**
- todo 第一条 = pick commit-4
- done 最后一条 = pick commit-4
- `getConflictedCommit` 检测到 rescheduled → 返回 nil
- UI 上 todo 列表第一条就是冲突的提交（但状态是 StatusRebasing）
- 用户通过文件视图的冲突文件感知冲突

**子情况 B：没有被 rescheduled**
- todo 列表里没有 commit-4
- done 最后一条 = pick commit-4
- `getConflictedCommit` 判定为冲突 → 返回 StatusConflicted 的提交
- UI 上 todo 列表和真实提交之间多了一条红色 "CONFLICT" 标记的提交

### 阶段 6：完成

所有 todo 执行完毕：
- `rebase-merge/` 目录被 Git 自动清理
- `WorkingTreeState.Rebasing` 变为 false
- CommitLoader 不再加载 todo 项
- UI 显示最终的提交列表

---

## 九、重要设计要点

### 1. Daemon 模式的巧思

利用 Git 的编辑器回调机制，让 lazygit 能以非侵入方式程序化控制交互式 rebase。相比手动调用 `git rebase --edit-todo` 或解析 git 输出，这种方式更稳定、更符合 Git 的设计哲学。

### 2. Todo 与真实提交的无缝融合

CommitLoader 将待办列表"水化"后插入真实提交列表前面，让用户在 UI 上看到连贯的视图 —— 仿佛 rebase 已经完成、只是还没"落实"。这种设计大大降低了用户的认知负担。

### 3. Rescheduled 检测的细腻处理

Git 在命令失败时会把命令重新放回 todo 队列顶部（rescheduled），导致 done 和 todo 出现重复项。代码中通过两层检测（标准情况 + 老版本 Git bug）处理 rescheduled 状态，避免 UI 上出现重复或错误的冲突提示。

### 4. Edit 状态的三级判定

Edit 命令是唯一需要通过辅助文件（amend/message）进一步区分的命令。从"done 里有没有 edit"到"amend 存不存在"再到"message 存不存在"，三级判定精细地区分了"冲突"和"正常编辑中"两种状态。

### 5. onSuccessfulContinue 回调链

通过注册回调函数的方式，支持多步骤的 rebase 操作（如批量 amend 多个 commit）。每步完成后自动触发下一步，对用户透明。

### 6. 多状态叠加与优先级

WorkingTreeState 支持四态叠加，Effective 状态确保用户永远处理最内层的操作。例如 rebase 中的 cherry-pick 冲突，用户必须先解决 cherry-pick，才能继续 rebase。

### 7. 移动时跳过非渲染项

移动 todo 时跳过 label/reset/comment/break 等不可见项，保证用户移动提交时的直觉正确 —— 按一下移动一个"可见提交"，而不是卡在看不见的内部命令上。
