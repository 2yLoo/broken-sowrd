## Docker撑爆系统盘①

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

## Docker `<none>:<none>` 镜像
在使用 docker-compose 命令构建服务时，经常要用到命令 `docker-compose pull` 。该命令会从远端拉取指定的镜像，并覆盖本地镜像。常在服务更新迭代时使用。如果我们在更新镜像时，总是选择一个tag（例如latest），那么旧镜像不会被移除，而是以另一种方式存在在服务器上：
```
$ docker images
REPOSITORY                              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
your.docker-hub.com/demo/demo-service   test                49f3d67cebc0        2 hours ago         695.1 MB
<none>                                  <none>              c209bdee54d2        6 days ago          691 MB
your.docker-hub.com/demo/demo-service   latest              c61dbf117f27        2 weeks ago         691 MB
```

像上面留下来的镜像 `c209bdee54d2` ，属于在构建镜像时，旧镜像被替换后变成的"dangling"镜像。
> PS: dangling暂未找到合适的翻译

在 `docker-compose --help` 以及[Compose官方文档](https://docs.docker.com/compose/)中，都没有指令与参数可以在pull时删除"dangling"镜像，或者通过简单指令删除"dangling"镜像。硬核删除方式如下：
```
docker rmi image_id1 image_id2 ...
```
如果不及时维护，久而久之，服务器上的"dangling"镜像越来越多，令人抓狂。

通过嵌套指令，可以达到删除"dangling"的`<none>:<none>`镜像目的：
```
docker rmi $(docker images -f "dangling=true" -q)
```

这样可以将"dangling"镜像删除，而不会删除同样为`<none>:<none>`的中间镜像（有益，用于构建更复杂镜像）。

## Docker撑爆系统盘②
当我们未将docker目录挂载到/data下时，可能造成磁盘爆满的灾难。而Docker中还有另一种日志的隐患，可能塞满磁盘。

#### 问题发现
将业务服务部署到Docker后，业务服务的日志记录量并没有改动，请求量也与往常相似，但通过命令 `df -h` 查看磁盘使用情况时，发现磁盘使用情况比以往要高。服务器并无其他服务，唯一变量即项目Docker化。后使用docker-compose停止并删除容器，再重新部署时，磁盘使用量有所下降。因此怀疑容器中有大量占空间日志。

当对文件进行全局搜索时，发现在/data/docker/目录下有一个文件大小已达到70+G：
```
sudo find /  -type f -size +100000000c -exec du -sh {} \;
```

其名称格式为 **hashcode-json.log** 。将其删除后磁盘使用量恢复正常。

#### 解决
针对该日志文件，我们无法保证充沛的精力定时去处理它。面对这样的问题，要分两种情况处理：
##### 高版本Docker(V1.12+)
在Docker V1.12版本后，用户可以创建 **daemon.json** 文件对Docker Engine进行配置。以下为解决步骤：
1. 创建或在已有 daemon.json 文件上添加一下内容：
```
{
    "log-opts": {
        "max-size": "1G"
    }
}
```

2. 重启服务
```
sudo service docker restart
```

3. 重启容器
```
略
```

除了全局配置，也可在启动时加上参数：
```
docker run -d --log-opt max-size=1g nginx
```

如果我们使用 `docker-compose` 启动服务，则可在 `docker-compose.yml` 中进行以下配置：
```
nginx:
  image: nginx:1.12.1
  restart: always
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
```
> **前提是docker-compose版本为2+**

##### 低版本Docker
目前在低版本中未从官方渠道获取到可靠的解决方案。

以下解决方案偏硬核一点：
1. 编写脚本 `del-docker-log.sh`：
```
#!/bin/sh
echo "======== start clean docker containers logs ========"
logs=$(sudo find /var/lib/docker/containers/ -name *-json.log)
for log in $logs
        do
                echo "clean logs : $log"
                echo "sudo cat /dev/null > $log" | sudo sh
        done
echo "======== end clean docker containers logs ========"
```
在线上服务器中我们可能并没有root权限，而当我们使用 `sudo service docker start` 启动Docker时，容器日志文件仅有root有权访问，我们可以使用 `echo "sudo cat /dev/null > $log" | sudo sh` 以sudo权限清理日志。
2. 通过定时任务执行该脚本
```
// 每天零点清理容器日志
0 0 * * * /data/script/del-docker-log.sh > /dev/null 2>&1
```
