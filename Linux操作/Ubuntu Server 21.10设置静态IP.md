Ubuntu Server 21.10 静态 IP 地址的设置和以前不同了.

- 经验证 22.04 该方案依旧可行.
- 默认动态 IP 地址对应的配置文件: /etc/netplan/\*.yaml
- 设置固定 ip 地址
  如:192.168.1.2 是本机 IP，192.168.1.1 是网关，f4:5b:11:4c:dc:bd 是网卡的 MAC 地址.

```shell
# 此处文件名按照自己的实际情况来
sudo nano /etc/netplan/*.yaml
```

```yaml
network:
  version: 2
  #renderer: networkd
  ethernets:
    eth0:
      #link-local: []
      optional: true
      dhcp4: no
      addresses:
       - 192.168.1.2/24
      routes:
        - to: default
          via: 192.168.1.1
      #match:
      #    macaddress: f4:5b:11:4c:dc:bd
      #set-name: eth0
      nameservers:
        addresses: [223.5.5.5,223.6.6.6,8.8.8.8,8.8.4.4]
        #search: []
```

- 然后执行命令应用设置

```shell
sudo netplan apply
```

- 重启系统

```shell
sudo reboot
```
