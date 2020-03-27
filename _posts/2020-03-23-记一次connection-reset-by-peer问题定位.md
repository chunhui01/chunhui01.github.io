---
layout:     post
title:      记一次connection-reset-by-peer问题定位
subtitle:   一次linux下connection reset by peer问题定位记录
date:       2020-03-23
author:     BaiYe
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Linux
    - Connection reset by peer
    - TCP
    - 网络
---

## 背景

有同事报客户端请求某核心服务出现大量connection reset by peer。线上故障，赶紧高优一起定位处理。

## 故障处理

### 线上故障处理原则
1.及时通报，及时止损。

2.保留现场，定位问题。

3.彻底修复，故障总结。

### 及时止损

看现象是个别实例集中出现，不是全部实例出现，那就和运行环境、流量、或者某个资源有关系。按照及时止损的原则，首先验证重启能否恢复，验证重启可以恢复，联系OP快速操作重启，服务恢复。由于不是稳定复现问题，需要保留现场用于问题定位，让OP保留两个故障实例，用作问题定位（保留的实例临时屏蔽流量）。

重启大法快速完成止损，服务恢复，观察段时间运行稳定。然后可以不慌不忙定位问题了。

注意：

1.重启大法虽然暴力，但是有用，关键时刻别忘记了，止损最重要。

2.由于各种原因，大部分公司的上线系统并没有OP直接操作快，影响大的故障优先联系OP支持。

3.必要时及时联系各相关角色支持，人多力量大，快速止损最重要，但是人多了要对信息有判断力，不要被带跑偏了。

4.合理、及时、明确的通报故障和处理进展。

5.止损后不要忘记彻底定位修复问题，要不指不准那个夜晚(正做着香甜的美梦)会被电话call醒，然后熬个通宵。

## 问题定位

线上问题定位其实和断案非常像，先初步收集信息，再根据收集到的有限的信息推断可能的真相，再定向寻找证据证明自己的推断，再设计实验模拟复现确认自己的推断。

绝大部分问题都是可以通过模拟复现的，只是有些问题找到一条正确复现的路径比较费劲，找到这条复现路径也就基本能发现问题了。通常是应用系统提供的相应工具分析问题case，获取详细的信息，根据这些信息结合相关知识，推断造成这个现象可能的原因，设计复现的途径，然后开发机模拟实验确认问题。

不能复现的问题可能和流量、机器的瞬时环境、依赖服务的瞬时抖动等有关系，处理这类问题完善的监控和日志就非常重要了，服务上线后要接入相关机器资源、流量、错误的监控，开发时日志记录要完善。日志通常是定位线上问题最重要也最高效的方式，开发阶段一定要重视日志(日志原则见另一篇博文)。

问题分析一般是从问题表象切入，结合问题表象和相关知识，寻找方向，逐个深入分析确认疑点，逐步找到那个最可能的原因。

### 问题分析

1.客户端请求出现connection reset by peer，验证问题实例稳定复现。

```
curl -v 'http://10.xx.xx.35:2133/xx/xx/checkalive'
```

2.查看日志，并没有access日志输出，而且响应connection reset by peer。

```
tail -f ./log/xxx.log
```

3.通过tcpdump查看请求详细数据包情况（有些机器tcpdump按照路径没有在PATH里，可以通过whereis检索下具体按照路径使用，通过ifconfig查看网络设备名）。通过tcpdump结果发现，TCP三次握手完成，在发送数据时服务端没有响应ACK，而响应了reset，导致客户端http请求响应connection reset by peer。
```
whereis tcpdump
tcpdump: /usr/sbin/tcpdump /usr/share/man/man8/tcpdump.8.gz /usr/share/man/man1/tcpdump.1 /usr/share/man/man1/tcpdump.1.gz

ifconfig
eth0      Link encap:Ethernet
lo        Link encap:Local Loopback

/usr/sbin/tcpdump -i eth0 -n -nn host 10.xx.xx.35

发现TCP三次握手完成，在发送数据时服务端没有响应ACK，而响应了reset，导致客户端http请求响应connection reset by peer。
```

