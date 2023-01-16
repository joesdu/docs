- 在中国的可以直接执行:

```shell
sudo timedatectl set-timezone Asia/Chongqing
```

- timedatectl 查看可以设置的时区

```shell
timedatectl list-timezones
```

- 带上 list-timezones 参数运行下，看到如下的结果：

```shell
timedatectl list-timezones
```

- timedatectl 设置时区

```shell
sudo timedatectl set-timezone Asia/Chongqing
```

- 设置成功后， 重新看下时间

```shell
date -R
```

- 输出如下:
  Wed, 05 May 2021 12:52:46 +0800
