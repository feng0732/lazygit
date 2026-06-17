# Tag 创建后的选中状态分析

> 仓库相对路径引用，聚焦刷新后的选中行为

---

## 核心结论速览

| 问题 | 答案 |
|------|------|
| 刷新后保持当前索引？ | ✅ 是的（不尝试定位原选中项） |
| 重新聚焦当前视图？ | ✅ 如果当前焦点在 Tags 面板则会 |
| 自动选中新标签？ | ❌ 只有当刷新前已选中第 0 项时才"恰好"选中新标签 |
| onCreate 回调是否工作？ | ❌ 声明但从未调用 |

---

## 一、标签排序：新标签在列表中的位置

首先明确标签的排序规则，这决定了新创建的标签会出现在哪里。

### 1.1 Git 命令排序

`pkg/commands/git_commands/tag_loader.go` 第 28-31 行：

```go
func (self *TagLoader) GetTags() ([]*models.Tag, error) {
    // get remote branches, sorted by creation date (descending)
    cmdArgs := NewGitCmd("tag").Arg("--list", "-n", "--sort=-creatordate").ToArgv()
    // ...
}
```

**关键参数**: `--sort=-creatordate` — 按**创建时间降序**排列。

**结果**：最新创建的标签永远出现在列表的**第 0 位**（顶部）。

---

## 二、刷新流程：从 Model 到 View

### 2.1 refreshTags 的完整调用链

```
Tag 创建成功
    │
    ▼
GpgHelper.WithGpgHandling → onSuccess + Refresh
    │
    ▼
RefreshHelper.Refresh(ASYNC, {COMMITS, TAGS})
    │
    ▼
OnWorker → refreshTags()   [worker goroutine]
    │
    ├── TagLoader.GetTags()             ← 重新加载所有标签
    │
    ├── self.c.Model().Tags = tags      ← 直接替换整个切片！
    │
    └── self.refreshView(TagsContext)
            │
            └── OnUIThread → 切回 UI 线程执行
                    │
                    ├── self.searchHelper.ReApplyFilter(context)   ← 重新应用过滤
                    │
                    ├── self.c.PostRefreshUpdate(context)         ← ↓ 见下节
                    │
                    └── AfterLayout → ReApplySearch(context)      ← 重新应用搜索
```

### 2.2 refreshTags 代码

`pkg/gui/controllers/helpers/refresh_helper.go` 第 461-471 行：

```go
func (self *RefreshHelper) refreshTags() error {
    tags, err := self.c.Git().Loaders.TagLoader.GetTags()
    if err != nil {
        return err
    }

    self.c.Model().Tags = tags   // 直接替换 Model 中的 Tags 切片
    self.refreshView(self.c.Contexts().Tags)  // 触发 UI 刷新
    return nil
}
```

**关键观察**：**没有保存原选中项，也没有根据 ID/Name 调整选中位置**。

---

## 三、PostRefreshUpdate：渲染与聚焦

`pkg/gui/view_helpers.go` 第 127-163 行：

```go
func (gui *Gui) postRefreshUpdate(c types.Context) {
    c.HandleRender()  // ← 第一步：重新渲染

    if gui.currentViewName() == c.GetViewName() {
        // 如果当前焦点就在 Tags 视图，调用 HandleFocus 重新聚焦
        c.HandleFocus(types.OnFocusOpts{})
    } else {
        // 如果焦点不在 Tags，只调用 FocusLine 确保选中位置绘制正确
        c.FocusLine(false)
        // ... 额外处理：渲染 main view 等
    }
}
```

---

## 四、HandleRender：ClampSelection 做了什么（以及没做什么）

`pkg/gui/context/list_context_trait.go` 第 110-128 行：

```go
func (self *ListContextTrait) HandleRender() {
    self.list.ClampSelection()  // ← 第一步也是唯一的选中处理
    
    // ... 渲染内容到视图（略）
}
```

### 4.1 ClampSelection 的实现

`pkg/gui/context/traits/list_cursor.go` 第 114-117 行：

