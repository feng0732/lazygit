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
| 集成测试 | [pkg/integration/tests/interactive_rebase/](pkg/integration/tests/interactive_rebase/) | 多种场景的端到端测试 |

---

## 二、Todo/Done 双队列模型

### 2.1 状态文件总览

Git 在交互式 rebase 进行中会在 `.git/rebase-merge/` 目录下维护一组状态文件，构成一个经典的"待办/已办"双队列系统：

| 文件 | 角色 | 说明 |
|------|------|------|
| `git-rebase-todo` | **待执行队列** | 自上而下排列，顶部 = 下一个要执行的命令 |
| `done` | **已完成队列** | 按执行顺序追加，末尾 = 最近完成或最近开始处理的命令 |
| `stopped-sha` | **当前停止点** | 当前停下处的提交 SHA（缩写），只有 stopped 状态时有意义 |
| `amend` | **edit 模式标记** | edit 命令**成功应用**时由 Git 自动创建，标记"下一次 commit 应以 --amend 模式执行" |
| `message` | **提交信息暂存** | 存放 edit/reword 命令的原始提交信息 |
| `head-name` | **原始分支名** | 正在被 rebase 的分支全名 |

### 2.2 关键理解：amend 文件的真实含义

> **这是之前理解错误的核心点**

`amend` 文件**不是**用户执行 `git commit --amend` 时才创建的。它是 Git 在 edit 命令**成功应用提交并正常停下时，自动创建**的。

它的作用是：标记当前处于 edit 模式，后续的 `git commit` 操作应该自动使用 `--amend` 模式。所以 amend 文件的存在就意味着 **edit 命令已经成功应用了提交，正在等待用户编辑**。

对应代码注释（[pkg/commands/git_commands/commit_loader.go](pkg/commands/git_commands/commit_loader.go) 第 454-455 行）：
```go
// Special case for "edit": if the "amend" file exists, the "edit"
// command was successful, otherwise it wasn't
```

### 2.3 双队列流转示意

```
git-rebase-todo (待执行)               done (已处理)
┌──────────────────────────┐           ┌──────────────────────────┐
│  pick C (下一个)          │──执行成功─▶│  pick A                   │
│  pick D                   │           │  pick B                   │
│  ...                      │           │  edit C  (当前停下处)     │ ← amend 存在 = edit 成功
│  旧的在下面                │           │  ...                      │
└──────────────────────────┘           └──────────────────────────┘
         ▲
         │ 失败则 rescheduled（重新放回顶部）
         ▼
    stopped-sha = 当前卡住的提交
```

**重要**：命令一旦被 Git 从 todo 顶部取出来开始处理，就会被**立即写入 done 文件**，无论它最终是成功还是失败。所以 done 文件的最后一条可能是：
- 已经成功完成的命令（继续下一条）
- 正在处理中但因为 edit/break/reword 而正常停下的命令
- 尝试执行但失败（冲突）的命令

### 2.4 Lazygit 中的展示顺序

Git 的 todo/done 文件都是"旧的在上、新的在下"，但 Lazygit 的提交视图是"新的在上、旧的在下"。因此加载时需要做一次倒序转换。

在 `getRebasingCommits` 函数（[pkg/commands/git_commands/commit_loader.go](pkg/commands/git_commands/commit_loader.go) 第 341 行）中：

1. 先检测是否有冲突提交，如果有，用 `append` 追加到列表**末尾**
2. 然后遍历 todos，每一条都用 `Prepend` 插到列表**前面**
3. 最终实现倒序：新的 todo 在上，旧的在下；冲突提交在 todo 和真实提交之间

例如 todo 文件内容是：
```
pick commit-A (旧)
pick commit-B
pick commit-C (新，下一个执行)
```

倒序后展示为：
```
pick commit-C       ← 最上面，下一个要执行的（StatusRebasing）
pick commit-B       ← StatusRebasing
pick commit-A       ← StatusRebasing
[冲突提交]          ← 如果有冲突，在 todo 和真实提交之间（StatusConflicted）
真实提交...          ← 从 HEAD 开始的真实提交
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

- **返回 nil**：当前不是冲突状态（是正常停止，或者状态无法判断，或者被 rescheduled）
- **返回 Commit 对象**：当前处于冲突状态，返回的 commit 状态为 `StatusConflicted`

### 3.2 完整判定流程（严格对照代码）

以下是完整的判定逻辑，**完全按照代码执行顺序**：

```
输入：todos, doneTodos, amendFileExists, messageFileExists
    │
    ▼
