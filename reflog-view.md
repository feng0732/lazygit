# Reflog 视图代码路径详解

本文档从代码实现角度梳理 lazygit 中 reflog 视图的完整调用链路，覆盖数据读取、列表组织和跳转动作三大核心环节。所有代码引用均为仓库相对路径。

---

## 一、整体架构分层

```
┌─────────────────────────────────────────┐
│  1. 数据读取层 (Git Loader)             │
│     ReflogCommitLoader                  │
├─────────────────────────────────────────┤
│  2. 数据模型层 (State/Model)            │
│     FilteredReflogCommits / ReflogCommits│
├─────────────────────────────────────────┤
│  3. 刷新调度层 (Refresh Helper)         │
│     refreshReflogCommits()              │
├─────────────────────────────────────────┤
│  4. 列表展示层 (Context + Presentation) │
│     ReflogCommitsContext + Display      │
├─────────────────────────────────────────┤
│  5. 动作控制层 (Controller + Helper)    │
│     Controllers + RefsHelper            │
└─────────────────────────────────────────┘
```

---

## 二、数据读取：从 Git 命令到 Commit 对象

### 2.1 核心入口：ReflogCommitLoader

文件：`pkg/commands/git_commands/reflog_commit_loader.go`

入口函数 `GetReflogCommits()` (第 27-63 行)：

```go
func (self *ReflogCommitLoader) GetReflogCommits(
    hashPool *utils.StringPool,
    lastReflogCommit *models.Commit,
    filterPath string,
    filterAuthor string,
) ([]*models.Commit, bool, error)
```

**Git 命令构造** (第 28-34 行)：

```go
cmdArgs := NewGitCmd("log").
    Config("log.showSignature=false").
    Arg("-g").                                          // --walk-reflogs：遍历 reflog
    Arg("--format=+%H%x00%ct%x00%gs%x00%P").           // 自定义输出格式
    ArgIf(filterAuthor != "", "--author="+filterAuthor).
    ArgIf(filterPath != "", "--follow", "--name-status", "--", filterPath).
    ToArgv()
```

格式字段说明：
| 占位符 | 含义 |
|--------|------|
| `%H` | 完整 commit hash |
| `%ct` | **提交者时间戳（committer date）**，Unix 秒。注意：是 commit 本身的 committer 时间，不是 reflog 条目自身的操作时间 |
| `%gs` | reflog 主题消息（如 "checkout: moving from A to B"） |
| `%P` | 父 commit hash（空格分隔） |
| `%x00` | 空字符（字段分隔符） |
| 前缀 `+` | 自定义起始标记，用于在流式输出中识别新条目起点 |

### 2.2 增量加载优化

`GetReflogCommits` 是 lazygit **唯一支持增量加载**的面板数据读取函数。核心逻辑 (第 50-54 行)：

```go
if lastReflogCommit != nil && self.sameReflogCommit(commit, lastReflogCommit) {
    onlyObtainedNewReflogCommits = true
    return nil, true  // 遇到已知条目，停止读取，返回增量
}
```

断点识别 `sameReflogCommit()` (第 65-67 行) 用**三元组**判断：

```go
return a.Hash() == b.Hash() &&
       a.UnixTimestamp == b.UnixTimestamp &&
       a.Name == b.Name
```

**时间戳的真实含义与碰撞边界**：

源码注释（第 46-49 行）明确指出：

> the unix timestamp here is the timestamp of the COMMIT, not the reflog entry itself,
> so two consecutive reflog entries may have both the same hash and therefore same timestamp.
> We use the reflog message to disambiguate, and fingers crossed that we never see the same
> of those twice in a row. Reason being that it would mean we'd be erroneously exiting early.

关键事实：
1. `%ct` 产出的是 **commit 的提交者时间戳（committer date）**，不是 reflog 条目自身的操作时间。因此当同一个 commit 连续出现在多条 reflog 记录中时，hash 和时间戳**完全相同**，三元组中只有 `Name`（reflog 消息，即 `%gs`）能消歧。
2. 存在**理论碰撞边界**：如果同一个 commit 连续产生两条 `%gs` 消息完全相同的 reflog 记录（例如快速连续执行两次完全一样的操作），三元组会错误命中断点，导致提前终止读取、丢失后续条目。代码用 "fingers crossed" 承认了这一风险，但在实践中极为罕见。