```go
func (self *ListCursor) ClampSelection() {
    self.selectedIdx = self.clampValue(self.selectedIdx)
    self.rangeStartIdx = self.clampValue(self.rangeStartIdx)
}

func (self *ListCursor) clampValue(value int) int {
    clampedValue := -1
    length := self.getLength()
    if length > 0 {
        clampedValue = lo.Clamp(value, 0, length-1)  // 只确保在 [0, length-1] 范围内
    }
    return clampedValue
}
```

**ClampSelection 只做一件事**：确保选中索引不会越界。它 **不会**：
- ❌ 记住刷新前选中项的 ID/Name
- ❌ 在新列表中查找原选中项
- ❌ 自动调整索引指向原选中项

---

## 五、刷新前后的索引变化（3 种场景）

新标签插入在**第 0 位**，原列表整体后移一位。以下是刷新前后的选中行为对比：

### 场景 1：刷新前选中第 0 项（最常见）

```
刷新前列表（N 个标签）：
[0] tag-v1.0  ← 选中 (selectedIdx = 0)
[1] tag-v0.9
[2] tag-v0.8
...

刷新后列表（N+1 个标签）：
[0] tag-v1.1  ← 新标签
[1] tag-v1.0
[2] tag-v0.9
[3] tag-v0.8
...

selectedIdx 仍然是 0 → Clamp 后还是 0 → ✅ 恰好选中新标签！
```

这就是为什么 onCreate 回调不调用也"看似正常"的原因——绝大多数用户在 Tags 面板时，选中位置默认就是第 0 项。

### 场景 2：刷新前选中第 K 项（K > 0）

```
刷新前列表：
[0] tag-v1.0
[1] tag-v0.9  ← 选中 (selectedIdx = 1)
[2] tag-v0.8

刷新后列表：
[0] tag-v1.1  ← 新标签
[1] tag-v1.0  ← selectedIdx = 1 → 现在指向这里！（原第 0 项）
[2] tag-v0.9  ← 原选中项现在在这里（索引 2）
[3] tag-v0.8

结果：选中位置"上移一位"，不再指向原标签。
```

### 场景 3：刷新前选中最后一项

```
刷新前列表（3 个标签）：
[0] tag-v1.0
[1] tag-v0.9
[2] tag-v0.8  ← 选中 (selectedIdx = 2)

刷新后列表（4 个标签）：
[0] tag-v1.1  ← 新标签
[1] tag-v1.0
[2] tag-v0.9  ← selectedIdx = 2 → Clamp 后仍然有效
[3] tag-v0.8  ← 原选中项现在在这里（索引 3）

结果：同样上移一位。
```

---

## 六、HandleFocus vs FocusLine：聚焦行为

### 6.1 如果当前焦点在 Tags 视图

`postRefreshUpdate` 第 135-136 行：
```go
if gui.currentViewName() == c.GetViewName() {
    c.HandleFocus(types.OnFocusOpts{})
}
```

`HandleFocus` 调用链（`pkg/gui/context/list_context_trait.go` 第 91-97 行）：
```go
func (self *ListContextTrait) HandleFocus(opts types.OnFocusOpts) {
    self.FocusLine(opts.ScrollSelectionIntoView)  // ← 滚动到选中位置
    self.GetViewTrait().SetHighlight(self.list.Len() > 0)  // 高亮视图边框
    self.Context.HandleFocus(opts)
}
```

**FocusLine 作用**（`pkg/gui/context/list_context_trait.go` 第 35-75 行）：
- 将视图滚动到选中位置，确保选中项可见
- 设置搜索位置、范围选择、脚注信息（"X of Y"）

### 6.2 如果当前焦点不在 Tags 视图

`postRefreshUpdate` 第 143 行：
```go
c.FocusLine(false)  // false = 不滚动到视图
```

只确保选中位置的内部状态正确，不影响当前焦点。

---

## 七、onCreate 回调：意图与现实的鸿沟

### 7.1 传入位置（意图）

