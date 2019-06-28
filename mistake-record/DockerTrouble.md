## Docker撑爆系统盘

#### 细节
无法`kinit`以及切换`work`权限，大致异常如下：
```
Credentials cache I/O operation failed XXX
```

后网上查询得知，是磁盘已满造成的原因。

通过命令`df -h`查看服务器磁盘使用情况，发现系统盘磁盘已满。

造成磁盘满的原因是 **未建立Docker目录软链接，Docker容器在系统盘下运行，业务日志塞满磁盘**


#### 问题发现
通过yum安装Docker后，运行Docker时，Docker的镜像与容器都默认保存在目录`/var/lib/docker`中，随着容器运行，日志文件逐步增大，撑爆系统盘，从而导致使用`kinit`认证用户时，无更多空间存储用户信息。

#### 解决
后请运维同事帮忙强制删除目录`/var/lib/docker`后，kinit正常。

为了不影响正式环境，在切换`work`后马上用原方式将项目运行起来（java）。但运行后无任何请求日志输出。

说明项目虽然运行正常，但并未接收任何请求。

登录Nginx服务器查看配置，配置正常。

由于是在Docker运行时强制删除Docker目录，把排查思路聚焦到Docker上。

回到业务服务器，通过`ps -ef | grep java`查看到两个进程：刚运行的Java项目以及该项目的Docker容器。

使用`docker ps`查看已运行容器，提示未运行Docker。

通过`ps -ef |grep docker`查看是否有Docker后台进程，未发现。

接着使用`kill -9 xxxx`强制杀掉该Docker容器进程。

此时监控日志仍无新日志输出。

后新建软链接`sudo ln -s /data/docker /var/lib/docker`，并重启Docker服务`sudo service docker restart`，日志正常输出。
