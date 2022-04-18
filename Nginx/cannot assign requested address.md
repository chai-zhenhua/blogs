### 1. 先说问题
某天晚上9点左右正是业务高峰期，我们有台Nginx触发了响应状态码异常的告警，大量502。登录机器查看error log 输出的都是
```
Cannot assign requested address     # 不能分配请求地址
```

### 2. 解决方案
```
[root@node-1 ~]# cat /etc/sysctl.conf
...
net.ipv4.ip_local_port_range=1024 65000     # 调大端口范围
net.ipv4.tcp_timestamps=1                   # 开启tcp链接数据包的时间戳
net.ipv4.tcp_tw_recycle=0                   # 关闭tcp time_wait 状态的快速回收
net.ipv4.tcp_tw_reuse=1                     # 开启tcp time_wait 状态的复用
net.ipv4.tcp_max_tw_buckets=8192            # 适量调整系统中可存在的最大time_wait状态数量
...

使其生效
[root@node-1 ~]# sysctl -p
```

### 3. 为什么这么做？
首先先分析下错误信息： "Cannot assign requested address", 不能分配请求地址。 在互联网中唯一标识一个会话的根本是什么？答案是： [五元组](https://baike.baidu.com/item/%E4%BA%94%E5%85%83%E7%BB%84/6489646)

-   五元组: 源ip:源port + 传输层协议 + 目的ip:目的port

知道了五元组的概念，还需要知道每个连接都唯一对应着一个五元组, 所以 "Cannot assign requested address" 意味着当前系统无法为新的连接构造/分配一个五元组, 其实这种错误一般出现在client端，因为在一个请求中目的ip和目的端口是固定的(因为server端启动就监听了一个ip+port),协议也不会少(一般是TCP/UDP), 因此这个问题仅会发生在client端(这里指CS架构且C/S分开部署&无其他干扰的情况下)。

继续分析client端，在同一个client中 源ip是固定的， 那么唯一可变的就是源port, 那么我们基本上就可以定位到此问题出现的原因是因为client端端口不足导致的。

#### 3.1 Client的端口都去哪里了？
此时你通过监控系统或者登录机器查看此时的tcp连接状态,会发现一些异常, time_wait 状态很多
```
# 我这里并没有复现， 只是展示说明一下，实际发生时time_wait状态会很多
[root@node-1 ~]# netstat -n|awk '/^tcp/{++S[$NF]} END {for(a in S) print a,S[a]}'

ESTABLISHED 35
SYN_SENT 1
TIME_WAIT 2100
```
![如图](http://cdn.weipaitang.com/sky/common/houtaitp/image/20210330/time_wait.png/w/320)
-   所以可以发现client的端口都被time_wait状态占用了

#### 3.2 如何复用time_wait状态的连接
即然读到了这里那么我默认你是知晓tcp的11种状态机的转换机制的，那么应该也会知道time_wait状态会维持2MSL(最大报文生存时间)之后才会转换到close状态释放该五元组。
```
# 在linux系统中2MSL是固定的60s
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
				  * state, about 60 seconds	*/
```

如果不做什么改动的话死等60s对整个系统来说是不可接受的，因此我们需要做上述第2点的内核参数的改动，去快速复用time_wait状态来建立新的连接，具体的参数解释如下:

-   net.ipv4.ip_local_port_range=1024 65000

通过上面对五元组的介绍这个参数改动就很显然易见了, 在4个量都不变的情况下， 可用的端口数量越多能够建立的连接就越多

-   net.ipv4.tcp_timestamps=1
-   net.ipv4.tcp_tw_reuse=1

开启time_wait复用的前提是需要开启tcp连接的时间戳，因为tw_reuse需要依赖时间戳判断一个time_wait是否可被复用

-   net.ipv4.tcp_tw_recycle=0

快速回收time_wait状态, 该参数建议关闭(置为0), 因为在NAT的网络环境下开启这个参数会产生很严重的问题。简单来说: 该参数开启后系统会依据tcp的时间戳先后顺序来回收time_wait,但是每个机器(运行环境)它内部的tcp时间戳是不一致的(tcp的时间戳要区别于我们认为的时间戳),所以仅依据时间戳来判断该time_wait是否可回收,太鲁莽了。

-   net.ipv4.tcp_max_tw_buckets=8192

该参数是让系统来帮我们维护time_wait的状态最大数量不超过指定值,当系统中存在超过设定值的time_wait状态时,会主动回收比较老的连接。

len(net.ipv4.ip_local_port_range) - net.ipv4.tcp_max_tw_buckets > 0 那不就是意味着系统一直会有可用端口了。