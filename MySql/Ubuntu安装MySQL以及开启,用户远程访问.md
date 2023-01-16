话不多说直接上操作指令.

- 先装好 Ubuntu 20.04 Server ,并使用 SSH 进入终端.
- **千万注意:**千万别用预览版的 Ubuntu 安装,
- 检查系统更新,并更新至最新,并安装 MySQL Server 服务
  **先决条件:**安装 MySQL 服务

```shell
sudo apt update
sudo apt full-upgrade -y
sudo apt install mysql-server -y
```

- 安装完成后,保护加固 MySQL

```shell
sudo mysql_secure_installation
```

你将会被要求配置 VALIDATE PASSWORD PLUGIN 它被用来测试 MySQL 用户密码的强度,并且提高安全性:

```
Securing the MySQL server deployment.
Connecting to MySQL using a blank password.

VALIDATE PASSWORD COMPONENT can be used to test passwords
and improve security. It checks the strength of password
and allows the users to set only those passwords which are
secure enough. Would you like to setup VALIDATE PASSWORD component?

Press y|Y for Yes, any other key for No: y
```

有三个级别的密码验证策略,低级,中级,高级.如果你想设置验证密码插件,按**y**或者其他任何按键,移动到下一个步骤:

```
There are three levels of password validation policy:

LOW    Length >= 8
MEDIUM Length >= 8, numeric, mixed case, and special characters
STRONG Length >= 8, numeric, mixed case, special characters and dictionary file

Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 2
```

下一次被提示时,你将被要求为 MySQL root 用户设置一个密码:

```
Please set the password for root here.

New password:

Re-enter new password:
```

如果你设置了验证密码插件,这个脚本将会显示你的新密码强度.输入**y**确认密码:

```
Estimated strength of the password: 50
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) : y
```

下一步,你将被要求移除任何匿名用户,限制 root 用户访问本地机器,移除测试数据库并且重新加载权限表.你应该对所有的问题回答**y**

---

##### 方案一:

- 完成后,进入 mysql shell.

```shell
sudo mysql
```

由于新版的 MySQL 不推荐使用 root 远程登录.所以我们创建一个新用户.

- 创建用户

```sql
use mysql;
create user 'root'@'%' identified by '密码';
flush privileges;
```

- 其中 localhost 表示本机访问,若是需要任意 IP 访问,可以替换成 %
- 修改密码

```sql
alter user 'root'@'%' identified by '新密码';
flush privileges;
```

- 授权

```sql
grant all privileges on *.* to 'root'@'%' with grant option;
```

with gran option 表示该用户可给其它用户赋予权限,但不可能超过该用户已有的权限
比如 A 用户有 select,insert 权限,也可给其它用户赋权,但它不可能给其它用户赋 delete 权限,除了 select,insert 以外的都不能,这句话可加可不加,视情况而定.

all privileges 可换成 select,update,insert,delete,drop,create 等操作,如:

```sql
grant select,insert,update,delete on *.* to 'root'@'%';
```

第一个\*表示通配数据库,可指定新建用户只可操作的数据库,如:

```sql
grant all privileges on 数据库.* to 'root'@'%';
```

第二个\*表示通配表,可指定新建用户只可操作的数据库下的某个表,如:

```sql
grant all privileges on 数据库.指定表名 to 'root'@'%';
```

- 查看用户授权信息

```sql
show grants for 'root'@'%';
```

- 撤销权限

```sql
# 用户有什么权限就撤什么权限
revoke all privileges on *.* from 'root'@'%';
```

- 删除用户

```sql
drop user 'root'@'%';
```

---

##### 方案二:

- 先进入 MySQL

```shell
sudo mysql -u root -p
```

- 输入密码进入后,执行如下命令,调整 root 账户

```sql
use mysql;
select host, user, authentication_string, plugin from user;
update user set host='%' where user='root';
alter user 'root'@'%' identified with mysql_native_password by '你的密码';
flush privileges;
```

- 再次执行如下命令查看 root 账户的可访问 host 是否变成%

```sql
select host, user, authentication_string, plugin from user;
```

- 执行后应该就已经可以了.

- **注意项**:这个时候可能使用 navicat 等工具还是无法正常链接.
  执行如下命令:

```shell
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

修改**bind-address**为 0.0.0.0 后保存退出.

- 重启 MySQL 服务

```shell
sudo service mysql restart
```
