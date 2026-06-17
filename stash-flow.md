# Lazygit Stash 操作流程分析

本文档从代码实现角度梳理 stash **入栈（push）** 与 **应用（apply/pop）** 两大操作的完整链路，包括**入口动作、命令包装和状态更新**三个层次，特别标注了**刷新时机与失败/冲突时的状态处理**。

---

## 一、核心代码文件索引

| 层次 | 文件 | 职责 |
|------|------|------|
| 入栈入口（Controller） | [files_controller.go](pkg/gui/controllers/files_controller.go) | stash 入栈的按键/菜单入口、Prompt 封装 |
| 出栈入口（Controller） | [stash_controller.go](pkg/gui/controllers/stash_controller.go) | stash apply/pop/drop/rename 的按键入口、Confirm 封装 |
| 上下文层 | [stash_context.go](pkg/gui/context/stash_context.go) | stash 列表 ViewModel 与渲染数据来源 |
| 命令包装层 | [stash.go](pkg/commands/git_commands/stash.go) | `StashCommands` 封装所有 git stash 子命令 |
| 数据加载层 | [stash_loader.go](pkg/commands/git_commands/stash_loader.go) | 解析 `git stash list` 输出为模型对象 |
| 数据模型 | [stash_entry.go](pkg/commands/models/stash_entry.go) | `StashEntry` 结构体 |
| 状态刷新层 | [refresh_helper.go](pkg/gui/controllers/helpers/refresh_helper.go) | `RefreshHelper` 调度各 Scope 对应的数据加载和视图重绘 |
| 刷新类型定义 | [refresh.go](pkg/gui/types/refresh.go) | `RefreshableView`、`RefreshOptions` 定义 |
| 键位默认值定义 | [user_config.go](pkg/config/user_config.go) | 所有默认键位的配置结构体和默认值 |
| 全局 Model 存储 | [common.go](pkg/gui/types/common.go) | `Model.StashEntries` 存放位置 |
| 顶层 Git 门面 | [git.go](pkg/commands/git.go) | `GitCommand.Stash` / `GitCommand.Loaders.StashLoader` 依赖注入点 |
| Confirm/Prompt 错误传递 | [popup_handler.go](pkg/gui/popup/popup_handler.go) | `Confirm` / `ConfirmIf` / `Prompt` 的 Popup 创建和错误返回机制 |

---

## 二、Stash 入栈（Push）流程

### 2.1 入口动作（按代码核对后的默认键位

入栈操作统一从 **Files 面板**触发，有两条入口路径：

#### 路径 A：快捷键直接入栈（默认 **`s`** 小写）

