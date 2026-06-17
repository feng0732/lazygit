# 日志命令与执行封装分层分析

## 整体架构概览

Lazygit 的命令执行与日志系统采用了清晰的分层架构，从命令构造到结果展示经过多层封装，每层职责明确：

```
┌─────────────────────────────────────────────────────────┐
│                   GUI 控制层 (Controllers)               │
│  触发命令、记录动作描述、处理用户交互                       │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│                   命令构造层 (Builders)                   │
│  GitCommandBuilder / CmdObjBuilder / gitCmdObjBuilder   │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│                   命令对象层 (CmdObj)                     │
│  封装 exec.Cmd，提供链式配置 API (DontLog/StreamOutput)  │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│                   命令运行层 (Runners)                    │
│  cmdObjRunner / gitCmdObjRunner (重试逻辑)               │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│                   GUI IO 桥接层 (guiIO)                   │
│  logCommandFn / newCmdWriterFn / promptForCredentialFn  │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│                   结果展示层 (Views)                      │
│  Extras 面板 / 主视图 / 命令日志面板                       │
└──────────────────────────────────────────────────────────┘
```

---

## 一、命令构造层

### 1.1 GitCommandBuilder - Git 命令参数构建器

**文件**: [git_command_builder.go](pkg/commands/git_commands/git_command_builder.go)

**职责**: 以流式 API 构建 git 命令的参数列表，支持条件参数。

**核心方法**:
- `NewGitCmd(command string)` - 创建命令构建器
- `Arg(args ...string)` - 添加参数
- `ArgIf(condition bool, ifTrue ...string)` - 条件添加参数
- `Config(value string)` - 添加 git 配置（`-c` 参数，位于命令名之前）
- `Dir(path string)` - 添加工作目录（`-C` 参数）
- `ToArgv()` - 转换为完整参数数组（含 `git` 前缀）

**示例**:
```go
cmdArgs := NewGitCmd("log").
    Arg("HEAD").
    ArgIf(all, "--all").
    Arg("--oneline").
    ToArgv()
// 结果: ["git", "log", "HEAD", "--all", "--oneline"]
```

### 1.2 CmdObjBuilder - 命令对象构建器

**文件**: [cmd_obj_builder.go](pkg/commands/oscommands/cmd_obj_builder.go)

**职责**: 将参数数组包装为可执行的 CmdObj，并注入 runner。

**核心方法**:
- `New(args []string) *CmdObj` - 从参数数组创建命令对象
- `NewShell(commandStr string, shellFunctionsFile string) *CmdObj` - 从 shell 命令字符串创建
- `Quote(str string) string` - 平台相关的字符串转义
- `CloneWithNewRunner(decorate func(ICmdObjRunner) ICmdObjRunner)` - 用装饰器模式替换 runner

### 1.3 gitCmdObjBuilder - Git 命令构建器装饰器

**文件**: [git_cmd_obj_builder.go](pkg/commands/git_cmd_obj_builder.go)

**职责**: 对基础 CmdObjBuilder 进行 git 特定增强：
- 添加 `GIT_OPTIONAL_LOCKS=0` 环境变量
- 用 gitCmdObjRunner 装饰内部 runner（增加重试逻辑）

**设计模式**: 装饰器模式（Decorator Pattern）

---

## 二、命令对象层 (CmdObj)

**文件**: [cmd_obj.go](pkg/commands/oscommands/cmd_obj.go)

**核心结构体**: `CmdObj` - 命令对象，封装 `exec.Cmd` 并提供丰富的配置选项。

### 2.1 链式配置方法

