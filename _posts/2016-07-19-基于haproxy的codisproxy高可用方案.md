## 基于haproxy的coids proxy高可用方案

### 简介

HAProxy是一款提供高可用性、负载均衡以及基于TCP（第四层）和HTTP（第七层）应用的代理软件，HAProxy是完全免费的、借助HAProxy可以快速并且可靠的提供基于TCP和HTTP应用的代理解决方案。

1. 免费开源；
2. 高性能，官方给的性能数据很高，具体需要测试压测决定；
3. 支持连接拒绝；
4. 支持全透明代理；
5. 自带监控；
6. 支持虚拟主机；

### 安装

```shell
mkdir -p /data/scguo/haproxy
cd /data/scguo/haproxy
wget http://www.haproxy.org/download/1.6/src/haproxy-1.6.7.tar.gz
tar -zxvf haproxy-1.6.7.tar.gz
make TARGET=linux26 ARCH=X84_64 PREFIX=/data/scguo/haproxy/sbin
make install PREFIX=/data/scguo/haproxy/sbin
```

### 使用

#### 结构

![haproxy]({{site.github.url}}/assets/haproxy/codis.png)

#### 配置

```properties
global
        maxconn 6000
defaults
        timeout client		30s
        timeout server		30s
        timeout connect		30s
        retries	3
listen	stats
        bind	0.0.0.0:8899
        mode            http
        log             global
        maxconn 10
        timeout client 	100s
        timeout server 	100s
        timeout connect 100s
        timeout queue   100s
        stats enable
        stats hide-version
        stats refresh 5s
        stats show-node
        stats auth admin:admin
        stats uri  /haproxy?stats
frontend MyFrontend
	bind	0.0.0.0:6389
	default_backend		codis

backend codis
	mode			tcp
	server			proxy1 192.168.45.170:19000 weight 1 check port 19000 inter 1s rise 2 fall 2
	server			proxy2 192.168.45.170:19001 weight 1 check port 19000 inter 1s rise 2 fall 2
```

#### 测试

启动haproxy

```shell
./haproxy -f haproxy.cfg
```

##### 功能测试

使用redis-cli连接haproxy验证功能

![功能验证]({{site.github.url}}/assets/haproxy/通过haproxy访问.png)

查看haproxy的状态

![正常状态]({{site.github.url}}/assets/haproxy/正常状态.png)

kill掉一个代理，验证是否可用

![]({{site.github.url}}/assets/haproxy/kill掉一个代理.png)

kill掉一个代理的haproxy状态

![]({{site.github.url}}/assets/haproxy/kill掉一个代理状态.png)

##### 性能测试

这里使用set进行测试

| 线程数  | haproxy  | codis-proxy |
| :--: | :------: | :---------: |
|  20  | 16430.88 |  22815.42   |
|  50  | 20554.14 |  40461.26   |
| 100  | 23183.03 |  58414.63   |

### 对比

#### LVS

>1、抗负载能力强。抗负载能力强、性能高，能达到F5硬件的60%；对内存和cpu资源消耗比较低
>2、工作在网络4层，通过vrrp协议转发（仅分发），具体的流量由linux内核处理，没有流量产生。
>3、稳定性、可靠性好，自身有完美的热备方案；（如：LVS+Keepalived）
>4、应用范围比较广，可以对所有应用做负载均衡；
>5、不支持正则处理，不能做动静分离。
>6、支持负载均衡算法：rr（轮循）、wrr（带权轮循）、lc（最小连接）、wlc（权重最小连接）
>7、配置 复杂，对网络依赖比较大，稳定性很高。

#### Nginx

>1、工作在网络的7层之上，可以针对http应用做一些分流的策略，比如针对域名、目录结构；
>2、Nginx对网络的依赖比较小，理论上能ping通就就能进行负载功能；
>3、Nginx安装和配置比较简单，测试起来比较方便；
>4、也可以承担高的负载压力且稳定，一般能支撑超过1万次的并发；
>5、对后端服务器的健康检查，只支持通过端口来检测，不支持通过url来检测；
>6、Nginx对请求的异步处理可以帮助节点服务器减轻负载；
>7、Nginx仅能支持http、https和Email协议，这样就在适用范围较小；
>8、不支持Session的直接保持，但能通过ip_hash来解决。对Big request header的支持不是很好；
>9、支持负载均衡算法：Round-robin、Weight-round-robin、Ip-hash；
>10、Nginx还能做Web服务器即Cache功能。

#### HaProxy

>1、支持两种代理模式：TCP（四层）和HTTP（七层），支持虚拟主机；
>2、能够补充Nginx的一些缺点比如Session的保持，Cookie的引导等工作
>3、支持url检测后端的服务器出问题的检测会有很好的帮助。
>4、更多的负载均衡策略比如：动态加权轮循(Dynamic Round Robin)，加权源地址哈希(Weighted Source Hash)，加权URL哈希和加权参数哈希(Weighted Parameter Hash)已经实现
>5、单纯从效率上来讲HAProxy更会比Nginx有更出色的负载均衡速度。
>6、HAProxy可以对Mysql进行负载均衡，对后端的DB节点进行检测和负载均衡。
>9、支持负载均衡算法：Round-robin（轮循）、Weight-round-robin（带权轮循）、source（原地址保持）、RI（请求URL）、rdp-cookie（根据cookie）
>10、不能做Web服务器即Cache。

### codis高可用

![高可用]({{site.github.url}}/assets/haproxy/高可用.jpg)