`pkg/gui/controllers/tags_controller.go` 第 349-354 行：
```go
func (self *TagsController) create() error {
    return self.c.Helpers().Tags.OpenCreateTagPrompt("", func() {
        self.context().SetSelection(0)  // ← 意图：创建成功后强制选中第 0 项
    })
}
```

### 7.2 接收位置（现实）

`pkg/gui/controllers/helpers/tags_helper.go` 第 25 行：
```go
func (self *TagsHelper) OpenCreateTagPrompt(ref string, onCreate func()) error {
    // onCreate 在整个函数体内出现次数：0（除函数签名外）
    // ... 从不调用 onCreate
}
```

### 7.3 其他调用点传空回调

- `pkg/gui/controllers/local_commits_controller.go` 第 1210-1212 行：`func() {}`
- `pkg/gui/controllers/branches_controller.go` 第 741-743 行：`func() {}`

### 7.4 结论

`onCreate` 是**死参数**。调用方期望的"创建后强制选中第 0 项"行为从未实现。

但由于 `ClampSelection` + 新标签在第 0 位的组合，在最常见的场景（刷新前选中第 0 项）下，结果恰好符合预期。

---

## 八、对比：refreshBranches 的智能选中保留

作为对照，分支刷新有完整的选中项保留逻辑，而标签刷新没有。

### refreshBranches 的实现

`pkg/gui/controllers/helpers/refresh_helper.go` 第 511-529 行：

```go
func (self *RefreshHelper) refreshBranches(...) {
    // 1. 保存原选中项
    prevSelectedBranch := self.c.Contexts().Branches.GetSelected()
    
    // 2. 替换 Model
    self.c.Model().Branches = branches
    
    // 3. 智能恢复选中位置
    if !keepBranchSelectionIndex && prevSelectedBranch != nil {
        self.searchHelper.ReApplyFilter(self.c.Contexts().Branches)
        
        // 根据 Name 在新列表中查找原选中项
        _, idx, found := lo.FindIndexOf(self.c.Contexts().Branches.GetItems(),
            func(b *models.Branch) bool { return b.Name == prevSelectedBranch.Name })
        if found {
            self.c.Contexts().Branches.SetSelectedLineIdx(idx)  // 调整到新位置
        }
    }
    
    self.refreshView(self.c.Contexts().Branches)
}
```

### refreshBranches vs refreshTags 对比

| 步骤 | refreshBranches | refreshTags |
|------|----------------|-------------|
| 保存原选中项 | ✅ `prevSelectedBranch := GetSelected()` | ❌ 无 |
| 根据 Name 查找新位置 | ✅ `lo.FindIndexOf(by Name)` | ❌ 无 |
| 调整选中索引 | ✅ `SetSelectedLineIdx(idx)` | ❌ 无 |
| 刷新后行为 | 保持原选中项（如果存在） | 保持原索引位置（可能指向不同项） |

---

## 九、完整时间线：Tag 创建后的选中变化

```
T0  用户在输入面板按确认
 │
 ├─ (假设用户选中了第 1 项：tag-v0.9，selectedIdx = 1)
 │
T1  git tag 命令执行完成
 │
T2  Refresh(ASYNC, {COMMITS, TAGS}) 触发
 │
T3  Worker goroutine 执行 refreshTags()
 │   ├─ GetTags() 返回 [tag-v1.1, tag-v1.0, tag-v0.9, tag-v0.8]
 │   ├─ Model().Tags = 新切片
 │   └─ refreshView → 调度 OnUIThread
 │
T4  UI 线程执行
 │   ├─ ReApplyFilter → 无过滤，filteredIndices = nil
 │   │
 │   ├─ PostRefreshUpdate(TagsContext)
 │   │   ├─ HandleRender()
 │   │   │   └─ ClampSelection()
 │   │   │       └─ selectedIdx = clamp(1, 0, 3) = 1  ← 保持索引 1 不变！
 │   │   │
 │   │   └─ currentViewName() == "tags" ?
 │   │       ├─ 是 → HandleFocus → FocusLine(true)  → 滚动到索引 1
 │   │       └─ 否 → FocusLine(false)                → 只更新内部状态
 │   │
 │   └─ AfterLayout → ReApplySearch → 无搜索
 │
T5  最终状态
     ├─ 列表显示：[0] tag-v1.1, [1] tag-v1.0, [2] tag-v0.9, [3] tag-v0.8
     ├─ 选中位置：索引 1 → tag-v1.0（原来的第 0 项）
     └─ ❌ 用户实际想选中的新标签 tag-v1.1 在索引 0，未被选中
```

