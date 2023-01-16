## Ubuntu安装PostgreSql

- 目前PgSql仅支持如下版本的Ubuntu系统,22.04还未正式支持.所以我服务器准备的是20.04 LTS
- The PostgreSQL Apt Repository supports the current versions of Ubuntu:
  - impish (21.10)
  - hirsute (21.04)
  - focal (20.04)
  - bionic (18.04)
- on the following architectures:
  - amd64
  - arm64 (18.04 and newer; LTS releases only)
  - i386 (18.04 and older)
  - ppc64el (LTS releases only)

```shell
# Create the file repository configuration:
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update the package lists:
sudo apt-get update

# Install the latest version of PostgreSQL.
# If you want a specific version, use 'postgresql-12' or similar instead of 'postgresql':
sudo apt-get -y install postgresql
```
- 执行完上述命令后,将安装成功.此时我们可以使用如下命令进行一些配置.
- 首先我们希望远程管理数据库,所以先修改一下配置文件,调整一些参数
- 先调整 postgresql.conf 文件
```shell
sudo nano /etc/postgresql/14/main/postgresql.conf
```
- 将 listen_addresses 项取消注释,并将值调整为 '*' ,如下:
```conf
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'                  # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = 5432                             # (change requires restart)
```
- 然后调整 pg_hba.conf 文件
```shell
sudo nano /etc/postgresql/14/main/pg_hba.conf
```
- 在文件后追加 0.0.0.0/0 的配置项,如下:
```conf
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
host    all             all             0.0.0.0/0               scram-sha-256
```
- 添加好后,使用命令重启数据库
```shell
sudo service postgresql restart
```
- 等待重启成功后,我们就需要进行下一步操作了.
- PgSql 在安装成功后会给我们的操作系统创建一个新用户(postgres),我们得切换到那个用户才能进一步操作数据库.
```shell
sudo -i -u postgres
```
- 进入后我们使用psql命令进入数据库.同时会输出数据库版本信息.
```shell
psql
```
- 这时相当于系统用户postgres以同名数据库用户的身份,登录数据库,这是不用输入密码的.如果一切正常,系统提示符会变为"postgres=#",表示这时已经进入了数据库控制台.以下的命令都在控制台内完成
- 然后我们输入如下命令,调整postgres用户的密码.
```shell
\password postgres
```
- 会提示我们输入两次密码,成功后即可通过账户密码进行远程登录了.
- 使用 \q 退出
```shell
\q
```

---
- 这里我们是将postgre用户当作超级用户使用,但是实际使用过程中并不建议使用该账户来进行所有操作.

- 首先,新建一个Linux新用户,可以取你想要的名字,这里为dbuser
```shell
sudo adduser dbuser
```
- 切换到 postgres 用户
```shell
sudo -i -u postgres
```
- 使用psql命令登录PostgreSQL控制台
```shell
psql
```
- 创建数据库用户 dbuser,并设置密码
```sql
CREATE USER dbuser WITH PASSWORD 'password';
```
- 创建用户数据库,这里为 testdb,并指定所有者为dbuser
```sql
CREATE DATABASE testdb OWNER dbuser;
```
- 将 testdb 数据库的所有权限都赋予 dbuser,否则dbuser只能登录控制台,没有任何数据库操作权限
```sql
GRANT ALL PRIVILEGES ON DATABASE testdb to dbuser;
```
- 使用 \q 命令退出控制台(也可以直接按ctrl+D)
```bash
\q
```

- [参考文章](https://www.ruanyifeng.com/blog/2013/12/getting_started_with_postgresql.html)