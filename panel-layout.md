# Lazygit 面板布局与切换代码链路分析

本文档严格对齐代码实现，精确区分：
- **按键处理阶段同步执行**的逻辑状态更新
- **HandleFocus 中同步修改**的视图内容（side panel）与**异步投递**的主视图渲染
- **布局阶段（Gui.layout）**才执行的尺寸计算、SetView、滚动定位
- **flush 末尾**才执行的 gocui 绘制（g.draw → Screen.Show）

---

## 1. 事件分发与阶段边界

### 1.1 四个阶段严格分离

| 阶段 | 发生位置 | 做什么 | 不做什么 |
|------|----------|--------|----------|
| **A. 按键处理** | `gocui onKey → execKeybindings → handler` | 修改逻辑状态：Context 栈、WindowViewNameMap、gocui.currentView、View.Title、View.Visible、view.content（side panel 的 SetContent） | 不调用 SetView（坐标未知）、不绘制任何东西 |
| **B. 布局计算** | `flush → Gui.layout` | 调 `GetWindowDimensions` 算所有 Window 坐标；`SetView` 写 View.x0/y0/x1/y1；尺寸变化触发 `HandleRender`；消费 `afterLayoutFuncs`（FocusLine、滚动定位） | 不输出到终端 |
| **C. gocui 绘制** | `flush` 遍历 views 调 `g.draw(v)` | 把 View 里的 cells 写入 tcell Screen | 不改 View 坐标、不改内容 |
| **D. 屏幕输出** | `flush` 末尾 `Screen.Show()` | tcell 同步到终端 | — |
| **E. 后台异步渲染** | ViewBufferManager goroutine（由 HandleRenderToMain 触发） | 生成主视图字符串，完成后 `gui.render()` 投递一个空的 OnUIThread 事件触发下一轮 flush | — |

### 1.2 processEvent 的精确流程

见 [gocui/gui.go:714-810](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L714-L810)：

```
gocui MainLoop
    ↓
processEvent():
    ├─ 加互斥锁 g.Mutex
    │
    ├─ 选一个事件（优先级：gEvents > userEvents > periodicEvents）
    │   ├─ case ev := <-g.gEvents:          ← 键盘/鼠标/Resize
    │   │     contentOnly = false            ★ 键盘事件永远走完整布局
    │   │     handleEvent(ev) → onKey → execKeybindings → handler
    │   │                                     （阶段 A 在这里执行）
    │   │
    │   ├─ case ev := <-g.userEvents:       ← OnUIThread / OnUIThreadContentOnly 投递
    │   │     contentOnly = ev.contentOnly   ★ OnUIThreadContentOnly 可跳过布局
    │   │     task = ev.task
    │   │     ev.f(g) → 执行回调（通常是 SetContent、HandleRender 等）
    │   │
    │   └─ case tick := <-periodicEvents:    ← 60fps 定时刷新
    │         contentOnly = false
    │
    ├─ processRemainingEvents()              ← 批量消费队列剩余事件
    │   （多个 userEvents 打包，最后用累积的 contentOnly 决定 flush 方式）
    │
    ├─ g.stopHandlingUI()                    ← 暂停接收新事件
    │
    ├─ contentOnly 为 true ?
    │   ├─ YES: flushContentOnly()           ← 只重绘被污染 View 的 cells
    │   │         （跳过 Gui.layout，不重算布局）
    │   └─ NO:  flush()                      ← 完整：Gui.layout + 绘制所有 View
    │         （阶段 B + C + D 在这里执行）
    │
    └─ [如果有 task] task.Done()
```

**关键事实**：阶段 A（按键 handler）和阶段 B/C/D（布局+绘制+输出）在同一个 `processEvent` 调用里，**顺序严格串行**，中间不会穿插其他 goroutine（因为持有 g.Mutex）。

---

## 2. 阶段 A：按键处理同步执行的内容

以"按 `]` 从 localBranches 切到 remotes Tab"为例，handler 同步做了什么：

### 2.1 入口调用链（全部同步）

```
execKeybindings("localBranches", KeyNextTab)
    ↓
命中全局 NextTab binding
    ↓
gui.handleNextTab()    [view_helpers.go:77]
    ├─ getTabbedView(gui):
    │    └─ CurrentStatic() = LocalBranchesContext → viewName = "localBranches"
    ├─ 遍历 context.Flatten() 找到 context.GetViewName()=="localBranches"
    │    → 命中 LocalBranchesContext（Window="branches"）
    └─ onViewTabClick("branches", TabIndex+1=1)   [view_helpers.go:60]
          ├─ viewTabMap()["branches"][1] → ViewName="remotes"
          ├─ ContextForView("remotes") → RemoteBranchesContext
          └─ ContextMgr.Push(RemoteBranchesContext, {})    ★★★ 核心
```

