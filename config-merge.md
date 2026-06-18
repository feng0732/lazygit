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

[app_config.go#L72-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L72-L126) 是配置文件选择的核心逻辑。

> ⚠️ **关键发现：LG_CONFIG_FILE 与默认配置文件是完全互斥的（仅指 config.yml 加载）**
>
> 代码使用 `if-else` 二选一结构决定**用户配置文件**的来源：
> - 设置了 `LG_CONFIG_FILE` → 使用用户指定的文件，完全不加载默认 `config.yml`
> - 未设置 `LG_CONFIG_FILE` → 使用默认路径的 `config.yml`
> - 两者**不会**同时加载，也没有叠加关系
>
> ⚠️ **但配置目录查找不受影响**：无论是否设置 `LG_CONFIG_FILE`，
> `findOrCreateConfigDir` 和 `findConfigFile` 仍然会被调用，
> 用于确定配置目录、查找 state.yml、日志文件路径等。

```go
func NewAppConfig(...) (*AppConfig, error) {
    // 步骤1：确定配置目录（无论是否设置 LG_CONFIG_FILE 都会调用，
    //        用于确定 userConfigDir、state.yml 查找路径等）
    configDir, err := findOrCreateConfigDir()

    // 步骤2：二选一决定用户配置文件来源
    var configFiles []*ConfigFile
    customConfigFiles := os.Getenv("LG_CONFIG_FILE")
    if customConfigFiles != "" {
        // ═══════════════════════════════════════
        // 分支A：使用 LG_CONFIG_FILE 指定的配置文件
        // 仅跳过默认 config.yml 加载
        // 配置目录查找和 state.yml 查找仍然执行
        // ═══════════════════════════════════════
        userConfigPaths := strings.Split(customConfigFiles, ",")
        configFiles = lo.Map(userConfigPaths, func(path string, _ int) *ConfigFile {
            return &ConfigFile{Path: path, Policy: ConfigFilePolicyErrorIfMissing}
        })
    } else {
        // ═══════════════════════════════════════
        // 分支B：使用默认配置文件路径
        // 只有这个分支才会使用 findConfigFile 查找的路径
        // ═══════════════════════════════════════
        path := filepath.Join(configDir, ConfigFilename)
        configFile := &ConfigFile{Path: path, Policy: ConfigFilePolicyCreateIfMissing}
        configFiles = []*ConfigFile{configFile}
    }

    // 步骤3：加载配置
    userConfig, err := loadUserConfigWithDefaults(configFiles, false)
    // ...
}
```

**决策树（仅针对 config.yml 加载的两条互斥路径）：**

```
启动 NewAppConfig()
    │
    ├─→ 读取环境变量 LG_CONFIG_FILE
    │       （注意：配置目录查找在此之前已完成，不受此影响）
    │
    ├─ LG_CONFIG_FILE 非空？
    │   │
    │   ├─ 是 ════════════════════════════════
    │   │    config.yml 来源：用户指定（逗号分隔）
    │   │    文件策略：ErrorIfMissing（不存在则报错）
    │   │    数量：可多个
    │   │    与默认 config.yml：互斥，不叠加
    │   │    配置目录查找：✅ 仍然执行（已完成）
    │   │    state.yml 加载：✅ 仍然执行
    │   │    ══════════════════════════════════
    │   │
    │   └─ 否 ════════════════════════════════
    │        config.yml 来源：configDir + config.yml
    │                        （configDir 由 findConfigFile 确定）
    │        文件策略：CreateIfMissing（不存在则创建）
    │        数量：1 个
    │        ══════════════════════════════════
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

**配置文件路径查找优先级（从高到低，仅用于确定默认 config.yml 路径和配置目录）：**

| 优先级 | 查找路径 | 环境变量/说明 |
|--------|----------|---------------|
| 1 | `$CONFIG_DIR/config.yml` | `CONFIG_DIR` 环境变量指定的目录 |
| 2 | `$XDG_CONFIG_HOME/jesseduffield/lazygit/config.yml` | 旧版路径，向后兼容 |
| 3 | `$XDG_CONFIG_HOME/lazygit/config.yml` | 新版标准路径 |
| 4 | 同上（默认路径，不检查存在性） | 以上都找不到时返回此路径用于创建 |

> ⚠️ **重要澄清**：`findConfigFile` 的调用**不受** `LG_CONFIG_FILE` 控制。即使设置了 `LG_CONFIG_FILE`，`findConfigFile` 仍然会被调用至少两次（用于确定配置目录和查找 state.yml）。`LG_CONFIG_FILE` **仅**跳过默认 `config.yml` 的加载，不跳过配置目录查找。

### 2.5 环境变量汇总

| 环境变量 | 用途 | 影响范围 | 代码位置 |
|----------|------|---------|----------|
| `LG_CONFIG_FILE` | 自定义配置文件路径（逗号分隔多个）。设置后**仅跳过默认 config.yml 加载**，不跳过配置目录查找 | 仅用户配置文件加载 | [app_config.go#L87-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L87-L93) |
| `CONFIG_DIR` | 指定配置目录，存在时跳过 XDG 路径查找 | 配置目录查找（含 state.yml、日志等） | [app_config.go#L591-L593](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L591-L593) |
| `LAZYGIT_LOG_PATH` | 指定日志文件路径 | 日志文件路径 | [app_config.go#L732-L738](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L732-L738) |

### 2.6 ConfigFilePolicy 文件策略

[app_config.go#L56-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L56-L62)

```go
const (
    ConfigFilePolicyCreateIfMissing ConfigFilePolicy = iota  // 0：不存在则创建空文件
    ConfigFilePolicyErrorIfMissing                          // 1：不存在则报错返回
    ConfigFilePolicySkipIfMissing                          // 2：不存在则跳过
)
```

### 2.7 配置目录与配置文件的关系详解

> ⚠️ **关键发现：配置目录查找与 config.yml 加载是两条相对独立的链路**
>
> 即使设置了 `LG_CONFIG_FILE`（**仅跳过默认 config.yml 加载**），**配置目录仍然会被查找和创建**，用于 state.yml、日志、更新下载等。

#### 2.7.1 NewAppConfig 执行顺序

[app_config.go#L80-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L80-L126)

```go
func NewAppConfig(...) (*AppConfig, error) {
    // ═══════════════════════════════════════
    // 第1步：查找并创建配置目录（总是执行）
    // ═══════════════════════════════════════
    configDir, err := findOrCreateConfigDir()
    // 调用链：findOrCreateConfigDir → ConfigDir → findConfigFile
    // 无论 LG_CONFIG_FILE 是否设置，这一步都会执行

    // ═══════════════════════════════════════
    // 第2步：二选一选择配置文件（分支执行）
    // ═══════════════════════════════════════
    var configFiles []*ConfigFile
    customConfigFiles := os.Getenv("LG_CONFIG_FILE")
    if customConfigFiles != "" {
        // 分支A：使用 LG_CONFIG_FILE 指定的文件
        // 不使用 configDir 下的 config.yml
    } else {
        // 分支B：使用 configDir + "config.yml"
    }

    // ═══════════════════════════════════════
    // 第3步：加载应用状态（独立查找，总是执行）
    // ═══════════════════════════════════════
    appState, err := loadAppState()
    // 调用链：loadAppState → stateFilePath → findConfigFile → xdg.StateFile
    // 与 LG_CONFIG_FILE 无关，独立查找

    // ═══════════════════════════════════════
    // 第4步：保存 configDir 到 AppConfig
    // ═══════════════════════════════════════
    appConfig := &AppConfig{
        userConfigDir: configDir,  // 总是保存配置目录路径
        // ...
    }
}
```

**执行顺序图：**

```
NewAppConfig()
    │
    ├─ 第1步：findOrCreateConfigDir() ← 总是执行
    │       （与 LG_CONFIG_FILE 完全无关）
    │       调用链：findOrCreateConfigDir → ConfigDir → findConfigFile
    │
    ├─ 第2步：if LG_CONFIG_FILE? ← 仅影响 config.yml 加载，二选一
    │       ├─ 是 → 使用 LG_CONFIG_FILE 指定的文件
    │       │         （仅跳过默认 config.yml）
    │       └─ 否 → 使用 configDir/config.yml
    │
    ├─ 第3步：loadAppState() ← 总是执行
    │       （独立调用 findConfigFile，与 LG_CONFIG_FILE 无关）
    │       调用链：loadAppState → stateFilePath → findConfigFile
    │
    └─ 第4步：保存 userConfigDir ← 总是保存
```

#### 2.7.2 配置目录的独立用途

配置目录（`userConfigDir`）不仅仅是存放 `config.yml` 的地方，它还有多个独立用途：

| 用途 | 文件 | 查找方式 | 受 LG_CONFIG_FILE 影响？ | 代码位置 |
|------|------|---------|------------------------|----------|
| 用户配置 | `config.yml` | `if-else` 二选一 | ✅ 是（设置了就**仅跳过默认 config.yml 加载**） | [app_config.go#L87-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L87-L98) |
| 应用状态 | `state.yml` | `stateFilePath` 独立查找 | ❌ 否 | [app_config.go#L612-L622](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L612-L622) |
| 日志文件 | `development.log` | `stateFilePath` 独立查找 | ❌ 否 | [app_config.go#L732-L738](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L732-L738) |
| 软件更新下载目录 | - | `GetUserConfigDir()` | ❌ 否 | [updates.go#L249](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/updates/updates.go#L249) |
| 路径信息暴露 | - | `GetUserConfigDir()` 对外接口 | ❌ 否 | [app_config.go#L544-L546](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L544-L546) |

#### 2.7.3 stateFilePath 独立查找逻辑

应用状态文件和日志文件通过 `stateFilePath` 查找，有自己独立的查找策略：

[app_config.go#L612-L622](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/app_config.go#L612-L622)

```go
func stateFilePath(filename string) (string, error) {
    // 优先级1：在配置目录中查找（通过 findConfigFile）
    exists, legacyStateFile := findConfigFile(filename)
    if exists {
        return legacyStateFile, nil
    }

    // 优先级2：XDG_STATE_HOME 状态目录
    // 搜索 XDG_STATE_HOME/lazygit/ 和 XDG_STATE_DIRS
    return xdg.StateFile(filepath.Join("lazygit", filename))
}
```

**state.yml 查找优先级：**
1. 配置目录中的 `state.yml`（与 `config.yml` 同目录查找规则）
2. `XDG_STATE_HOME/lazygit/state.yml`（标准状态目录）

#### 2.7.4 为什么设置了 LG_CONFIG_FILE 还要查找配置目录？

**设计原因分析：**

1. **职责分离**：
   - `LG_CONFIG_FILE` 只影响**用户配置文件**的加载
   - 配置目录还承担**状态存储、日志、更新下载**等其他职责
   - 这些职责与配置文件相互独立

2. **向后兼容**：
   - 旧版本状态文件可能存放在配置目录中
   - `stateFilePath` 会先在配置目录查找，找到就用

3. **接口一致性**：
   - `GetUserConfigDir()` 是对外公开的接口
   - 即使使用自定义配置文件，也需要返回一个合理的配置目录

4. **目录创建时机**：
   - `findOrCreateConfigDir` 在函数最开始调用
   - 保证后续如果需要写入状态文件，目录已存在

#### 2.7.5 设置 LG_CONFIG_FILE 后的行为总结

| 行为 | 是否发生 | 说明 |
|------|---------|------|
| 默认 `config.yml` 被加载 | ❌ 否 | **仅跳过 config.yml**，使用 LG_CONFIG_FILE 指定的文件替代 |
| `findConfigFile` 被调用 | ✅ 是 | 至少调用 2 次：1次给 configDir，1次给 state.yml |
| 配置目录被查找和创建 | ✅ 是 | `findOrCreateConfigDir` 始终执行，不受影响 |
| `userConfigDir` 有值 | ✅ 是 | 始终有值，通过 `findConfigFile` 确定 |
| `state.yml` 正常加载 | ✅ 是 | 独立查找，不受 LG_CONFIG_FILE 影响 |
| 日志文件正常写入 | ✅ 是 | 通过 `stateFilePath` 独立确定路径 |
| 更新下载目录正常 | ✅ 是 | 使用 `GetUserConfigDir()` |

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

### 4.1 空字符串回退规则详解

> ⚠️ **重要发现：string 类型字段无法区分"未设置"和"显式设置为空"**

#### 两种字段类型的回退行为差异

| 字段类型 | 示例字段 | 判断方式 | 未设置时 | 显式设为空时 | 能否区分 |
|----------|---------|----------|---------|-------------|---------|
| `string` | `os.open`, `os.edit`, `os.copyToClipboardCmd` | `== ""` | 回退 | **也回退** | ❌ 不能 |
| `*bool` (指针) | `os.editInTerminal` (`SuspendOnEdit`) | `!= nil` | 回退 | 不回退 | ✅ 能 |

**代码证据**：[user_config.go#L652-L690](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/user_config.go#L652-L690)

```go
type OSConfig struct {
    // string 类型：无法区分"未设置"和"显式空字符串"
    Edit                string `yaml:"edit,omitempty"`
    EditAtLine          string `yaml:"editAtLine,omitempty"`
    EditAtLineAndWait   string `yaml:"editAtLineAndWait,omitempty"`
    OpenDirInEditor     string `yaml:"openDirInEditor,omitempty"`
    EditPreset          string `yaml:"editPreset,omitempty"`
    Open                string `yaml:"open,omitempty"`
    OpenLink            string `yaml:"openLink,omitempty"`
    CopyToClipboardCmd  string `yaml:"copyToClipboardCmd,omitempty"`
    ReadFromClipboardCmd string `yaml:"readFromClipboardCmd,omitempty"`

    // 指针类型：可以区分 nil（未设置）和 false（显式设置为假）
    SuspendOnEdit       *bool  `yaml:"editInTerminal,omitempty"`
}
```

**关于 `omitempty` 标签的说明**：
- `omitempty` 只影响 **YAML 序列化输出**（空值不输出）
- 不影响 **YAML 反序列化输入**（反序列化时，没定义的字段保持零值）
- 因此 string 类型字段在配置文件中未定义时，值为 `""`（空字符串）

#### 回退行为对照表

| 配置项 | 字段类型 | 用户配置 | 回退触发？ | 最终使用值 |
|--------|---------|---------|-----------|-----------|
| `os.open` | `string` | 未设置 | ✅ 触发 | 平台默认值 |
| `os.open` | `string` | `""`（显式空串） | ✅ 触发 | 平台默认值 |
| `os.open` | `string` | `"mycmd"` | ❌ 不触发 | `"mycmd"` |
| `os.editInTerminal` | `*bool` | 未设置（nil） | ✅ 触发 | 预设值 |
| `os.editInTerminal` | `*bool` | `false` | ❌ 不触发 | `false` |
| `os.editInTerminal` | `*bool` | `true` | ❌ 不触发 | `true` |

### 4.2 回退机制总览

| 配置项 | 用户配置字段 | 字段类型 | 回退触发条件 | 回退逻辑位置 |
|--------|-------------|---------|-------------|-------------|
| 打开文件 | `os.open` | `string` | `== ""`（空字符串） | [os.go#L83-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L83-L93) |
| 打开链接 | `os.openLink` | `string` | `== ""` | [os.go#L95-L106](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L95-L106) |
| 编辑文件 | `os.edit` | `string` | `== ""` | [editor_presets.go#L8-L16](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go#L8-L16) |
| 编辑器预设 | `os.editPreset` | `string` | `== ""` | [editor_presets.go#L56-L181](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go#L56-L181) |
| 挂起终端 | `os.editInTerminal` | `*bool` | `== nil`（空指针） | [editor_presets.go#L193-L198](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/config/editor_presets.go#L193-L198) |
| 剪贴板写入 | `os.copyToClipboardCmd` | `string` | `== ""` | [os.go#L269-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L269-L288) |
| 剪贴板读取 | `os.readFromClipboardCmd` | `string` | `copyToClipboardCmd == ""` | [os.go#L290-L305](file:///d:/fz/0601-2/solo-dogfeeding/code/39-lazygit/pkg/commands/oscommands/os.go#L290-L305) |

### 4.3 打开文件/链接回退链路

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

### 4.4 编辑器回退链路

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

### 4.5 剪贴板回退链路

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

### 7.1 配置加载时优先级

> ⚠️ **注意**：LG_CONFIG_FILE 与默认 `config.yml` 是**互斥二选一**的关系，不是叠加关系。
>
> ⚠️ **但配置目录查找始终执行**：无论是否设置 `LG_CONFIG_FILE`，配置目录都会被查找，用于 state.yml、日志、更新下载等。

#### 场景A：未设置 LG_CONFIG_FILE（默认情况）

```
优先级1 (最低)
    ↓
通用默认配置 (GetDefaultConfigForPlatform)
    │  - GUI、Git、Keybinding 等所有非 OS 字段
    │  - OS 字段初始化为空结构体 OSConfig{}
    ↓
优先级2
    ↓
默认配置文件 (~/.config/lazygit/config.yml)
    │  - yaml.Unmarshal 覆盖通用默认值
    ↓
优先级3
    ↓
仓库级配置文件 (ReloadUserConfigForRepo 追加)
    │  - 通过 ReloadUserConfigForRepo 追加加载
    ↓
优先级4 (最高)
    ↓
运行时直接赋值修改
```

> 配置目录查找：✅ 始终执行（通过 `findOrCreateConfigDir`）
> state.yml 加载：✅ 始终执行（通过 `loadAppState`）

#### 场景B：设置了 LG_CONFIG_FILE

```
优先级1 (最低)
    ↓
通用默认配置 (GetDefaultConfigForPlatform)
    │  - GUI、Git、Keybinding 等所有非 OS 字段
    │  - OS 字段初始化为空结构体 OSConfig{}
    ↓
优先级2
    ↓
LG_CONFIG_FILE 指定的配置文件（按逗号顺序依次加载）
    │  - 多个文件用逗号分隔，按顺序加载（后面的覆盖前面的）
    │  - Policy=ErrorIfMissing（不存在则报错）
    │  - ⚠️ 仅跳过默认 config.yml 的加载
    │  - ⚠️ 配置目录查找和 state.yml 加载仍然执行
    ↓
优先级3
    ↓
仓库级配置文件 (ReloadUserConfigForRepo 追加)
    ↓
优先级4 (最高)
    ↓
运行时直接赋值修改
```

> 配置目录查找：✅ 仍然执行（不受 LG_CONFIG_FILE 影响）
> state.yml 加载：✅ 仍然执行（不受 LG_CONFIG_FILE 影响）

#### 两种场景对比

| 特性 | 未设置 LG_CONFIG_FILE | 设置了 LG_CONFIG_FILE |
|------|---------------------|---------------------|
| 用户配置文件来源 | 默认路径 config.yml | 用户指定路径（可多个） |
| 默认 config.yml 是否加载 | ✅ 是 | ❌ 否（仅跳过 config.yml） |
| 配置目录是否查找 | ✅ 是 | ✅ 是（始终查找，不受影响） |
| state.yml 是否加载 | ✅ 是 | ✅ 是（始终加载，不受影响） |
| 文件不存在策略 | 创建空文件 | 报错退出 |
| 与仓库级配置关系 | 叠加（仓库级在后面） | 叠加（仓库级在后面） |

### 7.2 运行时回退优先级（仅空值时触发）

> ⚠️ **关键规则**：
> - **string 类型字段**：空字符串 `""` 就回退（无法区分"未设置"和"显式设为空"）
> - **指针类型字段**（如 `SuspendOnEdit`）：只有 `nil`（未设置）才回退，显式设为 `false` 不回退

**打开文件/链接（string 类型，空即回退）：**
```
优先级1：用户配置 os.open / os.openLink
    ↓ （值 == "" 时触发，无论是未设置还是显式空串）
优先级2：GetPlatformDefaultConfig() (Windows/Linux/macOS)
```

**编辑器命令模板（string 类型，空即回退）：**
```
优先级1：用户配置 os.edit / os.editAtLine / ...
    ↓ （值 == "" 时触发）
优先级2：用户配置 os.editPreset 对应的预设模板
    ↓ （值 == "" 时触发）
优先级3：自动检测：git core.editor → $GIT_EDITOR → $VISUAL → $EDITOR
    ↓ （都未设置时）
优先级4：vim 预设（最终回退）
```

**挂起终端（指针类型，nil 才回退）：**
```
优先级1：用户配置 os.editInTerminal (SuspendOnEdit *bool)
    ↓ （值 == nil 时触发，显式设为 false 不触发）
优先级2：预设的 suspend() 函数返回值
```

**剪贴板（string 类型，空即回退）：**
```
优先级1：用户配置 os.copyToClipboardCmd / os.readFromClipboardCmd
    ↓ （值 == "" 时触发）
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

### 8.4 为什么 SuspendOnEdit 用指针类型而其他用 string？

**设计意图**：
- `string` 类型：命令类配置空字符串没有意义，设置为空等同于未设置，都应该回退到默认值
- `*bool` 指针类型：`editInTerminal` 场景需要区分三种状态：
  - `nil`：未设置，使用预设值
  - `true`：用户显式要求挂起
  - `false`：用户显式要求不挂起
- 这是 Go 语言中"可选布尔值"的标准模式

代码注释也明确说明了这一点：
```go
// [dev] Pointer to bool so that we can distinguish unset (nil) from false.
SuspendOnEdit *bool `yaml:"editInTerminal,omitempty"`
```

---

## 九、常见问题（代码级解答）

**Q: LG_CONFIG_FILE 和 CONFIG_DIR 有什么区别？**
A: 两者影响的范围完全不同：
- `LG_CONFIG_FILE`：直接指定**用户配置文件**路径（可多个），设置后**仅跳过默认 config.yml 的加载**，但配置目录查找、state.yml 查找、日志文件路径等仍然正常执行
- `CONFIG_DIR`：指定**配置目录**，在该目录下查找 config.yml、state.yml 等，影响配置目录查找的整个链路

两者可以同时设置，但 `LG_CONFIG_FILE` 会覆盖默认 config.yml 的加载路径，而 `CONFIG_DIR` 会影响配置目录的定位。

**Q: 设置了 LG_CONFIG_FILE 后，默认配置文件还会加载吗？**
A: **不会加载默认 config.yml**。代码是 `if-else` 二选一结构：设置了 `LG_CONFIG_FILE` 就走分支A（用户指定文件），未设置才走分支B（默认路径的 config.yml）。两者完全互斥，不会叠加。
但请注意：**配置目录查找和 state.yml 加载仍然正常执行**，不受 `LG_CONFIG_FILE` 影响。

**Q: 为什么 GetDefaultConfigForPlatform() 中 OS 是空结构体？**
A: 这是设计有意为之。OS 配置（open/editor/clipboard）在运行时根据实际环境动态回退，加载时不设置默认值。这样可以在运行时检测 WSL、容器环境等。

**Q: 配置文件加载顺序影响优先级吗？**
A: 影响。后面加载的配置文件优先级更高，会覆盖前面加载的同名配置（CustomCommands 除外，是追加）。

**Q: 如何强制让 lazygit 只使用我的命令，不触发运行时回退？**
A: 分两种情况：
- **string 类型字段**（如 `os.open`、`os.edit`）：只要设置一个非空字符串即可。但注意：**显式设置为空字符串 `""` 仍然会触发回退**，因为代码用 `== ""` 判断，无法区分"未设置"和"显式空值"。
- **指针类型字段**（如 `os.editInTerminal`）：显式设置为 `true` 或 `false` 就不会回退，只有 `nil`（未设置）才会回退。

**Q: 为什么 string 类型的配置项无法区分"未设置"和"显式设为空"？**
A: 因为 Go 语言中 string 的零值就是 `""`，yaml.Unmarshal 时，如果配置文件中没有定义该字段，结构体字段保持零值 `""`；如果显式定义为 `""`，反序列化后也是 `""`。两者结果完全相同，代码无法区分。

**Q: SuspendOnEdit 为什么用 `*bool` 指针而不是 `bool`？**
A: 为了区分三种状态：`nil`（未设置，用预设值）、`true`（用户要求挂起）、`false`（用户要求不挂起）。这是 Go 中"可选布尔值"的标准设计模式。代码注释也明确说明了这一点。

**Q: PasteFromClipboard 为什么检查的是 CopyToClipboardCmd 而不是 ReadFromClipboardCmd？**
A: 这是一个实现细节。代码假设如果用户配置了自定义剪贴板写入命令，那么也需要自定义读取命令；否则就统一使用第三方库。

**Q: 多个配置文件中的 CustomCommands 顺序是怎样的？**
A: 后加载文件的命令在前，先加载文件的命令在后。例如：全局[A,B] + 仓库[C,D] → 结果顺序是 [C,D,A,B]。

**Q: omitempty 标签会影响回退判断吗？**
A: **不影响**。`omitempty` 只影响 YAML **序列化输出**（空值不输出到文件），不影响 **反序列化输入**（读取配置文件时，没定义的字段保持零值）。回退判断是在运行时通过 `== ""` 或 `== nil` 做的，和 omitempty 无关。

**Q: 设置了 LG_CONFIG_FILE 后，配置目录还会被查找吗？**
A: **会的**。这是一个容易误解的点。`LG_CONFIG_FILE` 只影响**用户配置文件**的加载（跳过默认 config.yml），但配置目录仍然会被查找和创建，因为：
- `findOrCreateConfigDir()` 在 `if-else` 之前就被调用了
- 配置目录还用于存放 `state.yml`（应用状态）、日志文件、更新下载等
- 这些功能与 `LG_CONFIG_FILE` 是相互独立的

**Q: state.yml 的路径和 config.yml 的路径一定相同吗？**
A: **不一定**。`config.yml` 只有默认路径分支才会从配置目录加载；而 `state.yml` 通过 `stateFilePath` 独立查找，它会先在配置目录找，找不到就用 `XDG_STATE_HOME/lazygit/`。设置了 `LG_CONFIG_FILE` 后，config.yml 用用户指定的路径，但 state.yml 仍然用原来的查找逻辑。

**Q: NewAppConfig 中 findConfigFile 会被调用几次？**
A: 至少 **2 次**：
1. `findOrCreateConfigDir()` → `ConfigDir()` → `findConfigFile("config.yml")` — 确定配置目录
2. `loadAppState()` → `stateFilePath()` → `findConfigFile("state.yml")` — 查找状态文件

另外还有 `LogPath()` 也会调用 `stateFilePath()`，但那是在日志初始化时，不在 `NewAppConfig` 流程内。

**Q: GetUserConfigDir() 返回的路径和 LG_CONFIG_FILE 有关系吗？**
A: **没有关系**。`GetUserConfigDir()` 返回的是通过 `findConfigFile` 查找到的配置目录，与 `LG_CONFIG_FILE` 指定的文件路径完全无关。即使你设置了 `LG_CONFIG_FILE`，`GetUserConfigDir()` 仍然返回默认的配置目录路径。