### 2.3 行解析：parseLine()

文件：`pkg/commands/git_commands/reflog_commit_loader.go` 第 69-90 行

按 `\x00` 分割成 4 个字段，构造 `models.Commit`，关键标记：

```go
models.NewCommit(hashPool, models.NewCommitOpts{
    Hash:          fields[0],
    Name:          fields[2],     // 存的是 reflog 消息，不是 commit message
    UnixTimestamp: int64(unixTimestamp),
    Status:        models.StatusReflog,   // ← 重要：标记为 reflog 类型
    Parents:       parents,
})
```

`Status == StatusReflog` 的**真实用途**仅有一处：`pkg/gui/presentation/commits.go` 第 499 行的 hash 颜色 switch 分支，将 reflog 条目的 hash 映射为蓝色（`style.FgBlue`）。它**并不**用于控制"子提交是否显示分支头标记"——那是由 `ReflogCommitsContext.ShowBranchHeadsInSubCommits()` 直接返回 `false` 实现的。

### 2.4 通用行处理：loadCommits()

文件：`pkg/commands/git_commands/commit_loading_shared.go` 第 11-74 行

`loadCommits` 是 commit_loader 和 reflog_commit_loader 共用的流式解析框架：

- 以 `+` 开头的行 → 识别为新条目起点，调用 `parseLogLine` 解析
- 非 `+` 开头且启用了路径过滤 → 当作 `--name-status` 输出的文件路径处理
- `parseLogLine` 返回 `stop=true` 时 → 立即终止读取（增量加载的早停机制）

---

## 三、数据模型：两份 Reflog 数据

### 3.1 双模型设计

文件：`pkg/gui/types/common.go` 第 308-315 行

```go
// FilteredReflogCommits：显示在 reflog 面板中的条目
// 过滤模式下只包含匹配路径/作者的条目
FilteredReflogCommits []*models.Commit

// ReflogCommits：reflog 完整全集
// - 给分支面板用：按 recency（最近使用）排序
// - 给 undo 功能用：追溯历史操作
// 非过滤模式下，与 FilteredReflogCommits 指向同一份切片
ReflogCommits []*models.Commit
```

**设计意图：**
- 渲染用 `FilteredReflogCommits`，受过滤/搜索影响
- 业务逻辑用 `ReflogCommits`，始终保持完整，不受过滤影响
- 非过滤模式下指针共享，零拷贝

---

## 四、刷新调度：何时以及如何更新数据

### 4.1 刷新触发源

文件：`pkg/gui/controllers/helpers/refresh_helper.go`

**全量刷新入口** `Refresh()` (第 63-237 行)：

- `types.REFLOG` 属于默认刷新 scope (第 96 行)
- 分支排序为 `recency` 模式时：reflog 和 branches 串行刷新 (第 141-144 行)，因为分支排序依赖 reflog 数据
- 其他排序模式时：reflog 独立异步刷新 (第 151 行)

**启动两阶段优化** `refreshReflogCommitsConsideringStartup()` (第 283-296 行)：

- **INITIAL 阶段**：先不加载 reflog，后台异步加载，避免启动阻塞
- 加载完成后刷新分支排序，再切到 **COMPLETE 阶段**

### 4.2 核心刷新逻辑：refreshReflogCommits()

文件：`pkg/gui/controllers/helpers/refresh_helper.go` 第 648-687 行

```go
func (self *RefreshHelper) refreshReflogCommits() error {
    model := self.c.Model()

    refresh := func(stateCommits *[]*models.Commit, filterPath string, filterAuthor string) error {
        // 无过滤 + 已有数据 → 取第一条做断点，增量加载
        var lastReflogCommit *models.Commit
        if filterPath == "" && filterAuthor == "" && len(*stateCommits) > 0 {
            lastReflogCommit = (*stateCommits)[0]
        }

        commits, onlyObtainedNewReflogCommits, err :=
            self.c.Git().Loaders.ReflogCommitLoader.
                GetReflogCommits(self.c.Model().HashPool,
                    lastReflogCommit, filterPath, filterAuthor)

        if onlyObtainedNewReflogCommits {
            // 增量：新条目拼到前面，旧的接在后面
            *stateCommits = append(commits, *stateCommits...)
        } else {
            // 全量：直接替换
            *stateCommits = commits
        }
        return nil
    }

    // 第一步：刷新完整全集 ReflogCommits
    refresh(&model.ReflogCommits, "", "")

    // 第二步：刷新过滤子集 FilteredReflogCommits
    if self.c.Modes().Filtering.Active() {
        refresh(&model.FilteredReflogCommits,
            self.c.Modes().Filtering.GetPath(),
            self.c.Modes().Filtering.GetAuthor())
    } else {
        // 无过滤时共享同一份切片（零拷贝）
        model.FilteredReflogCommits = model.ReflogCommits
    }

    self.refreshView(self.c.Contexts().ReflogCommits)
    return nil
}
```