┌─ 1. done 为空？ ────────────────────────┐
│  是 → 返回 nil（防御性检查，不可能发生） │
└──────────────────────────────────────────┘
    │
    ▼
┌─ 2. done 最后一条是 break / exec / reword？ ───┐
│  是 → 返回 nil（这是正常停止，不是冲突）        │
│                                                  │
│  说明：                                          │
│   - break:  用户插入的断点，主动停下             │
│   - exec:   执行外部命令，停下等用户确认         │
│   - reword: 等待用户编辑提交信息                 │
│  这三种都是 Git 预期内的正常停止，不算冲突。      │
└──────────────────────────────────────────────────┘
    │
    ▼
┌─ 3. Rescheduled 检测（标准情况） ──────────────────────────────┐
│  done[最后一条] == todo[第一条]？                                │
│  是 → 返回 nil                                                  │
│                                                                  │
│  说明（注释 416-421 行）：                                       │
│   当一个命令失败时（如补丁覆盖了未跟踪文件，或冲突），Git 会把   │
│   这条命令重新放回 todo 列表的顶部等待重试，这叫 "rescheduled"。 │
│   此时 done 和 todo 各有一份相同的记录。                         │
│   因为 todo 列表里已经有这条命令了（StatusRebasing），不需要     │
│   额外再添加一个冲突提交，否则会重复显示。                       │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─ 4. Rescheduled 检测（老版本 Git bug） ─────────────────────────┐
│  len(done) >= 3 且                                              │
│  done[倒数第二项] == todo[第一条] 且                             │
│  done[最后一条] == done[倒数第三条]？                            │
│  是 → 返回 nil                                                   │
│                                                                  │
│  说明（注释 427-431 行）：                                       │
│   老版本 Git 有 bug：命令 rescheduled 时，会把"上一个成功的     │
│   命令"再追加一份到 done 末尾。需要额外检测这种情况，避免误判。  │
│                                                                  │
│  三个比较项的关系：                                              │
│    ① done[倒数第二项]  ==  ② todo[第一项]                        │
│    ③ done[最后一项]  ==  ④ done[倒数第三项]                      │
│    两个等式必须同时成立                                          │
│                                                                  │
│  场景 A：真正的 bug 场景（来自单元测试）                          │
│    原命令序列：pick deadbeaf → pick fa1afe1                      │
│    fa1afe1 失败被 rescheduled 后：                                │
│      done = [pick deadbeaf, pick fa1afe1, pick deadbeaf]         │
│                                          ↑ 重复了（bug 导致）    │
│      todo = [pick fa1afe1]                                       │
│    检测：                                                        │
│      ① done[1] = pick fa1afe1 == ② todo[0] = pick fa1afe1 ✓     │
│      ③ done[2] = pick deadbeaf == ④ done[0] = pick deadbeaf ✓   │
│      → 判定为 rescheduled，返回 nil                              │
│                                                                  │
│  场景 B：防止误判的场景（来自代码注释）                          │
│    原命令序列：pick A → exec make → pick B → exec make           │
│    pick B 失败（无 bug）：                                        │
│      done = [pick A, exec make, pick B]                          │
│      todo = [exec make]                                          │
│    如果只有第一个条件：                                          │
│      done[1] = exec make == todo[0] = exec make ✓                │
│      → 误判为 rescheduled                                        │
│    加上第二个条件：                                              │
│      done[2] = pick B == done[0] = pick A? ✗                     │
│      → 不判定，正确返回冲突                                      │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─ 5. 如果 done 最后一条是 edit 命令 ─────────────────────────┐
│  ┌─ 5a. amend 文件存在？ ───────────────────────────────┐   │
│  │  是 → 返回 nil                                        │   │
│  │                                                        │   │
│  │  注释 454-455 行：                                    │   │
│  │  "if the 'amend' file exists, the 'edit' command      │   │
│  │   was successful, otherwise it wasn't"                │   │
│  │                                                        │   │
│  │  含义：amend 文件是 Git 在 edit 命令成功应用提交后     │   │
│  │  自动创建的。它存在 = edit 命令成功了，正在等用户      │   │
│  │  编辑。这是正常停止，不是冲突。                        │   │
│  └───────────────────────────────────────────────────────┘   │
│         │                                                     │
│         ▼ 否（amend 不存在）                                  │
│  ┌─ 5b. message 文件不存在？ ───────────────────────────┐   │
│  │  是 → 返回 nil                                        │   │
│  │                                                        │   │
│  │  注释 459-463 行：                                    │   │
│  │  message 文件不存在说明有其他操作（如 multi-commit     │   │
│  │  cherry-pick 或 revert）干扰了 rebase 状态，删除了    │   │
│  │  amend 和 message 文件。这种情况不判定为冲突。        │   │
│  └───────────────────────────────────────────────────────┘   │
│         │                                                     │
│         ▼ 否（有 message 但没有 amend）                       │
│    继续往下走（edit 命令在应用阶段冲突了）                    │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
┌─ 6. done 最后一条没有 commit hash？ ──────────────┐
│  是 → 返回 nil（安全检查，没有提交 hash 无法显示） │
└────────────────────────────────────────────────────┘
    │
    ▼
