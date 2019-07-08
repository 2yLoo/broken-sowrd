# 迈入Docker的第一个脚印👣

## 前言

第一次听说Docker这门技术是在前年(2017)年底实习。当时的Docker已经很火，两年过去，Docker的热度并未下降，越来越多的公司选择了Docker。我们小组也计划今年实行项目的容器化。

这是前言。

## Docker简介

### 什么是容器？

了解Docker前，需要先知道容器的定义：**一个标准化的软件单元**

从历史发展角度看，线上环境中，越来越多的应用部署从物理机迁移到了虚拟机。

下图展示了应用运行在虚拟机上的结构：

![应用部署虚拟机](https://www.docker.com/sites/default/files/d8/2018-11/container-vm-whatcontainer_2.png)

每个虚拟机包括一个操作系统的完整副本、应用程序、必要的二进制文件和库。这会占用数十个GB大小。而虚拟机的启动速度通常也比较慢。

后来容器（Jail）的概念提了出来，用以描述一个修改过的运行时环境，以防止该程序访问收保护的资源。而后来目标也已从防止对受保护资源的访问扩展到隔离所有的资源（除非允许）。

手动创建容器的风险很高，配置易出错。**Docker解决了这个难题**，任何在Docker中运行的软件都是运行在一个容器中。

下面是应用运行在Docker上的结构：
![应用Docker化](https://www.docker.com/sites/default/files/d8/styles/large/public/2018-11/container-what-is-container.png?itok=vle7kjDj)

与前面不同的是，Docker将宿主机操作系统与实际应用隔离开，容器化的应用（也就是容器）直接运行在Docker上。多个容器可运行在同一台机器上，可与其他容器共享OS内核，每个容器都作为用户空间中的独立进程运行。容器比虚拟机占用更少的空间（几十MB大小），可以处理更多的应用程序，并需要更少的VM和操作系统。

### 什么是Docker？

**Docker一个轻量级容器管理引擎，它是操作系统级别的虚拟化。** Docker是一种强大的工具，可以帮助解决如安装、拆卸、升级、分发、信任和管理软件等常见问题。

使用Docker离不开以下几个组件：
- Docker引擎：也就是Docker的客户端与服务器，用于发送、执行Docker命令。

- Docker镜像：镜像是构建Docker世界的基石。它就像容器的“源代码”，属于Docker生命周期中的构建部分。镜像体积很小，易于分享、存储与更新。

- Registry：存放Docker镜像的仓库。

- Docker容器：Docker镜像的实例。一个容器可以根据镜像构建起来并运行，也可以停止与重启。

### Docker能做什么？
借用《第一本Docker书》的话来说，它能干事情：
- 加速本地开发和构建流程，使其高效化、轻量化
- 保证服务或应用的运行结构不受环境控制
- 创建隔离环境进行测试
- 将生产环境的复杂框架轻量的模拟到开发环境
- 构建多用户平台服务基础设施
- 提供轻量级独立沙盒环境
- 提供软件即服务应用程序
- 高性能、超大规模宿主机部署

具体来说，我用Docker干了这些事情：

- 在Mac上运行Linux环境，加深对Linux系统的了解
- 部署各种服务：Redis、MongoDB、RabbitMQ等
- 将公司项目Docker化
- 部署公司Docker私有仓库
- etc...

### 镜像与容器二三话

使用Docker时得频繁与镜像以及容器打交道。很有必要先搞清楚它们的概念与关系。

在Docker中，容器是根据相应镜像这个模板制造出来的。

想象我们在一台Linux机器上部署了一个Redis服务。我们想要运行Ubuntu的什么版本呢（抛开Docker，操作系统版本可能只能被动接受，而无法主动选择）？Redis需要什么版本呢？这些“约束”即镜像的管辖范围，它指定Ubuntu、Redis的版本。而这台Linux服务器即可理解为一个容器，我们直接通过对容器的启动、停止来控制Redis服务的启动、停止。只要生成一个镜像，就可使用它在各处运行容器。镜像本身的大小是很有限的，我们可以把镜像给存储到仓库中，方便我们重复使用，我们还可在仓库中对镜像进行更新。

## 安装Docker
使用Docker不受平台限制，Linux、MacOS以及Windows都可安装使用Docker。

### 在Ubuntu和Debian中安装Docker
```
// 更新apt源
sudo apt-get update

// 允许apt命令基于HTTPS使用仓库
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

// 添加Docker官方GPG秘钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

// 安装Docker组件
sudo apt-get install docker-ce docker-ce-cli containerd.io

// 或安装Docker引擎
sudo apt-get install docker-engine
```

### 在CentOS中安装Docker
如果该环境中存在旧版本的Docker，执行以下命令以卸载旧版本：
```
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
然后依次执行以下命令：
```
// 添加依赖包
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

// 添加Repository
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

// 国内环境可使用阿里云的Repository
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

// 安装Docker（默认最新稳定版本）
sudo yum install docker-ce docker-ce-cli containerd.io

// 启动服务
sudo systemctl start docker
```

当系统版本不高，以上步骤执行后可能出现安装失败的情况，下面是一个低版本（Docker 1.7.1）的安装步骤：
```
// 删除之前配置的Repository
sudo rm /etc/yum.repos.d/docker-ce.repo

// 添加Repository，内容在下文可见
sudo vim /etc/yum.repos.d/docker.repo

// 安装Docker引擎
sudo yum install docker-engine

// 启动服务
sudo service docker start
```

docker.repo内容：
```
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/6/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
```

### 在OS X中安装Docker
许多Mac用户都安装了Brew神器。使用Brew安装Docker是一件十分便捷的事情。

```
// 在Brew下搜索Docker
brew search docker

// 安装Docker Cask
brew cask install docker
```
然后，像安装其他软件一样，将Docker拖入Applications即可。

### 在Windows中安装Docker
1. 进入网站：https://www.docker.com/get-started

2. 右下角找到并点击 **Download Desktop and Take a Tutorial**

  - 如果没有注册过Docker，需要注册一个Docker账号，或选择已有账号登录。

3. 继续找到页面右边 **Download Docker Desktop** 弹出新界面

4. 在新界面中点击 **Download Docker Desktop for Windows**

5. 双击下载好的exe文件，安装Docker

## Docker入门
`docker --help` 是帮助快速入手Docker的利器，它会罗列许多Docker命令行工具的基本语法，具有 **权威性与可读性**。刚入门时可向其寻求大量帮助。

`docker ps --help`与`docker help ps`都可查看`docker ps`的使用帮助其他命令使用方式相似。

>我认为养成从官方获取信息的习惯对自己的提升很有帮助。

在入门阶段，以下几个命令与参数是我们经常打交道的，但不限于此：

| 命令                          | 参数         | 说明                                                                      |
| ----------------------------- | ------------ | ------------------------------------------------------------------------- |
| docker ps                     |              | 展示正在运行的容器                                                        |
|                               | -a           | 展示所有已存在容器（包括正在运行与未运行的容器）                          |
|                               | -q           | 仅展示容器的id                                                            |
|                               | -l           | 展示最近创建的容器                                                        |
|                               | -s           | 展示容器的大小                                                            |
| docker images                 |              | 展示现有的镜像（不包括中间镜像）                                          |
|                               | -a           | 展示所有镜像（包括中间镜像）                                              |
|                               | -q           | 仅展示镜像的id                                                            |
| docker pull imageName:tagName |              | 从镜像仓库拉取指定标签的指定镜像，标签省略时默认拉取latest标签的镜像      |
| docker run imageName          |              | 根据指定镜像运行容器                                                      |
|                               | -i           | 持保证容器中STDIN是开启的，持久的标准输入，是交互式容器的半边天           |
|                               | -t           | 为创建的容器分配一个伪tty终端，交互式容器的另半边天                       |
|                               | -d           | 后台运行                                                                  |
|                               | -p 8080      | Docker在宿主机上随机选择一个位于32768~61000的端口来与容器中的8080端口映射 |
|                               | -p 8080:8080 | 形成容器8080端口与宿主机端口8080之间的映射                                |
| docker start <container..>    |              | 运行指定容器，可根据id或名称指定容器，一次可传入多个id/名称               |
| docker stop <container..>     |              | 停止指定容器，规则同上                                                    |
| docker restart <container..>  |              | 重启指定容器，规则同上                                                    |
| docker rm <container..>       |              | 删除指定容器，规则同上                                                    |
| docker rmi <image..>          |              | 删除指定镜像，可根据id或名称指定镜像，一次可传入多个id/                   |
| docker attach                 |              | 将本地标准输入、输出和错误流附加到正在运行的容器中                                                                          |

### 第一个Docker容器：

运行`docker run --name first_container -it ubuntu /bin/bash`后，我们就启动了一个ubuntu系统的容器。它是依赖镜像ubuntu运行的，在此我们没有指定其镜像标签，默认使用ubuntu:latest这个镜像。如果我们本地没有该镜像，Docker则会从远程仓库将镜像下载到本地。并且这个容器名为"first_container"。我们可以在这个ubuntu系统里安装我们想要的软件，或者用于对ubuntu系统命令的学习，这都是老少咸宜的。

此时打开另一个命令行窗口，输入`docker ps`，即可查看到已运行的容器：

| CONTAINER ID         | IMAGE        | COMMAND | CREATED  | STATUS | PORTS      | NAMES    |
| -------------------- | ------------ | ------- | -------- | ------ | ---------- | -------- |
| 容器ID（具有唯一性） | 依赖的镜像名 | 命令    | 创建时间 | 状态   | 暴露的端口 | 容器名称 |

为了加深对容器这个概念的理解，我在根目录下创建了/data目录，并在/data目录创建了一个文件`temp.log`，写上`This's my first Docker container!`。此后在命令行输入`exit`退出该容器。

继续输入`docker ps`后，刚展示的容器信息消失了，说明此容器没有运行。

如果我们重新执行`docker run -it ubuntu /bin/bash`，会创建一个 **新容器** ，新容器中不会有`/data`目录以及刚我们新建的文件`temp.log`。如果我们想重新运行我们的第一个容器呢？

先输入`docker ps -a`将正在运行与未运行的容器都列出来，找到我们的初恋容器后，执行`docker start first_container`将容器运行起来，此处可通过容器名/容器id来启动容器。

容器运行后，通过`docker attach first_container`进入此容器内。我们可以在此容器中发现刚创建的文件。

上面的例子中我们使用了开源镜像 ubuntu ，官方Docker Hub有许多出色的开源镜像，也支持我们使用自定义的镜像。

### 使用Dockerfile构建镜像
构建自定义的镜像可以更充分的发挥Docker的能力。Docker镜像的构建有两种方式：
- ~使用docker commit命令（不推荐，也不做展开）~
- **使用docker build + Dockerfile（推荐）**

> 至于docker commit命令不被推荐的原因，我认为，commit需要额外的精力维护构建镜像时输入的命令，且无法保证下次构建目标镜像时输入统一性，不仅如此，如果镜像层级变复杂，commit会越来越乏力。Dockerfile更友好、更灵活、更强大👍

下面展示使用Dockerfile创建自定义镜像并运行的步骤：

在一个合适的目录创建Dockerfile文件：
```
touch Dockerfile
```

编辑该文件，输入以下内容：
```
# Docker支持以"#"号开头的注解
# 基于镜像ubuntu:14.04进行构建
FROM ubuntu:14.04
# 记录镜像的作者信息
MAINTAINER your name "your@email.com"
# 在构建时执行命令安装nginx
RUN apt-get update && apt-get install -y nginx
# 将指定语句输入到指定文件中
RUN echo 'Hi, I am in your container' >/usr/share/nginx/html/index.html
# 暴露端口80
EXPOSE 80
```
Dockerfile中的指令以从上到下的顺序执行，每条指令都必须为大写。

然后构建镜像：
```
docker build -t 2yloo/demo .
```

这一步需要一定的时间，构建完成后，验证镜像是否构建成功：
```
// 查看镜像
docker images 2yloo/demo
// 查看镜像构建历史
docker history 2yloo/demo
```

启动容器：
```
docker run -d -p 80 --name static_web 2yloo/demo nginx -g "daemon off;"
```

查看容器信息：
```
docker ps
```

在容器信息的 **PORTS** 列中，该镜像会绑定到本机的一个随机端口，当前为 `0.0.0.0:32771->80/tcp` ，那么我们访问 `localhost:32771`时，请求到的网页内容为"Hi, I am in your container"

下面展示了更多可以在Dockerfile中使用的指令：

| 指令       | 作用                                                       | 补充                                                                                                          |
| ---------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| CMD        | 指定容器启动时要运行的命令                                 | Dockerfile中只能指定一条CMD指令。如果在docker run 命令行中也指定了要执行的命令，将会覆盖Dockerfile中的CMD指令 |
| ENTRYPOINT | 与CMD类似，指定容器启动时要运行的命令                      | 相比于CMD，ENTRYPOINT中的命令不容易被覆盖                                                                     |
| WORKDIR    | 在容器内部设置一个工作目录                                 | ENTRYPOINT和/或CMD指定的程序会在这个目录下执行                                                                |
| ENV        | 在镜像构建过程中设置环境变量                               | 这些环境变量也会被持久保存到容器中                                                                            |
| USER       | 指定该镜像以什么样的用户去运行                             | 如果不通过USER指令指定用户，默认用户为root                                                                    |
| VALUME     | 向基于镜像创建的容器添加卷                                 | 卷让我们可以将数据添加（而不是提交）到镜像中，并允许多个容器共享这些内容                                      |
| ADD        | 将构建环境下的文件和目录复制到镜像中                       | 如果添加的是压缩文件（gzip、bzip2、xz），Docker会自动解压                                                     |
| COPY       | 功能与ADD相似                                              | 区别在于COPY不会做文件提取和解压的工作                                                                        |
| LABEL      | 为Docker镜像添加元数据（以键值对形式）                     | 可多次使用LABEL指令添加，也可在一条LABEL指令下添加，建议使用后者，以防创建过多镜像层                          |
| STOPSIGNAL | 设置停止容器时发送什么系统调用信号给容器                   | 该指令在Docker1.9中引入                                                                                       |
| ARG        | 用于定义可以在docker build命令运行时传递给构建运行时的变量 | 可添加多个参数                                                                                                |
| ONBUILD    | 为镜像添加触发器                                           | 在镜像层中，按照福镜像中的指令顺序执行，并且只被继承一次（只在子镜像中执行，而不会再孙子镜像中执行）          |

## 推荐阅读

- 《第一本Docker书》
  偏向对基础的总结。本书的作者是Docker公司的顾问，权威性自不用言。这本书对小白的支持十分友好。充足的案例与命令解释给枯燥的原理润色不少，我也是跟着书中的命令同步在本机实现，对本书知识点的吸收效果非常好。并且书中的大量Dockerfile都可在Github上找到，这简直是小白的福音。美中不足的是，我买的中文版还是有点表述上的疏漏，但它们往往出现在一些无关紧要的地方，不会干扰到对Docker的学习。

- 《Docker实战》
  《\*实战》系列丛书，运用更丰富的实战经验讲解了对Docker的使用，书中第一部分也对Docker进行了基础的介绍，后面重点集中在“Docker化”与服务部署编排等方向上。