**关键点：**
1. 只有**无过滤**模式才能增量加载（过滤后第一条不是全局第一条）
2. 数据顺序是**新的在前，旧的在后**（reflog 本身就是反向时间序）
3. `refreshView()` 会切到 UI 线程：重应用过滤器 → 重渲染 → 重应用搜索

---

## 五、列表组织：Context + 渲染

### 5.1 ReflogCommitsContext 结构

文件：`pkg/gui/context/reflog_commits_context.go`

```go
type ReflogCommitsContext struct {
    *FilteredListViewModel[*models.Commit]   // 数据模型 + 过滤/搜索
    *ListContextTrait                        // 列表渲染 + 光标 + 滚动
}
```

实现了两个接口：
- `types.IListContext` — 列表交互
- `types.DiffableContext` — 支持 diff 模式

### 5.2 ViewModel 数据绑定

构造函数 `NewReflogCommitsContext()` 第 22-27 行：

```go
viewModel := NewFilteredListViewModel(
    func() []*models.Commit { return c.Model().FilteredReflogCommits },
    func(commit *models.Commit) []string {
        return []string{commit.ShortHash(), commit.Name}  // 过滤匹配字段
    },
)
```

- 数据来源是 `FilteredReflogCommits`（过滤后的数据）
- 过滤器匹配 `ShortHash` 和 `Name`（reflog 消息）两个字段

### 5.3 显示字符串生成

文件：`pkg/gui/presentation/reflog_commits.go`

入口 `GetReflogCommitListDisplayStrings()` 第 15-36 行

**两种显示模式：**

| 模式 | 触发条件 | 列内容 |
|------|----------|--------|
| 紧凑模式 | 普通屏幕 | `短hash` + `reflog消息`（2 列） |
| 完整模式 | 全屏/宽屏 | `短hash` + `时间` + `reflog消息`（3 列） |

**颜色规则** `reflogHashColor()` 第 38-49 行：
- 被 diff 选中 → `theme.DiffTerminalColor`
- 被 cherry-pick 复制 → `theme.CherryPickedCommitTextStyle`
- 普通状态 → `style.FgBlue`（蓝色）

### 5.4 渲染优化

在 `NewReflogCommitsContext` 中设置了：

```go
renderOnlyVisibleLines: true  // 只渲染可见行，大数据量下不卡
```

配合 `ListRenderer` 的 `getDisplayStrings(startIdx, endIdx)` 只计算可视窗口内的条目。

---

## 六、跳转动作：Checkout / Reset / Cherry-Pick

### 6.1 控制器挂载

文件：`pkg/gui/controllers.go` 第 279-289 行

```go
// BasicCommitsController 被三个 context 共享
for _, context := range []controllers.ContainsCommits{
    gui.State.Contexts.LocalCommits,
    gui.State.Contexts.ReflogCommits,   // ← reflog 复用通用 commits 控制器
    gui.State.Contexts.SubCommits,
} {
    controllers.AttachControllers(context,
        controllers.NewBasicCommitsController(common, context))
}

// 再挂专属 controller
controllers.AttachControllers(gui.State.Contexts.ReflogCommits,
    reflogCommitsController,
)
```

所以 reflog 面板的操作键位和 commits 面板完全一致。

### 6.2 主面板渲染：GetOnRenderToMain()

文件：`pkg/gui/controllers/reflog_commits_controller.go` 第 40-62 行

**触发机制**：不是"光标移动时自动触发"，而是通过框架的 `onRenderToMainFn` 回调机制。注册发生在 `AttachControllers` 时（`pkg/gui/controllers/attach.go` 第 12 行），controller 的 `GetOnRenderToMain()` 返回值被挂到 context 上。该回调在两个时机被调用：