┌─ 7. 其他情况 → 判定为冲突 ───────────────────────────────┐
│  返回 StatusConflicted 的 Commit 对象                     │
│                                                           │
│  包含的场景：                                              │
│   - pick / squash / fixup / drop / merge 等命令冲突       │
│   - edit 命令在应用阶段冲突（有 message，没有 amend）      │
└───────────────────────────────────────────────────────────┘
```

### 3.3 Edit 状态的三种组合（对照测试用例）

以下三个测试用例来自 [pkg/commands/git_commands/commit_loader_test.go](pkg/commands/git_commands/commit_loader_test.go) 第 476-516 行，清晰展示了 done 最后一条是 edit 时的三种情况：

| 场景 | done 最后一条 | amend 文件 | message 文件 | 判定结果 | 代码分支 | 含义 |
|------|-------------|-----------|-------------|----------|----------|------|
| **edit 正常停止** | edit commit-X | 存在 | 存在/不存在 | **不是冲突**（nil） | 5a | edit 成功应用了提交，Git 自动创建了 amend 文件，正在等用户编辑。这是正常状态。 |
| **edit 应用阶段冲突** | edit commit-X | 不存在 | 存在 | **是冲突**（StatusConflicted） | 穿透 5a 和 5b，到第 7 步 | edit 命令在应用（pick）阶段就冲突了，还没到创建 amend 文件那一步。 |
| **状态被干扰** | edit commit-X | 不存在 | 不存在 | **不是冲突**（nil） | 5b | 其他操作（如 cherry-pick/revert）删除了 amend 和 message 文件，状态不可信。 |

### 3.4 Edit 冲突 vs Edit 正常停止：状态对比

这是理解整个系统的关键。让我们把 edit 命令的执行过程拆解，观察各状态文件的变化：

```
edit 命令完整生命周期：

  阶段 0：命令还在 todo 列表里
    git-rebase-todo 顶部 = edit commit-X
    done 文件中没有 edit
    ─────────────────────────────
    rebase 还没处理到这条命令

  阶段 1：Git 取出 edit 命令，开始处理
    Git 把 edit commit-X 追加到 done 文件末尾
    （无论后续成功还是失败，先写入 done）
    ─────────────────────────────
    done 最后一条 = edit commit-X

  阶段 2：尝试应用提交（pick 阶段）
    ├─ 应用成功
    │   ├─ Git 创建 message 文件（保存原始提交信息）
    │   ├─ Git 创建 amend 文件（标记 edit 模式）
    │   ├─ 设置 stopped-sha = commit-X
    │   └─ 停下，等待用户
    │     ─────────────────────────────
    │     done 最后一条 = edit commit-X
    │     amend 存在 ✓
    │     message 存在 ✓
    │     → 代码判定：不是冲突（走分支 5a）
    │     → UI 显示：正常，用户可以编辑
    │
    └─ 应用失败（冲突）
        ├─ Git 创建 message 文件
        ├─ 不创建 amend 文件（还没到那一步）
        ├─ 设置 stopped-sha = commit-X
        ├─ 可能把 edit 命令重新放回 todo 顶部（rescheduled）
        └─ 停下，返回错误
          ─────────────────────────────
          如果没有 rescheduled：
            done 最后一条 = edit commit-X
            amend 不存在 ✗
            message 存在 ✓
            → 代码判定：是冲突（到第 7 步）
            → UI 显示：StatusConflicted 的冲突提交

          如果被 rescheduled：
            todo 第一条 = edit commit-X
            done 最后一条 = edit commit-X
            → 代码判定：不是冲突（走分支 3，rescheduled）
            → UI 显示：todo 列表里有 edit commit-X（StatusRebasing）
                       用户通过文件视图的 UU 文件感知冲突

  阶段 3：用户执行 git commit --amend（可选）
    amend 文件已经存在（阶段 2 成功时创建的）
    此操作不改变 amend 文件的存在性
    ─────────────────────────────
    状态不变：仍然不是冲突

  阶段 4：用户执行 git rebase --continue
    Git 把当前 edit 命令标记为已完成（已经在 done 里了）
    从 todo 取下一条命令继续
    ─────────────────────────────
    rebase 继续运行
