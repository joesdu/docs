## 操作该文档基本需求

- 操作员需熟悉 Linux 的常用命令,知晓各命令的含义和用途
- 了解 Linux 服务器的运维
- 基本的编程知识,了解 JavaScript 和其他面向对象编程语言
- 熟悉 Mongodb 数据库的基本操作,若是精通最好
- 能阅读英文文档
- 会操作和配置 Linux 服务器的防火墙规则

---

### 下载 Mongodb 数据库

- 首先去[Mongodb 官网下载中心](https://www.mongodb.com/download-center/community/releases)下载 Mongodb 数据安装程序.根据不同的操作系统下载系统相对应的安装包.
- 选择 \*.tgz 格式的包下载,其他模式的安装不太容易自定义安装
- 如本次服务器安装的是 Ubuntu 21.xx 的操作系统,这里我们就选择最新的 Ubuntu 系统对应版本,然后根据服务器 CPU 架构选择相应的包.这次我们选择 X86_X64 架构,若是使用 ARM 芯片则需选择 ARM64 架构.
- 下载好对应包后使用 root 账号登录,然后上传到服务器,或者使用服务器的 wget 命令直接下载到 root 目录下.

---

### 安装 Mongodb 数据库

- 解决先决条件安装依赖库

```shell
sudo apt install libcurl4 openssl liblzma5
```

- 上传好数据库安装包后,可参考 [官方的安装教程](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu-tarball/) 进行安装,这里我们选择 Install using .tgz Tarball 阅读其中的操作步骤进行安装.
- 使用 root 账户登录服务器后在 root 目录下执行如下命令.

- 解压数据库文件

```shell
tar -zxvf mongodb-linux-*.tgz
```

- 移动数据库文件到执行目录 [**注意: 其中<mongodb-install-directory>需替换成真实的文件目录如:/root/mongodb-linux-x86_64-ubuntu2004-5.0.0/**]

```shell
sudo cp <mongodb-install-directory>/bin/* /usr/local/bin/
```

- 然后给可执行文件添加执行权限

```shell
sudo chmod 755 -R /usr/local/bin
```

---

### 创建数据库配置文件

- 在 root 文件夹下创建 KeyFile 用于多节点认证

```shell
openssl rand -base64 745 > /root/KeyFile
```

- 调整认证文件为只读

```shell
sudo chmod 444 /root/keyFile
```

- 使用 nano 编辑器创建 Mongodb 运行配置文件

```shell
sudo nano /root/node1/mdb.config
sudo nano /root/node2/mdb.config
```

- 创建 Mongodb 数据文件夹

```shell
sudo mkdir /root/node1/data
sudo mkdir /root/node2/data
```

- 在上述所有 mdb.config 配置文件中填入如下内容

```conf
systemLog:
   destination: file
   path: "/root/node1/mongod.log"
   logAppend: true
storage:
   dbPath: "/root/node1/data"
   journal:
      enabled: true
processManagement:
   fork: true
net:
   bindIp: 127.0.0.1,192.168.31.236
   port: 27017
security:
   keyFile: "/root/KeyFile"
replication:
   replSetName: "rs0"
```

其中 path 的值根据不同节点的路径调整,dbPath 的值和 KeyFile 的值均是如此.
<br>其中 bindIp 的值填入 127.0.0.1 和当前服务器的 IP 地址.
<br>port 值在 node1 中填入 27017,node2 中填入 27018,node3 中填入 27019
<br>[**该配置文件及其注重格式,请一定按照所示对齐进行输入**]

---

### 配置数据库

- 启动数据库

```shell
mongod -f /root/node1/mdb.config & mongod -f /root/node2/mdb.config
```

等待数据库启动完成.

- 登入数据库

```shell
mongo
```

- 切换到 admin 数据库

```shell
use admin;
```

- 初始化数据库

```shell
rs.initiate();
```

- 创建管理员用户

```javascript
db.createUser({
  user: "aaa",
  pwd: "123456",
  roles: [{ role: "root", db: "admin" }],
});
```

其中用户名和密码根据实际需求自行调整 [**规则切勿调整,错误的规则会使账号无法登录**]

- 使用刚才创建的用户登入数据库,会提示输入密码

```javascript
db.auth("aaa");
```

- 调整主数据库的优先级

```javascript
var rsc = rs.config();
rsc.members[0].priority = 2;
rs.reconfig(rsc);
```

- 添加数据库节点(数据库集群)

```javascript
rs.add("192.168.31.236:27018");
rs.add("192.168.31.236:27019");
```

- 等待执行完成退出数据库

```shell
exit
```

- 至此数据库配置完成,可使用数据库工具进行链接上述操作详细输出内容可参考[文档](https://www.cnblogs.com/dygood/p/14851591.html)
- 完成上述操作后,将数据库端口开放 (**这里以 Ubuntu 服务器做示例**)

```shell
sudo ufw allow 27017
sudo ufw allow 27018
sudo ufw allow 27019
```
