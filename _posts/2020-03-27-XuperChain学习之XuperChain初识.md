---
layout:     post
title:      XuperChain学习之XuperChain初识
subtitle:   百度开源的区块链XuperChain学习记录
date:       2020-03-27
author:     BaiYe
header-img: img/post-bg-01.png
catalog: true
tags:
    - 区块链
    - XuperChain
    - 学习笔记
---

# XuperChain学习（一）：XuperChain初识

## 简介
XuperChain是由百度开源的区块链基础组件，是构建区块链网络的底层方案。

## 优劣势

### 优势
高性能、灵活扩展、安全稳定、自主可控。

### 劣势
目前主要由百度主导开发，相比Hyperledger社区还不够完善，参与方和成功应用案例也比较少。

## 总体架构
![XuperChain架构](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/XuperChain架构.png)

架构总体采用可插拔、组件化的设计，可以方便的结合业务场景订制改造。分成系统内核层和网络组件层两大层，系统内核层提供平行链、提案投屏、自升级等区块链核心管理能力以及系统级API，网络组件层提供共识、智能合约、加密、帐号权限、分布式账本、P2P网络等基础组件能力。

## 核心模块
![XuperChain核心模块](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/XuperChain-Core-Module.png)

## 开源需求流程
![XuperChain开源需求研发流程](https://raw.githubusercontent.com/chunhui01/chunhui01.github.io/master/img/post_bg/XuperChain需求流程.png)

## 参考资料
- 【官网】 https://xchain.baidu.com/
- 【git仓库】 https://github.com/xuperchain/xuperchain
- 【技术文档】 https://xuperchain.readthedocs.io/zh/latest/index.html
- 【白皮书】 https://www.chainwhy.com/upload/default/20180927/05f0a782ed3d4a707d3459eef12b2aca.pdf
- 【开发流程】 https://github.com/xuperchain/XIP#%E4%B8%AD%E6%96%87-1