| 方法 | 作用 | 默认值 |
|------|------|--------|
| `DontLog()` | 不在 UI 日志面板显示该命令 | false (显示) |
| `StreamOutput()` | 将输出流式写入命令日志面板 | false |
| `SuppressOutputUnlessError()` | 出错时才显示输出 | false |
| `UsePty()` | 使用 PTY 运行命令 | false |
| `IgnoreEmptyError()` | 无 stderr 的错误视为成功 | false |
| `SetStdin(input string)` | 设置标准输入 | - |
| `SetWd(wd string)` | 设置工作目录 | - |
| `AddEnvVars(vars ...string)` | 添加环境变量 | - |
| `WithMutex(mutex)` | 加互斥锁防止并发 | - |
| `PromptOnCredentialRequest(task)` | 遇到凭证请求时提示用户 | NONE |
| `FailOnCredentialRequest()` | 遇到凭证请求时直接失败 | NONE |

### 2.2 运行方法

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `Run() error` | 仅错误 | 最简单的运行方式 |
| `RunWithOutput() (string, error)` | 合并输出+错误 | 获取 stdout+stderr |
| `RunWithOutputs() (string, string, error)` | stdout, stderr, error | 分离输出 |
| `RunAndProcessLines(onLine func) error` | 逐行处理回调 | 流式逐行处理 |

### 2.3 DontLog() 的设计哲学

> 读取类命令（如 `git diff`, `git log`）调用 `DontLog()`，因为它们不改变 git 状态。
> 写入类命令（如 `git add`, `git commit`）默认显示，让用户知道发生了什么。
> 例外：后台周期性命令（如 `git fetch`）也不显示。

### 2.4 PromptOnCredentialRequest 的副作用

调用 `PromptOnCredentialRequest(task)` 时，除了设置凭证策略为 `PROMPT`，还会自动将 `usePty` 设为 `true` 并绑定一个 `gocui.Task`。这是因为凭证检测需要逐字节扫描 PTY 输出来匹配密码提示，而 task 用于在等待用户输入凭证时暂停/恢复 lazygit 的忙碌状态。

---

## 三、命令运行层 (Runners)

### 3.1 ICmdObjRunner 接口

**文件**: [cmd_obj_runner.go](pkg/commands/oscommands/cmd_obj_runner.go)

```go
type ICmdObjRunner interface {
    Run(cmdObj *CmdObj) error
    RunWithOutput(cmdObj *CmdObj) (string, error)
    RunWithOutputs(cmdObj *CmdObj) (string, string, error)
    RunAndProcessLines(cmdObj *CmdObj, onLine func(line string) (bool, error)) error
}
```

### 3.2 cmdObjRunner - 基础运行器

**核心逻辑**:

1. **互斥锁处理**: 如果命令设置了 mutex，先加锁
2. **凭证策略处理**: 如果需要凭证处理走 `runWithCredentialHandling`
3. **流式输出**: 如果 `ShouldStreamOutput()`，走 `runAndStream`
4. **普通执行**: 否则走 `RunWithOutputAux` / `RunWithOutputsAux`

**关键方法 `RunWithOutputAux` 流程**:
```
记录 Debug 日志
    ↓
如果 ShouldLog() → 调用 logCmdObj() 写入 UI 日志面板
    ↓
计时开始 → cmd.CombinedOutput() → 计时结束
    ↓
记录 Info 日志（含耗时）
    ↓
返回输出和错误
```

**流式输出 `runAndStreamAux` 流程**:
```
获取 cmdWriter（来自 guiIO.newCmdWriterFn）
    ↓
记录命令到日志面板
    ↓
启动命令（PTY 或普通管道）
    ↓
onRun 回调处理输出（不同场景有不同回调）
    ↓
等待命令结束
    ↓
如果出错且 SuppressOutputUnlessError → 将缓存的输出写入面板
```

### 3.3 gitCmdObjRunner - Git 命令运行器装饰器

**文件**: [git_cmd_obj_runner.go](pkg/commands/git_cmd_obj_runner.go)

**职责**: 为 git 命令添加**锁冲突重试**逻辑。

**重试触发条件**（输出中包含）:
- `.git/index.lock`
- `cannot lock ref`

**重试策略**:
- 最多重试 5 次
- 每次等待 50ms
- `RunAndProcessLines` 不实现重试（这类命令通常不需要锁）