```

### 3.5 非 Edit 命令冲突的情况

对于 pick / squash / fixup / drop / merge 等非 edit 命令，判定逻辑更简单：

如果 rebase 停下了，且 done 最后一条是这些命令之一，说明该命令执行失败了。因为：
- 这些命令如果成功执行，会继续下一条，不会停下
- 只有失败（冲突）才会停下来

除非它被 rescheduled（检测 done[last] == todo[first]），否则直接判定为冲突。

**典型场景**：`AmendCommitWithConflict` 集成测试（[pkg/integration/tests/interactive_rebase/amend_commit_with_conflict.go](pkg/integration/tests/interactive_rebase/amend_commit_with_conflict.go)）

用户对历史提交执行 AmendToCommit，Lazygit 先创建一个 fixup commit，然后启动 rebase 把它移动到目标提交下方。fixup 命令应用时发生冲突：

```
此时状态：
  done 最后一条 = fixup commit-X
  todo 第一条   = pick commit-Y（下一个要执行的）
  fixup != pick → 不是 rescheduled
  不是 edit → 跳过 edit 分支
  commit hash 存在
  → 判定为冲突

UI 显示：
  --- Pending rebase todos ---
  pick commit-Y                          ← todo 列表，倒序
  fixup <-- CONFLICT --- fixup! target   ← 额外的冲突提交（StatusConflicted）
  --- Commits ---
  真实提交...
```

### 3.6 Rescheduled 的深入理解

**什么情况下会发生 rescheduled？**

根据代码注释（第 416-418 行）：
- 补丁会覆盖未跟踪的文件
- exec 命令执行失败
- 发生合并冲突（某些 Git 版本）

**rescheduled 发生时，为什么不需要额外显示冲突提交？**

因为该命令已经存在于 todo 列表的第一条（状态为 `StatusRebasing`）。虽然它没有被标记为红色 "CONFLICT"，但用户可以通过文件视图中的 UU（未合并）文件感知到冲突状态。

如果此时再额外添加一个 StatusConflicted 的提交，就会出现重复 —— 同一个提交既出现在 todo 列表里，又出现在冲突位置上。

**两种 rescheduled 检测的区别：**

| 检测方式 | 代码位置 | 适用场景 | 判定条件 |
|----------|----------|----------|----------|
| 标准检测 | L422-424 | 新版 Git 正常情况 | `done[最后一条] == todo[第一条]` |
| Bug 检测 | L446-450 | 老版 Git bug 情况 | `done[倒数第二条] == todo[第一条]` 且 `done[最后一条] == done[倒数第三条]` |

**Bug 检测的三个比较项：**

```
done 列表：[ ... , X , Y , X ]
           ↑       ↑   ↑   ↑
           │       │   │   └─ 最后一条（重复的 X，bug 导致）
           │       │   └─ 倒数第二条（被 rescheduled 的命令 Y）
           │       └─ 倒数第三条（原始的 X）
           └─ ...

