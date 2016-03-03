#### 1. 介绍

> Redis is an open source (BSD licensed), in-memory data structure store, used as database, cache and message broker.

这是官方的定义。说它是一个数据库，且是把数据存到内存中，能用作cache(缓存)和消息队列。说到数据库，可能大家用得最多的是关系型数据库，比如MySQL，PostgreSQL等。这种数据库是把数据存到磁盘中的，这种能存大量的数据，然而我们的应用是经常需要访问数据库来查找数据，每次访问，无论怎样，都是需要消耗CPU和IO等待。当应用的数据积累到很庞大时，这种性能的问题更严重，所以有一种解决方法是这样的，把经常被访问的数据放到内存中，因为内存的访问速度比磁盘快太多了，而这部分数据可能不会是全部的数据，因为内存的价格比磁盘贵多了。所以有了memcached，这种就是把数据放到内存中，但它支持一种最简单的数据结构，就是键值对，这种数据库又不同于传统结构型的关系型数据库，所以又被称为nosql。

而redis也是nosql的一种，但它相比memcached不仅支持键值对，还支持列表，集合，排序集合等结构，而且它还是可持久化的。持久化就是说在内存中的数据会以文件的形式同步到磁盘中，当下次重启时还能恢复，这点相比memcached，就能存相对重要的数据，毕竟，如果数据不能持久化，丢失了也是件麻烦的事，谁也保证不了不会出问题。

还有一个很重要的特征就是redis支持分布式集群，用它可以轻易地部署到多台机器中，成为一个集群。特别是3.0开始，redis对集群的支持比较健全了。

redis比较常见的作用，第一个是cache，这是由于它的数据存在内存中，访问速度比较快，它能让数据在一定时间后过期，且又有丰富的数据结构的支持，所以它能作为一个高性能的cache。第二个是作为消息队列，用的是它的sub/pub的功能，能具有消息费生产者的模式，且是数据存在内存中，访问速度高。

![](http://aliyun.rails365.net/uploads/photo/image/31/2015/a6e1d08fd24abac5e072eba1f695a1cb.png)

#### 2. 安装

比较常见的有两种安装方式，第一是命令安装，第二是源码编译安装。命令安装可能并不会安装到最新的版本，因为软件源的redis版本也未必是最新的，但是胜在方面快速。

如果是ubuntu系统，可以这样。

##### 2.1 命令安装

如果是ubuntu系统，可以这样。

``` bash
$ sudo apt-get install redis-server
```

mac系统用以下命令。

``` bash
$ brew install redis
```

##### 2.2 编译安装

首先下载官方稳定版的tar包。

```
$ wget http://download.redis.io/releases/redis-3.0.5.tar.gz
```

解压缩编译安装。

``` bash
$ tar xvf redis-3.0.5.tar.gz
$ cd redis-3.0.5
$ make
$ sudo make install
```

这样就安装好了，接下来启动服务器。

``` bash
$ src/redis-server
```

测试。

```
src/redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```

![](http://aliyun.rails365.net/uploads/photo/image/30/2015/40c397f856c27555a5907ea8b00cabef.png)

这样很不方便。每次都要进到那个目录才能执行redis-server或redis-cli。

其实redis的源码提供了一个工具，让你能够使用类似`/etc/init.d/server start`这样的命令来启动redis服务。

只要运行utils目录下的install_server.sh脚本文件就好了。

``` bash
$ cd utils
$ sudo ./install_server.sh
```

#### 3. 配置文件

正如大多数的linux服务一样，都是通过配置文件来改变应用的特性的。redis的配置文件一般是`/etc/redis/redis.conf`。

打开该文件，里面有很多详细的注释，解决了每个参数的意义和用法。

有一些配置项可常见于大多数linux服务，比如指定pid文件的位置，指定port(端口)，日志等。

除了这些，还有一些是redis特有的，我们只是对配置文件有个大局的掌握，先介绍两个比较重要，其他的留待后绪介绍具体功能的时候再来介绍。

假如现在你装好了redis服务，放在了远程服务器上，这个时候有个安全问题要注意，那就关于登录的。

默认情况下redis是不用密码就能登录的，即使是在远程上，所以很不安全。我们要设置一下。

在配置文件中把下面一行开启就好了。

```
bind 127.0.0.1
```

这代表redis和你本机的linux机器绑定在一起，只有那台linux机器才能登录。

还有另一个问题，就是关于内存的。作为缓存服务器，如果不加以限制内存的话，就很有可能出现将整台服务器内存都耗光的情况，可以在redis的配置文件里面设置。

``` bash
# 限定最多使用1.5GB内存
maxmemory 1536mb
```

另外，当你的redis已经在用了，这个时候你又要改配置，就是需要类似重新加载配置文件的功能。

而redis也提供了`CONFIG SET`和`CONFIG GET`功能，这个可以在线上更改redis的配置。

完结。
