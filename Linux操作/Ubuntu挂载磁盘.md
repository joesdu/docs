最近客户需要做资源服务器,所以整了个大容量磁盘给服务器,现需要挂载.话不多说,直接上步骤:

- 查看现有磁盘信息:

```shell
sudo fdisk -l
```

- 格式化磁盘:(其中磁盘路径由第一步查询出来,这里使用整个磁盘不进行分区)

```shell
sudo mkfs -t ext4 /dev/vdb
```

- 创建要挂载的文件夹

```shell
sudo mkdir /mnt/otherdevice
```

- 挂载硬盘到该文件夹上

```shell
sudo mount /dev/vdb /mnt/otherdevice
```

- 进入 fstab 修改配置(重启自动挂载)

```shell
sudo nano /etc/fstab
```

- 在最后一行添加如下内容:

```conf
/dev/vdb /mnt/otherdevice ntfs defaults 0 0
```

- 自此,Ubuntu 磁盘挂载就完成了.若是需要对磁盘分区,则可以在格式化之前进行分区后再格式化

- 卸载磁盘

```bash
sudo umount /mnt/otherdevice
```

/etc/fstab 添加 NTFS 挂载
