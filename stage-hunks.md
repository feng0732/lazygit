# Lazygit 暂存区按块编辑（Hunk Staging）代码分析

## 一、架构总览

Lazygit 的暂存区按块编辑由以下核心模块协作完成：

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户交互层                                │
│  StagingController / PatchBuildingController (键盘事件)          │
│         ↓ selectMode: LINE / RANGE / HUNK                       │
├─────────────────────────────────────────────────────────────────┤
│                      状态管理层                                  │
│  patch_exploring.State (光标位置、选中范围、视图索引映射)         │
│         ↓ SelectedPatchRange() → [firstLineIdx, lastLineIdx]    │
├─────────────────────────────────────────────────────────────────┤
│                      补丁构建层                                  │
│  patch.Parse → patch.Transform(IncludedLineIndices) → FormatPlain│
│         ↓ 统一diff格式补丁字符串                                  │
├─────────────────────────────────────────────────────────────────┤
│                      Git 命令层                                  │
│  PatchCommands.ApplyPatch → git apply --cached/--reverse        │
└─────────────────────────────────────────────────────────────────┘
```

**核心文件定位：**

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 状态管理 | [state.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go) | 光标位置、选择模式、视口-补丁行索引映射 |
| 暂存控制器 | [staging_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go) | 键盘绑定、`ToggleStaged` / `DiscardSelection` / `EditHunk` |
| 补丁数据结构 | [patch.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/patch.go) | `Patch` / `Hunk` / `PatchLine` 三层模型 |
| 补丁转换 | [transform.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go) | 按选中行过滤、hunk拆分、行重排序（核心算法） |
| 补丁构建器 | [patch_builder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/patch_builder.go) | 自定义补丁的文件级行集合管理 |
| Git 命令 | [patch.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/git_commands/patch.go) | `git apply` 参数组合、临时 patch 文件 |
| 辅助刷新 | [staging_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/helpers/staging_helper.go) | 双面板 diff 渲染、状态同步 |

---

## 二、交互状态管理：选择模式的拆分与流转

### 2.1 三种选择模式

定义在 [state.go#L40-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L40-L46)：

```go
const (
    LINE  selectMode = iota  // 单行模式：只选中光标所在行
    RANGE                    // 范围模式：从 rangeStartLineIdx 到 selectedLineIdx
    HUNK                     // 块模式：选中当前连续变更块
)
```

**State 关键字段**（[state.go#L16-L37](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L16-L37)）：

```go
type State struct {
    selectedLineIdx   int          // 当前光标行（视图行，非补丁行）
    rangeStartLineIdx int          // 范围选择起点
    rangeIsSticky     bool         // 范围是否随光标移动而扩展
    selectMode        selectMode   // 当前选择模式
    userEnabledHunkMode bool       // 用户是否手动开启了 hunk 模式

    // 索引映射：处理视图换行（wrapped lines）
    viewLineIndices  []int  // patchLineIdx → viewLineIdx（多对一）
    patchLineIndices []int  // viewLineIdx → patchLineIdx（一对多）
}
```

### 2.2 视图行与补丁行的映射（Wrap 处理）

由于视图可能开启自动换行（`Gui.WrapLinesInStagingView`），**一条补丁逻辑行可能对应多条视图显示行**。`wrapPatchLines()` 函数（[state.go#L426-L430](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L426-L430)）建立双向索引：

- `viewLineIndices[i]`：补丁第 `i` 行在视图中的**起始行号**
- `patchLineIndices[j]`：视图第 `j` 行对应的**补丁行号**

所有光标操作（上下移动、范围选择）都基于 `viewLineIdx`，在真正计算选中补丁范围时通过 `SelectedPatchRange()`（[state.go#L370-L373](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L370-L373)）转换为补丁行索引。

### 2.3 模式切换与状态流转

**模式切换入口：**

| 操作 | 函数 | 行为 |
|------|------|------|
| `ToggleSelectHunk`（默认键 `=`） | [state.go#L156-L168](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L156-L168) | HUNK ↔ LINE 互切，HUNK 模式下自动跳到下一条变更行 |
| `ToggleSelectRange`（Shift+方向） | [state.go#L174-L182](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L174-L182) | LINE → RANGE，sticky=false 时移动光标会自动取消范围 |
| `ToggleStickySelectRange`（v 键） | [state.go#L170-L172](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L170-L172) | 同上，但 sticky=true，光标移动时范围扩展 |
| `Escape` | [staging_controller.go#L171-L180](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L171-L180) | 若处于 RANGE 或手动 HUNK → 退回 LINE；否则退出面板 |

**HUNK 模式的选中计算**（[state.go#L329-L351](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L329-L351)）：

```go
func (s *State) selectionRangeForCurrentBlockOfChanges() (int, int) {
    // 从光标行向前回溯，直到遇到非变更行 → patchStart
    for patchStart > 0 && patchLines[patchStart-1].IsChange() { patchStart-- }
    // 从光标行向后扩展，直到遇到非变更行 → patchEnd
    for patchEnd < len(patchLines)-1 && patchLines[patchEnd+1].IsChange() { patchEnd++ }
    // 再转换为视图行范围（考虑换行展开）
}
```

> **关键洞察：HUNK 模式选中的不是 Git diff 中的 hunk（@@ 标记的块），而是**连续的变更行块**（连续的 +/- 行，不含 context 空格行）。这允许比原生 hunk 更细粒度的拆分。

### 2.4 选中范围的统一计算

`SelectedViewRange()` → `SelectedPatchRange()` 是状态层对外的核心输出，屏蔽三种模式差异：

```
LINE 模式:  (selectedLineIdx, selectedLineIdx)
RANGE 模式: (min(rangeStart, selected), max(rangeStart, selected))
HUNK 模式:  selectionRangeForCurrentBlockOfChanges()
```

后续补丁生成只依赖这对 `(firstLineIdx, lastLineIdx)`，完全不知道选择模式的存在。

---

## 三、补丁生成：Transform 的核心算法

### 3.1 Patch 三层数据模型

解析器 `Parse()`（[parse.go#L12-L42](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/parse.go#L12-L42)）将 diff 字符串解析为三层结构：

```
Patch
├── header: []string                    // diff --git / index / --- / +++
└── hunks: []*Hunk
    ├── oldStart, newStart int          // @@ -oldStart,oldLen +newStart,newLen @@
    ├── headerContext string            // @@ 后的函数上下文
    └── bodyLines: []*PatchLine
        ├── Kind: PATCH_HEADER / HUNK_HEADER / ADDITION / DELETION / CONTEXT / NEWLINE_MESSAGE
        └── Content: string             // 含前缀字符 +/- / 空格
