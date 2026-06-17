# 文件状态链路终核：重命名路径顺序、未跟踪状态重取、刷新后光标与渲染衔接

---

## 一、重命名状态输出：新旧路径的精确顺序

### 1.1 git status --porcelain -z 对重命名的输出格式

git 官方文档规定，`git status --porcelain -z` 对重命名/复制的输出为：

```
XY NEW_PATH\x00OLD_PATH\x00
```

**关键点**：第一行包含状态码和新路径，第二行（紧随 NUL 之后）是旧路径。

这与直觉可能相反：**新路径在前，旧路径在后**。

### 1.2 测试用例实证

位置：[file_loader_test.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/git_commands/file_loader_test.go#L136-L171)

```go
// 模拟 git status 输出：
// "R  after1.txt\x00before1.txt\x00RM after2.txt\x00before2.txt"
runner: oscommands.NewFakeRunner(t).
    ExpectGitArgs([]string{"status", "--untracked-files=yes", "--porcelain", "-z", "--find-renames=50%"},
        "R  after1.txt\x00before1.txt\x00RM after2.txt\x00before2.txt",
        nil,
    ),

// 期望解析结果：
expectedFiles: []*models.File{
    {
        Path:         "after1.txt",      // ← 新路径
        PreviousPath: "before1.txt",     // ← 旧路径
        DisplayString: "R  before1.txt -> after1.txt",
        ShortStatus:   "R ",
    },
    {
        Path:         "after2.txt",
        PreviousPath: "before2.txt",
        DisplayString: "RM before2.txt -> after2.txt",
        ShortStatus:   "RM",
    },
},
```

**确认**：git 输出 `R  after1.txt` 中 `after1.txt` 是新路径，下一个 NUL 段 `before1.txt` 是旧路径。

### 1.3 解析代码逐行核准

位置：[file_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/git_commands/file_loader.go#L190-L212)

```go
for i := 0; i < len(splitLines); i++ {
    original := splitLines[i]

    // 每段至少 3 字符："XY path" → 2字符状态码 + 1空格 + 路径
    if len(original) < 3 {
        continue
    }

    status := FileStatus{
        StatusString: original,
        Change:       original[:2],    // 取前两位：如 "R ", "RM"
        Path:         original[3:],    // 取第4位起：这是新路径
        PreviousPath: "",
    }

    if strings.HasPrefix(status.Change, "R") || strings.HasPrefix(status.Change, "C") {
        // 重命名/复制：下一段是旧路径
        status.PreviousPath = splitLines[i+1]
        // 拼接显示字符串：旧路径 → 新路径
        status.StatusString = fmt.Sprintf("%s %s -> %s", 
            status.Change, status.PreviousPath, status.Path)
        i++ // 跳过已消费的旧路径段
    }

    response = append(response, status)
}
```

**精确结论**：

| 字段 | 来源 | 含义 |
|------|------|------|
| `status.Change` | `original[:2]` | 状态码，如 `R `、`RM` |
| `status.Path` | `original[3:]` | **新路径**（after） |
| `status.PreviousPath` | `splitLines[i+1]` | **旧路径**（before） |
| `status.StatusString` | 拼接 | `"RM before -> after"` |

### 1.4 传递到 models.File

位置：[file_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/git_commands/file_loader.go#L70-L74)

```go
file := &models.File{
    Path:          status.Path,         // 新路径
    PreviousPath:  status.PreviousPath, // 旧路径
    DisplayString: status.StatusString, // "R  before -> after"
}
```

### 1.5 File.Names() 的精确语义

位置：[file.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/models/file.go#L44-L51)

```go
// Names returns an array containing just the filename, 
// or in the case of a rename, the after filename and the before filename
func (f *File) Names() []string {
    result := []string{f.Path}            // [0] = 新路径
    if f.PreviousPath != "" {
        result = append(result, f.PreviousPath)  // [1] = 旧路径
    }
    return result
}
```

**Names() 返回 [新路径, 旧路径]**，用于 findNewSelectedIdx 的路径匹配。

### 1.6 文件树中的位置

重命名文件在树中以 `file.Path`（新路径）作为节点路径。树节点和排序都基于新路径。

### 1.7 显示时的箭头方向

位置：[files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/presentation/files.go#L302-L327)

```go
if node.File != nil && node.File.IsRename() {
    // ...
    return prevName + " → " + name  // 旧路径 → 新路径
}
```

显示为 `旧路径 → 新路径`，与 DisplayString 一致。

---

## 二、只显示未跟踪文件时的状态重取

### 2.1 完整调用链

用户选择"只显示未跟踪文件"过滤后，代码执行路径如下：

```
handleStatusFilterPressed()  → 选择 DisplayUntracked
    ↓
setStatusFiltering(DisplayUntracked)
    ↓
步骤1: FileTreeViewModel.SetStatusFilter(DisplayUntracked)
    ├─ IFileTree.SetStatusFilter(DisplayUntracked)
    │   ├─ self.filter = DisplayUntracked     ← 设置过滤器
    │   └─ self.SetTree()                      ← 用新过滤器重建树
    │       └─ getFilesForDisplay()            ← 此时用旧 Model.Files 过滤
    │           └─ FilterFiles( !(Tracked || HasStagedChanges) )
    └─ IListCursor.SetSelection(0)             ← 光标归零
    ↓
步骤2: self.c.Contexts().Files.GetView().Subtitle = "Untracked files"
    ↓
步骤3: 判断是否涉及 DisplayUntracked 切换
    ↓  previousFilter != DisplayUntracked, filter == DisplayUntracked
    ↓  条件成立！走 Refresh 路径
    ↓
Refresh(Scope: [FILES], Mode: ASYNC)
    ↓
[worker goroutine]
    ↓
refreshFilesAndSubmodules()
    ├─ RefreshingFilesMutex.Lock()
    ├─ refreshStateSubmoduleConfigs()
    ├─ refreshStateFiles()
    │   ├─ FileLoader.GetStatusFiles(GetStatusFileOptions{
    │   │       ForceShowUntracked: 
    │   │           self.c.Contexts().Files.ForceShowUntracked()
    │   │   })
    │   │       ↓
    │   │   ForceShowUntracked() = true  （因为 filter == DisplayUntracked）
    │   │       ↓
    │   │   FileLoader.GetStatusFiles()
    │   │       ├─ untrackedFilesSetting = config.GetShowUntrackedFiles()
    │   │       ├─ ForceShowUntracked == true → untrackedFilesSetting = "all"
    │   │       ├─ git status --untracked-files=all --porcelain -z
    │   │       └─ 返回包含未跟踪文件的完整文件列表
    │   │
    │   ├─ Model.Files = files            ← 用真实数据替换
    │   ├─ fileTreeViewModel.SetTree()    ← 重建树 + 光标调整
    │   └─ fileTreeViewModel.RWMutex.Unlock()
    │
    └─ OnUIThread
        └─ refreshView(Files)
            ├─ ReApplyFilter(context)      ← 重新应用文本过滤
            ├─ PostRefreshUpdate(context)  ← 渲染 + 焦点
            │   ├─ HandleRender()          ← ClampSelection + renderLines + SetContent
            │   └─ HandleFocus()           ← FocusLine + AfterLayout
            │       └─ AfterLayout
            │           ├─ FocusPoint(选中行) ← 滚动到选中位置
            │           └─ setFooter()          ← "1 of N"
            └─ AfterLayout
                └─ ReApplySearch(context)  ← 重新应用搜索高亮
```

### 2.2 为什么需要全量重取

**核心问题**：如果用户配置了 `gui.showUntrackedFiles = "no"`，日常的 `git status` 不会返回未跟踪文件。此时 Model.Files 中根本没有未跟踪文件，本地过滤 `!(Tracked || HasStagedChanges)` 只能从已有数据中筛选，结果是空列表。

**解决方式**：
1. `ForceShowUntracked() == true` 使 `git status --untracked-files=all`
2. 返回包含未跟踪文件的完整列表
3. `Model.Files` 被替换为新的完整列表
4. 重建树时 `getFilesForDisplay()` 的 `DisplayUntracked` 过滤器才能找到未跟踪文件

### 2.3 切换离开 DisplayUntracked 时也需要重取

```go
if previousFilter != filter && 
   (previousFilter == filetree.DisplayUntracked || filter == filetree.DisplayUntracked) {
```

当从 DisplayUntracked 切换到其他过滤器时，也需要重取。因为如果用户配置了隐藏未跟踪文件，此时 Model.Files 中可能包含了之前强制获取的未跟踪文件，这些文件在非 DisplayUntracked 过滤下是多余的，但更重要的是——其他过滤模式下不需要 ForceShowUntracked，需要让 git status 回到正常模式获取数据。

不过实际上 git status 的结果是相同的（只是 `--untracked-files` 参数不同），重取的核心目的是确保 ForceShowUntracked 从 true 变回 false 时，后续的后台自动刷新不会一直强制获取未跟踪文件。

### 2.4 非未跟踪过滤器切换的快速路径

当过滤器切换不涉及 DisplayUntracked 时（如 staged → unstaged），走快速路径：

```go
self.c.PostRefreshUpdate(self.context())
```

此时：
- 不调用 git status
- 只重新渲染现有数据
- SetStatusFilter 内部已调用了 SetTree() 重建了树结构
- PostRefreshUpdate 直接进入 HandleRender + HandleFocus

### 2.5 注意：SetStatusFilter 中 SetTree() 被调用了两次

当涉及 DisplayUntracked 切换时，SetTree() 实际被调用了两次：

1. **第一次**：在 `FileTreeViewModel.SetStatusFilter()` 中，通过 `IFileTree.SetStatusFilter()` → `self.SetTree()`。此时用的是旧的 `Model.Files`，如果配置隐藏了未跟踪文件，过滤结果可能为空。

2. **第二次**：在 `refreshStateFiles()` 中，获取新的文件列表后调用 `fileTreeViewModel.SetTree()`。此时用的是新的 `Model.Files`，包含了未跟踪文件，过滤结果正确。

第一次调用虽然结果可能不完整，但它是同步的，给用户一个即时的界面响应。第二次调用会修正数据。

---

## 三、刷新后光标定位和界面渲染衔接

### 3.1 光标定位的三层机制

刷新后的光标定位涉及三层：

**第一层：FileTreeViewModel.SetTree() 中的智能定位**

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L103-L128)

```go
func (self *FileTreeViewModel) SetTree() {
    newFiles := self.GetAllFiles()
    selectedNode := self.GetSelected()      // ① 保存当前选中节点

    // ② 处理重命名导致的目录展开
    for _, file := range newFiles {
        if selectedNode != nil && selectedNode.path != "" && 
           file.PreviousPath == selectedNode.path {
            self.ExpandToPath(file.Path)    // 新文件可能在折叠目录中，展开它
        }
    }

    prevNodes := self.GetAllItems()
    prevSelectedLineIdx := self.GetSelectedLineIdx()

    self.IFileTree.SetTree()               // ③ 重建树

    if selectedNode != nil {
        newNodes := self.GetAllItems()
        newIdx := self.findNewSelectedIdx(  // ④ 在新树中查找旧选中节点
            prevNodes[prevSelectedLineIdx:], newNodes)
        if newIdx != -1 && newIdx != prevSelectedLineIdx {
            self.SetSelection(newIdx)       // ⑤ 移动光标到新位置
        }
    }

    self.ClampSelection()                   // ⑥ 确保光标不越界
}
```

**第二层：HandleRender() 中的 ClampSelection**

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L110-L128)

```go
func (self *ListContextTrait) HandleRender() {
    self.list.ClampSelection()   // ← 再次确保光标不越界
    // ... 渲染内容
}
```

这里调用了第二次 `ClampSelection`，是一个双重保障。

**第三层：FocusLine() 中的视觉定位**

位置：[list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/context/list_context_trait.go#L35-L75)

```go
func (self *ListContextTrait) FocusLine(scrollIntoView bool) {
    self.c.AfterLayout(func() error {
        // 模型索引 → 视图索引（考虑非模型项如 section headers）
        viewIdx := self.ModelIndexToViewIndex(self.list.GetSelectedLineIdx())
        self.GetViewTrait().FocusPoint(viewIdx, scrollIntoView)
        // ... 搜索位置、范围选择等
        return nil
    })
    self.setFooter()   // "x of y"
}
```

FocusPoint 负责滚动视图使选中行可见。它放在 AfterLayout 中，因为需要先完成视图尺寸计算。

### 3.2 findNewSelectedIdx 的精确匹配逻辑

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go#L137-L165)

```go
func (self *FileTreeViewModel) findNewSelectedIdx(prevNodes []*FileNode, currNodes []*FileNode) int {
    getPaths := func(node *FileNode) []string {
        if node == nil { return nil }
        if node.File != nil && node.File.IsRename() {
            return node.File.Names()  // [新路径, 旧路径]
        }
        return []string{node.path}
    }

    // 外层：从之前选中位置开始，依次遍历旧节点
    for _, prevNode := range prevNodes {
        selectedPaths := getPaths(prevNode)

        // 内层：从头遍历新节点
        for idx, node := range currNodes {
            paths := getPaths(node)

            foundOldFileInRename := prevNode.File != nil && 
                prevNode.File.IsRename() && 
                node.path == prevNode.File.PreviousPath

            foundNode := utils.StringArraysOverlap(paths, selectedPaths) && 
                !foundOldFileInRename

            if foundNode { return idx }
        }
    }
    return -1
}
```

**场景分析**：

#### 场景A：普通文件仍在

- 旧节点 path = `"a.txt"`
- 新节点 path = `"a.txt"`
- `getPaths` → `["a.txt"]` 和 `["a.txt"]`
- `StringArraysOverlap` → true
- 找到匹配，返回 idx

#### 场景B：重命名文件仍在

- 旧节点：path = `"new.txt"`, PreviousPath = `"old.txt"`
- `getPaths(prevNode)` → `["new.txt", "old.txt"]`
- 新节点：path = `"new.txt"`, PreviousPath = `"old.txt"`
- `getPaths(node)` → `["new.txt", "old.txt"]`
- `StringArraysOverlap` → true
- `foundOldFileInRename` = prevNode.IsRename() && node.path == prevNode.PreviousPath
  = `"new.txt" == "old.txt"` = false
- 找到匹配，返回 idx

#### 场景C：重命名文件拆分（暂存了重命名的旧文件部分）

- 旧节点（重命名）：path = `"new.txt"`, PreviousPath = `"old.txt"`
- 新节点列表中有两个文件：
  - 节点1：path = `"old.txt"`（删除状态 `D `，来自旧文件的暂存）
  - 节点2：path = `"new.txt"`（新文件 `??` 或 `A `）

遍历新节点时：
- 检查节点1：path = `"old.txt"`
  - `getPaths(node1)` → `["old.txt"]`
  - `StringArraysOverlap(["old.txt"], ["new.txt", "old.txt"])` → true
  - 但 `foundOldFileInRename` = `"old.txt" == "old.txt"` → true！
  - 所以 `foundNode = true && !true = false` → 跳过
- 检查节点2：path = `"new.txt"`
  - `getPaths(node2)` → `["new.txt"]`
  - `StringArraysOverlap(["new.txt"], ["new.txt", "old.txt"])` → true
  - `foundOldFileInRename` = prevNode.IsRename() && `"new.txt" == "old.txt"` → false
  - `foundNode = true` → 返回 idx

**关键设计**：`foundOldFileInRename` 确保光标跳到**新路径**而非旧路径，因为新路径在列表中的位置更接近原来的重命名条目。

#### 场景D：文件被删除

- 旧节点 path = `"deleted.txt"`
- 新节点列表中没有匹配项
- `findNewSelectedIdx` 继续遍历下一个旧节点
- 如果下一个旧节点在新列表中存在，就跳到那里
- 全部找不到返回 -1，ClampSelection 会把光标放到最后一项

### 3.3 从 Refresh 到屏幕更新的完整衔接

```
Refresh(ASYNC, FILES)
    │
    ├─ [worker goroutine]
    │   ├─ RefreshingFilesMutex.Lock()
    │   ├─ refreshStateFiles()
    │   │   ├─ GetStatusFiles(ForceShowUntracked) ← git status
    │   │   ├─ Model.Files = files
    │   │   ├─ SetTree()                            ← 重建树 + 光标调整
    │   │   │   ├─ ExpandToPath (重命名场景)
    │   │   │   ├─ IFileTree.SetTree() → getFilesForDisplay() → BuildTree
    │   │   │   ├─ findNewSelectedIdx()              ← 智能光标定位
    │   │   │   └─ ClampSelection()
    │   │   └─ RWMutex.Unlock()
    │   └─ RefreshingFilesMutex.Unlock()
    │
    └─ OnUIThread → refreshView(Files)
        │
        ├─ ① searchHelper.ReApplyFilter(context)
        │      └─ 重新应用文本过滤器（如果有）
        │      └─ SetTextFilter → SetTree() → 重建树
        │
        ├─ ② PostRefreshUpdate(context)
        │   │
        │   ├─ HandleRender()                       ← 渲染内容
        │   │   ├─ ClampSelection()                 ← 二次光标保护
        │   │   ├─ renderLines(-1, -1)              ← 生成所有行的渲染字符串
        │   │   │   └─ RenderFileTree()
        │   │   │       └─ getFileLine()
        │   │   │           ├─ formatFileStatus()   ← 状态码着色
        │   │   │           ├─ nameColor             ← 名称颜色（绿/黄/默认）
        │   │   │           └─ fileNameAtDepth()     ← 重命名显示 "old → new"
        │   │   ├─ SetContent(rendered)              ← 写入视图内容
        │   │   └─ setFooter()                       ← "x of y"
        │   │
        │   └─ 如果 Files 是当前焦点视图：
        │       └─ HandleFocus(OnFocusOpts{})
        │           ├─ FocusLine(scrollIntoView=true)
        │           │   └─ AfterLayout:
        │           │       ├─ FocusPoint(selectedIdx) ← 滚动到选中行
        │           │       ├─ SetNearestSearchPosition() ← 搜索位置
        │           │       └─ SetRangeSelectStart / CancelRangeSelect
        │           ├─ SetHighlight(true)
        │           └─ OnRenderToMain()             ← 更新主面板（diff）
        │               └─ GetOnRenderToMain()      ← 显示选中文件的 diff
        │
        └─ ③ AfterLayout → searchHelper.ReApplySearch(context)
               └─ 重新应用搜索高亮，更新 "x of y" 搜索状态
```

### 3.4 关键时序约束

1. **ReApplyFilter 必须在 HandleRender 之前**
   - 因为过滤会改变模型（重建树），渲染需要基于最新模型

2. **HandleRender 必须在 HandleFocus 之前**
   - 因为 FocusLine 中的 FocusPoint 需要内容已经渲染到视图中

3. **FocusPoint 必须在 AfterLayout 中**
   - 因为视图尺寸在布局阶段确定
   - FocusPoint 需要知道视口大小才能计算正确的滚动位置

4. **ReApplySearch 必须在 AfterLayout 中**
   - 需要先完成 FocusPoint，确保滚动位置正确
   - 否则搜索匹配的行可能在视口外，导致搜索计数错误

### 3.5 非焦点视图的处理

位置：[view_helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/view_helpers.go#L127-L163)

当 Files 视图不是当前焦点时：

```go
if gui.currentViewName() == c.GetViewName() {
    c.HandleFocus(types.OnFocusOpts{})  // 焦点视图：完整处理
} else {
    c.FocusLine(false)  // 非焦点视图：只定位光标，不滚动

    // 如果当前上下文是主面板，刷新关联的侧面板内容
    if currentCtx 是 normal main/secondary {
        sidePanelContext.HandleRenderToMain()  // 更新主面板 diff
    }
    // 如果有弹窗，也刷新主面板
    else if c 是当前 static context {
        c.HandleRenderToMain()
    }
}
```

**FocusLine(false)** 的 `false` 表示不滚动到选中行，因为非焦点视图不需要视觉滚动。但它仍然更新光标位置，确保集成测试能获取正确的选中状态。

---

## 四、总结对比表

### 重命名路径顺序全链路

| 阶段 | 字段/输出 | 新路径位置 | 旧路径位置 |
|------|----------|-----------|-----------|
| git status 输出 | `XY NEW\x00OLD` | 第1行 | 第2行 |
| FileStatus | `.Path` / `.PreviousPath` | Path | PreviousPath |
| models.File | `.Path` / `.PreviousPath` | Path | PreviousPath |
| File.Names() | 返回数组 | [0] | [1] |
| FileTree 节点 | `.path` | 新路径 | — |
| DisplayString | `"R  OLD -> NEW"` | 箭头后 | 箭头前 |
| 界面显示 | `old → new` | 箭头后 | 箭头前 |

### 未跟踪过滤器切换路径

| 切换类型 | 是否重取 git status | 刷新方式 | 光标 |
|----------|-------------------|---------|------|
| DisplayUntracked → 其他 | 是 | Refresh(ASYNC) | SetTree 智能定位 |
| 其他 → DisplayUntracked | 是 | Refresh(ASYNC) | SetTree 智能定位 |
| 其他 ↔ 其他 | 否 | PostRefreshUpdate | SetStatusFilter 归零 |

### 刷新后渲染衔接顺序

```
ReApplyFilter → HandleRender → HandleFocus → AfterLayout → ReApplySearch
    ↓               ↓              ↓              ↓             ↓
 重建树          渲染内容      FocusPoint      搜索位置      搜索高亮
```
