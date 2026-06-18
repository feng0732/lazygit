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

- **文件**: [pkg/gui/controllers/branches_controller.go](pkg/gui/controllers/branches_controller.go#L134-L139)
- **快捷键**: `Config.Branches.MergeIntoCurrentBranch`
- **处理函数**: [pkg/gui/controllers/branches_controller.go](pkg/gui/controllers/branches_controller.go#L682-L685)

```go
func (self *BranchesController) merge() error {
    selectedBranchName := self.context().GetSelected().Name
    // 传入选中分支名（短名，如 "feature"）
    return self.c.Helpers().MergeAndRebase.MergeRefIntoCheckedOutBranch(selectedBranchName)
}
```

### 2.2 入口 B：远端分支面板 (Remote Branches)

- **文件**: [pkg/gui/controllers/remote_branches_controller.go](pkg/gui/controllers/remote_branches_controller.go#L52-L58)
- **快捷键**: 同样是 `Config.Branches.MergeIntoCurrentBranch`
- **处理函数**: [pkg/gui/controllers/remote_branches_controller.go](pkg/gui/controllers/remote_branches_controller.go#L139-L141)

```go
func (self *RemoteBranchesController) merge(selectedBranch *models.RemoteBranch) error {
    // 传入远端分支全名（如 "origin/feature"）
    return self.c.Helpers().MergeAndRebase.MergeRefIntoCheckedOutBranch(selectedBranch.FullName())
}
```

**关键差异**: 本地分支传短名 `feature`，远端分支传全名 `origin/feature`。底层 `git merge` 都能识别这两种 ref。

### 2.3 合并策略菜单（共享）

两个入口都调用 [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L375-L489)，弹出 4 种合并方式：

| 菜单项 | 快捷键 | Git 参数 |
|--------|--------|----------|
| Regular Merge (Fast-Forward) | `m` / `f` | `--ff` |
| Regular Merge (Non-Fast-Forward) | `n` / `m` | `--no-ff` |
| Squash Merge (Uncommitted) | `s` | `--squash --ff` |
| Squash Merge (Committed) | `S` | `--squash --ff` + 自动 commit |

菜单项顺序由用户配置 `Git.Merging.Args` 或 git 配置 `merge.ff` 决定（如果用户偏好 ff，则 ff 项排第一）。

### 2.4 实际执行 Git Merge 命令

最终调用 Git 命令层：

- **文件**: [pkg/commands/git_commands/branch.go](pkg/commands/git_commands/branch.go#L262-L286)
- **函数**: `BranchCommands.Merge(branchName string, variant MergeVariant)`

```bash
git merge --no-edit ${用户配置 Merging.Args} ${策略参数} ${分支ref}
```

`--no-edit` 是硬编码参数，防止 git 弹出 commit message 编辑器。

### 2.5 命令执行后的结果检查

Merge 返回后走 [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L168-L170)：

1. 先 `Refresh()` 刷新文件视图
2. 错误分类处理：
   - `"No changes - did you forget to use"` → 自动 `--skip`
   - `"The previous cherry-pick is now empty"` → 自动 `--skip`
   - `"No rebase in progress?"` → 静默忽略
   - 命中 [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L142-L150) 的 7 种关键字之一 → 进入 `PromptForConflictHandling()`

---

## 三、冲突状态识别：三层校验机制

冲突识别分为 **仓库级状态** → **文件级短状态标记** → **文件内容真实扫描** 三层，层层递进确保准确性。

### 3.1 第一层：仓库级别 WorkingTreeState

定义在 [pkg/commands/models/working_tree_state.go](pkg/commands/models/working_tree_state.go#L9-L14)：

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

- **文件**: [pkg/commands/git_commands/status.go](pkg/commands/git_commands/status.go#L23-L83)
- **函数**: `StatusCommands.WorkingTreeState()`

| 状态 | 检测方式 |
|------|----------|
| `Rebasing` | `.git/rebase-merge/` 或 `.git/rebase-apply/` 目录存在 |
| `Merging` | `.git/MERGE_HEAD` 文件存在 |
| `CherryPicking` | `.git/CHERRY_PICK_HEAD` 文件存在，且 "不是 rebase-merge 中断时的 cherry-pick 残余" |
| `Reverting` | `.git/REVERT_HEAD` 文件存在 |

**CherryPick 的特殊排除逻辑（完整判定流程 + 判定标准）**：

#### 3.1.1.1 判定标准（3 条）

判断是否处于真正的 cherry-pick 冲突状态，**必须同时满足**：
1. ✅ `.git/CHERRY_PICK_HEAD` 文件存在（Git 正在 cherry-pick 过程中）
2. ❌ **要么** `.git/rebase-merge/stopped-sha` 文件不存在（没有进行交互式 rebase）
3. ❌ **要么** `CHERRY_PICK_HEAD` 的完整 SHA **不以前面** `stopped-sha` 的缩写 SHA 为前缀（正在 rebase 但当前 cherry-pick 冲突不是 rebase 引起的）

如果 `stopped-sha` 存在且完整 SHA **以前面** 缩写 SHA 为前缀 → 判定为 **rebase 残余**，不算真正的 cherry-pick。

#### 3.1.1.2 4 步判定流程图

代码在 [pkg/commands/git_commands/status.go](pkg/commands/git_commands/status.go#L49-L79) 中的完整实现：

```
Step 1: 检查 CHERRY_PICK_HEAD 是否存在
        ├─ 不存在 → return (false, nil)  ❌ 不是 cherry-pick
        └─ 存在 → 继续

Step 2: 读取 CHERRY_PICK_HEAD 内容（完整 SHA1，40 字符）
        ├─ 读取报错 → return (false, err)  ❌ IO 错误，透传错误
        └─ 成功 → 继续

Step 3: 读取 rebase-merge/stopped-sha（缩写 SHA1，7-12 字符）
        ├─ 读取报错 → ⚠️  return (true, nil)  ✅ 判定为真正的 cherry-pick
        │                 （代码注释："If we get an error we assume the file doesn't exist"）
        └─ 成功 → 继续

Step 4: 前缀比较
        if strings.HasPrefix(完整SHA, 缩写SHA) {
            return (false, nil)   ❌ 前缀匹配 → 是 rebase 残余
        }
        return (true, nil)        ✅ 前缀不匹配 → 是真正的 cherry-pick
```

#### 3.1.1.3 🚨 stopped-sha 缺失分支的误读风险（重中之重）

**Step 3 是最容易读错的地方**，请务必仔细看：

```go
stoppedSha, err := os.ReadFile(...)
if err != nil {
    // If we get an error we assume the file doesn't exist
    return true, nil   // ⚠️ 读取失败 → 返回 true → 判定为 ✅ 真正的 cherry-pick
}
```

| 现象 | 代码注释说 | 实际返回值 | 判定结果 |
|------|-----------|-----------|----------|
| `stopped-sha` 读取失败（文件不存在/权限不足/IO错误） | "assume the file doesn't exist" | `return true` | ✅ 是真正的 cherry-pick |

**最容易误读的点**：注释说"假设文件不存在"，而文件不存在意味着没有 rebase 正在进行。**没有 rebase 就意味着这个 cherry-pick 是真的**，所以返回 `true`（是 cherry-pick），而不是 `false`（不是 cherry-pick）。

很多人第一眼会读反："既然 stopped-sha 不存在，那 return false 才对"——这是错的。正确的逻辑链是：
> `stopped-sha` 不存在 → 没有进行 rebase → 这个 `CHERRY_PICK_HEAD` 不可能是 rebase 的残留 → 那必然是真正的 cherry-pick → 返回 `true`

这个分支覆盖了绝大多数正常场景：
- 普通 cherry-pick 冲突（没有任何 rebase 目录）→ `stopped-sha` 不存在 → 正确判定 ✅
- 交互式 rebase 中途产生的 cherry-pick 冲突 → `stopped-sha` 存在且前缀匹配 → 判定为 rebase 残余 ❌
- rebase 状态下 `stopped-sha` 文件损坏/丢失 → 误判为 cherry-pick（极罕见边缘情况）

#### 3.1.1.4 其他关键细节

**细节 #1 —— 只检查 rebase-merge，不检查 rebase-apply**：
代码只尝试读取 `.git/rebase-merge/stopped-sha`，而不检查 `.git/rebase-apply/stopped-sha`。这是因为：
- `rebase-merge` 目录对应交互式 rebase / merge-based rebase（Git 现代默认方式）
- `rebase-apply` 目录对应老式 patch-based rebase（`git rebase --apply`），通常不会产生 `CHERRY_PICK_HEAD` 残留

**细节 #2 —— 前缀比较方向不能反**：
```go
if strings.HasPrefix(cherryPickHeadStr, stoppedShaStr) {
```
`cherryPickHeadStr`（完整 40 位 SHA）作为被检查字符串，`stoppedShaStr`（缩写 7-12 位 SHA）作为前缀。方向不能反——因为缩写 SHA 永远不可能以 40 位完整 SHA 为前缀，反了会永远返回 false，把所有 rebase 残余都误判为真正的 cherry-pick。

**其他三个状态无同类歧义**：`IsInRebase()`、`IsInMergeState()`、`IsInRevert()` 都是简单的文件/目录存在性检查，没有字符串前缀比较逻辑和分支默认行为。只有 CherryPick 因为 Git 历史原因需要做这个特殊排除。

#### 3.1.2 Effective State 优先级

多状态并存（如 rebase 中途 cherry-pick 冲突）时，[pkg/commands/models/working_tree_state.go](pkg/commands/models/working_tree_state.go#L45-L59) 决定当前 UI 展示哪个状态：

**优先级**: `Reverting` > `CherryPicking` > `Merging` > `Rebasing`

这符合 Git 的行为：必须先完成/中止最内层操作，才能继续外层。

### 3.2 第二层：文件级别 shortStatus 解析

`git status --porcelain` 输出的双字符状态被解析为两个布尔字段。

#### 3.2.0 字符语义精确定义（对照 git status 官方文档）

**两个字符的位置约定（XY）**：
- 第 1 位 **X** = staged 侧 = ours = HEAD 分支（当前所在分支）
- 第 2 位 **Y** = unstaged 侧 = theirs = MERGE_HEAD / REBASE_HEAD 分支（被合并的分支）

**unmerged 状态下的字符精确含义（来自 git status 官方文档）**：

| 字符 | 在 unmerged 状态中的语义 |
|------|------------------------|
| `U` | **unmerged**（冲突，未合并）——不是 "updated" 也不是 "modified" |
| `A` | added（新增/添加） |
| `D` | deleted（删除） |

**Git 官方 7 种 unmerged 状态的完整对照**：

| shortStatus | Git 官方描述 | 精确含义 |
|-------------|-------------|----------|
| `DD` | unmerged, both deleted | 双方都删除了 |
| `AU` | unmerged, added by us | **我们新增**，冲突（theirs 侧状态为 unmerged） |
| `UD` | unmerged, deleted by them | **他们删除**，冲突（ours 侧状态为 unmerged） |
| `UA` | unmerged, added by them | **他们新增**，冲突（ours 侧状态为 unmerged） |
| `DU` | unmerged, deleted by us | **我们删除**，冲突（theirs 侧状态为 unmerged） |
| `AA` | unmerged, both added | 双方都新增了 |
| `UU` | unmerged, both modified | 双方都修改了 |

> **U 的语义修正（重要！）**：我之前把 AU 写成 "我们新增、他们删除"，UA 写成 "我们删除、他们新增"——这是**不精确的**。U 在 unmerged 状态里的标准语义是 **"conflict / unmerged"**，不是 "deleted"。AU 中 Y 位的 U 只表示 theirs 侧处于未合并冲突状态，具体是删除/新增/修改要看另一位的 A/D 以及实际场景。

- **文件**: [pkg/commands/models/file.go](pkg/commands/models/file.go#L146-L163)
- **函数**: `deriveStatusFields(shortStatus string)`

```go
hasInlineMergeConflicts := lo.Contains([]string{"UU", "AA"}, shortStatus)
hasMergeConflicts := hasInlineMergeConflicts || lo.Contains([]string{"DD", "AU", "UA", "UD", "DU"}, shortStatus)
```

| shortStatus | Git 官方含义（XY） | 字段标记 | 常见真实场景 |
|-------------|-------------------|----------|-------------|
| `UU` | unmerged, both modified | `HasInlineMergeConflicts=true` | 同一行都改了，文件内有 `<<<<<<<` 标记 |
| `AA` | unmerged, both added | `HasInlineMergeConflicts=true` | 两个分支都新增了同名不同内容的文件，文件内有标记 |
| `DD` | unmerged, both deleted | `HasMergeConflicts=true` | rename/rename 冲突的**源文件** |
| `AU` | unmerged, added by us | `HasMergeConflicts=true` | rename/rename 冲突的**ours 目标位置**（我们把源文件重命名到了这里） |
| `UA` | unmerged, added by them | `HasMergeConflicts=true` | rename/rename 冲突的**theirs 目标位置**（他们把源文件重命名到了这里） |
| `UD` | unmerged, deleted by them | `HasMergeConflicts=true` | 我们修改了某文件，他们删除了它 |
| `DU` | unmerged, deleted by us | `HasMergeConflicts=true` | 我们删除了某文件，他们修改了它 |

#### 3.2.1 同类歧义：rename/rename 冲突的三文件组合

`AU`/`UA`/`DD` 这三个状态通常**结伴出现**，是 rename/rename 冲突的完整信号（两个分支都把同一个源文件重命名到不同位置）：

```
源文件 old.go (DD, 双方都标记为删除)
├─ ours 分支: old.go → ours_new.go  →  AU (added by us, conflict)
└─ theirs 分支: old.go → theirs_new.go → UA (added by them, conflict)
```

| 状态 | 角色 | lazygit 翻译描述 [pkg/i18n/english.go](pkg/i18n/english.go#L1194-L1198) |
|------|------|---------------------------------------------------------------------|
| `DD` | 源文件 | "this file was moved or renamed both in current and incoming" |
| `AU` | ours 的目标 | "destination of a move or rename in the current changes" |
| `UA` | theirs 的目标 | "destination of a move or rename in the incoming changes" |

在 [pkg/gui/controllers/files_controller.go](pkg/gui/controllers/files_controller.go#L730-L744) 中，代码根据不同状态给用户推荐不同的操作顺序：
- `DD` → 只有删除选项（双方都删了源文件，只能删）
- `DU`/`UD` → 删除在前、保留在后（删除是更常见的选择）
- `AU`/`UA` → 保留在前、删除在后（保留更安全，概率 50/50）

**注意 AU/UA 不是 rename 冲突的专属状态**：你也可能在非 rename 场景看到单独的 AU 或 UA（例如我们新增了一个新文件，但 theirs 里恰好有一个同名旧文件处于冲突状态）。只有当 `AU` + `UA` + `DD` 三个同时出现时，才是典型的 rename/rename 冲突。

#### 3.2.2 同类歧义：Deleted 字段与冲突状态的重叠

`Deleted` 字段在 [pkg/commands/models/file.go](pkg/commands/models/file.go#L158) 中定义为：
```go
Deleted := unstagedChange == "D" || stagedChange == "D"
```

这意味着对于 `UD`（我们修改、他们删除），虽然 `HasMergeConflicts=true`，但 `Deleted` 也会是 true（因为第二位是 D）。同理 `DU` 的 `Deleted` 也为 true（因为第一位是 D）。

这个字段本身是正确的，但在读取代码时要注意：**一个文件可以同时是 Deleted 和 HasMergeConflicts**——不要看到 `Deleted` 就认为它已经被删除了，它可能正处于冲突状态等待用户决定是 keep 还是 delete。

### 3.3 第三层：文件内容真实扫描 FileHasConflictMarkers

`git status` 可能滞后（用户在外部编辑器解决了冲突但未 stage）。因此代码会在关键路径上**再次扫描文件真实内容**确认是否还有冲突标记。

#### 3.3.1 扫描函数实现

- **文件**: [pkg/gui/mergeconflicts/find_conflicts.go](pkg/gui/mergeconflicts/find_conflicts.go#L88-L117)
- **函数**: `FileHasConflictMarkers(path string)`

高效扫描（只查 `<<<<<<< ` 和 `>>>>>>> ` 前缀，不解析完整结构）：
```go
if bytes.HasPrefix(line, CONFLICT_START_BYTES) { return true, nil }
if bytes.HasPrefix(line, CONFLICT_END_BYTES)   { return true, nil }
```

#### 3.3.2 扫描触发点 #1：Stage/Unstage 保护性检查

- **文件**: [pkg/gui/filetree/file_node.go](pkg/gui/filetree/file_node.go#L49-L57)
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

在 [pkg/gui/controllers/files_controller.go](pkg/gui/controllers/files_controller.go#L467-L472) 中，如果此函数返回 true，会直接报错拦截：
```go
if node.GetHasInlineMergeConflicts() {
    return errors.New(self.c.Tr.ErrStageDirWithInlineMergeConflicts)
}
```
目的是防止 `>>>>>>>` 这些标记被意外 commit 进仓库。

#### 3.3.3 扫描触发点 #2：自动 stage 已解决文件

- **文件**: [pkg/gui/controllers/helpers/refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L570-L603)
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

- **文件**: [pkg/gui/controllers/files_controller.go](pkg/gui/controllers/files_controller.go#L273-L282)
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

刚执行 merge/rebase 检测到冲突时，调用 [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L184-L206)：

```
┌─ Found Conflicts ─────────────────────┐
│  View conflicts (跳 Files 面板)        │
│  Abort merge/rebase/cherry-pick (a)   │
└───────────────────────────────────────┘
```

### 4.2 Files 面板按 Enter：分歧点

在 Files 面板选中冲突文件按 Enter，[pkg/gui/controllers/files_controller.go](pkg/gui/controllers/files_controller.go#L674-L704) 分三条路径：

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

- **文件**: [pkg/gui/controllers/files_controller.go](pkg/gui/controllers/files_controller.go#L1098-L1105)

调用 [pkg/gui/controllers/helpers/merge_conflicts_helper.go](pkg/gui/controllers/helpers/merge_conflicts_helper.go#L85-L98)：
1. 读取文件内容 → `findConflicts()` 完整解析所有冲突块
2. 若解析出冲突 → 将 `MergeConflictsContext` 压入上下文栈
3. 渲染彩色冲突视图

#### 4.2.2 路径 B：非内联冲突 handleNonInlineConflict

- **文件**: [pkg/gui/controllers/files_controller.go](pkg/gui/controllers/files_controller.go#L706-L751)

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

主视图也会给这些状态渲染提示信息：`DU` 和 `UD` 还会附带 `git diff --base` 的输出，在 [pkg/gui/controllers/files_controller.go](pkg/gui/controllers/files_controller.go#L299-L305) 中：
- `DU`（我们删、他们改）→ 显示 "Incoming changes:"（因为 theirs 修改了，需要看改了什么再决定是否应用到别处）
- `UD`（我们改、他们删）→ 显示 "Current changes:"（因为 ours 修改了，需要看改了什么再决定是否应用到别处）

帮助用户决定是保留还是删除。

### 4.3 MergeConflicts 控制器：核心逐行解决

- **文件**: [pkg/gui/controllers/merge_conflicts_controller.go](pkg/gui/controllers/merge_conflicts_controller.go#L28-L114)

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

- **文件**: [pkg/gui/mergeconflicts/find_conflicts.go](pkg/gui/mergeconflicts/find_conflicts.go#L25-L58)
- **文件**: [pkg/gui/mergeconflicts/merge_conflict.go](pkg/gui/mergeconflicts/merge_conflict.go)

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

- **文件**: [pkg/gui/mergeconflicts/state.go](pkg/gui/mergeconflicts/state.go#L15-L17)

`State.contents` 是字符串切片作为栈：
```
[原始内容] → [第一次 pick 后] → [第二次 pick 后] → ...
                                                     ↑ 当前
```
- `PushContent()` 每次 resolve 后压入新内容 + 重新解析冲突
- `Undo()` 弹出栈顶，回退到上一版本并重新 `findConflicts`

### 4.4 批量策略菜单（非逐行解决）

- **文件**: [pkg/gui/controllers/helpers/working_tree_helper.go](pkg/gui/controllers/helpers/working_tree_helper.go#L355-L417)
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

- **文件**: [pkg/gui/controllers/merge_conflicts_controller.go](pkg/gui/controllers/merge_conflicts_controller.go#L254-L266)

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

FILES 刷新最终走到 [pkg/gui/controllers/helpers/refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go#L570-L619)。这个函数做了 3 件事：

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

- **文件**: [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L223-L263)

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

- **文件**: [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L74-L118)

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

实际 Git 命令执行在 [pkg/commands/git_commands/rebase.go](pkg/commands/git_commands/rebase.go#L482-L503)：
```go
func (self *RebaseCommands) GenericMergeOrRebaseAction(commandType string, command string) error {
    // 构造: git <commandType> --<command>
    // 如: git merge --continue / git rebase --abort / git cherry-pick --skip
    return self.runSkipEditorCommand(cmdObj)  // 设置 GIT_EDITOR=lazygit 抑制编辑器弹出
}
```

### 5.5 手动入口：Rebase Options 菜单

除了自动弹出的 Continue 提示，用户随时可通过菜单手动操作：

- **文件**: [pkg/gui/controllers/helpers/merge_and_rebase_helper.go](pkg/gui/controllers/helpers/merge_and_rebase_helper.go#L39-L68)
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

### 8.1 🟥 高危陷阱：CherryPick stopped-sha 读取失败的默认分支

**代码位置**: [pkg/commands/git_commands/status.go](pkg/commands/git_commands/status.go#L49-L79)

```go
stoppedSha, err := os.ReadFile(filepath.Join(self.repoPaths.WorktreeGitDirPath(), "rebase-merge", "stopped-sha"))
if err != nil {
    // If we get an error we assume the file doesn't exist
    return true, nil   // ⚠️ 读取失败 → 返回 true → 判定为 ✅ 真正的 cherry-pick
}
```

**🚨 最容易误读的地方（Step 3 分支）**：

代码注释写的是 *"If we get an error we assume the file doesn't exist"*，很多人第一眼会以为"文件不存在 = 没有 cherry-pick = 返回 false"——**这是完全读反了**！

正确的逻辑链（必须顺着推导 3 步）：
1. `stopped-sha` 读取失败 → 假设文件不存在
2. 文件不存在 → 没有进行交互式 rebase
3. 没有 rebase → 这个 `CHERRY_PICK_HEAD` 不可能是 rebase 残留 → **必然是真正的 cherry-pick** → 返回 `true`

| 现象 | 代码注释 | 实际返回值 | 判定结果 |
|------|---------|-----------|----------|
| `stopped-sha` 不存在（普通 cherry-pick） | assume file doesn't exist | `return true` | ✅ 是真正的 cherry-pick |
| `stopped-sha` 存在且前缀匹配（rebase 中途） | — | `return false` | ❌ 是 rebase 残余 |
| `stopped-sha` 存在但前缀不匹配（罕见） | — | `return true` | ✅ 是真正的 cherry-pick |

**其他易错点**：
1. 只检查 `rebase-merge/stopped-sha`，不检查 `rebase-apply/stopped-sha`。即只考虑现代 merge-based rebase，不考虑老式 patch-based rebase（`git rebase --apply`）。
2. 前缀比较方向不能反：`HasPrefix(完整SHA, 缩写SHA)`，反了永远返回 false，会把 rebase 残余全部误判为真正的 cherry-pick。

---

### 8.2 🟧 中危陷阱：U 在 unmerged 状态下的语义是 "conflict" 不是 "deleted"

**代码位置**: [pkg/commands/models/file.go](pkg/commands/models/file.go#L146-L163)

**Git 官方 unmerged 语义对照**：

| shortStatus | Git 官方描述 | 精确语义 |
|-------------|-------------|----------|
| `AU` | unmerged, added by us | 我们新增了，冲突（U≠deleted） |
| `UA` | unmerged, added by them | 他们新增了，冲突（U≠deleted） |
| `UD` | unmerged, deleted by them | 他们删除了，冲突 |
| `DU` | unmerged, deleted by us | 我们删除了，冲突 |

**已修正的错误**：之前把 AU 写成"我们新增、他们删除"，UA 写成"我们删除、他们新增"。

> **U 的精确含义**：在 unmerged 状态下，U = **conflict / unmerged**，不是 deleted。AU 中 Y 位的 U 只表示 theirs 侧处于冲突未合并状态，具体做了什么要看另一位的 A/D 以及实际上下文。真正表示"删除"的是字符 D。

**记忆法**: 字符只修饰它**所在位**代表的那一方。A 在第一位 = ours added；A 在第二位 = theirs added；D 在第一位 = ours deleted；D 在第二位 = theirs deleted。

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

**代码位置**: [pkg/commands/models/file.go](pkg/commands/models/file.go#L158)

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

**代码位置**: [pkg/commands/models/working_tree_state.go](pkg/commands/models/working_tree_state.go#L45-L59)

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
| WorkingTreeState | IsInCherryPick stopped-sha 读取失败默认分支 | ✅ 有（默认 true，宁杀错不放过） | ✅ 已详细说明 |
| WorkingTreeState | IsInCherryPick 只检查 rebase-merge 不检查 rebase-apply | ✅ 有（遗漏老式 apply rebase） | ✅ 已说明 |
| WorkingTreeState | IsInCherryPick 前缀比较方向 | ✅ 有（方向易错） | ✅ 已详细说明 |
| WorkingTreeState | IsInRebase/Merge/Revert | ❌ 无（纯文件存在性检查） | — |
| WorkingTreeState | Effective() 优先级 | ✅ 有（if 顺序隐含优先级） | ✅ 已说明 |
| File 状态 | unmerged 状态下 U 的语义 | ✅ 有（易误读为 deleted，实际是 conflict） | ✅ 已对照 git 官方文档修正 |
| File 状态 | shortStatus XY 位置约定 | ✅ 有（AU/UA 易写反） | ✅ 已修正并说明 |
| File 状态 | Deleted 字段含义 | ✅ 有（可与冲突状态重叠） | ✅ 已说明 |
| File 状态 | AU/UA/DD 组合语义 | ✅ 有（rename 冲突三胞胎） | ✅ 已说明 |
| File 状态 | HasInlineMergeConflicts | ✅ 有（git status 可能滞后） | ✅ 已说明 |
| mergeconflicts 包 | findConflicts 解析 | ❌ 无 | — |