### 3.4 两种命令处理器（cmdHandler）

Runner 在启动命令时会选择创建不同的 handler：

| 方法 | 平台 | stdoutPipe | stdinPipe | close | 用途 |
|------|------|-----------|-----------|-------|------|
| `getCmdHandlerPty` | 非 Windows | PTY 主设备 | PTY 主设备 | ptmx.Close | 凭证检测需要 PTY |
| `getCmdHandlerPty` | Windows | 回退到 NonPty | 同左 | 无操作 | Windows 无 PTY 支持 |
| `getCmdHandlerNonPty` | 通用 | io.Pipe 读取端 | Buffer | 无操作 | 流式输出 / 普通命令 |

**文件**: [cmd_obj_runner_default.go](pkg/commands/oscommands/cmd_obj_runner_default.go) (非 Windows)
**文件**: [cmd_obj_runner_windows.go](pkg/commands/oscommands/cmd_obj_runner_windows.go) (Windows)

---

## 四、GUI IO 桥接层 (guiIO)

**文件**: [gui_io.go](pkg/commands/oscommands/gui_io.go)

**设计思想**: 命令层不直接依赖 GUI，而是通过注入回调函数实现双向通信。

### 4.1 guiIO 结构体

```go
type guiIO struct {
    log                     *logrus.Entry
    logCommandFn            func(str string, isCommandLineCommand bool)
    newCmdWriterFn          func() io.Writer
    promptForCredentialFn   func(credential CredentialType) <-chan string
}
```

### 4.2 三个核心回调

| 回调 | 调用方 | 提供方 | 作用 |
|------|--------|--------|------|
| `logCommandFn` | cmdObjRunner.logCmdObj | Gui.LogCommand | 在 Extras 面板显示命令 |
| `newCmdWriterFn` | cmdObjRunner (流式输出时) | Gui.getCmdWriter | 获取输出写入器（带 "Git output:" 前缀） |
| `promptForCredentialFn` | cmdObjRunner (凭证处理时) | CredentialsHelper.PromptUserForCredential | 请求用户输入凭证 |

### 4.3 初始化位置