1. **Context 获得焦点**：`HandleFocus()` → `SimpleContext.HandleFocus()` → 调用 `onRenderToMainFn()`（`pkg/gui/context/simple_context.go` 第 44-46 行）。包括用户切换到 reflog 面板、从弹窗返回等场景。
2. **数据刷新后**：`PostRefreshUpdate` → `c.RenderToMainViews` 链路中调用 `HandleRenderToMain()`（`pkg/gui/view_helpers.go` 第 154、161 行）。侧边面板数据变化后，主面板内容需要同步更新。

因此主面板的刷新发生在"获得焦点"和"数据变更后"两个时机，而非"光标移动时"。

回调内部逻辑：

```go
func (self *ReflogCommitsController) GetOnRenderToMain() func() {
    return func() {
        self.c.Helpers().Diff.WithDiffModeCheck(func() {
            commit := self.context().GetSelected()
            var task types.UpdateTask
            if commit == nil {
                task = types.NewRenderStringTask("No reflog history")
            } else {
                // 用 git show 显示完整 diff
                cmdObj := self.c.Git().Commit.ShowCmdObj(
                    commit.Hash(),
                    self.c.Helpers().Diff.FilterPathsForCommit(commit))
                task = types.NewRunPtyTask(cmdObj.GetCmd())
            }
            self.c.RenderToMainViews(...)
        })
    }
}
```

### 6.3 动作一：Checkout（检出）

**按键定义**：`pkg/gui/controllers/basic_commits_controller.go` 第 53-60 行

**调用链：**

```
BasicCommitsController.checkout(commit)
  → RefsHelper.CreateCheckoutMenu(commit)
      弹出菜单：
      ├─ 选项 d：以 detached HEAD 检出该 commit
      │     → RefsHelper.CheckoutRef(hash, ...)
      │         → Git().Branch.Checkout(hash)
      │         → SelectFirstBranchAndFirstCommit()  // 光标归零
      │         → Refresh([COMMITS, BRANCHES, FILES, REFLOG, ...])
      │
      └─ 选项 1..n：检出指向该 commit 的分支
            → RefsHelper.CheckoutRef(branchName, ...)
```

**CheckoutRef 细节** (`pkg/gui/controllers/helpers/refs_helper.go` 第 43-131 行)：

- 先切到 branches 面板显示 inline "Checking out" 状态
- 遇到未提交改动 → 弹出 autostash 确认对话框
- 成功后限制 commit 加载数量（`SetLimitCommits(true)`）加快刷新

### 6.4 动作二：Reset（重置分支到该点）

**按键定义**：`pkg/gui/controllers/basic_commits_controller.go` 第 93-100 行

**调用链：**

```
BasicCommitsController.createResetMenu(commit)
  → RefsHelper.CreateGitResetMenu(name, ref)
      弹出菜单（三种强度）：
      ├─ mixed (m)：reset --mixed
      ├─ soft  (s)：reset --soft
      └─ hard  (h)：reset --hard （hard 会二次确认）
            → RefsHelper.ResetToRef(ref, strength)
                → Git().Commit.ResetToCommit(ref, strength)
                → LocalCommits.SetSelection(0)
                → ReflogCommits.SetSelection(0)
                → LocalCommits.SetLimitCommits(true)
                → Refresh([FILES, BRANCHES, REFLOG, COMMITS])
```

### 6.5 动作三：Cherry-Pick（复制提交）

**按键定义**：`pkg/gui/controllers/basic_commits_controller.go` 第 102-113 行

- 支持范围选择（range select）批量复制
- 复制后进入 cherry-picking 模式
- 在目标分支用 `PasteCommits` 粘贴（执行真正的 cherry-pick）

### 6.6 动作四：创建新分支

**按键定义**：`pkg/gui/controllers/basic_commits_controller.go` 第 76-80 行

- 以当前 reflog commit 为起点创建新分支
- 实现：`RefsHelper.NewBranch(from, fromFormattedName, suggestedBranchName)`

### 6.7 动作五：进入子提交视图（View Commits）

**控制器注册**：`pkg/gui/controllers.go` 第 241-248 行

`SwitchToSubCommitsController` 被挂载到 4 个 context 上共享使用，其中就包括 `ReflogCommits`：

