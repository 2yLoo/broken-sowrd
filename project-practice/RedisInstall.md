# Redis安装爬坑

![](https://redis.io/images/redis-white.png)

Redis的安装十分简单，在[Redis官网](https://redis.io/download)，通过4行命令的复制粘贴即可完成一个Redis指定版本的安装。

曾经，我也对Redis的各版本进行了好几次安装，且一身轻松。

但Linux环境错综复杂，你可以选择你的人生，但无法选择你的出生环境。

即使Redis的安装便捷，但Linux机器的环境还是一个爹。

先看正常流程：
```
// 下载安装包
$ wget http://download.redis.io/releases/redis-5.0.5.tar.gz
// 解压安装包
$ tar xzf redis-5.0.5.tar.gz
// 进入Redis目录
$ cd redis-5.0.5
// 编译
$ make
```

正常流程走完，再执行```src/redis-server```即可启动Redis服务。

官网的Download首页只贴出了最新稳定版的Redis安装教程，如有需求，可将安装包路径替换为指定版本，各版本路径参考[Redis各版本地址](http://download.redis.io/releases/)。

在以前，Redis的安装在此就云淡风轻的结束了，直到有一次，遇见了他。

辣是一个面无表情的蓝人，在我输入make敲下回车后，他像一个铁憨憨杵在我面前：
```
$ make
cd src && make all
make[1]: 进入目录“/data/softwares/redis-5.0.5/src”
    CC Makefile.dep
make[1]: 离开目录“/data/softwares/redis-5.0.5/src”
make[1]: 进入目录“/data/softwares/redis-5.0.5/src”
rm -rf redis-server redis-sentinel redis-cli redis-benchmark redis-check-rdb redis-check-aof *.o *.gcda *.gcno *.gcov redis.info lcov-html Makefile.dep dict-benchmark
(cd ../deps && make distclean)
make[2]: 进入目录“/data/softwares/redis-5.0.5/deps”
(cd hiredis && make clean) > /dev/null || true
(cd linenoise && make clean) > /dev/null || true
(cd lua && make clean) > /dev/null || true
(cd jemalloc && [ -f Makefile ] && make distclean) > /dev/null || true
(rm -f .make-*)
make[2]: 离开目录“/data/softwares/redis-5.0.5/deps”
(rm -f .make-*)
echo STD=-std=c99 -pedantic -DREDIS_STATIC='' >> .make-settings
echo WARN=-Wall -W -Wno-missing-field-initializers >> .make-settings
echo OPT=-O2 >> .make-settings
echo MALLOC=jemalloc >> .make-settings
echo CFLAGS= >> .make-settings
echo LDFLAGS= >> .make-settings
echo REDIS_CFLAGS= >> .make-settings
echo REDIS_LDFLAGS= >> .make-settings
echo PREV_FINAL_CFLAGS=-std=c99 -pedantic -DREDIS_STATIC='' -Wall -W -Wno-missing-field-initializers -O2 -g -ggdb   -I../deps/hiredis -I../deps/linenoise -I../deps/lua/src -DUSE_JEMALLOC -I../deps/jemalloc/include >> .make-settings
echo PREV_FINAL_LDFLAGS=  -g -ggdb -rdynamic >> .make-settings
(cd ../deps && make hiredis linenoise lua jemalloc)
make[2]: 进入目录“/data/softwares/redis-5.0.5/deps”
(cd hiredis && make clean) > /dev/null || true
(cd linenoise && make clean) > /dev/null || true
(cd lua && make clean) > /dev/null || true
(cd jemalloc && [ -f Makefile ] && make distclean) > /dev/null || true
(rm -f .make-*)
(echo "" > .make-cflags)
(echo "" > .make-ldflags)
MAKE hiredis
cd hiredis && make static
make[3]: 进入目录“/data/softwares/redis-5.0.5/deps/hiredis”
gcc -std=c99 -pedantic -c -O3 -fPIC  -Wall -W -Wstrict-prototypes -Wwrite-strings -g -ggdb  net.c
make[3]: gcc：命令未找到
make[3]: *** [net.o] 错误 127
make[3]: 离开目录“/data/softwares/redis-5.0.5/deps/hiredis”
make[2]: *** [hiredis] 错误 2
make[2]: 离开目录“/data/softwares/redis-5.0.5/deps”
make[1]: [persist-settings] 错误 2 (忽略)
    CC adlist.o
/bin/sh: cc: 未找到命令
make[1]: *** [adlist.o] 错误 127
make[1]: 离开目录“/data/softwares/redis-5.0.5/src”
make: *** [all] 错误 2
```

我心里：？？？

后来多番在网上查找，原来是因为系统未安装GCC环境。

不愿意透露姓名的网友说要安装GCC环境，然后再make clean一哈，再次make，如果仍然失败，可以删除整个Redis目录，重新解压，重新make。

嗯嗯，记住了。

那就安装一下GCC吧：
```
sudo yum -y install gcc
```

好景不长，结果安装GCC再次报错：
```
···balabala···
** 发现 3 个已存在的 RPM 数据库问题， 'yum check' 输出如下：
glibc-2.17-222.173.amzn1.x86_64 是 glibc-2.17-106.167.amzn1.x86_64 的副本
glibc-common-2.17-222.173.amzn1.x86_64 是 glibc-common-2.17-106.167.amzn1.x86_64 的副本
nss-softokn-freebl-3.28.3-8.41.amzn1.x86_64 是 nss-softokn-freebl-3.16.2.3-14.2.38.amzn1.x86_64 的副本
```

有3个RPM的包版本冲突，使用RPM命令分别查找这几个包，的确有包冲突的情况存在：
```
$ rpm -qa |grep nss-softokn-freebl
nss-softokn-freebl-3.28.3-8.41.amzn1.x86_64
nss-softokn-freebl-3.16.2.3-14.2.38.amzn1.x86_64

$ rpm -qa |grep glibc
glibc-common-2.17-106.167.amzn1.x86_64
glibc-2.17-222.173.amzn1.x86_64
glibc-2.17-106.167.amzn1.x86_64
glibc-common-2.17-222.173.amzn1.x86_64
```

征询Leader意见后，决定保留高版本包。

删除低版本的glibc：
```
$ rpm -e glibc-2.17-106.167.amzn1.x86_64
错误：依赖检测失败：
	glibc(x86-64) = 2.17-106.167.amzn1 被 (已安裝) glibc-common-2.17-106.167.amzn1.x86_64 需要
```

啊，它被Common包依赖了，得先删除Common包：
```
$ rpm -e glibc-common-2.17-106.167.amzn1.x86_64
错误：依赖检测失败：
	glibc-common = 2.17-106.167.amzn1 被 (已安裝) glibc-2.17-106.167.amzn1.x86_64 需要
```

我内心再次泛起涟(问)漪(号)，它们还能互相依赖的？

那就用在rpm命令后追加--nodeps吧：
```
$ sudo rpm --nodeps -e glibc-common-2.17-106.167.amzn1.x86_64

$ sudo rpm -e glibc-2.17-106.167.amzn1.x86_64

$ sudo rpm -e nss-softokn-freebl-3.16.2.3-14.2.38.amzn1.x86_64
```

DONE.

接下来重新安装GCC：
```
sudo yum -y install gcc
```

GCC环节终于翻篇。耳边不断回响起网友的话，先make clean，再重新make，如果重新make报错了，心态稳住，删除rm -rf Redis目录，回到最初的起点，按照官网的4条命令顺序执行。

BRAVO!🎉

Redis安装成功！😘
