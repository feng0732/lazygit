# 文件状态链路深入分析（续）

本文档深入分析三条关键代码路径：未跟踪过滤切换、重命名解析顺序、刷新后光标更新与界面衔接。

---

## 一、未跟踪过滤切换的完整代码走向

### 1.1 触发入口

位置：[files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/files_controller.go#L967-L1014)

用户按下过滤快捷键 → `handleStatusFilterPressed()` 弹出过滤菜单，包含 5 个选项：

- `DisplayStaged (s)
- `DisplayUnstaged (u)`
- `DisplayTracked (t)
- `DisplayUntracked (T)`
- `DisplayAll (r)`

每个选项点击后调用 `setStatusFiltering(filter)`。

### 1.2 setStatusFiltering 核心逻辑

位置：[files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/files_controller.go#L1035-L1049)

```go
func (self *FilesController) setStatusFiltering(filter filetree.FileTreeDisplayFilter) error {
    previousFilter := self.context().GetStatusFilter()

    // 步骤1：更新过滤器和视图副标题
    self.context().FileTreeViewModel.SetStatusFilter(filter)
    self.c.Contexts().Files.GetView().Subtitle = self.filteringLabel(filter)

    // 步骤2：判断是否需要全量刷新
    if previousFilter != filter && 
       (previousFilter == filetree.DisplayUntracked || filter == filetree.DisplayUntracked) {
        // 切换到/从"未跟踪"模式：需要重新跑 git status
        self.c.Refresh(types.RefreshOptions{
            Scope: []types.RefreshableView{types.FILES}, 
            Mode: types.ASYNC,
        })
    } else {
        // 其他过滤器切换：只需要重新渲染即可
        self.c.PostRefreshUpdate(self.context())
    }
    return nil
}
```

**关键判断逻辑**：只有当过滤器切换涉及 `DisplayUntracked` 时，才需要触发完整的 `git status` 刷新。其他过滤器（如 staged/unstaged/tracked/conflicted）只是对已有的文件列表进行本地过滤即可。

### 1.3 为什么未跟踪过滤器的特殊性

未跟踪过滤器之所以特殊，是因为它会影响 `git status` 命令本身的参数：

位置：[file_tree.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree.go#L115-L117)

```go
func (self *FileTree) ForceShowUntracked() bool {
    return self.filter == DisplayUntracked
}
```

这个值在刷新时被传递给 `FileLoader.GetStatusFiles()`：

位置：[refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L605-L608)

```go
files := self.c.Git().Loaders.FileLoader.
    GetStatusFiles(git_commands.GetStatusFileOptions{
        ForceShowUntracked: self.c.Contexts().Files.ForceShowUntracked(),
    })
```

进而影响 git status 命令的 `--untracked-files` 参数：

位置：[file_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/git_commands/file_loader.go#L41-L48)

```go
untrackedFilesSetting := self.config.GetShowUntrackedFiles()
if opts.ForceShowUntracked || untrackedFilesSetting == "" {
    untrackedFilesSetting = "all"
}
untrackedFilesArg := fmt.Sprintf("--untracked-files=%s", untrackedFilesSetting)
```

**设计原因**：如果用户配置了 `gui.showUntrackedFiles = no（隐藏未跟踪文件），但用户又选择了"只显示未跟踪文件"过滤，这时候需要强制 git status 返回未跟踪文件。

### 1.4 SetStatusFilter 内部流程

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L167-L170)

```go
func (self *FileTreeViewModel) SetStatusFilter(filter FileTreeDisplayFilter) {
    self.IFileTree.SetStatusFilter(filter)
    self.IListCursor.SetSelection(0)  // 光标重置到第 0 行
}
```

位置：[file_tree.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree.go#L123-L126)

```go
func (self *FileTree) SetStatusFilter(filter FileTreeDisplayFilter) {
    self.filter = filter
    self.SetTree()  // 重新构建树
}
```

### 1.5 两种刷新路径对比

| 切换类型 | 是否调用 git status | 是否重建树 | 光标位置 |
|----------|-------------------|---------|---------|
| 非未跟踪 ↔ 非未跟踪 | 否（本地过滤） | 是 | 重置为 0 |
| 未跟踪 ↔ 其他 | 是（重新获取文件列表） | 是 | 重置为 0 |

非未跟踪过滤器之间的切换（如 staged ↔ unstaged）只需要重新构建树（通过 `PostRefreshUpdate 立即重绘，不需要等待 git 命令。

---

## 二、重命名解析顺序与路径处理

### 2.1 git status 输出格式

`git status --porcelain -z` 对于重命名文件的输出格式（用 NUL 分隔）：

```
R100 oldname\x00newname\x00
```

- `R` 表示重命名，后面的数字是相似度百分比
- 第一行：状态码 + 旧文件名
- 第二行：新文件名
- `C` 表示复制，格式相同

### 2.2 解析逻辑

位置：[file_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/git_commands/file_loader.go#L190-L212)

```go
for i := 0; i < len(splitLines); i++ {
    original := splitLines[i]
    
    if len(original) < 3 {
        continue
    }

    status := FileStatus{
        StatusString: original,
        Change:       original[:2],   // 前两位是状态码
        Path:         original[3:], // 从第 3 位起是路径
        PreviousPath: "",
    }

    // 关键：检测到 R 或 C 开头，下一行就是旧路径
    if strings.HasPrefix(status.Change, "R") || strings.HasPrefix(status.Change, "C") {
        status.PreviousPath = splitLines[i+1]
        status.StatusString = fmt.Sprintf("%s %s -> %s", 
            status.Change, status.PreviousPath, status.Path)
        i++  // 跳过下一行（已经被消费了）
    }

    response = append(response, status)
}
```

**注意**：
- 解析是**当前路径（`Path`）存储的是新文件名
- 之前路径（`PreviousPath`）存储的是旧文件名
- `i++ 手动递增，因为重命名消费两条输出行

### 2.3 重命名文件在文件树中的位置

重命名文件在树中以**新路径**为准排序，因为 `Path` 字段是新路径。例如 `file.Path` 是新路径，所以树的路径也会出现在新路径的目录下。

位置：[build_tree.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/build_tree.go#L10-L68)

```go
for _, file := range files {
    splitPath := SplitFileTreePath(file.Path, showRootItem) // 使用 file.Path
    // ... 按新路径构建树节点
}
```

### 2.4 显示时的重命名展示

位置：[files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/presentation/files.go#L302-L327)

```go
func fileNameAtDepth(node *filetree.Node[models.File], depth int, showRootItem bool) string {
    // ...
    name := join(splitName[depth:])

    if node.File != nil && node.File.IsRename() {
        splitPrevName := filetree.SplitFileTreePath(node.File.PreviousPath, showRootItem)

        prevName := node.File.PreviousPath
        // 判断是否在同一目录下
        sameParentDir := len(splitName) == len(splitPrevName) && 
            join(splitName[0:depth]) == join(splitPrevName[0:depth])
        if sameParentDir {
            prevName = join(splitPrevName[depth:])
        }

        return prevName + " → " + name
    }

    return name
}
```

**显示逻辑**：
- 如果重命名在同一目录内：只显示文件名（如 `old.txt → new.txt）
- 如果重命名跨目录：显示完整旧路径 → 新文件名

### 2.5 树排序与路径顺序

文件树的排序是按**新路径**的字母顺序排序：

位置：[node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/node.go#L74-L113)

```go
func NodeSortComparator[T any](sortOrder string, caseSensitive bool) func(a, b *Node[T]) int {
    // ...
    return strCmp(a.path, b.path)  // 按 path 字母顺序
}
```

但在扁平模式下还有额外的分组排序：

位置：[build_tree.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/build_tree.go#L130-L170)

```go
// 扁平模式排序优先级：
// 1. 有合并冲突的文件（最前）
// 2. 已跟踪文件
// 3. 未跟踪文件（最后）
```

### 2.6 光标追踪

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L107-L112)

```go
// 当你暂存了重命名的旧文件，而新文件在折叠的目录中时，自动展开

for _, file := range newFiles {
    if selectedNode != nil && selectedNode.path != "" && 
       file.PreviousPath == selectedNode.path {
        self.ExpandToPath(file.Path)
    }
}
```

这是为了处理一种场景：用户选中了重命名文件（如 `a/old.txt`），当它显示为一个条目 `old.txt → new.txt`），当用户暂存后，重命名可能拆分成两个文件（旧文件删除 + 新文件添加），这时候新文件可能在折叠的目录里，需要自动展开以便用户看到。

---

## 三、刷新后光标更新与界面衔接

### 3.1 SetTree 完整流程

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L103-L128)

```go
func (self *FileTreeViewModel) SetTree() {
    // 步骤1：获取所有新文件（用于重命名展开检测
    newFiles := self.GetAllFiles()
    selectedNode := self.GetSelected()

    // 步骤2：处理重命名文件的目录自动展开
    for _, file := range newFiles {
        if selectedNode != nil && selectedNode.path != "" && 
           file.PreviousPath == selectedNode.path {
            self.ExpandToPath(file.Path)
        }
    }

    // 步骤3：保存旧状态
    prevNodes := self.GetAllItems()
    prevSelectedLineIdx := self.GetSelectedLineIdx()

    // 步骤4：真正重建树
    self.IFileTree.SetTree()

    // 步骤5：尝试找到新的选中位置
    if selectedNode != nil {
        newNodes := self.GetAllItems()
        // 从之前选中的位置开始向下找
        newIdx := self.findNewSelectedIdx(prevNodes[prevSelectedLineIdx:], newNodes)
        if newIdx != -1 && newIdx != prevSelectedLineIdx {
            self.SetSelection(newIdx)
        }
    }

    // 步骤6：确保光标不越界
    self.ClampSelection()
}
```

### 3.2 findNewSelectedIdx 匹配算法

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L137-L165)

```go
func (self *FileTreeViewModel) findNewSelectedIdx(prevNodes []*FileNode, currNodes []*FileNode) int {
    getPaths := func(node *FileNode) []string {
        if node == nil {
            return nil
        }
        if node.File != nil && node.File.IsRename() {
            return node.File.Names()  // 重命名文件返回两个路径
        }
        return []string{node.path}
    }

    // 外层循环：遍历旧节点（从之前选中的位置开始）
    for _, prevNode := range prevNodes {
        selectedPaths := getPaths(prevNode)

        // 内层循环：在新节点中查找匹配
        for idx, node := range currNodes {
            paths := getPaths(node)

            // 特殊情况：之前选中的是重命名文件，现在拆分了
            // 我们要跳到新文件，而不是旧文件
            foundOldFileInRename := prevNode.File != nil && 
                prevNode.File.IsRename() && 
                node.path == prevNode.File.PreviousPath

            // 路径有交集 且 不是"找到了旧文件
            foundNode := utils.StringArraysOverlap(paths, selectedPaths) && 
                !foundOldFileInRename

            if foundNode {
                return idx
            }
        }
    }

    return -1
}
```

**匹配策略**：
1. 从之前选中的节点开始，依次向下遍历旧节点列表
2. 对每个旧节点，在新节点列表中从头查找匹配
3. 找到第一个匹配的就返回其索引
4. 如果找不到当前节点（比如文件被删了），就继续找下一个旧节点
5. 全部找不到返回 -1（此时 ClampSelection 会把光标放到有效位置

**重命名特殊处理**：如果之前选中的是重命名文件，现在它被拆分成两个文件（删除旧文件删除 + 新增新文件），这时候优先匹配新文件，因为新文件在树中的位置和原重命名条目更接近。

### 3.3 从数据刷新到界面显示的完整链路

```
用户操作（如暂存文件）
    ↓
Refresh(ASYNC 模式
    ↓
refreshFilesAndSubmodules()  [worker goroutine
    ↓
refreshStateFiles()
    ├─ 获取互斥锁 RefreshingFilesMutex
    ├─ FileLoader.GetStatusFiles()  ← git status
    ├─ Model.Files = files
    ├─ fileTreeViewModel.SetTree()  ← 重建树 + 光标调整
    └─ 解锁
    ↓
OnUIThread → refreshView()
    ↓
searchHelper.ReApplyFilter(context)
    ↓
PostRefreshUpdate(context)
    ↓
    ┌──────────────────────────────────┐
    │ postRefreshUpdate()              │
    │   ├─ HandleRender()            │ ← 渲染列表内容
    │   ├─ 是当前焦点视图            │
    │   │   └─ HandleFocus()         │ ← 焦点 + 滚动到视图 + 页脚
    │   └─ 不是当前焦点视图        │
    │       ├─ FocusLine(false)     │ ← 只定位光标位置（不滚动）
    │       └─ 可能 HandleRenderToMain() │ ← 刷新主视图
    └──────────────────────────────────┘
    ↓
AfterLayout → ReApplySearch()  ← 重新应用搜索高亮
```

### 3.4 HandleRender 渲染流程

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L110-L128)

```go
func (self *ListContextTrait) HandleRender() {
    self.list.ClampSelection()  // 先确保光标不越界
    
    if self.renderOnlyVisibleLines {
        // 只渲染可见区域（大数据量）
        startIdx, length := self.GetViewTrait().ViewPortYBounds()
        content := self.renderLines(startIdx, startIdx+length)
        self.GetViewTrait().SetViewPortContentAndClearEverythingElse(totalLength, content)
        self.needRerenderVisibleLines = false
    } else {
        // 渲染全部
        content := self.renderLines(-1, -1)
        self.GetViewTrait().SetContent(content)
    }
    
    self.setFooter()  // 设置页脚 "x of y"
}
```

### 3.5 HandleFocus 与 FocusLine

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L35-L75)

```go
func (self *ListContextTrait) FocusLine(scrollIntoView bool) {
    self.Context.FocusLine(scrollIntoView)

    // AfterLayout 中执行，因为需要视图尺寸确定后才能计算滚动位置
    self.c.AfterLayout(func() error {
        oldOrigin, _ := self.GetViewTrait().ViewPortYBounds()

        // 将模型索引转视图索引（考虑非模型项目）
        self.GetViewTrait().FocusPoint(
            self.ModelIndexToViewIndex(self.list.GetSelectedLineIdx()), 
            scrollIntoView)
        
        // ... 搜索位置
        if !inOnSearchSelect {
            self.GetView().SetNearestSearchPosition()
        }

        // 范围选择
        selectRangeIndex, isSelectingRange := self.list.GetRangeStartIdx()
        if isSelectingRange {
            selectRangeIndex = self.ModelIndexToViewIndex(selectRangeIndex)
            self.GetViewTrait().SetRangeSelectStart(selectRangeIndex)
        } else {
            self.GetViewTrait().CancelRangeSelect()
        }

        // 视口刷新（按需）
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

    self.setFooter()
}
```

**为什么用 AfterLayout**：
- 视图的尺寸在布局阶段确定的
- 如果视图大小改变
- 否则如果直接计算会出错
- 所以要等布局完成后才能正确计算滚动位置

### 3.6 两种刷新方式的区别

| 方式 | 触发场景 | 线程 | 是否等 git status | 体验 |
|------|----------|------|-------------|------|
| `PostRefreshUpdate` | 本地状态改变（如过滤器切换、乐观渲染） | UI 线程 | 否 | 即时，快 |
| `Refresh` | 真实数据改变（如暂存、提交） | Worker 线程 | 是 | 准确，慢 |

**乐观渲染组合使用场景**：
- 乐观渲染先用 PostRefreshUpdate 先更新界面（PostRefreshUpdate 用 PostRefreshUpdate 更新界面（乐观更新界面（
- 然后用 Refresh 做真实同步（后台异步）

### 3.7 互斥锁的作用

位置：[refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L545-L551)

```go
func (self *RefreshHelper) refreshFilesAndSubmodules() error {
    self.c.Mutexes().RefreshingFilesMutex.Lock()
    self.c.State().SetIsRefreshingFiles(true)
    defer func() {
        self.c.State().SetIsRefreshingFiles(false)
        self.c.Mutexes().RefreshingFilesMutex.Unlock()
    }()
    // ...
}
```

`RefreshingFilesMutex` 保护的是：
- Model.Files 数据竞争
- 防止多个刷新同时修改文件列表
- 乐观渲染和真实刷新同时修改文件状态

乐观渲染也会获取这个锁：

位置：[files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/files_controller.go#L519-L524)

```go
func (self *FilesController) pressWithLock(selectedNodes []*filetree.FileNode) error {
    self.c.Mutexes().RefreshingFilesMutex.Lock()
    defer self.c.Mutexes().RefreshingFilesMutex.Unlock()
    // ... 乐观渲染修改 Model.Files
}
```

---

## 四、完整时序图：暂存未跟踪文件完整时序

以"暂存一个未跟踪文件"的完整时序：

```
用户按空格
  │
  ├─ pressWithLock()
  │   ├─ 获取 RefreshingFilesMutex
  │   ├─ toggleStaged()
  │   │   └─ optimisticChange()
  │   │       ├─ 找到 Model.Files 中对应的文件
  │   │       ├─ optimisticStage(file)  →  ?? 改为 A
  │   │       └─ PostRefreshUpdate()  ← 立即重绘界面
  │   └─ 执行 git add 命令
  │       └─ StageFiles()
  └─ Refresh(ASYNC, FILES)
      │
      └─ [worker goroutine
          └─ refreshFilesAndSubmodules()
              ├─ 获取 RefreshingFilesMutex（等待乐观渲染完成
              ├─ FileLoader.GetStatusFiles()  ← git status
              ├─ Model.Files = files  ← 用真实状态覆盖
              ├─ SetTree()  ← 重建树 + 调整光标
              └─ 解锁
                  └─ OnUIThread
                      └─ refreshView()
                          ├─ ReApplyFilter
                          └─ PostRefreshUpdate
                              ├─ HandleRender  ← 重新渲染
                              └─ HandleFocus   ← 调整光标和滚动
```

**关键点**：
1. 乐观渲染在获取锁后立即修改内存中的状态并重绘，用户感觉很快
2. 然后后台异步跑真正的 git 命令和 git status
3. 真实刷新也会获取同一个锁，确保不会和乐观渲染同时修改数据
4. 真实状态覆盖乐观状态可能有偏差）
5. 最后在 UI 线程重新渲染和调整光标位置
