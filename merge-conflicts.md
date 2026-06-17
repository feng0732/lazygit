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

## 二、合并触发流程：入口全景

合并操作有 **两个独立的 UI 入口**，最终都汇入同一个 `MergeRefIntoCheckedOutBranch` 函数。

### 2.1 入口 A：本地分支面板 (Local Branches)

- **文件**: [branches_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/branches_controller.go#L134-L139)
- **快捷键**: `Config.Branches.MergeIntoCurrentBranch`
- **处理函数**: [BranchesController.merge](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/branches_controller.go#L682-L685)

```go
func (self *BranchesController) merge() error {
    selectedBranchName := self.context().GetSelected().Name
    // 传入选中分支名（短名，如 "feature"）
    return self.c.Helpers().MergeAndRebase.MergeRefIntoCheckedOutBranch(selectedBranchName)
}
```

### 2.2 入口 B：远端分支面板 (Remote Branches)

- **文件**: [remote_branches_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/remote_branches_controller.go#L52-L58)
- **快捷键**: 同样是 `Config.Branches.MergeIntoCurrentBranch`
- **处理函数**: [RemoteBranchesController.merge](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/remote_branches_controller.go#L139-L141)

```go
func (self *RemoteBranchesController) merge(selectedBranch *models.RemoteBranch) error {
    // 传入远端分支全名（如 "origin/feature"）
    return self.c.Helpers().MergeAndRebase.MergeRefIntoCheckedOutBranch(selectedBranch.FullName())
}
```

**关键差异**: 本地分支传短名 `feature`，远端分支传全名 `origin/feature`。底层 `git merge` 都能识别这两种 ref。

### 2.3 合并策略菜单（共享）

两个入口都调用 [MergeAndRebaseHelper.MergeRefIntoCheckedOutBranch](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L375-L489)，弹出 4 种合并方式：

| 菜单项 | 快捷键 | Git 参数 |
|--------|--------|----------|
| Regular Merge (Fast-Forward) | `m` / `f` | `--ff` |
| Regular Merge (Non-Fast-Forward) | `n` / `m` | `--no-ff` |
| Squash Merge (Uncommitted) | `s` | `--squash --ff` |
| Squash Merge (Committed) | `S` | `--squash --ff` + 自动 commit |

菜单项顺序由用户配置 `Git.Merging.Args` 或 git 配置 `merge.ff` 决定（如果用户偏好 ff，则 ff 项排第一）。

### 2.4 实际执行 Git Merge 命令

最终调用 Git 命令层：

- **文件**: [branch.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/branch.go#L262-L286)
- **函数**: `BranchCommands.Merge(branchName string, variant MergeVariant)`

```bash
git merge --no-edit ${用户配置 Merging.Args} ${策略参数} ${分支ref}
```

`--no-edit` 是硬编码参数，防止 git 弹出 commit message 编辑器。

### 2.5 命令执行后的结果检查

Merge 返回后走 [CheckMergeOrRebase](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L168-L170)：

1. 先 `Refresh()` 刷新文件视图
2. 错误分类处理：
   - `"No changes - did you forget to use"` → 自动 `--skip`
   - `"The previous cherry-pick is now empty"` → 自动 `--skip`
   - `"No rebase in progress?"` → 静默忽略
   - 命中 [isMergeConflictErr](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L142-L150) 的 7 种关键字之一 → 进入 `PromptForConflictHandling()`

---

## 三、冲突状态识别：三层校验机制

冲突识别分为 **仓库级状态** → **文件级短状态标记** → **文件内容真实扫描** 三层，层层递进确保准确性。

### 3.1 第一层：仓库级别 WorkingTreeState

定义在 [working_tree_state.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/working_tree_state.go#L9-L14)：

```go
type WorkingTreeState struct {
    Rebasing      bool
    Merging       bool
    CherryPicking bool
    Reverting     bool
}
```

#### 3.1.1 状态检测原理

通过检查 `.git` 目录下的锁文件判断（任何 UI 刷新都会调用）：

- **文件**: [status.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/status.go#L23-L83)
- **函数**: `StatusCommands.WorkingTreeState()`

| 状态 | 检测方式 |
|------|----------|
| `Rebasing` | `.git/rebase-merge/` 或 `.git/rebase-apply/` 目录存在 |
| `Merging` | `.git/MERGE_HEAD` 文件存在 |
| `CherryPicking` | `.git/CHERRY_PICK_HEAD` 文件存在且值不匹配 `rebase-merge/stopped-sha` 前缀 |
| `Reverting` | `.git/REVERT_HEAD` 文件存在 |

**CherryPick 的特殊排除逻辑（重点注意前缀方向）**：
Git 历史上 rebase 用 cherry-pick 实现，rebase 中断时 `CHERRY_PICK_HEAD` 可能残留。代码在 [status.go:L71-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/status.go#L71-L77) 中做如下处理：

```go
cherryPickHeadStr := strings.TrimSpace(string(cherryPickHead))  // 完整 SHA1（40 字符）
stoppedShaStr := strings.TrimSpace(string(stoppedSha))          // 缩写 SHA1（通常 7-12 字符）

// 检查 完整SHA1 是否以 缩写SHA1 为前缀
if strings.HasPrefix(cherryPickHeadStr, stoppedShaStr) {
    return false, nil  // 前缀匹配 → 是 rebase 残余，不是真正的 cherry-pick
}
return true, nil       // 前缀不匹配 → 是真正的 cherry-pick 冲突
```

**前缀比较方向**：`cherryPickHeadStr`（长的完整 SHA）作为被检查字符串，`stoppedShaStr`（短的缩写 SHA）作为前缀。方向不能反——因为缩写 SHA 永远不可能以 40 位完整 SHA 为前缀。

**其他三个状态无同类歧义**：`IsInRebase()`、`IsInMergeState()`、`IsInRevert()` 都是简单的文件/目录存在性检查，没有字符串前缀比较逻辑。只有 CherryPick 因为 Git 历史原因需要做这个特殊排除。

#### 3.1.2 Effective State 优先级

多状态并存（如 rebase 中途 cherry-pick 冲突）时，[Effective()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/working_tree_state.go#L45-L59) 决定当前 UI 展示哪个状态：

**优先级**: `Reverting` > `CherryPicking` > `Merging` > `Rebasing`

这符合 Git 的行为：必须先完成/中止最内层操作，才能继续外层。

### 3.2 第二层：文件级别 shortStatus 解析

`git status --porcelain` 输出的双字符状态被解析为两个布尔字段。

**两个字符的约定（避免歧义）**：
- 第 1 位 = **X** = staged 侧（index / ours / HEAD 分支）
- 第 2 位 = **Y** = unstaged 侧（working tree / theirs / MERGE_HEAD 分支）
- `U` = unmerged（冲突）、`A` = added、`D` = deleted、`M` = modified

- **文件**: [file.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/file.go#L146-L163)
- **函数**: `deriveStatusFields(shortStatus string)`

```go
hasInlineMergeConflicts := lo.Contains([]string{"UU", "AA"}, shortStatus)
hasMergeConflicts := hasInlineMergeConflicts || lo.Contains([]string{"DD", "AU", "UA", "UD", "DU"}, shortStatus)
```

| shortStatus | 含义（XY） | 字段标记 | 常见场景 |
|-------------|-----------|----------|----------|
| `UU` | 双方都修改（U=our, U=their） | `HasInlineMergeConflicts=true` | 同一行都改了，文件内有 `<<<<<<<` 标记 |
| `AA` | 双方都新增（A=our, A=their） | `HasInlineMergeConflicts=true` | 两个分支都新增了同名不同内容的文件 |
| `DD` | 双方都删除（D=our, D=their） | `HasMergeConflicts=true` | rename/rename 冲突的源文件 |
| `AU` | **我们新增、他们删除**（A=our, U=their） | `HasMergeConflicts=true` | rename/rename 冲突的 ours 目标位置 |
| `UA` | **我们删除、他们新增**（U=our, A=their） | `HasMergeConflicts=true` | rename/rename 冲突的 theirs 目标位置 |
| `UD` | **我们修改、他们删除**（U=our, D=their） | `HasMergeConflicts=true` | 我们改了某文件，但 theirs 删了它 |
| `DU` | **我们删除、他们修改**（D=our, U=their） | `HasMergeConflicts=true` | 我们删了某文件，但 theirs 改了它 |

> **注意 AU/UA 方向**：我之前写的"AU=我们删他们加"是**反的**！正确的方向是 A 在第一位（ours 侧），U 在第二位（theirs 侧），所以 AU = ours added + theirs deleted/unmerged。

#### 3.2.1 同类歧义：rename/rename 冲突的三文件组合

`AU`/`UA`/`DD` 这三个状态通常**结伴出现**，是 rename/rename 冲突的完整信号（两个分支都把同一个源文件重命名到不同位置）：

| 状态 | 角色 | 翻译描述 [english.go:L1194-L1198](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/i18n/english.go#L1194-L1198) |
|------|------|---------------------------------------------------------------------|
| `DD` | 源文件 | "this file was moved or renamed both in current and incoming" |
| `AU` | ours 的目标 | "destination of a move or rename in the current changes" |
| `UA` | theirs 的目标 | "destination of a move or rename in the incoming changes" |

在 [handleNonInlineConflict](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L730-L744) 中，代码根据不同状态给用户推荐不同的操作顺序：
- `DD` → 只有删除选项（双方都删了，只能删）
- `DU`/`UD` → 删除在前、保留在后（删除是更常见的选择）
- `AU`/`UA` → 保留在前、删除在后（保留更安全，概率 50/50）

#### 3.2.2 同类歧义：Deleted 字段与冲突状态的重叠

`Deleted` 字段在 [deriveStatusFields:L158](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/file.go#L158) 中定义为：
```go
Deleted := unstagedChange == "D" || stagedChange == "D"
```

这意味着对于 `UD`（我们修改、他们删除），虽然 `HasMergeConflicts=true`，但 `Deleted` 也会是 true（因为第二位是 D）。同理 `DU` 的 `Deleted` 也为 true（因为第一位是 D）。

这个字段本身是正确的，但在读取代码时要注意：**一个文件可以同时是 Deleted 和 HasMergeConflicts**——不要看到 `Deleted` 就认为它已经被删除了，它可能正处于冲突状态等待用户决定是 keep 还是 delete。

### 3.3 第三层：文件内容真实扫描 FileHasConflictMarkers

`git status` 可能滞后（用户在外部编辑器解决了冲突但未 stage）。因此代码会在关键路径上**再次扫描文件真实内容**确认是否还有冲突标记。

#### 3.3.1 扫描函数实现

- **文件**: [find_conflicts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/mergeconflicts/find_conflicts.go#L88-L117)
- **函数**: `FileHasConflictMarkers(path string)`

高效扫描（只查 `<<<<<<< ` 和 `>>>>>>> ` 前缀，不解析完整结构）：
```go
if bytes.HasPrefix(line, CONFLICT_START_BYTES) { return true, nil }
if bytes.HasPrefix(line, CONFLICT_END_BYTES)   { return true, nil }
```

#### 3.3.2 扫描触发点 #1：Stage/Unstage 保护性检查

- **文件**: [file_node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/filetree/file_node.go#L49-L57)
- **函数**: `FileNode.GetHasInlineMergeConflicts()`
- **触发时机**: 用户对包含冲突文件的目录执行 stage/unstage 时

```go
func (self *FileNode) GetHasInlineMergeConflicts() bool {
    return self.SomeFile(func(file *models.File) bool {
        if !file.HasInlineMergeConflicts {
            return false  // 先过 git status 这一层
        }
        hasConflicts, _ := mergeconflicts.FileHasConflictMarkers(file.Path)
        return hasConflicts  // 二次扫描真实文件
    })
}
```

在 [files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L467-L472) 中，如果此函数返回 true，会直接报错拦截：
```go
if node.GetHasInlineMergeConflicts() {
    return errors.New(self.c.Tr.ErrStageDirWithInlineMergeConflicts)
}
```
目的是防止 `>>>>>>>` 这些标记被意外 commit 进仓库。

#### 3.3.3 扫描触发点 #2：自动 stage 已解决文件

- **文件**: [refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L570-L603)
- **函数**: `RefreshHelper.refreshStateFiles()`
- **触发时机**: 每次 FILES 范围刷新（冲突 pick 后、焦点回到窗口、定时刷新等）
- **前置开关**: 用户配置 `Git.AutoStageResolvedConflicts` 必须为 true

```go
if self.c.UserConfig().Git.AutoStageResolvedConflicts {
    pathsToStage := []string{}
    for _, file := range self.c.Model().Files {
        if file.HasMergeConflicts {
            prevConflictFileCount++   // 统计刷新前的冲突文件数（后面要用）
        }
        if file.HasInlineMergeConflicts {
            hasConflicts, err := mergeconflicts.FileHasConflictMarkers(file.Path)
            if err == nil && !hasConflicts {
                // git 仍显示 UU，但文件里已无标记 → 用户在外部解决了
                pathsToStage = append(pathsToStage, file.Path)
            }
        }
    }
    if len(pathsToStage) > 0 {
        self.c.Git().WorkingTree.StageFiles(pathsToStage, nil)  // 自动 stage
    }
}
```

这是用户说"在外部编辑器改完 lazygit 能自动识别"的根本原因。

#### 3.3.4 扫描触发点 #3：主视图渲染（findConflicts 完整解析）

严格来说这不是 `FileHasConflictMarkers`，但同样会扫描文件内容，且触发更频繁：

- **文件**: [files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L273-L282)
- **函数**: `FilesController.GetOnRenderToMain()`
- **触发时机**: 用户在 Files 面板上下移动光标选中带 `HasInlineMergeConflicts` 标记的文件时

```go
if node.File != nil && node.File.HasInlineMergeConflicts {
    hasConflicts, err := self.c.Helpers().MergeConflicts.SetMergeState(node.GetPath())
    // SetMergeState 内部会 cat 文件 → 调用 findConflicts() 完整解析所有冲突块
    if hasConflicts {
        self.c.Helpers().MergeConflicts.Render()  // 直接在主视图渲染彩色冲突
        return
    }
}
```

如果 `findConflicts` 解析出 0 个冲突（`!hasConflicts`），主视图会降级为普通 diff 渲染。这意味着即使 `git status` 还显示 UU，只要文件里没有标记，主视图也不会再以冲突模式渲染。

---

## 四、冲突解决流程与操作入口

### 4.1 冲突出现后的初始菜单

刚执行 merge/rebase 检测到冲突时，调用 [PromptForConflictHandling()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L184-L206)：

```
┌─ Found Conflicts ─────────────────────┐
│  View conflicts (跳 Files 面板)        │
│  Abort merge/rebase/cherry-pick (a)   │
└───────────────────────────────────────┘
```

### 4.2 Files 面板按 Enter：分歧点

在 Files 面板选中冲突文件按 Enter，[EnterFile](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L674-L704) 分三条路径：

```go
if file.HasInlineMergeConflicts {
    return self.switchToMerge()          // 路径A: 逐 hunk pick 编辑器
}
if file.HasMergeConflicts {
    return self.handleNonInlineConflict(file)  // 路径B: 文件级冲突菜单
}
// 路径C: 正常文件 → 进入 Staging 视图
```

#### 4.2.1 路径 A：内联冲突编辑器 switchToMerge

- **文件**: [files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L1098-L1105)

调用 [MergeConflictsHelper.SwitchToMerge](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_conflicts_helper.go#L85-L98)：
1. 读取文件内容 → `findConflicts()` 完整解析所有冲突块
2. 若解析出冲突 → 将 `MergeConflictsContext` 压入上下文栈
3. 渲染彩色冲突视图

#### 4.2.2 路径 B：非内联冲突 handleNonInlineConflict

- **文件**: [files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L706-L751)

针对没有 `<<<<<<<` 标记的文件级冲突（DD/AU/UA/UD/DU），根据 shortStatus 动态生成菜单项：

| shortStatus | 含义（XY 约定） | 菜单（按顺序） |
|-------------|----------------|----------------|
| `DD` | 双方删除（D=our, D=their） | Delete File |
| `DU` | 我们删、他们改（D=our, U=their） | Delete File → Keep File |
| `UD` | 我们改、他们删（U=our, D=their） | Delete File → Keep File |
| `AU` | **我们加、他们删**（A=our, U=their） | Keep File → Delete File |
| `UA` | **我们删、他们加**（U=our, A=their） | Keep File → Delete File |

> **修正说明**：之前这里 AU/UA 的含义也写反了。A 永远在第一位（ours 侧），所以 `AU` = ours 新增 + theirs 删除/冲突。

- **Keep File** → `git add <file>`（stage 当前工作树版本）
- **Delete File** → `git rm <file>`（从工作树和 index 移除）

主视图也会给这些状态渲染提示信息：`DU` 和 `UD` 还会附带 `git diff --base` 的输出，在 [files_controller.go:L299-L305](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/files_controller.go#L299-L305) 中：
- `DU`（我们删、他们改）→ 显示 "Incoming changes:"（因为 theirs 修改了，需要看改了什么再决定是否应用到别处）
- `UD`（我们改、他们删）→ 显示 "Current changes:"（因为 ours 修改了，需要看改了什么再决定是否应用到别处）

帮助用户决定是保留还是删除。

### 4.3 MergeConflicts 控制器：核心逐行解决

- **文件**: [merge_conflicts_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/merge_conflicts_controller.go#L28-L114)

#### 4.3.1 键绑定一览

| 操作 | 默认键 | 功能 |
|------|--------|------|
| `PickHunk` | 空格 / Enter | 采纳当前选中的 hunk（TOP/BOTTOM/MIDDLE） |
| `PickAllHunks` | `b` | 采纳全部（ours + theirs 拼接，去标记） |
| `SelectPrevHunk` | ↑ / `k` | 同一冲突块内切换选择 |
| `SelectNextHunk` | ↓ / `j` | 同上 |
| `PrevConflict` | `<` / `,` | 上一个冲突块 |
| `NextConflict` | `>` / `.` | 下一个冲突块 |
| `Undo` | `z` | 撤销（内容栈机制） |
| `EditFile` | `e` | 外部编辑器打开当前行 |
| `OpenMergeConflictMenu` | `o` | 打开批量策略菜单 |
| `Return` | Esc | 返回 Files 面板 |

#### 4.3.2 冲突块解析与 Selection 模型

- **文件**: [find_conflicts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/mergeconflicts/find_conflicts.go#L25-L58)
- **文件**: [merge_conflict.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/mergeconflicts/merge_conflict.go)

每个 `mergeConflict` 记录 4 个行号：`start`（<<<<<<<）、`ancestor`（\|\|\|\|\|\|\|）、`target`（=======）、`end`（>>>>>>>）。

选择模型：
| Selection | diff3 格式（有 ancestor） | 普通格式（无 ancestor） |
|-----------|--------------------------|------------------------|
| `TOP` | `start ~ ancestor` 之间（ours） | `start ~ target` 之间（ours） |
| `MIDDLE` | `ancestor ~ target` 之间（base） | — |
| `BOTTOM` | `target ~ end` 之间（theirs） | `target ~ end` 之间（theirs） |
| `ALL` | 所有非标记行（ours + base + theirs） | 所有非标记行 |

4 个标记行无论何种选择都会被丢弃。

#### 4.3.3 Undo 内容栈

- **文件**: [state.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/mergeconflicts/state.go#L15-L17)

`State.contents` 是字符串切片作为栈：
```
[原始内容] → [第一次 pick 后] → [第二次 pick 后] → ...
                                                     ↑ 当前
```
- `PushContent()` 每次 resolve 后压入新内容 + 重新解析冲突
- `Undo()` 弹出栈顶，回退到上一版本并重新 `findConflicts`

### 4.4 批量策略菜单（非逐行解决）

- **文件**: [working_tree_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/working_tree_helper.go#L355-L417)
- **入口**: Files 面板按 `o`，或 MergeConflicts 视图按 `o`

| 菜单项 | 快捷键 | 等价命令 |
|--------|--------|----------|
| Use Current Changes (ours) | `c` | `git merge-file --ours` |
| Use Incoming Changes (theirs) | `i` | `git merge-file --theirs` |
| Use Both Changes (union) | `b` | `git merge-file --union` |
| Open Merge Tool | `m` | `git mergetool` |

实现细节：取出 stage 1（base）/stage 2（ours）/stage 3（theirs）三个版本合并。Git ≥ 2.43 用 ObjectID 直接合并，旧版用临时文件中转。

---

## 五、冲突解决后的后续动作：从最后一个 Pick 到 Continue 确认

这是最绕的一段，因为跨越了多个刷新和异步回调。

### 5.1 起点：最后一个冲突在 MergeConflicts 视图内被解决

- **文件**: [merge_conflicts_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/merge_conflicts_controller.go#L254-L266)

```go
func (self *MergeConflictsController) pickSelection(selection mergeconflicts.Selection) error {
    ok, err := self.resolveConflict(selection)
    // ...
    if self.context().GetState().AllConflictsResolved() {
        self.onLastConflictResolved()
    }
    return nil
}

func (self *MergeConflictsController) onLastConflictResolved() {
    // 只刷新 FILES 范围，MODE=ASYNC
    self.c.Refresh(types.RefreshOptions{Mode: types.ASYNC, Scope: []types.RefreshableView{types.FILES}})
}
```

### 5.2 关键枢纽：refreshStateFiles 中的 Continue 触发逻辑

FILES 刷新最终走到 [RefreshHelper.refreshStateFiles](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L570-L619)。这个函数做了 3 件事：

**步骤 1**（如果开了 AutoStageResolvedConflicts）：扫描所有 `HasInlineMergeConflicts` 的文件，用 `FileHasConflictMarkers()` 二次确认，无标记的自动 stage。**同时统计 `prevConflictFileCount`**（刷新 Model 之前存在多少冲突文件）。

**步骤 2**：重新执行 `git status` 加载文件列表到新 `files` 变量，统计 `conflictFileCount`（刷新后 Model 中的冲突文件数）。

**步骤 3**：触发 Continue 的充要条件：
```go
if self.c.Git().Status.WorkingTreeState().Any()   // 仍处于 merge/rebase/... 中
    && conflictFileCount == 0                      // 刷新后所有冲突都消失了
    && prevConflictFileCount > 0 {                 // 刷新前确实有冲突（不是刚启动）
    self.c.OnUIThread(func() error { return self.mergeAndRebaseHelper.PromptToContinueRebase() })
}
```

三个条件缺一个都不会弹：
- 如果用户手动 Abort 了 → `WorkingTreeState().Any()` 为 false，不弹
- 如果还有文件没解决 → `conflictFileCount > 0`，不弹
- 如果启动时就处于已解决状态（中间状态）→ `prevConflictFileCount == 0`，不弹

注意用了 `OnUIThread` 异步调度，避免刷新锁和 UI 锁死锁。

### 5.3 PromptToContinueRebase 确认框

- **文件**: [merge_and_rebase_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L223-L263)

```
┌─ Continue ─────────────────────────────────────┐
│  Conflicts resolved, continue merge?           │
└────────────────────────────────────────────────┘
```

用户点确认后还有一个**二次检查**：
1. 同步刷新一次 FILES（SYNC 模式，确保拿到最新状态）
2. 检查是否存在未暂存的文件（例如用户解决冲突后改了别的地方修编译错误）
3. 有未暂存 → 再弹一次确认问是否自动 stage 这些文件
4. 最后执行 `merge --continue` / `rebase --continue` 等

### 5.4 genericMergeCommand：统一的 continue/abort/skip 执行器

- **文件**: [merge_and_rebase_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L74-L118)

```go
func (self *MergeAndRebaseHelper) genericMergeCommand(command string) error {
    status := self.c.Git().Status.WorkingTreeState()
    commandType := status.CommandName()  // "merge" / "rebase" / "cherry-pick" / "revert"

    needsSubprocess :=
        (effectiveStatus == MERGING && command != ABORT && config.ManualCommit) ||
        (effectiveStatus == REBASING && command != ABORT && hasExecTodos())

    if needsSubprocess {
        return self.c.RunSubprocessAndRefresh(
            self.c.Git().Rebase.GenericMergeOrRebaseActionCmdObj(commandType, command))
    }
    result := self.c.Git().Rebase.GenericMergeOrRebaseAction(commandType, command)
    return self.CheckMergeOrRebase(result)
}
```

决策逻辑：
| 场景 | 是否子进程 | 原因 |
|------|-----------|------|
| merge continue + ManualCommit=true | 是 | 用户需要在编辑器里写 merge commit message |
| rebase continue + 有 exec todo | 是 | exec 可能是耗时编译，用户想看终端输出 |
| 其他所有情况（abort/skip 等） | 否 | 内嵌执行，用 GIT_EDITOR=lazygit 跳过编辑器 |

实际 Git 命令执行在 [RebaseCommands.GenericMergeOrRebaseAction](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/rebase.go#L482-L503)：
```go
func (self *RebaseCommands) GenericMergeOrRebaseAction(commandType string, command string) error {
    // 构造: git <commandType> --<command>
    // 如: git merge --continue / git rebase --abort / git cherry-pick --skip
    return self.runSkipEditorCommand(cmdObj)  // 设置 GIT_EDITOR=lazygit 抑制编辑器弹出
}
```

### 5.5 手动入口：Rebase Options 菜单

除了自动弹出的 Continue 提示，用户随时可通过菜单手动操作：

- **文件**: [merge_and_rebase_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L39-L68)
- **函数**: `CreateRebaseOptionsMenu()`

菜单按当前 Effective State 动态生成：
| 菜单项 | 快捷键 | 条件 |
|--------|--------|------|
| continue | `c` | 始终 |
| abort | `a` | 始终 |
| skip | `s` | Rebasing / CherryPicking / Reverting（merge 没有 skip 语义） |

这个菜单对 rebase/cherry-pick/revert/merge 四种状态通用，标题用 `WorkingTreeState.OptionsMenuTitle()` 动态翻译（"Merge options" / "Rebase options" 等）。

---

## 六、完整调用链时序图（以 pick 完最后一个冲突为例）

```
用户按空格键 pick 最后一个冲突
    │
    ▼
MergeConflictsController.pickSelection()
    │  ├─ resolveConflict() → 写文件，PushContent()，findConflicts() → 0 个冲突
    │  └─ AllConflictsResolved() == true
    │         └─ onLastConflictResolved() → Refresh(Scope: FILES, ASYNC)
    │
    ▼
RefreshHelper.refreshStateFiles()  ← 异步执行
    │
    ├─ [如果 AutoStageResolvedConflicts]
    │     遍历 Files Model → FileHasConflictMarkers() 二次扫描
    │     收集没有标记的文件 → 自动 git add
    │     统计 prevConflictFileCount  ← 关键
    │
    ├─ 重新 git status → 新 files Model
    │     统计 conflictFileCount  ← 关键
    │
    └─ WorkingTreeState.Any() && conflictFileCount==0 && prevConflictFileCount>0
              │
              ▼
         OnUIThread 调度
              │
              ▼
    MergeAndRebaseHelper.PromptToContinueRebase()
              │  弹确认框
              ▼
    用户确认 → 二次 SYNC 刷新检查未暂存 →（可能）自动 stage
              │
              ▼
    genericMergeCommand("continue")
              │
              ├─ 决定是否子进程（ManualCommit? exec todos?）
              │
              ▼
    RebaseCommands.GenericMergeOrRebaseAction("merge"/"rebase"/..., "continue")
              │
              ▼
         git merge --continue （或 rebase/cherry-pick/revert）
              │
              ▼
    CheckMergeOrRebase() → 可能又发现新冲突？→ 整个流程再走一遍
```

---

## 七、关键数据结构关系图

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Controller 层                                 │
│                                                                       │
│  ┌─────────────────────┐  ┌────────────────────────────────────┐     │
│  │ BranchesController  │  │ RemoteBranchesController           │     │
│  │  - merge() [本地]   │  │  - merge() [远端分支 FullName]      │     │
│  └──────────┬──────────┘  └───────────────┬────────────────────┘     │
│             │                             │                          │
│             └──────────────┬──────────────┘                          │
│                            ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │              MergeAndRebaseHelper                           │     │
│  │  - MergeRefIntoCheckedOutBranch()  [弹出合并策略菜单]         │     │
│  │  - CheckMergeOrRebase()             [检查冲突关键字]         │     │
│  │  - genericMergeCommand()            [continue/abort/skip]    │     │
│  │  - PromptForConflictHandling()      [冲突初始菜单]           │     │
│  │  - PromptToContinueRebase()         [冲突解决后确认框]       │     │
│  │  - CreateRebaseOptionsMenu()        [手动操作入口]           │     │
│  └──────────────┬──────────────────────────────┬────────────────┘     │
│                 │                              │                      │
│                 ▼                              ▼                      │
│  ┌──────────────────────────────┐   ┌─────────────────────────┐      │
│  │    MergeConflictsHelper      │   │   WorkingTreeHelper     │      │
│  │  - SwitchToMerge()           │   │  - CreateMergeConflict- │      │
│  │  - SetMergeState()           │   │    Menu() [批量策略]    │      │
│  │  - RefreshMergeState()       │   │  - mergeFile()          │      │
│  │  - EscapeMerge()             │   │  - OpenMergeTool()      │      │
│  └──────────────┬───────────────┘   └────────────┬────────────┘      │
└─────────────────┼────────────────────────────────┼───────────────────┘
                  │                                │
┌─────────────────┼────────────────────────────────┼───────────────────┐
│                 ▼                                ▼                   │
│  ┌──────────────────────────────┐   ┌─────────────────────────┐      │
│  │   MergeConflictsContext      │   │      Git Commands       │      │
│  │  ┌────────────────────────┐  │   │  - Branch.Merge()      │      │
│  │  │ ConflictsViewModel     │  │   │  - Rebase.Generic-     │      │
│  │  │  - state (*State)      │  │   │    MergeOrRebaseAction │      │
│  │  │  - userVerticalScroll  │  │   │  - Status.-            │      │
│  │  └───────────┬────────────┘  │   │    WorkingTreeState()  │      │
│  │              │               │   └──────────┬──────────────┘      │
│  └──────────────┼───────────────┘              │                     │
│                 ▼                              ▼                     │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                    mergeconflicts 包                       │       │
│  │  ┌─────────────┐  ┌───────────────┐  ┌──────────────────┐  │       │
│  │  │ State       │  │ mergeConflict │  │ findConflicts()  │  │       │
│  │  │ - contents[]│  │  - start      │  │  - 逐行扫描 4 种 │  │       │
│  │  │ - conflicts│  │  - ancestor   │  │    标记          │  │       │
│  │  │ - path     │  │  - target     │  ├──────────────────┤  │       │
│  │  │ - index    │  │  - end        │  │FileHasConflict-  │  │       │
│  │  └─────────────┘  └───────────────┘  │  Markers()       │  │       │
│  │                                        │  - 只查首尾标记 │  │       │
│  │                                        └──────────────────┘  │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                    models 包                               │       │
│  │  ┌──────────────────────┐   ┌────────────────────────┐   │       │
│  │  │ WorkingTreeState     │   │ File (status 字段)     │   │       │
│  │  │  Rebasing/Merging/   │   │  HasInlineMerge-       │   │       │
│  │  │  CherryPicking/      │   │   Conflicts            │   │       │
│  │  │  Reverting           │   │  HasMergeConflicts     │   │       │
│  │  │  - Effective() 优先级│   │  ShortStatus UU/DD/... │   │       │
│  │  └──────────────────────┘   └────────────────────────┘   │       │
│  └───────────────────────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 八、歧义问题汇总（必读）

这是本次深度分析发现的所有容易理解错的地方，按"陷阱等级"排序：

### 8.1 🟥 高危陷阱：CherryPick 前缀比较方向

**代码位置**: [status.go:L71-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/git_commands/status.go#L71-L77)

```go
cherryPickHeadStr := strings.TrimSpace(string(cherryPickHead))  // 完整 SHA1 (40 chars)
stoppedShaStr := strings.TrimSpace(string(stoppedSha))          // 缩写 SHA1 (7-12 chars)

if strings.HasPrefix(cherryPickHeadStr, stoppedShaStr) {  // ✅ 正确方向
    return false, nil  // 是 rebase 残余，不是真正的 cherry-pick
}
```

**易错点**: 很容易写成 `strings.HasPrefix(stoppedShaStr, cherryPickHeadStr)`（前缀方向反了）。但因为完整 SHA 比缩写长，反了永远返回 false，会把 rebase 残余误判为真正的 cherry-pick，导致 UI 显示错误。

**为什么只有 CherryPick 需要？**：Git 历史上 rebase 用 cherry-pick 实现，rebase 中断时 `CHERRY_PICK_HEAD` 可能残留。其他三个状态（Rebasing/Merging/Reverting）都是简单的文件存在性检查，没有同类歧义。

---

### 8.2 🟧 中危陷阱：shortStatus 的 XY 约定

**代码位置**: [file.go:L146-L163](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/file.go#L146-L163)

**硬约定**:
- 第 1 位 **X** = staged 侧 = ours = HEAD 分支
- 第 2 位 **Y** = unstaged 侧 = theirs = MERGE_HEAD 分支

**已修正的错误**（原文档写反了）：

| shortStatus | 原错误理解 | ✅ 正确理解 |
|-------------|-----------|-----------|
| `AU` | 我们删、他们加 | **我们加**（A=our）、**他们删**（U=their） |
| `UA` | 他们删、我们加 | **我们删**（U=our）、**他们加**（A=their） |

**记忆法**: A/U/D 永远修饰它**所在位**代表的那一方。A 在第一位就是 ours added，A 在第二位就是 theirs added。

---

### 8.3 🟧 中危陷阱：AU/UA/DD 是 rename 冲突的三胞胎

这三个状态通常**结伴出现**，不是孤立事件：

| 状态 | 角色 | 含义 |
|------|------|------|
| `DD` | 源文件 | 双方都把 old_name 重命名/删除了 |
| `AU` | ours 的目标 | ours 把 old_name 重命名为 new_name_ours |
| `UA` | theirs 的目标 | theirs 把 old_name 重命名为 new_name_theirs |

如果你只看到 `AU` 而没看到 `DD` 和 `UA`，那不是 rename 冲突——很可能是普通的"我们新增了一个文件，但 theirs 刚好删除了同名旧文件"。

---

### 8.4 🟨 低危陷阱：Deleted 字段与冲突状态可重叠

**代码位置**: [file.go:L158](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/file.go#L158)

```go
Deleted := unstagedChange == "D" || stagedChange == "D"
```

**易错点**: 看到 `Deleted=true` 就以为文件已经不在工作树了。实际上：

| shortStatus | HasMergeConflicts | Deleted | 实际状态 |
|-------------|-------------------|---------|----------|
| `UD` | ✅ true | ✅ true | 文件还在，冲突中，等待用户决定 keep 还是 delete |
| `DU` | ✅ true | ✅ true | 文件还在，冲突中，等待用户决定 keep 还是 delete |
| `D ` | ❌ false | ✅ true | 真的被删除了，已 staged |

不要用 `Deleted` 字段判断文件是否真的被删除了，必须同时检查 `HasMergeConflicts`。

---

### 8.5 🟨 低危陷阱：WorkingTreeState 多状态并存的优先级

**代码位置**: [working_tree_state.go:L45-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/37-lazygit/pkg/commands/models/working_tree_state.go#L45-L59)

`Effective()` 方法的优先级是硬编码的 if 顺序：

```go
if self.Reverting       { return REVERTING }       // 最高
if self.CherryPicking   { return CHERRY_PICKING }  // 次高
if self.Merging         { return MERGING }         // 次低
if self.Rebasing        { return REBASING }        // 最低
```

**注释明确说明**：实际可能出现的多状态组合是 `Rebasing+CherryPicking` 和 `Rebasing+Reverting`。`Rebasing+Merging` 理论上可能但实际几乎不会发生。

**为什么是这个顺序？** 因为 Git 是栈式操作——你在 rebase 中途 cherry-pick 产生冲突，必须先解决 cherry-pick（内层）才能继续 rebase（外层）。优先级顺序恰好是内层操作优先。

---

### 8.6 🟨 低危陷阱：HasInlineMergeConflicts 的两种场景

这个字段为 true 时，文件里**不一定**真的有 `<<<<<<<` 标记：

| 场景 | HasInlineMergeConflicts | 真实冲突标记 |
|------|-------------------------|-------------|
| 刚 merge 失败 | ✅ true | ✅ 有 |
| 用户在外部编辑器手动解决但未 stage | ✅ true | ❌ 无 |

这就是为什么代码在三处关键路径（stage/unstage 拦截、自动 stage、主视图渲染）都用 `FileHasConflictMarkers()` 做二次扫描。永远不要只信 `git status` 的输出。

---

### 8.7 歧义检查结果总表

| 模块 | 检查项 | 有无歧义 | 是否已修正 |
|------|--------|----------|------------|
| WorkingTreeState | IsInCherryPick 前缀比较 | ✅ 有（方向易错） | ✅ 已详细说明 |
| WorkingTreeState | IsInRebase/Merge/Revert | ❌ 无 | — |
| WorkingTreeState | Effective() 优先级 | ✅ 有（if 顺序隐含优先级） | ✅ 已说明 |
| File 状态 | shortStatus XY 约定 | ✅ 有（AU/UA 易写反） | ✅ 已修正并说明 |
| File 状态 | Deleted 字段含义 | ✅ 有（可与冲突重叠） | ✅ 已说明 |
| File 状态 | AU/UA/DD 组合语义 | ✅ 有（rename 冲突三胞胎） | ✅ 已说明 |
| File 状态 | HasInlineMergeConflicts | ✅ 有（git status 可能滞后） | ✅ 已说明 |
| mergeconflicts 包 | findConflicts 解析 | ❌ 无 | — |
