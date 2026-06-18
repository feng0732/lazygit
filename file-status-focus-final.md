# 文件状态链路终版聚焦：默认刷新滚动逻辑、所有滚动场景、非焦点视图选中、过滤归零定位

---

## 一、当前视图默认刷新为什么不滚动

### 1.1 代码实证

位置：[view_helpers.go#L133-L137](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/view_helpers.go#L133-L137)

```go
c.HandleRender()

if gui.currentViewName() == c.GetViewName() {
    c.HandleFocus(types.OnFocusOpts{})  // ← 空结构体
}
```

`types.OnFocusOpts{}` 是零值初始化：

位置：[context.go#L229-L233](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/types/context.go#L229-L233)

```go
type OnFocusOpts struct {
    ClickedWindowName       string
    ClickedViewLineIdx      int
    ScrollSelectionIntoView bool  // ← bool 零值为 false
}
```

`ScrollSelectionIntoView` 的零值是 `false`。

### 1.2 传递链路

```
PostRefreshUpdate(currentContext)
    ↓
HandleFocus(OnFocusOpts{})  →  ScrollSelectionIntoView = false
    ↓
FocusLine(false)            →  scrollIntoView = false
    ↓
AfterLayout:
    FocusPoint(viewIdx, false)
        ↓
        scrollIntoView == false → 不计算新 origin → v.oy 不变 → 不滚动
        但 v.cy = cy - v.oy 更新 → 光标相对位置保持
```

### 1.3 gocui FocusPoint 的精确行为

位置：[view.go#L374-L391](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gocui/view.go#L374-L391)

```go
func (v *View) FocusPoint(cx int, cy int, scrollIntoView bool) {
    // ...
    if scrollIntoView {
        height := v.InnerHeight()
        v.oy = calculateNewOrigin(cy, v.oy, lineCount, height)  // ← 仅此处调整滚动
    }

    v.cx = cx
    v.cy = cy - v.oy  // ← 光标位置始终更新，无论是否滚动
}
```

**两个独立变量**：
- `v.oy`：视口 Y 原点（滚动了多少行）→ 仅 `scrollIntoView=true` 时可能变化
- `v.cy`：光标在视口内的相对 Y 位置 → 始终更新

**默认刷新时**：`scrollIntoView=false`
- `v.oy` 不变 → 视口不滚动
- `v.cy` 更新 → 如果选中行仍在可视范围内，高亮位置正确跟随模型索引
- 如果选中行已滚出视口外，用户看不到高亮，但 `SelectedLineIdx()` 仍返回正确值

### 1.4 calculateNewOrigin 的滚动策略

位置：[view.go#L401-L423](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gocui/view.go#L401-L423)

```go
func calculateNewOrigin(selectedLine int, oldOrigin int, lineCount int, viewHeight int) int {
    if viewHeight >= lineCount {
        return 0                              // 全部可见，归零
    } else if selectedLine < oldOrigin || selectedLine >= oldOrigin+viewHeight {
        newOrigin := selectedLine - viewHeight/2  // 居中
        // 边界夹紧 ...
        return newOrigin
    }
    return oldOrigin                        // 已在视口内，不动
}
```

仅当选中行在视口外时才滚动，滚动后选中行位于视口中间。

---

## 二、所有触发滚动的场景

按代码精确核准，`scrollIntoView=true` 只出现在以下场景：

### 2.1 用户在列表内按上下键移动光标

位置：[list_controller.go#L108-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/list_controller.go#L108-L135)

```go
if cursorMoved || rangeBefore != rangeAfter {
    // 光标确实移动了
    self.context.HandleFocus(types.OnFocusOpts{ScrollSelectionIntoView: true})
} else {
    // 光标没动（如在列表顶端按上），但鼠标可能把选中行滚出了视口
    self.context.FocusLine(true)
}
```

- 光标移动：`HandleFocus(ScrollSelectionIntoView: true)` → 滚动 + 高亮 + 主面板更新
- 光标未动：`FocusLine(true)` → 只确保选中行可见（鼠标滚出视口时）

### 2.2 提交上下移动

位置：[local_commits_controller.go#L767](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/local_commits_controller.go#L767)

```go
self.context().HandleFocus(types.OnFocusOpts{ScrollSelectionIntoView: true})
```

提交上移/下移、rebase todo 移动后都显式传 `true`。

### 2.3 stash 重命名后

位置：[stash_controller.go#L210](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/stash_controller.go#L210)

```go
self.context().SetSelection(0) // 选中重命名后的 stash
self.context().FocusLine(true)
```

### 2.4 进入子提交视图（sub_commits）

位置：[sub_commits_helper.go#L69](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/helpers/sub_commits_helper.go#L69)

```go
self.c.PostRefreshUpdate(self.c.Contexts().SubCommits)
subCommitsContext.FocusLine(true)  // ← 在 Push 之前先滚动好
self.c.Context().Push(self.c.Contexts().SubCommits, types.OnFocusOpts{})
```

### 2.5 fixup/cherry-pick 操作后跳转

位置：[fixup_helper.go#L144](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/helpers/fixup_helper.go#L144)
位置：[cherry_pick_helper.go#L110](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/helpers/cherry_pick_helper.go#L110)

```go
self.c.Contexts().LocalCommits.SetSelection(index)
self.c.Contexts().LocalCommits.FocusLine(true)
```

### 2.6 场景汇总表

| 场景 | scrollIntoView | 是否滚动 | 代码位置 |
|------|---------------|---------|---------|
| PostRefreshUpdate 刷新当前视图 | false | 否 | view_helpers.go:136 |
| PostRefreshUpdate 刷新非当前视图 | false | 否 | view_helpers.go:143 |
| 鼠标点击/Tab 切换到 Files 视图 | false | 否 | context.go:55/70 |
| 键盘上下移动光标 | true | 选中行在视口外则滚动 | list_controller.go:129 |
| 在列表顶端按上（光标不动） | true | 鼠标滚出时滚动 | list_controller.go:134 |
| 翻页（PageUp/PageDown） | true | 滚动 | 同 list_controller 路径 |
| 提交上下移动后 | true | 滚动 | local_commits_controller.go |
| stash 重命名后 | true | 滚动 | stash_controller.go |
| 进入子提交视图 | true | 滚动 | sub_commits_helper.go |
| fixup/cherry-pick 后跳转 | true | 滚动 | fixup_helper.go 等 |

**核心设计原则**：
- **被动刷新（PostRefreshUpdate、上下文切换）不滚动**：避免干扰用户正在查看的位置
- **主动导航（上下键、跳转）滚动**：确保选中行可见

---

## 三、非当前视图如何只更新选中状态

### 3.1 代码分支

位置：[view_helpers.go#L135-L163](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/view_helpers.go#L135-L163)

```go
if gui.currentViewName() == c.GetViewName() {
    c.HandleFocus(types.OnFocusOpts{})
} else {
    // 注释说明：
    // 非焦点视图必须单独调用 FocusLine
    // 目的 1：确保非激活选中状态被正确绘制
    // 目的 2：确保集成测试能看到最新选中状态
    c.FocusLine(false)  // ← scrollIntoView = false

    // 如果当前焦点在主面板，侧面板选中项变了 → 更新主面板 diff
    currentCtx := gui.State.ContextMgr.Current()
    if currentCtx 是 normal main/secondary {
        if !当前主面板正在搜索 {
            sidePanelContext := NextInStack(currentCtx)
            if sidePanelContext != nil && sidePanelContext.GetKey() == c.GetKey() {
                sidePanelContext.HandleRenderToMain()  // 刷新主面板
            }
        }
    } else if c 是当前静态上下文 {
        // 有弹窗，刷新弹窗后面的主视图
        c.HandleRenderToMain()
    }
}
```

### 3.2 FocusLine(false) 的精确行为

位置：[list_context_trait.go#L35-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L35-L75)

```go
func (self *ListContextTrait) FocusLine(scrollIntoView bool) {
    self.c.AfterLayout(func() error {
        // ① 更新光标：模型索引 → 视图索引
        self.GetViewTrait().FocusPoint(
            self.ModelIndexToViewIndex(self.list.GetSelectedLineIdx()),
            scrollIntoView)  // ← false

        // ② 更新搜索位置高亮
        if !inOnSearchSelect {
            self.GetView().SetNearestSearchPosition()
        }

        // ③ 更新范围选择起点
        if isSelectingRange {
            self.GetViewTrait().SetRangeSelectStart(selectRangeIndex)
        } else {
            self.GetViewTrait().CancelRangeSelect()
        }

        // ④ 按需刷新视口内容（仅 renderOnlyVisibleLines 模式）
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

    // ⑤ 更新页脚 "x of y"
    self.setFooter()
}
```

### 3.3 FocusPoint(false) 内部

```go
func (v *View) FocusPoint(cx int, cy int, scrollIntoView bool) {
    // scrollIntoView = false
    // → 跳过 v.oy = calculateNewOrigin(...)
    // → v.oy 不变（不滚动）

    v.cx = cx
    v.cy = cy - v.oy  // 光标相对位置更新
}
```

### 3.4 非焦点视图不会做的事

1. **不调用 HandleFocus** → 不调用 `SetHighlight(true)` → 选中行可能不显示高亮背景色
2. **不调用 HandleFocusLost 中的 `SetOriginX(0)`** → 水平滚动位置保持
3. **scrollIntoView=false** → 不调整垂直滚动

### 3.5 非焦点视图做了的事

| 操作 | 效果 |
|------|------|
| HandleRender | 重新渲染所有行（数据更新） |
| FocusPoint(cy, false) | 更新 v.cy（光标相对位置） |
| SetNearestSearchPosition | 更新搜索高亮位置 |
| SetRangeSelectStart/Cancel | 更新范围选择状态 |
| setFooter | 页脚 "x of y" 正确显示 |
| HandleRenderToMain | 条件性更新主面板 diff |

### 3.6 为什么需要 FocusLine(false)

注释给出两个理由：
1. **非激活选中状态绘制**：即使视图不是焦点，也需要选中行的某些视觉状态正确（如集成测试、非焦点视图高亮色 `HighlightInactive`）
2. **集成测试**：`SelectedLineIdx()` 通过 `v.cy + v.oy` 计算，`v.cy` 不更新则测试获取不到正确的选中行

位置：[view.go#L1648-L1651](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gocui/view.go#L1648-L1651)

```go
func (v *View) SelectedLineIdx() int {
    _, seletedLineIdx := v.SelectedPoint()
    return seletedLineIdx
}

func (v *View) SelectedPoint() (int, int) {
    cx, cy := v.Cursor()  // v.cx, v.cy
    ox, oy := v.Origin()  // v.ox, v.oy
    return cx + ox, cy + oy  // ← 依赖 v.cy 正确
}
```

---

## 四、过滤切换归零后重取状态对光标定位的影响

### 4.1 精确时序（以 DisplayAll → DisplayUntracked 为例）

```
用户选择"只显示未跟踪"
    ↓
setStatusFiltering(DisplayUntracked)
    │
    ├─ 步骤 A：FileTreeViewModel.SetStatusFilter(DisplayUntracked)
    │   │
    │   ├─ A1：IFileTree.SetStatusFilter(DisplayUntracked)
    │   │   ├─ self.filter = DisplayUntracked
    │   │   └─ self.SetTree()                  ← 第 1 次 SetTree
    │   │       └─ getFilesForDisplay()
    │   │           └─ FilterFiles(DisplayUntracked, 旧 Model.Files)
    │   │           （如果配置隐藏未跟踪，此处可能返回空列表）
    │   │
    │   └─ A2：IListCursor.SetSelection(0)     ← 光标归零！
    │       ├─ selectedIdx = clampValue(0)
    │       │   ├─ 如果第 1 次 SetTree 后有数据 → 0
    │       │   └─ 如果第 1 次 SetTree 后空列表 → -1
    │       └─ 取消范围选择
    │
    ├─ 步骤 B：设置视图副标题 "Untracked files"
    │
    └─ 步骤 C：previousFilter != DisplayUntracked
        ↓ 条件成立
        Refresh(ASYNC, [FILES])
            │
            ├─ [worker goroutine]
            │   │
            │   └─ refreshStateFiles()
            │       ├─ GetStatusFiles(ForceShowUntracked=true) ← git status
            │       ├─ Model.Files = newFiles                  ← 新数据
            │       └─ fileTreeViewModel.SetTree()             ← 第 2 次 SetTree
            │           │
            │           ├─ D1：newFiles = GetAllFiles()
            │           ├─ D2：selectedNode = GetSelected()
            │           │   ├─ 如果 A2 后 selectedIdx=-1 → Len()=0 → nil
            │           │   └─ 如果 A2 后 selectedIdx=0  → Get(0) → 第 0 个节点
            │           │
            │           ├─ D3：处理重命名目录展开
            │           ├─ D4：prevNodes = GetAllItems()          ← 当前树的节点
            │           ├─ D5：prevSelectedLineIdx = GetSelectedLineIdx() ← 0（或 -1）
            │           │
            │           ├─ D6：IFileTree.SetTree()                ← 用新数据重建
            │           │
            │           ├─ D7：if selectedNode != nil:
            │           │   newNodes = GetAllItems()
            │           │   newIdx = findNewSelectedIdx(prevNodes[0:], newNodes)
            │           │   if newIdx != -1 && newIdx != 0:
            │           │       SetSelection(newIdx)
            │           │
            │           └─ D8：ClampSelection()                    ← 确保不越界
            │
            └─ [UI 线程]
                └─ refreshView(Files)
                    ├─ ReApplyFilter
                    └─ PostRefreshUpdate
                        ├─ HandleRender
                        └─ 当前焦点 → HandleFocus(OnFocusOpts{}) → FocusLine(false)
                           非焦点 → FocusLine(false)
```

### 4.2 SetSelection(0) 的内部

位置：[list_cursor.go#L61-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/traits/list_cursor.go#L61-L64)

```go
func (self *ListCursor) SetSelection(value int) {
    self.selectedIdx = self.clampValue(value)
    self.CancelRangeSelect()
}

func (self *ListCursor) clampValue(value int) int {
    clampedValue := -1
    length := self.getLength()
    if length > 0 {
        clampedValue = lo.Clamp(value, 0, length-1)
    }
    return clampedValue
}
```

**如果第 1 次 SetTree（用旧数据过滤）后列表为空**：
- `getLength() = 0`
- `clampValue(0) = -1`
- `selectedIdx = -1`

**如果第 1 次 SetTree 后列表有数据**：
- `clampValue(0) = 0`
- `selectedIdx = 0`

### 4.3 第 2 次 SetTree 中 selectedNode 的值

位置：[file_tree_view_model.go#L43-L49](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L43-L49)

```go
func (self *FileTreeViewModel) GetSelected() *FileNode {
    if self.Len() == 0 {
        return nil
    }
    return self.Get(self.GetSelectedLineIdx())
}
```

**情况 1：selectedIdx = -1** → `Len()` 内部调用 `ClampSelection()`，如果此时第 1 次 SetTree 后列表仍为空 → `Len() = 0` → 返回 `nil`

**情况 2：selectedIdx = 0** → 返回旧列表（第 1 次 SetTree 后）的第 0 个节点

### 4.4 findNewSelectedIdx 在 selectedIdx=0 时的行为

```go
newIdx := self.findNewSelectedIdx(prevNodes[prevSelectedLineIdx:], newNodes)
// prevSelectedLineIdx = 0
// 即 findNewSelectedIdx(prevNodes[0:], newNodes)
```

`prevNodes[0:]` 是从第 0 项开始的完整旧节点列表。

匹配逻辑：
1. 先看旧第 0 个节点（即 SetSelection(0) 后选中的节点）在新列表中是否存在
2. 如果存在 → 光标跳到对应位置（通常也是 0）
3. 如果不存在 → 依次看旧第 1 个、旧第 2 个……
4. 全部找不到返回 -1

**由于用户原来的选中项（如第 5 个）已经被 SetSelection(0) 覆盖了**，findNewSelectedIdx 不会特意去恢复原来第 5 个的位置。只有当旧第 0 个找不到时，才会遍历到后面的节点。

### 4.5 两种切换方式对比

| 项目 | 非 DisplayUntracked 切换（如 All → Staged） | DisplayUntracked 切换 |
|------|------------------------------------------|----------------------|
| SetTree 次数 | 1 次（SetStatusFilter 内） | 2 次（+ Refresh 中） |
| 光标归零 | 是 | 是 |
| 智能匹配 findNewSelectedIdx | 否 | 是（第 2 次 SetTree） |
| 是否重新获取 git status | 否 | 是 |
| 最终光标位置 | 固定为 0 | 通常为 0，智能匹配失败时 Clamp 到 0 |
| 刷新方式 | PostRefreshUpdate（快） | Refresh(ASYNC)（慢） |

### 4.6 为什么过滤切换要归零

位置：[file_tree_view_model.go#L167-L170](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L167-L170)

```go
func (self *FileTreeViewModel) SetStatusFilter(filter FileTreeDisplayFilter) {
    self.IFileTree.SetStatusFilter(filter)
    self.IListCursor.SetSelection(0)
}
```

**归零的三个理由**：
1. **过滤后列表语义完全改变**：原来选中的第 5 个文件（如已修改的 `foo.c`）在 DisplayUntracked 过滤下可能根本不存在，原来的索引无意义
2. **越界保护**：DisplayStaged 下第 5 项，切换到 DisplayUntracked 可能只有 2 项，原索引会越界
3. **可预期行为**：用户切换过滤器后，从第 1 项开始浏览是最直觉的体验

### 4.7 DisplayUntracked 智能匹配的实际效果

由于光标已被归零为 0，第 2 次 SetTree 的智能匹配实际上是：
- **以旧列表第 0 个节点为起点**在新列表中查找
- 而不是以用户切换前选中的节点为起点

**典型场景**：
- 切换前：DisplayAll，选中第 5 个（`e.txt`，已修改）
- 切换到 DisplayUntracked
- 第 1 次 SetTree 后旧数据过滤结果只有 `new_a.txt`（假设旧数据有这个未跟踪文件）
- SetSelection(0) → 选中 `new_a.txt`
- 第 2 次 SetTree 获取新数据，DisplayUntracked 有更多未跟踪文件
- findNewSelectedIdx 以 `new_a.txt` 为起点查找
- 如果新列表中也有 `new_a.txt` 且在位置 0 → 光标仍在 0

**用户原来选中的 `e.txt` 不会被恢复**，因为切换过滤器意味着用户想看不同类别的文件，原来的选中项在新类别中可能不存在。

---

## 五、总结速查表

### 5.1 滚动行为

| 触发方式 | ScrollSelectionIntoView | 是否滚动 |
|---------|------------------------|---------|
| 后台自动刷新（当前在 Files） | false | 否 |
| PostRefreshUpdate（当前焦点） | false | 否 |
| PostRefreshUpdate（非焦点） | false（FocusLine(false)） | 否 |
| 鼠标点击/Tab 切换到 Files | false（Push OnFocusOpts{}） | 否 |
| 键盘上下/翻页移动光标 | true | 选中行在视口外则滚动居中 |
| 在列表顶端按上（光标不动） | true（FocusLine(true)） | 鼠标滚出时滚动 |
| 提交上下移、stash 重命名等 | true | 滚动 |

**设计原则**：被动刷新不滚动（不打扰用户），主动导航才滚动（确保可见）。

### 5.2 非焦点视图选中更新

| 操作 | 执行 | 效果 |
|------|------|------|
| HandleRender | ✅ | 所有行重新渲染（数据最新） |
| FocusPoint(cy, false) | ✅ | v.cy 更新，v.oy 不变（不滚动） |
| SetNearestSearchPosition | ✅ | 搜索高亮正确 |
| 范围选择状态 | ✅ | 正确更新 |
| 页脚 "x of y" | ✅ | 正确显示 |
| HandleRenderToMain | 条件 | 侧面板选中变化→主面板更新 |
| SetHighlight(true) | ❌ | 仅焦点视图调用 |
| v.oy 更新（滚动） | ❌ | scrollIntoView=false |

### 5.3 过滤切换对光标影响

| 切换类型 | 归零 | 智能匹配 | 最终光标 |
|---------|------|---------|---------|
| DisplayUntracked ↔ 其他 | 是 | 是（但从 0 开始匹配） | 0（几乎总是） |
| 其他 ↔ 其他 | 是 | 否 | 0 |
