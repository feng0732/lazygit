# Lazygit 主题与样式渲染链路分析

## 整体架构概览

Lazygit 的主题与样式系统分为三大阶段，形成一条清晰的链路：

```
用户配置文件 (config.yml)
    ↓ YAML 反序列化
ThemeConfig 结构体
    ↓ UpdateTheme() / GetTextStyle() / GetGocuiStyle()
双轨样式对象 (TextStyle + gocui.Attribute)
    ↓ Sprint() / 视图属性赋值 / tcell 渲染
终端屏幕输出
```

核心设计思想：**双轨制**。一套配置同时驱动两条渲染路径：
- **TextStyle 路径**：用于内容文本（commit 列表、文件名等），由 `gookit/color` 库生成 ANSI 转义序列
- **gocui.Attribute 路径**：用于 UI 框架层（边框颜色、选中行背景等），由 `tcell` 库直接控制终端单元格

---

## 第一阶段：配置读取

### 1.1 配置文件定位与加载

入口在 [app_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/config/app_config.go#L72-L101) 的 `NewAppConfig()`：

```
NewAppConfig()
  → findOrCreateConfigDir()              // 定位 XDG 配置目录 (~/.config/lazygit/)
  → loadUserConfigWithDefaults()         // 加载配置
      → loadUserConfig()
          → GetDefaultConfigForPlatform() // 先生成默认值
          → yaml.Unmarshal(content, base) // 用 YAML 内容覆盖默认值
          → base.Validate()               // 校验
```

关键点：**默认值先填充，再被用户配置覆盖**。这意味着用户只需在 `config.yml` 中声明需要修改的字段，未声明的字段保持默认。

### 1.2 ThemeConfig 结构体

定义在 [user_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/config/user_config.go#L215-L241)：

```go
type ThemeConfig struct {
    ActiveBorderColor               []string  // 激活窗口边框
    InactiveBorderColor             []string  // 非激活窗口边框
    SearchingActiveBorderColor      []string  // 搜索中窗口边框
    OptionsTextColor                []string  // 底部快捷键文字
    SelectedLineBgColor             []string  // 选中行背景
    InactiveViewSelectedLineBgColor []string  // 非焦点选中行背景
    CherryPickedCommitFgColor       []string  // cherry-pick 提交前景
    CherryPickedCommitBgColor       []string  // cherry-pick 提交背景
    MarkedBaseCommitFgColor         []string  // rebase 基准提交前景
    MarkedBaseCommitBgColor         []string  // rebase 基准提交背景
    UnstagedChangesColor            []string  // 未暂存变更颜色
    DefaultFgColor                  []string  // 默认文字颜色
}
```

**每个字段都是 `[]string`**，这是 Lazygit 主题配置的核心设计：一个颜色字段可以由多个 token 组合，例如：

```yaml
activeBorderColor:
  - green    # 颜色
  - bold     # 装饰
```

`[]string` 中每个元素可以是：
- **命名颜色**：`default`、`black`、`red`、`green`、`yellow`、`blue`、`magenta`、`cyan`、`white`
- **装饰属性**：`bold`、`reverse`、`underline`、`strikethrough`
- **十六进制 RGB**：`#ff0000`、`#f00`

### 1.3 默认主题值

在 [user_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/config/user_config.go#L849-L862) 中定义：

```go
Theme: ThemeConfig{
    ActiveBorderColor:               []string{"green", "bold"},
    SearchingActiveBorderColor:      []string{"cyan", "bold"},
    InactiveBorderColor:             []string{"default"},
    OptionsTextColor:                []string{"blue"},
    SelectedLineBgColor:             []string{"blue"},
    InactiveViewSelectedLineBgColor: []string{"bold"},
    CherryPickedCommitBgColor:       []string{"cyan"},
    CherryPickedCommitFgColor:       []string{"blue"},
    MarkedBaseCommitBgColor:         []string{"yellow"},
    MarkedBaseCommitFgColor:         []string{"blue"},
    UnstagedChangesColor:            []string{"red"},
    DefaultFgColor:                  []string{"default"},
},
```

### 1.4 配置热重载

配置可以在运行时修改并自动生效。触发链路：

```
终端获得焦点 (FocusHandler)
  → Config.ReloadChangedUserConfigFiles()   // 检测文件修改时间
  → onUserConfigLoaded()                     // 重新应用配置
      → setColorScheme()                     // 重新设置颜色
      → configureViewProperties()            // 重新配置视图属性
      → authors.SetCustomAuthors()           // 重新设置作者颜色
      → presentation.SetCustomBranches()     // 重新设置分支颜色
```

参考 [gui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/gui.go#L351-L375) 中的 FocusHandler 逻辑。

---

## 第二阶段：样式合成（双轨转换）

### 2.1 转换入口：UpdateTheme()

定义在 [theme.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/theme/theme.go#L51-L76)：

```go
func UpdateTheme(themeConfig config.ThemeConfig) {
    // 轨道一：gocui.Attribute（UI 框架层样式）
    ActiveBorderColor = GetGocuiStyle(themeConfig.ActiveBorderColor)
    InactiveBorderColor = GetGocuiStyle(themeConfig.InactiveBorderColor)
    SearchingActiveBorderColor = GetGocuiStyle(themeConfig.SearchingActiveBorderColor)
    GocuiSelectedLineBgColor = GetGocuiStyle(themeConfig.SelectedLineBgColor)
    GocuiInactiveViewSelectedLineBgColor = GetGocuiStyle(themeConfig.InactiveViewSelectedLineBgColor)
    OptionsColor = GetGocuiStyle(themeConfig.OptionsTextColor)
    GocuiDefaultTextColor = GetGocuiStyle(themeConfig.DefaultFgColor)

    // 轨道二：TextStyle（内容文本样式）
    SelectedLineBgColor = GetTextStyle(themeConfig.SelectedLineBgColor, true)
    InactiveViewSelectedLineBgColor = GetTextStyle(themeConfig.InactiveViewSelectedLineBgColor, true)
    DefaultTextColor = GetTextStyle(themeConfig.DefaultFgColor, false)
    OptionsFgColor = GetTextStyle(themeConfig.OptionsTextColor, false)
    UnstagedChangesColor = GetTextStyle(themeConfig.UnstagedChangesColor, false)

    // 复合样式：cherry-picked 和 marked base 需要前景 + 背景合并
    CherryPickedCommitTextStyle = bgStyle.MergeStyle(fgStyle)
    MarkedBaseCommitTextStyle = bgStyle.MergeStyle(fgStyle)
}
```

**同一个 ThemeConfig 字段可能同时生成两种样式对象**。例如 `SelectedLineBgColor` 同时产出：
- `GocuiSelectedLineBgColor`（gocui.Attribute）→ 用于 View 的 `SelBgColor` 属性
- `SelectedLineBgColor`（TextStyle）→ 目前在代码中声明但实际选中行背景由 gocui 层处理

### 2.2 轨道一：gocui.Attribute 转换

**GetGocuiStyle()** → [gocui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/theme/gocui.go#L39-L45)

```
GetGocuiStyle(keys []string)
  → 遍历每个 key
  → GetGocuiAttribute(key)
      → 若为十六进制值: color.HEX(key).Values() → gocui.NewRGBColor(r, g, b)
      → 若为命名颜色/装饰: gocuiColorMap[key]
      → 默认: gocui.ColorWhite
  → 各 Attribute 按位 OR (|=) 合并
```

gocui.Attribute 的内部结构（定义在 [attribute.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gocui/attribute.go#L10-L64)）：

```
8 字节 uint64:
  - 低 5 字节 (AttrColorBits): 存储 tcell 颜色值
  - 高 3 字节 (AttrStyleBits): 存储文本效果（bold、underline 等）
    - bit 40: AttrBold
    - bit 41: AttrBlink
    - bit 42: AttrReverse
    - bit 43: AttrUnderline
    - bit 44: AttrDim
    - bit 45: AttrItalic
    - bit 46: AttrStrikeThrough
```

颜色映射表（[gocui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/theme/gocui.go#L9-L22)）：

| key | gocui.Attribute |
|---|---|
| default | gocui.ColorDefault |
| black | gocui.ColorBlack |
| red | gocui.ColorRed |
| green | gocui.ColorGreen |
| yellow | gocui.ColorYellow |
| blue | gocui.ColorBlue |
| magenta | gocui.ColorMagenta |
| cyan | gocui.ColorCyan |
| white | gocui.ColorWhite |
| bold | gocui.AttrBold |
| reverse | gocui.AttrReverse |
| underline | gocui.AttrUnderline |

**位运算合并示例**：`["green", "bold"]` → `gocui.ColorGreen | gocui.AttrBold`

### 2.3 轨道二：TextStyle 转换

**GetTextStyle()** → [style.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/theme/style.go#L9-L44)

```
GetTextStyle(keys []string, background bool)
  → 创建空 TextStyle: style.New()
  → 遍历每个 key:
      → "bold"          → s.SetBold()
      → "reverse"       → s.SetReverse()
      → "underline"     → s.SetUnderline()
      → "strikethrough" → s.SetStrikethrough()
      → 命名颜色        → ColorMap[key] → 按背景/前景取 .Background/.Foreground → s.MergeStyle()
      → 十六进制值      → color.HEX(key, background) → s.SetBg()/s.SetFg()
  → 返回合成后的 TextStyle
```

**`background` 参数的关键作用**：同一个 key（如 `"blue"`）在 `background=true` 时取背景色变体，在 `background=false` 时取前景色变体。这来自 ColorMap 的双向映射（[basic_styles.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/basic_styles.go#L38-L51)）：

```go
ColorMap = map[string]struct{ Foreground TextStyle; Background TextStyle }{
    "blue": {FgBlue, BgBlue},
    // ...
}
```

### 2.4 TextStyle 内部结构

定义在 [text_style.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/text_style.go#L28-L36)：

```go
type TextStyle struct {
    fg         *Color       // 前景色（可选）
    bg         *Color       // 背景色（可选）
    decoration Decoration   // 装饰属性
    Style      Sprinter     // 预推导的渲染器（缓存优化）
}
```

**关键设计：值对象 + 预推导缓存**。每次 `SetBold()`、`SetFg()` 等操作都会：
1. 创建新的 TextStyle 副本（值语义，不修改原对象）
2. 在新副本上应用修改
3. 立即调用 `deriveStyle()` 重新推导底层渲染器
4. 将渲染器缓存在 `Style` 字段中

这样 `Sprint()`/`Sprintf()` 调用时无需重复计算，直接使用缓存的 `Style`。

### 2.5 deriveStyle() 分支逻辑

```
deriveStyle()
  → fg==nil && bg==nil ?
      → 是: color.Style(decoration.ToOpts())    // 纯装饰，无颜色
      → 否: 有任一 RGB 颜色 ?
          → 是: deriveRGBStyle()                 // 使用 color.RGBStyle
          → 否: deriveBasicStyle()               // 使用 color.Style (16色)
```

- **deriveBasicStyle**：将 `color.Color` 切片 + 装饰选项组合为 `color.Style`
- **deriveRGBStyle**：创建 `color.RGBStyle`，设置前景/背景 RGB，同时将非 RGB 的一方提升为 RGB（保持兼容性）

### 2.6 Color 与 Decoration

[Color](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/color.go#L5-L8) 是对 `gookit/color` 的封装：

```go
type Color struct {
    rgb   *color.RGBColor   // 24位真彩色
    basic *color.Color       // 16色基本色
}
```

- `IsRGB()` 判断是否为真彩色
- `ToRGB(isBg)` 将基本色转换为 RGB（背景色需要特殊处理 `*c.basic - 10`，这是 gookit/color 的已知问题）

[Decoration](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/decoration.go#L5-L10) 是纯布尔组合：

```go
type Decoration struct {
    bold, underline, reverse, strikethrough bool
}
```

- `Merge()` 是加法合并：任一方为 true 则结果为 true
- `ToOpts()` 转换为 `color.Opts`（即 `[]color.Color`）

### 2.7 MergeStyle 合并规则

```
TextStyle.MergeStyle(other)
  → decoration: 加法合并
  → fg: other.fg 不为 nil 则覆盖
  → bg: other.bg 不为 nil 则覆盖
  → 重新 deriveStyle()
```

这用于合成如 `CherryPickedCommitTextStyle = bgColor.MergeStyle(fgColor)` 这样的复合样式。

---

## 第三阶段：界面应用

### 3.1 应用入口：setColorScheme()

定义在 [gui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/gui.go#L1174-L1183)：

```go
func (gui *Gui) setColorScheme() {
    userConfig := gui.UserConfig()
    theme.UpdateTheme(userConfig.Gui.Theme)

    gui.g.FgColor = theme.InactiveBorderColor
    gui.g.SelFgColor = theme.ActiveBorderColor
    gui.g.FrameColor = theme.InactiveBorderColor
    gui.g.SelFrameColor = theme.ActiveBorderColor
}
```

这里将 gocui.Attribute 赋值给 gocui.Gui 的四个全局颜色属性，控制所有 View 的边框和标题渲染。

### 3.2 视图属性配置：configureViewProperties()

定义在 [views.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/views.go#L159-L179)：

```go
func (gui *Gui) configureViewProperties() {
    // 边框样式
    frameRunes := []rune{'─', '│', '┌', '┐', '└', '┘'}
    switch gui.c.UserConfig().Gui.Border {
        case "double":  frameRunes = []rune{'═', '║', '╔', '╗', '╚', '╝'}
        case "rounded": frameRunes = []rune{'─', '│', '╭', '╮', '╰', '╯'}
        case "hidden":  frameRunes = []rune{' ', ' ', ' ', ' ', ' ', ' '}
        case "bold":    frameRunes = []rune{'━', '┃', '┏', '┓', '┗', '┛'}
    }

    for _, mapping := range gui.orderedViewNameMappings() {
        (*mapping.viewPtr).FrameRunes = frameRunes
        (*mapping.viewPtr).BgColor = gui.g.BgColor
        (*mapping.viewPtr).FgColor = theme.GocuiDefaultTextColor
        (*mapping.viewPtr).SelBgColor = theme.GocuiSelectedLineBgColor
        (*mapping.viewPtr).SelFgColor = gui.g.SelFgColor
        (*mapping.viewPtr).InactiveViewSelBgColor = theme.GocuiInactiveViewSelectedLineBgColor
    }
}
```

每个 View（status、files、branches、commits、stash、main 等）都统一设置了：
- `FgColor`：默认文字颜色（来自 `defaultFgColor`）
- `SelBgColor`：选中行背景（来自 `selectedLineBgColor`）
- `SelFgColor`：选中行前景（与激活边框色相同）
- `InactiveViewSelBgColor`：非焦点视图选中行背景

### 3.3 gocui 渲染管线

当 gocui 绘制一帧时，核心流程：

```
Gui.flush()
  → 遍历所有 View
  → 绘制 View 内容 (drawContent)
      → 每个 cell 有 fgColor/bgColor
      → 选中行: fgColor=SelFgColor, bgColor=SelBgColor
      → 非焦点视图选中行: bgColor=InactiveViewSelBgColor
  → 绘制 View 边框 (drawFrame)
      → 激活视图: fgColor=SelFgColor (ActiveBorderColor), frameColor=SelFrameColor
      → 非激活视图: fgColor=FgColor, frameColor=FrameColor (InactiveBorderColor)
  → tcellSetCell(x, y, ch, fg, bg, outputMode)
      → getTcellStyle(oldStyle{fg, bg, outputMode})
          → 从 Attribute 中提取颜色和效果
          → 设置 tcell.Style 的 Foreground/Background
          → setTcellFontEffectStyle() 应用 bold/underline/reverse 等
      → Screen.Put(x, y, ch, style)  // 最终写入终端
```

参考 [tcell_driver.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gocui/tcell_driver.go#L104-L124) 和 [view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gocui/view.go#L1520-L1534)。

### 3.4 内容文本的样式应用

与 UI 框架层的 gocui.Attribute 不同，内容文本（commit 列表、文件名、分支名等）使用 TextStyle 的 `Sprint()`/`Sprintf()` 方法，直接在字符串中嵌入 ANSI 转义序列。

**presentation 层**是主要消费者，典型模式：

```go
// commits.go
hashColor := theme.DefaultTextColor
if isCherryPicked {
    hashColor = theme.CherryPickedCommitTextStyle
}
result := hashColor.Sprint(hash) + " " + theme.DefaultTextColor.Sprint(name)
```

各 presentation 文件使用的主题变量：

| 文件 | 使用的主题变量 |
|---|---|
| [commits.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/commits.go) | DefaultTextColor, CherryPickedCommitTextStyle, DiffTerminalColor |
| [files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/files.go) | DefaultTextColor, UnstagedChangesColor |
| [branches.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/branches.go) | DefaultTextColor, DiffTerminalColor |
| [tags.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/tags.go) | DefaultTextColor, DiffTerminalColor |
| [stash_entries.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/stash_entries.go) | DefaultTextColor, DiffTerminalColor |
| [remotes.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/remotes.go) | DefaultTextColor, DiffTerminalColor |
| [reflog_commits.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/reflog_commits.go) | DefaultTextColor, CherryPickedCommitTextStyle, DiffTerminalColor |
| [remote_branches.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/remote_branches.go) | DiffTerminalColor |
| [submodules.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/submodules.go) | DefaultTextColor |
| [worktrees.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/worktrees.go) | DefaultTextColor |

### 3.5 Options 底栏的样式应用

[options_map.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/options_map.go#L57-L66) 中：

```go
displayStyle := theme.OptionsFgColor  // 默认使用 optionsTextColor
if binding.DisplayStyle != nil {
    displayStyle = *binding.DisplayStyle  // 特定绑定可覆盖
}
formatted := info.style.Sprintf(plainText)  // TextStyle.Sprintf() 嵌入 ANSI
```

模式激活时使用特殊颜色：
- cherry-picking: `style.FgCyan`
- bisect: `style.FgGreen`
- rebase/merge 进行中: `style.FgYellow`
- patch building: `style.FgYellow`

### 3.6 动态颜色：作者颜色与分支颜色

**作者颜色**（[authors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/authors/authors.go#L76-L98)）：

```
SetCustomAuthors(customAuthorColors)
  → utils.SetCustomColors() 将 map[string]string 转为 map[string]*TextStyle
  → 写入 authorStyleCache
  → 匹配 "*" 通配符时所有作者统一颜色
  → 未配置时: trueColorStyle() 用 MD5 哈希生成 HSL 随机色
```

**分支颜色**（[branches.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/branches.go#L268-L273)）：

```
SetCustomBranches(customBranchColors, isRegex)
  → utils.SetCustomColors() 将 map[string]string 转为 map[string]*TextStyle
  → 包装为 colorMatcher{patterns, isRegex}
  → 匹配时: colorMatcher.match(name) 支持正则或精确匹配
```

### 3.7 图标颜色

图标系统（[icons.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/icons/icons.go)）由 `nerdFontsVersion` 配置驱动，图标本身携带颜色字符串（如 `"green"`、`"#ff0000"`），通过 [file_icons.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/icons/file_icons.go) 和 [git_icons.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/icons/git_icons.go) 中的硬编码映射表定义。用户可通过 `customIcons` 配置覆盖。

---

## 完整调用链路图

```
┌───────────────────────────────────────────────────────────────────┐
│  config.yml (用户配置文件)                                         │
│    gui:                                                            │
│      theme:                                                        │
│        activeBorderColor: [green, bold]                            │
│        ...                                                         │
│      authorColors: { "John": "#ff0000" }                           │
│      branchColorPatterns: { "feature/.*": "yellow" }              │
└─────────────────────────────┬─────────────────────────────────────┘
                              │ yaml.Unmarshal
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  config.ThemeConfig + config.GuiConfig                             │
│  (所有字段均为 []string 或 map[string]string)                       │
└─────────────────────────────┬─────────────────────────────────────┘
                              │ onUserConfigLoaded()
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  setColorScheme()                                                  │
│  ├── theme.UpdateTheme(themeConfig)                               │
│  │   ├── GetGocuiStyle() → gocui.Attribute (位运算OR合并)           │
│  │   │   输出: ActiveBorderColor, GocuiSelectedLineBgColor, ...   │
│  │   └── GetTextStyle()  → style.TextStyle (对象组合)              │
│  │       输出: DefaultTextColor, CherryPickedCommitTextStyle, ... │
│  ├── gui.g.FgColor/SelFgColor/FrameColor/SelFrameColor 赋值       │
│  └── configureViewProperties()                                    │
│      └── 每个 View.FgColor/SelBgColor/SelFgColor/InactiveViewSelBgColor │
│                                                                    │
│  authors.SetCustomAuthors()     → authorStyleCache                 │
│  presentation.SetCustomBranches() → colorMatcher.patterns          │
└─────────────────────────────┬─────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌──────────────────────────┐   ┌──────────────────────────────────┐
│  UI 框架层渲染            │   │  内容文本渲染                      │
│  (gocui.Attribute)       │   │  (style.TextStyle)                │
│                          │   │                                    │
│  View.FgColor            │   │  theme.DefaultTextColor.Sprint()  │
│  View.SelBgColor         │   │  theme.UnstagedChangesColor       │
│  View.SelFgColor         │   │  theme.CherryPickedCommitTextStyle│
│  Gui.FrameColor          │   │  theme.OptionsFgColor             │
│  Gui.SelFrameColor       │   │  authorStyle / branchStyle        │
│          │               │   │          │                         │
│          ▼               │   │          ▼                         │
│  tcellSetCell()          │   │  TextStyle.Sprint/Sprintf          │
│  → getTcellStyle()       │   │  → deriveStyle()                  │
│  → tcell.Style           │   │  → gookit/color.Style             │
│  → Screen.Put()          │   │  → ANSI 转义序列                   │
│                          │   │  → 写入 View 内部 buffer           │
└──────────────────────────┘   └──────────────────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              ▼
                    终端屏幕输出 (tcell)
```

---

## 关键文件索引

| 层次 | 文件 | 职责 |
|---|---|---|
| 配置定义 | [user_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/config/user_config.go#L215-L241) | ThemeConfig 结构体与默认值 |
| 配置加载 | [app_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/config/app_config.go#L139-L208) | YAML 加载、迁移、校验 |
| 主题转换 | [theme.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/theme/theme.go#L51-L76) | UpdateTheme()，双轨分发 |
| gocui 样式转换 | [gocui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/theme/gocui.go#L25-L45) | GetGocuiAttribute/GetGocuiStyle |
| TextStyle 转换 | [style.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/theme/style.go#L9-L44) | GetTextStyle() |
| TextStyle 核心 | [text_style.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/text_style.go#L28-L36) | TextStyle 结构体、deriveStyle |
| 颜色封装 | [color.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/color.go#L5-L8) | Color 结构体 (basic+RGB) |
| 装饰封装 | [decoration.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/decoration.go#L5-L10) | Decoration 结构体 |
| 预定义样式 | [basic_styles.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/style/basic_styles.go#L9-L52) | ColorMap、FgXxx、BgXxx |
| Attribute 定义 | [attribute.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gocui/attribute.go#L10-L64) | gocui.Attribute 位布局 |
| tcell 桥接 | [tcell_driver.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gocui/tcell_driver.go#L104-L124) | Attribute → tcell.Style |
| GUI 应用 | [gui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/gui.go#L1174-L1183) | setColorScheme() |
| 视图配置 | [views.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/views.go#L159-L179) | configureViewProperties() |
| 作者颜色 | [authors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/authors/authors.go#L76-L98) | 哈希着色 + 自定义覆盖 |
| 分支颜色 | [branches.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/gui/presentation/branches.go#L268-L273) | 正则/精确匹配着色 |
| 通用颜色工具 | [color.go](file:///d:/fz/0601-2/solo-dogfeeding/code/29-lazygit/pkg/utils/color.go#L60-L68) | SetCustomColors() |
