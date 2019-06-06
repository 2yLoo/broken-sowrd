# 记一次MySQL服务迁移
![](http://img3.imgtn.bdimg.com/it/u=2197205783,1315817698&fm=26&gp=0.jpg)

## 背景
项目A的MySQL与MongoDB分别占用了一台服务器，且服务器资源消耗低，并且业务不准备继续扩展，为节省服务器成本，经过对两台服务器的硬件比较，决定将MySQL迁移到MongoDB所在服务器上。

由于之前缺少MySQL相关操作经验，在此记录一份迁移文档。

| 操作系统版本       | MySQL版本 |
| ------------------ | --------- |
| RedHat4.8（Linux） | MySQL 5.6 |


## MySQL安装
MySQL有3种安装方式：yum、rpm、tar。

其中rpm与tar所需的包可在官网中根据操作系统、版本进行查询。[MySQL官网-安装包路径](https://downloads.mysql.com/archives/community/)

MySQL的Server包有对其他包的依赖，最省心的安装方式还是通过yum进行安装。

官网的安装说明实际上已经很简洁易读了，我也是根据官网所述进行安装的。[MySQL官网-安装文档](https://dev.mysql.com/doc/refman/5.6/en/linux-installation-yum-repo.html)

首先在yum的仓库路径下配置MySQL的仓库地址。

在Linux终端执行命令：

`vim /etc/yum.repos.d/mysql-community.repo`

mysql-community.repo 内容如下：

```
[mysql56-community]
name=MySQL 5.6 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.6-community/el/6/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```

然后查看是否配置成功：

`yum repolist enabled | grep mysql`

安装过程中有一个小插曲，在测试环境上，该命令执行成功，但迁移到目的服务器时，yum命令使用失败，报错如下：

```
error: Failed to initialize NSS library
There was a problem importing one of the Python modules
required to run yum. The error leading to this problem was:


   cannot import name ts


Please install a package which provides this module, or
verify that the module is installed correctly.


It's possible that the above module doesn't match the
current version of Python, which is:
2.7.5 (default, Nov 20 2015, 02:00:19) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-4)]


If you cannot solve this problem yourself, please go to 
the yum faq at:
  http://yum.baseurl.org/wiki/Faq
```

看上去是Python的锅，但实际上不是这样，而FAQ地址也是一个空地址（吐槽），解决方案如下：

```
下载软件：
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/nspr-4.19.0-1.el7_5.x86_64.rpm

安装软件：
rpm2cpio nspr-4.19.0-1.el7_5.x86_64.rpm | cpio -idmv

加载更新到nspr中：
LD_PRELOAD=./usr/lib64/libnspr4.so yum update nspr --nogpgcheck
```

这一步动作是下载并更新nspr。

> 在找到此解决方案时，还在犹豫Linux系统版本的问题。于是访问父目录```http://mirror.centos.org/centos/```，希望找到对应版本（4.8）的安装包，但对应的子目录下已空，后选择Centos7下的对应版本，并通过wget下载。虽然软件包会更新，下载地址可能发生变化，但此方法思路是可靠的。

在解决完这个问题后，继续回到官网文档。

执行

`sudo yum install mysql-community-server`

在需要用户输入y/d/N时输入y并回车。此时可能会提示获取GPG秘钥失败。这在文档中是有提到的，但是很容易被忽略。

我们需要配置自己的秘钥，可参考[官方文档-配置秘钥](https://dev.mysql.com/doc/refman/5.6/en/checking-gpg-signature.html)。

先创建一个公钥认证asc文件：touch mysql_pubkey.asc。

此时有两种方式引入公钥：从官网文档将公钥直接copy过来或通过公钥ID同步进来。

为了避免Copy时一些不必要的失误，可以执行```gpg --recv-keys 5072E1F5```同步公钥。然后执行```rpm --import mysql_pubkey.asc```导入公钥。

继续尝试安装 —> 仍然失败。

执行

```
gpg --export -a 5072e1f5 > 5072e1f5.asc
rpm --import 5072e1f5.asc
// or
// rpm --import https://dev.mysql.com/doc/refman/5.6/en/checking-gpg-signature.html
```

继续尝试安装 -> 成功！

安装成功过后，启动MySQL服务：

`sudo service mysqld start`

至此MySQL的安装已经成功，接下来要做的事情就是分配操作角色以及导入原数据库。

## 账号与数据的迁移

由于此次为项目迁移，需要保留原MySQL服务的配置内容。所以在启动MySQL前，先根据需求编辑配置文件。默认生效的配置文件地址为`/etc/my.cnf`。

在命令行输入mysql后即可通过客户端访问MySQL，但此用户权限很少，我们需要更高的权限（root）来新建用户，在进入客户端时需指定用户：`mysql -uroot`。

创建用户：

`create user username@'%' identified by 'password'`

此命令创建一个用户名为username，密码为password的用户，并允许其在任意IP下访问。

创建数据库：

`create database test_db`

然后分配用户操作权限：

`grant select on test_db.* to 'username'@'%'`

接着导入数据：

`mysql -h127.0.0.1 -P3306 -uusername -ppassword < /tmp/test_db.sql`

这份数据是如何导出的？

`mysqldump -h127.0.0.1 -P3306 -uusername -ppassword --database test_db > /tmp/test_db.sql`

DONE!
