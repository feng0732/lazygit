# 文件状态与未跟踪展示链路分析

## 整体架构概览

文件状态展示链路涉及以下核心模块，按数据流向排列：

```
git status 命令
    ↓
FileLoader (状态收集)
    ↓
models.File (数据模型 + 状态字段推导)
    ↓
FileTree (文件树构建 + 过滤分类)
    ↓
FileTreeViewModel (视图模型 + 光标管理)
    ↓
WorkingTreeContext (上下文 + 渲染)
    ↓
presentation (UI 展示)
```

触发刷新的入口包括：
- 后台定时自动刷新
- 用户操作后主动刷新
- 乐观渲染即时更新

---

## 一、状态收集：FileLoader

### 1.1 核心入口

位置：[file_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/git_commands/file_loader.go)

核心方法：`GetStatusFiles(opts GetStatusFileOptions) []*models.File`

### 1.2 未跟踪文件配置处理

```go
untrackedFilesSetting := self.config.GetShowUntrackedFiles()
if opts.ForceShowUntracked || untrackedFilesSetting == "" {
    untrackedFilesSetting = "all"
}
untrackedFilesArg := fmt.Sprintf("--untracked-files=%s", untrackedFilesSetting)
```

- 配置项 `gui.showUntrackedFiles` 控制是否显示未跟踪文件
- `ForceShowUntracked` 参数用于强制显示（如用户选择了"只显示未跟踪文件"过滤时）
- 默认值为 `"all"`（显示所有未跟踪文件）

### 1.3 git status 命令调用

```go
func (self *FileLoader) gitStatus(opts GitStatusOptions) ([]FileStatus, error) {
    cmdArgs := NewGitCmd("status").
        Arg(opts.UntrackedFilesArg).
        Arg("--porcelain").
        Arg("-z").
        ArgIfElse(
            opts.NoRenames,
            "--no-renames",
            fmt.Sprintf("--find-renames=%d%%", self.UserConfig().Git.RenameSimilarityThreshold),
        ).
        ToArgv()
    // ...
}
```

关键参数：
- `--porcelain`: 稳定的机器可读输出格式
- `-z`: 用 NUL 字符分隔文件名（处理含空格的文件名）
- `--find-renames`: 重命名检测阈值
- `--untracked-files`: 未跟踪文件展示模式

### 1.4 输出解析

输出格式为 `XX 文件名`，其中 `XX` 是两位状态码：
- 第 1 位：暂存区状态（staged）
- 第 2 位：工作区状态（unstaged）

对于重命名/复制文件，格式为 `RX 旧文件名` + `新文件名`（两行）。

### 1.5 额外数据：numstat

如果配置 `gui.showNumstatInFilesView` 为 true，会额外调用 `git diff --numstat -z HEAD` 获取每个文件的增删行数。

---

## 二、状态分类规则：deriveStatusFields

### 2.1 核心推导函数