比较关系：
  倒数第二条 Y == todo[0] Y     ✓
  最后一条 X == 倒数第三条 X     ✓
  → 判定为 rescheduled
```

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
当前状态：stopped（可能是冲突停下，也可能是 edit/break/reword 正常停下）
    │
    │  用户执行 continue
    ▼
1. 提交当前改动
   - edit 模式且有 staged → git commit --amend
   - 冲突已解决 → git commit（完成合并）
   - 没有改动 → 直接继续
    │
    ▼
2. 当前命令已经在 done 里了（从 todo 取出时就写入了）
   如果是 edit 且用户 amend 过，amend 文件可能还在
    │
    ▼
3. 从 git-rebase-todo 顶部取下一条命令
    │
    ▼
4. 把这条命令追加到 done 文件末尾（标记为"开始处理"）
    │
    ▼
5. 执行该命令
   ├─ 成功
   │   ├─ 如果是 edit → 创建 amend 和 message 文件，停下
   │   ├─ 如果是 break/exec/reword → 停下
   │   └─ 如果是 pick/squash/fixup → 回到步骤 3，继续下一条
   ├─ 失败（冲突）
   │   ├─ 创建 message 文件（如果需要）
   │   ├─ 不创建 amend 文件（如果是 edit 且在应用阶段失败）
   │   ├─ 可能把该命令重新放回 todo 顶部（rescheduled）
   │   └─ 停止，返回错误
   └─ ...
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
- 在 cherry-pick/revert sequencer 中的 update-ref / exec

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

## 八、典型场景详解

### 场景 A：Edit + Amend 正常流程（无冲突）

以集成测试 `EditAndAutoAmend`（[pkg/integration/tests/interactive_rebase/edit_and_auto_amend.go](pkg/integration/tests/interactive_rebase/edit_and_auto_amend.go)）为例：修改历史中第 2 个提交（commit-02）。

提交历史：
```
commit-03 (最新，HEAD)
commit-02 (目标：edit 它)
commit-01 (最旧，作为 rebase base)
```

#### 阶段 1：启动 Rebase

用户选中 commit-02，按 `e`（edit）：
- `BeginInteractiveRebaseForCommit` 被调用
- 构造 `ChangeTodoAction`：将 commit-02 从 pick 改为 edit
- `PrepareInteractiveRebaseCommand` 构建 `git rebase -i commit-01` 命令
- 设置 daemon 指令为 `ChangeTodoActions`
- 启动 git 命令

Git 内部：
1. 启动交互式 rebase，reset 到 base（commit-01）
2. 调用 GIT_SEQUENCE_EDITOR（lazygit daemon）
3. daemon 修改 git-rebase-todo 文件（pick commit-02 → edit commit-02）
4. Git 继续执行

#### 阶段 2：处理 commit-02（应用成功，正常停下）

Git 从 todo 取出 edit commit-02：
1. 把 `edit commit-02` 追加到 done 文件末尾
2. 尝试应用 commit-02 的补丁
3. 应用成功！
4. 创建 `message` 文件（保存 commit-02 的原始提交信息）
5. 创建 `amend` 文件（标记进入 edit 模式）
6. 设置 `stopped-sha = commit-02`
7. 停下，等待用户

此时 `.git/rebase-merge/` 中的状态：
- `git-rebase-todo`：`pick commit-03`（剩下的）
- `done`：`edit commit-02`
- `stopped-sha`：`commit-02`
- `amend`：**存在**（Git 自动创建的）
- `message`：存在

Lazygit 加载状态：
1. `getRebasingCommits` 读取 todo → `[pick commit-03]`
2. 调用 `getConflictedCommit`：
   - done 最后一条 = `edit commit-02`
   - 不是 break/exec/reword → 继续
   - `done[last]=edit 02` vs `todo[first]=pick 03` → 不是 rescheduled
   - lastTodo 是 edit，检查 amend：**amend 存在** → **返回 nil（不是冲突）**
3. todo 倒序 + Prepend → `[pick commit-03 (StatusRebasing)]`
4. 冲突提交 = nil，不追加

最终 UI 显示：
```
--- Pending rebase todos ---
pick commit-03                ← StatusRebasing
--- Commits ---
commit-02 (HEAD，当前)         ← 真实提交，正常显示
commit-01
```

没有冲突标记！一切正常。

#### 阶段 3：用户修改并自动 Amend

用户：
1. 创建新文件并 stage
2. 按 continue

Lazygit 执行 `git rebase --continue`：
- Git 检测到有 staged 改动 + amend 文件存在 → 自动执行 `git commit --amend`
- commit-02 被修改（哈希变化）
- edit 命令完成

Git 继续：
1. 从 todo 取出 pick commit-03
2. 追加到 done 文件末尾
3. 应用 commit-03 → 成功
4. 没有更多 todo → 删除 rebase-merge 目录
5. rebase 完成

#### 阶段 4：完成

UI 刷新：
- `WorkingTreeState.Rebasing` = false
- 不再加载 todo 项
- 显示最终的提交列表（commit-02 哈希已变）

---

### 场景 B：Edit 应用阶段冲突

假设在场景 A 的阶段 2，commit-02 在应用时发生冲突：

Git 处理 edit commit-02：
1. 把 `edit commit-02` 追加到 done 文件末尾
2. 尝试应用 commit-02 的补丁 → **冲突！**
3. 创建 `message` 文件（准备了原始提交信息）
4. **不创建 amend 文件**（应用阶段就失败了，还没到那一步）
5. 设置 `stopped-sha = commit-02`
6. 可能把 `edit commit-02` 重新放回 todo 顶部（rescheduled，取决于 Git 版本）
7. 停下，返回错误

有两种子情况：

#### 子情况 B1：没有被 rescheduled

此时 `.git/rebase-merge/` 中的状态：
- `git-rebase-todo`：`pick commit-03`
- `done`：`edit commit-02`
- `amend`：**不存在**
- `message`：存在

Lazygit 判定：
- done 最后一条 = `edit commit-02`
- 不是 break/exec/reword
- done[last] != todo[first] → 不是 rescheduled
- lastTodo 是 edit：
  - amend 不存在 → 继续检查
  - message 存在 → 不返回 nil，继续往下
- commit hash 存在 → **返回 StatusConflicted**

UI 显示：
```
--- Pending rebase todos ---
pick commit-03                          ← StatusRebasing
edit <-- CONFLICT --- commit-02         ← StatusConflicted（额外追加的）
--- Commits ---
commit-01
```

#### 子情况 B2：被 rescheduled

此时 `.git/rebase-merge/` 中的状态：
- `git-rebase-todo`：`edit commit-02`（被放回顶部）, `pick commit-03`
- `done`：`edit commit-02`
- `amend`：不存在
- `message`：存在

Lazygit 判定：
- done 最后一条 = `edit commit-02`
- `done[last] == todo[first]` → **是 rescheduled，返回 nil**

UI 显示：
```
--- Pending rebase todos ---
pick commit-03                          ← StatusRebasing
edit commit-02                          ← StatusRebasing（不是红色冲突标记）
--- Commits ---
commit-01
```

用户通过文件视图中的 UU 文件感知冲突。

#### 子情况 B3：被 rescheduled 且触发老版本 Git bug

假设 Git 版本有 bug，且之前有一个成功的 `pick commit-01`：

```
原命令序列：pick commit-01 → edit commit-02 → pick commit-03
```

Git 处理 edit commit-02 时冲突，被 rescheduled，且触发 bug：
1. `pick commit-01` 成功 → `done = [pick commit-01]`
2. `edit commit-02` 被取出 → `done = [pick commit-01, edit commit-02]`
3. `edit commit-02` 冲突，rescheduled → `todo = [edit commit-02, pick commit-03]`
4. **Bug 触发**：上一个成功命令 `pick commit-01` 被再次追加 → `done = [pick commit-01, edit commit-02, pick commit-01]`

此时 `.git/rebase-merge/` 中的状态：
- `git-rebase-todo`：`edit commit-02`, `pick commit-03`
- `done`：`pick commit-01`, `edit commit-02`, `pick commit-01`（最后一条重复了）
- `amend`：不存在
- `message`：存在

Lazygit 判定：
- done 最后一条 = `pick commit-01`
- 不是 break/exec/reword
- `done[last] = pick commit-01` != `todo[first] = edit commit-02` → 标准检测不命中
- `len(done) >= 3` ✓
- `done[倒数第二] = edit commit-02` == `todo[第一] = edit commit-02` ✓
- `done[最后] = pick commit-01` == `done[倒数第三] = pick commit-01` ✓
- → **老版本 bug 检测命中，返回 nil**

UI 显示与 B2 相同，没有红色冲突标记。

---

### 场景 C：Fixup 冲突（非 edit 命令）

以集成测试 `AmendCommitWithConflict`（[pkg/integration/tests/interactive_rebase/amend_commit_with_conflict.go](pkg/integration/tests/interactive_rebase/amend_commit_with_conflict.go)）为例：

用户对历史提交 "two" 执行 AmendToCommit（把暂存改动合入 "two"）：
1. Lazygit 先创建一个 fixup commit（`fixup! two`）
2. 启动 rebase，把 fixup commit 移动到 "two" 下方
3. fixup commit 在应用时冲突

此时状态：
- `git-rebase-todo`：`pick three`
- `done`：`..., pick two, fixup! two`（最后一条是 fixup）
- fixup != pick three → 不是 rescheduled
- 不是 edit → 跳过 edit 分支
- commit hash 存在 → **返回 StatusConflicted**

UI 显示：
```
--- Pending rebase todos ---
pick three                                ← StatusRebasing
fixup <-- CONFLICT --- fixup! two         ← StatusConflicted
--- Commits ---
two
one
```

---

## 九、重要设计要点

### 1. Daemon 模式的巧思

利用 Git 的编辑器回调机制，让 lazygit 能以非侵入方式程序化控制交互式 rebase。相比手动调用 `git rebase --edit-todo` 或解析 git 输出，这种方式更稳定、更符合 Git 的设计哲学。

### 2. Todo 与真实提交的无缝融合

CommitLoader 将待办列表"水化"后插入真实提交列表前面，让用户在 UI 上看到连贯的视图 —— 仿佛 rebase 已经完成、只是还没"落实"。这种设计大大降低了用户的认知负担。

### 3. Rescheduled 检测的细腻处理

Git 在命令失败时会把命令重新放回 todo 队列顶部（rescheduled），导致 done 和 todo 出现重复项。代码中通过两层检测（标准情况 + 老版本 Git bug）处理 rescheduled 状态，避免 UI 上出现重复或错误的冲突提示。

### 4. Edit 状态的三级判定

Edit 命令是唯一需要通过辅助文件（amend/message）进一步区分的命令：
1. amend 存在 → edit 成功应用，正常停下
2. amend 不存在但 message 存在 → edit 在应用阶段冲突
3. 两者都不存在 → 状态被干扰，不判定

其中 amend 文件的存在是"edit 成功"的决定性标志 —— 它是 Git 在 edit 命令成功应用后自动创建的，不是用户 amend 之后才创建的。

### 5. "Done" 的含义是"已处理"，不是"已成功"

Done 文件记录的是所有被 Git 从 todo 取出并开始处理的命令，无论它们最终是成功、失败还是暂停等待。所以 done 最后一条可能是成功完成的命令，也可能是正在等用户的 edit，还可能是失败的冲突命令。需要结合 amend/message 文件和 todo 列表才能判断具体状态。

### 6. onSuccessfulContinue 回调链

通过注册回调函数的方式，支持多步骤的 rebase 操作（如批量 amend 多个 commit）。每步完成后自动触发下一步，对用户透明。

### 7. 多状态叠加与优先级

WorkingTreeState 支持四态叠加，Effective 状态确保用户永远处理最内层的操作。例如 rebase 中的 cherry-pick 冲突，用户必须先解决 cherry-pick，才能继续 rebase。

### 8. 移动时跳过非渲染项

移动 todo 时跳过 label/reset/comment/break 等不可见项，保证用户移动提交时的直觉正确 —— 按一下移动一个"可见提交"，而不是卡在看不见的内部命令上。
