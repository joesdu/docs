话不多说直接上操作指令.

- 先装好 Ubuntu 22.04 Server ,并使用 SSH 进入终端.
- **千万注意:** 千万别用预览版的 Ubuntu 安装,
- 检查系统更新,并更新至最新,并安装 MySQL Server 服务
  **先决条件:** 安装 MySQL 服务

```shell
sudo apt update
sudo apt full-upgrade -y
sudo apt install mysql-server -y
```

- 先进入 MySQL

```shell
sudo mysql -u root -p
```

- 输入密码进入后(没密码就算),执行如下命令,调整 root 账户

```sql
use mysql;
select host, user, authentication_string, plugin from user;
update user set host='%' where user='root';
flush privileges;
alter user 'root'@'%' identified with mysql_native_password by '你的密码';
flush privileges;
```

- 再次执行如下命令查看 root 账户的可访问 host 是否变成%

```sql
select host, user, authentication_string, plugin from user;
```

由于新版的 MySQL 不推荐使用 root 远程登录.所以我们创建一个新用户.

- 创建用户

```sql
use mysql;
create user 'xiaokeai'@'%' identified by '密码';
flush privileges;
```

- 其中 localhost 表示本机访问,若是需要任意 IP 访问,可以替换成 %
- 修改密码

```sql
alter user 'xiaokeai'@'%' identified by '新密码';
flush privileges;
```

- 授权

```sql
grant all privileges on *.* to 'xiaokeai'@'%' with grant option;
```

with gran option 表示该用户可给其它用户赋予权限,但不可能超过该用户已有的权限
比如 A 用户有 select,insert 权限,也可给其它用户赋权,但它不可能给其它用户赋 delete 权限,除了 select,insert 以外的都不能,这句话可加可不加,视情况而定.

all privileges 可换成 select,update,insert,delete,drop,create 等操作,如:

```sql
grant select,insert,update,delete on *.* to 'xiaokeai'@'%';
```

第一个 \* 表示通配数据库,可指定新建用户只可操作的数据库,如:

```sql
grant all privileges on 数据库.* to 'xiaokeai'@'%';
```

第二个 \* 表示通配表,可指定新建用户只可操作的数据库下的某个表,如:

```sql
grant all privileges on 数据库.指定表名 to 'xiaokeai'@'%';
```

- 查看用户授权信息

```sql
show grants for 'xiaokeai'@'%';
```

- 撤销权限

```sql
# 用户有什么权限就撤什么权限
revoke all privileges on *.* from 'xiaokeai'@'%';
```

- 删除用户

```sql
drop user 'xiaokeai'@'%';
```

- 执行后应该就已经可以了.

- **注意项**:这个时候可能使用 navicat 等工具还是无法正常链接.
  执行如下命令:

```shell
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

修改 **bind-address** 为 0.0.0.0 后保存退出.

- 重启 MySQL 服务

```shell
sudo service mysql restart
```
