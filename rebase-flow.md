# 交互式 Rebase 代码流程梳理

## 概述

Lazygit 的交互式 rebase 系统围绕 **待办列表 (Todo List)**、**命令触发**、**冲突回退** 三个核心概念构建。整个系统通过 Git 的 `GIT_SEQUENCE_EDITOR` 机制切入，使用 lazygit 自身作为 daemon 进程来编辑 todo 文件，从而实现程序化的 rebase 操作。

---

## 一、核心文件索引

| 模块 | 文件 | 职责 |
|------|------|------|
| Git 命令层 | [rebase.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/git_commands/rebase.go) | 构建 rebase 命令、封装 continue/abort/skip |
| 待办列表工具 | [rebase_todo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/utils/rebase_todo.go) | 读写 git-rebase-todo 文件、移动/删除 todo 项 |
| Daemon 层 | [daemon.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/app/daemon/daemon.go) + [rebase.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/app/daemon/rebase.go) | 作为 Git 的编辑器被调用，执行具体的 todo 修改指令 |
| GUI 控制层 | [merge_and_rebase_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go) | rebase 选项菜单、冲突检测与处理、continue/abort 调度 |
| 提交加载器 | [commit_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/git_commands/commit_loader.go) | 从 git-rebase-todo 加载待办项并与真实提交合并展示 |
| 工作树状态 | [working_tree_state.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/models/working_tree_state.go) + [status.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/git_commands/status.go) | 检测当前是否处于 rebase/merge/cherry-pick 状态 |

---

## 二、待办列表 (Todo List) 编排

### 2.1 Todo 文件的位置与格式

Git 在交互式 rebase 进行中会在 `.git/rebase-merge/` 目录下维护以下关键文件：

- **git-rebase-todo**：待执行的 todo 列表（尚未执行的）
- **done**：已执行完成的 todo 列表
- **stopped-sha**：当前停止处的提交 SHA
- **amend** / **message**：edit 模式下的标记文件

Todo 格式由第三方库 `git-todo-parser/todo` 解析，每条记录包含：
- `Command`：pick / reword / edit / squash / fixup / exec / break / label / reset / merge / update-ref 等
- `Commit`：提交哈希
- `Flag`：如 `-C`（fixup -C 保留提交信息）
- `Msg`：提交信息或 exec 命令内容

### 2.2 Todo 列表的加载与展示

