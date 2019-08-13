## Docker容器日志

### 契机
在最开始使用Docker部署业务服务时，发现磁盘空间使用率比以往更高。原以为是镜像与容器会占用部分磁盘（拉取镜像时会产生\<none\>镜像）。后发现事情没有想象中的简单，将积累了一定日质量的容器停止并删除，再新建容器后，磁盘使用量显著降低。

得出结论：容器中还有我们不熟悉的地方占用了大量磁盘空间。

### 查看容器日志
Docker允许我们使用 `docker logs <Container_Id>` 命令查看容器中的日志。
```
// 通过 --help 查看 logs使用帮助
$ docker logs --help

Usage:	docker logs [OPTIONS] CONTAINER

Fetch the logs of a container

Options:
      --details        Show extra details provided to logs
  -f, --follow         Follow log output
      --since string   Show logs since timestamp (e.g. 2013-01-02T13:23:37) or relative (e.g. 42m for 42 minutes)
      --tail string    Number of lines to show from the end of the logs (default "all")
  -t, --timestamps     Show timestamps
      --until string   Show logs before a timestamp (e.g. 2013-01-02T13:23:37) or relative (e.g. 42m for 42 minutes)
```

经常我们会使用 `docker-compose` 部署服务。使用Compose也可很方便的编排多个服务。如果我们使用Compose部署服务，可通过命令查看容器日志:
```
docker-compose -f docker-compose.yml logs -f
```

该命令会输出所有编排服务中的日志。如果只想查看指定服务的日志，可在该命令后加上服务名，例如：
```
docker-compose -f docker-compose.yml logs -f my-server
```

以上方式都是官方提供的命令，可以很方便我们使用。但容器日志的文件毕竟有它存储的地方。也就是目录 `/var/lib/docker/containers` 下。

通过命令
```
sudo find /var/lib/docker/containers/ -type f -size +200k -exec du -sh {} \;
```
可以查询到该目录下文件大小超过200k的文件，其中以 `\*-json.log` 为后缀命名的文件即容器日志。而符号 \* 处的内容为容器id。

### 设置日志驱动
日志驱动：Docker中的各种日志处理机制统称。

**Docker默认的日志驱动为 json-file** 。

#### 设置默认驱动
我们可通过命令查看Docker服务当前的默认日志驱动：
```
$ docker info --format '{{.LoggingDriver}}'"
json-file
```

官网提供了配置默认日志驱动的方式。方式是在目录 `/etc/docker/` 下配置 `daemon.json` 的 `json-file` 字段，例如：
```
{
  "log-driver": "syslog"
}
```

我们也可指定日志文件的容量上限以及数量上限，也是通过 `daemon.json` 配置，例如：
```
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
> 官网提到，max-file虽为数字，但需要加双引号""将其配置为字符串。

在 `daemon.json` 中可以配置许多服务的内容，具体可参考文档 [《Dockerd 后台配置》](https://docs.docker.com/engine/reference/commandline/dockerd/)。

但前提是Docker版本要支持通过 `daemon.json` 文件配置Docker服务内容。更具体的说， **Docker版本需大于 V1.12.6** 。
> 可通过 `docker -v` 查看Docker版本。

#### 官方支持的日志驱动
| 驱动       | 描述                                                        |
| ---------- | ----------------------------------------------------------- |
| none       | 相当于禁用容器日志，此时使用 `docker logs` 不会返回任何输出 |
| local      | 日志以自定义格式存储，这种格式的设计目的是将开销降到最低    |
| json-file  | 日志格式为JSON，这也是Docker的默认日志驱动                  |
| syslog     | 将日志输出到 syslog 中                                      |
| journald   | 将日志输出到 journald 中                                    |
| gelf       | 将日志输出到 GELF 终端，例如 Graylog 或 Logstash            |
| fluentd    | 将日志输出到 fluentd 中                                     |
| awslogs    | 将日志输出到亚马逊云日志中                                  |
| etwlogs    | 将日志输出到 ETW 事件中，仅支持Windows平台                  |
| gcplogs    | 将日志输出到谷歌云平台日志中                                |
| logentries | 将日志输出到 Rapid7 Logentries 中                                                            |

虽然有这么多的日志驱动，但在 Docker CE（社区版）中，仅支持以下驱动：
- local
- json-file
- journald

### 设置容器日志驱动
如果我们在启动容器时未指定日志驱动，则会选择默认的Docker日志驱动。

通过命令查看指定容器使用的日志驱动：
```
$ docker inspect -f '{{.HostConfig.LogConfig.Type}}' <CONTAINER>
```

Docker支持两种模式将容器日志传给日志驱动：
- 直接阻塞式传输（默认）
- 非阻塞传输，将日志存储到中间环形缓存区中，供驱动消费。在这种模式下，如果缓存区已满，新日志将覆盖未消费的旧日志。

#### 通过docker参数配置
```
// 设置日志驱动为 json-file，并且使用非阻塞模式传输，缓冲区大小为4m
$ docker run -it --log-driver json-file --log-opt mode=non-blocking --log-opt max-buffer-size=4m  alpine ash
```

#### 通过docker-compose配置
如果我们使用Compose发布服务，也可在配置文件中指定日志驱动等信息。

需要注意的是， **配置格式与Docker版本相关** 。

> [文件-版本对应关系](https://docs.docker.com/compose/compose-file/compose-versioning/)

- V3版本配置格式：
  ```
  services:
    some-service:
      image: some-service
      logging:
        driver: "json-file"
        options:
          max-size: "200k"
          max-file: "10"
  ```

- V2版本配置格式：
  ```
  services:
    some-service:
    logging:
      driver: json-file
      options:
        max-size: '12m'
        max-file: '5'
  ```

- V1版本格式：
  ```
  services:
    some-service:
    log_driver: syslog
    log_opt:
      syslog-address: "tcp://192.168.0.42:123"
  ```
  其中V1版本在官网未明确指出是否支持配置日志文件大小与数量。



### 限制容器日志
默认情况下，Docker不会处理容器日志，任由其增长。如果放任日志增长，有风险造成 **磁盘爆满** 。

我们可以从日志驱动处着手，例如配置 `max-size` 为 `100m` ，这样则不用担心日志量过大撑爆磁盘。这在Docker的 `daemon.json` 或 Compose V2+ 中配置都是十分便捷的。

但有时服务器并不支持高版本Docker与Compose。但办法总比困难多。

1. 升级服务器，使其支持更高版本的Docker以及Compose。但线上环境往往不建议这样的解决方案

2. 将日志驱动设置为none，以此禁止容器日志输出。这样的弊端是我们也无法查看到容器日志，因为日志文件不存在。

3. 使用脚本定时清除日志，类似的方案也有通过监控磁盘使用来清除日志。这两者的区别在于触发机制。缺点是需要额外对脚本进行维护。删除日志的脚本如下：
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

我在一次实践中采用了第二种方案。因为当时场景下，服务稳定性 > 部署便捷性 > 容器日志重要性。

关键点在于：我们在指定的日志目录下已分类录入了需要采集的日志，因此容器日志的重要性并不大。

### 参考文档
- [[官] 设置日志驱动](https://docs.docker.com/config/containers/logging/configure/)

- [[官] Compose V3文档](https://docs.docker.com/compose/compose-file/)

- [[官] Compose V2文档](https://docs.docker.com/compose/compose-file/compose-file-v2/)

- [[官] Compose V1文档](https://docs.docker.com/compose/compose-file/compose-file-v1/)

- [[官] Dockerd 后台配置](https://docs.docker.com/engine/reference/commandline/dockerd/)
