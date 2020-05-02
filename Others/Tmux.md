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

```bash
tmux # 创建一个无名称 session，tmux 会自动分配一个编号
tmux new -s temp # 创建命名为 temp 的 session
tmux rename-session -t 0 temp # 重命名一个 session
<prefix> + d # 断开当前会话，同 tmux detach 命令
tmux a # 进入第一个 session
tmux a -t temp # 进入指定名称的 session
tmux kill-session -t temp # 关闭名称为 temp 的 session
tmux kill-server # 关闭 Server
<prefix> + s # 查看所有的 session，也可以使用 tmux ls 命令
tmux switchc -t temp # 切换到指定的 session
<prefix> + $ # 重命名当前 session
```

### Window

```bash
<prefix> c # 创建 window
tmux new-window -n t-win # 创建一个命名 window
<prefix> + [number] # 选择第 n 个 window
<prefix> + p/n # 选择上一个或下一个 window
<prefix> + & # 关闭 window 同 exit 
<prefix> + w # 列出所有 window 然后进行选择，通过 j/k 前后选择
<prefix> + f # 搜索 window
<prefix> + , # 重命名当前 window 同 tmux rename-window 
```

### Panel

```bash
<prefix> + % # 水平分割
<prefix> + \" # 垂直分割
<prefix> + ; # 光标移动到上一个 panel
<prefix> + o # 光标移动到下一个 panel
<prefix> + q # 显示 panel 序号进行切换
<prefix> + b # 关闭当前 panel
<prefix> + ! # 将当前 panel 移动到新的 window
<prefix> + {/} # 当前 panel 左移/右移
<prefix> + Ctrl + o # 当前 panel 上移
<prefix> + Alt + o # 当前 panel 下移
```

### 其他快捷键

```bash
tmux list-keys # 列出所有的 keys
tmux list-commands # 列出所有的命令
tmux info # 当前 session 的信息
```

参考：[Tmux 使用教程](https://www.ruanyifeng.com/blog/2019/10/tmux.html)