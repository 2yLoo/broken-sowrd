# 项目“Docker化”

在进行项目“Docker化”前，了解相关基础可能会更有帮助：
1. Docker（可参考[Docker入门](https://github.com/2yLoo/broken-sowrd/blob/master/open-sources/docker/FirstInDocker.md)）
2. Registry（镜像仓库，此处使用了[Harbor](https://github.com/2yLoo/broken-sowrd/blob/master/project-practice/docker/Harbor.md)）

接下来按工作重心可划分为3步：
1. SpringBoot + Maven：选用SpringBoot项目作为Demo，易构建；使用Maven插件配置镜像相关信息
2. Jenkins + Maven + Docker：通过Jenkins的Maven插件进行项目打包配置，并使用Maven结合Demo项目执行Docker操作
3. docker-compose：从Registry获取镜像并部署实施

## Demo项目（镜像生成前）：
为此项目搭建了一个基于SpringBoot的Demo。该Demo用于模拟实际项目，是镜像生成的出发点。

Demo对外暴露了一个接口，通过请求`localhost:8080/hello`，返回`Hi!`。

Controller代码：
```
@RestController
public class HelloController {
  @RequestMapping("/hello")
  public ResponseResult hello() {
    return "Hi!";
  }
}
```

为了使用Maven插件生成镜像，需要在pom文件中添加如下配置：
```
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>0.4.13</version>
  <executions>
    <execution>
      <phase>package</phase>
      <goals>
        <goal>build</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <serverId>harbor</serverId> <!-- 通过此ID认证Registry -->
    <imageName>your.docker-hub.com/library/demo:latest</imageName>
    <registryUrl>your.docker-hub.com</registryUrl>
    <dockerDirectory>${project.basedir}/src/main/docker</dockerDirectory>
    <resources>
      <resource>
        <targetPath>/</targetPath>
        <directory>${project.build.directory}</directory>
        <include>demo.jar</include>
      </resource>
    </resources>
  </configuration>
</plugin>
```

## Jenkins + Maven + Docker（生成镜像）
这一步有着承上启下的关键作用，使用Jenkins打包的服务器上至少需要3个软件：Jenkins、Maven以及Docker。在它们的相互作用下才能完成镜像的生成与上传。

### Jenkins
登录Jenkins后新建项目，选择 **构建一个Maven项目** 。

在Source Code Management模块配置代码信息，包括拉取的git地址、账户以及拉取的分支。

在Build模块配置执行的Maven命令：`clean package docker:build -Dmaven.test.skip=true  -DpushImage`

点击保存。

### Maven
此时不要急着构建，登录到Jenkins服务器，在Maven的配置文件`settings.xml`中，找到`<servers></servers>`标签，添加新的`<server></server>`，用于镜像推送时的用户认证：
```
<server>
        <!-- 此处ID与Demo项目POM文件中保持一致 -->
        <id>harbor</id>
        <username>username</username>
        <password>password</password>
        <configuration>
                <email>your@email.com</email>
        </configuration>
</server>
```

### Docker
现在第二步的工作已完成2/3，但还有至关重要的1/3 —— Docker。

Docker的安装可参考[官方文档](https://docs.docker.com/install/linux/docker-ce/centos/)，我的另一篇MD也提到了安装步骤（[Here](https://github.com/2yLoo/broken-sowrd/blob/master/open-sources/docker/FirstInDocker.md)）。

在此服务器中安装Docker后，需验证镜像的push是否可正常使用。

验证成功后，即可回到Jenkins网站构建项目。

#### 错误记录

##### 支持HTTP请求Docker仓库私服
Docker仓库默认仅支持HTTPS协议的请求，这也是出于保护仓库安全的考虑。

我新建的Docker第三方仓库（Harbor）暂没配置HTTPS证书，于是需要在客户端进行额外的配置才可使用HTTP请求连接仓库。

###### 解决办法1
在`/etc/docker/`目录下，如果没有`daemon.json`文件，就新建一个，如果已有，则通过vim命令添加配置，添加内容如下：
```
{
    "insecure-registries": ["your.docker-hub.com"]
}
```

执行`service restart docker`后，登录仍然报错。很可能是docker启动时没有扫描此配置文件。

###### 解决办法2
找到docker的配置文件`/etc/sysconfig/docker`，打印其原有内容是这样的：
```
# /etc/sysconfig/docker
#
# Other arguments to pass to the docker daemon process
# These will be parsed by the sysv initscript and appended
# to the arguments list passed to docker -d

other_args=""
```

我新增参数后重启Docker仍然失效，当时配置如下：
```
ADD_REGISTRY='--add-registry your.docker-hub.com'
INSECURE_REGISTRY='--insecure-registry your.docker-hub.com'

other_args=""
```

最终解决方法为：
```
other_args="--insecure-registry your.docker-hub.com"
```
此时再重启，登录成功✌️

##### 推送镜像
在登录成功后，推送镜像失败，报错大致如下：
```
The push refers to a repository [your.docker-hub.com/library/demo] (len: 1)
3c14b3d7a7e5: Image already exists
e63e1e9e4acd: Image already exists
b0223240aa19: Image already exists
6cea6d86999e: Image already exists
9b59875cc1a9: Image already exists
d11c3799fa6a: Image already exists
b819dc613ab1: Image push failed
Error pushing to registry: Server error: 413 trying to push library/demo blob - sha256:7dc326fbbb7f14743c0116125fc18c4ebdb43c17734d05c31bc86836c716b2d9
```

为了排查原因，我拉了一个名为`busybox`的镜像，并尝试将其推送到私有库中，推送成功。
> busybox是一个大小为 **1.224MB** 的官方镜像，方便做推送测试。

留意到错误信息中的`Server error: 413`，HTTP Error Code 413表示请求实体过大。

有两种解决方案：
- 提高可接受实体大小：

  在Nginx服务器上配置
  ```
  server{
        listen 80;
        listen 443 ssl;
        server_name your.docker-hub.com;
        access_log  /data/log/nginx/harbor/access.log  web;
        # 设置最大Body大小为1GB
        client_max_body_size 1024M;
        location / {
                include proxy_params.conf;
                proxy_pass  http://harbor.com;
        }
  }
  ```
  检查Nginx配置`./nginx -t`，校验通过后重启Nginx`./nginx -s reload`

- 优化镜像大小

  由于Docker中镜像是通过子镜像一层一层叠上去的，很可能存在优化空间。为了进一步了解镜像过大的原因，查看该镜像的构建历史，命令为`docker history demo`。

  单单`java:8`的镜像大小就达到了643M。



## docker-compose（使用镜像）
最后一步是运行容器，此收尾工作较前两步简单。

拉取镜像后，通过docker-compose将其运行起来。

具体安装哪个版本的docker-compose由Docker版本决定，而Docker版本由操作系统及版本决定。

如需安装高版本docker-compose可以查看[官方文档](https://docs.docker.com/compose/install/)

From 官方文档：
```
// 安装docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

// 添加用户执行权限
sudo chmod +x /usr/local/bin/docker-compose
```

低版本安装：
```
// 安装docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.4.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

// 添加用户执行权限
sudo chmod +x /usr/local/bin/docker-compose
```

安装完成后，通过`docker-compose -v`查看该命令版本。

> 如果此时提示找不到`docker-compose`命令，可能是因为该用户的执行路径下没有此命令。
>
> 上面的安装步骤中，将`docker-compose`安装到目录`/usr/local/bin`下，而root的目录在`/usr/bin`下，将其添加到相应目录中：
>
> `sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose`

接着登录到Registry中，命令为：`docker login your.docker-hub.com`，如遇HTTPS的问题，可参考前面 错误记录->支持HTTP请求Docker仓库私服。

登录成功后，即可使用`docker-compose`命令从远程仓库拉下镜像并运行。

当Docker方面已经准备就绪，其实已经可以直接使用Docker命令来运行项目了。

docker-compose提供了更加优雅的部署方式。在自定义目录下创建一个`docker-compose.yml`作为运行容器的配置文件：

```
demo:
  image: your.docker-hub.com/library/demo
  ports:
   - "8080:8080"
  environment:
    SPRING_PROFILES_ACTIVE: prod
```

运行命令：`docker-compose -f docker-compose.yml up -d`。

等待容器运行后，通过命令通过命令`docker ps`查看项目是否运行成功。

运行成功后，请求`localhost:8080/hello`，返回`Hi!`，Bravo🎉

## Kubernetes
> TODO 使用Kubernetes

## 参考
- 《第一本Docker书》
- [[Office]Docker Document](https://docs.docker.com/)
- [[Github] Maven Plugin Spotify](https://github.com/spotify/docker-maven-plugin)
