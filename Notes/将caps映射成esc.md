---
created: 2026-04-01
tags:
  - note
  - journal
---
#### 方案 A：针对 X11 图形界面（最简单）

如果你使用的是桌面环境（如 GNOME 默认的 X11 或手动配置的 X），只需运行一行命令：

Bash

```
setxkbmap -option caps:escape
```

_注：若要永久生效，可将此命令加入 `.xinitrc`。_

#### 方案 B：针对 Wayland (GNOME)

如果你使用 Ubuntu 默认的 GNOME Wayland，可以通过 `gsettings` 修改：

Bash

```
gsettings set org.gnome.desktop.input-sources xkb-options "['caps:escape']"
```