```

### 3.2 Transform 的四个关键参数

`TransformOpts`（[transform.go#L14-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L14-L38)）：

| 参数 | 作用 | 暂存场景取值 |
|------|------|------------|
| `Reverse` | 反转补丁语义（+/- 角色互换） | `staged=true`（取消暂存）或 `DiscardSelection` 时为 true |
| `IncludedLineIndices` | 选中的补丁行号切片（由 `ExpandRange` 展开） | `ExpandRange(firstLineIdx, lastLineIdx)` |
| `FileNameOverride` | 强制重写 patch 头的文件名 | 设为当前文件路径，避免 `a/` `b/` 前缀混淆 |
| `TurnAddedFilesIntoDiffAgainstEmptyFile` | 新文件以 `/dev/null` 对比 | 仅自定义补丁构建用 |

### 3.3 transformHunkLines：行重排序的核心算法

这是整个按块编辑最精妙的部分，位于 [transform.go#L125-L205](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L125-L205)。

**问题背景：** 当只选择一个连续变更块中的部分行时，未选中的行不能直接丢弃——否则 hunk 的行数统计（@@ -x,y +a,b @@）会出错，且 `git apply` 无法定位上下文。

**处理规则（非 Reverse 时，即正向暂存）：**

| 行类型 | 选中 | 未选中 |
|--------|------|--------|
| `-` 删除行（old file 行） | 保留 `-` | 转为 ` ` 上下文行，进入 pendingContext 缓冲 |
| `+` 新增行（new file 行） | 保留 `+` | 直接丢弃 |
| ` ` 上下文行 | 保留 ` ` | 保留 ` ` |

**为什么需要 pendingContext 缓冲？** 考虑这样的变更块：

```diff
- old line 1   // 未选中
- old line 2   // 选中
+ new line A   // 选中
+ new line B   // 未选中
```

如果未选中的 `- old line 1` 立即转为上下文输出，结果会是：

```diff
  old line 1   // context