3.服务端通过listen(sockfd, backlog)方法告诉内核监听该socket并设置队列大小（未完成链接队列+已完成连接队列），然后当客户端通过connect()方法请求链接时，由系统内核完成TCP三次握手，并把请求放入已完成连接队列，等待调用accept()方法取走，accept()需要先通过socket()创建新的句柄。golang实现是：框架通过net/http包Server.Serve()方法开启服务，标准库中通过net包TCPListener.AcceptTCP()等待获取新的链接，最终通过internal/poll包的accept()发起系统调用accept4() or accept()，golang这个accept和c的accept()还不一样，golang不需要提前创建套接字句柄传入，而且由accept()直接返回新套接字句柄。也就是客户端请求时，内核完成了TCP三次握手，并把请求放入已完成连接队列，但是accept时发生了错误，直接响应了客户端reset。accept发生错误最常见就是句柄被打满了，查看进程监听端口链接情况和进程句柄使用情况。

```
net/http/server.go func (srv *Server) Serve(l net.Listener) error
net/tcpsock.go func (l *TCPListener) AcceptTCP() (*TCPConn, error)
net/tcpsock_posix.go func (ln *TCPListener) accept() (*TCPConn, error)
net/fd_unix.go  func (fd *netFD) accept() (netfd *netFD, err error)
internal/poll/fd_unix.go  func (fd *FD) Accept() (int, syscall.Sockaddr, string, error)
internal/poll/sock_cloexec.go func accept(s int) (int, syscall.Sockaddr, string, error)
```

4.通过netstat or ss查看监听端口的链接情况，通过lsof查看进程句柄占用情况，通过ulimit查看系统限制。发现果然进程句柄被打满了，超过了10240的限制。确认是由于进程句柄被打满导致客户端请求响应connection reset by peer。同时通过netstat的统计信息还发现，处于CLOSE_WAIT状态的链接很多，但是也远小于打开的句柄数。至此，虽然明确了客户端请求会响应connection reset by peer是由于服务进程句柄被打满导致的，但是依然不知道什么原因导致了服务进程句柄被打满。
```
netstat -an | grep port 或者  ss -ant | grep port
lsof -p port
ulimit -a
```

5.CLOSE_WAIT状态链接太多，可能会占用大量句柄，从CLOSE_WAIT状态入手分析。结合TCP状态机，四次挥手过程中，被动关闭的一方收到第一次断开链接的FIN包后进入CLOSE_WAIT状态，等待发送完数据，然后发出第二次FIN包后进入LAST_ACK状态，收到对端ACK后进入CLOSED状态完成，另外CLOSE_WAIT状态有超时时间（一般默认是2H），超时会被系统关闭。三次握手实在系统内核完成的，但是四次挥手由于要等待数据发送完成，是和应用程序相关的，内核收到第一个FIN后会通知应用程序，应该是应用程序要响应后才能再发送第二个FIN。
结合这些信息猜测：服务句柄是被逐渐累积打满的，出现大量CLOSE_WAIT是由于客户端先断开链接（很可能是请求超时），服务端在收到客户端超时端口请求后，由于用户态请求处理阻塞，导致第二次FIN无法发送，而且应该是出现了死锁等问题，持久阻塞（句柄一致没有被释放）。客户端应该是先有大量io timeout，等服务端句柄被打满后才出现connect reset by peer的，而客户端io timeout增多很可能是服务端处理请求耗时突增或者阻塞导致。
理论上能解释通了，线下模拟实现验证，在接口中sleep(100s)，压测很快就复现了connect reset by peer，现象和线上问题case完全一致，确认猜想。那么接下来定位的重点就是为什么服务端会突然出现阻塞？由于不稳定复现，是什么触发了阻塞？

#### SOCKET工作流程
![套接字工作流程](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/套接字工作流程.png)

#### epoll
![epoll原理](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/epoll原理.png)

#### TCP状态流转图：
![TCP状态流转图](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/TCP状态流转图.png)

#### TCP SOCKET状态表：
- CLOSED: 关闭状态，没有连接活动
- LISTEN: 监听状态，服务器正在等待连接进入
- SYN_SENT: 已经发出连接请求，等待确认 
- SYN_RCVD: 收到一个连接请求，尚未确认 
- ESTABLISHED: 连接建立，正常数据传输状态 
- FIN_WAIT_1:（主动关闭）已经发送关闭请求，等待确认 
- FIN_WAIT_2:（主动关闭）收到对方关闭确认，等待对方关闭请求 
- CLOSE_WAIT:（被动关闭）收到对方关闭请求，已经确认 
- LAST_ACK: （被动关闭）等待最后一个关闭确认，并等待所有分组死掉 
- TIMED_WAIT: 完成双向关闭，等待所有分组死掉 
- CLOSING: 双方同时尝试关闭，等待对方确认 

#### 三次握手
![TCP三次握手](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/TCP三次握手.png)