### 2.2 ContextMgr.Push 同步做的 4 件事

见 [context.go:58-202](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go#L58-L202)：

#### ① 操作 Context 栈（纯内存，无 IO）
```go
// RemoteBranchesContext.GetKind() == SIDE_CONTEXT
if c.GetKind() == types.SIDE_CONTEXT {
    contextsToDeactivate = 栈中所有 != RemoteBranchesContext 的
    self.ContextStack = []types.Context{RemoteBranchesContext} // 清空重建
}
```

#### ② deactivate 旧 Context（同步修改 View 样式）
```go
// deactivate(LocalBranchesContext):
c.HandleFocusLost(opts)
    └─ [ListContextTrait]
        ├─ GetViewTrait().SetOriginX(0)        ← 同步：View.originX = 0
        ├─ if refreshViewportOnChange: refreshViewport()  ← 同步：View 内容重绘
        └─ [BaseContext via self.Context.HandleFocusLost]
            ├─ GetViewTrait().SetHighlight(false)   ← 同步：View.hasHighlight = false
            └─ 遍历 onFocusLostFns（如果有，同步执行）
```

#### ③ Activate 新 Context（这是内容最多的同步步骤）

```go
Activate(RemoteBranchesContext, opts):
    ├─ helpers.Window.SetWindowContext(RemoteBranchesContext)
    │    └─ self.windowViewNameMap().Set("branches", "remotes")   ← 改 map
    │
    ├─ helpers.Window.MoveToTopOfWindow(RemoteBranchesContext)
    │    └─ g.SetViewOnTopOf("remotes", topView)   ← 改 g.views 列表顺序（Z-order）
    │
    ├─ oldView = g.CurrentView() = "localBranches"
    │   oldView.HighlightInactive = true          ← 同步：View 字段
    │
    ├─ g.SetCurrentView("remotes")                ← 同步：g.currentView = "remotes" view
    │
    ├─ RenderSearchStatus(RemoteBranchesContext)  ← 同步：View 搜索状态
    │
    ├─ v = View("remotes")
    │   v.Title = "远程"                          ← 同步：View 标题
    │   v.Visible = true                          ← 同步：View 可见性
    │
    ├─ 根据视图是否可编辑显示光标                  ← 同步：View.CanEdit / 光标
    │
    └─ RemoteBranchesContext.HandleFocus(opts)    ★★★ 内容最多，见下节
```

#### ④ HandleFocus 的同步 vs 异步边界

见 [list_context_trait.go:91-97](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/list_context_trait.go#L91-L97) 和 [simple_context.go:35-47](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/simple_context.go#L35-L47)：

```
HandleFocus(opts):
    │
    ├─ [同步] self.FocusLine(scrollIntoView)
    │    ├─ self.Context.FocusLine(scrollIntoView)  ← BaseContext 的空实现或 SimpleContext 的空
    │    ├─ inOnSearchSelect = self.inOnSearchSelect
    │    ├─ ★ self.c.AfterLayout(func() { ... })    ← 【不立即执行】写入 channel，等到布局阶段末尾
    │    │    （里面做：FocusPoint → 光标/Origin 计算 → refreshViewport）
    │    └─ [同步] self.setFooter()
    │         └─ GetViewTrait().SetFooter("1 of 42") ← 同步：View 页脚内容
    │
    ├─ [同步] GetViewTrait().SetHighlight(self.list.Len() > 0)
    │    └─ view.hasHighlight = true, view.tainted = true  ← 同步
    │
    └─ [同步] self.Context.HandleFocus(opts)  → 最终走到 SimpleContext.HandleFocus
         ├─ if highlightOnFocus: SetHighlight(true)   ← 同步（side panel 一般走 ListContextTrait 的，这里不会重复）
         ├─ [同步] 遍历 self.onFocusFns → fn(opts)    ← 同步执行各 controller 注册的回调
         └─ [同步 or 异步？] if onRenderToMainFn: self.onRenderToMainFn()
                └─ ★★★ 这里是关键点，见下节
```

### 2.3 HandleRenderToMain：side panel → main view 的内容刷新

`onRenderToMainFn` 由各 side controller 在初始化时通过 `AddOnRenderToMainFn` 注册（比如 files controller、branches controller 等）。

看一个典型例子（[files_controller.go:258](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/files_controller.go#L258) 的 `refreshMain`），最终都会走到：

```
onRenderToMainFn()
    ↓
Gui.refreshMainViews(opts)
    ↓
moveMainContextPairToTop(pair)
    ├─ SetWindowContext(mainContext) / SetWindowContext(secondaryContext)
    ├─ moveMainContextToTop:
    │    ├─ SetWindowContext(context)
    │    ├─ topView = TopViewInWindow(windowName, true)
    │    └─ if topView != view:
    │          ├─ CopyContent(topView, view)   ← 同步：复制 cells 防闪烁
    │          └─ SetViewOnTopOf(view, topView) ← 同步：改 Z-order
    │
Gui.RefreshMainView(opts, context)               ← 对每个 main context
    ├─ view = context.GetView()
    ├─ view.Title = opts.Title                    ← 同步：改标题
    ├─ view.Subtitle = opts.SubTitle              ← 同步：改副标题
    └─ gui.runTaskForView(view, opts.Task)        ★★★ 关键分叉
         ├─ RenderStringTask → newStringTask(view, str)
         ├─ RenderStringWithoutScrollTask → newStringTaskWithoutScroll(view, str)
         ├─ RenderStringWithScrollTask → newStringTaskWithScroll(view, str, x, y)
         ├─ RunCommandTask → newCmdTask(...)
         └─ RunPtyTask → newPtyTask(...)
```

#### `newStringTask` 的真实行为（★ 最容易搞错的时序）

见 [tasks_adapter.go:53-103](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/tasks_adapter.go#L53-L103) 和 [tasks.go:373-434](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/tasks/tasks.go#L373-L434)：

```go
func (gui *Gui) newStringTask(view, str, key) error {
    manager = gui.getManager(view)

    f = func(TaskOpts) error {
        gui.c.ResetViewOrigin(view)     // ← 这个 f 是在后台 goroutine 里执行的！
        gui.c.SetViewContent(view, str) // ← 也是后台 goroutine
        return nil
    }

    manager.NewTask(f, key)  // ★ 启动 goroutine，不等结果
    return nil
}

// tasks.go:373
func (self *ViewBufferManager) NewTask(f func(TaskOpts) error, key string) error {
    gocuiTask := self.newGocuiTask()
    go utils.Safe(func() {      // ★★★ 开 goroutine
        defer completeGocuiTask()
        // ... taskID、stop 处理 ...
        f(TaskOpts{Stop: stop, InitialContentLoaded: completeGocuiTask})
        // f 执行完，SetViewContent 已经把字符串写入 View.textArea
        // 然后 f 的 onRefresh 回调被触发（如果有）
        //   onRefresh = gui.render() → gui.c.OnUIThread(func() error { return nil })
        //     → 投递一个空 userEvent，触发下一轮 flush
        close(notifyStopped)
    })
    return nil  // ★ 立即返回，不等待 goroutine
}
```

**这个时序极其重要，严格写出来是：**

| 步骤 | 线程 | 动作 | 时间点 |
|------|------|------|--------|
| A1 | UI 线程 | `onRenderToMainFn()` 调用 `newStringTask` | t=0 |
| A2 | UI 线程 | `manager.NewTask(f, key)` **启动 goroutine**，**立即返回 nil** | t≈0 |
| A3 | UI 线程 | `HandleFocus` 返回 → `Activate` 返回 → `Push` 返回 → handler 返回 | t≈0 |
| A4 | UI 线程 | `processEvent` 继续 → `processRemainingEvents` → `flush()`（阶段 B/C/D） | t≈0 |
| A5 | UI 线程 | Gui.layout 执行（此时主视图内容还是旧的！） | t≈0 |
| A6 | UI 线程 | g.draw 画所有 View → Screen.Show → 终端显示（主视图还是旧 diff） | t≈0.5ms |
| A7 | UI 线程 | `processEvent` 返回，释放 g.Mutex | t≈1ms |
| **B1** | **后台 goroutine** | **f() 开始执行**：`ResetViewOrigin` + `SetViewContent(view, str)` | t≈1ms（与 A7 并行，或更晚） |
| B2 | 后台 goroutine | manager 的 `onRefresh` 回调被触发 → `gui.render()` | t≈1.1ms |
| B3 | 后台 goroutine | `gui.c.OnUIThread(func() error { return nil })` → 投递 `userEvent{f:空函数}` 到 g.userEvents | t≈1.1ms |
| B4 | 后台 goroutine | `task.Done()` → gocuiTask 完成 | t≈1.2ms |
| **C1** | UI 线程 | MainLoop 下一次 `processEvent` 从 userEvents 拿到空函数，执行 | t≈1.5ms+ |
| C2 | UI 线程 | 空函数什么都不做；但 `contentOnly` 取决于投递方式（这里是普通 OnUIThread，contentOnly=false）→ 走完整 `flush()` | t≈1.5ms |
| C3 | UI 线程 | Gui.layout → g.draw（这次主视图 View.content 已经是新的了）→ Screen.Show | t≈2ms |

**结论：主视图内容刷新需要两次 flush。**
- 第 1 次 flush（紧跟按键）：side panel 的新内容、新标题、新边框颜色被绘制；主视图还是旧内容（因为后台 goroutine 还没跑）
- 第 2 次 flush（由后台 goroutine 完成后投递的空 userEvent 触发）：主视图新内容才被画出来

### 2.4 阶段 A 小结：同步 vs 异步

**同步完成（按键 handler 返回时，已经写到内存）：**
- Context 栈内容（`ContextStack = [新context]`）
- WindowViewNameMap（`"branches" → "remotes"`）
- Z-order（`g.views` 顺序）
- `g.currentView`（当前焦点 View 指针）
- 新 View 的 Title、Visible、HighlightInactive
- side panel 自身的 View 内容（side panel 内容是同步 `HandleRender` / `SetContent`，不经过 ViewBufferManager）
- side panel 的 footer（"1 of 42"）
- side panel 的 originX = 0
- `afterLayoutFuncs` channel 里已经写入 FocusLine 的回调

**异步进行（handler 返回时还没做）：**
- 主视图（main/secondary）的内容（ViewBufferManager 后台 goroutine）
- FocusPoint（具体的光标位置、滚动 Origin）（在 afterLayoutFuncs，等到布局阶段）
- View 坐标（x0/y0/x1/y1）（在 Gui.layout 里 SetView）
- 实际绘制到屏幕（flush → Screen.Show）

---

## 3. 阶段 B：Gui.layout 布局计算

### 3.1 精确执行顺序

见 [layout.go:13-207](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/layout.go#L13-L207)：

```
Gui.layout(g):
    │
    ├─ [0] 首次初始化（ViewsSetup = false）
    │    └─ SetCurrentView(defaultSideContext.GetViewName())
    │
    ├─ [1] 收集输入参数
    │    ├─ width, height = g.Size()              ← tcell Screen 的尺寸
    │    ├─ informationStr = gui.informationStr()  ← 底部模式/捐赠信息
    │    └─ appStatus = View("appStatus").Buffer() ← 加载状态
    │
    ├─ [2] ★ 算所有 Window 坐标（纯内存计算，不改任何 View）
    │    │  helpers.WindowArrangement.GetWindowDimensions(informationStr, appStatus)
    │    │  (window_arrangement_helper.go:124)
    │    │
    │    ├─ args.CurrentSideWindow = ContextMgr.CurrentSide().GetWindowName()
    │    │    ← 从 Context 栈拿第一个 SIDE_CONTEXT 的窗口名（现在是 "branches"）
    │    │
    │    ├─ 构建 Box 树（ROOT → 中部 + 底部信息栏）
    │    │    sidePanelChildren(args):
    │    │      └─ 如果 Accordion 模式：
    │    │           "branches" 的 Weight = ExpandedSidePanelWeight（默认 2）
    │    │           其他侧边 Window Weight = 1
    │    │
    │    └─ boxlayout.ArrangeWindows(root, 0, 0, width, height)
    │         返回 map[string]Dimensions{
    │           "status": {1,1,40,3},
    │           "files":  {1,4,40,20},
    │           "branches": {1,21,40,40},  ← Accordion 模式下 branches 更高
    │           ...
    │         }
    │
    ├─ [3] 主视图变高触发 ReadLines（PTY 输出视图读取更多行）
    │
    ├─ [4] 遍历 ContextManager.Flatten() 的所有 Context，逐个 setViewFromDimensions
    │    │  (layout.go:77-117)
    │    │
    │    ├─ if Window 不在 dimensions（被 FULL/HALF 模式隐藏）:
    │    │    └─ SetView(name, 0, 0, width, height)  ← 后台尺寸用于 Pty
    │    │       View.Visible = false
    │    │
    │    └─ else:
    │         ├─ oldInnerWidth, oldInnerHeight = v.InnerWidth(), v.InnerHeight()
    │         ├─ g.SetView(viewName, x0, y0, x1, y1, 0)   ★★★
    │         │    ← 写 v.x0/v.y0/v.x1/v.y1 + v.tainted = true + markViewsBelowAsTainted
    │         │    （见 gocui/gui.go:309 SetView 实现）
    │         ├─ newInnerWidth, newInnerHeight = v.InnerWidth(), v.InnerHeight()
    │         ├─ View.Visible = true
    │         ├─ if OriginY > maxOriginY（尺寸变小导致滚动超界）: mustRerender = true
    │         ├─ if c.NeedsRerenderOnWidthChange && oldInnerWidth≠newInnerWidth: mustRerender = true
    │         ├─ if c.NeedsRerenderOnHeightChange && oldInnerHeight≠newInnerHeight: mustRerender = true
    │         └─ if mustRerender: contextsToRerender = append(..., c)
    │
    ├─ [5] transient Context 的可见性控制
    │    for c := range transientContexts:
    │        view.Visible = GetViewNameForWindow(c.GetWindowName()) == c.GetViewName()
    │        （根据 WindowViewNameMap 判断"这个 View 是不是当前 Window 的顶视图"）
    │
    ├─ [6] 信息栏内容变更检测（informationStr vs 上次）
    │
    ├─ [7] 首次初始化的额外处理
    │
    ├─ [8] 主视图尺寸变化 → onResize()（重新计算 Pty 输出行数等）
    │
    ├─ [9] 对 contextsToRerender 调 c.HandleRender()
    │
    ├─ [10] ResizeCurrentPopupPanels
    │
    ├─ [11] renderContextOptionsMap（底部快捷键提示）
    │
    └─ [12] ★ 消费 afterLayoutFuncs（FocusLine 在这里执行！）
         for {
             select {
             case f := <-gui.afterLayoutFuncs:
                 f()   ← FocusLine 的回调在这里执行
             default:
                 return
             }
         }
```

### 3.2 afterLayoutFuncs 的典型内容（FocusLine）

以 RemoteBranchesContext（ListContextTrait）为例，它在阶段 A 的 HandleFocus 里注册了：

```go
self.c.AfterLayout(func() error {
    oldOrigin, _ := self.GetViewTrait().ViewPortYBounds()

    // ★ 此时 View 已经在步骤 [4] 里 SetView 过，有精确的 x0/y0/x1/y1
    self.GetViewTrait().FocusPoint(
        self.ModelIndexToViewIndex(self.list.GetSelectedLineIdx()),
        scrollIntoView=true)

    self.GetView().SetNearestSearchPosition()

    // range select 处理...

    if self.refreshViewportOnChange {
        self.refreshViewport()         // 同步：重绘可视区域
    } else if self.renderOnlyVisibleLines {
        newOrigin, _ := self.GetViewTrait().ViewPortYBounds()
        if oldOrigin != newOrigin || self.needRerenderVisibleLines {
            self.refreshViewport()     // 同步：重绘可视区域
        }
    }
    return nil
})
```

**为什么 FocusPoint 必须在布局之后？**
- `FocusPoint` 需要知道 View 的 `InnerHeight()` 才能决定"选中行是否在可视范围外、需要滚多少"
- 在阶段 A 按键处理时，Gui.layout 还没跑，View 坐标还是旧的（甚至可能因为 Accordion 模式导致高度从 20 变成 40）
- 所以必须用 AfterLayout 延后到 SetView 之后

---

## 4. 阶段 C/D：gocui 绘制与屏幕输出

见 [gocui/gui.go:1143-1226](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L1143-L1226)：

```
flush():
    ├─ [如果屏幕尺寸变化了] markAllViewsAsTainted()
    │
    ├─ GuiManagers.Layout(g)  → 实际就是 Gui.layout（阶段 B 在这里执行，前面已讲）
    │
    ├─ 遍历 g.views（按 Z-order）：
    │    ├─ if !v.Visible: continue
    │    ├─ if !v.tainted: continue      ← ★ 性能优化：没被污染的 View 跳过
    │    └─ g.draw(v)
    │         ├─ clearViewLines(v)     ← 清空 Screen 上对应矩形区域的 cells
    │         ├─ v.draw(interruptedRuneOpacities)
    │         │    ├─ drawFrameEdges()   ← 边框 + 滚动条（按 Focused / Highlighted 选色）
    │         │    ├─ drawFrameCorners() ← 四角
    │         │    └─ 逐行绘制 textArea 中的 cells（考虑 OriginX/OriginY 偏移）
    │         └─ v.tainted = false
    │
    └─ Screen.Show()                 ← tcell 把缓存的 cells 同步到终端
```

**v.tainted 的设置时机（决定哪些 View 要重绘）：**
- `SetView`（坐标变了）: [gocui/gui.go:309](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L309) 设置 `v.tainted = true` + `markViewsBelowAsTainted`
- `SetContent` / `Write`（内容变了）: [gocui/view.go:200](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/view.go#L200) 设置 `v.tainted = true`
- `SetHighlight` / `SetOrigin` / `SetCursor`（样式变了）: 均设置 `v.tainted = true`

---

## 5. 完整时序线：按 `]` 从 localBranches → remotes

下面用精确的时间顺序列出每一行代码在哪个阶段执行：

```
═══════════════════════════════════════════════════════════════════════════
 时间线（单线程 UI goroutine + 1 个后台 goroutine）
═══════════════════════════════════════════════════════════════════════════

 ┌── UI 线程（持有 g.Mutex）─────────────────────────────────────────────┐
 │                                                                       │
 │  t=0   processEvent() 从 gEvents 取到 Key(']')                         │
 │          contentOnly = false                                           │
 │                                                                       │
 │  t≈0   handleEvent(ev) → onKey → execKeybindings                      │
 │          ├─ 命中全局 NextTab binding                                   │
 │          └─ gui.handleNextTab()                                        │
 │                                                                       │
 │  t≈0   ├─ getTabbedView → LocalBranchesContext                         │
 │        ├─ onViewTabClick("branches", 1)                                │
 │        ├─ viewTabMap()["branches"][1] → ViewName="remotes"            │
 │        └─ ContextMgr.Push(RemoteBranchesContext, {})                   │
 │                                                                       │
 │  t≈0      ├─ pushToContextStack:                                       │
 │  t≈0      │   SIDE_CONTEXT → 清空，栈变为 [RemoteBranchesContext]      │
 │  t≈0      │   contextsToDeactivate = [LocalBranchesContext]            │
 │  t≈0      ├─ deactivate(LocalBranchesContext)                          │
 │  t≈0      │   └─ HandleFocusLost →                                     │
 │  t≈0      │        SetOriginX(0)                                      │
 │  t≈0      │        SetHighlight(false)                                │
 │  t≈0      └─ Activate(RemoteBranchesContext)                           │
 │  t≈0           ├─ SetWindowContext → WindowViewNameMap["branches"]="remotes"│
 │  t≈0           ├─ MoveToTopOfWindow → SetViewOnTopOf("remotes", ...) │
 │  t≈0           ├─ oldView("localBranches").HighlightInactive = true   │
 │  t≈0           ├─ SetCurrentView("remotes")                            │
 │  t≈0           ├─ RenderSearchStatus                                   │
 │  t≈0           ├─ v.Title = "远程"                                     │
 │  t≈0           ├─ v.Visible = true                                     │
 │  t≈0           └─ RemoteBranchesContext.HandleFocus({ScrollIntoView:true})│
 │  t≈0                ├─ FocusLine(true)                                 │
 │  t≈0                │  ├─ afterLayoutFuncs <- func() { // 还不执行 } │
 │  t≈0                │  │    （FocusPoint、refreshViewport 都在这里面）│
 │  t≈0                │  └─ setFooter() → SetFooter("1 of 42")          │
 │  t≈0                ├─ SetHighlight(true)                              │
 │  t≈0                └─ self.Context.HandleFocus                        │
 │  t≈0                     ├─ 遍历 onFocusFns → fn(opts)                 │
 │  t≈0                     └─ onRenderToMainFn()                         │
 │  t≈0                          └─ refreshMainViews                      │
 │  t≈0                               ├─ moveMainContextPairToTop         │
 │  t≈0                               │   ├─ SetWindowContext              │
 │  t≈0                               │   └─ CopyContent + SetViewOnTopOf │
 │  t≈0                               └─ RefreshMainView(NormalContext)   │
 │  t≈0                                    ├─ view.Title = diff 标题     │
 │  t≈0                                    └─ runTaskForView → newStringTask│
 │  t≈0                                         └─ manager.NewTask(f, key)│
 │  t≈0                                              └─ go utils.Safe(...) ★ 启后台 goroutine │
 │  t≈0                                              └─ 立即返回 nil       │
 │                                                                       │
 │  t≈0   handler 返回，execKeybindings 返回                               │
 │                                                                       │
 │  t≈0   processRemainingEvents() → userEvents 为空                      │
 │                                                                       │
 │  t≈0   g.stopHandlingUI()                                              │
 │                                                                       │
 │  t≈0   contentOnly=false → flush()                                     │
 │        ┌──────────────────────────────────────────────────────────┐    │
 │        │ 阶段 B/C/D 开始                                            │    │
 │        ├─ Gui.layout()                                             │    │
 │        │  ├─ [1] width, height, informationStr, appStatus          │    │
 │        │  ├─ [2] GetWindowDimensions                               │    │
 │        │  │    CurrentSideWindow = "branches"                      │    │
 │        │  │    Accordion → branches Weight=2，其他=1               │    │
 │        │  │    boxlayout 计算所有 Window → Dimensions               │    │
 │        │  ├─ [4] 对每个 context:                                    │    │
 │        │  │    ├─ SetView("remotes", x0, y0, x1, y1, 0)            │    │
 │        │  │    │  → v.tainted = true                                │    │
 │        │  │    └─ SetView("main", x0, y0, x1, y1, 0)               │    │
 │        │  │       → v.tainted = true                               │    │
 │        │  ├─ [9] 对 contextsToRerender: HandleRender()             │    │
 │        │  ├─ [11] renderContextOptionsMap（底部快捷键）             │    │
 │        │  └─ [12] 消费 afterLayoutFuncs:                            │    │
 │        │       └─ FocusLine 的回调执行：                             │    │
 │        │            FocusPoint(...) → SetCursor + SetOrigin         │    │
 │        │            if 需要：refreshViewport() → SetViewPortContent │    │
 │        │                                                            │    │
 │        ├─ 遍历 views: g.draw(v)                                     │    │
 │        │  draw("remotes") → 新内容、激活边框色（侧栏已更新）          │    │
 │        │  draw("main")    → 旧 diff 内容（后台 goroutine 还没跑！）  │    │
 │        │  draw("status"), draw("files"), ...                        │    │
 │        │                                                            │    │
 │        └─ Screen.Show() → 终端显示                                    │    │
 │                                                                       │
 │  t≈1ms task.Done()（gocuiTask，因为按键事件自带 1 个 task）            │
 │  t≈1ms processEvent() 返回，释放 g.Mutex                              │
 │                                                                       │
 └───────────────────────────────────────────────────────────────────────┘
                                     │
                                     │ UI 线程释放锁后，后台 goroutine 才可能拿到 View 锁
                                     ▼
 ┌── 后台 goroutine（ViewBufferManager 启动的）────────────────────────────┐
 │                                                                       │
 │  t≈1ms f(TaskOpts{Stop: stop, InitialContentLoaded: completeGocuiTask})│
 │        ├─ ResetViewOrigin("main") → SetCursor(0,0) + SetOrigin(0,0)  │
 │        │   → v.tainted = true                                         │
 │        └─ SetViewContent("main", diffString)                          │
 │            → v.SetContent(str) → 写 textArea                         │
 │            → v.tainted = true                                         │
 │                                                                       │
 │  t≈1.1ms onRefresh 回调（ViewBufferManager 创建时注册的）               │
 │        gui.render()                                                   │
 │          └─ gui.c.OnUIThread(func() error { return nil })             │
 │               └─ gui.g.Update(func(*Gui) error { return nil })        │
 │                    ├─ task = g.NewTask()                              │
 │                    └─ go g.updateAsyncAux(f, task)                    │
 │                         └─ g.userEvents <- userEvent{f:空, task:task} │
 │                            ★ 投递到 userEvents 通道                    │
 │                                                                       │
 │  t≈1.2ms close(notifyStopped)                                         │
 │        completeGocuiTask() → gocuiTask.Done()                         │
 │        goroutine 退出                                                  │
 └───────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
 ┌── UI 线程（下一次 processEvent）──────────────────────────────────────┐
 │                                                                       │
 │  t≈1.5ms processEvent()                                               │
 │          从 userEvents 取出空 userEvent                                │
 │          contentOnly = false（因为是 Update 不是 UpdateContentOnly）   │
 │          ev.f(g) → 空函数，什么都不做                                   │
 │          processRemainingEvents() → 没有更多 userEvents               │
 │          flush() → Gui.layout → g.draw("main")（这次 main View        │
 │                                          textArea 已经是新 diff 了！）│
 │          Screen.Show() → 主视图新内容才显示出来                         │
 │                                                                       │
 └───────────────────────────────────────────────────────────────────────┘
```

---

## 6. 三条入口的时序差异对比

| 操作 | side panel 内容 | side panel 坐标/滚动 | main view 内容 | 何时可见 |
|------|----------------|---------------------|----------------|----------|
| **按 `]`（同 Window Tab 切换，localBranches → remotes）** | 第 1 次 flush 绘制（side panel 同步 SetContent） | 第 1 次 flush（SetView + afterLayout FocusPoint） | 第 2 次 flush（后台 goroutine + 空 userEvent 触发） | side 切换即时可见，main 稍晚 |
| **按数字键 3（files → branches，跨 Window）** | 第 1 次 flush（side 同步） | 第 1 次 flush（Accordion 模式下 branches 更高 → SetView 尺寸 + FocusPoint 适配新高度） | 第 2 次 flush | side 即时，main 稍晚 |
| **鼠标点 Tab 标题（localBranches → remotes，同 Window）** | 第 1 次 flush | 第 1 次 flush | 第 2 次 flush | 与按键完全相同 |
| **按 `h/l`（PrevBlock/NextBlock，files → branches）** | 第 1 次 flush | 第 1 次 flush（Accordion 权重重算） | 第 2 次 flush | 与数字键完全相同 |
| **按 `j`（列表下移动，纯内容变化，不切 Context）** | 第 1 次 flush（side 同步 SetOrigin + SetCursor） | 不涉及 | 不变 | 即时（只 1 次 flush） |
| **`git status` 外部变化触发刷新** | postRefreshUpdate → HandleRender → 同步 SetContent | 如果是当前视图，HandleFocus → afterLayout FocusPoint | ViewBufferManager 异步 → 第 2 次 flush | side 即时，main 稍晚 |

---

## 7. 关键设计洞察

### 7.1 "两次 flush" 的设计取舍

side panel 的内容直接同步写入 View（`SetContent`），因为 side panel 是简单列表，渲染成本低；
main view 的内容通过 ViewBufferManager 的后台 goroutine 生成，因为 diff/merge/patch 可能非常大：
- 防止 UI 线程被长字符串渲染阻塞
- 支持取消（用户切文件时，`stopCurrentTask()` 中断旧的渲染）
- 支持"相同内容不重复渲染"（通过 taskKey 去重）

副作用：用户每次切 side panel 的选中项，主视图都要多一帧才更新。这个延迟在实现上用 `CopyContent(topView, view)`（[main_panels.go:50](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/main_panels.go#L50)）来减少视觉闪烁——在新内容渲染完之前，先把旧 View 的内容复制到新 View 上。

### 7.2 AfterLayout：把"尺寸相关操作"从"尺寸未知阶段"解耦

FocusLine 的逻辑需要知道 View 的 InnerHeight，但是它在 HandleFocus（阶段 A）中被触发，此时 Gui.layout 还没跑，View 坐标还是旧的。

解决方案：
- 阶段 A（HandleFocus）只表达意图：`AfterLayout(func FocusPoint(...))`
- 阶段 B（Gui.layout 末尾）在 SetView 之后统一执行队列

这层解耦避免了"如果 Accordion 模式导致 View 高度变化，FocusLine 算出来的 OriginY 就错了"的问题。

### 7.3 taint 机制：增量绘制

每个 View 有一个 `tainted` bool 字段，任何改坐标/内容/样式的操作都会置 true。`flush()` 时只重绘被污染的 View 和它下方重叠的区域（`markViewsBelowAsTainted`），避免 60fps 下每一帧都重绘全部。

### 7.4 contentOnly：跳过布局的快速路径

对于纯内容变化（例如滚动列表、刷新文本），用 `OnUIThreadContentOnly` 投递事件：
- `processEvent` 的 contentOnly 累积为 true
- 走 `flushContentOnly()`：不调用 GuiManagers.Layout（即跳过 Gui.layout），直接遍历 tainted views 调 g.draw

键盘事件永远不会走这条路径，因为它可能引起 Context 变化 → Accordion 重算 → Window 尺寸变化。

---

## 8. 关键文件索引

| 功能 | 文件 |
|------|------|
| processEvent / flush 时序总控 | [gocui/gui.go:714-810](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L714-L810) |
| flush 与 g.draw 绘制 | [gocui/gui.go:1143-1226](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L1143-L1226) |
| SetView（改坐标 + tainted） | [gocui/gui.go:309-358](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L309-L358) |
| View.SetContent / taint 机制 | [gocui/view.go:198-244](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/view.go#L198-L244) |
| Gui.layout 入口 | [layout.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/layout.go) |
| Tab 切换 / onViewTabClick / postRefreshUpdate | [view_helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/view_helpers.go) |
| ContextMgr.Push / Activate / deactivate | [context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context.go) |
| ListContextTrait.HandleFocus / FocusLine / AfterLayout 使用 | [list_context_trait.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/list_context_trait.go) |
| SimpleContext.HandleFocus / onRenderToMainFn 调用 | [simple_context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/simple_context.go) |
| BaseContext（onFocusFns / onRenderToMainFn 存储） | [base_context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/context/base_context.go) |
| Box 布局（CurrentSideWindow → Accordion 权重） | [window_arrangement_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/window_arrangement_helper.go) |
| Window→View 映射 + SideWindows | [window_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/helpers/window_helper.go) |
| RefreshMainView + runTaskForView + moveMainContextPairToTop | [main_panels.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/main_panels.go) |
| newStringTask / ViewBufferManager 适配层 | [tasks_adapter.go:53-140](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/tasks_adapter.go#L53-L140) |
| ViewBufferManager.NewTask（后台 goroutine + onRefresh 触发 gui.render） | [tasks/tasks.go:373-434](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/tasks/tasks.go#L373-L434) |
| gui.render()（投递空 OnUIThread 触发下一轮 flush） | [view_helpers.go:120-122](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/view_helpers.go#L120-L122) |
| OnUIThread / onUIThread（投递 userEvent） | [gui_common.go:119-125](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/gui_common.go#L119-L125) / [gui.go:1185-1195](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/gui.go#L1185-L1195) |
| gocui Update / UpdateContentOnly（投递 userEvent） | [gocui/gui.go:619-642](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gocui/gui.go#L619-L642) |
| JumpToSideWindow（数字键入口） | [jump_to_side_window_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/jump_to_side_window_controller.go) |
| PrevBlock/NextBlock（h/l 入口） | [side_window_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/30-lazygit/pkg/gui/controllers/side_window_controller.go) |