- old line 2
+ new line A
```

看似没问题，但如果选中的是后面的新增行、未选中的是前面的删除行呢？输出顺序会混乱。因此算法使用 **延迟冲刷（lazy flush）** 策略：

1. **未选中删除行 → 入队 pendingContext**（不立即输出）
2. **遇到选中删除行 / 遇到选中新增行（且之前跳过了未选中新增行） → 冲刷 pendingContext**
3. **遇到上下文行 / 遍历结束 → 冲刷 pendingContext**

这样保证了输出顺序永远是：
```
[选中删除行] → [选中新增行] → [未选中删除行转的上下文]
```

**Reverse=true 时（取消暂存 / 丢弃变更）：** 规则对称反转——未选中的 `+` 转上下文缓冲，未选中的 `-` 直接丢弃。

### 3.4 Hunk 头的动态重算

行过滤完成后，`transformHunkHeader()`（[transform.go#L207-L227](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L207-L227)）重新计算每个 hunk 的 `@@ -oldStart,oldLen +newStart,newLen @@`：

```
oldLen = count(CONTEXT) + count(DELETION)
newLen = count(CONTEXT) + count(ADDITION)
```

还维护了 `startOffset` 跨 hunk 累计偏移：前一个 hunk 的 `newLen - oldLen` 决定了下一个 hunk 的 `newStart` 需要偏移多少。

**边界处理：**
- `oldLen == 0`：hunk 变成纯插入，`newStart += 1`
- `newLen == 0`：hunk 变成纯删除，`newStart -= 1`

### 3.5 空 Hunk 过滤

`transformHunks()`（[transform.go#L92-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L92-L109)）最后检查 `formattedHunk.containsChanges()`，没有任何 +/- 的纯上下文 hunk 会被整体丢弃。这就是"拆分"的本质：**一个原始 @@ hunk 经行过滤后，可能被切成 0 个、1 个或多个新的 @@ hunk**。

---

## 四、写回流程：从补丁字符串到 Git 索引

### 4.1 StagingController 的主路径

`ToggleStaged` → `applySelectionAndRefresh(reverse=false)` → `applySelection(reverse)`：

核心代码位于 [staging_controller.go#L236-L280](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L236-L280)：

```go
func (self *StagingController) applySelection(reverse bool) error {
    // 1. 从状态层取选中范围
    firstLineIdx, lastLineIdx := state.SelectedPatchRange()

    // 2. 二次 Parse + Transform（不依赖缓存，保证幂等）
    patchToApply := patch.
        Parse(state.GetDiff()).
        Transform(patch.TransformOpts{
            Reverse:             reverse,
            IncludedLineIndices: patch.ExpandRange(firstLineIdx, lastLineIdx),
            FileNameOverride:    path,
        }).
        FormatPlain()

    // 3. 调用 Git 命令层
    err := self.c.Git().Patch.ApplyPatch(patchToApply, git_commands.ApplyPatchOpts{
        Reverse: reverse,
        Cached:  !reverse || self.staged,   // ← 关键参数组合
    })
}
```

### 4.2 ApplyPatchOpts 参数组合矩阵

这是理解"暂存/取消暂存/丢弃"三种操作差异的关键：

| 用户操作 | 控制器 | `reverse` 参数 | `staged` 上下文 | `git apply` 标志 | 效果 |
|---------|--------|---------------|----------------|-----------------|------|
| **暂存**（按空格） | 主面板（unstaged） | false | false | `--cached` | 工作区变更 → 暂存区 |
| **取消暂存**（按空格） | 副面板（staged） | false | true | `--cached` | 暂存区变更 → 工作区（反向应用索引补丁） |
| **丢弃变更**（按 d） | 主面板（unstaged） | true | false | `--cached` `--reverse` | 工作区选中行被丢弃（反向应用） |
| **丢弃暂存**（按 d） | 副面板（staged） | true | true | `--cached` `--reverse` | 暂存区选中行被丢弃，工作区保留 |

`Cached` 标志的计算：`!reverse || self.staged`，解读为：
- 正向操作（reverse=false）：总是 `--cached`，写入索引
- 反向操作（reverse=true）：只有在 staged 面板才 `--cached`，否则只改工作区

### 4.3 Git 命令执行

`ApplyPatch()` 位于 [git_commands/patch.go#L60-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/git_commands/patch.go#L60-L79)：

```
1. SaveTemporaryPatch(patch) → 生成临时文件 ~/lazygit/RepoName/Jun_17_14.30.00.123456.patch
2. 组装 git apply 参数: --3way / --cached / --index / --reverse 按需组合
3. cmd.New(cmdArgs).Run()
```

**为什么不直接 stdin 管道？** 因为 `git apply` 处理中文文件名、换行等边界情况时，临时文件方案更稳定。

### 4.4 刷新与光标保持

应用成功后，`applySelectionAndRefresh()` 触发：

```go
self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.FILES, types.STAGING}})
```

`StagingHelper.RefreshStagingPanel()`（[staging_helper.go#L22-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/helpers/staging_helper.go#L22-L115)）会：

1. 重新读取 `WorktreeFileDiff()`（未暂存 + 已暂存两份）
2. 调用 `patch_exploring.NewState()` 重建状态
3. `NewState()` 的智能光标定位逻辑（[state.go#L87-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L87-L112)）：
   - 取旧状态的 `patchLineIdx` → `GetNextChangeIdx()` 跳到下一条变更
   - 特殊处理"暂存 addition 后 cursor 落在 deletion 上"的重排问题

---

## 五、按块编辑（EditHunk）：手动修改补丁

按 `e`（`EditHunk` 键）走一条独立路径，位于 [staging_controller.go#L291-L349](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L291-L349)：

```
流程：
  1. CurrentHunkBounds() → (hunkStartIdx, hunkEndIdx)
  2. Transform 生成**完整 hunk** 补丁（ExpandRange 整个 hunk）
  3. SaveTemporaryPatch → 写入 .patch 临时文件
  4. EditFileAtLineAndWait(patchFilepath, 当前行在 hunk 内的偏移 + 3)
        ↑ +3 是因为 patch 头有 3 行（diff --git / --- / +++）
  5. 等待编辑器关闭（这是同步调用，用户可以任意修改）
  6. Cat() 重新读取编辑后的 .patch 文件
  7. Parse + Transform（ExpandRange(0, lineCount)）→ 规范化
  8. ApplyPatch(Reverse=self.staged, Cached=true)