```go
for _, context := range []controllers.CanSwitchToSubCommits{
    gui.State.Contexts.Branches,
    gui.State.Contexts.RemoteBranches,
    gui.State.Contexts.Tags,
    gui.State.Contexts.ReflogCommits,    // ← reflog 也支持进入子提交视图
} {
    controllers.AttachControllers(context, controllers.NewSwitchToSubCommitsController(...))
}
```

`CanSwitchToSubCommits` 接口 (`pkg/gui/controllers/switch_to_sub_commits_controller.go` 第 11-15 行) 要求实现：
- `GetSelectedRef()` — 返回选中的 ref（对 reflog 来说就是 commit 本身）
- `ShowBranchHeadsInSubCommits()` — 是否在子提交中显示分支头标记

**触发方式**：
1. **GoInto 键**：默认 `Universal.GoInto`（通常是 Enter 键），见 `switch_to_sub_commits_controller.go` 第 50 行
2. **双击**：`GetOnDoubleClick()` 同样绑定到 `viewCommits()`，见第 58-60 行

**完整跳转链路**：

```
用户按 Enter / 双击 reflog 条目
    │
    ▼
SwitchToSubCommitsController.viewCommits()   pkg/gui/controllers/switch_to_sub_commits_controller.go:62
    │
    ├─ ref := context.GetSelectedRef()       // ReflogCommitsContext 第 73-79 行
    │                                       // 返回当前选中的 commit
    │
    └─ SubCommitsHelper.ViewSubCommits()     pkg/gui/controllers/helpers/sub_commits_helper.go:34
        │
        ├─ 1. 加载子提交数据
        │     CommitLoader.GetCommits(
        │       RefName = commit.FullRefName(), // 即 commit hash
        │       Limit = true,                    // 只加载有限数量
        │       ...
        │     )
        │
        ├─ 2. 写入 Model.SubCommits           // setSubCommits() 第 75-79 行
        │
        ├─ 3. 配置 SubCommitsContext           // 第 55-66 行
        │     ├─ SetSelection(0)               // 光标归零
        │     ├─ SetParentContext(ReflogCommits)  // 按 ESC 返回 reflog
        │     ├─ SetTitleRef(commit hash 前 50 字符)
        │     ├─ SetRef(commit)
        │     ├─ SetLimitCommits(true)         // 限制加载数量
        │     ├─ SetShowBranchHeads(false)     // ← 关键：reflog 入口不显示分支头
        │     └─ ClearSearchString()
        │
        ├─ 4. PostRefreshUpdate(SubCommits)   // 刷新主面板和侧边
        │
        ├─ 5. FocusLine(true)                 // 聚焦到第一条
        │
        └─ 6. Context.Push(SubCommits)        // 压入 context 栈，切到子提交面板
```

**Reflog 的特殊配置**：

1. **不显示分支头标记**：`ReflogCommitsContext.ShowBranchHeadsInSubCommits()` 第 100-102 行直接返回 `false`。这是因为 reflog 条目代表的是一个任意的历史时间点，在这个时间点上显示"哪些分支头恰好位于这些 commit"没有明确语义，反而容易引起误解。分支、Tag 等 context 则会返回 `true`。

2. **GetSelectedRef() 返回 commit 自身**：`pkg/gui/context/reflog_commits_context.go` 第 73-79 行直接返回 `self.GetSelected()`（即 `*models.Commit`），因为 reflog 条目本身就代表了一个可寻址的 commit ref。

---

## 七、完整调用链路图

