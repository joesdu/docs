- step1: 安装语言包

```shell
sudo apt install language-pack-zh-hans
```

- step2: 个人建议不用装这个命令的内容

```shell
sudo apt install $(check-language-support)
```

- step3:

```shell
sudo nano /etc/default/locale
```

- step4: 修改为:

```conf
# 第一句最主要,其他可以不添加,但是为了保险往往都加上比较稳,亲测只写第一句就行
LANG="zh_CN.UTF-8"
# LANGUAGE="zh_CN:zh"
# LC_NUMERIC="zh_CN"
# LC_TIME="zh_CN"
# LC_MONETARY="zh_CN"
# LC_PAPER="zh_CN"
# LC_NAME="zh_CN"
# LC_ADDRESS="zh_CN"
# LC_TELEPHONE="zh_CN"
# LC_MEASUREMENT="zh_CN"
# LC_IDENTIFICATION="zh_CN"
# LC_ALL="zh_CN.UTF-8"
```

- step5:

```shell
sudo nano /etc/environment
```

- step6: 添加以下内容:

```conf
# 第一句最主要,其他可以不添加,但是为了保险往往都加上比较稳
LANG="zh_CN.UTF-8"
# LANGUAGE="zh_CN:zh"
# LC_NUMERIC="zh_CN"
# LC_TIME="zh_CN"
# LC_MONETARY="zh_CN"
# LC_PAPER="zh_CN"
# LC_NAME="zh_CN"
# LC_ADDRESS="zh_CN"
# LC_TELEPHONE="zh_CN"
# LC_MEASUREMENT="zh_CN"
# LC_IDENTIFICATION="zh_CN"
# LC_ALL="zh_CN.UTF-8"
```

- step7:

```shell
sudo reboot
```
