> 为了学习 Redis,所以打算在自己的服务器上安装一个 Redis 集群用于学习.话不多说,直接上干货.查阅资料后了解到创建 Redis-Cluster 集群建议至少 6 台服务器.所以首先进入 ESXi 后台创建 6 台服务器.为了防止服务器 IP 变化,所以先设置好静态 IP.具体的系统安装步骤和静态 IP 设置这里就不赘述了.

- 创建的服务器列表如下: [ 192.168.2.15, 192.168.2.16, 192.168.2.17, 192.168.2.18, 192.168.2.19, 192.168.2.20 ]

创建好后,我们就进入服务器中.
接下来就进入正题了.

- 首先要创建集群,每一台服务器上都应该安装好 redis-server.
- 查看官网可以发现 Ubuntu 的安装命令如下.

```shell
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt update
sudo apt install redis
```

- 新版 Ubuntu22.04 以后推荐使用如下命令,新版的 Ubuntu 好像是不推荐使用 gpg 密钥了.

```shell
sudo add-apt-repository ppa:redislabs/redis
sudo apt update
sudo apt install redis
```

- 每台服务器安装完成后,先调整 Redis 配置文件
- 由于我们是使用 apt 包管理工具安装的 Redis,所以无法使用自定义配置文件来启动 Redis,所以我们得更改默认配置文件

```shell
sudo nano /etc/redis/redis.conf
```

- 然后调整一些东西具体内容如下:

```conf
# 绑定IP
bind 0.0.0.0
# 保护模式
protected-mode no
# 是否后台运行
daemonize yes
# 设置主服务器密码
requirepass 123456
# 设置从服务器密码
masterauth 123456
# 开启AOF持久化
appendonly yes
# 集群相关设置
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000
```

- 将此配置文件保存后,复制到其他节点上,所有节点使用相同的配置文件.
- 然后分别启动各个服务器的 redis 服务

```shell
sudo systemctl start redis-server
```

- 所有的服务器成功开启 reids 后,即可创建集群

```shell
redis-cli --cluster create 192.168.2.15:6379 192.168.2.16:6379 192.168.2.17:6379 192.168.2.18:6379 192.168.2.19:6379 192.168.2.20:6379 --cluster-replicas 1 -a '123456'
```

- 等待数秒.然后会输出确认信息,输入 yes 即可成功创建集群
- 使用命令登入客户端即可正常使用 redis-cli

```shell
redis-cli -c -h 192.168.2.15 -a '123456'
```

- 设置 redis 开机自启

```shell
sudo systemctl enable redis-server.service
```
