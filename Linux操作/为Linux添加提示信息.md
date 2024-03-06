#### 为 Linux 操作系统设置提示信息

有时候我们为了让自己的 Linux 操作系统看起来更有个性一些会希望登录成功后输出一些内容.比如如下这种效果:

```text
  System information as of Wed Mar  6 04:34:39 PM CST 2024

  System load:  0.0                Temperature:           48.0 C
  Usage of /:   5.8% of 113.80GB   Processes:             154
  Memory usage: 5%                 Users logged in:       1
  Swap usage:   0%                 IPv4 address for wlo1: 192.168.2.127


0 updates can be applied immediately.


 ________________________________________
/     Only two things are infinite,      \
\   the universe and human stupidity.    /
 ----------------------------------------
             ^__^     O   ^__^
     _______/(oo)      o  (oo)\_______
 /\/(       /(__)         (__)\       )\/\
    ||w----||                 ||----w||
    ||     ||                 ||     ||
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Last login: Wed Mar  6 16:09:48 2024 from 192.168.2.124
```

这个输出信息主要又两部分,上面的一些信息是 Ubuntu 系统自动输出的,显示了一些基本的信息.而下面的一些信息是我们自己设置的.

```text
 ________________________________________
/     Only two things are infinite,      \
\   the universe and human stupidity.    /
 ----------------------------------------
             ^__^     O   ^__^
     _______/(oo)      o  (oo)\_______
 /\/(       /(__)         (__)\       )\/\
    ||w----||                 ||----w||
    ||     ||                 ||     ||
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

然而设置这个内容其实非常简单,只需要更改 **/etc/motd** 文件将上面的内容粘贴进去并保存就行了.

当然也可以是别的字符.这个可以根据自己的喜好去设计和修改.

其实 Linux 系统中还有另外两个文件也可以显示输出信息,只是不是在登陆后.分别是 **/etc/issue** 和 **/etc/issue.net**.这两个文件分别是通过串口登录系统显示信息和通过 Telnet 登录显示.

由于这里我们只需要调整登陆后的输出信息,所以我们只需要修改 **/etc/motd** 文件就行了.
