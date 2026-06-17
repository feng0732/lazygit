# Lazygit 自定义命令运行机制

## 整体架构概览

自定义命令的代码集中在 [custom_commands](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands) 包中，由以下核心文件组成：

| 文件 | 职责 |
|------|------|
| [client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/client.go) | 入口：遍历配置，生成键绑定或命令菜单 |
| [keybinding_creator.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/keybinding_creator.go) | 将自定义命令映射为特定视图的键绑定 |
| [handler_creator.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go) | 核心：创建按键处理函数，编排 prompt 递归链与最终命令执行 |
| [resolver.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/resolver.go) | 模板解析：将 Go 模板字符串解析为实际值 |
| [session_state_loader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/session_state_loader.go) | 加载当前会话状态（选中项等）供模板使用 |
| [menu_generator.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/menu_generator.go) | 从命令输出解析出菜单条目（menuFromCommand 类型） |
| [models.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/models.go) | Shim 模型层：隔离内部模型与用户模板 API |

配置结构定义在 [user_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/config/user_config.go#L696-L785)，实际命令执行在 [custom.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/commands/git_commands/custom.go)。

---

## 一、配置结构

用户在 `config.yml` 中通过 `customCommands` 数组定义自定义命令。每个条目对应一个 `CustomCommand` 结构体（[user_config.go#L696-L719](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/config/user_config.go#L696-L719)）：

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

- **`CustomCommandPrompt`**（[user_config.go#L729-L767](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/config/user_config.go#L729-L767)）：定义交互提示，`type` 决定提示类型，`key` 用于在模板中通过 `{{.Form.xxx}}` 引用用户输入值，`condition` 可控制提示是否跳过。
- **`CustomCommandMenuOption`**（[user_config.go#L776-L785](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/config/user_config.go#L776-L785)）：menu 类型提示的选项，包含 `name`、`description`、`value`、`key`。
- **`CustomCommandSuggestions`**（[user_config.go#L769-L774](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/config/user_config.go#L769-L774)）：input 类型提示的自动补全，`preset` 和 `command` 互斥。
- **`CustomCommandAfterHook`**（[user_config.go#L692-L694](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/config/user_config.go#L692-L694)）：命令执行后的钩子，目前仅支持 `checkForConflicts`。

---

## 二、入口：从配置到键绑定

### 2.1 Client.GetCustomCommandKeybindings

入口函数是 [Client.GetCustomCommandKeybindings](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/client.go#L39-L64)，在 [keybindings.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/keybindings.go#L344) 中被调用。

它遍历 `UserConfig().CustomCommands`，对每个命令做分支判断：

1. **有 `commandMenu`**：创建全局键绑定（`ViewName: ""`），handler 指向 `showCustomCommandsMenu`——弹出一个菜单列出子命令。该键绑定的 `OpensMenu: true`。
2. **无 `commandMenu`**：走正常流程——先由 `handlerCreator.call(customCommand)` 生成 handler 函数，再由 `keybindingCreator.call(customCommand, handler)` 将 handler 包装成特定上下文视图上的键绑定。

### 2.2 命令菜单（commandMenu）

[showCustomCommandsMenu](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/client.go#L66-L110) 递归处理嵌套的 `commandMenu`：

- 对每个子命令，如果自身也有 `commandMenu`，则 `OnPress` 递归调用 `showCustomCommandsMenu`，`OpensMenu: true`。
- 如果没有 `commandMenu`，但子命令有非 `global` 的 `context`，则检查当前视图是否匹配该上下文，不匹配则 `continue` 跳过该子命令。
- 最终通过 `self.c.Menu(...)` 渲染菜单。

### 2.3 KeybindingCreator.call

[keybindingCreator.call](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/keybinding_creator.go#L25-L43) 负责将 handler 绑定到正确的视图上：

- `context` 为 `"global"` 时，`ViewName: ""`（全局生效）。
- 否则，将 `context` 字符串按逗号拆分，查找每个 context key 对应的视图名称，为每个视图创建一个 `Binding`。

---

## 三、变量解析：模板系统

### 3.1 模板数据对象

模板解析的核心函数是 [getResolveTemplateFn](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L273-L286)，它构造了模板数据对象 `CustomCommandObjects`：

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
| `{{.SelectedCommitRange.From}}` | 选中的 commit 范围起始 |
| `{{.SelectedCommitRange.To}}` | 选中的 commit 范围结束 |

此外还有两个模板函数：

- **`quote`**：对字符串进行 shell 引号包裹（`self.c.OS().Quote`）
- **`runCommand`**：在模板解析时同步执行命令并返回单行输出（[TemplateFunctionRunCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/commands/git_commands/custom.go#L28-L39)），如果输出含多行则报错

### 3.2 SessionState 加载

[SessionStateLoader.call](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/session_state_loader.go#L215-L259) 在用户按下键绑定时被调用，快照当前 GUI 状态：

- 从各 Context 获取选中项（`GetSelected()`），通过 shim 函数转换为稳定 API 模型
- `SelectedCommit` 的解析有优先级逻辑：如果当前上下文是 reflog 或 subCommits，则使用对应的 commit，否则使用 localCommits
- `SelectedPath` 根据当前上下文决定来源：如果当前在 commitFiles 上下文则取 commit file 路径，否则取 files 的路径

**Shim 层设计意图**（[models.go](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/models.go#L8-L14)）：为内部模型类创建 shim，使自定义命令的 API 更稳定。例如 `Commit.Sha` 被废弃改为 `Commit.Hash`，shim 中同时保留了两者。

### 3.3 Resolver.resolvePrompt

[Resolver.resolvePrompt](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/resolver.go#L17-L70) 对 prompt 中的所有模板字符串进行解析，包括：

- `Title`、`InitialValue`、`Suggestions.Preset`、`Suggestions.Command`、`Body`、`Command`、`Filter`
- 如果是 `menu` 类型，还会递归解析每个 `Option` 的 `Name`、`Description`、`Value`

`resolveTemplate` 函数由 `getResolveTemplateFn` 提供，底层调用 [utils.ResolveTemplate](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/utils/template.go#L9-L21)——标准 Go `text/template`，开启 `missingkey=error`。

### 3.4 条件解析

[prompt.Condition](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L110-L127) 也是一个模板表达式，通过 [resolveCondition](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L133-L142) 解析：

- 解析结果去除空白后，如果为空字符串或 `"false"` 则条件为假
- 条件为假时跳过该 prompt，直接调用下一个 handler（`g()`），并将该 prompt 的 key 设为空字符串写入 Form

---

## 四、Prompt 递归链：菜单入口机制

### 4.1 核心设计：闭包递归包装

[HandlerCreator.call](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L47-L131) 是整个自定义命令运行机制最精巧的部分。它通过**从后向前**遍历 prompts，用闭包递归包装形成一条 prompt 链：

```
按键 → f() → prompt[0] → 用户响应 → prompt[1] → ... → prompt[N] → finalHandler()
```

具体步骤：

1. 初始化 `promptResponses` 数组和 `form` map
2. 设置 `f = finalHandler`（最终命令执行）
3. **从后向前**遍历 prompts（`idx = len(prompts) - 1 - reverseIdx`），每一轮：
   - 保存当前 `f` 为 `g`
   - 创建 `wrappedF`：将用户响应写入 `promptResponses[idx]` 和 `form[prompt.Key]`，然后调用 `g()`
   - 根据 prompt 类型创建新的 `f`：先 `resolvePrompt` 解析模板，再弹出对应 UI
   - 如果有 `condition`，再用条件判断包装 `f`
4. 最终返回的 `f()` 就是按键 handler

这样，按下键后依次弹出 prompt，每个 prompt 的用户响应被记录到 Form 中，供后续 prompt 和最终命令的模板使用。

### 4.2 四种 Prompt 类型

#### input（[handler_creator.go#L144-L162](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L144-L162)）

弹出输入框，支持初始值和自动补全。自动补全有两种来源（互斥）：
- **preset**：内置补全（authors/branches/files/refs/remotes/remoteBranches/tags）
- **command**：运行 shell 命令，每行输出作为一个补全建议

#### menu（[handler_creator.go#L228-L240](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L228-L240)）

弹出选择菜单，从配置的 `Options` 中生成菜单条目。每个选项的 `OnPress` 调用 `wrappedF(option.Value)`，将选中值传入。

#### menuFromCommand（[handler_creator.go#L242-L265](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L242-L265)）

先运行 `prompt.Command` 获取命令输出，再通过 `menuGenerator.call()` 将输出解析为菜单条目：

1. 运行命令获取输出
2. 按行分割输出
3. 对每行用 `filter` 正则匹配，提取命名捕获组
4. 用 `valueFormat` 和 `labelFormat` 模板格式化匹配结果

#### confirm（[handler_creator.go#L218-L226](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L218-L226)）

弹出确认对话框，确认后执行 `g()`（注意：confirm 不需要 `wrappedF`，因为它不产生响应值）。

### 4.3 menuGenerator 解析细节

[MenuGenerator.call](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/menu_generator.go#L30-L50) 的解析流程：

1. 如果 `filter`、`valueFormat`、`labelFormat` 全为空，则每行原样作为 label 和 value
2. 否则编译 `filter` 为正则
3. 对每行调用 `parseLine`（[menu_generator.go#L117-L133](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/menu_generator.go#L117-L133)）：
   - 用正则 `FindAllStringSubmatch` 匹配
   - 按组索引存为 `group_0`、`group_1`...
   - 命名捕获组额外以名称为 key 存储
4. 用 `valueFormat` 模板渲染值，`labelFormat` 渲染标签（支持颜色函数）
5. 如果没有 `labelFormat`，则使用 `valueFormat` 作为标签模板

---

## 五、最终命令执行：结果如何衔接

### 5.1 finalHandler

[finalHandler](file:///d:/fz/0601-2/solo-dogfeeding/code/28-lazygit/pkg/gui/services/custom_commands/handler_creator.go#L288-L344) 在所有 prompt 链执行完毕后被调用：

1. 再次获取 `resolveTemplate` 函数（此时 Form 已填满所有 prompt 响应）
2. 用 `resolveTemplate(customCommand.Command)` 解析最终命令字符串
3. 创建 shell 命令对象 `cmdObj`
4. 根据 `output` 字段选择执行方式

### 5.2 输出模式

| output 值 | 行为 |
|-----------|------|
| `terminal` | 调用 `RunSubprocessAndRefresh`：暂停 lazygit，在终端中运行命令，完成后刷新 |
| `log` | 在 WithWaitingStatus 中执行，开启 StreamOutput，输出流到命令日志面板 |
| `logWithPty` | 同 log，但额外使用 PTY（伪终端），可保留彩色输出 |
| `popup` | 执行后获取输出，用 `Alert` 弹窗展示。弹窗标题优先使用 `outputTitle`（也可含模板），否则使用命令字符串本身。空输出时显示 "Empty output" |
| `none`（默认） | 在 WithWaitingStatus 中执行，输出被丢弃 |

### 5.3 执行后钩子

如果命令执行出错且 `after.checkForConflicts` 为 true，则调用 `mergeAndRebaseHelper.CheckForConflicts(err)` 检查是否存在合并冲突。

无论命令成功或失败，都会触发异步刷新 `self.c.Refresh(types.RefreshOptions{Mode: types.ASYNC})`。

---

## 六、完整执行流程图

```
用户按键
  │
  ├─ 有 commandMenu? ──是──→ showCustomCommandsMenu()
  │                            │
  │                            ├─ 子项有 commandMenu → 递归
  │                            └─ 子项无 commandMenu → handlerCreator.call()
  │
  └─ 无 commandMenu → handlerCreator.call()
                          │
                          ▼
                    加载 SessionState（快照当前选中项等）
                    初始化 promptResponses[], form{}
                    设置 f = finalHandler
                    │
                    ┌─ 从后向前遍历 prompts ─┐
                    │                          │
                    │  保存当前 f 为 g          │
                    │  创建 wrappedF（记录响应后调用 g）
                    │  resolvePrompt 解析模板   │
                    │  根据 type 创建新 f       │
                    │    - input: 输入框        │
                    │    - menu: 选择菜单       │
                    │    - menuFromCommand: 命令菜单
                    │    - confirm: 确认框      │
                    │  如有 condition 再包装     │
                    │                          │
                    └──────────────────────────┘
                          │
                          ▼
                    返回 f() 作为按键 handler
                          │
                          ▼ （运行时）
                    依次弹出 prompt → 用户响应 → 记入 Form
                          │
                          ▼
                    finalHandler()
                      ├─ resolveTemplate(command) 解析最终命令
                      ├─ 创建 cmdObj
                      └─ 根据 output 执行
                           ├─ terminal: 暂停 lazygit，终端运行
                           ├─ log/logWithPty: 流式输出到日志面板
                           ├─ popup: 弹窗显示输出
                           └─ none: 静默执行
```

---

## 七、关键设计洞察

1. **闭包递归包装**：prompt 链通过从后向前的闭包嵌套实现，每个闭包捕获自己的 `g`（下一步）和 `wrappedF`（记录响应后前进），这是整个机制最核心也最不易理解的设计。

2. **两阶段模板解析**：模板解析发生在两个时刻——SessionState 在按键时加载一次（不变），而 prompt 每一步都会重新解析模板（因为 Form 在不断被填充），这意味着后续 prompt 的 Title/Options 等可以引用前序 prompt 的响应。

3. **Shim 层隔离**：内部模型（`models.Commit` 等）通过 shim 转换为自定义命令专用模型，保证了 API 稳定性，允许内部重命名而不破坏用户配置。

4. **menuFromCommand 的正则解析**：通过 Go 正则命名捕获组 + Go 模板格式化的组合，提供了灵活的命令输出解析能力，`group_0`、`group_1` 等数字索引和命名 key 同时可用。

5. **condition 的模板求值**：条件表达式本质上是一个模板，解析后非空且非 `"false"` 即为真，这允许用 Go 模板的比较函数（如 `eq`）做条件判断。

6. **confirm 不记录响应**：confirm 类型直接调用 `g()` 而非 `wrappedF()`，因为它不产生值，仅作为确认门控。
