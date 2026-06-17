# 文件状态链路聚焦：自动滚动、非焦点视图选中、过滤归零对定位的影响

---

## 一、当前视图刷新时是否自动滚动到选中行

### 1.1 结论：会自动滚动

当 Files 视图是当前焦点视图时，刷新**会自动将选中行滚动到可见区域（并居中）**。

### 1.2 精确调用链

```
refreshView(Files)
    ↓
PostRefreshUpdate(context)
    ├─ HandleRender()              ← 渲染内容到视图
    └─ 当前视图 == Files
        └─ HandleFocus(types.OnFocusOpts{})
            └─ FocusLine(opts.ScrollSelectionIntoView)
                └─ OnFocusOpts{}.ScrollSelectionIntoView = true  ← 默认值
                    ↓
                AfterLayout:
                    FocusPoint(viewIdx, scrollIntoView=true)
```

### 1.3 OnFocusOpts.ScrollSelectionIntoView 的默认值

位置：[context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/types/context.go#L229-L233)

```go
type OnFocusOpts struct {
    ClickedWindowName       string
    ClickedViewLineIdx      int
    ScrollSelectionIntoView bool  // 默认零值为 false
}
```

**注意**：结构体的零值是 `false`。需要看 `HandleFocus` 如何处理这个零值。

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L91-L97)

```go
func (self *ListContextTrait) HandleFocus(opts types.OnFocusOpts) {
    self.FocusLine(opts.ScrollSelectionIntoView)
    // ...
}
```

