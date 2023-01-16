# exsi升级

#### 一、登录VMware官网下载最新的升级包
[官网地址](https://my.vmware.com)

- 选择 产品-产品与补丁
- 选择ESXI 选择最新版本7.0点搜索
- 下载日期更新最新的这个
#### 二、升级ESXI至最新版本
- ①将之前下载好的升级包上传到数据存储区
- ②开启ESXI的SSH功能
- ③通过SSH连接到ESXI确认下目前ESXI的版本及版本号
```shell
vmware -vl
```
- ④接下来我们需要找到7.x的配置文件名称
```shell	
esxcli software sources profile list -d /vmfs/volumes/61ebf497-ca311138-661d-b82a72dfb37b/VMware-ESXi-7.0U3d-19482537-depot.zip
```
- 以上代码里面所用到的地址路径,依据实际情况输入
- ⑤输入以上命令获得如下结果
```text
Name                            Vendor        Acceptance Level  Creation Time        Modification Time
------------------------------  ------------  ----------------  -------------------  -----------------
ESXi-7.0U3sd-19482531-standard  VMware, Inc.  PartnerSupported  2022-03-29T00:00:00  2022-03-29T00:00:00
ESXi-7.0U3sd-19482531-no-tools  VMware, Inc.  PartnerSupported  2022-03-29T00:00:00  2022-03-11T13:53:29
ESXi-7.0U3d-19482537-standard   VMware, Inc.  PartnerSupported  2022-03-29T00:00:00  2022-03-29T00:00:00
ESXi-7.0U3d-19482537-no-tools   VMware, Inc.  PartnerSupported  2022-03-29T00:00:00  2022-03-11T15:01:02
```
- ⑥现在正式来升级
```shell
esxcli software profile update -d /vmfs/volumes/61ebf497-ca311138-661d-b82a72dfb37b/VMware-ESXi-7.0U3d-19482537-depot.zip -p ESXi-7.0U3d-19482537-standard --no-hardware-warning
```
- 等待一会,会提示重启
- ⑦升级完成后输入从起的命令
```shell
reboot
```
- ⑧重启完成后进入后台即可看到已经升级至最新版本