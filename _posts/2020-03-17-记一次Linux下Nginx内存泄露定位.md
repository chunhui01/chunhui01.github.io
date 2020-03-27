---
layout:     post
title:      记一次Linux下Nginx内存泄露定位
subtitle:   一次Linux下Nginx内存泄露问题分析
date:       2020-03-17
author:     BaiYe
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Linux
    - Nginx
    - 内存泄露
---

## 背景

有同事报他的机器上nginx存在内存泄露，都吃了4G内存没法忍了，于是赶紧查查。

## 问题定位

1.先top -u  work 查看进程内存占用情况，确认确实是占了4G没法忍了（图239M是后来补的别在意）。

![](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/20200317_01_1.png)

2.查看nginx进程确实是业务的nginx的某个worker子进程疑似存在内存泄露占了大量内存。
ps -ef | grep nginx | grep -v grep | grep work

3.为什么不是几个worker进程都内存增大，只是个别worker进程内存占用很大？ 查看日志，过滤掉干扰内容。

- cat error.log | grep -v '\[dns\]' | grep -v 'access forbidden by rule’
![](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/20200317_01_2.png)
  发现并不是那个子进程没有内存泄露，而是那个子进程频繁被kill，然后master又重启新的子进程。查询系统日志确认。

- dmesg | grep pid
  发现并不是那个子进程没有内存泄露，而是那个子进程频繁被kill，然后master又重启新的子进程。查询系统日志确认。
![](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/20200317_01_3.png)
  确认那些内存占用底的worker进程是被oom kill了，然后被master又重启新的子进程。

4.确定了是指定进程内存泄露后，查看该进程的内存分配，定位泄露信息。
- dump出改进程的内存分配，确认确实存在超大块内存分配。
  pmap -x 42102

![](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/20200317_01_4.png)

- 查看内存段的具体起始位置。
  cat /proc/42102/smaps

![](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/20200317_01_5.png)

- 通过gdb dump出那段内存存储内容。
gdb -p 4210
dump binary memory ./memory2.log 0x7fa1d0b57000 0x7FA1D0B70000

![](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/20200317_01_6.png)

- 查看dump出的内容，发现都是某个扩展的配置数据，确定是该扩展存在内存泄露。

至此，确定是某个nginx扩展存在内容泄露，该扩展进一步分析代码定位解决。

## Linux进程内存分析工具

- top：查看机器整体内存使用情况和各进程内存使用情况。
VIRT：虚拟内存。
RES：常驻内存，一般比较关心这个。
SHR：共享内存。
DATA：数据占用内存。
参考：[https://javawind.net/p131](https://javawind.net/p131)

- pmap：pmap -x pid dump进程的内存分配情况。

- cat /proc/pid/smaps : 查看内存块具体开始结束位置。

- gdb -p pid  连接到进程，gdb调试，当然gdb功能非常强大。
  dump binary memory ./out.log 0x7fa1d0b57000 0x7FA1D0B70000  dump出指定位置存储的内容。

- mtrace 可以跟踪记录进程的内存分配。
参考：[https://www.jianshu.com/p/d9e12b66096a](https://www.jianshu.com/p/d9e12b66096a)