---

## 十、设计观察与可能的改进

### 10.1 现有设计的合理之处

1. **ClampSelection 的极简设计**：只做边界检查，不引入额外复杂度
2. **新标签在顶部**：按创建时间降序排列符合用户直觉（最新的最显眼）
3. **聚焦逻辑分离**：HandleFocus/FocusLine 清晰区分是否需要滚动

### 10.2 可能的改进

#### 选项 A：修复 onCreate 回调，让其真正工作

在 `OpenCreateTagPrompt` 的 `onSuccess` 回调中调用 `onCreate`：

```go
// tags_helper.go 第 49-51 行
return self.gpg.WithGpgHandling(command, git_commands.TagGpgSign, 
    self.c.Tr.CreatingTag, 
    func() error { 
        onCreate()  // ← 加上这一行
        return nil 
    },
    []types.RefreshableView{types.COMMITS, types.TAGS}
)
```

这样无论刷新前选中什么，创建成功后都会强制选中第 0 项（新标签）。

#### 选项 B：像 refreshBranches 一样，根据 ID 智能保留原选中项

```go
// refresh_helper.go refreshTags()
func (self *RefreshHelper) refreshTags() error {
    prevSelectedTag := self.c.Contexts().Tags.GetSelected()  // 保存原选中项
    
    tags, err := self.c.Git().Loaders.TagLoader.GetTags()
    if err != nil { return err }
    
    self.c.Model().Tags = tags
    
    // 新增：智能恢复选中位置
    if prevSelectedTag != nil {
        _, idx, found := lo.FindIndexOf(tags,
            func(t *models.Tag) bool { return t.Name == prevSelectedTag.Name })
        if found {
            self.c.Contexts().Tags.SetSelectedLineIdx(idx)
        }
    }
    
    self.refreshView(self.c.Contexts().Tags)
    return nil
}
```

这样刷新后会保持原选中项（除非它被删除了），与分支刷新行为一致。

#### 选项 C：创建标签后始终选中新标签（结合 A + B 的思路）

如果是"创建新标签"场景，用户期望看到新创建的标签。可以在 `refreshTags` 中增加一个可选参数，或者在调用方（`OpenCreateTagPrompt`）的 onSuccess 中显式设置。

---

## 十一、关键文件索引

| 文件 | 关注点 |
|------|--------|
| `pkg/commands/git_commands/tag_loader.go` | `--sort=-creatordate` 决定新标签在第 0 位 |
| `pkg/gui/controllers/helpers/refresh_helper.go` | `refreshTags()` 无智能选中保留 |
| `pkg/gui/controllers/helpers/refresh_helper.go` | `refreshBranches()` 有智能选中保留（对比参照） |
| `pkg/gui/controllers/helpers/refresh_helper.go` | `refreshView()` 调度 OnUIThread 执行刷新 |
| `pkg/gui/view_helpers.go` | `postRefreshUpdate()` 区分 HandleFocus / FocusLine |
| `pkg/gui/context/list_context_trait.go` | `HandleRender()` → `ClampSelection()` 只做边界检查 |
| `pkg/gui/context/traits/list_cursor.go` | `ClampSelection()` / `clampValue()` 实现 |
| `pkg/gui/controllers/tags_controller.go` | `create()` 传入 `onCreate` 回调 |
| `pkg/gui/controllers/helpers/tags_helper.go` | `OpenCreateTagPrompt` 从不调用 `onCreate` |
