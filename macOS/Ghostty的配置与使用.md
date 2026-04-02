# 1. 简介

Ghostty 是一款在 macOS 和 Linux 上的高性能的终端模拟器，使用 Zig 语言编写，使用 GPU 加速渲染，使得体验非常流畅。

官方网站：https://ghostty.org/

# 2. 配置

配置文件存放在 ~/.config/ghostty/config，也可以通过 ⌘ + , 打开配置文件编辑窗口。

我的配置：

```ini
### 字体
font-family = JetBrainsMonoNerdFont
font-size = 14
font-thicken = true
adjust-cell-height = 1

### 主题
#theme = Catppuccin Mocha
#theme = Catppuccin Latte
#theme = light:Catppuccin Latte, dark:Catppuccin Mocha
theme = GitHub Dark
#theme = GitHub Light Default
#theme = Monokai Classic
#theme = Dracula

### 窗口外观
background-opacity = 1
background-blur-radius = 20
macos-titlebar-style = transparent
window-padding-x = 10
window-padding-y = 8
window-save-state = always
window-theme = auto
window-colorspace = display-p3

### 光标
cursor-style = bar
cursor-style-blink = true
cursor-opacity = 1

### 鼠标
mouse-hide-while-typing = true
copy-on-select = clipboard
right-click-action = paste

### 安全
clipboard-paste-protection = true
clipboard-paste-bracketed-safe = true
window-padding-balance = true
confirm-close-surface = false

### Shell 集成
shell-integration = detect
shell-integration-features = ssh-terminfo,ssh-env,title

### 快捷键
keybind = cmd+shift+comma=reload_config
keybind = cmd+shift+e=equalize_splits

### 历史记录
scrollback-limit = 25000000
```

更多配置见官方说明：https://ghostty.org/docs/config/reference

# 3. 快捷键

⌘ + D 向右分屏

⌘ + ⇧ + D 向下分屏

⌘ + ⇧ + E 调整分屏为相同大小

⌘ + ⌥ + 方向键 切换分屏焦点

⌘ + [ 切换上一个分屏

⌘ + ] 切换下一个分屏

⌘ + ⇧ + enter 将当前分屏最大化

⌘ + enter 全屏显示

⌘ + , 打开配置

⌘ + ⇧ + , 重载配置

# 4. 命令

查看绑定的快捷键

```bash
ghostty +list-keybinds
```

自定义语法，并在配置文件添加。

```bash
# 格式：keybind = <修饰键+按键>=<动作>
keybind = cmd+n=new_tab
```

打开主题中心

```bash
ghostty +list-themes
```

列出字体

```bash
ghostty +list-fonts
```

列出当前绑定的快捷键

```bash
ghostty +list-keybinds
```

列出可用的动作

```bash
ghostty +list-actions
```

重置配置

```bash
ghostty --reset-config
```

