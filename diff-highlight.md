# Diff 视图与高亮生成机制

本文档梳理了 lazygit 中 diff 视图的**三条路径**的内容获取、行级标记和渲染输出的完整流程。

---

## 一、三条路径概览

Lazygit 存在三种不同的 diff 渲染场景，对应三条完全不同的代码路径：

| 路径 | 使用场景 | 入口函数 | Git 命令 | 行级交互 |
|------|---------|---------|---------|---------|
| **路径 A：普通查看** | 分支/commit diff、文件列表选中查看 | `RenderDiff()` / `WorktreeFileDiffCmdObj` | `git diff --color=always` | 不支持（仅浏览） |
| **路径 B：Staging** | 工作区暂存/取消暂存行 | `RefreshStagingPanel()` | `git diff --cached/--no-index --color=never`（plain=true） | 支持 |
| **路径 C：Patch Building** | 从 commit 中选择性提取行构建 patch | `RefreshPatchBuildingPanel()` | `git diff from to --color=never`（plain=true） | 支持 |

---

## 二、路径 A：普通查看（PTY 直接渲染）

### 2.1 Diff 文本来源

适用于以下场景：
- 在分支/提交上按 `d` 进入 diff 模式
- 文件列表中选中文件后右侧预览

**入口 1：Diff 模式**（[diff_helper.go:101-119](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/helpers/diff_helper.go#L101-L119)）：

```go
func (self *DiffHelper) RenderDiff() {
    args := self.DiffArgs()
    cmdObj := self.c.Git().Diff.DiffCmdObj(args)
    prefix := style.FgMagenta.Sprintf(
        "%s %s\n\n",
        self.c.Tr.ShowingGitDiff,
        "git diff "+strings.Join(args, " "),
    )
    task := types.NewRunPtyTaskWithPrefix(cmdObj.GetCmd(), prefix)
    
    self.c.RenderToMainViews(types.RefreshMainOpts{
        Pair: self.c.MainViewPairs().Normal,
        Main: &types.ViewUpdateOpts{
            Title:    "Diff",
            SubTitle: self.IgnoringWhitespaceSubTitle(),
            Task:     task,
        },
    })
}
```

**入口 2：文件列表预览**（[files_controller.go:322](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/files_controller.go#L322-L322)）：

```go
cmdObj := self.c.Git().WorkingTree.WorktreeFileDiffCmdObj(node, false, mainShowsStaged, pathOverrides)
```

### 2.2 Git 命令构建

**Diff 模式命令**（[diff.go:21-40](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/git_commands/diff.go#L21-L40)）：

```go
func (self *DiffCommands) DiffCmdObj(diffArgs []string) *oscommands.CmdObj {
    extDiffCmd := self.pagerConfig.GetExternalDiffCommand()
    // ...
    return self.cmd.New(
        NewGitCmd("diff").
            Config("diff.noprefix=false").
            Arg("--submodule").
            Arg(fmt.Sprintf("--color=%s", self.pagerConfig.GetColorArg())). // 关键：输出 ANSI 颜色
            Arg(fmt.Sprintf("--unified=%d", self.UserConfig().Git.DiffContextSize)).
            Arg(diffArgs...).
            Dir(self.repoPaths.worktreePath).
            ToArgv(),
    )
}
```

**文件列表命令**（[working_tree.go:395-431](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/git_commands/working_tree.go#L395-L431)）：

```go
func (self *WorkingTreeCommands) WorktreeFileDiffCmdObj(node models.IFile, plain bool, cached bool, ...) *oscommands.CmdObj {
    colorArg := self.pagerConfig.GetColorArg()
    if plain {
        colorArg = "never"  // plain=true 时关闭颜色
    }
    // ...
    cmdArgs := NewGitCmd("diff").
        Arg(fmt.Sprintf("--color=%s", colorArg)). // plain=false → 输出颜色
        ArgIf(cached, "--cached").
        ArgIf(noIndex, "--no-index").             // 未追踪文件使用 /dev/null 对比
        // ...
}
```

### 2.3 行级标记与高亮

**路径 A 不做行级标记**，它依赖 git 本身输出的 ANSI 颜色代码。渲染流程：

1. `RunPtyTask` 启动 PTY 子进程执行 git diff
2. git 进程输出带 ANSI 转义序列的文本（如 `\x1b[32m+新增行\x1b[0m`）
3. gocui 的 `View` 通过 `escapeInterpreter` 解析 ANSI 序列并渲染颜色
4. 光标选中行由 gocui View 层处理（[view.go:567-588](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gocui/view.go#L567-L588)）：

```go
if v.Highlight {
    rangeSelectStart := v.cy
    rangeSelectEnd := v.cy
    if v.rangeSelectStartY != -1 {
        relativeRangeSelectStart := v.rangeSelectStartY - v.oy
        rangeSelectStart = min(relativeRangeSelectStart, v.cy)
        rangeSelectEnd = max(relativeRangeSelectStart, v.cy)
    }

    if y >= rangeSelectStart && y <= rangeSelectEnd {
        // 选中行：前景色加亮 + 粗体 + 选中背景色
        fgColorComponent := fgColor & ^AttrAll
        if fgColorComponent >= AttrIsValidColor && fgColorComponent < AttrIsValidColor+8 {
            fgColor += 8  // 颜色加亮（如红→亮红）
        }
        fgColor = fgColor | AttrBold
        bgColor = (bgColor & AttrStyleBits) | v.SelBgColor
    }
}
```

**高亮特点**：
- 颜色由 git 控制（绿色新增、红色删除等）
- 选中行由 gocui 视图层处理：前景色加亮 + 粗体 + 背景色
- 无法实现首字符特殊高亮（因为不解析行内容）

---

## 三、路径 B：Staging（暂存区交互）

### 3.1 Diff 文本来源

在文件面板按 `Enter` 进入 staging 视图，左侧显示未暂存变更，右侧显示已暂存变更。

**入口**（[staging_helper.go:22-115](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/helpers/staging_helper.go#L22-L115)）：

```go
func (self *StagingHelper) RefreshStagingPanel(focusOpts types.OnFocusOpts) {
    // ...
    var file *models.File
    node := self.c.Contexts().Files.GetSelected()
    if node != nil {
        file = node.File
    }
    
    // 获取未暂存 diff（左侧主视图）
    mainDiff := self.c.Git().WorkingTree.WorktreeFileDiff(file, true, false)  // plain=true, cached=false
    // 获取已暂存 diff（右侧副视图）
    secondaryDiff := self.c.Git().WorkingTree.WorktreeFileDiff(file, true, true) // plain=true, cached=true
    
    // 初始化状态
    mainContext.SetState(
        patch_exploring.NewState(mainDiff, mainSelectedLineIdx, mainContext.GetView(), ...),
    )
    secondaryContext.SetState(
        patch_exploring.NewState(secondaryDiff, secondarySelectedLineIdx, ...),
    )
    
    // 获取渲染内容（带高亮）
    mainContent := mainContext.GetContentToRender()
    secondaryContent := secondaryContext.GetContentToRender()
    
    self.c.RenderToMainViews(types.RefreshMainOpts{
        Pair: self.c.MainViewPairs().Staging,
        Main: &types.ViewUpdateOpts{
            Task:  types.NewRenderStringWithoutScrollTask(mainContent),
            Title: self.c.Tr.UnstagedChanges,
        },
        Secondary: &types.ViewUpdateOpts{
            Task:  types.NewRenderStringWithoutScrollTask(secondaryContent),
            Title: self.c.Tr.StagedChanges,
        },
    })
}
```

### 3.2 Git 命令详解

Staging 使用 `WorktreeFileDiff(file, plain=true, cached)`，**强制 plain=true（无颜色）**，因为需要自己解析并重新染色。

**核心命令**（[working_tree.go:395-431](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/git_commands/working_tree.go#L395-L431)）：

```go
func (self *WorkingTreeCommands) WorktreeFileDiffCmdObj(node models.IFile, plain bool, cached bool, pathOverrides []string) *oscommands.CmdObj {
    colorArg := "never"  // plain=true → 不输出 ANSI 颜色
    contextSize := self.UserConfig().Git.DiffContextSize
    
    noIndex := !node.GetIsTracked() && !node.GetHasStagedChanges() && !cached && node.GetIsFile()
    
    cmdArgs := NewGitCmd("diff").
        Arg("--submodule").
        Arg(fmt.Sprintf("--unified=%d", contextSize)).
        Arg("--color=never").                           // 纯文本，无 ANSI 颜色
        Arg(fmt.Sprintf("--find-renames=%d%%", ...)).
        ArgIf(cached, "--cached").                       // 已暂存：对比 HEAD 与 index
        ArgIf(noIndex, "--no-index").                    // 未追踪：对比 /dev/null 与工作区
        Arg("--").
        ArgIf(noIndex, "/dev/null").
        Arg(paths...).
        // ...
}
```

**三种 diff 对比方式**：
1. **工作区 vs Index（未暂存）**：`git diff` （无参数）
2. **Index vs HEAD（已暂存）**：`git diff --cached`
3. **未追踪文件**：`git diff --no-index /dev/null <file>`

### 3.3 Staging 的上下文初始化

Staging 视图使用 `PatchExplorerContext`，在 [setup.go:44-57](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/context/setup.go#L44-L57) 中注册：

```go
Staging: NewPatchExplorerContext(
    c.Views().Staging,
    "main",
    STAGING_MAIN_CONTEXT_KEY,
    func() []int { return nil },  // Staging 没有"已包含行"概念，始终返回 nil
    c,
),
StagingSecondary: NewPatchExplorerContext(
    c.Views().StagingSecondary,
    "secondary",
    STAGING_SECONDARY_CONTEXT_KEY,
    func() []int { return nil },
    c,
),
```

**Staging 中 `GetIncludedLineIndices()` 返回 nil**，因为 Staging 的高亮不区分"已包含/未包含"——它只有光标选中高亮。

### 3.4 行级标记（Patch 解析）

Staging 获取到 plain diff 文本后，通过 `patch_exploring.NewState()` 进行解析：

```go
func NewState(diff string, selectedLineIdx int, view *gocui.View, oldState *State, useHunkModeByDefault bool) *State {
    patch := patch.Parse(diff)  // 结构化解析
    if !patch.ContainsChanges() {
        return nil
    }
    
    // 计算换行映射（处理视图宽度不足时的自动换行）
    viewLineIndices, patchLineIndices := wrapPatchLines(diff, view)
    // ...
}
```

`Parse()` 解析过程同路径 C，详见 4.4 节。

### 3.5 Staging 的交互式高亮

**两层高亮叠加**：

**第一层：内容颜色（Patch 格式化层）**

由 `formatView()` 生成，通过 `patch.FormatView()` 输出带 ANSI 颜色的文本：

```go
func (s *State) RenderForLineIndices(includedLineIndices []int) string {
    includedLineIndicesSet := set.NewFromSlice(includedLineIndices)
    return s.patch.FormatView(patch.FormatViewOpts{
        IncLineIndices: includedLineIndicesSet,  // Staging 传入 nil/空集合
    })
}
```

Staging 传入空集合，所以没有"已包含"高亮（无绿色背景），但仍然有：
- 新增行：绿色前景（`style.FgGreen`）
- 删除行：红色前景（`style.FgRed`）
- Hunk 头：青色前景（`style.FgCyan`）

**第二层：选中范围高亮（gocui View 层）**

通过 `FocusSelection()` 配置视图的选中范围（[patch_explorer_context.go:96-115](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/context/patch_explorer_context.go#L96-L115)）：

```go
func (self *PatchExplorerContext) FocusSelection() {
    view := self.GetView()
    state := self.GetState()
    
    newOriginY := state.CalculateOrigin(origin, bufferHeight, numLines)
    view.SetOriginY(newOriginY)
    
    startIdx, endIdx := state.SelectedViewRange()
    // 设置范围选择起始行（相对于整个内容，不是相对于可视区域）
    view.SetRangeSelectStart(startIdx)
    // 设置光标位置（相对于可视区域）
    view.SetCursorY(endIdx - newOriginY)
}
```

`SelectedViewRange()` 根据选择模式返回不同范围（[state.go:353-368](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/patch_exploring/state.go#L353-L368)）：

```go
func (s *State) SelectedViewRange() (int, int) {
    switch s.selectMode {
    case HUNK:
        return s.selectionRangeForCurrentBlockOfChanges()  // 当前连续变更块
    case RANGE:
        if s.rangeStartLineIdx > s.selectedLineIdx {
            return s.selectedLineIdx, s.rangeStartLineIdx
        }
        return s.rangeStartLineIdx, s.selectedLineIdx       // 用户选择的范围
    case LINE:
        return s.selectedLineIdx, s.selectedLineIdx         // 单行
    }
}
```

gocui View 层在渲染每个字符时判断是否在选中范围内（[view.go:567-588](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gocui/view.go#L567-L588)），在范围内则叠加：
- 前景色加亮 +8（如红→亮红）
- 粗体
- 选中背景色

### 3.6 Staging 选择的应用

用户按 `Space` 暂存选中范围时，从 State 中提取选中的 patch 行并应用：

```go
func (self *StagingController) applySelection(reverse bool) error {
    state := self.context.GetState()
    firstLineIdx, lastLineIdx := state.SelectedPatchRange()
    
    patchToApply := patch.
        Parse(state.GetDiff()).
        Transform(patch.TransformOpts{
            Reverse:             reverse,
            IncludedLineIndices: patch.ExpandRange(firstLineIdx, lastLineIdx),
            FileNameOverride:    path,
        }).
        FormatPlain()
    
    err := self.c.Git().Patch.ApplyPatch(patchToApply, ...)
}
```

---

## 四、路径 C：Patch Building（自定义 Patch 构建）

### 4.1 Diff 文本来源

在 Commit Files 面板按 `Enter` 进入 patch building 视图，左侧显示 commit diff，右侧显示已选中的自定义 patch。

**启动 PatchBuilder**（[commits_files_controller.go:508-516](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/commits_files_controller.go#L508-L516)）：

```go
func (self *CommitFilesController) startPatchBuilder() error {
    commitFilesContext := self.context()
    canRebase := commitFilesContext.GetCanRebase()
    from, to, reverse := self.currentFromToReverseForPatchBuilding()
    
    // 初始化 PatchBuilder，记录 from/to/reverse
    self.c.Git().Patch.PatchBuilder.Start(from, to, reverse, canRebase)
    return nil
}
```

**from/to 来源**（[commit_files_context.go:82-88](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/context/commit_files_context.go#L82-L88)）：

```go
func (self *CommitFilesContext) GetFromAndToForDiff() (string, string) {
    if refs := self.GetRefRange(); refs != nil {
        return refs.From.ParentRefName(), refs.To.RefName()  // 范围：A^..B
    }
    ref := self.GetRef()
    return ref.ParentRefName(), ref.RefName()                // 单 commit：A^..A
}
```

**面板刷新入口**（[patch_building_helper.go:57-114](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/helpers/patch_building_helper.go#L57-L114)）：

```go
func (self *PatchBuildingHelper) RefreshPatchBuildingPanel(opts types.OnFocusOpts) {
    path := self.c.Contexts().CommitFiles.GetSelectedPath()
    
    from, to := self.c.Contexts().CommitFiles.GetFromAndToForDiff()
    from, reverse := self.c.Modes().Diffing.GetFromAndReverseArgsForDiff(from)
    
    // 左侧：完整 commit diff（plain=true，无颜色）
    diff, err := self.c.Git().WorkingTree.ShowFileDiff(from, to, reverse, path, true)
    
    // 右侧：已选中的自定义 patch（渲染时自带颜色）
    secondaryDiff := self.c.Git().Patch.PatchBuilder.RenderPatchForFile(patch.RenderPatchForFileOpts{
        Filename:                               path,
        Plain:                                  false,
        Reverse:                                false,
        TurnAddedFilesIntoDiffAgainstEmptyFile: true,
    })
    
    context := self.c.Contexts().CustomPatchBuilder
    state := patch_exploring.NewState(diff, selectedLineIdx, context.GetView(), oldState, ...)
    context.SetState(state)
    
    // 左侧：完整 diff（带光标选中高亮）
    mainContent := context.GetContentToRender()
    
    self.c.RenderToMainViews(types.RefreshMainOpts{
        Pair: self.c.MainViewPairs().PatchBuilding,
        Main: &types.ViewUpdateOpts{
            Task:  types.NewRenderStringWithoutScrollTask(mainContent),
            Title: self.c.Tr.Patch,
        },
        Secondary: &types.ViewUpdateOpts{
            Task:  types.NewRenderStringWithoutScrollTask(secondaryDiff),
            Title: self.c.Tr.CustomPatch,
        },
    })
}
```

### 4.2 Git 命令详解

Patch Building 使用 `ShowFileDiff(from, to, reverse, fileName, plain=true)`：

**核心命令**（[working_tree.go:439-469](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/git_commands/working_tree.go#L439-L469)）：

```go
func (self *WorkingTreeCommands) ShowFileDiffCmdObj(from string, to string, reverse bool, fileNames []string, plain bool) *oscommands.CmdObj {
    colorArg := "never"  // plain=true → 不输出 ANSI 颜色
    
    cmdArgs := NewGitCmd("diff").
        Config("diff.noprefix=false").
        Arg("--submodule").
        Arg(fmt.Sprintf("--unified=%d", contextSize)).
        Arg("--no-renames").
        Arg("--color=never").  // 纯文本
        Arg(from).             // 起始引用（如 commit^）
        Arg(to).               // 结束引用（如 commit）
        ArgIf(reverse, "-R").  // 是否反向
        Arg("--").
        Arg(fileNames...).
        // ...
}
```

**典型命令**：`git diff --no-renames --color=never abc123^ abc123 -- path/to/file.go`

### 4.3 PatchBuilder 状态管理

PatchBuilder 维护每个文件的选中状态（[patch_builder.go:24-51](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/patch_builder.go#L24-L51)）：

```go
type fileInfo struct {
    mode                PatchStatus   // UNSELECTED / WHOLE / PART
    includedLineIndices []int         // 选中的 patch 行索引
    diff                string        // 该文件的完整 diff（缓存）
}

type PatchBuilder struct {
    To         string
    From       string
    reverse    bool
    CanRebase  bool
    fileInfoMap map[string]*fileInfo   // 文件名 → 文件信息
    loadFileDiff loadFileDiffFunc      // 加载 diff 的回调
}
```

**首次访问文件时懒加载 diff**（[patch_builder.go:127-145](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/patch_builder.go#L127-L145)）：

```go
func (p *PatchBuilder) getFileInfo(filename string) (*fileInfo, error) {
    info, ok := p.fileInfoMap[filename]
    if ok {
        return info, nil  // 已有缓存
    }
    
    // 首次访问：调用 git diff 获取该文件完整 diff
    diff, err := p.loadFileDiff(p.From, p.To, p.reverse, filename, true)
    info = &fileInfo{
        mode: UNSELECTED,
        diff: diff,
    }
    p.fileInfoMap[filename] = info
    return info, nil
}
```

### 4.4 行级标记（Patch 解析）

Patch Building 同样使用 `patch.Parse()` 进行结构化解析。

**解析流程**（[parse.go:12-42](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/parse.go#L12-L42)）：

```go
func Parse(patchStr string) *Patch {
    lines := strings.Split(strings.TrimSuffix(patchStr, "\n"), "\n")
    
    hunks := []*Hunk{}
    patchHeader := []string{}
    
    var currentHunk *Hunk
    for _, line := range lines {
        if strings.HasPrefix(line, "@@") {
            // 正则提取 hunk 头信息：@@ -oldStart,oldCount +newStart,newCount @@ context
            oldStart, newStart, headerContext := headerInfo(line)
            currentHunk = &Hunk{
                oldStart:      oldStart,
                newStart:      newStart,
                headerContext: headerContext,
                bodyLines:     []*PatchLine{},
            }
            hunks = append(hunks, currentHunk)
        } else if currentHunk != nil {
            // 按首字符标记行类型
            currentHunk.bodyLines = append(currentHunk.bodyLines, newHunkLine(line))
        } else {
            patchHeader = append(patchHeader, line)
        }
    }
    
    return &Patch{
        hunks:  hunks,
        header: patchHeader,
    }
}
```

**行类型判别**（[parse.go:72-85](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/parse.go#L72-L85)）：

```go
func parseFirstChar(firstChar string) PatchLineKind {
    switch firstChar {
    case " ":
        return CONTEXT          // 空格开头 → 上下文行
    case "+":
        return ADDITION         // + 开头 → 新增行
    case "-":
        return DELETION         // - 开头 → 删除行
    case "\\":
        return NEWLINE_MESSAGE  // \ 开头 → 换行消息（如 "\ No newline at end of file"）
    }
    return CONTEXT
}
```

**数据结构**：

```go
type PatchLineKind int
const (
    PATCH_HEADER PatchLineKind = iota  // 0: diff --git ... 等头部
    HUNK_HEADER                        // 1: @@ -x,y +a,b @@ ...
    ADDITION                           // 2: + 新增行
    DELETION                           // 3: - 删除行
    CONTEXT                            // 4: 空格 上下文
    NEWLINE_MESSAGE                    // 5: \ No newline...
)
```

### 4.5 Patch Building 的上下文初始化

Patch Building 视图也使用 `PatchExplorerContext`，但 `getIncludedLineIndices` 不为空（[setup.go:58-73](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/context/setup.go#L58-L73)）：

```go
CustomPatchBuilder: NewPatchExplorerContext(
    c.Views().PatchBuilding,
    "main",
    PATCH_BUILDING_MAIN_CONTEXT_KEY,
    func() []int {
        // 从 PatchBuilder 获取当前文件已选中的行索引
        filename := commitFilesContext.GetSelectedPath()
        includedLineIndices, err := c.Git().Patch.PatchBuilder.GetFileIncLineIndices(filename)
        if err != nil {
            c.Log.Error(err)
            return nil
        }
        return includedLineIndices
    },
    c,
),
```

### 4.6 行选择交互

用户按 `Space` 切换选中状态（[patch_building_controller.go:137-179](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/patch_building_controller.go#L137-L179)）：

```go
func (self *PatchBuildingController) toggleSelection() error {
    filename := self.c.Contexts().CommitFiles.GetSelectedPath()
    state := self.context().GetState()
    
    // 获取选中范围内的新增/删除行（排除上下文行）
    lineIndicesToToggle := state.LineIndicesOfAddedOrDeletedLinesInSelectedPatchRange()
    
    // 查询当前文件已包含的行
    includedLineIndices, err := self.c.Git().Patch.PatchBuilder.GetFileIncLineIndices(filename)
    
    // 判断是添加还是移除：根据第一条选中行是否已包含
    firstSelectedChangeLineIsStaged := lo.Contains(includedLineIndices, lineIndicesToToggle[0])
    
    toggleFunc := self.c.Git().Patch.PatchBuilder.AddFileLineRange
    if firstSelectedChangeLineIsStaged {
        toggleFunc = self.c.Git().Patch.PatchBuilder.RemoveFileLineRange
    }
    
    toggleFunc(filename, lineIndicesToToggle)
    
    // 跳到下一个同状态的可选择行
    state.SelectNextStageableLineOfSameIncludedState(
        self.context().GetIncludedLineIndices(), 
        firstSelectedChangeLineIsStaged,
    )
    return nil
}
```

**PatchBuilder 的行索引管理**（[patch_builder.go:147-170](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/patch_builder.go#L147-L170)）：

```go
func (p *PatchBuilder) AddFileLineRange(filename string, lineIndices []int) error {
    info, err := p.getFileInfo(filename)
    info.mode = PART
    info.includedLineIndices = lo.Union(info.includedLineIndices, lineIndices)  // 并集
    return nil
}

func (p *PatchBuilder) RemoveFileLineRange(filename string, lineIndices []int) error {
    info, err := p.getFileInfo(filename)
    info.mode = PART
    info.includedLineIndices, _ = lo.Difference(info.includedLineIndices, lineIndices)  // 差集
    if len(info.includedLineIndices) == 0 {
        p.removeFile(info)
    }
    return nil
}
```

### 4.7 Patch Building 的三层高亮

Patch Building 有**三层高亮叠加**：

**第一层：内容颜色（format.go）**

- 新增行：`style.FgGreen` 绿色前景
- 删除行：`style.FgRed` 红色前景
- Hunk 头：`style.FgCyan` 青色前景
- Patch 头：粗体默认色

**第二层：已包含行标记（format.go → 首字符绿底）**

由 `incLineIndices` 控制，已包含的变更行首字符叠加绿色背景：

```go
func (self *patchPresenter) formatLineAux(str string, textStyle style.TextStyle, included bool) string {
    firstCharStyle := textStyle
    if included {
        // MergeStyle 合并前景色（如绿色）与背景色（BgGreen）
        firstCharStyle = firstCharStyle.MergeStyle(style.BgGreen)
    }
    
    if len(str) < 2 {
        return firstCharStyle.Sprint(str)
    }
    
    // 首字符：前景色 + 可选绿底
    // 其余字符：仅前景色
    return firstCharStyle.Sprint(str[:1]) + textStyle.Sprint(str[1:])
}
```

**效果**：
- 已选中的新增行：`+` 号显示绿底绿字，其余绿色
- 已选中的删除行：`-` 号显示绿底红字，其余红色
- 未选中的新增行：全绿色
- 未选中的删除行：全红色

**第三层：光标/范围选中高亮（gocui View 层）**

与 Staging 相同，通过 `SetRangeSelectStart()` 和 `SetCursorY()` 配置，gocui 在渲染时对范围内字符叠加：
- 前景色加亮
- 粗体
- 选中背景色

**三层叠加示意**：

```
未选中的 +新增行     → 仅第一层：绿色前景
已选中的 +新增行     → 第一层(绿前景) + 第二层(首字符绿底)
光标所在的 +新增行   → 第一层(绿前景) + 第二层(首字符绿底) + 第三层(加亮+粗体+选中背景)
```

### 4.8 右侧自定义 Patch 渲染

右侧副视图通过 `PatchBuilder.RenderPatchForFile()` 渲染：

```go
func (p *PatchBuilder) RenderPatchForFile(opts RenderPatchForFileOpts) string {
    info, err := p.getFileInfo(opts.Filename)
    if info.mode == UNSELECTED {
        return ""
    }
    
    // 解析 → Transform（仅保留 IncludedLineIndices 的行）→ 格式化
    patch := Parse(info.diff).
        Transform(TransformOpts{
            Reverse:                                opts.Reverse,
            TurnAddedFilesIntoDiffAgainstEmptyFile: opts.TurnAddedFilesIntoDiffAgainstEmptyFile,
            IncludedLineIndices:                    info.includedLineIndices,
        })
    
    if opts.Plain {
        return patch.FormatPlain()
    }
    return patch.FormatView(FormatViewOpts{})  // 右侧不区分 included，只显示颜色
}
```

---

## 五、三种渲染路径对比表

| 维度 | 路径 A：普通查看 | 路径 B：Staging | 路径 C：Patch Building |
|------|----------------|----------------|----------------------|
| **使用场景** | 浏览 diff、文件预览 | 暂存/取消暂存行 | 从 commit 提取行构建 patch |
| **入口函数** | `RenderDiff()` / `WorktreeFileDiffCmdObj(plain=false)` | `RefreshStagingPanel()` → `WorktreeFileDiff(plain=true)` | `RefreshPatchBuildingPanel()` → `ShowFileDiff(plain=true)` |
| **Git 命令颜色** | `--color=always`（git 自带颜色） | `--color=never`（lazygit 重新染色） | `--color=never`（lazygit 重新染色） |
| **渲染方式** | PTY 流式输出 | `RenderStringWithoutScrollTask` | `RenderStringWithoutScrollTask` |
| **Patch 解析** | 不解析 | `patch.Parse()` 结构化 | `patch.Parse()` 结构化 |
| **行类型标记** | 不做（git 已输出颜色） | 首字符判别 `+/-/空格` | 首字符判别 `+/-/空格` |
| **内容颜色层** | git ANSI 输出 | `formatView()` 生成 | `formatView()` 生成 |
| **已包含行标记** | 无 | 无（返回 nil） | 有（`PatchBuilder.GetFileIncLineIndices`）→ 首字符绿底 |
| **光标选中高亮** | gocui View 层（加亮+粗体+背景） | gocui View 层（加亮+粗体+背景） | gocui View 层（加亮+粗体+背景） |
| **高亮层数** | 2 层（git 颜色 + 选中） | 2 层（内容颜色 + 选中） | 3 层（内容颜色 + 已包含标记 + 选中） |
| **行选择模式** | 无 | LINE / RANGE / HUNK | LINE / RANGE / HUNK |
| **选中范围持久化** | 无 | 不持久化（每次 Space 即应用） | `PatchBuilder.fileInfoMap` 持久化到内存 |

---

## 六、完整流程图

```
用户操作
    │
    ├───────────── 浏览 Diff ─────────────┐
    │                                      │
    │  ┌────────────────────────────┐      │  ┌────────────────────────────┐
    │  │ 路径 A: PTY 直接渲染       │      │  │ 路径 B/C: Patch 格式化渲染 │
    │  └─────────────┬──────────────┘      │  └──────────────┬─────────────┘
    │                │                     │                 │
    │                ▼                     │                 ▼
    │  DiffCmdObj/DiffFileCmdObj           │  WorktreeFileDiff(plain=true)
    │  (--color=always)                    │  ShowFileDiff(plain=true)
    │                │                     │  (--color=never)
    │                ▼                     │                 │
    │         RunPtyTask 执行              │                 ▼
    │  git 输出带 ANSI 颜色的文本          │        patch.Parse() 解析
    │                │                     │    ┌────────────────────────────┐
    │                ▼                     │    │ 首字符判别行类型            │
    │     gocui View 解析 ANSI             │    │ +→ADDITION, -→DELETION     │
    │                │                     │    │ 空格→CONTEXT               │
    │                ▼                     │    └──────────────┬─────────────┘
    │    颜色层：git 自带颜色               │                 │
    │    选中层：gocui 加亮+粗体+背景       │                 ▼
    │                                      │    patch_exploring.NewState()
    │                                      │  ┌─────────────────────────────┐
    │                                      │  │ State: 选中行索引、选择模式 │
    │                                      │  │  LINE / RANGE / HUNK        │
    │                                      │  │ viewLine/patchLine 双映射   │
    │                                      │  └──────────────┬──────────────┘
    │                                      │                 │
    │                                      │                 ▼
    │                                      │    Patch.FormatView() 渲染
    │                                      │  ┌─────────────────────────────┐
    │                                      │  │ 内容颜色层：                 │
    │                                      │  │   ADDITION→FgGreen          │
    │                                      │  │   DELETION→FgRed            │
    │                                      │  │   HUNK_HEADER→FgCyan        │
    │                                      │  │                             │
    │                                      │  │ 已包含标记层（仅 Path C）：  │
    │                                      │  │   incLineIndices 行         │
    │                                      │  │   首字符 BgGreen            │
    │                                      │  └──────────────┬──────────────┘
    │                                      │                 │
    │                                      │                 ▼
    │                                      │    FocusSelection() 配置选中
    │                                      │    SetRangeSelectStart(start)
    │                                      │    SetCursorY(end)
    │                                      │                 │
    │                                      │                 ▼
    │                                      │    选中层：gocui 加亮+粗体+背景
    │                                      │
    └──────────────────────────────────────┴─────────────────────────────────┘
                                      ▼
                            gocui 绘制到终端
```

---

## 七、关键技术点总结

### 7.1 Diff 文本获取的三条路径

1. **普通查看**：`git diff --color=always`，直接通过 PTY 流式输出，git 负责颜色
2. **Staging**：`git diff [--cached|--no-index] --color=never`，获取纯文本后 lazygit 自行解析染色
3. **Patch Building**：`git diff from to --no-renames --color=never`，同样获取纯文本，且在 PatchBuilder 中按文件缓存 diff

### 7.2 行级标记核心实现

1. **首字符判别法**：`+` → ADDITION，`-` → DELETION，`空格` → CONTEXT，`\` → NEWLINE_MESSAGE
2. **Hunk 头正则**：`^@@ -(\d+)[^\+]+\+(\d+)[^@]+@@(.*)$` 提取旧/新起始行号和上下文
3. **双索引映射**：`viewLineIndices`（patch 行→视图行）和 `patchLineIndices`（视图行→patch 行）处理自动换行

### 7.3 交互式高亮的三层机制

| 层级 | 实现位置 | 作用 | 适用路径 |
|------|---------|------|---------|
| 内容颜色层 | `format.go:patchLineStyle()` | 区分新增/删除/hunk头 | B、C |
| 已包含标记层 | `format.go:formatLineAux()` | 首字符绿底标记已选中行 | 仅 C |
| 光标选中层 | `view.go:567-588` | 加亮+粗体+选中背景色 | A、B、C |

### 7.4 范围选择的两种粒度控制

- **State 层**：`SelectedViewRange()` 返回 LINE/RANGE/HUNK 三种模式的视图行范围
- **gocui View 层**：`rangeSelectStartY` 和 `cy` 决定最终渲染时的高亮范围

### 7.5 关键文件索引

| 文件 | 作用 |
|------|------|
| [diff_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/helpers/diff_helper.go) | 路径 A：Diff 模式渲染入口 |
| [staging_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/helpers/staging_helper.go) | 路径 B：Staging 面板刷新 |
| [patch_building_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/controllers/helpers/patch_building_helper.go) | 路径 C：Patch Building 面板刷新 |
| [working_tree.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/git_commands/working_tree.go) | `WorktreeFileDiff` / `ShowFileDiff` 命令构建 |
| [diff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/git_commands/diff.go) | 路径 A：Diff 模式命令构建 |
| [parse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/parse.go) | Diff 文本解析与行类型标记 |
| [format.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/format.go) | Patch 格式化渲染与颜色层/已包含标记层 |
| [patch_builder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/commands/patch/patch_builder.go) | 路径 C：PatchBuilder 状态与行索引管理 |
| [state.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/patch_exploring/state.go) | Patch 浏览状态（选中行、选择模式、双索引映射） |
| [patch_explorer_context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/context/patch_explorer_context.go) | PatchExplorer 视图上下文与 FocusSelection |
| [setup.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/context/setup.go) | 上下文注册（含 getIncludedLineIndices 回调） |
| [text_style.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gui/style/text_style.go) | 样式系统（颜色、粗体、MergeStyle） |
| [view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-lazygit/pkg/gocui/view.go) | gocui 终端视图渲染与光标选中高亮 |
