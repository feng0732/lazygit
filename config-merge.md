# 配置加载与合并优先级详解（代码级分析）

本文档从代码层面深入分析 lazygit 配置系统的**配置文件选择**、**默认值覆盖**和**运行时回退**的完整链路。

---

## 一、核心代码文件与职责

| 文件 | 职责 |
|------|------|
| [app_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go) | 配置入口：文件选择、加载、合并、迁移、重载 |
| [user_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/user_config.go) | 用户配置结构定义、通用默认值、按键绑定合并 |
| [user_config_validation.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/user_config_validation.go) | 配置验证逻辑 |
| [config_linux.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/config_linux.go) | Linux 平台 OS 默认配置 |
| [config_windows.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/config_windows.go) | Windows 平台 OS 默认配置 |
| [config_default_platform.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/config_default_platform.go) | macOS/其他平台 OS 默认配置 |
| [editor_presets.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go) | 编辑器预设动态解析 |
| [os.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go) | 运行时 OS 命令：文件打开、链接打开、剪贴板 |
| [file.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/git_commands/file.go) | 运行时编辑器命令调用 |
| [entry_point.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/app/entry_point.go) | 应用启动入口，调用 NewAppConfig |

---

## 二、配置文件选择逻辑（代码级）

### 2.1 应用启动入口