位置：[file.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/commands/models/file.go#L146-L163)

```go
func deriveStatusFields(shortStatus string) StatusFields {
    stagedChange := shortStatus[0:1]
    unstagedChange := shortStatus[1:2]
    tracked := !lo.Contains([]string{"??", "A ", "AM"}, shortStatus)
    hasStagedChanges := !lo.Contains([]string{" ", "U", "?"}, stagedChange)
    hasInlineMergeConflicts := lo.Contains([]string{"UU", "AA"}, shortStatus)
    hasMergeConflicts := hasInlineMergeConflicts || 
        lo.Contains([]string{"DD", "AU", "UA", "UD", "DU"}, shortStatus)
    // ...
}
```

### 2.2 状态字段详解

| 字段 | 含义 | 推导规则 |
|------|------|----------|
| `HasStagedChanges` | 是否有暂存变更 | 暂存位不是空格、U、? |
| `HasUnstagedChanges` | 是否有未暂存变更 | 工作区位不是空格 |
| `Tracked` | 是否被 git 跟踪 | 状态不是 `??`、`A `、`AM` |
| `Added` | 是否是新增文件 | 工作区位是 A，或未被跟踪 |
| `Deleted` | 是否是删除文件 | 暂存位或工作区位是 D |
| `HasMergeConflicts` | 是否有合并冲突 | 包含内联冲突或其他冲突类型 |
| `HasInlineMergeConflicts` | 是否有内联合并冲突 | 状态是 `UU` 或 `AA` |

### 2.3 常见状态码对照表

| 状态码 | 含义 | Tracked | HasStaged | HasUnstaged |
|--------|------|---------|-----------|-------------|
| `??` | 未跟踪文件 | false | false | true |
| `A ` | 新增并暂存 | false | true | false |
| `AM` | 新增暂存但工作区有修改 | false | true | true |
| ` M` | 工作区修改，未暂存 | true | false | true |
| `M ` | 修改已暂存 | true | true | false |
| `MM` | 修改已暂存，工作区又有修改 | true | true | true |
| ` D` | 工作区删除，未暂存 | true | false | true |
| `D ` | 删除已暂存 | true | true | false |
| `UU` | 未合并冲突（双方修改）| true | - | - |
| `AA` | 未合并冲突（双方新增）| true | - | - |
| `DD` | 未合并冲突（双方删除）| true | - | - |
| `AU` | 未合并冲突（新增/修改）| true | - | - |
| `UA` | 未合并冲突（修改/新增）| true | - | - |
| `UD` | 未合并冲突（修改/删除）| true | - | - |
| `DU` | 未合并冲突（删除/修改）| true | - | - |

---

## 三、文件树构建与过滤

### 3.1 FileTree 结构

位置：[file_tree.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree.go)

`FileTree` 负责：
- 持有文件列表的获取函数 `getFiles`
- 管理显示过滤器
- 构建树状或扁平结构
- 管理目录折叠状态

### 3.2 显示过滤器

定义了 6 种过滤模式：

| 过滤器 | 显示条件 |
|--------|----------|
| `DisplayAll` | 显示所有文件 |
| `DisplayStaged` | `file.HasStagedChanges == true` |
| `DisplayUnstaged` | `file.HasUnstagedChanges == true` |
| `DisplayTracked` | `file.Tracked || file.HasStagedChanges`（含已暂存的未跟踪文件） |
| `DisplayUntracked` | `!(file.Tracked || file.HasStagedChanges)` |
| `DisplayConflicted` | `file.HasMergeConflicts == true` |

过滤逻辑在 `getFilesForDisplay()` 方法中实现。

### 3.3 树构建

位置：[build_tree.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/build_tree.go)

`BuildTreeFromFiles()` 将扁平文件列表构建为树：
- 按路径分割逐层创建目录节点
- 支持目录压缩（单目录子目录合并显示，节省垂直空间）
- 按配置的排序规则排序

### 3.4 扁平模式的特殊排序

`BuildFlatTreeFromFiles()` 构建扁平列表时有特殊排序规则：

```go
// 排序优先级：
// 1. 有合并冲突的文件（最前）
// 2. 已跟踪文件
// 3. 未跟踪文件（最后）
```

---

## 四、视图模型：FileTreeViewModel

位置：[file_tree_view_model.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/filetree/file_tree_view_model.go)

### 4.1 职责

`FileTreeViewModel` 组合了：
- `IFileTree`：文件树数据结构
- `IListCursor`：列表光标（选中位置）
- `sync.RWMutex`：并发读写锁

### 4.2 刷新时的光标保持

`SetTree()` 方法在刷新树时会尝试保持选中位置：
1. 记录之前选中的节点
2. 重新构建树
3. 在新树中查找之前选中的文件
4. 如果找不到，继续向下查找下一个存在的文件
5. 确保光标不越界

---

## 五、界面刷新机制

### 5.1 刷新入口类型

位置：[refresh.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/types/refresh.go)

三种刷新模式：

| 模式 | 含义 |
|------|------|
| `SYNC` | 同步刷新，等待所有刷新完成后返回 |
| `ASYNC` | 异步刷新，立即返回，各部分独立更新 |
| `BLOCK_UI` | 阻塞 UI，一次性更新所有内容 |

### 5.2 刷新调度中心

位置：[refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/helpers/refresh_helper.go)

`RefreshHelper.Refresh()` 是刷新调度中心：
- 接收刷新范围（Scope）和模式（Mode）
- 按依赖关系分组并行刷新
- 使用 WaitGroup 协调并发

文件刷新相关代码：

```go
if scopeSet.Includes(types.FILES) || scopeSet.Includes(types.SUBMODULES) {
    fileWg.Add(1)
    refresh("files", func() {
        _ = self.refreshFilesAndSubmodules()
        fileWg.Done()
    })
}
```

### 5.3 文件刷新流程

`refreshFilesAndSubmodules()` → `refreshStateFiles()`

核心步骤：
1. 获取互斥锁 `RefreshingFilesMutex`
2. 刷新子模块配置
3. 处理已解决的冲突文件自动暂存
4. 调用 `FileLoader.GetStatusFiles()` 获取最新状态
5. 检测冲突文件数量变化
6. 更新 `Model.Files`
7. 调用 `fileTreeViewModel.SetTree()` 重建树
8. 在 UI 线程刷新视图

### 5.4 冲突自动切换过滤

当检测到冲突文件数量变化时，会自动切换过滤器：

```go
if conflictFileCount > 0 && prevConflictFileCount == 0 {
    if fileTreeViewModel.GetStatusFilter() == filetree.DisplayAll {
        fileTreeViewModel.SetStatusFilter(filetree.DisplayConflicted)
        self.c.Contexts().Files.GetView().Subtitle = self.c.Tr.FilterLabelConflictingFiles
    }
} else if conflictFileCount == 0 && 
           fileTreeViewModel.GetStatusFilter() == filetree.DisplayConflicted {
    fileTreeViewModel.SetStatusFilter(filetree.DisplayAll)
    self.c.Contexts().Files.GetView().Subtitle = ""
}
```

### 5.5 后台定时刷新

位置：[background.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/background.go)

`BackgroundRoutineMgr` 管理后台刷新：

```go
func (self *BackgroundRoutineMgr) startBackgroundFilesRefresh() {
    // ...
    self.goEvery(userConfig.Refresher.RefreshIntervalDuration(), 
                 self.gui.stopChan, 
                 func(_ bool) error {
        self.gui.c.Refresh(types.RefreshOptions{
            Scope: []types.RefreshableView{types.FILES},
        })
        return nil
    })
}
```

配置项：
- `git.autoRefresh`: 是否启用自动刷新
- `refresher.refreshInterval`: 刷新间隔（秒）

### 5.6 用户操作触发刷新

常见触发点：
- 暂存/取消暂存文件后
- 提交后
- 切换过滤器后（仅在未跟踪过滤器切换时需要全量刷新）
- 用户手动按刷新键

---

## 六、乐观渲染（Optimistic Rendering）

### 6.1 设计动机

位置：[files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/controllers/files_controller.go#L415-L421)

```go
// Running a git add command followed by a git status command can take some time (e.g. 200ms).
// Given how often users stage/unstage files in Lazygit, we're adding some
// optimistic rendering to make things feel faster.
```

### 6.2 状态转换映射

暂存映射（stageStatusMap）：

| 原状态 | 暂存后 |
|--------|--------|
| `??` | `A ` |
| ` M` | `M ` |
| `MM` | `M ` |
| ` D` | `D ` |
| ` A` | `A ` |
| `AM` | `A ` |
| `MD` | `D ` |

取消暂存映射（unstageStatusMap）：

| 原状态 | 取消暂存后 |
|--------|------------|
| `A ` | `??` |
| `M ` | ` M` |
| `D ` | ` D` |
| `MM` | ` M` |

### 6.3 执行流程

1. 调用 `optimisticChange()` 更新内存中的文件状态
2. 调用 `PostRefreshUpdate()` 立即重绘界面
3. 执行实际的 git 命令
4. 调用 `Refresh()` 进行真实状态同步（修正可能的偏差）

---

## 七、UI 展示

### 7.1 渲染入口

位置：[files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/34-lazygit/pkg/gui/presentation/files.go)

`RenderFileTree()` 递归渲染整棵树，每个节点调用 `getFileLine()`。

### 7.2 状态颜色规则

```go
if hasStagedChanges && !hasUnstagedChanges {
    nameColor = style.FgGreen      // 纯暂存：绿色
} else if hasStagedChanges {
    nameColor = style.FgYellow     // 混合状态：黄色
} else {
    nameColor = theme.DefaultTextColor  // 纯未暂存：默认色
}
```

### 7.3 状态码显示

`formatFileStatus()` 将两位状态码分别着色：
- 第 1 位（暂存状态）：绿色或默认色
- 第 2 位（工作区状态）：红色（UnstagedChangesColor）或默认色

未跟踪文件（`??`）两个字符都显示为红色。

---

## 八、关键数据结构关系

```
File (models)
  ├─ Path: string
  ├─ ShortStatus: string (两位状态码，如 "??", "M ")
  ├─ HasStagedChanges: bool
  ├─ HasUnstagedChanges: bool
  ├─ Tracked: bool
  ├─ Added: bool
  ├─ Deleted: bool
  └─ HasMergeConflicts: bool

Node[T] (filetree)
  ├─ File: *T           (nil 表示目录)
  ├─ Children: []*Node[T]
  ├─ path: string
  └─ CompressionLevel: int

FileNode (filetree)
  └─ *Node[models.File]
     ├─ GetHasStagedChanges() bool  (递归检查子节点)
     ├─ GetHasUnstagedChanges() bool
     └─ GetIsTracked() bool

FileTree (filetree)
  ├─ getFiles: func() []*models.File
  ├─ filter: FileTreeDisplayFilter
  ├─ collapsedPaths: *CollapsedPaths
  └─ tree: *Node[models.File]

FileTreeViewModel (filetree)
  ├─ IFileTree
  ├─ IListCursor
  └─ sync.RWMutex
```

---

## 九、完整链路示例：暂存一个未跟踪文件

1. **用户按空格** → `FilesController.press()`
2. **乐观渲染** → `optimisticStage()` 将 `??` 改为 `A `，立即重绘
3. **执行 git 命令** → `git add <file>`
4. **异步刷新** → `Refresh(Scope: [FILES], Mode: ASYNC)`
5. **状态收集** → `FileLoader.GetStatusFiles()` 调用 `git status --porcelain -z`
6. **状态推导** → `SetStatusFields()` 解析状态码设置各字段
7. **树重建** → `FileTreeViewModel.SetTree()` 重建树并保持光标
8. **视图更新** → UI 线程调用 `refreshView()` 重绘文件面板