```

**设计要点：**
- 第 7 步的二次 `Transform` 即使选中了全部行也不是多余的——它会：
  - 重新计算正确的 @@ hunk 头（用户可能改乱了）
  - 标准化文件名（用 `FileNameOverride`）
  - 过滤掉空 hunk
- 永远使用 `Cached=true`：编辑后的补丁总是先作用到索引，再由用户决定是否 `git restore --staged` 或继续编辑

---

## 六、自定义补丁构建（PatchBuilder）

除了暂存面板的即时暂存/取消暂存，Lazygit 还有一套更强大的 **自定义补丁** 机制（从历史 commit 中抽取行、移动到其他 commit 或索引）。

`PatchBuilder`（[patch_builder.go#L35-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/patch_builder.go#L35-L51)）维护一个文件级映射：

```
fileInfoMap[filename] → {
    mode: UNSELECTED / WHOLE / PART,
    includedLineIndices: []int,   // 已选中的补丁行号（按文件维度累积）
    diff: string                  // 该文件的完整 diff
}
```

**与即时暂存的核心差异：**

| 维度 | StagingController（即时暂存） | PatchBuilder（自定义补丁） |
|------|------------------------------|--------------------------|
| 选择状态 | State.selectMode（LINE/RANGE/HUNK，临时） | PatchBuilder.fileInfoMap（持久，跨文件累积） |
| 选中语义 | 本次操作的范围 | 补丁中要包含的所有行（多次 toggle 累积） |
| 应用时机 | 按空格立即应用 | 选择"丢弃/移动到索引/新 commit"等动作时批量应用 |
| 撤销方式 | 反向 `git apply` | 从 includedLineIndices 移除行号 |

`PatchBuildingController.toggleSelection()`（[patch_building_controller.go#L137-L179](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/patch_building_controller.go#L137-L179)）的智能切换：

```go
// 取选中范围内的变更行（只处理 +/-，跳过上下文）
lineIndicesToToggle := state.LineIndicesOfAddedOrDeletedLinesInSelectedPatchRange()