在 [commit_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/git_commands/commit_loader.go#L155-L186) 的 `MergeRebasingCommits` 方法中：

1. 先过滤掉已有的 TODO 提交（避免重复）
2. 检查 `WorkingTreeState`：
   - 如果在 cherry-pick / revert 状态，从 `.git/sequencer/todo` 加载
   - 如果在 rebase 状态，从 `.git/rebase-merge/git-rebase-todo` 加载
3. 将 todo 项"水化"（hydrate）：通过 `git show` 获取提交的完整信息（作者、日期等）
4. 以 **倒序** 插入到真实提交列表的前面（因为 git log 是新的在上，而 todo 是旧的在上）

冲突提交的识别逻辑在 [getConflictedCommit](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/git_commands/commit_loader.go#L386-L481)：
- 解析 `rebase-merge/done` 文件
- 取最后一条 done 记录，如果它不是 break/exec/reword，且没有被 rescheduled，则认为是冲突的
- 标记为 `StatusConflicted` 状态

### 2.3 Todo 的修改操作

[rebase_todo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/utils/rebase_todo.go) 提供了多种原子操作：

| 函数 | 用途 | 触发时机 |
|------|------|----------|
| `EditRebaseTodo` | 修改指定 commit 的 action（pick→edit 等） | reword / edit / squash / fixup / drop |
| `MoveTodosUp` / `MoveTodosDown` | 移动 todo 项（上/下） | 用户按上下键移动提交 |
| `DeleteTodos` | 删除 todo 项 | 删除 update-ref todo |
| `MoveFixupCommitDown` | 移动 fixup 提交到目标提交下方 | amend / fixup 操作 |
| `DropMergeCommit` | 删除合并提交的 todo 项 | 删除 merge commit |
| `PrependStrToTodoFile` | 在 todo 文件开头插入内容 | insert break |

**移动逻辑的注意点**：
- Git 的 todo 文件是 **自上而下** 执行（旧的在上）
- Lazygit 的提交视图是 **自下而上**（新的在上）
- 所以在 UI 上"向下移动"对应 todo 列表中向 **起始方向** 移动
- 移动时会跳过非渲染项（label / reset / comment 等），只在 `isRenderedTodo` 之间移动

### 2.4 已渲染 Todo 的判定

[isRenderedTodo](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/utils/rebase_todo.go#L292-L294) 决定了哪些 todo 项会显示在 UI 中：

```
有 commit hash 的 → 显示（pick/reword/edit/squash/fixup/drop/merge 等）
在 rebase 中且是 update-ref → 显示
在 rebase 中且是 exec → 显示
其他（label/reset/comment）→ 不显示
```

---

## 三、命令触发机制

### 3.1 Daemon 模式：Lazygit 作为 Git 的编辑器

Lazygit 通过将自身设置为 `GIT_SEQUENCE_EDITOR` 和 `GIT_EDITOR`，在 Git 需要编辑 todo 文件或提交信息时被调用。这就是所谓的 "daemon 模式"。

环境变量传递指令：
- `LAZYGIT_DAEMON_KIND`：daemon 类型枚举
- `LAZYGIT_DAEMON_INSTRUCTION`：JSON 序列化的指令参数

[daemon.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/app/daemon/daemon.go#L27-L42) 中定义了以下 daemon 种类：

| 类型 | 用途 |
|------|------|
| `ExitImmediately` | 直接退出（跳过编辑器） |
| `RemoveUpdateRefsForCopiedBranch` | 移除复制分支末尾的 update-ref |
| `ChangeTodoActions` | 批量修改 todo 的 action |
| `DropMergeCommit` | 删除合并提交 todo |
| `MoveFixupCommitDown` | 移动 fixup 提交 |
| `MoveTodosUp` / `MoveTodosDown` | 移动 todo 项 |
| `InsertBreak` | 插入 break |
| `WriteRebaseTodo` | 写入完整的 todo 文件内容 |

### 3.2 启动交互式 Rebase 的完整流程

以 `InteractiveRebase`（修改多个提交的 action）为例，完整链路如下：

```
用户操作 (local_commits_controller)
    ↓
interactiveRebase(action, startIdx, endIdx)
    ↓
RebaseCommands.InteractiveRebase
    ├─ 计算 baseHashOrRoot（endIdx+1，squash/fixup 则 +2）
    ├─ 构造 ChangeTodoAction 列表（每个 commit 对应一个 action 变更）
    └─ PrepareInteractiveRebaseCommand
        ├─ 构造 git rebase -i 命令
        │   ├─ --interactive
        │   ├─ --autostash
        │   ├─ --keep-empty
        │   ├─ --no-autosquash
        │   └─ --rebase-merges
        ├─ 设置 GIT_SEQUENCE_EDITOR = lazygit 路径
        ├─ 设置 GIT_EDITOR = lazygit 路径（overrideEditor 时）
        ├─ 通过环境变量传递 daemon instruction
        └─ 执行命令
            ↓
Git 启动交互式 rebase
    ↓
Git 调用 GIT_SEQUENCE_EDITOR（即 lazygit）
    ↓
daemon.Handle() 被触发
    ├─ 解析环境变量中的 instruction
    └─ instruction.run()
        └─ handleInteractiveRebase
            ├─ 检查参数是 git-rebase-todo 文件
            ├─ 执行具体的 todo 修改（如 ChangeTodoActions）
            └─ 退出 daemon
                ↓
Git 继续执行 rebase
```

### 3.3 Rebase 进行中修改 Todo

当 rebase 已经在进行中（stopped 状态），可以直接修改 `git-rebase-todo` 文件而无需重新启动 rebase。相关方法在 [rebase.go](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/git_commands/rebase.go#L338-L387)：

- `EditRebaseTodo`：直接修改 todo 文件中指定提交的 action
- `MoveTodosUp` / `MoveTodosDown`：直接移动 todo 项
- `DeleteUpdateRefTodos`：删除 update-ref todo（需要通过 `git rebase --edit-todo` 让 Git 做 housekeeping）

### 3.4 Continue / Abort / Skip

[GenericMergeOrRebaseAction](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/git_commands/rebase.go#L482-L503) 是统一入口：

```
continue → git rebase --continue
abort    → git rebase --abort
skip     → git rebase --skip
```

特殊机制：**onSuccessfulContinue 回调**
- 有些操作需要在 rebase 中分多步执行（如 GenericAmend 循环处理多个 commit）
- 通过 `self.onSuccessfulContinue` 注册下一步回调
- 每次成功 continue 后自动执行回调
- abort 时清空回调

---

## 四、冲突检测与回退机制

### 4.1 工作树状态检测

[WorkingTreeState](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/commands/models/working_tree_state.go#L9-L14) 有四个独立的布尔状态，可同时存在多个：

- `Rebasing`：检测 `rebase-merge` 或 `rebase-apply` 目录是否存在
- `Merging`：检测 `MERGE_HEAD` 文件
- `CherryPicking`：检测 `CHERRY_PICK_HEAD` 文件（需排除 rebase 过程中的残留）
- `Reverting`：检测 `REVERT_HEAD` 文件

**Effective 状态**（优先级从高到低）：
Reverting > CherryPicking > Merging > Rebasing > None

这决定了 UI 显示哪种状态的标题和操作菜单。例如，在 rebase 中执行 cherry-pick 发生冲突时，effective 状态是 cherry-picking，用户只能先处理 cherry-pick 的 continue/abort。

### 4.2 冲突检测

冲突检测发生在执行 rebase 命令之后，由 [CheckMergeOrRebase](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L168-L170) 处理：

1. 先刷新 UI
2. 如果命令返回 `nil`（成功），直接返回
3. 检查特殊情况：
   - "No changes - did you forget to use" → 自动 skip（空提交）
   - "The previous cherry-pick is now empty" → 自动 skip
   - "No rebase in progress?" → 视为已完成
4. 调用 `CheckForConflicts` 检测冲突

[isMergeConflictErr](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L142-L150) 通过匹配错误字符串判断是否是冲突：
- "Failed to merge in the changes"
- "fix conflicts"
- "CONFLICT (content):"
- 等 7 种关键字

### 4.3 冲突处理流程

检测到冲突后，弹出 [PromptForConflictHandling](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L184-L206) 菜单：

```
┌─ 发现冲突 ──────────────┐
│  查看冲突 (ViewConflicts)│  → 跳转到 Files 视图
│  中止 rebase (Abort)     │  → 执行 rebase --abort
└─────────────────────────┘
```

当用户解决冲突后（文件状态变为已暂存），通过 [PromptToContinueRebase](file:///d:/fz/0601-2/solo-dogfeeding/code/22-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L223-L263) 提示继续：

1. 刷新文件状态
2. 如果有未暂存文件，询问是否自动暂存
3. 执行 `rebase --continue`

### 4.4 回退（Abort）机制

Abort 的完整路径：
1. 用户选择 abort 或调用 `AbortRebase()`
2. 执行 `git rebase --abort`
3. Git 清理 `rebase-merge/` 目录
4. `onSuccessfulContinue` 回调被清空
5. 状态刷新：`WorkingTreeState.Rebasing` 变为 false

Abort 可以在任何时候执行，无论是否有冲突。执行后工作树回到 rebase 开始前的状态。

---

## 五、关键数据结构关系图

```
┌─────────────────────────────────────────────────────┐
│                  GUI 层                              │
│  local_commits_controller ─→ merge_and_rebase_helper │
└──────────────────────┬──────────────────────────────┘
                       │ 调用
┌──────────────────────▼──────────────────────────────┐
│              Git Commands 层                         │
│  RebaseCommands ─→ PrepareInteractiveRebaseCommand  │
│                    (设置 GIT_SEQUENCE_EDITOR)        │
└──────────────────────┬──────────────────────────────┘
                       │ 执行 git 命令
┌──────────────────────▼──────────────────────────────┐
│                    Git                               │
│  git rebase -i → 调用 $GIT_SEQUENCE_EDITOR           │
└──────────────────────┬──────────────────────────────┘
                       │ 启动 lazygit daemon
┌──────────────────────▼──────────────────────────────┐
│                 Daemon 层                            │
│  daemon.Handle → instruction.run()                   │
│                   └→ utils/rebase_todo.go            │
│                      (读写 .git/rebase-merge/...)     │
└──────────────────────┬──────────────────────────────┘
                       │ 修改文件
┌──────────────────────▼──────────────────────────────┐
│              .git/rebase-merge/                      │
│  git-rebase-todo  ──  待执行列表                     │
│  done            ──  已执行列表                     │
│  stopped-sha     ──  当前停止点                     │
│  amend / message ──  edit 状态标记                  │
└─────────────────────────────────────────────────────┘
```

---

## 六、典型场景：Edit + Amend 流程

以"修改历史中某个提交"为例，完整走一遍流程：

1. **用户选中第 N 个提交，按 e (edit)**
   - `BeginInteractiveRebaseForCommit(commits, index, false)`
   - 构造 `ChangeTodoAction` 将该提交改为 `todo.Edit`
   - 启动交互式 rebase

2. **Git 执行 rebase，停在该提交处**
   - `rebase-merge/` 目录被创建
   - `stopped-sha` 指向该提交
   - `amend` 文件存在 → 表示 edit 模式

3. **UI 显示 "You are here" 状态**
   - CommitLoader 从 done 文件识别出当前提交
   - 标记为 StatusRebasing（非冲突状态）

4. **用户修改文件并暂存，执行 amend**
   - `git commit --amend`
   - 提交被修改

5. **用户按 continue**
   - `git rebase --continue`
   - 如果有 onSuccessfulContinue 回调，执行之
   - 否则继续后续的 pick

6. **如果中途发生冲突**
   - Git 停止，返回错误
   - CheckForConflicts 检测到冲突关键字
   - 弹出冲突处理菜单
   - 用户解决冲突后 continue

7. **所有 todo 执行完毕**
   - `rebase-merge/` 目录被清理
   - WorkingTreeState 变为 None
   - 提交列表刷新，显示新的提交哈希

---

## 七、重要设计要点

1. **Daemon 模式的巧思**：利用 Git 的编辑器回调机制，让 lazygit 能以非侵入方式程序化控制交互式 rebase，而不需要手动解析和重放 git 命令。

2. **Todo 与真实提交的融合**：CommitLoader 将待办列表"水化"后插入真实提交列表前面，让用户在 UI 上看到连贯的视图，仿佛 rebase 已经完成。

3. **状态优先级**：WorkingTreeState 支持多状态叠加，Effective 状态确保用户永远处理最内层的操作（如 rebase 中的 cherry-pick 冲突）。

4. **onSuccessfulContinue 回调链**：支持多步骤的 rebase 操作（如批量 amend 多个 commit），每步完成后自动触发下一步。

5. **移动时跳过非渲染项**：保证用户移动提交时的直觉正确，不会"卡"在看不见的 label 或 comment 行上。