定义在 [user_config.go:1053](pkg/config/user_config.go#L1053)：

```go
StashAllChanges: Keybinding{"s"},
```

在 [files_controller.go:118-124](pkg/gui/controllers/files_controller.go#L118-L124) 绑定 Handler：

```go
{
    Keys:        opts.GetKeys(opts.Config.Files.StashAllChanges),
    Handler:     self.stash,
    Description: self.c.Tr.Stash,
    Tooltip:     self.c.Tr.StashTooltip,
    DisplayOnScreen: true,
},
```

`self.stash()` 位于 [files_controller.go:1316-1318](pkg/gui/controllers/files_controller.go#L1316-L1318)：

```go
func (self *FilesController) stash() error {
    return self.handleStashSave(self.c.Git().Stash.Push, self.c.Tr.Actions.StashAllChanges)
}
```

#### 路径 B：菜单多选项入栈（默认 **`S`** 大写 / Shift+S）

定义在 [user_config.go:1054](pkg/config/user_config.go#L1054)：

```go
ViewStashOptions: Keybinding{"S"},
```

在 [files_controller.go:125-131](pkg/gui/controllers/files_controller.go#L125-L131) 绑定 `createStashMenu`。

菜单定义在 [files_controller.go:1107-1165](pkg/gui/controllers/files_controller.go#L1107-L1165)，共 **5 个菜单项**：

| 菜单热键 | 菜单项标签 | 调用的 StashCommands 方法 | 前置条件检查 |
|----------|-----------|------------------------|-------------|
| `a` | StashAllChanges | `Push` | 工作树有改动（排除子模块）|
| `i` | StashAllChangesKeepIndex | `StashAndKeepIndex` | 工作树有改动 |
| `U` | StashIncludeUntrackedChanges | `StashIncludeUntrackedChanges` | 无 |
| `s` | StashStagedChanges | `SaveStagedChanges` | 存在已暂存文件（排除子模块）|
| `u` | StashUnstagedChanges | 有暂存 → `StashUnstagedChanges`<br>无暂存 → 降级 `Push` | 工作树有改动 |

#### 通用外壳：`handleStashSave

所有入栈路径最终汇聚到 [files_controller.go:1344-1360](pkg/gui/controllers/files_controller.go#L1344-L1360)：

```go
func (self *FilesController) handleStashSave(stashFunc func(message string) error, action string) error {
    self.c.Prompt(types.PromptOpts{
        Title: self.c.Tr.StashChanges,
        HandleConfirm: func(stashComment string) error {
            self.c.LogAction(action)                      // ① 写操作审计日志
            if err := stashFunc(stashComment); err != nil {  // ② 执行 git stash 命令
                return err                                   // ⚠️ 失败：直接 return，**不刷新**
            }
            // ③ 仅在成功时刷新 STASH 和 FILES
            self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH, types.FILES}})
            return nil
        },
        AllowEmptyInput: true,
    })
    return nil  // Prompt 是异步创建 popup，外层始终立即 return nil
}
```

**⚠️ 关键边界结论 1 — 入栈刷新策略：**
- ✅ **成功** → `Refresh({STASH, FILES})
- ❌ **失败** → **不刷新任何视图**，`stashFunc 返回的 error 由 Prompt 的 OnConfirm → `self.context().State.OnConfirm() 错误冒泡到 GUI 框架层显示错误消息

---

### 2.2 命令包装层（StashCommands）

全部定义在 [stash.go](pkg/commands/git_commands/stash.go) 中的 `StashCommands` 结构体。

底层执行模式**全部走相同模式：

```go
cmdArgs := NewGitCmd("stash").Arg(...).ToArgv()
return self.cmd.New(cmdArgs).Run()  // 返回 error
```

| 方法 | 实际 Git 命令 | 说明 |
|------|---------------|------|
| [Push(message)](pkg/commands/git_commands/stash.go#L56-L61) | `git stash push -m <msg>` | 基础入栈 |
| [StashAndKeepIndex(message)](pkg/commands/git_commands/stash.go#L105-L110) | `git stash push --keep-index -m <msg>` | 入栈后保留暂存区 |
| [StashIncludeUntrackedChanges(message)](pkg/commands/git_commands/stash.go#L195-L200) | `git stash push --include-untracked -m <msg>` | 含未跟踪文件 |
| [SaveStagedChanges(message)](pkg/commands/git_commands/stash.go#L133-L193) | Git ≥2.35: `git stash push --staged -m <msg>`<br>Git <2.35: **6 步复合操作**（见下文） | 仅暂存区入栈 |
| [StashUnstagedChanges(message)](pkg/commands/git_commands/stash.go#L112-L130) | **3 步复合**：临时 commit → stash → reset soft | 仅未暂存入栈 |
| [Store(hash, message)](pkg/commands/git_commands/stash.go#L63-L72) | `git stash store [-m msg] <hash>` | Rename 内部使用 |

#### 复合命令细节：SaveStagedChanges（Git < 2.35 兼容方案）

6 个步骤：
1. `git stash --keep-index` — 临时藏起未暂存改动
2. `git stash push -m <msg>` — 保存暂存改动入栈
3. `git stash apply refs/stash@{1}` — 恢复步骤 1 临时藏的改动
4. `git stash show -p \| git apply -R` — 反向应用补丁，从工作树移除临时 stash 的内容
5. `git stash drop refs/stash@{1}` — 删除步骤 1 的临时 stash
6. 遍历文件列表，**清理 `AD` 状态**（新增已暂存 + 工作树已删除）的文件

#### 复合命令细节：StashUnstagedChanges

3 个步骤：
1. `git commit --no-verify -m "[lazygit] stashing unstaged changes` — 先把暂存改动临时提交（跳过 githooks）
2. `git stash push -m <msg>` — 把真正要保存的未暂存入栈
3. `git reset --soft HEAD^` — 回滚步骤 1 的临时提交，恢复暂存区

---

## 三、Stash 应用/出栈（Apply / Pop）流程

### 3.1 入口动作（按代码核对默认键位）

应用操作从 **Stash 面板**触发，按键绑定在 [stash_controller.go:36-78](pkg/gui/controllers/stash_controller.go#L36-L78)，组合自 `KeybindingUniversalConfig` 和 `KeybindingStashConfig`：

| 默认键位来源 | 默认值 | Handler | 说明 |
|------------|--------|---------|------|
| `Universal.Select` | **`<space>` 空格** | `handleStashApply` | 应用 stash（**不删除**）|
| `Stash.PopStash` | **`g`** | `handleStashPop` | 弹出 stash（应用后删除）|
| `Universal.Remove` | **`d`** | `handleStashDrop` | 删除 stash（**支持多选范围删除）|
| `Universal.New` | **`n`** | `handleNewBranchOffStashEntry` | 基于 stash 创建分支 |
| `Stash.RenameStash` | **`r`** | `handleRenameStashEntry` | 重命名 stash |

> ⚠️ **注意**：`<enter` 在 Stash 面板上**不绑定** apply 操作，绑定的是 `space`。

#### handleStashApply

[stash_controller.go:111-129](pkg/gui/controllers/stash_controller.go#L111-L129)：

```go
func (self *StashController) handleStashApply(stashEntry *models.StashEntry) error {
    return self.c.ConfirmIf(!self.c.UserConfig().Gui.SkipStashWarning,
        types.ConfirmOpts{
            Title:  self.c.Tr.StashApply,
            Prompt: self.c.Tr.SureApplyStashEntry,
            HandleConfirm: func() error {
                self.c.LogAction(self.c.Tr.Actions.ApplyStash)          // ① 日志
                err := self.c.Git().Stash.Apply(stashEntry.Index)         // ② 执行 git stash apply
                self.postStashRefresh()                               // ③ ⚠️ 先刷新（无论成功失败都刷新）
                if err != nil {                                       // ④ 再检查错误
                    return err
                }
                if self.c.UserConfig().Gui.SwitchToFilesAfterStashApply {
                    self.c.Context().Push(self.c.Contexts().Files, ...)  // ⑤ 成功才跳 Files
                }
                return nil
            },
        })
}
```

#### handleStashPop

[stash_controller.go:131-159](pkg/gui/controllers/stash_controller.go#L131-L159)，与 Apply 结构基本一致：

```go
pop := func() error {
    self.c.LogAction(self.c.Tr.Actions.PopStash)
    self.c.LogCommand(...) // 额外记录到 Command Log 面板
    err := self.c.Git().Stash.Pop(stashEntry.Index)
    self.postStashRefresh()  // ⚠️ 先刷新，再检错
    if err != nil { return err }
    if SwitchToFilesAfterStashPop { 切换到 Files }
    return nil
}
```

如果 `SkipStashWarning=true`，直接走 `pop()`；否则弹 Confirm。

#### handleStashDrop

[stash_controller.go:161-181](pkg/gui/controllers/stash_controller.go#L161-L181)，**支持多选删除，倒序遍历**：

```go
for i := len(stashEntries) - 1; i >= 0; i-- {   // ⚠️ 从大 index 往小删，避免索引重排
    self.c.LogCommand(...)
    err := self.c.Git().Stash.Drop(stashEntries[i].Index)
    self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH}})  // 每删一个刷一次
    if err != nil {
        return err  // 中途失败终止，但前面的 Refresh 已经执行了
    }
}
self.context().CollapseRangeSelectionToTop()
```

#### handleRenameStashEntry

[stash_controller.go:191-218](pkg/gui/controllers/stash_controller.go#L191-L218)，**特殊的刷新策略**——**成败都刷新**：

```go
HandleConfirm: func(response string) error {
    self.c.LogAction(...)
    err := self.c.Git().Stash.Rename(stashEntry.Index, response)
    if err != nil {
        self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH}})  // ❌ 失败也刷新
        return err
    }
    self.context().SetSelection(0)
    self.context().FocusLine(true)
    self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH}})  // ✅ 成功也刷新
    return nil
}
```

Rename 是 3 步复合操作（`Hash → Drop → Store`），Drop 失败会导致 stash 条目丢失但新的还没存进去，所以**即使失败也要刷新**反映真实状态。

#### 公共刷新方法 postStashRefresh

[stash_controller.go:183-185](pkg/gui/controllers/stash_controller.go#L183-L185)：

```go
func (self *StashController) postStashRefresh() {
    self.c.Refresh(types.RefreshOptions{Scope: []types.RefreshableView{types.STASH, types.FILES}})
}
```

---

### 3.2 命令包装层

| 方法 | Git 命令 |
|------|----------|
| [Apply(index)](pkg/commands/git_commands/stash.go#L48-L53) | `git stash apply refs/stash@{<index>}` |
| [Pop(index)](pkg/commands/git_commands/stash.go#L41-L46) | `git stash pop refs/stash@{<index>}` |
| [Drop(index)](pkg/commands/git_commands/stash.go#L34-L39) | `git stash drop refs/stash@{<index>}` |
| [DropNewest()](pkg/commands/git_commands/stash.go#L28-L32) | `git stash drop`（无参数 = drop stash@{0}）|
| [Rename(index, msg)](pkg/commands/git_commands/stash.go#L202-L218) | 3 步：`Hash(index)` → `Drop(index)` → `Store(hash, msg)` |
| [Hash(index)](pkg/commands/git_commands/stash.go#L74-L81) | `git rev-parse refs/stash@{<index>}`（Rename 内部用）|

---

## 四、状态更新流程（Refresh 机制与失败处理）

### 4.1 调用链总览

```
Controller.c.Refresh(opts)
    └─► guiCommon.Refresh()           [gui_common.go:29-31]
          └─► RefreshHelper.Refresh(options)  [refresh_helper.go:63-237]
```

### 4.2 RefreshOptions 结构

定义在 [refresh.go:36-47](pkg/gui/types/refresh.go#L36-L47)：

| 字段 | 说明 | Stash 相关操作的取值 |
|------|------|---------------------|
| `Scope []RefreshableView` | 指定刷新哪些视图（空=全部）| `{STASH, FILES}`（入栈/应用/弹出）<br>`{STASH}`（删除/重命名仅刷列表）|
| `Mode RefreshMode` | 刷新模式：`SYNC`（默认，等待所有 goroutine 结束）/ `ASYNC` / `BLOCK_UI` | 始终默认 `SYNC`
| `Then func()` | 刷新完成回调（仅 SYNC 可用）| 未使用 |

### 4.3 RefreshHelper.Refresh 调度逻辑

在 [refresh_helper.go:63-237](pkg/gui/controllers/helpers/refresh_helper.go#L63-L237) 核心调度：

1. **Scope → scopeSet 构建**：空 Scope 用默认集合（包含 STASH、FILES 等）
2. **并行调度**：`sync.WaitGroup + goroutine` 并行执行各 scope 的刷新函数
3. **同步等待**：`wg.Wait()`
4. **执行 Then 回调**

STASH 与 FILES 的调度分支在 [refresh_helper.go:168-179](pkg/gui/controllers/helpers/refresh_helper.go#L168-L179)：

```go
// FILES 和 SUBMODULES 共享同一个 goroutine（内部 fileWg 同步）
if scopeSet.Includes(types.FILES) || scopeSet.Includes(types.SUBMODULES) {
    fileWg.Add(1)
    refresh("files", func() { _ = self.refreshFilesAndSubmodules(); fileWg.Done() })
}
// STASH 独立 goroutine
if scopeSet.Includes(types.STASH) {
    refresh("stash", func() { self.refreshStashEntries() })
}
```

### 4.4 Stash 列表刷新：refreshStashEntries()

[refresh_helper.go:741-746](pkg/gui/controllers/helpers/refresh_helper.go#L741-L746)：

```go
func (self *RefreshHelper) refreshStashEntries() {
    // ① 重新加载数据写入全局 Model
    self.c.Model().StashEntries = self.c.Git().Loaders.StashLoader.
        GetStashEntries(self.c.Modes().Filtering.GetPath())
    // ② 触发 StashContext 对应视图重绘
    self.refreshView(self.c.Contexts().Stash)
}
```

### 4.5 数据加载层：StashLoader

[stash_loader.go](pkg/commands/git_commands/stash_loader.go)：

| 方法 | Git 命令 | 适用场景 |
|------|----------|----------|
| `getUnfilteredStashEntries()` | `git stash list -z --pretty=%H\|%ct\|%gs` | 无过滤，按 NUL 分隔解析 |
| `GetStashEntries(filterPath)` | `git stash list --name-only --pretty=%gd:%H\|%ct\|%gs` | 按路径过滤，逐文件前缀匹配 |

解析函数 `stashEntryFromLine` 将 `hash|unix_timestamp|message` 格式化为 `StashEntry`，`Recency` 由 `utils.UnixToTimeAgo` 生成相对时间字符串。

### 4.6 数据模型：StashEntry

[stash_entry.go](pkg/commands/models/stash_entry.go)：

```go
type StashEntry struct {
    Index   int      // stash@{index} 中的数字索引（0 是最新）
    Recency string   // "2 hours ago" 相对时间
    Name    string   // stash message（用户写的备注）
    Hash    string   // stash 对应的 commit hash
}
```

全局存储：`gui.State.Model.StashEntries []*models.StashEntry`，定义于 [common.go:301](pkg/gui/types/common.go#L301)。

---

## 五、刷新时机与失败/冲突时的状态更新先后

### 5.1 各操作刷新策略对照表（核心修正后的完整对比）

| 操作 | git 命令位置 | Refresh 调用位置 | ✅ 成功时 | ❌ 失败/冲突时 |
|------|-------------|-----------------|----------|---------------|
| **入栈 Push**（5 种） | `stashFunc(msg)` 之后 | `if err != nil { return }` 之后 | ✅ Refresh `{STASH, FILES}` | ❌ **不刷新**，error 上抛 |
| **Apply** | `Apply(index)` 之后 | `postStashRefresh()` 先于 err 判断 | ✅ 同上 + 可选跳 Files | ✅ **刷新**（先 Refresh 再 return err |
| **Pop** | `Pop(index)` 之后 | `postStashRefresh()` 先于 err 判断 | ✅ 同上 + 可选跳 Files | ✅ **刷新**（先 Refresh 再 return err |
| **Drop**（多选）| 每次 `Drop` 之后立即 Refresh，每次 `Drop` 之后 | 每删一个 Refresh `{STASH}` | ✅ 全部删完后 `CollapseRangeSelectionToTop` | ✅ 已执行的 Refresh 已生效，中途停止 |
| **Rename** | `Rename` 之后 | err 分支和成功分支**都 Refresh `{STASH}` | ✅ 刷新 + 选第 0 项 + Focus | ✅ **刷新**（Drop 失败但已生效的状态 |

### 5.2 为什么 Apply/Pop **冲突场景的状态更新先后分析

Apply/Pop **先 Refresh 再判断 err 是不是刻意为之的设计，冲突后果：

```
时间线  ─────────────────────────────────────────────────────────────────────►

Step 1  用户按 <space>（Apply）
         │
         ├─ ① git stash apply refs/stash@{0}
         │     Git 执行 apply，
         │     ├─ 若冲突：Git 写入 <<<<<<< 标记到工作树文件，返回 exit code 1
         │     └─ 若干净：工作树变干净，stash entry 仍在（apply 不删）
         │
         ├─ ② postStashRefresh()  ← **无论上面返回什么都立刻执行**
         │     ├─ 刷新 Stash 列表（STASH scope）→ 重新 git stash list
         │     │   Apply 冲突：stash@{0} 仍在列表
         │     │   Apply 干净：stash@{0} 仍在（apply 本来就不删除）
         │     │   Pop 冲突：stash@{0} 仍在（Git 不删除冲突的 pop）
         │     │   Pop 干净：stash@{0} 被删除
         │     └─ 刷新 Files 面板（FILES scope）→ 重新 git status
         │         Apply/Pop 冲突：显示冲突文件状态（UU 等）
         │         Apply/Pop 干净：显示工作树变干净或变更
         │
         └─ ③ if err != nil { return err }
               返回错误 → GUI 弹出错误提示框显示冲突信息
```

**设计合理性分析**：冲突时工作树已经被 Git 写入了冲突标记，stash 列表状态也改变了（Pop 冲突时 Git 不会删除 stash），**先 Refresh 保证 UI 立即反映真实的磁盘状态**，再通过 error 提醒用户有冲突需要解决。

### 5.3 ConfirmIf / Prompt 的错误传递机制

| API | 返回值 | HandleConfirm 返回的 error 去向 |
|-----|--------|------------------------------|
| `Confirm(opts)` | **无返回值 (void)** | 创建 ConfirmationContext Popup → 用户按 enter → `context.State.OnConfirm()` 执行 → error 冒泡到 keybinding handler 层 → GUI error toast |
| `ConfirmIf(cond, opts)` | **error** | cond=true：同上，外层始终 `return nil`<br>cond=false：直接 `return opts.HandleConfirm()`，error 原样返回给调用方 |
| `Prompt(opts)` | **无返回值 (void)** | 创建 PromptContext Popup → 用户 enter → `context.State.OnConfirm()` 执行 → error 冒泡到 GUI 层显示 |

**影响**：Apply 走 `ConfirmIf`，SkipStashWarning=false 时弹框 error 走 GUI；SkipStashWarning=true 时直接返回 error 给调用方。Pop 两种路径都有。

---

## 六、操作边界总结图

### Stash Push 入栈边界（键位 s 和 S）

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. 入口层  FilesController                                            │
│     • s → stash() → handleStashSave(Push)                            │
│     • S → createStashMenu() → 5 选项 → handleStashSave(5种func)      │
│     • Prompt(AllowEmptyInput) → 输入 message（可空）                    │
│     • HandleConfirm 内：LogAction → stashFunc → [err 不刷/才刷 Refresh] │
├─────────────────────────────────────────────────────────────────────┤
│  2. 命令层  StashCommands.*                                            │
│     • NewGitCmd("stash").Arg(...).ToArgv()                            │
│     • self.cmd.New(cmdArgs).Run() → 返回 error                        │
│     • 复合命令多步执行，任一步失败即 error 终止                         │
├─────────────────────────────────────────────────────────────────────┤
│  3. 状态层  Refresh({STASH, FILES})  ← 仅成功时执行                   │
│     • parallel: refreshStashEntries() + refreshFilesAndSubmodules()     │
│     • Model.StashEntries ← StashLoader.GetStashEntries()              │
│     • refreshView(StashContext) 重绘列表                                │
└─────────────────────────────────────────────────────────────────────┘
```

### Stash Apply / Pop 应用边界（键位 space / g）

```
┌──────────────────────────────────────────────────────────────────────┐
│  1. 入口层  StashController                                            │
│     • space → handleStashApply                                             │
│     • g → handleStashPop                                                  │
│     • ConfirmIf(SkipStashWarning)                                         │
│     • HandleConfirm 内：                                                  │
│         LogAction[+LogCommand] → Apply/Pop → postStashRefresh → if err │
│         ✅ 成功才跳 Files（可配置）                                        │
├──────────────────────────────────────────────────────────────────────┤
│  2. 命令层  StashCommands.Apply / Pop                                    │
│     • git stash apply/pop refs/stash@{index}                              │
│     • 冲突时 Git 返回 exit code 1 + 工作树写冲突标记                       │
│     • Pop 冲突时 Git 不删 stash（apply 本来就不删）                        │
├──────────────────────────────────────────────────────────────────────┤
│  3. 状态层  postStashRefresh() → Refresh({STASH, FILES})  ← 必执行    │
│     （**无论成功失败**：先反映真实磁盘状态，                                │
│      再 return err 给 GUI 显示错误消息                                           │
└──────────────────────────────────────────────────────────────────────┘
```

### 关键设计观察（修正清单

1. **入口与状态解耦**：Controller 从不直接操作 Model，统一通过 `Refresh(Scope)` 发出刷新信号，由 `RefreshHelper` 统一调度数据加载和视图重绘。

2. **Scope 精准控制**：
   - 入栈/应用/弹出：`{STASH, FILES}`（stash 内容变更 + 工作树变更
   - 删除/重命名：`{STASH}`（仅列表变动而已）

3. **刷新时机因场景差异是核心**：
   - **入栈**：失败不刷新（命令失败 = 没写任何事情发生，不需要刷了反而旧 UI 还是老状态）
   - **Apply/Pop**：**先刷新再报错误**（Git 即使冲突时文件内容，先确保 UI 显示真实状态）
   - **Rename**：成败都刷新（Drop 中间步骤失败可能丢失数据需要显示）

4. **索引安全策略**：多选 Drop **倒序**从大 index 向小 index 删除，避免 `stash@{n}` 重新编号错位。

5. **命令可组合外壳**：`handleStashSave(func(msg) error, action string)`，5 种入栈命令共享 Prompt + Log + Refresh 外壳，函数参数即差异内部实现多态。

6. **Git 版本兼容**：`SaveStagedChanges` 分 ≥2.35 一条命令 vs <2.356 步复合作旧方案，对上层 Controller 透明。
