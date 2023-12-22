#### 基于 Ubuntu Server 环境安装 Cinnamon 桌面环境并开启 VNC Server

> 最近由于一些原因,需要给之前的 Ubuntu Server 电脑安装一个桌面环境,并不希望丢失原来的数据.所以就有了这个教程

- 首先安装 Cinnamon 桌面环境

```bash
# 想将操作系统升级到最新版
sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y && sudo apt autoclean -y && sudo snap refresh
# 然后安装 Cinnamon
sudo apt install cinnamon-desktop-environment -y
```

- 等待安装完成后我们就先安装 VNC 服务,并完成后一起重启生效.

> 有几种不同的 VNC 服务器,例如 TightVNC ,TigerVNC 和 x11vnc.每个 VNC 服务器在速度和安全性方面都有其优点和缺点.我们将使用 TigerVNC,它是积极维护的高性能 VNC 服务器.主要是免费.

- [TigerVNC 官网](https://tigervnc.org)
- 安装 TigerVNC 服务端程序

```bash
sudo apt install tigervnc-standalone-server tigervnc-common
```

- 安装完成后我们通过启动命令来初始化一下服务.

```bash
vncserver
```

- 系统将提示您输入并确认密码,以及是否将其设置为仅供查看的密码,这里我们选择 n,如果您选择设置仅查看密码.则用户将无法使用鼠标和键盘与 VNC 实例进行交互.
- 然后就会输出如下信息.

```text
You will require a password to access your desktops.

Password:
Verify:
Would you like to enter a view-only password (y/n)? n
A view-only password is not used
/usr/bin/xauth:  file /home/joe/.Xauthority does not exist

New Xtigervnc server 'm920x:1 (joe)' on port 5901 for display :1.
Use xtigervncviewer -SecurityTypes VncAuth -passwd /tmp/tigervnc.4s0mxZ/passwd :1 to connect to the VNC server.
```

- 首次运行 vncserver 命令时,它将创建密码文件并将其存储在~/.vnc 目录中.
- 请注意上面输出中主机名后的:1,这是 vnc 服务器的显示端口号.vnc 服务器将会监听 TCP 端口 5901,即 5900 + 1.
- 如果运行 vncserver 命令创建第二个实例,它将在使用下一个显示端口即:2,这意味着 VNC 服务器将会监听端口 5902,即 5900 + 2.
- 在继续下一步之前,请先停止 VNC 实例.在我们的例子中,VNC 服务器在端口 5901 运行,显示端口是:1.因此停止显示端口:1 的是命令如下:

```bash
vncserver -kill :1
```

- 输出类似如下内容表示停止成功.

```text
Killing Xtigervnc process ID 32513... success!
```

- 由于我们的 Ubuntu Server 没有显示器和键鼠,所以我们希望 VNC 服务开机自启.所以我们创建如下配置文件

```bash
# 创建 service 服务文件
sudo nano /etc/systemd/system/vncserver.service
```

- 写入如下内容

```conf
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=simple
User=pi
PAMName=login
PIDFile=/home/%u/.vnc/%H%i.pid
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/vncserver %i -geometry 1920x1200 -alwaysshared -fg -localhost no
ExecStop=/usr/bin/vncserver -kill %i
Restart=always

[Install]
WantedBy=multi-user.target
```

- 其中需要注意的有：
- 1.修改 User=pi 为真实用户名.我这里是 NanoPi 所以用户名为 pi.
- 2.ExecStart=/usr/bin/vncserver %i -geometry 1920x1200 -alwaysshared -fg -localhost no 中-fg 表示进程在前台运行并在 VNC 服务器的 X 会话终止后终止它,-localhost no 表示所有客户端都可以连接.
- 3.Restart=always 重启 VNC Server.在客户端注销后,可以重新连接服务器.如果没有加这一条,客户端注销后,vncserver.service 会变成 stop 状态.

- 上面的服务文件写好后就可以启动服务了.

```bash
# 配置服务开机自启
sudo systemctl enable vncserver.service
# 启动服务
sudo systemctl start vncserver.service
# 查看服务状态
sudo systemctl status vncserver.service
```

- 完成上述操作后,我们重启一下服务器,方便我们的服务完成安装.

```bash
sudo reboot
```

- 然后我们就可以通过[下载的 TigerVNC 客户端](https://sourceforge.net/projects/tigervnc/files/stable/1.13.1/tigervnc64-1.13.1.exe/download)访问服务器了

- 写在最后,VNC 不是加密协议,可能会受到数据包嗅探.推荐的方法是创建 SSH 隧道,使用加密的数据连接到远程服务器.
- 新版的 Windows 已经内置了 OpenSSH 所以,Windows,Linux 和 MacOS 都可以使用如下命令进行 SSH 隧道创建.

```bash
ssh -L 127.0.0.1:5901:192.168.2.11:5901 -N -f -l username remote_server_ip -p 2333 -o TCPKeepAlive=yes
```

- 上述命令的各个 IP 以及端口的配置需要自行替换一下,若是不懂参数的含义,建议使用 AI 查询或者其他搜索引擎进行查询.
- 开启 SSH 隧道后,我们便可以用 VNC 客户端通过本机的 5901 端口进行连接,这样相当于利用 SSH 隧道对 VNC 服务的数据进行加密了.
