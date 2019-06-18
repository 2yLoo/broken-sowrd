# 实战第三方Docker-hub——Harbor

## 背景

官方的Docker Hub提供了镜像存储功能，并支持Docker私有仓库。但在免费范围内，一个账号仅允许创建一个私有仓库。这无法满足我们团队的需求。

于是产生了以下几种解决方案：

1. 充钱（经费有限，pass）

2. 使用官方免费私有库，用tag区分各个项目（不易于维护，pass）

3. 搭建私有Docker-hub进行项目管理 ✅

Docker开源了Registry，搭建私有仓库并不难。但原生Registry没有可视化管理界面，我们也渴望一个类似于Docker Hub一样的界面来进行镜像的管理。

在网上查找解决方案时，在Github上找到以下两个高收藏的开源项目：

- [Harbor 8k⭐️](https://github.com/goharbor/harbor)

- [Portus 2k⭐️](https://github.com/SUSE/Portus)

Harbor的入口更清晰，安装也很简单，经过尝试后，它的功能已经足够满足我们的需求。

## Harbor简介&安装

### Harbor简介
Harbor是一个Github开源的第三方Docker-hub，使用Go语言开发。有以下特性与功能：

- 云注册中心
- 基于角色的访问控制
- 多注册中心备份
- 漏洞扫描
- 支持LDAP/AD
- 支持OIDC
- 镜像删除 & 垃圾回收
- 权限认证
- 图形界面
- 操作审查
- RESTful API
- 部署简单

### Harbor安装
Harbor在Github上有很详细的搭建流程：[安装向导](https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md)

作者推荐使用[release](https://github.com/goharbor/harbor/releases)版本，release比master版本更稳定。

要求Linux宿主机环境：**docker 17.03.0-ce+ and docker-compose 1.18.0+**

先获取其压缩包：

`wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-online-installer-v1.8.0.tgz`

> PS:此源似乎需借用梯子获取。

然后进行解压：

`tar xvf harbor-online-installer-v1.8.0.tgz`

解压后会在当前目录生成一个新文件夹`harbor`，进入此目录后，先修改其配置文件，我修改了以下3个配置：
```
# hostname改成指定ip或域名，不要用localhost 或者 127.0.0.1
# 例如我本机ip为：10.22.33.44
hostname: 10.22.33.44

# 访问的端口，默认为80端口，为避免端口冲突，可指定一个未被使用的端口号
http:
  port: 9012

# 指定伴随整个Harbor项目启动产生的各种持久化文件以及其他数据文件目录
data_volume: /data/docker/harbor

# 配置日志输出目录
# 如果是Mac环境，需要在docker中添加该目录为容器可读写目录
log:
  location: /data/log/harbor
```

配置完成后，运行当前目录下的`install.sh`脚本：

`./install.sh`

耐心等待Harbor初始化：
```
✔ ----Harbor has been installed and started successfully.----
Now you should be able to visit the admin portal at http://10.22.33.44.
For more details, please visit https://github.com/goharbor/harbor .
```

然后访问`10.22.33.44:9012/harbor`即可。

如果是在本地搭建，也可访问`localhost:port/harbor`。

在Harbor中，任何新增用户的默认初始密码都相同，具体内容可在配置文件中修改，默认为Harbor12345。

admin是该系统的第一名用户，也是最高级系统管理员，拥有整个系统权限。

## Harbor使用

### 后台结构
![后台结构](http://assets.processon.com/chart_image/5d087b65e4b024123ddead45.png?_=1560837334910)

### 权限说明
![权限说明](https://github.com/goharbor/harbor/raw/master/docs/img/rbac.png)

Harbor后台系统的账号分为两类：系统管理员/非系统管理员

非系统管理员登录后只可看到公共项目与有权私有项目，以及日志信息。

而系统管理员拥有系统的操作管理权限，可以对用户、项目等进行管理。

除了后台系统的角色区分，每个项目有4种角色：

| 角色       | 镜像拉取权限 | 镜像推送权限 | 标签管理权限 | 用户管理权限 |
| ---------- | ------------ | ------------ | ------------ | ------------ |
| 访客       | 有           | 无           | 无           | 无           |
| 开发人员   | 有           | 有           | 无           | 无           |
| 维护人员   | 有           | 有           | 有           | 无           |
| 项目管理员 | 有           | 有           | 有           | 有           |

对普通用户来说，项目管理员在项目中的权限最高，但 **系统管理员拥有所有项目的项目管理员权限** 。也就是说，即使私有项目的项目管理员不给系统管理员分配任何角色，系统管理员仍然可以对项目进行管理操作。

### 小魔法
Harbor后台系统支持多语言，可在系统右上角切换，汉化做的已经十分到位了。操作界面也很友好，下面是一点可能被忽视的管理操作：

- 注册入口

  默认情况下，新增用户有两种途经：登录首页注册或通过管理员添加。如果搭建的Harbor仅用于公司/团队内部使用，建议关闭注册入口，由系统管理员新增用户。

- 接入注册中心

  如果想接入其他仓库注册中心，需由系统管理员在仓库管理中新建目标仓库注册中心。

- 同步注册中心

  如果需要同步各注册中心镜像，可在同步管理中配置同步规则。

- 机器账号

  在项目管理中，项目管理员可配置机器人账户。这样做是有好处的。如果使用Maven插件对一个Java项目进行打包，需要填写用户、密码信息，为了保护Harbor系统安全以及个人账户安全，可在项目中配置机器人账户，并将机器人账户的账号密码配置到项目中。

- 标签管理

  汉化有一个小坑：管理员可以在系统管理-配置管理-标签中添加新标签，这些标签可以运用在具体镜像上。我们在 `demo:1.0.0` 这个镜像上再贴一个标签 `best` ，这么做是没有问题的。但是，我们不可以用 `best` 来拉取此镜像。仔细查看后台，这两个标签所在的位置不同， `1.0.0` 在最左边，有一丝"id"的意味， `best` 在最右边，更像是对这个镜像的额外说明，并且不同镜像可贴相同标签，同一个镜像可贴多个标签。

  将简体中文系统切换成English后，发现第一个标签的英文是Tag，而后面那个标签的英文是Labels。Labels用于对镜像的额外说明，而无法成为镜像的唯一标识。
