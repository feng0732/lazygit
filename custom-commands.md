# Lazygit 自定义命令运行机制

## 整体架构概览

自定义命令的代码集中在 [custom_commands](pkg/gui/services/custom_commands) 包中，由以下核心文件组成：

| 文件 | 职责 |
|------|------|
| [client.go](pkg/gui/services/custom_commands/client.go) | 入口：遍历配置，生成键绑定或命令菜单 |
| [keybinding_creator.go](pkg/gui/services/custom_commands/keybinding_creator.go) | 将自定义命令映射为特定视图的键绑定 |
| [handler_creator.go](pkg/gui/services/custom_commands/handler_creator.go) | 核心：创建按键处理函数，编排 prompt 递归链与最终命令执行 |
| [resolver.go](pkg/gui/services/custom_commands/resolver.go) | 模板解析：将 Go 模板字符串解析为实际值 |
| [session_state_loader.go](pkg/gui/services/custom_commands/session_state_loader.go) | 加载当前会话状态（选中项等）供模板使用 |
| [menu_generator.go](pkg/gui/services/custom_commands/menu_generator.go) | 从命令输出解析出菜单条目（menuFromCommand 类型） |
| [models.go](pkg/gui/services/custom_commands/models.go) | Shim 模型层：隔离内部模型与用户模板 API |

