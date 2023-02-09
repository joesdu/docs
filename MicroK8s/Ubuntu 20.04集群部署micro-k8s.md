最近由于我一个学习群里的大佬过于卷.所以没办法,只能强迫自己跟着学习提升下自己的能力.
于是乎就打算先学学 k8s,但是传统的 k8s 过于复杂.为了方便.就选择了 micro-k8s,结合熟悉的 Ubuntu 系统.配置安装都非常的方便.
这里我们照常准备了 3 台机器,1 台 master2 台 workder.

- 具体的配置如下:

```json
master: {
  CPU: 2,
  RAM: 4GB,
  ROM: 50GB,
   IP: 192.168.2.24
},
worker: {
  CPU: 8,
  RAM: 16GB,
  ROM: 50GB,
   IP: 192.168.22,192.168.23
}
```

- 首先配置好 3 台服务器.然后使用 XShell 或者类似工具链接上每一台服务器.
- 安装 micro-k8s
- micro-k8s 的安装非常简单,可以参考[GitHub 的 Readme.md](https://github.com/ubuntu/microk8s)
- https://github.com/ubuntu/microk8s
- 安装命令如下:

```shell
sudo snap install microk8s --classic
```

- 三台服务器均需要安装,等待数秒(具体时间视网络环境而定),安装完成后,就可以开始配置了.
- 在主节点上,执行如下命令,获取添加从节点的命令.

```shell
sudo microk8s.add-node
```

- 可以看到控制台会输出如下结果的信息:

```
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.2.24:25000/f872a823e90242ff12a3e3db202e3e05/030ba9fe293e

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 192.168.2.24:25000/f872a823e90242ff12a3e3db202e3e05/030ba9fe293e --worker

If the node you are adding is not reachable through the default interface you can use one of the following:
microk8s join 192.168.2.24:25000/f872a823e90242ff12a3e3db202e3e05/030ba9fe293e
```

- 其中 microk8s join 192.168.2.24:25000/f872a823e90242ff12a3e3db202e3e05/030ba9fe293e 就是添加子节点的命令.
- 这个时候切换到从节点的两台服务器分别执行如下命令,由于我们是非 root 账户登录,执行命令需要添加 sudo

```shell
# 可在后边添加 --worker参数,直接添加为worker节点,就不再需要后边设置了.
sudo microk8s join 192.168.2.24:25000/f872a823e90242ff12a3e3db202e3e05/030ba9fe293e --worker
```

- 执行这个命令的时间不要拖得太长,避免 key 失效,若是失效了,从新执行上述添加节点的命令获取新的即可.
- 在从节点执行加入主节点的命令后,从节点就可以不用管了.这个时候返回到主节点.
- 为了方便我们需要将 microk8s 的命令弄个别名,不然每次需要输入的命令太长.

```shell
sudo snap alias microk8s.kubectl kubectl
```

- 添加 dns 服务

```shell
sudo microk8s enable dns
```

- 设置好别名后,即可通过 kubectl 来执行命令了.
  如:

```shell
# 获取节点信息
sudo kubectl get nodes
# 查看集群信息
sudo kubectl cluster-info
# 查看pods
sudo kubectl get pods
sudo kubectl get services
```

我这里的环境下会输出如下信息:(这里的 worker 和 master 的设置下边会讲到.一开始所有节点 ROLES 信息应该是一致的)
NAME STATUS ROLES AGE VERSION
k8s-02 Ready worker 113d v1.25.6
k8s-01 Ready master 113d v1.25.6
k8s-03 Ready worker 113d v1.25.6

- 接下来设置 master 节点和 worker 节点

```shell
# 设置master节点不调度pod,其中的k8s-01需要根据自身实际情况进行替换,这里为服务器的名字,下方的命令也是如此
sudo kubectl taint node k8s-01 node-role.kubernetes.io/master="":NoSchedule
# 设置角色为master
sudo kubectl label node k8s-01 node-role.kubernetes.io/master=master
# 设置工作节点
sudo kubectl label node k8s-02 node-role.kubernetes.io/worker=worker
sudo kubectl label node k8s-03 node-role.kubernetes.io/worker=worker
```

- 设置完成后,再次获取 nodes 的节点信息,就可以看到工作节点和主节点的分布了.

```shell
# 获取节点信息
sudo kubectl get nodes
```

- 获取 k8s 的 config 信息,使用[Lens](https://k8slens.dev)
- https://k8slens.dev
- Lens 开始收费了,所以这里推荐他的同款替代品[OpenLens](https://github.com/MuhammedKalkan/OpenLens/releases)
- https://github.com/MuhammedKalkan/OpenLens/releases

```shell
sudo microk8s.config
```

- 为主节点添加 IP 并重新生成证书
- 注意: MicroK8s 版本 1.21 有个 bug(目前我是 1.25.6 所以没执行这句), 需要修改/var/snap/microk8s/current/var/lock/no-cert-reissue 文件,避免其无法触发 apiserver 更新 csr

```shell
sudo mv /var/snap/microk8s/current/var/lock/no-cert-reissue /var/snap/microk8s/current/var/lock/no-cert-reissue.back
```

- 编辑 /var/snap/microk8s/current/certs/csr.conf.template 文件, 找到 alt_names 节点, 刚安装完毕一般只到 IP.3, 我们就可以再往后加 IP.4 IP.5 一般就加其他的局域网 IP,或者映射后的公网 IP 之类的

```conf
[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = 127.0.0.1
IP.2 = 10.152.183.1
IP.3 = 192.168.31.121
IP.4 = 192.168.192.121
```

复制代码
更改完毕执行下

```shell
ll /var/snap/microk8s/current/certs/csr.conf*
```

- 查看下 csr.conf 和 csr.conf.rendered 更新时间是不是大于等于 csr.conf.template, 是的话就说明重新生成了 csr,然后再执行

- 最后再把前边改了的文件名改回去,当然,如果后续版本修复了这个问题也就不用再改名称了

```shell
sudo mv /var/snap/microk8s/current/var/lock/no-cert-reissue.back /var/snap/microk8s/current/var/lock/no-cert-reissue
```

- 复制输出的内容,添加到 lens 的 config 文件中即可链接 micro-k8s 了
- 使用 Lens 链接成功后,可以进入集群的 Setting 设置中开启一些服务监控集群状态.
