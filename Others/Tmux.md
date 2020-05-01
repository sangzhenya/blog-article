# Tmux

Tmux 是一个终端复用器类自由软件，用户可以通过 tmux 在一个终端内管理多个分离的会话，窗口及面板，对于同时使用多个命令行，或多个任务时非常方便。

tmux 主要由以下几个模块组成：

1. server 服务，tmux 运行的基础服务，以下的模块均依赖此服务
2. session 会话，一个服务可以包含多个会话
3. window 窗口，一个会话可以包含多个窗口
4. panel 面板，一个窗口可以包含多个面板】

## 配置

用户私人配置文件在 `~/.tmux.conf` 中，全局文件在 `/etc/tmux.conf` 中。默认前缀快捷键是 `Ctrl + b`，如果要修改为 `Ctrl + a` 可以在 `.tmux.conf` 中配置如下

```conf
unbind C-b
set -g prefix C-a
bind C-a send-prefix
```

如果需要使用鼠标，则可以另外添加 `set-option -g mouse on`。然后使用 `tmux source .tmux.conf` 重新加载配置即可。


## 命令及快捷键

### Session

### Window

### Panel

