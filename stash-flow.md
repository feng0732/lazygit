# Lazygit Stash 操作流程分析

本文档从代码实现角度梳理 stash **入栈（push）** 与 **应用（apply/pop）** 两大操作的完整链路，包括入口动作、命令包装和状态更新三个层次。

---

## 一、核心代码文件索引

| 层次 | 文件 | 职责 |
|------|------|------|
| 入口层（Controller） | [files_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/files_controller.go) | stash 入栈的按键/菜单入口 |
| 入口层（Controller） | [stash_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/stash_controller.go) | stash apply/pop/drop/rename 的按键入口 |
| 上下文层（Context） | [stash_context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/context/stash_context.go) | stash 列表视图的 ViewModel 与渲染 |
| 命令包装层 | [stash.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go) | `StashCommands` 封装所有 git stash 子命令 |
| 数据加载层 | [stash_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash_loader.go) | 解析 `git stash list` 输出为模型对象 |
| 数据模型 | [stash_entry.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/models/stash_entry.go) | `StashEntry` 结构体定义 |
| 状态刷新层 | [refresh_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/helpers/refresh_helper.go) | `RefreshHelper` 调度各视图刷新 |
| 刷新类型定义 | [refresh.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/types/refresh.go) | `RefreshableView`、`RefreshOptions` 定义 |
| 全局状态 | [common.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/types/common.go) | `Model.StashEntries` 存储位置 |
| 顶层 Git 门面 | [git.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git.go) | `GitCommand.Stash` / `GitCommand.Loaders.StashLoader` 注入点 |

---

## 二、Stash 入栈（Push）流程

### 2.1 入口动作

入栈操作统一从 **Files 面板**触发，有两条入口路径：

#### 路径 A：快捷键直接入栈（默认 `shift+s`）

