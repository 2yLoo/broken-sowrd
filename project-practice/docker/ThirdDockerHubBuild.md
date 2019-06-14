# 第三方Docker-hub搭建

## 背景

官方的Docker Hub提供了镜像存储功能，并支持Docker私有仓库。但在免费范围内，一个账号仅允许创建一个私有仓库。这无法满足我们团队的需求。

于是产生了以下几种解决方案：

1. 充钱（经费有限，pass）

2. 使用官方免费私有库，用tag区分各个项目（不易于维护，pass）

3. 搭建私有Docker-hub进行项目管理 ✅

Docker开源了Registry，搭建私有仓库并不难。但原生Registry没有可视化管理界面，我们也渴望一个类似于Docker Hub一样的界面来进行镜像的管理。

在网上查找解决方案时，找到以下两个开源项目：

- [Harbor ⭐️8k](https://github.com/goharbor/harbor)

- [Portus ⭐️2k](https://github.com/SUSE/Portus)

## Harbor搭建

Harbor在Github上有很详细的搭建流程：[安装向导](https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md)

作者推荐的[release](https://github.com/goharbor/harbor/releases)版本，release比master版本更稳定。

要求Linux宿主机环境：**docker 17.03.0-ce+ and docker-compose 1.18.0+**

先获取其压缩包：

`wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-online-installer-v1.8.0.tgz`

此源似乎需要一定手段才可以下载，如有需要可以联系我。

然后进行解压：

`tar xvf harbor-online-installer-v1.8.0.tgz`

解压后会在当前目录生成一个新文件夹`harbor`，进入此目录后，先修改其配置文件，我修改了以下3个配置：
```
# hostname改成指定ip或域名，不要用localhost 或者 127.0.0.1
# 例如我本机ip为：10.22.33.44
hostname: 10.22.33.44

# 访问的端口，默认为80端口，为避免端口冲突，可指定一个未被使用的端口号
http:
  port: 8012

# 指定伴随整个Harbor项目启动产生的各种持久化文件以及其他数据文件目录
data_volume: /data/docker/harbor

# 配置日志输出目录
# 如果是Mac环境，需要在docker中添加该目录为容器可读写目录
log:
  location: /data/log/harbor
```

配置完成后，运行当前目录下的`install.sh`脚本：

`./install.sh`

然后访问`10.22.33.44:port/harbor`即可。

如果是在本地搭建，也可访问localhost:port/harbor。

初始账户admin的密码为Harbor12345。

https://github.com/docker/docker-registry

## Portus搭建

## Harbor & Portus 比较