// 检查第一条选中行的状态，决定批量 add 还是批量 remove
firstSelectedChangeLineIsStaged := lo.Contains(includedLineIndices, lineIndicesToToggle[0])
if firstSelectedChangeLineIsStaged {
    toggleFunc = PatchBuilder.RemoveFileLineRange
} else {
    toggleFunc = PatchBuilder.AddFileLineRange
}
```

> 这意味着：**如果选中范围跨越了已选和未选区域，整体行为以第一条行为准**——这是为了交互一致性而牺牲的边界精确性。

---

## 七、关键算法细节

### 7.1 Hunk 边界定位

`Patch.HunkContainingLine()`（[patch.go#L118-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/patch.go#L118-L126)）线性遍历所有 hunk，用累计 `lineCount()` 定位。注意：**索引包含 header 行**，所以 `HunkStartIdx = len(header) + sum(prevHunk.lineCount())`。

### 7.2 光标智能跳转

`State` 构造时的特殊逻辑（[state.go#L99-L108](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L99-L108)）：

```
背景：当暂存了一个 addition 后，未被暂存的 deletion 在新 diff 中会被
     重新排到 remaining addition 之前，导致 cursor 恰好落在 deletion 上。

检测条件：
  - newPatchLineIdx == oldPatchLineIdx（位置没变）
  - 原来是 Addition，现在变成 Deletion（类型翻转）
  - 仍在同一个 hunk 内（HunkOldStartForLine 相同）

处理：跳过所有连续 deletion，再 GetNextChangeIdx 找下一条有效变更
```

### 7.3 行号映射（编辑器跳转）

`Patch.LineNumberOfLine()`（[patch.go#L92-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/patch.go#L92-L115)）将补丁行号映射为**新文件中的实际行号**（用于按 e 跳转到编辑器对应行）：

```
行号 = hunk.newStart + count(hunk内此前的 CONTEXT + ADDITION 行数)
```

因为 deletion 不出现在新文件中，所以只累计 `CONTEXT` 和 `ADDITION`。

---

## 八、协作流程时序图（暂存单行场景）

```
用户按 [空格]
    │
    ▼
StagingController.ToggleStaged()
    │  reverse = false (正向暂存)
    ▼
applySelection(reverse=false)
    │
    ├─► State.SelectedPatchRange()          ──── (3,5)
    │
    ├─► patch.Parse(currentDiff)            ──── Patch{header, hunks}
    │
    ├─► patch.Transform(                    ──── 核心处理
    │     Reverse:false,
    │     IncludedLineIndices:[3,4,5],
    │     FileNameOverride:"foo.go"
    │   )
    │     ├─ transformHeader()              ──── 重写 ---/+++ 行
    │     └─ transformHunks()
    │         └─ transformHunkLines()
    │             ├─ pendingContext 缓冲
    │             ├─ 保留选中 +/-
    │             └─ transformHunkHeader()  ──── 重算 @@ 行
    │
    ├─► FormatPlain()                       ──── 生成标准 .patch 字符串
    │
    └─► Git().Patch.ApplyPatch(
          patchText,
          { Reverse:false, Cached:true }
        )
          ├─ SaveTemporaryPatch(patchText)  ──── 写临时文件
          └─ git apply --cached /tmp/xxx.patch
```

---

## 九、边界情况汇总

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| Diff 上下文为 0 | 直接拒绝操作并提示增加上下文 | [staging_controller.go#L205-L208](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L205-L208) |
| Transform 后生成空 patch | `patchToApply == ""` 时直接 return，不调 git apply | [staging_controller.go#L256-L258](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L256-L258) |
| 只选中了上下文行（空格开头） | `LineIndicesOfAddedOrDeletedLines` 返回空 → 不操作 | [state.go#L376-L387](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L376-L387) |
| 视图宽度变化（换行改变） | `OnViewWidthChanged()` 重建索引映射并保留选中行锚点 | [state.go#L127-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L127-L142) |
| 跨视图面板切换 | `TogglePanel()` 在主/副面板间跳转而不返回文件面板 | [staging_controller.go#L196-L202](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L196-L202) |