配置加载从应用启动入口开始：[entry_point.go#L139-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/app/entry_point.go#L139-L142)

```go
appConfig, err := config.NewAppConfig(
    "lazygit",
    buildInfo.Version,
    buildInfo.Commit,
    buildInfo.Date,
    buildInfo.BuildSource,
    cliArgs.Debug,
    tempDir,
)
```

### 2.2 NewAppConfig 中的配置文件选择

[app_config.go#L72-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L72-L126) 是配置文件选择的核心逻辑：

```go
func NewAppConfig(...) (*AppConfig, error) {
    // 步骤1：确定配置目录
    configDir, err := findOrCreateConfigDir()

    // 步骤2：通过环境变量 LG_CONFIG_FILE 判断使用哪种配置文件
    var configFiles []*ConfigFile
    customConfigFiles := os.Getenv("LG_CONFIG_FILE")
    if customConfigFiles != "" {
        // 分支A：使用环境变量指定的配置文件
        userConfigPaths := strings.Split(customConfigFiles, ",")
        configFiles = lo.Map(userConfigPaths, func(path string, _ int) *ConfigFile {
            return &ConfigFile{Path: path, Policy: ConfigFilePolicyErrorIfMissing}
        })
    } else {
        // 分支B：使用默认配置文件路径
        path := filepath.Join(configDir, ConfigFilename)
        configFile := &ConfigFile{Path: path, Policy: ConfigFilePolicyCreateIfMissing}
        configFiles = []*ConfigFile{configFile}
    }

    // 步骤3：加载配置
    userConfig, err := loadUserConfigWithDefaults(configFiles, false)
    // ...
}
```

**决策树：**

```
启动 NewAppConfig()
    │
    ├─→ 读取环境变量 LG_CONFIG_FILE
    │
    ├─→ LG_CONFIG_FILE 非空？
    │   ├─→ 是 → 按逗号分割路径，每个文件 Policy=ErrorIfMissing
    │   │         （文件不存在则报错）
    │   │
    │   └─→ 否 → 调用 findOrCreateConfigDir() 确定目录
    │             拼接 config.yml，Policy=CreateIfMissing
    │             （文件不存在则创建空文件）
    │
    └─→ 调用 loadUserConfigWithDefaults(configFiles)
```

### 2.3 配置目录查找：findOrCreateConfigDir

[app_config.go#L128-L137](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L128-L137)

```go
func ConfigDir() string {
    _, filePath := findConfigFile(ConfigFilename)
    return filepath.Dir(filePath)
}

func findOrCreateConfigDir() (string, error) {
    folder := ConfigDir()
    return folder, os.MkdirAll(folder, 0o755)
}
```

### 2.4 配置文件路径查找：findConfigFile

[app_config.go#L588-L608](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L588-L608)

```go
func findConfigFile(filename string) (exists bool, path string) {
    // 优先级1：CONFIG_DIR 环境变量
    if envConfigDir := os.Getenv("CONFIG_DIR"); envConfigDir != "" {
        return true, filepath.Join(envConfigDir, filename)
    }

    // 优先级2：旧版路径（向后兼容）
    // 搜索 XDG_CONFIG_HOME/jesseduffield/lazygit/ 和 XDG_CONFIG_DIRS
    legacyConfigPath, err := xdg.SearchConfigFile(
        filepath.Join("jesseduffield", "lazygit", filename))
    if err == nil {
        return true, legacyConfigPath
    }

    // 优先级3：新版路径
    // 搜索 XDG_CONFIG_HOME/lazygit/ 和 XDG_CONFIG_DIRS
    configFilepath, err := xdg.SearchConfigFile(
        filepath.Join("lazygit", filename))
    if err == nil {
        return true, configFilepath
    }

    // 优先级4：默认路径（不检查是否存在）
    return false, filepath.Join(xdg.ConfigHome, "lazygit", filename)
}
```

**配置文件路径查找优先级（从高到低）：**

| 优先级 | 查找路径 | 环境变量/说明 |
|--------|----------|---------------|
| 1 | `$CONFIG_DIR/config.yml` | `CONFIG_DIR` 环境变量指定的目录 |
| 2 | `$XDG_CONFIG_HOME/jesseduffield/lazygit/config.yml` | 旧版路径，向后兼容 |
| 3 | `$XDG_CONFIG_HOME/lazygit/config.yml` | 新版标准路径 |
| 4 | 同上（默认路径，不检查存在性） | 以上都找不到时返回此路径用于创建 |

### 2.5 环境变量汇总

| 环境变量 | 用途 | 代码位置 |
|----------|------|----------|
| `LG_CONFIG_FILE` | 自定义配置文件路径（逗号分隔多个），存在时跳过默认路径 | [app_config.go#L87-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L87-L93) |
| `CONFIG_DIR` | 指定配置目录，存在时跳过 XDG 路径查找 | [app_config.go#L591-L593](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L591-L593) |
| `LAZYGIT_LOG_PATH` | 指定日志文件路径 | [app_config.go#L732-L738](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L732-L738) |

### 2.6 ConfigFilePolicy 文件策略

[app_config.go#L56-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L56-L62)

```go
const (
    ConfigFilePolicyCreateIfMissing ConfigFilePolicy = iota  // 0：不存在则创建空文件
    ConfigFilePolicyErrorIfMissing                          // 1：不存在则报错返回
    ConfigFilePolicySkipIfMissing                          // 2：不存在则跳过
)
```

---

## 三、配置加载与合并（代码级）

### 3.1 loadUserConfigWithDefaults 入口

[app_config.go#L139-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L139-L141)

```go
func loadUserConfigWithDefaults(configFiles []*ConfigFile, isGuiInitialized bool) (*UserConfig, error) {
    return loadUserConfig(configFiles, GetDefaultConfigForPlatform(runtime.GOOS), isGuiInitialized)
}
```

**关键点**：先调用 `GetDefaultConfigForPlatform()` 获得通用默认值作为 `base`，再传入 `loadUserConfig`。

### 3.2 loadUserConfig 合并核心算法

[app_config.go#L143-L208](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L143-L208)

```go
func loadUserConfig(configFiles []*ConfigFile, base *UserConfig, isGuiInitialized bool) (*UserConfig, error) {
    // 按顺序遍历每个配置文件（后加载的优先级更高）
    for _, configFile := range configFiles {
        path := configFile.Path

        // 步骤1：处理文件不存在的情况
        statInfo, err := os.Stat(path)
        if err != nil {
            if os.IsNotExist(err) {
                switch configFile.Policy {
                case ConfigFilePolicyErrorIfMissing:
                    return nil, err                          // 报错
                case ConfigFilePolicySkipIfMissing:
                    continue                                  // 跳过
                case ConfigFilePolicyCreateIfMissing:
                    os.Create(path)                           // 创建空文件
                }
            }
        }

        // 步骤2：读取文件内容
        content, err := os.ReadFile(path)

        // 步骤3：配置迁移（旧格式 → 新格式）
        content, err = migrateUserConfig(path, content, isGuiInitialized)

        // 步骤4：保存当前 base 中的 CustomCommands
        existingCustomCommands := base.CustomCommands

        // 步骤5：YAML 反序列化 → 覆盖 base 中对应字段
        // 注意：yaml.Unmarshal 会覆盖已定义的字段，未定义的字段保持不变
        if err := yaml.Unmarshal(content, base); err != nil {
            return nil, fmt.Errorf(...)
        }

        // 步骤6：CustomCommands 特殊处理 —— 追加而非覆盖
        // 先保存旧的，Unmarshal 会用新的覆盖，然后把旧的追加到后面
        base.CustomCommands = append(base.CustomCommands, existingCustomCommands...)

        // 步骤7：配置验证
        if err := base.Validate(); err != nil {
            return nil, fmt.Errorf(...)
        }
    }

    // 步骤8：合并遗留的 Alt 按键绑定
    base.Keybinding.MergeLegacyAltKeybindings()

    return base, nil
}
```

**合并算法图解：**

```
base (通用默认值)
   │
   ├─→ configFile_1.yml
   │     ├─→ yaml.Unmarshal(content_1, base)
   │     │     覆盖 base 中 content_1 定义的字段
   │     │
   │     └─→ CustomCommands 追加：base.CustomCommands + existingCustomCommands
   │
   ├─→ configFile_2.yml
   │     ├─→ yaml.Unmarshal(content_2, base)
   │     │     覆盖 base 中 content_2 定义的字段
   │     │
   │     └─→ CustomCommands 追加：...
   │
   ...  (更多配置文件)
   │
   └─→ MergeLegacyAltKeybindings()
         合并遗留按键绑定
```

### 3.3 通用默认值：GetDefaultConfigForPlatform

[user_config.go#L827-L949](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/user_config.go#L827-L949)

```go
func GetDefaultConfigForPlatform(platform string) *UserConfig {
    return &UserConfig{
        Gui: GuiConfig{
            ScrollHeight:             2,
            ScrollPastBottom:         true,
            // ... GUI 默认值
        },
        Git: GitConfig{
            AutoFetch:   true,
            AutoRefresh: true,
            // ... Git 默认值
        },
        Keybinding: KeybindingConfig{
            Universal: KeybindingUniversalConfig{
                Quit: Keybinding{"q"},
                // ... 按键绑定默认值
            },
            // ...
        },
        // ... 其他配置
        OS: OSConfig{},  // ⚠️ 重要：OS 字段初始化为空结构体
                         // 平台相关的 OS 默认值在运行时动态回退
    }
}
```

### 3.4 CustomCommands 追加规则详解

[app_config.go#L193-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L193-L199)

```go
existingCustomCommands := base.CustomCommands  // 保存之前的
yaml.Unmarshal(content, base)                  // 新配置覆盖 base（包括 CustomCommands）
base.CustomCommands = append(base.CustomCommands, existingCustomCommands...)
```

**举例：**
- 全局配置定义了命令 A、B
- 仓库配置定义了命令 C、D
- 结果顺序：C, D, A, B（仓库配置在前，全局配置在后）

**设计意图**：后加载的配置文件（优先级更高）的命令排在前面。

---

## 四、运行时回退（空值回退）完整链路

> **核心概念**：配置系统有两种优先级机制——**加载时覆盖**和**运行时回退**。OS 相关配置（打开文件、编辑器、剪贴板）是在运行时根据空值动态回退的，不是在加载时合并的。

### 4.1 回退机制总览

| 配置项 | 用户配置字段 | 回退触发条件 | 回退逻辑位置 |
|--------|-------------|-------------|-------------|
| 打开文件 | `os.open` | 字段为空字符串 | [os.go#L83-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L83-L93) |
| 打开链接 | `os.openLink` | 字段为空字符串 | [os.go#L95-L106](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L95-L106) |
| 编辑文件 | `os.edit` / `os.editPreset` | 字段为空字符串 | [editor_presets.go#L8-L16](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go#L8-L16) |
| 剪贴板写入 | `os.copyToClipboardCmd` | 字段为空字符串 | [os.go#L269-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L269-L288) |
| 剪贴板读取 | `os.readFromClipboardCmd` | `copyToClipboardCmd` 为空 | [os.go#L290-L305](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L290-L305) |

### 4.2 打开文件/链接回退链路

#### 代码实现：[os.go#L83-L106](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L83-L106)

```go
func (c *OSCommand) OpenFile(filename string) error {
    // 步骤1：从用户配置获取
    commandTemplate := c.UserConfig().OS.Open
    // 步骤2：如果为空，运行时回退到平台默认值
    if commandTemplate == "" {
        commandTemplate = config.GetPlatformDefaultConfig().Open
    }
    // 步骤3：替换占位符并执行
    command := utils.ResolvePlaceholderString(commandTemplate, templateValues)
    return c.Cmd.NewShell(command, ...).Run()
}

func (c *OSCommand) OpenLink(link string) error {
    commandTemplate := c.UserConfig().OS.OpenLink
    if commandTemplate == "" {
        commandTemplate = config.GetPlatformDefaultConfig().OpenLink
    }
    // ...
}
```

#### 平台默认值实现

**Windows**：[config_windows.go#L4-L8](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/config_windows.go#L4-L8)
```go
func GetPlatformDefaultConfig() OSConfig {
    return OSConfig{
        Open:     `start "" {{filename}}`,
        OpenLink: `start "" {{link}}`,
    }
}
```

**Linux**：[config_linux.go#L21-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/config_linux.go#L21-L33)
```go
func GetPlatformDefaultConfig() OSConfig {
    if isWSL() && !isContainer() {
        return OSConfig{
            Open:     `powershell.exe start explorer.exe "$(wslpath -w {{filename}})" >/dev/null`,
            OpenLink: `powershell.exe start '{{link}}' >/dev/null`,
        }
    }
    return OSConfig{
        Open:     `xdg-open {{filename}} >/dev/null`,
        OpenLink: `xdg-open {{link}} >/dev/null`,
    }
}
```

**macOS/其他**：[config_default_platform.go#L6-L10](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/config_default_platform.go#L6-L10)
```go
func GetPlatformDefaultConfig() OSConfig {
    return OSConfig{
        Open:     "open -- {{filename}}",
        OpenLink: "open {{link}}",
    }
}
```

**打开文件回退优先级：**

```
优先级1：用户配置 os.open
    ↓ （为空时）
优先级2：GetPlatformDefaultConfig().Open
         ├─ Windows: start "" {{filename}}
         ├─ Linux(WSL): powershell.exe start explorer.exe ...
         ├─ Linux: xdg-open {{filename}} >/dev/null
         └─ macOS: open -- {{filename}}
```

### 4.3 编辑器回退链路

#### 调用入口：[file.go#L32-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/git_commands/file.go#L32-L77)

```go
func (self *FileCommands) GetEditCmdStr(filenames []string) (string, bool) {
    // 传入：shell类型、用户OS配置、默认编辑器猜测函数
    template, suspend := config.GetEditTemplate(
        self.os.Platform.Shell,
        &self.UserConfig().OS,
        self.guessDefaultEditor,
    )
    // ...
}
```

#### 默认编辑器自动检测：[file.go#L79-L100](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/git_commands/file.go#L79-L100)

```go
func (self *FileCommands) guessDefaultEditor() string {
    // 按优先级检测编辑器环境变量
    editor := self.config.GetCoreEditor()  // git config core.editor
    if editor == "" {
        editor = self.os.Getenv("GIT_EDITOR")
    }
    if editor == "" {
        editor = self.os.Getenv("VISUAL")
    }
    if editor == "" {
        editor = self.os.Getenv("EDITOR")
    }
    // 取第一个空格前的部分作为编辑器名称
    if editor != "" {
        editor = strings.Split(editor, " ")[0]
    }
    return editor
}
```

#### GetEditTemplate 回退逻辑：[editor_presets.go#L8-L16](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go#L8-L16)

```go
func GetEditTemplate(shell string, osConfig *OSConfig, guessDefaultEditor func() string) (string, bool) {
    preset := getPreset(shell, osConfig, guessDefaultEditor)
    template := osConfig.Edit              // 步骤1：用户配置 os.edit
    if template == "" {
        template = preset.editTemplate     // 步骤2：用户未配置，使用预设模板
    }
    return template, getEditInTerminal(osConfig, preset)
}
```

#### getPreset 预设选择：[editor_presets.go#L56-L181](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go#L56-L181)

```go
func getPreset(shell string, osConfig *OSConfig, guessDefaultEditor func() string) *editPreset {
    // 预设映射表：vim, nvim, vscode, sublime, emacs, nano, helix, etc.
    presets := map[string]*editPreset{...}

    // 步骤1：用户显式指定的 os.editPreset
    presetName := osConfig.EditPreset

    // 步骤2：用户未指定 → 自动检测默认编辑器
    if presetName == "" {
        defaultEditor := guessDefaultEditor()
        if presets[defaultEditor] != nil {
            presetName = defaultEditor
        } else if p := editorToPreset[defaultEditor]; p != "" {
            presetName = p
        }
    }

    // 步骤3：都没有 → 最终回退到 vim
    if presetName == "" || presets[presetName] == nil {
        presetName = "vim"
    }

    return presets[presetName]
}
```

#### SuspendOnEdit 回退：[editor_presets.go#L193-L198](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go#L193-L198)

```go
func getEditInTerminal(osConfig *OSConfig, preset *editPreset) bool {
    // 用户显式设置 → 使用用户值
    if osConfig.SuspendOnEdit != nil {
        return *osConfig.SuspendOnEdit
    }
    // 否则使用预设的 suspend() 函数
    return preset.suspend()
}
```

**编辑器回退完整优先级：**

```
编辑命令模板：
  优先级1：用户配置 os.edit
      ↓ （为空时）
  优先级2：os.editPreset 对应的预设模板
      ↓ （未设置时）
  优先级3：自动检测编辑器
      │    ├─ git config core.editor
      │    ├─ $GIT_EDITOR
      │    ├─ $VISUAL
      │    └─ $EDITOR
      ↓ （都没找到）
  优先级4：vim 预设模板（最终回退）

是否挂起终端：
  优先级1：用户配置 os.editInTerminal (SuspendOnEdit)
      ↓ （为 nil 时）
  优先级2：预设的 suspend() 函数返回值
```

### 4.4 剪贴板回退链路

#### 代码实现：[os.go#L269-L305](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L269-L305)

```go
func (c *OSCommand) CopyToClipboard(str string) error {
    // 用户配置了自定义命令 → 使用用户命令
    if c.UserConfig().OS.CopyToClipboardCmd != "" {
        cmdStr := utils.ResolvePlaceholderString(
            c.UserConfig().OS.CopyToClipboardCmd,
            map[string]string{"text": c.Cmd.Quote(str)},
        )
        return c.Cmd.NewShell(cmdStr, ...).Run()
    }
    // 否则 → 回退到第三方库 atotto/clipboard
    return clipboard.WriteAll(str)
}

func (c *OSCommand) PasteFromClipboard() (string, error) {
    var s string
    var err error
    // 注意：这里检查的是 CopyToClipboardCmd，不是 ReadFromClipboardCmd
    if c.UserConfig().OS.CopyToClipboardCmd != "" {
        cmdStr := c.UserConfig().OS.ReadFromClipboardCmd
        s, err = c.Cmd.NewShell(cmdStr, ...).RunWithOutput()
    } else {
        s, err = clipboard.ReadAll()  // 回退到库
    }
    // ...
}
```

**剪贴板回退优先级：**

```
写入剪贴板：
  优先级1：用户配置 os.copyToClipboardCmd
      ↓ （为空时）
  优先级2：github.com/atotto/clipboard 库

读取剪贴板：
  优先级1：os.copyToClipboardCmd 非空时，使用 os.readFromClipboardCmd
      ↓ （copyToClipboardCmd 为空时）
  优先级2：github.com/atotto/clipboard 库
```

---

## 五、仓库级配置加载

### 5.1 ReloadUserConfigForRepo

[app_config.go#L548-L558](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L548-L558)

```go
func (c *AppConfig) ReloadUserConfigForRepo(repoConfigFiles []*ConfigFile) error {
    // 拼接：全局配置文件 + 仓库级配置文件
    // 仓库级追加在后面 → 优先级更高
    configFiles := append(c.globalUserConfigFiles, repoConfigFiles...)
    userConfig, err := loadUserConfigWithDefaults(configFiles, true)

    c.userConfig = userConfig
    c.userConfigFiles = configFiles
    return nil
}
```

**优先级**：仓库级配置 > 全局配置（因为追加在后面，后加载覆盖先加载）

---

## 六、运行时配置重载

### 6.1 ReloadChangedUserConfigFiles

[app_config.go#L560-L582](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L560-L582)

```go
func (c *AppConfig) ReloadChangedUserConfigFiles() (error, bool) {
    // 检查每个配置文件是否变化（存在性或修改时间）
    fileHasChanged := func(f *ConfigFile) bool {
        info, err := os.Stat(f.Path)
        exists := err == nil
        return exists != f.exists || (exists && info.ModTime() != f.modDate)
    }

    if lo.NoneBy(c.userConfigFiles, fileHasChanged) {
        return nil, false  // 无变化
    }

    // 有变化 → 重新加载所有配置文件
    userConfig, err := loadUserConfigWithDefaults(c.userConfigFiles, true)
    c.userConfig = userConfig
    return nil, true
}
```

---

## 七、完整优先级总结

### 7.1 配置加载时优先级（从低到高）

```
优先级1 (最低)
    ↓
通用默认配置 (GetDefaultConfigForPlatform)
    │  - GUI、Git、Keybinding 等所有非 OS 字段
    │  - OS 字段初始化为空结构体 OSConfig{}
    ↓
优先级2
    ↓
全局用户配置文件 (~/.config/lazygit/config.yml)
    │  - yaml.Unmarshal 覆盖通用默认值
    ↓
优先级3
    ↓
环境变量 LG_CONFIG_FILE 指定的配置文件
    │  - 多个文件用逗号分隔，按顺序加载
    │  - Policy=ErrorIfMissing
    ↓
优先级4
    ↓
仓库级配置文件
    │  - 通过 ReloadUserConfigForRepo 追加加载
    │  - Policy=通常为 SkipIfMissing
    ↓
优先级5 (最高)
    ↓
运行时直接赋值修改 (测试中常用)
```

### 7.2 运行时回退优先级（从低到高，仅空值时触发）

**打开文件/链接：**
```
优先级1：用户配置 os.open / os.openLink
    ↓ （为空）
优先级2：GetPlatformDefaultConfig() (Windows/Linux/macOS)
```

**编辑器：**
```
优先级1：用户配置 os.edit / os.editAtLine / ...
    ↓ （为空）
优先级2：用户配置 os.editPreset 对应的预设模板
    ↓ （未设置）
优先级3：自动检测：git core.editor → $GIT_EDITOR → $VISUAL → $EDITOR
    ↓ （都未设置）
优先级4：vim 预设（最终回退）
```

**剪贴板：**
```
优先级1：用户配置 os.copyToClipboardCmd / os.readFromClipboardCmd
    ↓ （为空）
优先级2：atotto/clipboard 第三方库
```

### 7.3 两种优先级机制对比

| 特性 | 加载时覆盖 | 运行时回退 |
|------|-----------|-----------|
| 触发时机 | 应用启动 / 配置重载时 | 实际使用配置时 |
| 实现方式 | `yaml.Unmarshal()` 覆盖结构体字段 | 检查字段是否为空，为空则动态替换 |
| 是否持久 | 加载完成后固定（直到下次重载） | 每次使用时动态计算 |
| 影响范围 | 所有配置字段 | 仅 OS 相关字段（open/edit/clipboard） |
| 代码位置 | [loadUserConfig](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L143-L208) | [os.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go) / [editor_presets.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go) |

---

## 八、关键设计决策与原因

### 8.1 为什么 OS 配置运行时回退而非加载时合并？

**原因**：
1. **动态环境检测**：Linux 需要检测 WSL、容器环境，这些在加载时可能不准确或需要运行时上下文
2. **跨平台兼容**：同一份配置文件在不同平台运行，平台差异由运行时处理
3. **编辑器检测**：编辑器预设需要根据当前 shell 类型（bash/fish/nu）和系统环境变量动态选择模板
4. **第三方库依赖**：剪贴板使用第三方库，只有在用户未配置时才使用库

### 8.2 为什么 CustomCommands 追加而非覆盖？

**原因**：支持"全局通用命令 + 仓库特定命令"的使用场景，两者共存不丢失。

### 8.3 为什么使用 yaml.Unmarshal 进行覆盖？

**原因**：
- 简单高效：利用 yaml 库自动处理零值和部分配置
- 未定义的字段自动保持默认值
- 符合 YAML 用户的预期（部分定义，其余默认）

**缺点**：切片类型默认会被完全覆盖，所以 CustomCommands 需要特殊处理。

---

## 九、常见问题（代码级解答）

**Q: LG_CONFIG_FILE 和 CONFIG_DIR 有什么区别？**
A: `LG_CONFIG_FILE` 直接指定**配置文件**路径（可多个），存在时完全跳过默认路径查找；`CONFIG_DIR` 指定**配置目录**，在该目录下查找 config.yml。`LG_CONFIG_FILE` 优先级更高。

**Q: 为什么 GetDefaultConfigForPlatform() 中 OS 是空结构体？**
A: 这是设计有意为之。OS 配置（open/editor/clipboard）在运行时根据实际环境动态回退，加载时不设置默认值。

**Q: 配置文件加载顺序影响优先级吗？**
A: 影响。后面加载的配置文件优先级更高，会覆盖前面加载的同名配置（CustomCommands 除外，是追加）。

**Q: 如何强制让 lazygit 只使用我的命令，不回退？**
A: 在配置文件中显式设置对应字段（即使是空字符串也不会触发回退，但空字符串可能导致命令执行失败）。

**Q: PasteFromClipboard 为什么检查的是 CopyToClipboardCmd 而不是 ReadFromClipboardCmd？**
A: 这是一个实现细节。代码假设如果用户配置了自定义剪贴板写入命令，那么也需要自定义读取命令；否则就统一使用第三方库。

**Q: 多个配置文件中的 CustomCommands 顺序是怎样的？**
A: 后加载文件的命令在前，先加载文件的命令在后。例如：全局[A,B] + 仓库[C,D] → 结果顺序是 [C,D,A,B]。