在 [files_controller.go:118-124](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/files_controller.go#L118-L124)：

```go
{
    Keys:        opts.GetKeys(opts.Config.Files.StashAllChanges),
    Handler:     self.stash,
    Description: self.c.Tr.Stash,
    ...
}
```

`self.stash()` 在 [files_controller.go:1316-1318](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/files_controller.go#L1316-L1318) 直接调用通用处理器：

```go
func (self *FilesController) stash() error {
    return self.handleStashSave(self.c.Git().Stash.Push, self.c.Tr.Actions.StashAllChanges)
}
```

#### 路径 B：菜单多选项入栈（默认 `s`）

在 [files_controller.go:125-131](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/files_controller.go#L125-L131) 绑定 `ViewStashOptions` 打开菜单。

菜单定义在 [files_controller.go:1107-1163](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/files_controller.go#L1107-L1163)，包含 5 个选项：

| 菜单项 | 调用的 `StashCommands` 方法 | 前置检查 |
|--------|------------------------------|----------|
| StashAllChanges (a) | `Push` | 工作树有改动 |
| StashAllChangesKeepIndex (i) | `StashAndKeepIndex` | 工作树有改动 |
| StashIncludeUntrackedChanges (U) | `StashIncludeUntrackedChanges` | 无 |
| StashStagedChanges (s) | `SaveStagedChanges` | 存在已暂存文件 |
| StashUnstagedChanges (u) | `StashUnstagedChanges` 或 `Push` | 工作树有改动（无暂存时降级为普通 Push） |

#### 通用处理器：`handleStashSave`

所有入栈路径最终汇聚到 [files_controller.go:1344-1360](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/files_controller.go#L1344-L1360)：

```go
func (self *FilesController) handleStashSave(stashFunc func(message string) error, action string) error {
    self.c.Prompt(types.PromptOpts{
        Title: self.c.Tr.StashChanges,
        HandleConfirm: func(stashComment string) error {
            self.c.LogAction(action)                 // 1. 记录用户操作日志
            if err := stashFunc(stashComment); err != nil {  // 2. 执行具体 git 命令
                return err
            }
            // 3. 刷新 STASH 和 FILES 两个视图
            self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH, types.FILES}})
            return nil
        },
        AllowEmptyInput: true,
    })
    return nil
}
```

**操作边界**：
- `LogAction` → 写操作审计日志
- `stashFunc` → 真正执行 git stash 命令
- `Refresh({STASH, FILES})` → 触发状态更新（详见第四节）

---

### 2.2 命令包装层（StashCommands）

所有 git stash 命令通过 [stash.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go) 中的 `StashCommands` 封装。

底层执行模式统一为：

```go
cmdArgs := NewGitCmd("stash").Arg(...).ToArgv()
return self.cmd.New(cmdArgs).Run()
```

其中 `self.cmd` 是 `oscommands.ICmdObjBuilder`，由 [GitCommon](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/common.go) 注入。

#### 入栈方法对照表

| 方法 | 实际 Git 命令 | 说明 |
|------|---------------|------|
| [Push(message)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L56-L61) | `git stash push -m <msg>` | 基础入栈 |
| [StashAndKeepIndex(message)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L105-L110) | `git stash push --keep-index -m <msg>` | 入栈后保留暂存区 |
| [StashIncludeUntrackedChanges(message)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L195-L200) | `git stash push --include-untracked -m <msg>` | 包含未跟踪文件 |
| [SaveStagedChanges(message)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L133-L193) | **Git ≥2.35**: `git stash push --staged -m <msg>`<br>**Git <2.35**: 6 步复合操作 | 仅暂存区入栈 |
| [StashUnstagedChanges(message)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L112-L130) | 3 步复合：`git commit --no-verify` → `stash push` → `git reset --soft HEAD^` | 仅未暂存入栈 |

#### SaveStagedChanges 的兼容分支（Git < 2.35）

这是最复杂的一个入栈命令，包含 6 个步骤：

1. `git stash --keep-index` — 临时把未暂存的改动藏起来
2. `git stash push -m <msg>` — 把真正要保存的暂存改动入栈
3. `git stash apply refs/stash@{1}` — 恢复第一步临时藏起的改动
4. `git stash show -p | git apply -R` — 反向应用补丁，把临时 stash 的内容从工作树移除
5. `git stash drop refs/stash@{1}` — 删除第一步的临时 stash
6. 遍历文件列表，清理 `"AD"` 状态（新增已暂存 + 工作树已删除）的文件

---

## 三、Stash 应用/出栈（Apply / Pop）流程

### 3.1 入口动作

应用操作从 **Stash 面板**触发，由 [stash_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/stash_controller.go) 管理。

按键绑定在 [stash_controller.go:36-78](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/stash_controller.go#L36-L78)：

| 按键 | Handler | 描述 |
|------|---------|------|
| Select (Enter) | `handleStashApply` | 应用 stash（不删除） |
| PopStash (默认 `g`) | `handleStashPop` | 弹出 stash（应用后删除） |
| Remove (默认 `d`) | `handleStashDrop` | 删除 stash（多选支持） |
| RenameStash | `handleRenameStashEntry` | 重命名 |
| New (默认 `n`) | `handleNewBranchOffStashEntry` | 基于 stash 建分支 |

#### handleStashApply

[stash_controller.go:111-129](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/stash_controller.go#L111-L129)：

```go
func (self *StashController) handleStashApply(stashEntry *models.StashEntry) error {
    return self.c.ConfirmIf(!self.c.UserConfig().Gui.SkipStashWarning,
        types.ConfirmOpts{
            Title:  self.c.Tr.StashApply,
            Prompt: self.c.Tr.SureApplyStashEntry,
            HandleConfirm: func() error {
                self.c.LogAction(self.c.Tr.Actions.ApplyStash)
                err := self.c.Git().Stash.Apply(stashEntry.Index)
                self.postStashRefresh()
                if err != nil { return err }
                if self.c.UserConfig().Gui.SwitchToFilesAfterStashApply {
                    self.c.Context().Push(self.c.Contexts().Files, types.OnFocusOpts{})
                }
                return nil
            },
        })
}
```

#### handleStashPop

[stash_controller.go:131-159](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/stash_controller.go#L131-L159) 与 Apply 结构一致，区别在于：

- 调用 `self.c.Git().Stash.Pop(stashEntry.Index)`
- 配置 `SwitchToFilesAfterStashPop` 决定是否跳转

#### handleStashDrop

[stash_controller.go:161-181](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/stash_controller.go#L161-L181) 支持多选，关键细节：

```go
for i := len(stashEntries) - 1; i >= 0; i-- {  // 倒序删除（index 大的先删，避免索引偏移）
    err := self.c.Git().Stash.Drop(stashEntries[i].Index)
    self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH}})  // 每删一个刷一次
    ...
}
```

#### 公共刷新方法 postStashRefresh

[stash_controller.go:183-185](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/stash_controller.go#L183-L185)：

```go
func (self *StashController) postStashRefresh() {
    self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH, types.FILES}})
}
```

> **与入栈操作一致**：Apply/Pop 后同样刷新 `{STASH, FILES}` 两个视图。

---

### 3.2 命令包装层

| 方法 | Git 命令 |
|------|----------|
| [Apply(index)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L48-L53) | `git stash apply refs/stash@{<index>}` |
| [Pop(index)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L41-L46) | `git stash pop refs/stash@{<index>}` |
| [Drop(index)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L34-L39) | `git stash drop refs/stash@{<index>}` |
| [DropNewest()](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L28-L32) | `git stash drop` |
| [Rename(index, msg)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash.go#L202-L218) | 3 步：`Hash()` → `Drop()` → `Store(hash, msg)` |

---

## 四、状态更新流程（Refresh 机制）

### 4.1 调用链总览

```
Controller.c.Refresh(opts)
    └─► guiCommon.Refresh()                 [gui_common.go:29-31]
          └─► RefreshHelper.Refresh(options)  [refresh_helper.go:63-237]
```

### 4.2 RefreshOptions 结构

定义在 [refresh.go:36-47](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/types/refresh.go#L36-L47)：

- `Scope []RefreshableView` — 指定刷新哪些视图（空 = 全部）
- `Mode RefreshMode` — `SYNC`（默认）/ `ASYNC` / `BLOCK_UI`
- `Then func()` — 刷新完成后的回调（仅 SYNC 模式可用）

Stash 相关操作始终使用 `{STASH, FILES}` Scope，Mode 为默认 SYNC。

### 4.3 RefreshHelper.Refresh 调度逻辑

在 [refresh_helper.go:63-237](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L63-L237) 中：

1. **Scope 解析**：根据 `options.Scope` 构建 `scopeSet`（空则用默认全量集合）
2. **并行调度**：使用 `sync.WaitGroup + goroutine` 并行执行各视图的刷新函数（`refresh(name, f)` 内部封装）
3. **同步等待**：`wg.Wait()` 确保所有刷新完成
4. **执行 Then**：调用 `options.Then()`

STASH 和 FILES 的分支在 [refresh_helper.go:168-179](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L168-L179)：

```go
if scopeSet.Includes(types.FILES) || scopeSet.Includes(types.SUBMODULES) {
    fileWg.Add(1)
    refresh("files", func() { _ = self.refreshFilesAndSubmodules(); fileWg.Done() })
}

if scopeSet.Includes(types.STASH) {
    refresh("stash", func() { self.refreshStashEntries() })
}
```

### 4.4 Stash 列表刷新：`refreshStashEntries`

[refresh_helper.go:741-746](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/controllers/helpers/refresh_helper.go#L741-L746)：

```go
func (self *RefreshHelper) refreshStashEntries() {
    self.c.Model().StashEntries = self.c.Git().Loaders.StashLoader.
        GetStashEntries(self.c.Modes().Filtering.GetPath())
    self.refreshView(self.c.Contexts().Stash)
}
```

两步操作：
1. **更新 Model**：通过 `StashLoader.GetStashEntries()` 重新加载数据，写入全局 `Model.StashEntries`
2. **刷新 View**：调用 `refreshView()` 触发 `StashContext` 对应的视图重绘

### 4.5 数据加载层：StashLoader

[stash_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/git_commands/stash_loader.go) 提供两条加载路径：

| 方法 | Git 命令 | 适用场景 |
|------|----------|----------|
| `getUnfilteredStashEntries()` | `git stash list -z --pretty=%H|%ct|%gs` | 无过滤条件（默认），按 NUL 分隔解析 |
| `GetStashEntries(filterPath)` | `git stash list --name-only --pretty=%gd:%H|%ct|%gs` | 按路径过滤，逐行匹配文件前缀 |

解析函数 `stashEntryFromLine` 将每行 `hash|timestamp|message` 格式化为 `StashEntry` 模型。

### 4.6 数据模型：StashEntry

[stash_entry.go](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/commands/models/stash_entry.go) 字段：

```go
type StashEntry struct {
    Index   int      // stash@{index} 中的数字索引
    Recency string   // "2 hours ago" 相对时间
    Name    string   // stash message
    Hash    string   // stash commit hash
}
```

全局存储位置：`gui.State.Model.StashEntries []*models.StashEntry`，定义在 [common.go:301](file:///d:/fz/0601-2/solo-dogfeeding/code/33-lazygit/pkg/gui/types/common.go#L301)。

---

## 五、操作边界总结

### Stash Push 入栈边界

```
┌────────────────────────────────────────────────────────────┐
│  1. 入口层  FilesController                                 │
│     • 快捷键 shift+s → stash() → handleStashSave           │
│     • 菜单键 s → createStashMenu() → handleStashSave       │
│     • Prompt 获取 message → LogAction 记日志               │
├────────────────────────────────────────────────────────────┤
│  2. 命令层  StashCommands.Push / StashAndKeepIndex / ...   │
│     • NewGitCmd("stash").Arg(...).ToArgv()                 │
│     • self.cmd.New(cmdArgs).Run()                          │
├────────────────────────────────────────────────────────────┤
│  3. 状态层  Refresh({STASH, FILES})                         │
│     • StashLoader.GetStashEntries() → Model.StashEntries   │
│     • refreshView(StashContext) 重绘列表                    │
│     • refreshFilesAndSubmodules() 重绘文件树                │
└────────────────────────────────────────────────────────────┘
```

### Stash Apply/Pop 应用边界

```
┌────────────────────────────────────────────────────────────┐
│  1. 入口层  StashController                                 │
│     • Enter → handleStashApply                              │
│     • g → handleStashPop                                    │
│     • Confirm 弹窗（可配置跳过）                             │
│     • LogAction + LogCommand（仅 Pop 记录日志）              │
│     • 可选：SwitchToFilesAfterStashApply/Pop 跳转 Files     │
├────────────────────────────────────────────────────────────┤
│  2. 命令层  StashCommands.Apply/Pop/Drop(index)             │
│     • git stash apply/pop/drop refs/stash@{<index>}         │
├────────────────────────────────────────────────────────────┤
│  3. 状态层  postStashRefresh → Refresh({STASH, FILES})      │
│     • 与入栈完全一致的刷新链路                               │
└────────────────────────────────────────────────────────────┘
```

### 关键设计观察

1. **入口与状态解耦**：Controller 层从不直接操作 Model，只通过 `Refresh(Scope)` 发出刷新信号，由 `RefreshHelper` 统一调度数据加载和视图重绘。
2. **Scope 精准控制**：Stash 操作只刷新 `{STASH, FILES}` 而非全量，避免不必要的 I/O。
3. **索引安全**：多选 Drop 采用倒序遍历，规避 `stash@{n}` 索引在删除后重新编号导致的错位。
4. **命令可组合**：`handleStashSave` 接收 `func(message string) error` 作为参数，5 种入栈方式共享同一 Prompt + Log + Refresh 外壳。
