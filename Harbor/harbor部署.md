## Ubuntu 系统 Harbor 部署

> Harbor 是 VMware 公司开源的企业级 DockerRegistry 项目,其目标是帮助用户迅速搭建一个企业级的 Dockerregistry 服务.它以 Docker 公司开源的 registry 为基础,提供了管理 UI,基于角色的访问控制(Role Based Access Control),AD/LDAP 集成、以及审计日志(Auditlogging) 等企业用户需求的功能,同时还原生支持中文.

本文属于写欠下的内容了,[Harbor](https://goharbor.io)已经部署了很久了,答应朋友写的文章却一直忘记写了.所以这里先补上来.本文包含如下内容.

- harbor 的基本部署.
- https 证书的配置(由于本人弄了 https 证书).
- Nginx 反向代理的配置.
- 域名的设置以及 Docker Desktop 登录和推送镜像.

#### Harbor 部署

- 首先我们在 VPS 或者 ESXi 中安装好操作系统.并安装好 docker 引擎和 docker-compose.
- docker-compose 需要安装新版的 V2.
- 至于 [Docker Engine](https://docs.docker.com/engine/install/ubuntu) 和 [Docker Compose](https://docs.docker.com/compose/install/linux) 的配置可以参考 docker 官网,这里不再赘述.参考链接如下:
- Docker Engine: https://docs.docker.com/engine/install/ubuntu
- Docker Compose: https://docs.docker.com/compose/install/linux
- harbor 我们这里选择最新版本,因为 2.5.2 之后的版本才支持 docker-compose v2(没记错的话.)写这篇文章的时候发现 harbor 都已经 2.7.0 了.不过这个都不影响我们使用.
- 首先在官网点击[Download Now](https://github.com/goharbor/harbor/releases)会跳转到 GitHub 的 Release,选择自己喜欢的版本下载就行.本文以 2.7.0 版本为例.
- https://github.com/goharbor/harbor/releases

```shell
sudo wget https://github.com/goharbor/harbor/releases/download/v2.7.0/harbor-offline-installer-v2.7.0.tgz
```

- GitHub 的安装包分为**在线安装**和**离线安装**两种,可以根据自己的网络情况选择适合自己的包即可.
- 首先解压下载的包

```shell
tar xzvf harbor-*.tgz ./
```

- 解压后当前命令符目录下应该会多出来一个 harbor 文件夹.执行 cd harbor 进入文件夹内.
- 复制配置文件到 harbor.yml,我这里是 root 目录,需要根据自己的实际情况设置.

```shell
sudo cp /root/harbor.yml.tmpl /root/harbor/harbor.yml
```

- 接着打开 harbor.yml 文件.

```shell
sudo nano /root/harbor/harbor.yml
```

- 调整如下内容.

```yaml
# hostname直接填写为本机的主机名就行,如我这里就叫harbor
hostname: harbor
# http我这里没有使用,所以注释掉了.
#http:
#  port: 80
# https配置
https:
  # https port for harbor, default is 443
  port: 443
  # 这里写入证书的路径,需要先将域名服务商颁发的证书复制到对应目录,若是没有证书,可以仅使用http.
  certificate: /data/cert/harbor.altzyxy.com.crt
  private_key: /data/cert/harbor.altzyxy.com.key
#这个url写自己在域名注册商填写的域名即可,我这里用的三级域名.
external_url: https://harbor.altzyxy.com
# 这里写入自己希望文件等配置的路径,如我的证书就存在/data/cert下.
data_volume: /data
```

- 别的配置非常建议不要修改默认值.亲身经历,改过默认密码后,无论如何都进不去.密码等部署成功后,进入系统再改.
- 配置调整好后,就可以执行命令安装了.

```shell
# 后边的--with-trivy是添加了一个扫描器,用来扫描镜像漏洞的.若是不需要可以不加.也可以后期再执行这个命令添加上去.
sudo ./install.sh --with-trivy
```

- 等待一段时间,就可以部署成功了.

---

#### Nginx 反向代理的配置

- 首先我们添加 nginx 的配置.

```shell
# 这里的文件名可以根据自己的情况随便起.
sudo nano /etc/nginx/conf.d/harbor.conf
```

- 在打开的文件中写入如下内容

```conf
server{
    # 这里填入你域名服务商的域名
    server_name harbor.altzyxy.com;

    #listen 80;
    # 开启443端口监听,我这里同时添加了http2支持,若是没有证书,可仅使用http
    listen 443 ssl http2;
    #ssl on;
    ssl_certficate conf.d/crts/harbor.altzyxy.com.crt;
    ssl_certificate_key conf.d/crts/harbor.altzyxy.com.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #按照这个协议配置
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;#按照这个套件配置

    location / {
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-NginX-Proxy true;
        # 这里填入你自己的内网IP若是是本机直接填127.0.0.1即可,我这里是两台机器,所以填了内网IP
        proxy_pass https://192.168.1.2:443$request_uri;
        proxy_redirect off;
    }
}
```

- 按 Ctrl+O 保存,再按 Ctrl+X 退出编辑
- 同时调整 nginx 的一些默认配置,不然可能可能无法上传大的镜像

```shell
sudo nano /etc/nginx/nginx.conf
```

- 将 http 中的 client_max_body_size 调整为 1024m 或者更大.如: client_max_body_size 1024m;这表示客户端上传的内容最大为 1G,若是超过 1G 就会上传失败,所以这里需要根据自己的实际情况来调整.
- 完事后,按 Ctrl+O 保存,再按 Ctrl+X 退出编辑
- 接着重新加载 nginx 配置.

```shell
sudo nginx -s reload
```

- 这个时候我们就可以通过域名访问 harbor 的后台服务了.
- 输入默认用户名: admin 默认密码: Harbor12345 登录后台后就可以进行配置与改密码.
- 最后在需要推送镜像和拉取镜像的机器上执行登录即可.也可以创建公共镜像,这样就无需登录也可以拉取.

```shell
# 语法: docker login [OPTIONS] [SERVER]
# 请将域名替换为自己的.
docker login -u username harbor.altzyxy.com
# 按下回车键后,等待一下会提示输入密码,输入正确的修改后的harbor该用户的密码后即可登录成功.
```

- 然后就是 enjoy.
- docker 退出仓库命令.
- docker logout : 登出一个 Docker 镜像仓库,如果未指定镜像仓库地址,默认为官方仓库 Docker Hub

```shell
#语法: docker logout [SERVER]
docker logout harbor.altzyxy.com
```