配置结构定义在 [user_config.go](pkg/config/user_config.go#L696-L785)，实际命令执行在 [custom.go](pkg/commands/git_commands/custom.go)。

---

## 一、配置结构

用户在 `config.yml` 中通过 `customCommands` 数组定义自定义命令。每个条目对应一个 `CustomCommand` 结构体（[user_config.go#L696-L719](pkg/config/user_config.go#L696-L719)）：

```yaml
customCommands:
  - key: <按键>
    context: <上下文>        # 如 files, localBranches, global 等
    command: <命令模板>       # 如 git checkout {{.Form.Branch}}
    description: <描述>
    output: <输出方式>        # none | terminal | log | logWithPty | popup
    outputTitle: <弹窗标题>
    loadingText: <等待文本>
    prompts:                  # 交互提示链
      - type: input|menu|confirm|menuFromCommand
        key: <表单键名>
        title: <提示标题>
        # ... 各类型特有字段
    after:
      checkForConflicts: true
    commandMenu:              # 嵌套菜单（与 command 二选一）
      - key: ...
        command: ...
```

关键子结构：

- **`CustomCommandPrompt`**（[user_config.go#L729-L767](pkg/config/user_config.go#L729-L767)）：定义交互提示，`type` 决定提示类型，`key` 用于在模板中通过 `{{.Form.xxx}}` 引用用户输入值，`condition` 可控制提示是否跳过。
- **`CustomCommandMenuOption`**（[user_config.go#L776-L785](pkg/config/user_config.go#L776-L785)）：menu 类型提示的选项，包含 `name`、`description`、`value`、`key`。
- **`CustomCommandSuggestions`**（[user_config.go#L769-L774](pkg/config/user_config.go#L769-L774)）：input 类型提示的自动补全，`preset` 和 `command` 互斥。
- **`CustomCommandAfterHook`**（[user_config.go#L692-L694](pkg/config/user_config.go#L692-L694)）：命令执行后的钩子，目前仅支持 `checkForConflicts`。

---

## 二、入口：从配置到键绑定

### 2.1 Client.GetCustomCommandKeybindings

入口函数是 [Client.GetCustomCommandKeybindings](pkg/gui/services/custom_commands/client.go#L39-L64)，在 [keybindings.go#L344](pkg/gui/keybindings.go#L344) 中被调用，生成的自定义键绑定会**放在默认键绑定前面**（[keybindings.go#L349](pkg/gui/keybindings.go#L349)），从而获得更高的匹配优先级。

它遍历 `UserConfig().CustomCommands`，对每个命令做分支判断：

1. **有 `commandMenu`**：创建全局键绑定（`ViewName: ""`），handler 指向 `showCustomCommandsMenu`——弹出一个菜单列出子命令。该键绑定的 `OpensMenu: true`。
2. **无 `commandMenu`**：走正常流程——先由 `handlerCreator.call(customCommand)` 生成 handler 函数，再由 `keybindingCreator.call(customCommand, handler)` 将 handler 包装成特定上下文视图上的键绑定。

### 2.2 KeybindingCreator.call 与上下文映射

[keybindingCreator.call](pkg/gui/services/custom_commands/keybinding_creator.go#L25-L43) 负责将 handler 绑定到正确的视图上。

上下文解析逻辑在 [getViewNamesAndContexts](pkg/gui/services/custom_commands/keybinding_creator.go#L45-L66)：

- `context` 为 `"global"` 时，返回 `[""]`（空字符串表示全局绑定）
- 否则，将 `context` 字符串按逗号拆分为多个 context key
- 每个 context key 通过 [contextForContextKey](pkg/gui/services/custom_commands/keybinding_creator.go#L68-L76) 查找对应的 Context，再取其 `GetViewName()`
- 一个 context key 对应一个 view name；多个 context 会生成多个键绑定

注意：**命令菜单（commandMenu）的顶级键绑定永远是全局的**（`ViewName: ""`），上下文过滤在子命令展开时才进行（参见下文 §3.2）。

---

## 三、菜单入口：从菜单按键到叶子命令

### 3.1 showCustomCommandsMenu 递归展开

[showCustomCommandsMenu](pkg/gui/services/custom_commands/client.go#L66-L110) 是 commandMenu 类型命令的渲染入口。它递归处理嵌套的 `commandMenu`，构建 `menuItems` 数组：

```
自定义命令按键
  │
  ▼
GetCustomCommandKeybindings() 发现有 commandMenu
  │
  ▼
注册全局键绑定，handler = showCustomCommandsMenu(顶层cmd)
  │
  ├─ 用户按下菜单键
  │    │
  │    ▼
  │  showCustomCommandsMenu() 被调用
  │    │
  │    ├─ 遍历 commandMenu 子项
  │    │    │
  │    │    ├─ 子项也有 commandMenu ──→ OnPress 递归调用 showCustomCommandsMenu（OpensMenu: true）
  │    │    │
  │    │    └─ 子项是叶子（有 command）
  │    │         │
  │    │         ├─ 上下文过滤（详见 §3.2）
  │    │         │
  │    │         └─ OnPress = handlerCreator.call(子项)  ──→ 进入交互提示链
  │    │
  │    ├─ 空菜单兜底（详见 §3.3）
  │    │
  │    └─ 调用 self.c.Menu() 渲染菜单
```

### 3.2 上下文过滤：只显示当前可用的子命令

叶子子命令的上下文过滤发生在 `showCustomCommandsMenu` 内部（[client.go#L80-L91](pkg/gui/services/custom_commands/client.go#L80-L91)）：

```go
if subCommand.Context != "" && subCommand.Context != "global" {
    viewNames, err := self.keybindingCreator.getViewNamesAndContexts(subCommand)
    // ...
    currentView := self.c.GocuiGui().CurrentView()
    enabled := currentView != nil && lo.Contains(viewNames, currentView.Name())
    if !enabled {
        continue  // 不加入菜单
    }
}
```

要点：

1. 只有 `context` 非空且不是 `"global"` 的子命令才需要过滤
2. 过滤时机是**菜单弹出时**（即按键按下时），而非初始化时
3. 过滤依据是 `CurrentView().Name()` 是否在该子命令对应的 view names 列表中
4. 不匹配的子命令直接 `continue`，不会出现在菜单里

这意味着：同一个 commandMenu，在不同视图下按下，显示的可用命令可能不同。

### 3.3 空菜单兜底：NoApplicableCommandsInThisContext

如果所有叶子子命令都被上下文过滤掉了（`menuItems` 为空），会触发兜底逻辑（[client.go#L101-L106](pkg/gui/services/custom_commands/client.go#L101-L106)）：

```go
if len(menuItems) == 0 {
    menuItems = append(menuItems, &types.MenuItem{
        Label:   self.c.Tr.NoApplicableCommandsInThisContext,
        OnPress: func() error { return nil },
    })
}
```

这个兜底项是一个不可执行的占位菜单项，`OnPress` 什么也不做。菜单标题由 [getCustomCommandsMenuDescription](pkg/gui/services/custom_commands/client.go#L112-L118) 决定——有 `description` 用 description，否则用 `tr.CustomCommands`（翻译后的"自定义命令"）。

菜单通过 `self.c.Menu(types.CreateMenuOptions{..., HideCancel: true})` 弹出，`HideCancel: true` 表示不显示取消选项。

### 3.4 叶子命令如何进入交互提示链

对于叶子命令（有 `command` 的项），菜单条目的 `OnPress` 是 `self.handlerCreator.call(subCommand)` 返回的函数。注意这里不是立即调用，而是**在菜单条目的 OnPress 闭包里被调用**——也就是说，只有用户选中这个菜单项时，才会真正开始执行 prompt 链。

因为 `handlerCreator.call(subCommand)` 返回的是一个 `func() error`，而 `MenuItem.OnPress` 也是 `func() error`，类型完全匹配，所以可以直接赋值。

---

## 四、交互提示链：闭包递归包装的精妙设计

### 4.1 核心机制：从后向前构建闭包链

[HandlerCreator.call](pkg/gui/services/custom_commands/handler_creator.go#L47-L131) 是整个自定义命令运行机制的核心。它通过**从后向前**遍历 prompts，用闭包一层一层包装，最终形成一条从第一个 prompt 到 finalHandler 的调用链。

```
用户按下叶子命令
  │
  ├─ sessionStateLoader.call() ── 快照当前选中项等状态
  ├─ promptResponses := make([]string, N)  ── 按索引存响应（已废弃，保留兼容）
  ├─ form := make(map[string]string)  ── 按 key 存响应
  │
  ├─ f = finalHandler  ── 最内层：真正执行命令
  │
  ┌─ reverseIdx=0 ───────────────────────────────────┐
  │  g = f  (即 finalHandler)                          │
  │  idx = N-1  (最后一个 prompt)                      │
  │  wrappedF = func(response) { 写入响应; g() }       │
  │  f = 弹出 prompt[N-1] 的闭包                        │
  │                                                     │
  ├─ reverseIdx=1 ───────────────────────────────────┤
  │  g = f  (即 prompt[N-1] 的闭包)                    │
  │  idx = N-2                                        │
  │  wrappedF = func(response) { 写入响应; g() }       │
  │  f = 弹出 prompt[N-2] 的闭包                        │
  │                                                     │
  │                 ...                                │
  │                                                     │
  ├─ reverseIdx=N-1 ─────────────────────────────────┤
  │  g = f  (即 prompt[1] 的闭包)                      │
  │  idx = 0  (第一个 prompt)                          │
  │  wrappedF = func(response) { 写入响应; g() }       │
  │  f = 弹出 prompt[0] 的闭包                        │
  └───────────────────────────────────────────────────┘
  │
  └─ 返回 f()  —— 这就是按键 handler，调用时从 prompt[0] 开始
```

为什么要从后向前？因为每一层需要知道"下一步"是什么（即 `g`），而 `wrappedF` 必须捕获这个 `g` 才能在用户确认后调用它。从后向前构建保证了每一步的 `g` 都是已经构建好的下一步。

### 4.2 wrappedF：响应的记录与传递

`wrappedF` 是每个 prompt 与下一步之间的衔接点（[handler_creator.go#L65-L69](pkg/gui/services/custom_commands/handler_creator.go#L65-L69)）：

```go
wrappedF := func(response string) error {
    promptResponses[idx] = response  // 按索引存（兼容旧版）
    form[prompt.Key] = response      // 按 key 存（推荐用法）
    return g()                       // 调用下一步
}
```

它做了三件事：
1. 把用户响应存入 `promptResponses` 数组（按索引，已废弃）
2. 把用户响应存入 `form` map（按 key，`{{.Form.xxx}}` 用的就是这个）
3. 调用 `g()` 进入下一层（下一个 prompt 或 finalHandler）

### 4.3 resolveTemplate 的动态构建

每一轮循环都会调用 [getResolveTemplateFn](pkg/gui/services/custom_commands/handler_creator.go#L273-L286) 重新生成一个 `resolveTemplate` 函数。这个函数捕获了当前的 `form`、`promptResponses`、`sessionState`。

关键点：**`form` 是一个 map，是引用类型**。虽然 `resolveTemplate` 在循环中创建时 `form` 还是空的或部分填充的，但由于闭包捕获的是 map 的引用，当实际执行模板解析时（即用户按下键、prompt 弹出前），`form` 中已经包含了前面所有 prompt 的响应。

这就是为什么后续 prompt 的标题、选项等可以引用前序 prompt 的用户输入——因为每次 `resolvePrompt` 时 `form` 里已经有了前面的值。

### 4.4 四种 Prompt 类型的衔接方式

| 类型 | 衔接函数 | 是否使用 wrappedF | 说明 |
|------|---------|-----------------|------|
| `input` | [inputPrompt](pkg/gui/services/custom_commands/handler_creator.go#L144-L162) | 是 | 输入框确认后调用 `wrappedF(str)` |
| `menu` | [menuPrompt](pkg/gui/services/custom_commands/handler_creator.go#L228-L240) | 是 | 选菜单项后调用 `wrappedF(option.Value)` |
| `menuFromCommand` | [menuPromptFromCommand](pkg/gui/services/custom_commands/handler_creator.go#L242-L265) | 是 | 先跑命令生成菜单，选中后调用 `wrappedF(candidate.value)` |
| `confirm` | [confirmPrompt](pkg/gui/services/custom_commands/handler_creator.go#L218-L226) | **否，直接用 g** | 确认后调用 `g()`，不产生响应值 |

注意 `confirm` 类型的特殊之处：它直接把 `g` 传给 `confirmPrompt` 作为 `handleConfirm`，而不是 `wrappedF`。因为确认框不收集用户输入，只是一个门控。

### 4.5 condition：有条件的 prompt

如果 prompt 设置了 `condition`，会在 prompt 闭包外再包一层条件判断（[handler_creator.go#L110-L126](pkg/gui/services/custom_commands/handler_creator.go#L110-L126)）：

```go
if prompt.Condition != "" {
    showPrompt := f
    conditionTemplate := prompt.Condition
    f = func() error {
        resolved, err := resolveCondition(conditionTemplate, resolveTemplate)
        if err != nil { return err }
        if resolved {
            return showPrompt()  // 条件为真：显示 prompt
        }
        if _, exists := form[prompt.Key]; !exists {
            form[prompt.Key] = ""  // 条件为假：填空值占位
        }
        return g()  // 跳过，直接进入下一步
    }
}
```

[resolveCondition](pkg/gui/services/custom_commands/handler_creator.go#L133-L142) 的规则：
- 空字符串 → false
- `"false"` → false
- 其他非空值 → true

因此你可以用 Go 模板函数写条件，例如 `{{ eq .Form.Choice "yes" }}`，模板解析后为 `"true"` 或 `"false"`。

条件为假时，也会在 form 里填一个空字符串——这保证了最终命令模板里引用的 key 一定存在，不会因为 `missingkey=error` 而报错。

### 4.6 menuFromCommand 的特殊流程

`menuFromCommand` 类型比较特殊，它需要先执行命令才能知道菜单选项：

1. `menuPromptFromCommand` 被调用
2. 运行 `prompt.Command` 拿到输出（[handler_creator.go#L244](pkg/gui/services/custom_commands/handler_creator.go#L244)）
3. 通过 [menuGenerator.call](pkg/gui/services/custom_commands/menu_generator.go#L30-L50) 解析输出为菜单条目
4. 解析过程：按行分割 → 每行用 filter 正则匹配 → 用 valueFormat/labelFormat 模板格式化
5. 构造 MenuItem，OnPress 调用 `wrappedF(candidate.value)`
6. 弹出菜单

[MenuGenerator](pkg/gui/services/custom_commands/menu_generator.go) 的解析细节：

- `filter` 为空且 `valueFormat` 为空且 `labelFormat` 为空时，每行原样作为 label 和 value
- 否则编译 filter 正则，用命名捕获组提取数据
- 组索引（`group_0`, `group_1`...）和组名同时可用
- `labelFormat` 支持颜色函数（通过 `style.TemplateFuncMapAddColors`）
- `TrimmerTemplate` 包装了 Go template，自动 trim 输出

---

## 五、变量解析：模板系统

### 5.1 模板数据对象

模板解析的核心函数是 [getResolveTemplateFn](pkg/gui/services/custom_commands/handler_creator.go#L273-L286)，它构造了模板数据对象 `CustomCommandObjects`：

```go
type CustomCommandObjects struct {
    *SessionState        // 当前会话状态（选中项等）
    PromptResponses []string  // 按序号的 prompt 响应（已废弃，保留兼容）
    Form            map[string]string  // 按 key 的 prompt 响应
}
```

用户在模板中可以通过以下路径访问数据：

| 模板表达式 | 含义 |
|-----------|------|
| `{{.SelectedFile.Name}}` | 当前选中的文件名 |
| `{{.SelectedLocalBranch.Name}}` | 当前选中的本地分支 |
| `{{.SelectedCommit.Hash}}` | 当前选中的 commit hash |
| `{{.Form.Branch}}` | 之前 prompt 中 key 为 "Branch" 的用户输入 |
| `{{.PromptResponses.0}}` | 第 1 个 prompt 的响应（已废弃） |
| `{{.CheckedOutBranch.Name}}` | 当前检出的分支 |
| `{{.SelectedPath}}` | 当前选中的路径 |
| `{{.SelectedCommitRange.From}}` / `{{.SelectedCommitRange.To}}` | 选中的 commit 范围 |

此外还有两个模板函数：

- **`quote`**：对字符串进行 shell 引号包裹（`self.c.OS().Quote`）
- **`runCommand`**：在模板解析时同步执行命令并返回单行输出（[TemplateFunctionRunCommand](pkg/commands/git_commands/custom.go#L28-L39)），如果输出含多行则报错

### 5.2 Resolver.resolvePrompt 可复用性

[Resolver.resolvePrompt](pkg/gui/services/custom_commands/resolver.go#L17-L70) 是一个独立的纯函数式模块，接收 `resolveTemplate` 函数作为参数，因此**不依赖具体的模板数据源**。

它解析 prompt 中所有含模板的字段：
- `Title`、`InitialValue`、`Suggestions.Preset`、`Suggestions.Command`
- `Body`、`Command`、`Filter`
- menu 类型的 `Options`（递归解析每个 option 的 Name/Description/Value）

这种设计使得 Resolver 可以在不同上下文中复用——只要提供一个 `func(string) (string, error)` 类型的模板解析函数即可。

`resolveTemplate` 本身也在多处被复用：
- prompt 标题解析
- prompt 初始值解析
- menu 选项解析
- condition 条件解析
- 最终命令字符串解析
- popup 标题解析

### 5.3 SessionState 加载

[SessionStateLoader.call](pkg/gui/services/custom_commands/session_state_loader.go#L215-L259) 在用户按下键绑定时被调用，快照当前 GUI 状态：

- 从各 Context 获取选中项（`GetSelected()`），通过 shim 函数转换为稳定 API 模型
- `SelectedCommit` 的解析有优先级逻辑：如果当前上下文是 reflog 或 subCommits，则使用对应的 commit，否则使用 localCommits
- `SelectedPath` 根据当前上下文决定来源：如果当前在 commitFiles 上下文则取 commit file 路径，否则取 files 的路径

**Shim 层设计意图**（[models.go](pkg/gui/services/custom_commands/models.go#L8-L14)）：为内部模型类创建 shim，使自定义命令的 API 更稳定。例如 `Commit.Sha` 被废弃改为 `Commit.Hash`，shim 中同时保留了两者。

### 5.4 模板引擎

底层调用 [utils.ResolveTemplate](pkg/utils/template.go#L9-L21)——标准 Go `text/template`，开启 `missingkey=error`。这意味着模板中引用了不存在的字段会报错，而不是默默输出空值。

---

## 六、最终命令执行：结果如何衔接

### 6.1 finalHandler 入口

[finalHandler](pkg/gui/services/custom_commands/handler_creator.go#L288-L344) 在所有 prompt 链执行完毕后被调用。它是整个执行流程的终点。

注意：`finalHandler` 的参数 `sessionState`、`promptResponses`、`form` 都是**在 `handlerCreator.call` 里定义、被闭包捕获**的变量。当 finalHandler 执行时，form 已经被所有前面的 prompt 填满了。

### 6.2 命令字符串的最终解析

finalHandler 做的第一件事就是再次调用 `getResolveTemplateFn` 生成 `resolveTemplate`，然后用它解析 `customCommand.Command`：

```go
resolveTemplate := self.getResolveTemplateFn(form, promptResponses, sessionState)
cmdStr, err := resolveTemplate(customCommand.Command)
```

这是**最后一次、也是最终的模板解析**——此时 Form 里有所有 prompt 的响应，SessionState 是按键时的快照，runCommand 可以被用来做最终的动态计算。

### 6.3 输出模式与执行方式

解析出 `cmdStr` 后，创建 shell 命令对象，然后根据 `output` 字段选择执行方式：

| output 值 | 执行方式 | 刷新行为 |
|-----------|---------|---------|
| `terminal` | `RunSubprocessAndRefresh`：暂停 lazygit，在真实终端运行 | 子进程结束后自动刷新 |
| `log` | `WithWaitingStatus` 中执行，`StreamOutput()` 流到命令日志面板 | ASYNC 异步刷新 |
| `logWithPty` | 同 log，但额外 `UsePty()`（伪终端，保留彩色输出） | ASYNC 异步刷新 |
| `popup` | 同步执行获取输出，然后 `Alert` 弹窗展示 | ASYNC 异步刷新 |
| `none`（默认） | `WithWaitingStatus` 中执行，输出丢弃 | ASYNC 异步刷新 |

`WithWaitingStatus` 会在状态栏显示加载文字——优先使用 `customCommand.LoadingText`，否则用翻译的 `RunningCustomCommandStatus`（[handler_creator.go#L301-L304](pkg/gui/services/custom_commands/handler_creator.go#L301-L304)）。

### 6.4 popup 输出的标题处理

对于 `popup` 模式，标题也支持模板（[handler_creator.go#L332-L338](pkg/gui/services/custom_commands/handler_creator.go#L332-L338)）：

- 如果 `customCommand.OutputTitle` 非空，用 `resolveTemplate` 解析它作为标题
- 否则用 `cmdStr`（解析后的完整命令字符串）作为标题
- 空输出时显示翻译的 "Empty output" 而不是空白弹窗

### 6.5 执行后钩子

如果命令执行出错且 `after.checkForConflicts` 为 true，则调用 `mergeAndRebaseHelper.CheckForConflicts(err)` 检查是否存在合并冲突。

无论命令成功或失败，都会调用 `self.c.Refresh(types.RefreshOptions{Mode: types.ASYNC})` 触发异步刷新。

---

## 七、完整执行流程图

```
用户按下自定义命令键
  │
  ├─ 顶层是 commandMenu?
  │    │
  │    ├─ 是 → showCustomCommandsMenu(顶层命令)
  │    │       │
  │    │       ├─ 遍历子命令
  │    │       │    │
  │    │       │    ├─ 子命令也有 commandMenu
  │    │       │    │    → OnPress: 递归 showCustomCommandsMenu
  │    │       │    │
  │    │       │    └─ 子命令是叶子（有 command）
  │    │       │         ├─ context != "" && != "global" ?
  │    │       │         │    ├─ 是 → 检查 CurrentView 是否匹配
  │    │       │         │    │      不匹配 → continue (不出现在菜单中)
  │    │       │         │    └─ 否 → 直接加入菜单
  │    │       │         │
  │    │       │         └─ OnPress: handlerCreator.call(子命令)
  │    │       │
  │    │       ├─ menuItems 为空?
  │    │       │    └─ 是 → 添加 "NoApplicableCommandsInThisContext" 兜底项
  │    │       │
  │    │       └─ self.c.Menu() 弹出菜单
  │    │                  │
  │    │                  └─ 用户选中叶子项 → OnPress 触发
  │    │
  │    └─ 否 → 直接走 handlerCreator.call()
  │
  ▼
handlerCreator.call() 返回的函数被调用
  │
  ├─ sessionStateLoader.call() 快照当前状态
  ├─ 初始化 promptResponses[] 和 form{}
  ├─ f = finalHandler
  │
  ├─ 从后向前遍历 prompts（reverseIdx）
  │    ├─ g = f
  │    ├─ wrappedF = func(response) { 写入 form/promptResponses; g() }
  │    ├─ resolveTemplate = getResolveTemplateFn(form, ...)  ← 捕获 form 引用
  │    ├─ 根据 type 创建新 f（闭包）
  │    │    - input:   resolvePrompt → inputPrompt(resolvedPrompt, wrappedF)
  │    │    - menu:    resolvePrompt → menuPrompt(resolvedPrompt, wrappedF)
  │    │    - menuFromCommand: resolvePrompt → menuPromptFromCommand(...)
  │    │    - confirm: resolvePrompt → confirmPrompt(resolvedPrompt, g)  ← 不用 wrappedF
  │    └─ 有 condition? → 再包一层条件闭包
  │
  └─ f()  ← 开始执行 prompt 链
        │
        ▼
  prompt[0] 弹出
    用户输入/选择 → wrappedF(response)
      → 写入 form["Key0"]
      → 调用 g() → prompt[1]
        → resolvePrompt 时 form 已有 Key0 的值
          ...
            → prompt[N-1]
              → wrappedF(response)
                → 写入 form["KeyN-1"]
                → 调用 g() → finalHandler
                     │
                     ▼
                finalHandler()
                  ├─ resolveTemplate(command) → 得到 cmdStr
                  ├─ cmdObj = NewShell(cmdStr)
                  │
                  ├─ output == "terminal"
                  │    → RunSubprocessAndRefresh(cmdObj)
                  │
                  └─ 其他输出
                       → WithWaitingStatus(loadingText, func(task) {
                            ├─ log / logWithPty → StreamOutput() [+ UsePty()]
                            ├─ cmdObj.RunWithOutput()
                            ├─ Refresh(ASYNC)
                            ├─ 出错 && checkForConflicts → CheckForConflicts(err)
                            └─ output == "popup" → Alert(title, output)
                          })
```

---

## 八、代码引用可复用性分析

### 8.1 Resolver：纯函数式设计，高度可复用

[Resolver](pkg/gui/services/custom_commands/resolver.go) 采用依赖注入设计：

- 不持有任何业务状态，仅依赖传入的 `resolveTemplate func(string) (string, error)`
- 职责单一：只负责遍历 prompt 的各个字段并应用模板解析
- `resolveMenuOptions` 和 `resolveMenuOption` 是独立的子函数，也可单独复用
- 可以用于任何需要解析 `CustomCommandPrompt` 模板的场景

### 8.2 MenuGenerator：独立的命令输出解析器

[MenuGenerator](pkg/gui/services/custom_commands/menu_generator.go) 也是一个独立组件：

- 输入：命令输出字符串、filter 正则、valueFormat、labelFormat
- 输出：`[]*commandMenuItem`
- 不依赖 GUI 状态，可以脱离 lazygit 单独测试（有独立的单元测试 [menu_generator_test.go](pkg/gui/services/custom_commands/menu_generator_test.go)）
- `TrimmerTemplate` 包装了 Go template，自动 trim，是一个可复用的小工具
- `parseLine` 的正则命名组提取逻辑也可以单独复用

### 8.3 SessionStateLoader + models Shim：稳定 API 层

[SessionStateLoader](pkg/gui/services/custom_commands/session_state_loader.go) + [models.go](pkg/gui/services/custom_commands/models.go) 构成了一个"防崩溃层"：

- 内部模型可以随意改名、加字段
- Shim 层对外保持稳定 API
- 这种模式在需要暴露 API 给用户（如自定义命令、插件系统）时非常值得借鉴

### 8.4 HandlerCreator.call：编排模式，但耦合度高

[HandlerCreator.call](pkg/gui/services/custom_commands/handler_creator.go#L47-L131) 的闭包递归包装是核心设计，但也导致了较高的耦合：

- 依赖 Resolver、MenuGenerator、SessionStateLoader、SuggestionsHelper、MergeAndRebaseHelper
- prompt 类型判断用 switch-case，新增类型需要修改这里（不符合开闭原则）
- 但作为一个编排器（orchestrator），这种程度的耦合是合理的——它的职责就是把各个组件串起来

### 8.5 utils.ResolveTemplate：通用模板工具

[utils.ResolveTemplate](pkg/utils/template.go#L9-L21) 是整个项目通用的模板工具，不局限于自定义命令。它封装了 Go `text/template` 的常见用法（带 Funcs、带 missingkey=error、buffer 输出）。

### 8.6 KeybindingCreator：context→view 的映射器

[KeybindingCreator](pkg/gui/services/custom_commands/keybinding_creator.go) 的 `getViewNamesAndContexts` 和 `contextForContextKey` 提供了 context key 到 view name 的映射能力，在 `Client.showCustomCommandsMenu` 中也被复用（用于上下文过滤）。

---

## 九、关键设计洞察

1. **闭包递归包装**：prompt 链通过从后向前的闭包嵌套实现，每个闭包捕获自己的 `g`（下一步）和 `wrappedF`（记录响应后前进），这是整个机制最核心也最不易理解的设计。

2. **两阶段模板解析**：模板解析发生在两个时刻——SessionState 在按键时加载一次（不变），而 prompt 每一步都会重新解析模板（因为 Form 在不断被填充），这意味着后续 prompt 的 Title/Options 等可以引用前序 prompt 的响应。

3. **commandMenu 全局绑定 + 动态过滤**：commandMenu 的顶层键绑定是全局的，但子命令在菜单弹出时才根据当前视图做上下文过滤。这样做的好处是同一个菜单键在任何地方都能按，但只显示当前上下文适用的命令。

4. **空菜单兜底**：当所有子命令都被上下文过滤掉时，不会显示空菜单，而是显示 "No applicable commands in this context" 占位项，避免用户困惑。

5. **Shim 层隔离**：内部模型（`models.Commit` 等）通过 shim 转换为自定义命令专用模型，保证了 API 稳定性，允许内部重命名而不破坏用户配置。

6. **menuFromCommand 的正则+模板组合**：通过 Go 正则命名捕获组 + Go 模板格式化的组合，提供了灵活的命令输出解析能力，`group_0`、`group_1` 等数字索引和命名 key 同时可用。

7. **condition 的模板求值**：条件表达式本质上是一个模板，解析后非空且非 `"false"` 即为真，这允许用 Go 模板的比较函数（如 `eq`）做条件判断。条件为假时也会填空值占位，防止 `missingkey=error`。

8. **confirm 不记录响应**：confirm 类型直接调用 `g()` 而非 `wrappedF()`，因为它不产生值，仅作为确认门控。