```
启动 / Git 操作后触发刷新
    │
    ▼
RefreshHelper.Refresh(options)         pkg/gui/controllers/helpers/refresh_helper.go:63
    │  scope 包含 types.REFLOG
    ▼
refreshReflogCommits()                  pkg/gui/controllers/helpers/refresh_helper.go:648
    │
    ├─ 刷新 ReflogCommits（完整集）
    │    └─ ReflogCommitLoader.GetReflogCommits()
    │         pkg/commands/git_commands/reflog_commit_loader.go:27
    │         └─ git log -g --format=+%H%x00%ct%x00%gs%x00%P
    │         └─ 增量断点: hash + commit提交者时间戳 + reflog消息 三元组
    │
    ├─ 刷新 FilteredReflogCommits（过滤子集）
    │
    └─ refreshView(ReflogCommits)  →  UI 线程重绘
              │
              ▼
用户切换到 Reflog 面板 / 数据刷新后
    │
    ▼
HandleFocus / HandleRenderToMain       ← 触发主面板渲染的两种时机
    │
    ▼
ReflogCommitsController.GetOnRenderToMain()  → git show <hash>
    │
    ▼
ReflogCommitsContext                    pkg/gui/context/reflog_commits_context.go
    数据来源: FilteredReflogCommits
    │
    ├─ presentation.GetReflogCommitListDisplayStrings()
    │  pkg/gui/presentation/reflog_commits.go:15
    │  渲染: [蓝色短hash]  [reflog消息]  (可选时间列)
    │
    ▼
用户按操作键
    │
    ├─ Enter / 双击 → SwitchToSubCommitsController.viewCommits()
    │                  pkg/gui/controllers/switch_to_sub_commits_controller.go:62
    │                    → SubCommitsHelper.ViewSubCommits()
    │                       pkg/gui/controllers/helpers/sub_commits_helper.go:34
    │                    → 加载子提交 → SetShowBranchHeads(false) → Push(SubCommits)
    │
    ├─ checkout 键 → BasicCommitsController.checkout()
    │                 pkg/gui/controllers/basic_commits_controller.go:53
    │                   → RefsHelper.CreateCheckoutMenu()
    │                      pkg/gui/controllers/helpers/refs_helper.go:301
    │                   → git checkout → Refresh()
    │
    └─ reset 键 → BasicCommitsController.createResetMenu()
                   pkg/gui/controllers/basic_commits_controller.go:93
                     → RefsHelper.CreateGitResetMenu()
                        pkg/gui/controllers/helpers/refs_helper.go:259
                     → git reset --xxx → Refresh()
```

---

## 八、关键设计要点总结

### 1. 增量加载
Reflog 是 lazygit 中**唯一做增量加载**的面板。通过 hash + commit 的提交者时间戳 + reflog 消息三元组识别断点，遇到已加载条目立即终止 git 命令输出，避免每次刷新都从头重读全部 reflog 历史。需注意：时间戳是 commit 的 committer date（`%ct`），不是 reflog 条目自身的操作时间，因此同一 commit 的连续 reflog 条目 hash 和时间戳完全相同，只能靠 reflog 消息消歧，存在理论碰撞风险。

### 2. 双模型设计
`ReflogCommits`（完整全集）与 `FilteredReflogCommits`（渲染用）分离：
- 业务逻辑（分支排序、undo）始终能用完整数据
- 过滤模式下渲染数据和业务数据互不干扰
- 非过滤模式下指针共享，零拷贝

### 3. 控制器组合复用
Reflog 不重复实现 commit 操作逻辑，直接通过 `BasicCommitsController` 复用 checkout / reset / cherry-pick 等功能，与 LocalCommits、SubCommits 保持一致的行为和键位。

### 4. 启动两阶段加载
启动时先进入 INITIAL 阶段快速显示 UI，后台异步加载 reflog，加载完成后再刷新分支排序，避免 reflog 数据量大时阻塞首屏。

### 5. Status 标记的用途
`StatusReflog` 在 gui 包中仅有一处实际使用：`pkg/gui/presentation/commits.go` 的 hash 颜色 switch 分支，将 reflog 条目 hash 映射为蓝色。它并不用于控制分支头标记等逻辑——那些由 `ReflogCommitsContext` 的方法（如 `CanRebase()` 返回 `false`、`ShowBranchHeadsInSubCommits()` 返回 `false`）直接决定。`StatusReflog` 本质上是一个颜色分型标记。

### 6. 可见行渲染优化
设置 `renderOnlyVisibleLines: true`，配合按视口范围计算显示字符串的机制，即使 reflog 条目极多也不会因全量渲染导致卡顿。

---

## 九、集成测试与单元测试证据

以下测试文件从代码层面验证了 reflog 视图的各项行为，是理解其正确性的直接证据。

### 9.1 单元测试：增量加载与数据解析

**文件**：`pkg/commands/git_commands/reflog_commit_loader_test.go`

