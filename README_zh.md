gopush-cluster
==============
`Terry-Mao/gopush-cluster` 是一个支持集群的comet服务（支持websocket，和tcp协议）。

---------------------------------------
  * [特性](#特性)
  * [安装](#安装)
  * [使用](#使用)
  * [配置](#配置)
  * [例子](#例子)
  * [文档](#文档)
  * [更多](#更多)

---------------------------------------

## 特性
 * 轻量级
 * 高性能
 * 纯Golang实现
 * 支持消息过期
 * 支持离线消息存储
 * 支持全量推送和单个私信推送
 * 支持单个Key多个订阅者（可限制订阅者最大人数）
 * 心跳支持（应用心跳和tcp keepalive）
 * 支持安全验证（未授权用户不能订阅）
 * 多协议支持（websocket，tcp）
 * 详细的统计信息
 * 可拓扑的架构（支持增加和删除comet节点，web节点，message节点）
 * 利用Zookeeper支持故障转移

## 安装
### 一、安装依赖
```sh
$ yum -y install java-1.7.0-openjdk$ yum -y install gcc-c++
```

### 二、搭建zookeeper
1.新建目录
```sh
$ mkdir -p /data/apps$ mkdir -p /data/logs$ mkdir -p /data/programfiles
```

2.下载[zookeeper](http://www.apache.org/dyn/closer.cgi/zookeeper/)，推荐下载3.4.5版本
```sh
$ cd /data/programfiles
$ wget http://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.4.5/zookeeper-3.4.5.tar.gz
$ tar -xvf zookeeper-3.4.5.tar.gz -C ./
```
3.编译及安装
``` sh
$ cp /data/programfiles/zookeeper-3.4.5/conf/zoo_sample.cfg /data/programfiles/zookeeper-3.4.5/conf/zoo.cfg
```
4.启动zookeeper(zookeeper配置在这里不做详细介绍)
```sh
$ cd /data/programfiles/zookeeper-3.4.5/bin
$ nohup ./zkServer.sh start &
```
### 三、搭建redis
```sh
$ cd /data/programfiles
$ wget https://redis.googlecode.com/files/redis-2.6.4.tar.gz
$ tar -xvf redis-2.6.4.tar.gz -C ./
$ cd redis-2.6.4
$ make
$ make test
$ make install
$ mkdir /etc/redis
$ cp /data/programfiles/redis-2.6.4/redis.conf /etc/redis/
$ cp /data/programfiles/redis-2.6.4/src/redis-server /etc/init.d/redis-server
$ /etc/init.d/redis-server /etc/redis/redis.conf
```
* 如果如下报错,则安装tcl8.5(参考附资料2)
```sh
which: no tclsh8.5 in (/usr/lib64/qt-3.3/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin:/home/geffzhang/bin)
You need 'tclsh8.5' in order to run the Redis test
Make[1]: *** [test] error 1
make[1]: Leaving directory ‘/data/program files/redis-2.6.4/src’
Make: *** [test] error 2！
```
### 四、安装git工具（如果已安装则可跳过此步）
参考：[git](http://git-scm.com/download/linux)
```sh
$ yum -y install git
```
### 五、搭建golang环境
1.下载源码(根据自己的系统下载对应的安装包)
```sh
$ cd /data/programfiles
$ wget -c --no-check-certificate https://go.googlecode.com/files/go1.2.linux-amd64.tar.gz
$ tar -xvf go1.2.linux-amd64.tar.gz -C /usr/local
```
2.配置GO环境变量
(这里我加在/etc/profile.d/golang.sh)
```sh
$ vim /etc/profile.d/golang.sh
# 将以下环境变量添加到profile最后面
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=/data/apps/go
$ source /etc/profile
```
### 六、部署gopush-cluster
1.下载gopush-cluster及依赖包
```sh
$ ./dependencies.sh
```
* 如果提示如下,说明需要安装谷歌的hg工具（安装mercurial,参考附资料1）

go: missing Mercurial command. See http://golang.org/s/gogetcmd

package code.google.com/p/go.net/websocket: exec: "hg": executable file not found in $PATH
* 如果提示如下,说明需要安装bzr工具（参考附资料2）

go: missing Bazaar command. See http://golang.org/s/gogetcmd

2.安装message、comet、web模块
```sh
$ cd $GOPATH/src/github.com/Terry-Mao/gopush-cluster/message
$ go install
$ cp message-example.conf $GOPATH/bin/message.conf
$ cd ../comet/
$ go install
$ cp comet-example.conf /data/apps/go/bin/comet.conf
$ cd ../web/
$ go install
$ cp web-example.conf /data/apps/go/bin/web.conf
```
到此所有的环境都搭建完成！
### 七、启动gopush-cluster
```sh
$ cd /$GOPATH/bin
$ nohup ./message -c message.conf &
$ nohup ./comet -c comet.conf &
$ nohup ./web -c web.conf &
```

### 八、测试
1.推送公信（消息过期时间为expire=600秒）
```sh
$ curl -d "test2" 'http://localhost:8091/admin/push/public?expire=600'
```
成功返回：{"msg":"ok","ret":0}

2.推送私信（消息过期时间为expire=600秒）
```sh
$ curl -d "test" 'http://localhost:8091/admin/push?key=Terry-Mao&expire=600&gid=0'
```
成功返回：{“msg":"ok","ret":0}

3.获取离线消息接口
在浏览器中打开：http://localhost:8090/msg/get?key=Terry-Mao&mid=1&pmid=0
成功返回：
```json
{
    "data":{
        "msgs":[
            "{"msg":"test","expire":1391943609703654726,"mid":13919435497036558}"
        ],
        "pmsgs":[
            "{"msg":"test2","expire":1391943637016665915,"mid":13919435770166656}"
        ]
    },
    "msg":"ok",
    "ret":0
}
```
4.获取节点接口
在浏览器中打开：http://localhost:8090/server/get?key=Terry-Mao&proto=2
成功返回：
```json
{
    "data":{
        "server":"localhost:6969"
    },
    "msg":"ok",
    "ret":0
}
```
### 九、附资料
1.下载安装[hg](code.google.com/p/go.net/websocket)
```sh
$ wget http://mercurial.selenic.com/release/mercurial-1.4.1.tar.gz 
$ tar -xvf mercurial-1.4.1.tar.gz
$ cd mercurial-1.4.1
$ make
$ make install
```
* 如果安装提示找不到文件‘Python.h’ 则需要安装 python-devel
```sh
$ yum -y install python-devel
```
* 如果报错：couldn`t find libraries,则添加环境变量
```sh
$ export PYTHONPATH=/usr/local/lib64/python2.6/site-packages
```
2.安装tcl8.5
```sh
$ cd /data/programfiles
$ wget http://downloads.sourceforge.net/tcl/tcl8.5.10-src.tar.gz
$ tar -xvf tcl8.5.10-src.tar.gz -C ./
$ cd tcl8.5.10
$ cd unix
$ ./configure
$ make
$ make install
```

## 配置
### web节点的配置文件示例：
[web](https://github.com/Terry-Mao/gopush-cluster/blob/master/web/web-example.conf)

### comet节点的配置文件示例：
[comet](https://github.com/Terry-Mao/gopush-cluster/blob/master/comet/comet-example.conf)

### message节点的配置文件示例：
[message](https://github.com/Terry-Mao/gopush-cluster/blob/master/message/message-example.conf)

## 例子
java: [gopush-cluster-sdk](https://github.com/Terry-Mao/gopush-cluster-sdk)

ios: [GoPushforIOS](https://github.com/roy5931/GoPushforIOS)

javascript: [gopush-cluster-javascript-sdk](https://github.com/Lanfei/gopush-cluster-javascript-sdk)

## 文档
### web节点相关的文档：
[内部协议](https://github.com/Terry-Mao/gopush-cluster/blob/master/wiki/web/internal_proto_zh.textile)主要针对内部管理如推送消息、管理comet节点等。

[客户端协议](https://github.com/Terry-Mao/gopush-cluster/blob/master/wiki/web/external_proto_zh.textile)主要针对客户端使用，如获取节点、获取离线消息等。
### comet节点相关的文档：
[客户端协议](https://github.com/Terry-Mao/gopush-cluster/blob/master/wiki/comet/client_proto_zh.textile)主要针对客户端连接comet节点的协议说明。

[内部RPC协议](https://github.com/Terry-Mao/gopush-cluster/blob/master/wiki/comet/rpc_proto_zh.textile)主要针对内部RPC接口使用的说明。
### message节点的相关文档：
[内部RPC协议](https://github.com/Terry-Mao/gopush-cluster/blob/master/wiki/message/rpc_proto_zh.textile)主要针对内部RPC接口的使用说明。

## 更多
TODO