#### 四次挥手
![TCP四次挥手](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/TCP四次挥手.png)

6.到了应用程序层面，要分析进程过去发生了什么，只能从应用日志和服务监控入手了，从历史监控曲线（内存、句柄、流量、耗时等）查找可能出现异常的时间点，再找关键时间点的日志仔细分析。发现刚开始是处理耗时增长，然后只能输出access_log，最后才到请求无日志输出，从日志完成验证上面的分析猜想。发现耗时突增是关键点，仔细分析业务日志，发现是请求DB耗时增加，再进一步看访问DB的统计信息，发现DB连接池一直在被打满，请求排队等空闲待链接，导致请求处理耗时增加，然后排队请求越来越多，直到句柄数被打满。由于DB连接池新建链接需要句柄，句柄被排队等空闲链接的请求给打满了，形成了死锁。也就出现了从超时到句柄被打满还无法释放的情况。线上环境修改DB连接池配置，压测果然很快复现了。至此，终于发现了真相（哭晕，再次证明了完善的日志和监控的重要性）。

```
type DBStats struct {
    MaxOpenConnections int // Maximum number of open connections to the database; added in Go 1.11

    // Pool Status
    OpenConnections int // The number of established connections both in use and idle.
    InUse           int // The number of connections currently in use; added in Go 1.11
    Idle            int // The number of idle connections; added in Go 1.11

    // Counters
    WaitCount         int64         // The total number of connections waited for; added in Go 1.11
    WaitDuration      time.Duration // The total time blocked waiting for a new connection; added in Go 1.11
    MaxIdleClosed     int64         // The total number of connections closed due to SetMaxIdleConns; added in Go 1.11
    MaxLifetimeClosed int64         // The total number of connections closed due to SetConnMaxLifetime; added in Go 1.11
}
```

### 故障原因
为了防止DB链接数被打满，刚开始DB连接池最大连接数配置的比较小，流量慢慢上涨逼近平衡点。当天由于运营活动稍微增加的点流量就成了压死骆驼的最后一根稻草，导致查询DB请求排队等待空闲链接，排队时间越长积压的请求越多，请求处理耗时越大，直到积压请求太多把句柄打满，出现了死锁。

### 复现验证
修改DB最大连接数配置压测，很快就能复现。

## 问题修复
去掉DB连接池最大连接数限制。

## 总结反思
1.重要服务日志、统计、监控一定要全，日志最少要保留7天，核心错误和统计信息一定要输出（比如DB连接池的统计信息），统计和监控要持久保存可以追溯，cpu、内存、句柄、磁盘占用、磁盘io、网络io等机器资源这些一定要有监控，各关键请求的耗时一定要输出到日志，请求整体耗时要监控起来。

2.服务相关配置，包括机器相关配置也要跟的上流量上涨。

3.对系统底层知识（内功）和常用的系统工具（招式）要熟练，要不遇到网络类问题特别容易抓瞎。

4.要从开发阶段重视日志的重要性，完备又不多余的输出日志。

5.定位线上问题时，结合监控+系统工具+日志定位，优先仔细分析日志，如果日志完备，大部分问题都能从日志中发现。

## 常用工具
```
curl -v 'http://10.xx.xx.35:21xx/xx/xx/checkalive'
whereis tcpdump
ifconfig
/usr/sbin/tcpdump -i eth0 -n -nn host 10.xx.xx.35
netstat -an | grep xxxx
ps -ef | grep xxx
lsof -p xxx
ulimit -a
pmap -x xxx
cat /proc/$pid/smaps
strace -p $pid
pstack $pid
ls /proc/$pid/fd/  | wc -l
```

## 参考资料
- [TCP协议总结]https://www.jianshu.com/p/4697b2781e86
- [浅谈TCP四次挥手]https://blog.csdn.net/yeweilei/article/details/79279963
- [线上大量CLOSE_WAIT原因深入分析]https://segmentfault.com/a/1190000017313251?utm_source=tag-newest
- [Socket之accept与三次握手的关系]https://blog.csdn.net/smart55427/article/details/8431827?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task
- [epoll原理剖析]https://medium.com/@heshaobo2012/epoll%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90-3-epoll-bf9cdcf5e50
- [epoll原理图解]https://blog.csdn.net/qq_35433716/article/details/85345907
- [TCP网络编程中connect()、listen()和accept()三者之间的关系]https://blog.csdn.net/dengjin20104042056/article/details/52357452
- [从源码角度看Golang的TCP Socket(epoll)实现]https://www.jianshu.com/p/3ff0751dfa04