覆盖场景：
1. **空 reflog**：断言 git 参数正确、返回空切片
2. **全量加载**（`lastReflogCommit=nil`）：5 条 reflog 条目完整解析
   - 测试数据（第 17-22 行）特意构造了**同一 hash（c3c4b66...）+ 同一时间戳（1643150483）**的 4 条连续条目，只有 reflog 消息不同，直接印证了"同一 commit 的连续 reflog 条目 hash 和时间戳相同、靠消息消歧"这一事实
3. **增量加载**（提供断点 commit）：断点是第 2 条 `"checkout: moving from B to A"`，断言只返回断点之前的 1 条增量 `"checkout: moving from A to B"`，且 `onlyObtainedNewReflogCommits=true`
4. **路径过滤**：验证 `--follow --name-status -- path` 参数正确拼接
5. **作者过滤**：验证 `--author=John Doe <john@doe.com>` 参数正确拼接
6. **错误处理**：git 命令返回 error 时向上透传

### 9.2 集成测试：交互行为

reflog 集成测试位于 `pkg/integration/tests/reflog/`，共 5 个用例。

#### checkout.go：检出 reflog commit 为 detached HEAD
- 预置场景：3 个 commit（one/two/three）后 `git reset --hard HEAD^^` 回退到 one
- 操作路径：ReflogCommits 面板 → 选中 `commit: three` → 按主键 → 菜单选 "Checkout commit xxx as detached head"
- 断言：
  - ReflogCommits 顶部新增 `checkout: moving from master to <hash>`
  - Branches 面板显示 `(HEAD detached at ...)`
  - LocalCommits 面板恢复 three/two/one

#### reset.go：硬重置到 reflog commit
- 预置场景：同上（回退后 two/three 在 reflog 中）
- 操作路径：ReflogCommits → 选中 `commit: three` → 按 `ViewResetOptions` 键 → 菜单选 Hard reset
- 断言：
  - ReflogCommits 顶部新增 `reset: moving to <hash>`
  - LocalCommits 面板恢复 three/two/one

#### cherry_pick.go：从 reflog 复制并 cherry-pick
- 预置场景：同上
- 操作路径：ReflogCommits → 选中 three → `CherryPickCopy` → 切到 LocalCommits → `PasteCommits` → 确认弹窗
- 断言：LocalCommits 变为 three、one（cherry-pick 产生新的 three 提交）

#### patch.go：从 reflog commit 构建 patch 并应用
- 预置场景：commit three 包含 file1 和 file2，回退后 file1/file2 消失
- 操作路径：ReflogCommits → 选中 three → **按 Enter 进入 SubCommits** → 选中 three 按 Enter 进入 CommitFiles → 选中 file1 → 按主键构建 patch → 菜单选 Apply patch
- 断言：Files 面板出现 file1（patch 被应用到工作区）
- 此用例同时验证了 **reflog 进入子提交视图** 的完整链路

#### do_not_show_branch_markers_in_reflog_subcommits.go：reflog 子提交不显示分支头标记
- 预置场景：branch1 含 one/two，branch2 基于 two 新增 three
- 对照：先在 Branches 面板中进入 branch2 的 SubCommits，断言显示 `CI * two`（星号表示有分支头标记）
- 验证路径：切到 ReflogCommits → 选中任意条目 → **按 Enter 进入 SubCommits** → 断言三条提交分别显示为 `CI three`、`CI two`、`CI one`，**均无 `*` 分支头标记**
- 直接证明了 `ShowBranchHeadsInSubCommits()` 返回 `false` 的行为

### 9.3 测试用例与代码的对应关系

| 测试文件 | 验证的代码路径 |
|----------|----------------|
| `reflog_commit_loader_test.go` | `ReflogCommitLoader.GetReflogCommits()` 增量/全量、过滤、错误处理 |
| `checkout.go` | `BasicCommitsController.checkout()` → `RefsHelper.CheckoutRef()` |
| `reset.go` | `BasicCommitsController.createResetMenu()` → `RefsHelper.ResetToRef()` |
| `cherry_pick.go` | `BasicCommitsController.cherryPickCopy()` + `PasteCommitsController` |
| `patch.go` | `SwitchToSubCommitsController.viewCommits()` → `SubCommitsHelper.ViewSubCommits()` |
| `do_not_show_branch_markers_in_reflog_subcommits.go` | `ReflogCommitsContext.ShowBranchHeadsInSubCommits()` 返回 `false` |