**文件**: [gui.go](pkg/gui/gui.go#L776-L781)

```go
guiIO := oscommands.NewGuiIO(
    cmn.Log,
    gui.LogCommand,        // ← 命令日志回调
    gui.getCmdWriter,      // ← 输出写入器回调
    credentialsHelper.PromptUserForCredential,  // ← 凭证提示回调
)
```

---

## 五、结果展示层

### 5.1 命令日志面板 (Extras Panel)

**文件**: [command_log_panel.go](pkg/gui/command_log_panel.go)

**两个核心概念**:
- **Action (动作)**: 人类可读的描述，如 "Stage File"，黄色显示
- **Command (命令)**: 实际执行的命令，如 `git add -- 'filename'`，默认颜色（或紫色如果不是命令行命令）

**方法**:
- `LogAction(action string)` - 记录动作（一级标题）
- `LogCommand(cmdStr string, commandLine bool)` - 记录命令（缩进显示）

**显示效果**:
```
Stage File:
  git add -- 'filename'
Unstage File:
  git reset HEAD 'filename'
```

### 5.2 getCmdWriter - 命令输出写入器

**文件**: [extras_panel.go](pkg/gui/extras_panel.go#L96-L98)

返回一个 `prefixWriter`，首次写入时自动添加紫色 "Git output:" 前缀，后续写入直接透传到 Extras 面板视图。

### 5.3 主视图任务系统 (ViewBufferManager)

**文件**: [tasks.go](pkg/tasks/tasks.go)

对于需要在主视图展示输出的命令（如 `git show`、`git diff`），使用任务系统管理：

**特性**:
- 逐行读取，按需加载（初始加载一屏，滚动时再加载）
- 任务抢占：新任务开始时自动终止旧任务
- 节流机制：快速切换时延迟启动，避免 CPU 高峰
- gocui Task 集成：显示加载状态

---

## 六、完整调用链路示例

### 6.1 写入类命令（如 git branch -D）

```
用户按键触发
    ↓
BranchCommands.Delete(name)
    ├─ 构造: NewGitCmd("branch").Arg("-D", name).ToArgv()
    ├─ 构建: self.cmd.New(cmdArgs)  → 得到 CmdObj
    └─ 执行: .Run()
         └─ cmdObjRunner.Run()
              ├─ CredentialStrategy == NONE
              ├─ ShouldStreamOutput() == false
              ├─ 检查 ShouldLog() → true
              ├─ 调用 logCmdObj() → guiIO.logCommandFn()
              │    └─ Gui.LogCommand() → 写入 Extras 面板
              └─ cmd.CombinedOutput()
```

### 6.2 读取类命令（如 git log）

```
CommitLoader.GetCommits()
    ├─ getLogCmd(opts) 构造命令
    │    └─ NewGitCmd("log")...ToArgv()
    ├─ self.cmd.New(cmdArgs).DontLog()  → CmdObj
    └─ loadCommits(cmdObj, ...)
         └─ cmdObj.RunAndProcessLines(onLine)
              └─ cmdObjRunner.RunAndProcessLines()
                   ├─ 检查 ShouldLog() → false (因为 DontLog())
                   └─ 逐行扫描处理
```

### 6.3 推送命令与凭证提示（如 git push）

推送命令是 lazygit 中最复杂的命令流程之一，它**不使用** `StreamOutput()`，而是通过 `PromptOnCredentialRequest()` 走凭证处理路径。凭证处理路径内部会复用 `runAndStreamAux` 的流式输出能力，同时叠加了凭证检测逻辑。

**文件**: [sync.go](pkg/commands/git_commands/sync.go) (命令构造)
**文件**: [sync_controller.go](pkg/gui/controllers/sync_controller.go) (GUI 入口)
**文件**: [cmd_obj_runner.go](pkg/commands/oscommands/cmd_obj_runner.go#L314-L444) (凭证运行时)
**文件**: [credentials_helper.go](pkg/gui/controllers/helpers/credentials_helper.go) (凭证 UI)

#### 第一阶段：GUI 触发与命令构造

```
用户按 'P' 键
    ↓
SyncController.push(currentBranch)
    ↓
SyncController.pushAux(currentBranch, opts)
    ├─ c.WithInlineStatus(branch, ItemOperationPushing, ...)  ← 显示推送状态
    │    ├─ c.LogAction(c.Tr.Actions.Push)  ← 记录 "Push" 动作到 Extras 面板
    │    └─ c.Git().Sync.Push(task, opts)
    │         └─ SyncCommands.PushCmdObj(task, opts)
    │              ├─ NewGitCmd("push")...ToArgv()  ← 构造 git push 参数
    │              └─ self.cmd.New(cmdArgs).PromptOnCredentialRequest(task)
    │                   → 设置 credentialStrategy=PROMPT, usePty=true, task=task
    │         └─ cmdObj.Run()
```

#### 第二阶段：Runner 分派到凭证处理路径

```
cmdObjRunner.Run(cmdObj)
    ├─ CredentialStrategy == PROMPT  ← 命中凭证分支
    └─ runWithCredentialHandling(cmdObj)
         ├─ getCredentialPromptFn(cmdObj)
         │    └─ 返回 guiIO.promptForCredentialFn  (= CredentialsHelper.PromptUserForCredential)
         └─ runAndDetectCredentialRequest(cmdObj, promptFn)
```

#### 第三阶段：凭证检测 - 启动命令并监控输出

`runAndDetectCredentialRequest` 是核心，它调用 `runAndStreamAux` 并注入一个自定义的 `onRun` 回调：

```
runAndDetectCredentialRequest(cmdObj, promptFn)
    ├─ cmdObj.AddEnvVars("LANG=C", "LC_ALL=C", "LC_MESSAGES=C")
    │    ← 设置英语环境，确保密码提示可被正则匹配
    │
    └─ runAndStreamAux(cmdObj, onRun 回调):
         ├─ 获取 cmdWriter = guiIO.newCmdWriterFn()
         │    ← 每次调用返回新的 prefixWriter (带 "Git output:" 前缀)
         ├─ logCmdObj() → 记录 "git push ..." 到 Extras 面板
         ├─ cmd.Stderr = io.MultiWriter(cmdWriter, &stderr)
         │    ← stderr 同时写入 Extras 面板和内部缓冲
         ├─ getCmdHandlerPty(cmd)  ← 使用 PTY 启动命令
         │    ├─ 非 Windows: pty.Start(cmd) → 返回 PTY 主设备
         │    └─ Windows: 回退到 getCmdHandlerNonPty (无法检测凭证)
         ├─ handler.stdoutPipe = io.TeeReader(stdoutPipe, &stdout)
         │    ← stdout 同时写入内部缓冲（用于错误处理）
         │
         └─ onRun(handler, cmdWriter):
              ├─ tr = io.TeeReader(handler.stdoutPipe, cmdWriter)
              │    ← PTY 输出同时流向：①凭证检测器 ②Extras 面板
              └─ go processOutput(tr, stdinPipe, promptFn, close, cmdObj)
                   ← 后台协程逐字节扫描输出
```

**关键点**: 推送命令的输出通过 `TeeReader` 分成两路：
1. 一路写入 `cmdWriter` → Extras 面板实时显示 `git push` 的进度输出
2. 另一路进入 `processOutput` 进行凭证检测

#### 第四阶段：凭证检测 - 逐字节匹配

```
processOutput(reader, writer, promptFn, closeFunc, cmdObj)
    ├─ checkForCredentialRequest = getCheckForCredentialRequestFunc()
    │    ← 返回闭包，内部维护 ttyText 累积缓冲区
    │
    └─ 逐字节扫描 (bufio.ScanBytes):
         ├─ newBytes → ttyText.Write(newBytes)  ← 累积输出
         ├─ 遍历预编译正则，匹配 ttyText:
         │    ├─ "Password:" → Password
         │    ├─ ".+'s password:" → Password
         │    ├─ "Password\s*for\s*'.+':" → Password
         │    ├─ "Username\s*for\s*'.+':" → Username
         │    ├─ "Enter\s*passphrase\s*for\s*key\s*'.+':" → Passphrase
         │    ├─ "Enter\s*PIN\s*for\s*.+\s*key\s*.+:" → PIN
         │    ├─ "Enter\s*PIN\s*for\s*'.+':" → PIN
         │    └─ ".*2FA Token.*" → Token
         │
         ├─ 如果匹配成功 → ttyText.Reset()
         │    ├─ responseChan = promptFn(credentialType)
         │    │    └─ CredentialsHelper.PromptUserForCredential(type) → 返回 channel
         │    ├─ 如果 responseChan == nil → closeFunc() 杀死进程 (FAIL 策略)
         │    ├─ task.Pause() ← 暂停 gocui 忙碌状态计时
         │    ├─ toInput = <-responseChan ← 阻塞等待用户输入
         │    ├─ task.Continue() ← 恢复忙碌状态计时
         │    └─ writer.Write([]byte(toInput)) ← 将用户输入写入 PTY stdin
         │
         └─ 如果遇到换行 → 截断 ttyText 到最后一行（避免缓冲区无限增长）
```

#### 第五阶段：凭证 UI 弹窗

**文件**: [credentials_helper.go](pkg/gui/controllers/helpers/credentials_helper.go)

```
CredentialsHelper.PromptUserForCredential(passOrUname)
    ├─ 创建 channel ch
    └─ c.OnUIThread(func() error):  ← 切到 UI 线程
         ├─ getTitleAndMask(passOrUname):
         │    ├─ Username → "Username:", mask=false (明文)
         │    ├─ Password → "Password:", mask=true (密文)
         │    ├─ Passphrase → "Passphrase:", mask=true
         │    ├─ PIN → "PIN:", mask=true
         │    └─ Token → "Token:", mask=true
         └─ c.Prompt(PromptOpts):
              ├─ Title: "Password:" 等
              ├─ Mask: true/false
              ├─ HandleConfirm: ch <- input + "\n"  ← 用户确认后写入 channel
              └─ HandleClose: ch <- "\n"  ← 用户关闭时写入空行（使命令失败）
    ← 返回 ch（在 UI 弹窗创建后立即可用）
```

**时序关系**:
1. `processOutput` 协程检测到凭证请求 → 调用 `promptFn` → `PromptUserForCredential` 在 UI 线程创建弹窗并返回 channel
2. `processOutput` 协程从 channel 阻塞读取 → 等待用户输入
3. 用户在弹窗中输入 → `HandleConfirm` 将输入写入 channel → `processOutput` 收到输入 → 写入 PTY stdin
4. `git push` 收到密码继续执行 → 后续输出继续流式显示到 Extras 面板

#### 第六阶段：命令结束与错误处理

```
runAndStreamAux 继续执行:
    ├─ cmd.Wait()  ← 等待 git push 结束
    ├─ 记录耗时日志
    └─ 如果出错:
         ├─ SuppressOutputUnlessError → 将缓存输出写入面板 (push 不使用此选项)
         ├─ 检查 stderr 缓冲 → 有内容则返回 error
         ├─ 检查 IgnoreEmptyError → push 不使用此选项
         └─ 检查 stdout 缓冲 → 有内容则返回 error
```

回到 SyncController:
```
SyncController.pushAux:
    ├─ 如果 err 包含 "Updates were rejected" → 弹窗确认是否 force push
    └─ 否则 → c.Refresh(ASYNC) 刷新界面
```

### 6.4 后台 Fetch 命令（无凭证提示）

**文件**: [sync.go](pkg/commands/git_commands/sync.go#L77-L84)

后台 fetch 是推送的对照场景，展示了 `FailOnCredentialRequest` + `DontLog` + `SuppressOutputUnlessError` 的组合使用：

```
SyncCommands.FetchBackgroundCmdObj()
    ├─ self.cmd.New(cmdArgs)
    ├─ .DontLog()                      ← 不在面板记录命令
    ├─ .FailOnCredentialRequest()      ← 遇到凭证请求直接终止
    └─ .SuppressOutputUnlessError()    ← 成功时不显示输出
```

运行时：
```
cmdObjRunner.Run(cmdObj)
    ├─ CredentialStrategy == FAIL
    └─ runWithCredentialHandling(cmdObj)
         ├─ getCredentialPromptFn → failPromptFn (返回 nil channel)
         └─ runAndDetectCredentialRequest(cmdObj, failPromptFn)
              ├─ runAndStreamAux:
              │    ├─ cmdWriter = &combinedOutput (缓存而非面板)
              │    └─ ShouldSuppressOutputUnlessError → true → 输出不写入面板
              └─ processOutput:
                   └─ 检测到凭证请求 → responseChan == nil → closeFunc() 杀死进程
```

### 6.5 流式输出命令（如 git commit 通过 GpgHelper）

**文件**: [gpg_helper.go](pkg/gui/controllers/helpers/gpg_helper.go)

与推送不同，`git commit` 通过 `GpgHelper.runAndStream` 使用 `StreamOutput()` 模式：

```
GpgHelper.runAndStream(cmdObj, waitingStatus, onSuccess, refreshScope)
    └─ c.WithWaitingStatus(waitingStatus, func(task) {
         cmdObj.StreamOutput().Run()
       })
         └─ cmdObjRunner.Run(cmdObj)
              ├─ CredentialStrategy == NONE
              ├─ ShouldStreamOutput() == true
              └─ runAndStream(cmdObj)
                   └─ runAndStreamAux(cmdObj, onRun):
                        ├─ cmdWriter = guiIO.newCmdWriterFn()  ← prefixWriter
                        ├─ logCmdObj() → 记录命令
                        ├─ getCmdHandlerNonPty(cmd)  ← 不需要 PTY
                        └─ onRun: go io.Copy(cmdWriter, handler.stdoutPipe)
                             ← stdout 全部流式写入 Extras 面板
```

### 6.6 子进程挂起模式（如编辑文件）

**文件**: [gui.go](pkg/gui/gui.go#L986-L1062)

```
runSubprocessWithSuspense(cmdObj)
    ├─ gui.suspend() → 挂起 TUI
    ├─ runSubprocess(cmdObj)
    │    ├─ Gui.LogCommand() → 记录命令
    │    ├─ 连接 stdin/stdout/stderr 到终端
    │    ├─ cmd.Run()
    │    └─ 提示按回车返回
    └─ gui.resume() → 恢复 TUI
```

---

## 七、关键设计模式总结

### 7.1 装饰器模式 (Decorator)

- `gitCmdObjBuilder` 装饰 `CmdObjBuilder`
- `gitCmdObjRunner` 装饰 `cmdObjRunner`
- 通过 `CloneWithNewRunner` 实现 runner 的替换

### 7.2 构建器模式 (Builder)

- `GitCommandBuilder` 流式构建命令参数
- `CmdObj` 链式配置运行选项

### 7.3 依赖注入 (Dependency Injection)

- `guiIO` 通过构造函数注入回调，解耦命令层与 GUI 层
- 每个命令结构体都显式声明依赖，便于单元测试

### 7.4 策略模式 (Strategy)

- `CredentialStrategy`: NONE / PROMPT / FAIL
- 不同的运行策略根据 CmdObj 的配置动态选择：
  - `NONE` → 普通执行或流式输出
  - `PROMPT` → `runWithCredentialHandling` → `runAndDetectCredentialRequest`
  - `FAIL` → 同上路径，但 `failPromptFn` 返回 nil channel 导致进程被杀

### 7.5 模板方法模式 (Template Method)

- `runAndStreamAux` 是模板方法，接受 `onRun` 回调
- `runAndStream` 传入简单的 `io.Copy` 回调（纯流式）
- `runAndDetectCredentialRequest` 传入 `TeeReader + processOutput` 回调（凭证检测 + 流式）

---

## 八、分层不清晰的可能原因

1. **装饰器链不直观**: `gitCmdObjBuilder → innerBuilder` 和 `gitCmdObjRunner → innerRunner` 的嵌套关系需要追踪代码才能理清

2. **Runner 嵌入 Builder**: CmdObj 内部持有 runner 引用，构建和运行的职责在对象层面合并了（虽然实现是分离的）

3. **多种运行入口并存**:
   - `CmdObj.Run()` - 最常用，由 Runner 内部分派到具体执行路径
   - `runSubprocessWithSuspense()` - 挂起 UI 的子进程，直接调用 exec.Cmd.Run()
   - `ViewBufferManager.NewCmdTask()` - 主视图渲染任务，自行管理进程生命周期

4. **日志有两套**:
   - logrus 日志（文件日志）- 用于调试，记录所有命令（含 DontLog 的）
   - GUI 命令日志（Extras 面板）- 给用户看，仅显示 ShouldLog() 为 true 的命令

5. **LogAction vs LogCommand**:
   - Action 由 Controller 层手动调用（如 `c.LogAction(c.Tr.Actions.Push)`）
   - Command 由 Runner 自动调用（在 `logCmdObj` 中通过 `guiIO.logCommandFn`）
   - 两者是父子关系，但代码中没有显式关联——时序靠调用顺序隐式保证

6. **推送命令的输出路径容易误解**:
   - 推送命令**不使用** `StreamOutput()`，而是通过 `PromptOnCredentialRequest()` 进入凭证处理路径
   - 凭证处理路径内部复用了 `runAndStreamAux`，通过 `TeeReader` 同时实现流式输出和凭证检测
   - 这意味着 "流式输出" 和 "凭证检测" 是叠加关系，不是二选一
