# Lazygit Merge 与冲突解决流程分析

## 一、整体流程图概览

```
用户选择分支 → 弹出 Merge 菜单 → 选择合并策略 → 执行 git merge
                                                        ↓
                                              命令执行结果检查
                                            ┌───┴───┐
                                            │       │
                                         成功     冲突
                                          │        │
                                    刷新状态    检测冲突状态
                                                        ↓
                                              弹出冲突处理菜单
                                            ┌───┴───────────┐
                                            │               │
                                     查看冲突文件       Abort 合并
                                            │
                                     进入冲突编辑视图
                                  ┌─────────┼─────────┐
                                  │         │         │
                            Pick Top    Pick Both   Pick Bottom
                                  │         │         │
                                  └─────────┼─────────┘
                                            ↓
                                    所有冲突已解决?
                                            ↓ (是)
                              弹出确认 Continue 提示
                                            ↓
                                    执行 merge --continue
```

---

## 二、合并触发流程

### 2.1 UI 触发入口（分支面板）

合并操作的触发起点在分支面板的快捷键绑定：

- **文件**: [branches_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/branches_controller.go#L134-L139)
- **快捷键**: `Config.Branches.MergeIntoCurrentBranch`
- **处理函数**: `self.merge()` → 调用 [BranchesController.merge](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/branches_controller.go#L682-L685)

```go
func (self *BranchesController) merge() error {
    selectedBranchName := self.context().GetSelected().Name
    return self.c.Helpers().MergeAndRebase.MergeRefIntoCheckedOutBranch(selectedBranchName)
}
```

### 2.2 合并策略菜单

`MergeRefIntoCheckedOutBranch` 会弹出合并选项菜单，支持 4 种合并方式：

- **文件**: [merge_and_rebase_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L375-L489)

| 菜单项 | 快捷键 | Git 参数 |
|--------|--------|----------|
| Regular Merge (Fast-Forward) | `m` / `f` | `--ff` |
| Regular Merge (Non-Fast-Forward) | `n` / `m` | `--no-ff` |
| Squash Merge (Uncommitted) | `s` | `--squash --ff` |
| Squash Merge (Committed) | `S` | `--squash --ff` + 自动 commit |

合并策略会根据用户配置（`Git.Merging.Args`）和 git 配置（`merge.ff`）智能调整菜单项顺序和可用性。

### 2.3 实际执行 Git Merge 命令

最终调用 Git 命令层执行合并：

- **文件**: [branch.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/branch.go#L262-L286)
- **函数**: `BranchCommands.Merge(branchName string, variant MergeVariant)`

实际执行的命令结构：
```bash
git merge --no-edit ${用户配置的 Merging.Args} ${策略参数} ${分支名}
```

其中 `--no-edit` 是固定参数，避免合并时弹出编辑器。

### 2.4 命令执行后的结果检查

Merge 命令返回后，会经过 [CheckMergeOrRebase](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L168-L170) 流程：

1. 先调用 `Refresh()` 刷新文件视图
2. 对错误信息进行分类处理：
   - **"No changes"** → 自动执行 `--skip`
   - **"No rebase in progress?"** → 静默忽略
   - **冲突关键字匹配** → 进入 `PromptForConflictHandling()`

冲突关键字匹配由 [isMergeConflictErr](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L142-L150) 函数完成，识别以下特征字符串：
- `"Failed to merge in the changes"`
- `"fix conflicts"`
- `"CONFLICT (content):"`
- `"Merge conflict in file"`
- 等 7 种模式

---

## 三、冲突状态识别机制

冲突状态分为 **仓库级别** 和 **文件级别** 两层。

### 3.1 仓库级别：WorkingTreeState

定义在 [working_tree_state.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/working_tree_state.go#L9-L14)：

```go
type WorkingTreeState struct {
    Rebasing      bool
    Merging       bool
    CherryPicking bool
    Reverting     bool
}
```

#### 3.1.1 状态检测实现

状态通过检查 `.git` 目录下的特定文件来判断：

- **文件**: [status.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/status.go#L23-L83)
- **函数**: `StatusCommands.WorkingTreeState()`

| 状态 | 检测文件 |
|------|----------|
| `Rebasing` | `rebase-merge/` 或 `rebase-apply/` 目录存在 |
| `Merging` | `MERGE_HEAD` 文件存在 |
| `CherryPicking` | `CHERRY_PICK_HEAD` 文件存在（需排除 rebase 残余） |
| `Reverting` | `REVERT_HEAD` 文件存在 |

**CherryPick 特殊处理**: 由于历史原因，rebase 过程中可能残留 `CHERRY_PICK_HEAD`。代码通过比较 `CHERRY_PICK_HEAD` 与 `rebase-merge/stopped-sha` 的值来区分真正的 cherry-pick 冲突。

#### 3.1.2 Effective State 优先级

当多种状态同时存在时（如 Rebase + CherryPick），通过 [Effective()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/working_tree_state.go#L45-L59) 方法决定优先处理哪个：

**优先级从高到低**: `Reverting` → `CherryPicking` → `Merging` → `Rebasing`

原因：必须先解决内层操作（如 cherry-pick）才能继续外层的 rebase。

### 3.2 文件级别：冲突状态字段

Git 状态输出中的 `shortStatus` 被解析为文件级别的冲突标记：

- **文件**: [file.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/file.go#L146-L163)
- **函数**: `deriveStatusFields(shortStatus string)`

两个关键布尔字段：

| 字段 | 含义 | 对应 shortStatus |
|------|------|------------------|
| `HasInlineMergeConflicts` | 文件包含 `<<<<<<<` 等冲突标记 | `UU`（双方修改）、`AA`（双方新增） |
| `HasMergeConflicts` | 广义冲突（含非内联冲突） | 上述 + `DD`/`AU`/`UA`/`UD`/`DU` |

**注意**: 即使 `git status` 显示 `UU`，用户可能已在外部编辑器中手动解决了冲突。因此进入冲突视图前还会通过 `FileHasConflictMarkers()` 二次确认文件中是否真的存在标记。

---

## 四、冲突解决流程与操作入口

### 4.1 冲突出现后的初始菜单

当 `CheckForConflicts` 检测到冲突后，调用 [PromptForConflictHandling()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L184-L206)：

```
┌─ 发现冲突 ──────────────────────────┐
│  查看冲突 (跳转到 Files 面板)        │
│  中止 merge/rebase/cherry-pick (a)  │
└──────────────────────────────────────┘
```

### 4.2 Files 面板：进入冲突编辑视图

在 Files 面板选中冲突文件并按 Enter，触发不同处理逻辑：

- **文件**: [files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L692-L697)

```go
if file.HasInlineMergeConflicts {
    return self.switchToMerge()    // 进入内联冲突编辑器
}
if file.HasMergeConflicts {
    return self.handleNonInlineConflict(file)  // 弹出策略菜单
}
```

#### 4.2.1 跳转 MergeConflicts 上下文

`switchToMerge()` 最终调用：

- **文件**: [merge_conflicts_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_conflicts_helper.go#L85-L98)
- **函数**: `MergeConflictsHelper.SwitchToMerge(path string)`

该方法：
1. 读取文件内容，调用 `findConflicts()` 解析所有冲突块
2. 将 `MergeConflictsContext` 压入上下文栈
3. 渲染带颜色标记的文件内容

### 4.3 MergeConflicts 控制器：核心冲突编辑

- **文件**: [merge_conflicts_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/merge_conflicts_controller.go#L28-L114)

#### 4.3.1 键绑定操作一览

| 操作 | 默认键 | 功能 |
|------|--------|------|
| `PickHunk` | 空格 / Enter | 采纳当前选中的 hunk |
| `PickAllHunks` | `b` | 采纳全部 (ours + theirs) |
| `SelectPrevHunk` | ↑ / `k` | 在同一冲突块内切换选择 TOP/MIDDLE/BOTTOM |
| `SelectNextHunk` | ↓ / `j` | 同上，向后切换 |
| `PrevConflict` | `<` / `,` | 跳转到上一个冲突块 |
| `NextConflict` | `>` / `.` | 跳转到下一个冲突块 |
| `Undo` | `z` | 撤销上一次 resolve 操作（内容栈机制） |
| `EditFile` | `e` | 在外部编辑器打开当前行 |
| `OpenMergeConflictMenu` | `o` | 打开 merge 策略菜单 |
| `Return` | Esc | 返回 Files 面板 |

#### 4.3.2 冲突块解析机制

- **文件**: [find_conflicts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/mergeconflicts/find_conflicts.go#L25-L58)
- **函数**: `findConflicts(content string)`

逐行扫描识别 4 种冲突标记：

| 标记 | 行类型 | 含义 |
|------|--------|------|
| `<<<<<<< ` | `START` | ours 版本开始（通常为当前分支） |
| `\|\|\|\|\|\|\| ` | `ANCESTOR` | 共同祖先版本（仅 diff3 格式） |
| `=======` | `TARGET` | 分隔符，以下为 theirs 版本 |
| `>>>>>>> ` | `END` | 冲突块结束 |

每个冲突块被解析为 `mergeConflict` 结构，记录这 4 行的行号。如果有 ANCESTOR 行则可选择 TOP/MIDDLE/BOTTOM，否则只有 TOP/BOTTOM 两种选择。

#### 4.3.3 采纳 (Pick) 的实现原理

- **文件**: [merge_conflict.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/mergeconflicts/merge_conflict.go#L34-L69)

`Selection.isIndexToKeep()` 方法决定每一行是否保留：

- **TOP**: 保留 `START` ~ `ANCESTOR`/`TARGET` 之间的行（ours）
- **MIDDLE**: 保留 `ANCESTOR` ~ `TARGET` 之间的行（base）
- **BOTTOM**: 保留 `TARGET` ~ `END` 之间的行（theirs）
- **ALL**: 保留全部非标记行（ours + theirs 拼接）

冲突标记行（START/ANCESTOR/TARGET/END）始终被丢弃。

#### 4.3.4 Undo 内容栈机制

- **文件**: [state.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/mergeconflicts/state.go#L15-L17)

`State.contents` 是一个字符串栈：
- 初始加载时压入原始内容
- 每次 resolve 成功后，将新内容压入栈
- Undo 时弹出栈顶，恢复上一版内容并重新解析冲突

### 4.4 冲突策略菜单（非逐行解决方式）

对于不想逐行 pick 的用户，提供批量策略菜单：

- **文件**: [working_tree_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/working_tree_helper.go#L355-L417)
- **入口**: Files 面板选中冲突文件按 `o`，或 MergeConflicts 视图按 `o`

| 菜单项 | 快捷键 | 等价命令 |
|--------|--------|----------|
| Use Current Changes (ours) | `c` | `git merge-file --ours` |
| Use Incoming Changes (theirs) | `i` | `git merge-file --theirs` |
| Use Both Changes (union) | `b` | `git merge-file --union` |
| Open Merge Tool | `m` | `git mergetool` |

**实现细节**: 通过临时文件（旧版 git）或 ObjectID（git ≥ 2.43）获取 stage 1/2/3 的内容，再调用 `git merge-file` 合并。

---

## 五、冲突解决后的后续动作

### 5.1 自动检测：最后一个冲突解决后

当 `AllConflictsResolved()` 返回 true 时：

- **文件**: [merge_conflicts_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/merge_conflicts_controller.go#L264-L266)

```go
if self.context().GetState().AllConflictsResolved() {
    self.onLastConflictResolved()  // 触发 FILES 范围刷新
}
```

### 5.2 Continue 提示弹出

Files 刷新后，如果检测到：
1. 仓库仍处于 Merging/Rebasing 等状态
2. 所有文件的冲突都已解决（不再有 `HasMergeConflicts`）

则会触发 [PromptToContinueRebase()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L223-L263)：

```
┌─ Continue ─────────────────────────────────────┐
│  Conflicts resolved, continue merge?           │
│  （如果有未暂存的修改还会二次询问是否自动 stage）│
└────────────────────────────────────────────────┘
```

### 5.3 统一的 Continue/Abort/Skip 执行

所有后续动作统一由 [genericMergeCommand()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L74-L118) 处理：

1. 根据 `WorkingTreeState.Effective()` 决定 `commandType`（merge/rebase/cherry-pick/revert）
2. 构造命令 `git ${commandType} --${command}`
3. 是否需要子进程：
   - **merge continue** + 用户配置 `ManualCommit` → 子进程
   - **rebase continue** + 存在 exec todo → 子进程
   - 其他 → 内嵌执行（跳过编辑器）

实际的 Git 命令执行在 [rebase.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/rebase.go#L482-L503) 的 `GenericMergeOrRebaseAction`，通过设置环境变量 `GIT_EDITOR` 为 lazygit 自身来避免弹出编辑器。

### 5.4 Rebase Options 菜单入口

除了 Continue 提示外，用户也可以通过 `RebaseOptionsTitle` 菜单手动操作：

- **文件**: [merge_and_rebase_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L39-L68)

菜单项（按当前有效状态动态生成）：
| 菜单项 | 快捷键 | 条件 |
|--------|--------|------|
| continue | `c` | 始终显示 |
| abort | `a` | 始终显示 |
| skip | `s` | Rebasing/CherryPicking/Reverting 时显示 |

---

## 六、关键数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Controller 层                               │
│  ┌─────────────────────┐    ┌──────────────────────────────────┐    │
│  │ BranchesController  │    │ MergeConflictsController         │    │
│  │  - merge()          │───▶│  - HandlePickHunk()              │    │
│  └──────────┬──────────┘    │  - openMergeConflictMenu()       │    │
│             │               └───────────────┬──────────────────┘    │
│             ▼                               ▼                       │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │              MergeAndRebaseHelper                        │      │
│  │  - MergeRefIntoCheckedOutBranch() [弹出合并菜单]          │      │
│  │  - CheckMergeOrRebase() [检查冲突]                       │      │
│  │  - genericMergeCommand() [continue/abort/skip]           │      │
│  │  - PromptForConflictHandling() [冲突初始菜单]             │      │
│  │  - PromptToContinueRebase() [解决后提示]                  │      │
│  └───────────────┬───────────────────────────┬──────────────┘      │
│                  │                           │                     │
│                  ▼                           ▼                     │
│  ┌──────────────────────────────┐  ┌─────────────────────────┐    │
│  │    MergeConflictsHelper      │  │   WorkingTreeHelper     │    │
│  │  - SwitchToMerge()           │  │  - CreateMergeConflict- │    │
│  │  - SetMergeState()           │  │    Menu() [策略菜单]    │    │
│  │  - EscapeMerge()             │  │  - OpenMergeTool()      │    │
│  └───────────────┬──────────────┘  └────────────┬────────────┘    │
└──────────────────┼──────────────────────────────┼─────────────────┘
                   │                              │
┌──────────────────┼──────────────────────────────┼─────────────────┐
│                  ▼                              ▼                 │
│  ┌──────────────────────────────┐  ┌─────────────────────────┐    │
│  │   MergeConflictsContext      │  │      Git Commands       │    │
│  │  ┌────────────────────────┐  │  │  - Branch.Merge()      │    │
│  │  │ ConflictsViewModel     │  │  │  - Rebase.Generic-     │    │
│  │  │  - state (State)       │  │  │    MergeOrRebaseAction │    │
│  │  │  - userVerticalScroll  │  │  │  - Status.-            │    │
│  │  └───────────┬────────────┘  │  │    WorkingTreeState()  │    │
│  │              │               │  └──────────┬──────────────┘    │
│  └──────────────┼───────────────┘             │                   │
│                 ▼                             ▼                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    mergeconflicts 包                      │     │
│  │  ┌─────────────┐  ┌───────────────┐  ┌────────────────┐   │     │
│  │  │ State       │  │ mergeConflict │  │ findConflicts()│   │     │
│  │  │ - contents[]│  │  - start      │  │ - 逐行扫描标记 │   │     │
│  │  │ - conflicts│  │  - ancestor   │  └────────────────┘   │     │
│  │  │ - path     │  │  - target     │                       │     │
│  │  │ - index    │  │  - end        │                       │     │
│  │  └─────────────┘  └───────────────┘                       │     │
│  └──────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
```