位置：[view_helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/view_helpers.go#L127-L137)

```go
if gui.currentViewName() == c.GetViewName() {
    c.HandleFocus(types.OnFocusOpts{})  // ← 空结构体，ScrollSelectionIntoView = false
}
```

**更正**：`PostRefreshUpdate` 中调用 `HandleFocus(types.OnFocusOpts{})`，此时 `ScrollSelectionIntoView = false`。

但等等——让我确认 `FocusLine` 的逻辑。

### 1.4 FocusPoint 的实际滚动行为

位置：[view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gocui/view.go#L374-L391)

```go
func (v *View) FocusPoint(cx int, cy int, scrollIntoView bool) {
    v.writeMutex.Lock()
    defer v.writeMutex.Unlock()

    v.refreshViewLinesIfNeeded()
    lineCount := len(v.viewLines)
    if cy < 0 || cy > lineCount {
        return
    }

    if scrollIntoView {
        height := v.InnerHeight()
        v.oy = calculateNewOrigin(cy, v.oy, lineCount, height)  // ← 仅此处调整原点
    }

    v.cx = cx
    v.cy = cy - v.oy  // ← 光标位置始终更新
}
```

**关键差异**：
- `scrollIntoView = true`：更新 `v.oy`（视口原点）→ 滚动视图
- `scrollIntoView = false`：**不**更新 `v.oy` → 不滚动
- 无论哪种：`v.cy = cy - v.oy` 都会更新（光标相对位置）

### 1.5 calculateNewOrigin 的滚动策略

位置：[view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gocui/view.go#L401-L423)

```go
func calculateNewOrigin(selectedLine int, oldOrigin int, lineCount int, viewHeight int) int {
    if viewHeight >= lineCount {
        return 0  // 全部可见，原点归零
    } else if selectedLine < oldOrigin || selectedLine >= oldOrigin+viewHeight {
        // 选中行在视口外：滚动使选中行居中
        newOrigin := selectedLine - viewHeight/2
        // 边界夹紧
        maxOrigin := lineCount - viewHeight
        if newOrigin > maxOrigin { newOrigin = maxOrigin }
        if newOrigin < 0 { newOrigin = 0 }
        return newOrigin
    }
    return oldOrigin  // 选中行已在视口内，不动
}
```

**滚动策略**：仅当选中行在视口外时才滚动，滚动后选中行位于视口中间。

### 1.6 PostRefreshUpdate 中刷新当前视图的实际行为

位置：[view_helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/view_helpers.go#L127-L137)

```go
if gui.currentViewName() == c.GetViewName() {
    c.HandleFocus(types.OnFocusOpts{})  // ScrollSelectionIntoView = false
}
```

**所以 PostRefreshUpdate 刷新当前视图时，scrollIntoView 是 false**，不会强制滚动。

但如果选中行的模型索引没变，内容重新渲染后光标还在原来的相对位置（`v.cy = cy - v.oy` 不变），用户看到的选中行不会跳动。

**什么时候会真正滚动？**
- 用户首次切换到 Files 视图（通过键盘快捷键），此时上下文切换会传 `ScrollSelectionIntoView = true`
- 用户搜索跳转时

### 1.7 场景对比

| 场景 | scrollIntoView | 是否滚动 |
|------|---------------|---------|
| 后台自动刷新（当前在 Files 视图） | false | 不滚动 |
| 用户操作后 PostRefreshUpdate（当前在 Files） | false | 不滚动 |
| 切换到 Files 视图（键盘） | true | 选中行不在视口内则滚动居中 |
| 搜索结果跳转 | true | 滚动 |
| 非焦点视图刷新 | false | 不滚动（连光标视觉更新也受限） |

---

## 二、非当前视图如何只更新选中状态

### 2.1 非焦点视图的处理分支

位置：[view_helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/view_helpers.go#L135-L163)

```go
if gui.currentViewName() == c.GetViewName() {
    c.HandleFocus(types.OnFocusOpts{})
} else {
    // 注释说明：FocusLine 不包含在 HandleFocus 中，需要单独调用
    // 目的：确保非激活选中状态被正确绘制，且集成测试能看到最新选中状态
    c.FocusLine(false)  // ← scrollIntoView = false

    // 如果当前上下文在主面板，且这个侧面板是"堆栈中的下一个"
    currentCtx := gui.State.ContextMgr.Current()
    if currentCtx 是 normal main/secondary {
        if !currentCtx.GetView().IsSearching() {
            sidePanelContext := gui.State.ContextMgr.NextInStack(currentCtx)
            if sidePanelContext != nil && sidePanelContext.GetKey() == c.GetKey() {
                sidePanelContext.HandleRenderToMain()  // ← 更新主面板的 diff 显示
            }
        }
    } else if c.GetKey() == gui.State.ContextMgr.CurrentStatic().GetKey() {
        // 当前有弹窗，c 是弹窗后面的静态上下文
        c.HandleRenderToMain()  // ← 刷新弹窗后面的主视图
    }
}
```

### 2.2 FocusLine(false) 做了什么

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L35-L75)

```go
func (self *ListContextTrait) FocusLine(scrollIntoView bool) {
    self.Context.FocusLine(scrollIntoView)  // 基类空实现

    self.c.AfterLayout(func() error {
        oldOrigin, _ := self.GetViewTrait().ViewPortYBounds()

        // ① 更新模型光标 → 视图光标的映射
        self.GetViewTrait().FocusPoint(
            self.ModelIndexToViewIndex(self.list.GetSelectedLineIdx()),
            scrollIntoView)  // ← false，不滚动

        // ② 更新搜索位置（视觉高亮）
        if !inOnSearchSelect {
            self.GetView().SetNearestSearchPosition()
        }

        // ③ 更新范围选择起点
        selectRangeIndex, isSelectingRange := self.list.GetRangeStartIdx()
        if isSelectingRange {
            self.GetViewTrait().SetRangeSelectStart(selectRangeIndex)
        } else {
            self.GetViewTrait().CancelRangeSelect()
        }

        // ④ 如果只渲染可见行，按需刷新视口内容
        if self.refreshViewportOnChange {
            self.refreshViewport()
        } else if self.renderOnlyVisibleLines {
            newOrigin, _ := self.GetViewTrait().ViewPortYBounds()
            if oldOrigin != newOrigin || self.needRerenderVisibleLines {
                self.refreshViewport()
            }
        }
        return nil
    })

    self.setFooter()  // ⑤ 更新页脚 "x of y"
}
```

**scrollIntoView=false 时的具体行为**：

在 gocui 的 FocusPoint 中：

```go
func (v *View) FocusPoint(cx int, cy int, scrollIntoView bool) {
    // ...
    if scrollIntoView {
        v.oy = calculateNewOrigin(cy, v.oy, lineCount, height)
    }
    v.cx = cx
    v.cy = cy - v.oy  // ← 即使不滚动，cy 也会更新
}
```

所以：
- `v.oy`（视口原点）**不变** → 不滚动
- `v.cy`（光标在视口内的相对位置）**更新** → 选中行高亮位置跟随模型变化

### 2.3 非焦点视图的高亮颜色

位置：[view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/view_trait.go#L46-L49)

```go
func (self *ViewTrait) SetHighlight(highlight bool) {
    self.view.Highlight = highlight
    self.view.HighlightInactive = false  // ← 非焦点视图使用 Inactive 颜色
}
```

但非焦点视图不会调用 `HandleFocus`，所以不会调用 `SetHighlight(true)`。

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L91-L97)

```go
func (self *ListContextTrait) HandleFocus(opts types.OnFocusOpts) {
    self.FocusLine(opts.ScrollSelectionIntoView)
    self.GetViewTrait().SetHighlight(self.list.Len() > 0)  // ← 仅 HandleFocus 调用
    self.Context.HandleFocus(opts)
}
```

**非焦点视图不会调用 SetHighlight**。这意味着非焦点视图的 `view.Highlight` 保持为 `false`。

但在渲染时：

位置：[view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gocui/view.go#L567-L590)

```go
} else if v.Highlight {
    // 使用选中行颜色（焦点视图）
    if v.HighlightInactive {
        bgColor = v.InactiveViewSelBgColor  // 非焦点视图选中色
    } else {
        bgColor = v.SelBgColor              // 焦点视图选中色
    }
}
```

如果 `v.Highlight = false`，即使 `v.cy` 指向某一行，该行也不会被高亮。

等等——让我再仔细看。非焦点视图的 `Highlight` 应该是 `HighlightInactive`。让我重新查找 ViewTrait.SetHighlight 的调用点。

### 2.4 非焦点视图的实际高亮机制

实际上 `FocusPoint` 只更新光标位置，但渲染选中高亮需要 `Highlight = true`。

非焦点视图虽然不调用 `HandleFocus`，但在 `HandleRender` 中已经完成了内容渲染。渲染时选中行是否被高亮取决于 `view.Highlight`。

当焦点从 Files 视图移走时：

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L99-L107)

```go
func (self *ListContextTrait) HandleFocusLost(opts types.OnFocusLostOpts) {
    self.GetViewTrait().SetOriginX(0)
    if self.refreshViewportOnChange {
        self.refreshViewport()
    }
    self.Context.HandleFocusLost(opts)
}
```

`HandleFocusLost` **没有**设置 `HighlightInactive`。但 Context 基类可能会处理。

但根据注释：
> "The FocusLine call is included in the HandleFocus method which we call for focused views above; but we need to call it here for non-focused views to ensure that an inactive selection is painted correctly, and that integration tests see the up to date selection state."

注释明确说明：非焦点视图调用 `FocusLine(false)` 是为了确保**非激活选中状态被正确绘制**，且**集成测试能看到最新的选中状态**。

结合代码逻辑，`FocusPoint` 更新了 `v.cy`，即使 `Highlight = false`，gocui 内部仍能在需要时（如集成测试）通过 `SelectedLineIdx()` 获取光标位置：

位置：[view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gocui/view.go#L1648-L1651)

```go
func (v *View) SelectedLineIdx() int {
    _, seletedLineIdx := v.SelectedPoint()
    return seletedLineIdx
}

func (v *View) SelectedPoint() (int, int) {
    cx, cy := v.Cursor()
    ox, oy := v.Origin()
    return cx + ox, cy + oy
}
```

**结论**：非焦点视图通过以下方式"只更新选中状态"：
1. `HandleRender()` → 渲染全部内容（含数据更新）
2. `FocusLine(false)` → `FocusPoint(cy, false)`
   - 更新 `v.cy`：光标相对位置，使 `SelectedLineIdx()` 返回正确值
   - 不更新 `v.oy`：不滚动视口
   - scrollIntoView=false：不调用 `calculateNewOrigin`
3. 页脚更新：`"x of y"` 正确显示当前选中位置
4. 如果是非焦点但影响主面板（如侧面板选中文件变了），调用 `HandleRenderToMain()` 更新主面板 diff

---

## 三、过滤切换归零后重取状态对光标定位的影响

### 3.1 过滤切换的精确时序

以切换到 `DisplayUntracked` 为例：

```
用户选择"只显示未跟踪"
    ↓
setStatusFiltering(DisplayUntracked)
    ↓
步骤1：FileTreeViewModel.SetStatusFilter(DisplayUntracked)
    ├─ ① IFileTree.SetStatusFilter(DisplayUntracked)
    │   ├─ self.filter = DisplayUntracked
    │   └─ self.SetTree()                       ← 用旧 Model.Files 重建树
    │       └─ getFilesForDisplay()
    │           └─ FilterFiles(DisplayUntracked) ← 旧数据过滤，可能为空
    │
    └─ ② IListCursor.SetSelection(0)             ← 光标归零！
        └─ selectedIdx = 0，取消范围选择

步骤2：视图副标题 = "Untracked files"

步骤3：previousFilter != DisplayUntracked → 触发 Refresh(ASYNC)
    ↓
[worker goroutine]
    ↓
refreshFilesAndSubmodules()
    ├─ 获取 RefreshingFilesMutex
    ├─ refreshStateSubmoduleConfigs()
    └─ refreshStateFiles()
        ├─ GetStatusFiles(ForceShowUntracked=true) ← 重新获取文件列表
        ├─ Model.Files = files                       ← 新数据（含未跟踪文件）
        └─ fileTreeViewModel.SetTree()               ← 再次重建树
            ├─ newFiles = GetAllFiles()              ← 新文件列表
            ├─ selectedNode = GetSelected()         ← GetSelectedLineIdx()=0 → Get(0)
            │    ↓
            │    如果 SetStatusFilter 后旧数据过滤结果为空 → Len()=0 → GetSelected()=nil
            │    如果有数据 → GetSelected()=第0个节点
            │
            ├─ ExpandToPath 处理（仅 selectedNode 存在时）
            ├─ prevNodes = GetAllItems()
            ├─ prevSelectedLineIdx = GetSelectedLineIdx()  ← 0（被归零过）
            ├─ IFileTree.SetTree()                              ← 用新 Model.Files 重建树
            ├─ if selectedNode != nil:
            │   newNodes = GetAllItems()
            │   newIdx = findNewSelectedIdx(prevNodes[0:], newNodes)  ← 从第0个旧节点开始找
            │   if newIdx != -1 && newIdx != 0:
            │       SetSelection(newIdx)
            └─ ClampSelection()
```

### 3.2 两个 SetTree() 的关键区别

| SetTree() 调用时机 | 数据来源 | 光标状态 | 目的 |
|-------------------|---------|---------|------|
| SetStatusFilter 内部 | 旧 `Model.Files` + 新过滤器 | 之后被归零为 0 | 即时响应用户操作 |
| Refresh 中 refreshStateFiles | 新 `Model.Files` + 新过滤器 | 从 0 开始智能匹配 | 用真实数据修正 |

### 3.3 SetSelection(0) 的具体行为

位置：[list_cursor.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/traits/list_cursor.go#L61-L64)

```go
func (self *ListCursor) SetSelection(value int) {
    self.selectedIdx = self.clampValue(value)  // clampValue: value = 0 → 0（如果 length > 0）
    self.CancelRangeSelect()                    // 取消任何范围选择
}

func (self *ListCursor) clampValue(value int) int {
    clampedValue := -1
    length := self.getLength()
    if length > 0 {
        clampedValue = lo.Clamp(value, 0, length-1)
    }
    return clampedValue  // 如果列表为空，返回 -1
}
```

**如果 SetStatusFilter 后列表为空**（配置隐藏未跟踪文件时切换到 DisplayUntracked）：
- `getLength()` 返回 0
- `clampValue(0)` 返回 -1
- `selectedIdx = -1`

**后续 Refresh 中 SetTree() 时**：
- `GetSelected()` 检查 `Len() == 0` → 返回 `nil`
- `selectedNode == nil`，跳过 findNewSelectedIdx
- `ClampSelection()` 会把 -1 clamp 到有效范围（如果新列表有数据）

### 3.4 findNewSelectedIdx 在 prevSelectedLineIdx=0 时的行为

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L121)

```go
newIdx := self.findNewSelectedIdx(prevNodes[prevSelectedLineIdx:], newNodes)
// 即: findNewSelectedIdx(prevNodes[0:], newNodes)
```

`prevNodes[0:]` 是全部旧节点（因为从 0 开始切片）。

findNewSelectedIdx 会遍历所有旧节点（从第0个开始），在新节点中找路径匹配。

**两种场景**：

#### 场景A：用户之前在第 5 行，切换过滤器后归零

- 实际光标位置：第 0 行（被 SetSelection(0) 覆盖了）
- 旧节点列表的第 0 个：旧列表第 0 个文件
- 匹配过程：从旧第 0 个开始查找
- 如果旧第 0 个在新列表中存在 → 光标可能留在第 0 个（如果新索引也是 0）
- 如果旧第 0 个不在新列表中 → 继续找旧第 1 个、旧第 2 个……

**问题**：用户原来选中的第 5 个文件的信息被 SetSelection(0) 丢失了。findNewSelectedIdx 不会去恢复原来第 5 个的位置，因为 `prevSelectedLineIdx` 已经是 0，`prevNodes[0:]` 虽然包含了第 5 个，但只有第 0 个找不到时才会遍历到第 5 个。

#### 场景B：切换前就是第 0 个

- 匹配直接命中，光标保持在 0

### 3.5 为什么过滤切换要归零

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L167-L170)

```go
func (self *FileTreeViewModel) SetStatusFilter(filter FileTreeDisplayFilter) {
    self.IFileTree.SetStatusFilter(filter)
    self.IListCursor.SetSelection(0)
}
```

**归零的原因**：
1. 过滤后列表内容完全改变，原来的索引可能越界或指向完全不同的文件
2. 保持简单可预期的行为：切换过滤后从第 1 项开始
3. 避免光标指向不存在的项（如 DisplayStaged 第 5 项切换到 DisplayUntracked 可能只有 2 项）

### 3.6 归零与后续 Refresh 的交互

时序关键点：

```
SetStatusFilter
    ├─ SetTree()          ← 用旧数据重建
    └─ SetSelection(0)    ← 光标归零（同步完成）

Refresh(ASYNC) 启动
    ↓
[ worker goroutine: 可能几十到几百毫秒 ]
    ↓
refreshStateFiles()
    ├─ Model.Files = newFiles
    └─ SetTree()
        ├─ selectedNode = GetSelected()  ← 取当前光标（已归零）对应节点
        ├─ prevSelectedLineIdx = 0        ← 从 0 开始
        └─ findNewSelectedIdx(prevNodes[0:], newNodes)
```

**如果 SetStatusFilter 和 Refresh 之间列表内容没变（如空列表）**：
- GetSelected() 返回 nil，findNewSelectedIdx 不执行，ClampSelection 处理

**如果 SetStatusFilter 后列表有数据**：
- 旧第 0 个节点（过滤后）可能和新第 0 个节点（重新获取后）是同一个文件
- findNewSelectedIdx 可能快速命中，光标仍在 0
- 或者旧第 0 个不存在于新列表中（如该文件状态变了），继续向下找

### 3.7 非 DisplayUntracked 过滤切换的快速路径

当过滤器切换不涉及 DisplayUntracked（如 DisplayAll → DisplayStaged）：

```go
self.c.PostRefreshUpdate(self.context())
```

此时：
- 不重新获取 git status
- 只有一次 SetTree()：SetStatusFilter 内部的那次
- 光标被归零为 0
- PostRefreshUpdate 直接渲染 + HandleFocus/ FocusLine(false)
- 没有第二次 SetTree() 来修正光标

**所以非 DisplayUntracked 切换后，光标一定在第 0 项。**

### 3.8 DisplayUntracked 切换的特殊性

DisplayUntracked 切换有两次 SetTree()：
1. SetStatusFilter 内部：用旧数据 + DisplayUntracked 过滤（可能数据不完整）
2. Refresh 中：用新数据 + DisplayUntracked 过滤（完整数据）

光标在第 1 次 SetTree 后被归零为 0，第 2 次 SetTree 基于 0 的位置做智能匹配。

**如果旧数据过滤后为空（配置隐藏未跟踪文件）**：
- 第 1 次 SetTree 后 Len() = 0
- SetSelection(0) → clampValue(0) → length=0 → 返回 -1
- selectedIdx = -1
- 第 2 次 SetTree() 中 GetSelected() → 返回 nil
- findNewSelectedIdx 不执行
- ClampSelection()：新列表长度 > 0 时，-1 被 clamp 到 0

**所以最终光标仍然是第 0 项。**

**如果旧数据过滤后有数据**：
- 第 1 次 SetTree 后 Len() > 0
- SetSelection(0) → selectedIdx = 0
- 第 2 次 SetTree() 中 GetSelected() → 返回旧列表第 0 个节点
- findNewSelectedIdx(prevNodes[0:], newNodes) → 从旧第 0 个开始匹配
- 如果旧第 0 个文件（如 "new.txt"）在新列表中也存在 → 可能保持在 0
- 如果不存在，找下一个

---

## 四、三种场景总结对比表

### 4.1 当前视图刷新的滚动行为

| 触发方式 | scrollIntoView | 是否滚动 | 说明 |
|---------|---------------|---------|------|
| PostRefreshUpdate 刷新 | false | 否 | 光标位置更新，视口不动 |
| 切换到视图（键盘） | true | 选中行在视口外则滚动居中 | 完整焦点处理 |
| 搜索跳转 | true | 是 | 强制滚动到匹配行 |

### 4.2 非焦点视图的选中状态更新

| 操作 | 是否执行 | 效果 |
|------|---------|------|
| HandleRender | 是 | 重新渲染所有内容（数据更新） |
| FocusLine(false) | 是 | 更新 v.cy（光标位置）、页脚、范围选择 |
| FocusPoint(scroll=false) | 是 | v.cy 更新，v.oy 不变（不滚动） |
| SetHighlight | 否 | 仅焦点视图调用 |
| HandleRenderToMain | 条件 | 侧面板→主面板关联更新 |

### 4.3 过滤切换对光标定位的影响

| 切换类型 | SetTree 次数 | 光标归零 | 智能匹配 | 最终效果 |
|---------|------------|---------|---------|---------|
| DisplayUntracked ↔ 其他 | 2 次 | 是（第 1 次后） | 是（第 2 次中，从 0 开始） | 通常为 0，除非旧第 0 个匹配到新索引 |
| 其他 ↔ 其他 | 1 次 | 是（仅这次） | 否 | 固定为 0 |

**归零的设计意图**：过滤后列表语义完全改变，原来的索引无意义，从头开始更符合直觉。

**DisplayUntracked 的第二次智能匹配**：弥补 ForceShowUntracked 带来的额外数据获取，但由于已归零，匹配起点是第 0 项而非用户原来的选中项。
