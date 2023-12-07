---
title: X11 Forward
created: 2023-09-05 16:13:23, 星期二
modified: 2023-09-21 18:51:21, 星期四
tags: [vscode]
---
#vscode

Windows 用经典的 MobaXTerm 就可以，这里解决 Ubuntu 连接问题

### 加 -X 参数 ssh 连接

教程：[How to remotely display X11 application on an Ubuntu 20 desktop](https://www.it3.be/2021/06/24/remote-x11-on-ubuntu/)

首先确认 remote 端 `/etc/ssh/sshd_config` 开启了：

```
X11Forwarding yes
```

通过本地设置如下参数连接：

```shell
DISPLAY=:1 ssh -X 10.76.2.233
```

测试可成功（前一阵不行可能是服务端 sshd X11 配置没有开）
有一个小缺陷是 open3d 的显示界面相当卡，交互很不实时。偶尔有 BadWindow 报错

#### Remote X11 插件（废弃）

**==2023-04：[开发者表示](https://github.com/joelspadin/vscode-remote-x11) 此插件已经 deprecated==**

远程 server 上，扩展中安装 `Remote X11` 插件；
另一个带 SSH 括号的插件据说要本地装

### 快速测试

两个 GUI 小工具：xeyes 或者 xclock

### 中断连接

![[ssh配置使用#中断连接]]

### DISPLAY 窗口

`echo $DISPLAY` 可查看环境变量，此参数设置图形显示位置。可 X11 时默认值输出为 `localhost:10.0`，修改为其他报错

使用 xdpyinfo 可以查看到当前显示的更详细的信息，输出例如：

```
visual:
   visual id:    0xa3
   class:    TrueColor
   depth:    32 planes
   available colormap entries:    256 per subfield
   red, green, blue masks:    0xff0000, 0xff00, 0xff
   significant bits in color specification:    8 bits
```

DISPLAY 环境变量格式如下 `​​​host:NumA.NumB`​​。host 指 Xserver 所在的主机主机名或者 ip 地址，图形将显示在这一机器上，可以是启动了图形界面的 Linux/Unix 机器，也可以是安装了 Exceed、X-Deep/32 等 Windows 平台运行的 Xserver 的 Windows 机器。如果 Host 为空，则表示 Xserver 运行于本机，并且图形程序（Xclient）使用 unix socket 方式连接到 Xserver，而不是 TCP 方式。使用 TCP 方式连接时，NumA 为连接的端口减去 6000 的值，如果 NumA 为 0，则表示连接到 6000 端口；使用 unix socket 方式连接时则表示连接的 unix socket 的路径，如果为 0，则表示连接到 /tmp/.X11-unix/X0。NumB 则几乎总是 0

### 相关资料

感觉很少

[linux 图形显示 环境变量设置 export DISPLAY=:0.0的解释-CSDN博客](https://blog.csdn.net/unixumin/article/details/103400033)

[本机运行x程序出现：Can't open display 原因及其解决方法\_muye0503的博客-CSDN博客](https://blog.csdn.net/muye0503/article/details/8949840)

### 基本概念（反常）

关于基础介绍的教程：[本机运行x程序出现：Can't open display 原因及其解决方法\_muye0503的博客-CSDN博客](https://blog.csdn.net/muye0503/article/details/8949840)

通常当你从 hostA 登陆到 hostB 上运行 hostB 上的应用程序时：做为应用程序来说，hostA 是 client，但是作为图形来说，是在 hostA 上显示的，需要使用 hostA 的 Xserver，所以 hostA 是 server。因此在登陆到 hostB 前，需要在 hostA 上运行 `xhost +` 来使其它用户能够访问 hostA 的 Xserver

**Xserver 指的是本机！Xclient 反而是远程机器！**

### xhost 使用

linux-command 教程：[xhost 命令，Linux xhost 命令详解：制哪些X客户端能够在X服务器上显示 - Linux 命令搜索引擎](https://wangchujiang.com/linux-command/c/xhost.html)

xhost 命令 是 X 服务器的访问控制工具，用来控制哪些 X 客户端能够在 X 服务器上显示。该命令必须从有显示连接的机器上运行

直接输入 `xhost` 输出当前连接用户信息，如本机上输出结果为 `SI:localuser:uu`


【指令用法】
xhost + 是使所有用户都能访问 Xserver.
xhost + ip 使 ip 上的用户能够访问 Xserver.
xhost + nis:user@domain 使 domain 上的 nis 用户 user 能够访问
xhost + inet:user@domain 使 domain 上的 inet 用户能够访问。



