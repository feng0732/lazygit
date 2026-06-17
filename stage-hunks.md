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
| 补丁转换 | [transform.go](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go) | 按选中行过滤、hunk 内容过滤、行重排序（核心算法） |
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

`transformHunks()`（[transform.go#L92-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L92-L109)）遍历所有原始 hunk，对每个 hunk 调用 `transformHunk()`，然后检查 `formattedHunk.containsChanges()`，没有任何 +/- 的纯上下文 hunk 会被整体丢弃。

```go
func (self *patchTransformer) transformHunks() []*Hunk {
    newHunks := make([]*Hunk, 0, len(self.patch.hunks))
    startOffset := 0
    for i, hunk := range self.patch.hunks {
        startOffset, formattedHunk = self.transformHunk(hunk, startOffset, self.patch.HunkStartIdx(i))
        if formattedHunk.containsChanges() {
            newHunks = append(newHunks, formattedHunk)
        }
    }
    return newHunks
}
```

**一个关键事实：** `transformHunk()` 签名返回 `(int, *Hunk)`，单个原始 hunk 输入 → 单个新 hunk 输出。**一个原始 @@ hunk 经过 Transform 后，最多产出 1 个新 @@ hunk（或因无变更被丢弃），永远不会被拆成多个新 @@ hunk。** 之前认为"会被切成多个"是错误理解。

---

### 3.6 拆分机制真相：内容拆分 ≠ 边界拆分

#### 3.6.1 算法层面的两个硬约束

从代码结构可以推导出两个硬约束：

| 约束 | 代码证据 | 含义 |
|------|---------|------|
| ① 1:1 映射 | `transformHunk()` 返回单个 `*Hunk`，不是 `[]*Hunk` | 一个原始 hunk → 0 或 1 个新 hunk，边界不会被拆分 |
| ② 顺序保留 | `transformHunkLines()` 按原始顺序遍历 `hunk.bodyLines`，输出是单 `[]*PatchLine` | 单个 hunk 内部行不会被切开分配到多个 hunk |

#### 3.6.2 "拆分"到底是什么？

用户感知的"按块拆分"其实是 **hunk 内部的内容过滤**，不是 hunk 边界的拆分。看测试用例 `TestTransform` → `"adding part of a hunk"`（[patch_test.go#L485-L501](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/patch_test.go#L485-L501)）：

**输入（原始 diff，1 个 @@ hunk 内含两处变更）：**
```diff
@@ -1,5 +1,5 @@
 apple
-grape          // 变更1：删除 grape
+kiwi           // 变更1：新增 kiwi
 orange
-pear           // 变更2：删除 pear
+banana         // 变更2：新增 banana
 lemon
```

**用户选中范围：** lineIndex 6-7（只选中第一处变更 `-grape` 和 `+kiwi`）

**Transform(Reverse=false) 行处理：**

| 原行 | 选中？ | isOldFileLine | 处理结果 |
|------|-------|--------------|---------|
| ` apple` | — | — | 保留 context |
| `-grape` | ✅ | true（DELETION） | 保留 `-grape` |
| `+kiwi` | ✅ | false（ADDITION） | flush pendingContext 后保留 `+kiwi` |
| ` orange` | — | — | flush pendingContext 后保留 context |
| `-pear` | ❌ | true（DELETION） | 转 ` pear` context 入 pendingContext |
| `+banana` | ❌ | false（ADDITION） | 直接丢弃，didSeeUnselectedNewFileLine=true |
| ` lemon` | — | — | flush pendingContext（` pear`）后保留 context |

**输出（仍然是 1 个 @@ hunk）：**
```diff
@@ -1,5 +1,5 @@
 apple
-grape
+kiwi
 orange
 pear     // ← 原本的 -pear 变成了 context 行
 lemon
```

> **关键发现：** 输出仍然只有 1 个 @@ hunk。未选中的 `-pear` 被转成 context 行保住了定位锚点，未选中的 `+banana` 被丢弃。hunk 边界从始至终没有变化，变化的只是** hunk 内部的内容**（哪些行被保留、哪些行被转 context、哪些行被丢弃）。

#### 3.6.3 什么时候输出会有多个 @@ hunk？

只有当**用户选中的范围跨越了多个原始 @@ hunk 的边界**时，输出才会有多个 hunk。看测试用例 `"staging part of both hunks"`（[patch_test.go#L406-L429](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/patch_test.go#L406-L429)）：

**输入（原始 diff，2 个 @@ hunk）：**
```diff
@@ -1,5 +1,5 @@    // hunk 0：line 0-9
 apple
-grape
+orange
 ...
 ...
 ...
@@ -8,6 +8,8 @@    // hunk 1：line 10-19
 ...
 ...
 ...
+pear
+lemon
 ...
 ...
 ...
```

**用户选中范围：** lineIndex 7-15（跨越 hunk 0 和 hunk 1 的边界）
- line 7 属于 hunk 0（`+orange`）
- line 15 属于 hunk 1（`+pear`）

**输出（2 个 @@ hunk，对应原始 2 个 hunk）：**
```diff
@@ -1,5 +1,6 @@    // hunk 0：只保留了选中的 +orange，-grape 转 context
 apple
+orange
 grape
 ...
 ...
 ...
@@ -8,6 +9,7 @@    // hunk 1：只保留了选中的 +pear，+lemon 被丢弃
 ...
 ...
 ...
+pear
 ...
 ...
 ...
```

每个原始 hunk 各自独立处理、各自产出 1 个新 hunk。**输出 hunk 数量 = 被选中范围覆盖的原始 hunk 数量**，和原始 hunk 数量是线性对应的。

#### 3.6.4 pendingContext 冲刷机制对 hunk 边界的影响

`pendingContext` 缓冲和冲刷是**单 hunk 内部的行重排序机制**（[transform.go#L137-L203](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L137-L203)），不会跨越 hunk 边界：

1. **缓冲生命周期：** `pendingContext` 在 `transformHunkLines()` 入口处初始化 `= []*PatchLine{}`，在函数末尾 `flushPendingContext()`，随函数调用栈结束而销毁
2. **冲刷时机：** 遇到 context 行、遇到选中的 oldFileLine、遇到跳过新增行后的选中新增行、遇到 NEWLINE_MESSAGE、遍历结束时
3. **跨 hunk 隔离：** 每个 hunk 调用 `transformHunkLines()` 时有独立的 `pendingContext`，前一个 hunk 未冲刷的上下文绝不会泄漏到后一个 hunk

**为什么需要 pendingContext？** 看这个变更块：

```diff
- old 1    // 未选中 → 转 context，入 pendingContext
- old 2    // ✅选中 → flush pendingContext（old 1 先输出），再输出 -old 2
+ new A    // ✅选中 → 输出 +new A
+ new B    // 未选中 → 丢弃，didSeeUnselectedNewFileLine=true
```

如果没有缓冲直接输出，顺序会是：`-old 2` → `+new A` → ` old 1`（未选中的 old 1 跑到了最后）。pendingContext 保证了正确顺序：` old 1` → `-old 2` → `+new A`。

但这**完全是单个 hunk 内部的行重排序**，不会产生新的 hunk 边界。

#### 3.6.5 不会拆分的边界情况验证

即使在最"应该"拆分的场景下，算法也不会真的拆分 hunk 边界：

**场景：** 一个 hunk 内有两处不相邻的变更，只选中第一处和第三处，跳过中间第二处。

```diff
@@ -1,9 +1,9 @@
 line 1
-del A      // 选中
+add A      // 选中
 line 3
 line 4
-del B      // 未选中
+add B      // 未选中
 line 6
 line 7
-del C      // 选中
+add C      // 选中
```

**Transform 后输出（仍然 1 个 hunk）：**
```diff
@@ -1,9 +1,9 @@
 line 1
-del A
+add A
 line 3
 line 4
 del B      // 未选中的 -del B 转 context
 line 6
 line 7
-del C
+add C
```

未选中的 `-del B` 转成 context 行、`+add B` 被丢弃，中间用 context 行"桥接"起来，整体仍然是**一个连续的 @@ hunk**。算法永远不会在中间插入新的 `@@ ... @@` 行把它切成两个。

#### 3.6.6 拆分机制总结

| 维度 | 事实 | 误区纠正 |
|------|------|---------|
| **hunk 边界** | 原始 hunk 边界是硬边界，不会被切开 | ❌ 一个原始 hunk 会被拆成多个 |
| **输出 hunk 数** | = 选中范围覆盖的原始 hunk 数 | ❌ 可以任意拆分出更多 hunk |
| **pendingContext** | 单 hunk 内部行重排序，不跨边界 | ❌ 会在 hunk 之间传递 |
| **"拆分"的真正含义** | hunk 内部行的选择性保留（+/-）与转 context（未选中的旧文件行） | ❌ 物理上切成多个独立 @@ 块 |

这个设计保证了 `git apply` 始终能通过连续的上下文行定位到正确的应用位置——如果真的把一个 hunk 切成两个，中间的 context 行就会断裂，`git apply` 可能定位失败。

---

## 四、写回流程：从补丁字符串到索引与工作区

### 4.1 写回入口点：两个函数，四种场景

`StagingController` 对外暴露两个写回入口（[staging_controller.go#L204-L234](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L204-L234)）：

```go
// 空格键：暂存 / 取消暂存
func (self *StagingController) ToggleStaged() error {
    return self.applySelectionAndRefresh(self.staged)
    // 主面板 staged=false → reverse=false
    // 副面板 staged=true  → reverse=true
}

// d 键：丢弃变更
func (self *StagingController) DiscardSelection() error {
    // 仅主面板(!staged)且未配置跳过警告时弹窗确认
    return self.c.ConfirmIf(!self.staged && !skipWarning,
        ..., HandleConfirm: func() { return self.applySelectionAndRefresh(true) })
    // 主面板/副面板都是 reverse=true
}
```

两个入口最终都调用 `applySelection(reverse bool)`，参数组合为：

| 场景 | 调用 | `reverse` | `self.staged` |
|------|------|-----------|---------------|
| ①主面板+空格（暂存） | `ToggleStaged()` | `false` | `false` |
| ②副面板+空格（取消暂存） | `ToggleStaged()` | `true` | `true` |
| ③主面板+d（丢弃变更） | `DiscardSelection()` | `true` | `false` |
| ④副面板+d（丢弃暂存） | `DiscardSelection()` | `true` | `true` |

> **关键洞察：** 场景②和场景④的参数完全相同（reverse=true, staged=true），在副面板中按空格和按 d 走的是完全相同的写回路径，均为"取消暂存"。

### 4.2 背景：主副面板的 Diff 语义

理解写回效果的前提是明确两个面板显示的 diff 含义。`StagingHelper` 刷新时（[staging_helper.go#L56-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/helpers/staging_helper.go#L56-L57)）：

```go
mainDiff      = WorktreeFileDiff(file, true, false)  // git diff（无 --cached）
secondaryDiff = WorktreeFileDiff(file, true, true)   // git diff --cached
```

对应底层的 `git diff` 命令（[working_tree.go#L413-L426](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/git_commands/working_tree.go#L413-L426)）：

| 面板 | Git 命令 | 对比双方 | `-` 含义 | `+` 含义 |
|------|---------|---------|---------|---------|
| **主面板（Unstaged）** | `git diff` | **索引 ↔ 工作区** | 索引中有、工作区已删除的行 | 工作区新增、索引中没有的行 |
| **副面板（Staged）** | `git diff --cached` | **HEAD ↔ 索引** | HEAD 中有、索引已删除的行 | 索引新增、HEAD 中没有的行 |

### 4.3 Reverse 参数的双重语义

`reverse` 参数同时作用于**两个独立环节**，不能混淆：

#### 环节 A：Transform.Reverse → 影响哪些行转 context、哪些行丢弃

位于 [transform.go#L167](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L167)：

```go
isOldFileLine := (line.Kind == DELETION && !self.opts.Reverse) || (line.Kind == ADDITION && self.opts.Reverse)
```

这个变量决定了"未选中行"的处理策略：

| Reverse 值 | `isOldFileLine` 为真的行类型 | 未选中时处理 | 另一类行未选中时 |
|-----------|----------------------------|------------|---------------|
| **false**（正向） | DELETION（`-` 行） | 转为 ` ` context（保留在旧文件侧） | ADDITION → 直接丢弃 |
| **true**（反向） | ADDITION（`+` 行） | 转为 ` ` context（保留在旧文件侧） | DELETION → 直接丢弃 |

设计意图：**`isOldFileLine` 标记的是"属于旧文件侧"的行**，对于未选中的旧文件侧行不能真的丢弃（会导致上下文缺失，`git apply` 定位失败），必须转成 context 行保住定位锚点。而属于新文件侧的未选中行本就不存在于旧文件中，直接丢弃不影响上下文。

#### 环节 B：ApplyPatchOpts.Reverse → 传给 `git apply --reverse`

位于 [git_commands/patch.go#L70-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/git_commands/patch.go#L70-L76)：

```
git apply --reverse <patch>
```

含义：**把 patch 整体反方向应用**（原本 `+` 的行改为删除，原本 `-` 的行改为新增）。

#### 两个 Cached/Reverse 的计算

`applySelection()` 中的核心代码（[staging_controller.go#L246-L269](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L246-L269)）：

```go
patchToApply := patch.Parse(state.GetDiff()).
    Transform(patch.TransformOpts{
        Reverse:             reverse,          // ← Transform 用
        IncludedLineIndices: ExpandRange(firstLineIdx, lastLineIdx),
        FileNameOverride:    path,
    }).FormatPlain()

err := self.c.Git().Patch.ApplyPatch(patchToApply, git_commands.ApplyPatchOpts{
    Reverse: reverse,                         // ← git apply 用
    Cached:  !reverse || self.staged,         // ← --cached 标志
})
```

`Cached` 表达式 `!reverse || self.staged` 的真值表：

| reverse | self.staged | Cached | 含义 |
|---------|-------------|--------|------|
| false   | false       | true   | 只改索引 |
| true    | true        | true   | 只改索引 |
| true    | false       | false  | 只改工作区 |
| false   | true        | —      | 此组合永远不会出现 |

### 4.4 四条路径的逐行精确推导

#### 路径①：主面板 + 空格（暂存选中行）

**参数：** `reverse=false, self.staged=false, Cached=true`
**对应 Git 命令：** `git apply --cached <patch>`（正向应用到索引）

**Diff 语义：** 索引 vs 工作区，`-` = 索引内容，`+` = 工作区内容

**Transform(Reverse=false) 的行处理：**

| 原 diff 行 | 选中？ | isOldFileLine=DELETION | 处理结果 |
|-----------|-------|----------------------|---------|
| `- old`（索引中被删） | ✅选中 | true（DELETION类） | 保留 `- old`（表示从索引删除） |
| `- old`（索引中被删） | ❌未选 | true | 转 ` ` context（索引中保留，作为上下文） |
| `+ new`（工作区新增） | ✅选中 | false（ADDITION类） | 保留 `+ new`（表示加入索引） |
| `+ new`（工作区新增） | ❌未选 | false | 直接丢弃（不加入索引，留在工作区） |
| ` ` context | — | — | 原样保留 |

**应用效果：**

| 目标 | 变化 |
|------|------|
| ✅ **索引** | 正向应用选中的变更：选中的工作区新增行 → 加入索引；选中的工作区删除行 → 从索引移除 |
| ✅ **工作区** | `--cached` 标志不触碰工作区文件，**完全不变** |

**变更流向：** 工作区（选中部分） → 索引

---

#### 路径②：副面板 + 空格（取消暂存选中行）

**参数：** `reverse=true, self.staged=true, Cached=true`
**对应 Git 命令：** `git apply --cached --reverse <patch>`（反向应用到索引）

**Diff 语义：** HEAD vs 索引，`-` = HEAD 内容，`+` = 索引内容

**Transform(Reverse=true) 的行处理：**

| 原 diff 行 | 选中？ | isOldFileLine=ADDITION | 处理结果 |
|-----------|-------|----------------------|---------|
| `- old`（HEAD被删） | ✅选中 | false | 保留 `- old`（flush后） |
| `- old`（HEAD被删） | ❌未选 | false | 直接丢弃（didSeeUnselectedNewFileLine=true） |
| `+ new`（索引中新增） | ✅选中 | true（ADDITION类） | flush pendingContext 后保留 `+ new` |
| `+ new`（索引中新增） | ❌未选 | true | 转 ` ` context 入 pendingContext（最后flush） |
| ` ` context | — | — | 原样保留 |

**git apply --reverse 的效果：** 把 diff 意义反转——原本描述"HEAD → 索引"的补丁，反向应用就是还原回 HEAD。

- patch 中 `+ new_line`（索引比 HEAD 多的行）→ reverse = **从索引中删除 new_line**
- patch 中 `- old_line`（索引比 HEAD 少的行）→ reverse = **把 old_line 加回索引**

**应用效果：**

| 目标 | 变化 |
|------|------|
| ✅ **索引** | 选中的变更被还原为 HEAD 状态（即从索引中撤销） |
| ✅ **工作区** | `--cached` 标志不触碰工作区文件，**完全不变** |

**视觉上的"流回工作区"：** 工作区本来就包含这些变更（工作区 = HEAD + 全部暂存变更 + 未暂存变更），当从索引中移除选中变更后，它们自动被 Git 状态识别为"未暂存的工作区变更"，所以看起来像是从暂存区"流回"了工作区面板。

**变更流向：** 索引（选中部分） → 还原为 HEAD → 工作区中自动显现

---

#### 路径③：主面板 + d（丢弃工作区选中变更）

**参数：** `reverse=true, self.staged=false, Cached=false`
**对应 Git 命令：** `git apply --reverse <patch>`（反向应用到工作区，**无 --cached**）

**Diff 语义：** 索引 vs 工作区，`-` = 索引内容，`+` = 工作区内容

**Transform(Reverse=true) 的行处理：** 同路径②（ADDITION 是 oldFileLine 类）

**生成的 patch：** 只保留选中的 +/- 变更行，未选中的 ADDITION 转 context，未选中的 DELETION 丢弃。

**git apply --reverse（无 --cached）的效果：** 反向应用到**工作区**（因为无 --cached 默认作用于工作区）。

- patch 中 `+ new_line`（工作区新增了 new_line）→ reverse = **从工作区中删除 new_line** → 还原为索引状态
- patch 中 `- old_line`（工作区删除了 old_line）→ reverse = **把 old_line 加回工作区** → 还原为索引状态

**应用效果：**

| 目标 | 变化 |
|------|------|
| ✅ **索引** | 无 --cached 标志，索引**完全不变** |
| ✅ **工作区** | 选中的变更被还原为与索引一致（即丢弃本地修改） |

**变更流向：** 工作区（选中部分） → 被索引内容覆盖

---

#### 路径④：副面板 + d（丢弃暂存区选中变更）

**参数：** `reverse=true, self.staged=true, Cached=true`

> 与路径②的参数完全相同 → 走完全相同的代码路径 → **效果与副面板按空格（取消暂存）完全一致。**

副面板中"丢弃暂存的变更"在设计上被等价于"取消暂存"。如果真的要连同工作区一起彻底丢弃（即 `git checkout HEAD -- file` 的行级等价物），需要先取消暂存（路径②④）回到主面板，再在主面板中执行丢弃（路径③）两步操作。

### 4.5 写回路径总表（修订版）

| # | 场景 | reverse | staged | Cached | git apply 标志 | 索引变化 | 工作区变化 | 用户感知 |
|---|------|---------|--------|--------|---------------|---------|-----------|---------|
| ① | 主面板+空格（暂存） | false | false | true | `--cached` | 选中变更加入索引 | 不变 | 选中行从左面板消失 → 出现在右面板 |
| ② | 副面板+空格（取消暂存） | true | true | true | `--cached --reverse` | 选中变更还原为HEAD | 不变 | 选中行从右面板消失 → 出现在左面板 |
| ③ | 主面板+d（丢弃变更） | true | false | false | `--reverse` | 不变 | 选中变更还原为索引 | 选中行从左面板消失，文件中恢复为原始内容 |
| ④ | 副面板+d（丢弃暂存） | true | true | true | `--cached --reverse` | 选中变更还原为HEAD | 不变 | 与②完全相同，取消暂存 |

### 4.6 Git 命令执行细节

`ApplyPatch()` 位于 [git_commands/patch.go#L60-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/git_commands/patch.go#L60-L79)：

```
1. SaveTemporaryPatch(patch)
   → 生成临时文件：~/AppData/Local/Temp/lazygit/RepoName/Jun_17_14.30.00.123456.patch

2. 组装 git apply 参数（按条件组合）：
   git apply
     [--3way]       (用于自定义补丁的冲突处理，暂存面板不用)
     [--cached]     (写入索引，路径①②④使用)
     [--index]      (用于 PatchBuilder 的移动补丁)
     [--reverse]    (反向应用，路径②③④使用)
     <临时文件路径>

3. cmd.New(cmdArgs).Run()
```

**为什么必须用临时文件？** `git apply` 通过 stdin 处理中文文件名、混合换行符、BOM 头等边界情况时存在兼容性问题；写入临时文件再传路径的方案在跨平台场景下稳定得多。

### 4.7 刷新与智能光标定位

应用成功后，`applySelectionAndRefresh()` 触发（[staging_controller.go#L232](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/staging_controller.go#L232)）：

```go
self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.FILES, types.STAGING}})
```

`StagingHelper.RefreshStagingPanel()`（[staging_helper.go#L22-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/controllers/helpers/staging_helper.go#L22-L115)）会：

1. **重算 Diff：** 再次调用 `WorktreeFileDiff()` 生成 unstaged + staged 两份最新 diff
2. **重建 State：** `patch_exploring.NewState(diff, ...)` 解析新 diff 并建立索引映射
3. **智能跳行：** `NewState()` 的光标定位逻辑（[state.go#L87-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/gui/patch_exploring/state.go#L87-L112)）：
   - 取旧状态的 `patchLineIdx` → `GetNextChangeIdx()` 跳到下一条变更行
   - 特殊处理重排：暂存 addition 后未选中的 deletion 会被重排到前面，导致 cursor 恰好落在 deletion 上 → 检测到这种状态时跳过所有连续 deletion，再找下一条有效变更

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
| 选中范围跨越多个原始 hunk | 每个原始 hunk 独立处理、独立产出 1 个新 hunk，输出 hunk 数 = 被覆盖的原始 hunk 数 | [transform.go#L92-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L92-L109) |
| 单个 hunk 内部分变更被跳过 | 未选中的 old file 行转 context 行桥接，不会在中间插入新 `@@` 头拆分 hunk | [transform.go#L125-L205](file:///d:/fz/0601-2/solo-dogfeeding/code/21-lazygit/pkg/commands/patch/transform.go#L125-L205) |
