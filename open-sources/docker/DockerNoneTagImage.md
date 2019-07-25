## 追溯Docker `<none>:<none>` 镜像
当我们使用命令 `docker images` 或者 `docker images -a` 查看镜像时，有可能会发现这样的镜像：
```
$ docker images
REPOSITORY            TAG            IMAGE ID            CREATED             VIRTUAL SIZE
<none>                <none>         c209bdee54d2        6 days ago          691 MB
<none>                <none>         ea01b3098dc4        2 years ago         691 MB
···
```

它们没有镜像名，更没有Tag，唯一标识其身份的信息为镜像ID。

`<none>:<none>` 镜像可分为两种：**中间镜像** 与 **"dangling"镜像**。
> Dangling 有悬空、摇摆的意思，但一直没有找到贴切的中文翻译，为不作误导，后文都使用"dangling"作为称呼。

### 中间镜像
在Docker中，root文件系统永远为只读状态，并且Docker利用联合加载技术，在root文件系统层上加载更多的只读层。我们在构建镜像时，是将各个需要的镜像顺序叠加到镜像层中。位于下面的镜像称为父镜像，最底部的镜像称为基础镜像。而只有顶层为可写层。下图为一个基于Ubuntu 15.04镜像的层级结构：

![镜像构建图](https://docs.docker.com/v17.09/engine/userguide/storagedriver/images/container-layers.jpg)

Docker在构建镜像时，通过允许缓存每个步骤，可以 **提高中间镜像的可重用性、减少磁盘使用并且加快Docker构建** 。这也是为什么构建镜像时，第一次构建花费的时间会比之后的构建时间长。在命令 `docker images` 下会展示所有的顶层镜像信息，默认是不显示中间镜像的。

通过命令 `docker images -a` 可以展示所有镜像，包括顶层镜像与中间镜像:
```
$ sudo docker images -a
REPOSITORY                              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
your.docker-hub.com/demo/demo-service   test                49f3d67cebc0        3 hours ago         695.1 MB
<none>                                  <none>              158e414a230c        3 hours ago         695.1 MB
<none>                                  <none>              592d7ca0fe1d        3 hours ago         695.1 MB
<none>                                  <none>              723550e026e7        3 hours ago         669.1 MB
<none>                                  <none>              c209bdee54d2        6 days ago          691 MB
<none>                                  <none>              1eda0c3a125d        6 days ago          691 MB
<none>                                  <none>              62192bebad43        6 days ago          691 MB
<none>                                  <none>              163aff9a1d81        6 days ago          667.1 MB
your.docker-hub.com/demo/demo-service   latest              c61dbf117f27        2 weeks ago         691 MB
<none>                                  <none>              3a355665823e        2 weeks ago         691 MB
<none>                                  <none>              f67ed46f8073        2 weeks ago         691 MB
<none>                                  <none>              7ebd53b246b0        2 weeks ago         667.1 MB
<none>                                  <none>              08f6bdbf25d4        4 weeks ago         643.1 MB
<none>                                  <none>              d11c3799fa6a        2 years ago         643.1 MB
<none>                                  <none>              b819dc613ab1        2 years ago         642.7 MB
<none>                                  <none>              557ddbce174f        2 years ago         291.2 MB
<none>                                  <none>              25d2b6364de8        2 years ago         291.2 MB
<none>                                  <none>              7558653b0734        2 years ago         291.2 MB
<none>                                  <none>              5cf01c8803ea        2 years ago         291.2 MB
<none>                                  <none>              c5360aaa8537        2 years ago         291.2 MB
<none>                                  <none>              4ef6b3e5c60b        2 years ago         291.2 MB
<none>                                  <none>              b520541a1ecf        2 years ago         291.2 MB
<none>                                  <none>              3c40a2ed2543        2 years ago         291.2 MB
<none>                                  <none>              ea75de9256e4        2 years ago         289.9 MB
<none>                                  <none>              b171076e9dd9        2 years ago         167.3 MB
<none>                                  <none>              5b1f17435ed5        2 years ago         123 MB
<none>                                  <none>              a29f2b1e7978        2 years ago         123 MB
```

我们可以通过命令 `docker history c61dbf117f27` 查看上面 `demo-service:latest` 镜像的层级：
```
$ sudo docker history c61dbf117f27
IMAGE               CREATED             CREATED BY                                      SIZE                 COMMENT
c61dbf117f27        2 weeks ago         /bin/sh -c #(nop) ENTRYPOINT &{["java" "-Djav   0 B
3a355665823e        2 weeks ago         /bin/sh -c #(nop) EXPOSE 8080/tcp               0 B
f67ed46f8073        2 weeks ago         /bin/sh -c bash -c 'touch /app.jar'             23.94 MB
7ebd53b246b0        2 weeks ago         /bin/sh -c #(nop) ADD file:56602d998373c5002e   23.94 MB
08f6bdbf25d4        4 weeks ago         /bin/sh -c #(nop) VOLUME [/tmp]                 0 B
d11c3799fa6a        2 years ago         /bin/sh -c /var/lib/dpkg/info/ca-certificates   418.1 kB
b819dc613ab1        2 years ago         /bin/sh -c set -x                               && apt-get update    && apt-             351.5 MB
7558653b0734        2 years ago         /bin/sh -c #(nop)  ENV CA_CERTIFICATES_JAVA_V   0 B
557ddbce174f        2 years ago         /bin/sh -c #(nop)  ENV JAVA_DEBIAN_VERSION=8u   0 B
25d2b6364de8        2 years ago         /bin/sh -c #(nop)  ENV JAVA_VERSION=8u111       0 B
c5360aaa8537        2 years ago         /bin/sh -c #(nop)  ENV JAVA_HOME=/usr/lib/jvm   0 B
5cf01c8803ea        2 years ago         /bin/sh -c {                                                         echo '#!/bin/sh';                        echo 'set           87 B
b520541a1ecf        2 years ago         /bin/sh -c #(nop)  ENV LANG=C.UTF-8             0 B
4ef6b3e5c60b        2 years ago         /bin/sh -c echo 'deb http://deb.debian.org/de   55 B
3c40a2ed2543        2 years ago         /bin/sh -c apt-get update && apt-get install    1.284 MB
ea75de9256e4        2 years ago         /bin/sh -c apt-get update && apt-get install    122.6 MB
b171076e9dd9        2 years ago         /bin/sh -c apt-get update && apt-get install    44.29 MB
5b1f17435ed5        2 years ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0 B
a29f2b1e7978        2 years ago         /bin/sh -c #(nop) ADD file:89ecb642d662ee7edb   123 MB
```
可以发现构建该镜像时使用到的中间镜像。

### "Dangling"镜像
中间镜像存在的价值是很高的，发挥它的功能也并不依赖于镜像名与Tag，是我们平时使用时不太需要关注的一类 `<none>:<none>` 镜像。

而"dangling"镜像则有些不同。在Docker中，"dangling"镜像是未使用的（也许曾经使用过），并且没有任何镜像引用它们。这样子的 `<none>:<none>` 镜像在输入命令 `docker images` 时即可看到。
> 有时也会产生 `<image>:<none>` 的无标签镜像，它们也是"dangling"镜像。

"dangling镜像"是在我们使用 `docker build` 或者 `pull` 命令时产生的。

当本地已有 `demo:latest` 镜像（镜像A）时，如果通过构建或者拉取的方式继续获取 `demo:latest` 镜像，本地会出现新的 `demo:latest` 镜像（镜像B），而之前已存在的镜像A则会变成"dangling"镜像。

因为 镜像名+Tag 可以锁定一个镜像，所以当重名重标签镜像出现时，旧镜像将会被“闲置”为"dangling"镜像。

我们可以通过命令 `docker images -f "dangling=true"` 查看"dangling"镜像。

#### 重新使用"dangling"镜像
"dangling"镜像可用于历史版本的镜像储备，但作为储备，它提供的信息似乎又太少了点。

如果服务器中部署的Docker容器不止一个，有很多不同的镜像都有机会产生"dangling"镜像，仅通过镜像ID是无法直观获取其镜像含义的。

即使我们在服务器中只部署了一个Docker容器，通过镜像创建时间来确定该镜像的历史版本对应关系也有些牵强。

不过Docker还是给了我们重新使用它们的权力。

通过命令 `docker tag image_id name:tag` 可以给指定镜像赋值name与tag。

此时再查看"dangling"镜像时，刚被赋予了名字与tag的镜像不会出现在该列表中。

#### 不友好的处理"dangling"镜像
我是在使用 `docker-compose` 的过程中注意到"dangling"镜像的。 `docker-compose pull` 只负责拉取相关镜像，而不会处理旧镜像，这样也就产生了"dangling"镜像了。

如何才能优雅的处理"dangling"镜像呢？

- 向官方求助失败：

  ```
  $ docker-compose pull --help
  Pulls images for services defined in a Compose file, but does not start the containers.

  Usage: pull [options] [SERVICE...]

  Options:
      --ignore-pull-failures  Pull what it can and ignores images with pull failures.
      --parallel              Deprecated, pull multiple images in parallel (enabled by default).
      --no-parallel           Disable parallel pulling.
      -q, --quiet             Pull without printing progress information
      --include-deps          Also pull services declared as dependencies
  ```
  该命令并没有参数可以处理旧镜像。

- 向官方求助失败 x2：
  ```
  $ docker-compose --help
  ···
  Commands:
  build              Build or rebuild services
  bundle             Generate a Docker bundle from the Compose file
  config             Validate and view the Compose file
  create             Create services
  down               Stop and remove containers, networks, images, and volumes
  events             Receive real time events from containers
  exec               Execute a command in a running container
  help               Get help on a command
  images             List images
  kill               Kill containers
  logs               View output from containers
  pause              Pause services
  port               Print the public port for a port binding
  ps                 List containers
  pull               Pull service images
  push               Push service images
  restart            Restart services
  rm                 Remove stopped containers
  run                Run a one-off command
  scale              Set number of containers for a service
  start              Start services
  stop               Stop services
  top                Display the running processes
  unpause            Unpause services
  up                 Create and start containers
  version            Show the Docker-Compose version information
  ```
  当前版本下也没有指令可以处理旧镜像。

  并且在[Docker Compose官网](https://docs.docker.com/compose/)中也没有说明如何处理旧镜像。

- 向网友求助成功（不推荐）：

  寄希望于 `docker-compose` 无果。在网上查到以下解决方案：
  ```
  docker rmi $(docker images | grep "none" | awk '{print $3}')
  ```
  这样做是行得通的，先通过命令 `docker images | grep "none" | awk '{print $3}'` 将镜像列表中存在none的行过滤出来，再拿到第三列（镜像ID），最后使用命令 `docker rmi` 删除。

  但如果镜像名或tag中本来就存在字符串none呢，那样会造成误删。这样的解决方式还是有点硬核。

- 向网友求助成功（不推荐）：

  在 **Start Overflow** 中，也有许多人有相似疑问。可使用以下命令 **精确** 删除"dangling"镜像：
  ```
  docker rmi $(docker images -f "dangling=true" -q)
  ```
  先查询镜像中的"dangling"镜像，再通过参数 -q 获取它们的id，最后使用 `docker rmi` 删除。

  这种方式比上一种更优雅一点，至少不会误删，我也以为这就是最优解了，后来还是在官网上找到了宝藏💎。

  👇

### 清理Docker无用对象

Docker官方文档中有一篇文档针对清理无用的Docker对象：[Prune unused Docker objects](https://docs.docker.com/config/pruning/)

其中也包括了处理中间镜像与"dangling"镜像。

官文上说，Docker使用一种保守的策略来清理未使用对象（也称为垃圾收集）。这些对象包括镜像、容器、挂载以及网络。

这些无用的Docker对象会使Docker使用额外的磁盘空间，所以合理的进行断舍离是OK的！

#### 清理镜像：

- 默认清理所有"dangling"镜像
  ```
  docker image prune
  ```

- 清理所有无用镜像，包括 `<none>:<none>` 的中间镜像
  ```
  docker images prune -a
  ```

- 清理1天前的镜像
  ```
  docker image prune -a --filter "until=24h"
  ```

#### 清理容器：
- 默认清理所有已停止容器
  ```
  docker container prune
  ```

- 清理1天前的已停止容器
  ```
  docker container prune --filter "until=24h"
  ```

#### 清理挂载：
- 清理没有任何容器使用的挂载
  ```
  docker volume prune
  ```

- 清理没有贴"keep"标签的挂载
  ```
  docker volume prune --filter "label!=keep"
  ```

#### 清理网络：
- 清理没有任何容器使用的网络
  ```
  docker network prune
  ```

- 清理1天前的无用网络
  ```
  docker network prune --filter "until=24h"
  ```

#### 清理以上所有类型的垃圾：
除了针对性的清理镜像、容器、挂载和网络对象，还可通过命令一次性清理：
```
docker system prune
```

该命令相当于一次性执行：
```
docker image prune
docker container prune
docker volume prune
docker network prune
```

### 参考文档

- [[官] Docker 镜像介绍](https://docs.docker.com/engine/reference/commandline/images/)

- [[官] 镜像、容器与存储引擎介绍](https://docs.docker.com/v17.09/engine/userguide/storagedriver/imagesandcontainers/)

- [[官] 清理未使用的Docker对象](https://docs.docker.com/config/pruning/)

- [[个] What are Docker `<none>:<none>` images?](http://www.projectatomic.io/blog/2015/07/what-are-docker-none-none-images/)
