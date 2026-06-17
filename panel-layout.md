# Lazygit 面板布局与切换代码链路分析

本文档基于代码事实，逐一梳理三种入口（Tab切换、侧边跳转、鼠标点击）如何触发**上下文栈变化 → 布局计算 → 视图重绘**的完整链路。

---

## 1. 核心概念与代码事实

### 1.1 三个入口函数的真实位置

| 操作 | 入口函数 | 文件位置 |
|------|----------|----------|
| 同窗口 Tab 切换（NextTab / PrevTab） | [handleNextTab](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/view_helpers.go#L77-L118) | `pkg/gui/view_helpers.go:77-118` |
| 同窗口 Tab 点击（鼠标点 tab） | [onViewTabClick](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/view_helpers.go#L60-L75) | `pkg/gui/view_helpers.go:60-75` |
| 侧边窗口跳转（数字键 1-5 / JumpToBlock） | [goToSideWindow](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/jump_to_side_window_controller.go#L48-L60) | `pkg/gui/controllers/jump_to_side_window_controller.go:48-60` |
| 侧边窗口顺序切换（PrevBlock / NextBlock，h/l） | [previousSideWindow / nextSideWindow](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/side_window_controller.go#L47-L93) | `pkg/gui/controllers/side_window_controller.go:47-93` |

### 1.2 三者的共性：最终都调用 `ContextMgr.Push`

**关键事实**：上面四个入口函数，最终无一例外地调用了 `self.c.Context().Push(context, types.OnFocusOpts{})`。区别只在于：
- 跳转目标如何确定
- 目标 context 的 Window/View 是否与当前 context 在同一个 Window 中

### 1.3 viewTabMap：Window 与多 Tab 的关系

定义在 [gui.go:836-879](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/gui.go#L836-L879)，它定义了哪些 Window 支持多 Tab：

```go
func (gui *Gui) viewTabMap() map[string][]context.TabView {
    result := map[string][]context.TabView{
        "branches": {  // Window 名
            {Tab: "本地分支", ViewName: "localBranches"},
            {Tab: "远程",     ViewName: "remotes"},
            {Tab: "标签",     ViewName: "tags"},
        },
        "commits": {
            {Tab: "提交历史", ViewName: "commits"},
            {Tab: "Reflog",    ViewName: "reflogCommits"},
        },
        "files": {
            {Tab: "文件",       ViewName: "files"},
            {Tab: "工作树",     ViewName: "worktrees"},
            {Tab: "子模块",     ViewName: "submodules"},
        },
    }
    return result
}
```

**重要关系**：
- 一个 Window 可以有多个 Tab（对应多个 View / Context）
- 其他 Window（status、stash、main、extras）只有一个 Tab，不支持多 Tab 切换
- Tab 绑定注册在 [keybindings.go:368-378](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/keybindings.go#L368-L378)：对每个有 Tab 的 View 注册 `SetTabClickBinding`

### 1.4 SideWindows 列表

定义在 [window_helper.go:137-139](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/window_helper.go#L137-L139)：
```go
func (self *WindowHelper) SideWindows() []string {
    return []string{"status", "files", "branches", "commits", "stash"}
}
```
这 5 个 Window 与 JumpToBlock 的 5 个数字键一一对应。

---

## 2. ContextMgr.Push 核心逻辑：一切的交汇点

所有焦点切换入口最终都调用 [ContextMgr.Push](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go#L58-L72)。我们在此详细拆解。

### 2.1 Push 的两层逻辑

```
Push(c, opts)
    ↓
1) pushToContextStack(c)   → 操作 Context 栈，返回 (要失活的列表, 要激活的目标)
    ↓
2) 对每个要失活的 context 调用 deactivate()
    ↓
3) 如果有激活目标，调用 Activate(target, opts)
```

### 2.2 pushToContextStack 的分类规则

[context.go:76-130](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go#L76-L130) 的逻辑是 **按目标 Context 的 Kind 分类**：

| 目标 c 的 Kind | 栈操作规则 | 结果栈形态 |
|---|---|---|
| **SIDE_CONTEXT**（files/branches/commits/...） | 清空整个栈，只剩目标 c | 长度恒为 1 |
| **MAIN_CONTEXT**（staging/patchBuilding/...） | 移除栈中所有旧 MAIN_CONTEXT，把 c 追加在末尾 | 栈 = [..., 新的 MAIN] |
| **TEMPORARY_POPUP / PERSISTENT_POPUP** | 若栈顶也是 TEMPORARY_POPUP（且不去 search），先弹出旧的；然后追加 | 栈 = [..., popup] |
| 目标已在栈顶（key 相等） | 什么都不做 | 返回 (空, nil) |
| 栈为空 | 直接追加 | 栈 = [c] |

### 2.3 核心事实：SIDE_CONTEXT 切换总是清空栈

**这是理解"侧边跳转"和"同窗口 Tab 切换"差异的关键**：

当切换到任何一个 SIDE_CONTEXT 时（不论是否在同一个 Window），`pushToContextStack` 会执行：
```go
if c.GetKind() == types.SIDE_CONTEXT {
    contextsToDeactivate = 栈中所有不等于 c 的
    self.ContextStack = []types.Context{c}  // 清空重建
}
```

也就是说：**只要切到侧边窗口，上下文栈长度恒为 1**。不存在"切换同一个 Window 的不同 Tab 保留栈"这种特殊规则——因为 SIDE_CONTEXT 的处理逻辑就是清空整个栈。

### 2.4 deactivate 做了什么

[context.go:153-170](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go#L153-L170)：
```
deactivate(c, opts):
    ├─ 如果进入 search context 则不取消搜索
    ├─ 否则，如果 c 是 MAIN 或 TEMPORARY_POPUP → 取消搜索状态
    ├─ 如果 c 是弹窗（TEMPORARY_POPUP / PERSISTENT_POPUP）
    │     └─ view.Visible = false
    └─ c.HandleFocusLost(opts)
          ├─ 调用所有 onFocusLostFns
          └─ SetHighlight(false)
```

### 2.5 Activate 做了什么

[context.go:172-202](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go#L172-L202)：

```
Activate(c, opts):
    ├─ helpers.Window.SetWindowContext(c)  ← 更新 Window→View 映射
    ├─ helpers.Window.MoveToTopOfWindow(c) ← Z-order 调整
    ├─ oldView.HighlightInactive = true    ← 旧视图显示为"非激活高亮"
    ├─ g.SetCurrentView(viewName)          ← gocui 设置当前焦点视图
    ├─ RenderSearchStatus(c)               ← 渲染搜索状态
    ├─ v.Title = c.Title()                 ← 更新视图标题
    ├─ v.Visible = true
    ├─ 根据视图是否可编辑显示光标
    └─ c.HandleFocus(opts)
          ├─ 调用所有 onFocusFns
          ├─ SetHighlight(true)
          ├─ FocusLine(scrollIntoView=true)
          └─ HandleRenderToMain()   ← 如有需要，刷新主视图
```

**SetWindowContext 的关键细节**（[window_helper.go:51-57](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/window_helper.go#L51-L57)）：
```go
func (self *WindowHelper) SetWindowContext(c types.Context) {
    if c.IsTransient() {
        self.resetWindowContext(c)  // transient context（如 commitFiles）会先清理它在其他 Window 的旧映射
    }
    self.windowViewNameMap().Set(c.GetWindowName(), c.GetViewName())
}
```

### 2.6 ContextMgr.Replace 的使用场景

`Replace` 与 `Push` 的不同在于：**它不按 Kind 分类，总是替换栈顶元素**（[context.go:39-56](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go#L39-L56)）：
```go
func (self *ContextMgr) Replace(c types.Context) {
    if len(self.ContextStack) == 0 {
        self.ContextStack = []types.Context{c}
    } else {
        // 替换最后一个元素
        self.ContextStack = append(self.ContextStack[0:len(self.ContextStack)-1], c)
    }
    self.Activate(c, types.OnFocusOpts{})
}
```

Replace 主要用于以下场景（搜索到的实际使用）：
- [suggestions_controller.go:89](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/suggestions_controller.go#L89)：Suggestions ↔ Prompt 之间相互替换
- [prompt_controller.go:94](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/prompt_controller.go#L94)：Prompt → Suggestions
- [commit_message_controller.go:112](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/commit_message_controller.go#L112) / [commit_description_controller.go:85](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/commit_description_controller.go#L85)：CommitMessage ↔ CommitDescription 之间相互替换

**设计意图**：弹窗类 Context 之间切换时，用户按 Escape 应该回到弹窗的**父级**，而不是上一个弹窗。如果用 Push 会形成 [PopupA, PopupB]，Escape 先回 PopupA 再回主界面；用 Replace 栈始终是 [PopupX]，Escape 直接回主界面。

---

## 3. 三个入口的具体触发链路

### 3.1 场景一：同窗口 Tab 切换（按 ] 键，handleNextTab）

假设当前在 localBranches（Window="branches"，TabIndex=0），按 `]`：

```
步骤 1：键盘事件
────────────────────────────────────────
 gocui MainLoop → processEvent()
   → handleEvent(eventKey) → onKey(ev)
   → execKeybindings(currentView, ev)
   │  （currentView = g.currentView = 之前激活的 "localBranches"）
   │  （Keybinding 是 ViewName="" 的全局绑定，因为 NextTab 绑定在 keybindings.go:313）
   └─ 命中 NextTab，调用 gui.handleNextTab()

步骤 2：handleNextTab 查找目标 Tab
────────────────────────────────────────
 handleNextTab():
   ├─ getTabbedView(gui)
   │    └─ CurrentStatic() = branchesContext（因为栈中只有 SIDE_CONTEXT）
   │       └─ view = "localBranches"（该 context 当前绑定的 view）
   ├─ 遍历所有 context，找到 GetViewName() == "localBranches" 的
   │    （即 branchesContext，它属于 Window="branches"）
   └─ 调用 onViewTabClick("branches", TabIndex+1)
          TabIndex+1 = 1（下一个 tab，即 "remotes"）

步骤 3：onViewTabClick 转为 Context.Push
────────────────────────────────────────
 onViewTabClick("branches", 1):
   ├─ tabs = viewTabMap()["branches"]
   │    = [{Tab:本地分支, View:localBranches},
   │       {Tab:远程,     View:remotes},
   │       {Tab:标签,     View:tags}]
   ├─ viewName = tabs[1].ViewName = "remotes"
   ├─ 用 ViewHelper 查找对应 context：context = RemoteBranchesContext
   │    （注意：RemoteBranchesContext 的 Kind = SIDE_CONTEXT，
   │       GetWindowName() = "branches"）
   └─ ContextMgr.Push(RemoteBranchesContext, {})

步骤 4：ContextMgr.Push → 栈变化（关键！）
────────────────────────────────────────
 pushToContextStack(RemoteBranchesContext):
   ├─ 当前栈 = [LocalBranchesContext]（长度 1）
   ├─ c.GetKind() == SIDE_CONTEXT → 命中分支
   ├─ contextsToDeactivate = 过滤栈中 != 目标的
   │    = [LocalBranchesContext]
   └─ ContextStack = [RemoteBranchesContext]（重建为只有目标）

 → deactivate(LocalBranchesContext):
      └─ HandleFocusLost → SetHighlight(false)

 → Activate(RemoteBranchesContext):
      ├─ SetWindowContext(RemoteBranchesContext)
      │    └─ WindowViewNameMap.Set("branches", "remotes")
      │       （同 Window，只是把 View 从 localBranches 换成 remotes）
      ├─ MoveToTopOfWindow → "remotes" view 排到 "branches" window 的 views 最上层
      ├─ SetCurrentView("remotes")
      └─ HandleFocus → 重绘 branches 面板内容为 remotes，
                      → HandleRenderToMain → 刷新主视图为远程分支信息

步骤 5：processEvent 结束 → flush()
────────────────────────────────────────
 processEvent() 处理完第一个事件后：
   → processRemainingEvents()（批量消费剩余事件，此处为空）
   → contentOnly = false（键盘事件不是 contentOnly）
   └─ flush()
         ↓
      Gui.layout() → 见第 4 节布局计算
         ↓
      遍历所有 View 绘制
         ↓
      Screen.Show()
```

**关键事实**：同 Window 的 Tab 切换，Context 栈也会被清空重建（因为 SIDE_CONTEXT 规则）。WindowViewNameMap 的 Window 名不变，只是 View 名从 localBranches 变为 remotes。

---

### 3.2 场景二：侧边窗口跳转（按数字键 3，从 files → branches）

```
步骤 1：键盘事件 → 命中 JumpToBlock 绑定
────────────────────────────────────────
 JumpToSideWindowController.GetKeybindings()（views.go:220 注册）
   遍历 SideWindows() = ["status", "files", "branches", "commits", "stash"]
   index=2 → window="branches"
   Key=JumpToBlock[2]（默认是 '3'）
   Handler=NoPopupPanel(goToSideWindow("branches"))

步骤 2：goToSideWindow("branches")
────────────────────────────────────────
 goToSideWindow("branches"):
   ├─ CurrentWindow() = "files"（之前的 side window）
   ├─ sideWindowAlreadyActive = false（不是当前 window）
   ├─ GetContextForWindow("branches"):
   │    └─ GetViewNameForWindow("branches") = "localBranches"（根据 WindowViewNameMap）
   │       └─ ContextForView("localBranches") = LocalBranchesContext
   │          （注意：如果当前 branches window 的 Tab 停在 remotes，
   │             这里会拿到 RemotesContext，不是 LocalBranchesContext！）
   └─ ContextMgr.Push(LocalBranchesContext, {})

步骤 3：ContextMgr.Push → 与场景一完全相同的栈逻辑
────────────────────────────────────────
 pushToContextStack(LocalBranchesContext):
   ├─ 栈 = [FilesContext]
   ├─ SIDE_CONTEXT → 清空
   ├─ contextsToDeactivate = [FilesContext]
   └─ ContextStack = [LocalBranchesContext]

 → deactivate(FilesContext) → HandleFocusLost
 → Activate(LocalBranchesContext):
      ├─ SetWindowContext:
      │    └─ WindowViewNameMap.Set("branches", "localBranches")
      │       （注意：Window 名从 "files" 变为 "branches"）
      ├─ MoveToTopOfWindow
      ├─ SetCurrentView("localBranches")
      └─ HandleFocus → FocusLine → HandleRenderToMain

步骤 4：flush() → Gui.layout()
────────────────────────────────────────
 布局计算中会通过 CurrentSide() 获取新的 "branches" 作为 CurrentSideWindow，
 影响侧边面板的权重分配（Accordion 模式）。
 详见第 4 节。
```

### 3.3 场景三：鼠标点击 Tab 标题

```
步骤 1：鼠标事件
────────────────────────────────────────
 onKey(eventMouse):
   ├─ 计算点击坐标 mx, my
   ├─ VisibleViewByPosition(mx, my) → 找到被点击的 view（如 "localBranches"）
   ├─ my == v.y0（点击在标题行） && len(v.Tabs) > 0
   │    ├─ GetClickedTabIndex(mx - v.x0) → 算出 tabIndex（如点击 "Tags" → 2）
   │    ├─ 遍历 tabClickBindings，找到 viewName == "localBranches" 的
   │    │    （在 keybindings.go:372-375 注册：
   │    │      tabClickCallback = onViewTabClick(WindowForView("localBranches"), tabIndex)
   │    │      = onViewTabClick("branches", 2)）
   │    └─ 调用 binding.handler(2) → onViewTabClick("branches", 2)
   └─（与场景一的步骤 3-5 完全一致）

步骤 2-5：与场景一的 3-5 完全相同
```

### 3.4 场景四：h / l 键（NextBlock，从 files → branches）

```
步骤 1：SideWindowController 的 keybinding
────────────────────────────────────────
 SideWindowController.GetKeybindings():
   Keys=NextBlock, Handler=nextSideWindow()

步骤 2：nextSideWindow() 循环查找下一个 window
────────────────────────────────────────
 nextSideWindow():
   ├─ windows = ["status", "files", "branches", "commits", "stash"]
   ├─ currentWindow = "files"
   ├─ 找到 index=1 → newWindow = windows[2] = "branches"
   ├─ GetContextForWindow("branches")
   │    └─（与场景二相同，根据当前 WindowViewNameMap 获取）
   └─ ContextMgr.Push(...) → 与之前所有场景相同的栈逻辑
```

---

## 4. 布局计算：在 Activate 之后由 flush() 统一触发

### 4.1 触发时机

**关键事实**：上述场景中，`ContextMgr.Push` → `Activate` 都是在 `execKeybindings` 中**同步执行**的。`Activate` 并不会直接触发布局计算。布局计算是在 `processEvent` 函数的**末尾**通过 `flush()` 触发的（因为键盘事件不是 contentOnly）：

```
processEvent():
  ├─ select {
  │    case ev := <-g.gEvents:    ← 键盘/鼠标事件走这里
  │    │     contentOnly = false  ← 键盘事件永远不是 contentOnly
  │    │     handleEvent(ev) → onKey → execKeybindings → handler
  │    │                                     ↓
  │    │                              所有 Push/Replace/Pop
  │    │                              所有 SetWindowContext
  │    │                              所有 SetCurrentView
  │    │                              所有 SetViewOnTopOf
  │    │                              （所有 View 内容修改、尺寸修改都未执行！）
  │    └─ case ev := <-g.userEvents: ...
  │
  ├─ processRemainingEvents()     ← 批量消费队列中剩余事件
  ├─ contentOnly = false          ← 键盘事件 → 走完整 flush
  └─ flush()
        ↓
    真正的布局、重绘在这里发生 ★★★
```

### 4.2 Gui.layout 完整流程

入口在 [layout.go:13-207](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/layout.go#L13-L207)。按执行顺序拆解：

```
Gui.layout(g):
  │
  ├─ 0) 首次初始化逻辑（ViewsSetup = false）
  │     └─ SetCurrentView(defaultSideContext.GetViewName())
  │
  ├─ 1) 获取窗口尺寸参数
  │     ├─ width, height = g.Size()
  │     ├─ informationStr = gui.informationStr()
  │     └─ appStatus = View("appStatus").Buffer()
  │
  ├─ 2) getWindowDimensions() ★ 核心布局计算
  │     │  调用 WindowArrangementHelper.GetWindowDimensions(args)
  │     │  （window_arrangement_helper.go:124）
  │     │
  │     ├─ args.CurrentSideWindow 的获取（★ 布局如何感知上下文切换）：
  │     │    └─ args.CurrentSideWindow = self.c.Context().CurrentSide().GetWindowName()
  │     │       （context.go:220-236 向下查找栈中第一个 SIDE_CONTEXT）
  │     │       因为 SIDE_CONTEXT 栈长度恒为 1，所以实际就是栈顶 context.GetWindowName()
  │     │
  │     ├─ 构建 Box 树（ROOT → 中部 + 底部信息栏）
  │     │   中部（sidePanelsDirection = COLUMN 或 ROW if PortraitMode）
  │     │     ├─ sideSection（sidePanelChildren 动态计算）
  │     │     │     sidePanelChildren(args):
  │     │     │     ├─ SCREEN_FULL/HALF →
  │     │     │     │     只有 args.CurrentSideWindow 的 Window 有 Weight=1，
  │     │     │     │     其他 Size=0（隐藏）
  │     │     │     ├─ 高度 >= 28（有 Accordion 空间）→
  │     │     │     │     ExpandFocusedSidePanel=true 时，
  │     │     │     │     CurrentSideWindow 对应 Window 权重设为 ExpandedSidePanelWeight（默认 2）
  │     │     │     │     其他侧边 Window 权重为 1
  │     │     │     │     stash 默认 Size=3，若被聚焦则 Weight=1
  │     │     │     └─ 高度 < 28 →
  │     │     │           CurrentSideWindow = Weight:1，其他 = squashedHeight(1 或 3)
  │     │     │
  │     │     └─ mainSection（mainPanelChildren）
  │     │
  │     └─ boxlayout.ArrangeWindows(root, 0, 0, width, height)
  │          递归计算每个 Window → Dimensions(X0, Y0, X1, Y1)
  │
  ├─ 3) 如果主视图变高，触发 ReadLines（PTY 输出视图读取更多行）
  │
  ├─ 4) 对所有 Flatten() 的 Context 执行 setViewFromDimensions()：
  │     ├─ 如果 Window 不在 dimensions（被隐藏）：
  │     │    SetView(0,0,width,height)，View.Visible=false（后台尺寸便于 Pty 计算）
  │     ├─ 否则：
  │     │    ├─ 检测尺寸变化，标记 mustRerender：
  │     │    │   ├─ OriginY 超出 maxOriginY（ScrollUp 后 + HeightChange 需要重绘）
  │     │    │   ├─ NEEDS_RERENDER_ON_WIDTH_CHANGE 且宽度变化
  │     │    │   └─ NeedsRerenderOnHeightChange 且高度变化
  │     │    ├─ g.SetView(viewName, x0, y0, x1, y1, 0)  ← gocui 设置 View 坐标
  │     │    └─ View.Visible = true
  │     └─ 收集 contextsToRerender
  │
  ├─ 5) transient Context 的可见性控制：
  │     transientContexts（即 IsTransient=true，如 commitFiles）
  │     view.Visible = (GetViewNameForWindow(context.GetWindowName()) == context.GetViewName())
  │     （★ 通过 WindowViewNameMap 判断当前 Window 是否"属于"我，属于则显示）
  │
  ├─ 6) 信息栏内容变更检测（informationStr）
  │
  ├─ 7) onInitialViewsCreation / onInitialViewsCreationForRepo（仅首次）
  │     └─ 最后会 Activate(initialContext)，刷新首个 context 的内容
  │
  ├─ 8) 主视图尺寸变化 → onResize()（重新计算命令日志、子进程输出等）
  │
  ├─ 9) 对 contextsToRerender 中所有 context 调用 HandleRender()
  │
  ├─ 10) ResizeCurrentPopupPanels（当前弹窗大小重算）
  │
  ├─ 11) renderContextOptionsMap（底部快捷键提示）
  │
  └─ 12) afterLayoutFuncs 队列消费：
        FocusLine() 等需要 View 尺寸确定后执行的操作，
        通过 gui.afterLayout(f) 注册，在这里依次执行
```

### 4.3 afterLayoutFuncs 的典型使用：FocusLine

`FocusLine` 是 `HandleFocus` 中调用的函数（列表类 context 在激活时把选中行滚动到可见区域），但它需要知道 View 的精确高度。

在 [list_context_trait.go:FocusLine](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/list_context_trait.go#L190-L253) 中：
```go
func (self *ListContextTrait) FocusLine(scrollIntoView bool) {
    if scrollIntoView {
        self.parentContext.AfterLayout(func() error {
            return self.focusLineAfterLayout(selectedLineIdx)
        })
    }
    self.parentContext.AfterLayout(func() error {
        return self.FocusPoint(selectedLineIdx, ...)  // 设置 cursor/origin
    })
}
```

**执行顺序**：
1. Activate → HandleFocus → FocusLine（只是注册回调，不立即执行）
2. processEvent → flush() → Gui.layout()（View 坐标被 SetView 确定）
3. Gui.layout 末尾 → afterLayoutFuncs 消费（focusLineAfterLayout 根据 View 实际尺寸计算滚动）

---

## 5. 视图重绘的两条路径

### 5.1 路径 A：布局触发的重绘（layout 检测尺寸变化）

在 `Gui.layout` 的 `setViewFromDimensions()` 中检测宽高变化，收集 `contextsToRerender`，最后调用 `HandleRender()`：
- 触发场景：终端 resize、从 FULL/HALF mode 切换、Accordion mode 下侧边焦点变化导致权重变化
- 典型需要重绘的 Context：MAIN 类型的上下文（内容换行依赖宽度、文本对齐依赖高度）

### 5.2 路径 B：数据刷新触发的重绘（postRefreshUpdate）

在 `RefreshHelper.Refresh` 中刷新数据后调用（[refresh_helper.go:785](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L785)）：
```go
refreshView(context):
  OnUIThread(func() error {
    ReApplyFilter(context)      // 重新应用过滤
    PostRefreshUpdate(context)  // ← 核心
    AfterLayout(func() ReApplySearch(context))
  })
```

`postRefreshUpdate` 的逻辑（[view_helpers.go:127-168](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/view_helpers.go#L127-L168)）：
```
postRefreshUpdate(c):
  ├─ c.HandleRender()          ← 无论是否在焦点都重绘内容
  ├─ 当前视图 == c.GetViewName()（即正在看这个）
  │    └─ c.HandleFocus({})   ← 重新聚焦 + 滚动 + 渲染主视图
  └─ 否则
       ├─ c.FocusLine(false)  ← 只定位光标，不滚动
       └─ 若是 NORMAL main view 且对应的 side panel 被刷新
          或正在看弹窗（currentCtx 是静态 ctx 的父级弹窗）
              → 渲染主视图内容
```

### 5.3 HandleRender 的两种实现

**ListContextTrait**（列表类面板）([list_context_trait.go:110-128](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/list_context_trait.go#L110-L128))：
- `renderOnlyVisibleLines = true`（节省内存模式）：只渲染可视范围 N 行
- 否则：渲染全部内容，再设到 View 中
- 最后设置页脚 "x of y"

**SimpleContext**（简单面板）([simple_context.go:66-70](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/simple_context.go#L66-L70))：
- 调用外部注入的 `handleRenderFunc()`（如状态面板、主视图 diff）

---

## 6. 完整链路总览（以"按 ] 切 Tab"为例）

```
 ┌───────────────────────────────────────────────────────────────────┐
 │ 1. 键盘事件（gocui 层）                                            │
 │  MainLoop → processEvent → handleEvent(eventKey) → onKey          │
 │  → execKeybindings(currentView="localBranches")                   │
 │  → 命中全局 NextTab binding → gui.handleNextTab()                  │
 └─────────────────────────────┬─────────────────────────────────────┘
                               │
 ┌─────────────────────────────▼─────────────────────────────────────┐
 │ 2. 查找目标 Tab（应用层）                                          │
 │  handleNextTab()                                                  │
 │  → getTabbedView() → CurrentStatic()="branches" → view="localBranches" │
 │  → onViewTabClick(window="branches", tabIndex=1)                  │
 │  → viewTabMap()["branches"][1] → ViewName="remotes"               │
 │  → ContextForView("remotes") = RemoteBranchesContext              │
 └─────────────────────────────┬─────────────────────────────────────┘
                               │
 ┌─────────────────────────────▼─────────────────────────────────────┐
 │ 3. 操作 Context 栈（ContextMgr）                                   │
 │  ContextMgr.Push(RemoteBranchesContext, {})                       │
 │                                                                   │
 │  3a) pushToContextStack                                           │
 │      c.GetKind()=SIDE_CONTEXT                                     │
 │      ├─ 旧栈 = [LocalBranchesContext]                             │
 │      ├─ contextsToDeactivate = [LocalBranchesContext]             │
 │      └─ 新栈 = [RemoteBranchesContext]  (清空重建)                │
 │                                                                   │
 │  3b) deactivate(LocalBranchesContext)                             │
 │      └─ HandleFocusLost → SetHighlight(false)                     │
 │                                                                   │
 │  3c) Activate(RemoteBranchesContext)                              │
 │      ├─ SetWindowContext → WindowViewNameMap["branches"]="remotes"│
 │      ├─ MoveToTopOfWindow → SetViewOnTopOf("remotes", topView)    │
 │      ├─ SetCurrentView("remotes")                                 │
 │      ├─ v.Title = "远程"                                          │
 │      ├─ v.Visible = true                                          │
 │      └─ HandleFocus                                               │
 │          ├─ SetHighlight(true)                                    │
 │          ├─ FocusLine(scrollIntoView=true)                        │
 │          │   → 注册 afterLayout 回调（不立即执行）                 │
 │          └─ HandleRenderToMain → 渲染 remotes 对应 diff 到主视图  │
 └─────────────────────────────┬─────────────────────────────────────┘
                               │
 ┌─────────────────────────────▼─────────────────────────────────────┐
 │ 4. processEvent 批量消费剩余事件（一般为空）                        │
 │ contentOnly = false（键盘事件始终走完整布局）                       │
 └─────────────────────────────┬─────────────────────────────────────┘
                               │
 ┌─────────────────────────────▼─────────────────────────────────────┐
 │ 5. 布局计算（flush → Gui.layout）★                                │
 │                                                                   │
 │ 5a) getWindowDimensions                                           │
 │     CurrentSide() = RemoteBranchesContext.GetWindowName()="branches" │
 │     sidePanelChildren → "branches" window 权重 = 2（Accordion）    │
 │     boxlayout.ArrangeWindows → 输出所有 Window 的 Dimensions       │
 │                                                                   │
 │ 5b) setViewFromDimensions(遍历所有 Context)                        │
 │     对每个 view: SetView(x0,y0,x1,y1)，Visible=true               │
 │     检测宽高变化 → contextsToRerender（切 Tab 一般不会触发）        │
 │                                                                   │
 │ 5c) transientContexts 可见性控制                                  │
 │                                                                   │
 │ 5d) 主视图尺寸变化 → onResize()（如果有）                          │
 │                                                                   │
 │ 5e) contextsToRerender → HandleRender()（如果有）                 │
 │                                                                   │
 │ 5f) ResizeCurrentPopupPanels / renderContextOptionsMap            │
 │                                                                   │
 │ 5g) ★ afterLayoutFuncs 队列消费：                                 │
 │     → focusLineAfterLayout(1)：根据 View 实际高度                 │
 │       计算 OriginY，滚动到选中行                                  │
 │     → FocusPoint：设置 cursor/origin 到正确位置                   │
 └─────────────────────────────┬─────────────────────────────────────┘
                               │
 ┌─────────────────────────────▼─────────────────────────────────────┐
 │ 6. gocui 绘制所有 View（flush 后续）                              │
 │  对每个 view: drawFrameEdges / drawFrameCorners / drawTitle / ... │
 │  Screen.Show() → 终端输出                                         │
 └───────────────────────────────────────────────────────────────────┘
```

---

## 7. 三个入口的差异对比表

| 维度 | Tab 切换（]） | 侧边跳转（数字键） | 侧边切换（h/l） | 鼠标点 Tab |
|------|--------------|------------------|---------------|-----------|
| **入口函数** | handleNextTab | goToSideWindow | nextSideWindow | onViewTabClick |
| **绑定类型** | 全局 Keybinding（ViewName=""） | 全局 Keybinding（ViewName=""） | 各 Side Context 自身 Keybinding | tabClickBindings（按 View 注册） |
| **目标确定方式** | CurrentStatic() + View.TabIndex | 参数 window 直接指定 | SideWindows[] 顺序查找 | 鼠标点击位置算 tabIndex |
| **Window 是否变更** | 不变（如 branches→branches） | 变（如 files→branches） | 变 | 不变 |
| **View 是否变更** | 变（localBranches→remotes） | 不变或变（取决于目标 Window 上次停在哪个 Tab） | 不变或变 | 变 |
| **栈操作** | Push(SIDE_CONTEXT) → 清空重建 | 完全相同 | 完全相同 | 完全相同 |
| **WindowViewNameMap 变化** | 同 Window，换 View | 换 Window，换 View | 换 Window，换 View | 同 Window，换 View |
| **布局权重是否重算** | 是（flush 后走 Gui.layout，CurrentSideWindow 相同所以通常无变化；但 ScreenMode/FULL/HALF 下仍走布局） | 是（CurrentSideWindow 变了，Accordion 权重重分配） | 是 | 是 |
| **CurrentSideWindow** | 不变（如一直是 "branches"） | 变（如 "files"→"branches"） | 变 | 不变 |

---

## 8. 设计洞察

### 8.1 SIDE_CONTEXT 栈恒为 1 的设计意图

为什么 SIDE_CONTEXT 切换要清空整个栈？
- 侧边窗口之间是**互斥**关系，不存在"进入某侧边窗口后返回另一个"的场景
- Escape 键（Pop）在 SIDE_CONTEXT 上无效（因为栈长度为 1，Pop 直接返回），符合用户预期
- Push 的语义对所有 Side Context 统一，不需要额外判断"同 Window Tab 切换 vs 跨 Window 跳转"

### 8.2 SetWindowContext + MoveToTopOfWindow 分离设计

- **SetWindowContext**：更新**逻辑映射**（Window → ViewName），用于：
  - `GetContextForWindow()` 查找某 Window 当前显示的 Context
  - `transientContexts` 可见性判断（layout.go:147）
- **MoveToTopOfWindow**：更新**物理 Z-order**（gocui views 列表的顺序），用于：
  - 同 Window 中多个 View 重叠时，最上层的 View 可见并接收点击事件

### 8.3 contentOnly 优化为何不适用于键盘事件

键盘事件可能引起布局变化（如 SIDE_CONTEXT 切换 → Accordion 权重变化 → 窗口尺寸变化）。因此键盘事件始终走完整 `flush()`，不能走 `flushContentOnly()`。

只有纯内容变更（如滚动列表、刷新文本）才通过 `OnUIThreadContentOnly` 投递，跳过布局计算。

### 8.4 AfterLayout 解耦"操作意图"与"执行时机"

FocusLine 需要 View 尺寸确定后才能执行，但 Activate（触发 FocusLine）发生在 layout 之前。通过 AfterLayout 队列，实现了：
- 触发方（ContextMgr.Activate）只表达意图（"我要聚焦选中行"）
- 执行方（Gui.layout 末尾）在合适时机统一执行
- 避免"先 SetView 再 Activate"导致的内容焦点错位

---

## 9. 关键文件索引

| 功能 | 文件 |
|------|------|
| 布局入口与视图尺寸应用 | [layout.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/layout.go) |
| Tab 切换 / 点击 + PostRefreshUpdate | [view_helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/view_helpers.go) |
| TabMap 定义（Window 与多 Tab） | [gui.go:836-879](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/gui.go#L836-L879) |
| Keybinding 注册（含 NextTab/TabClick） | [keybindings.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/keybindings.go) |
| 上下文栈与焦点切换（Push/Pop/Replace/Activate） | [context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go) |
| Box 布局计算（CurrentSideWindow → 权重） | [window_arrangement_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/window_arrangement_helper.go) |
| Window→View 映射 + SideWindows 列表 | [window_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/window_helper.go) |
| 数字键 JumpToSideWindow | [jump_to_side_window_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/jump_to_side_window_controller.go) |
| h/l PrevBlock/NextBlock | [side_window_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/side_window_controller.go) |
| Context 基类实现（onFocusFns 等） | [base_context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/base_context.go) |
| 列表类 Context 渲染 + FocusLine/AfterLayout | [list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/list_context_trait.go) |
| 刷新与数据重绘（postRefreshUpdate 调用方） | [refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/refresh_helper.go) |
| gocui 主循环 + processEvent/flush | [gocui/gui.go:714-810](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L714-L810) |
| gocui onKey 鼠标 Tab 点击分发 | [gocui/gui.go:1386-1397](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L1386-L1397) |
| Context 类型定义 | [types/context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/types/context.go) |
